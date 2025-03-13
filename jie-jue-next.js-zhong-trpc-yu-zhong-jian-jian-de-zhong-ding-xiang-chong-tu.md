# 解决Next.js中tRPC与中间件的重定向冲突

本文将探讨一个常见问题：**认证中间件如何导致tRPC API调用被意外重定向**，并提供解决方案和最佳实践。

### 问题背景

在使用Next.js与认证系统（如Clerk）时，我们通常会设置中间件来保护路由和处理用户认证流程。同时，如果我们使用tRPC作为类型安全的API层，这些中间件可能会无意中干扰API调用。

#### 症状识别

如果你的tRPC调用遇到了以下问题，你可能正面临我们讨论的问题：

* tRPC请求返回307临时重定向而不是正常响应
* 在Network选项卡中看到API请求被重定向到登录页面或其他页面
* 即使用户已登录，仍然无法完成特定的API操作

### 问题分析

让我们看看导致这个问题的典型中间件配置：

```typescript
// middleware.ts
export default clerkMiddleware(async (auth, req: NextRequest) => {
    const { userId, sessionClaims } = await auth()

    // 如果访问onboarding路由但未登录，重定向到登录页面
    if (!userId && isOnboardingRoute(req)) {
        const signInUrl = new URL("/sign-in", req.url)
        signInUrl.searchParams.set("redirect_url", req.url)
        return NextResponse.redirect(signInUrl)
    }

    // 已登录但未完成onboarding的用户
    if (userId && !sessionClaims?.metadata?.onboardingComplete) {
        if (isOnboardingRoute(req)) {
            return NextResponse.next()
        }
        
        const onboardingUrl = new URL("/onboarding", req.url)
        return NextResponse.redirect(onboardingUrl)
    }

    // 其他路由保护逻辑...

    return NextResponse.next()
})

export const config = {
    matcher: ["/((?!.*\\..*|_next).*)", "/", "/(api|trpc)(.*)"],
}
```

**问题关键点**：matcher配置包含了`/(api|trpc)(.*)`，这意味着所有API请求（包括tRPC请求）都会经过这个中间件。当tRPC mutation尝试更新onboarding状态时，中间件会检测到用户尚未完成onboarding并重定向请求，阻止了API操作的完成。

### 解决方案

#### 方案1：排除tRPC路由（推荐）

最简单的解决方案是修改中间件，让它在处理tRPC请求时跳过重定向逻辑：

```typescript
export default clerkMiddleware(async (auth, req: NextRequest) => {
    const { userId, sessionClaims } = await auth()
    
    // 对API/tRPC请求跳过重定向逻辑
    if (req.nextUrl.pathname.startsWith('/api/trpc')) {
        // 仅进行认证检查，不重定向
        if (isProtectedRoute(req)) await auth.protect()
        return NextResponse.next()
    }

    // 其余的中间件逻辑保持不变...
})
```

这个解决方案的优势在于它保持了API路由的正常工作方式，同时仍然保护UI路由不被未授权访问。

#### 方案2：修改matcher配置

另一种方法是修改matcher配置，完全排除tRPC路由：

```typescript
export const config = {
    matcher: [
        "/((?!.*\\..*|_next).*)", 
        "/",
        "/(api(?!/trpc))(.*)" // 包括api路由但排除/api/trpc
    ],
}
```

#### 方案3：使用条件重定向

为所有路由添加更精细的条件判断：

```typescript
if (userId && !sessionClaims?.metadata?.onboardingComplete) {
    // 仅为非API请求添加重定向
    if (!req.nextUrl.pathname.startsWith('/api/')) {
        if (isOnboardingRoute(req)) {
            return NextResponse.next()
        }
        
        const onboardingUrl = new URL("/onboarding", req.url)
        return NextResponse.redirect(onboardingUrl)
    }
}
```

### 最佳实践

以下是处理Next.js中间件与tRPC集成的一些最佳实践：

1. **分离关注点**: API请求和页面导航应该有不同的处理逻辑
2. **早期检查**: 在中间件开始处检查请求类型，并为API请求提供快速路径
3. **明确的matcher配置**: 使用精确的matcher模式，避免不必要的中间件执行
4. **双重保护**: 即使API路由绕过导航重定向，也应保持认证检查
5. **错误处理**: API路由应该返回适当的错误状态码而不是重定向
6. **避免共享matcher**: 考虑为API和UI路由使用不同的中间件

#### 推荐的中间件结构

```typescript
export default clerkMiddleware(async (auth, req: NextRequest) => {
    const { userId, sessionClaims } = await auth()
    
    // 1. 区分API和UI请求
    const isApiRequest = req.nextUrl.pathname.startsWith('/api/');
    
    // 2. API请求的快速路径
    if (isApiRequest) {
        // 仍然执行认证检查，但不重定向
        if (isProtectedRoute(req)) {
            try {
                await auth.protect();
                return NextResponse.next();
            } catch (error) {
                // 对于API返回401而不是重定向
                return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
            }
        }
        return NextResponse.next();
    }
    
    // 3. UI路由的正常导航逻辑
    // ...其余的中间件逻辑
})

export const config = {
    matcher: [
        "/((?!.*\\..*|_next).*)", // UI路由
        "/", 
        "/(api|trpc)(.*)" // API路由
    ],
}
```

### 总结

在构建使用Next.js、tRPC和认证系统的应用时，中间件配置需要特别注意。通过正确区分API请求和页面导航，你可以避免意外的重定向问题，同时保持应用的安全性。

以上提供的解决方案不仅解决了当前的问题，还体现了一个更广泛的原则：**API请求和UI导航在认证流程中应该有不同的处理方式**。API请求通常应该返回状态码而不是重定向，而页面请求则可能需要更复杂的导航逻辑。

通过遵循这些最佳实践，你可以构建一个既安全又用户友好的应用，同时保持代码的可维护性和可扩展性。

***

你的应用架构是否遇到过类似的问题？你是如何解决的？欢迎在评论中分享你的经验！
