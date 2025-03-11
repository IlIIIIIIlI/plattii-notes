# 解决重复组件问题：共享UI但权限不同的场景

当管理员和普通用户使用相同界面但只是权限不同时，如果使用嵌套路由组会导致组件重复问题。这在实际项目中很常见，有几种更优的解决方案：

### 1. 基于角色的条件渲染（推荐）

使用单一路由结构，但在组件内部根据用户角色条件渲染不同功能：

```tsx
// app/(protected)/dashboard/page.tsx
import { auth } from '@clerk/nextjs'

export default async function DashboardPage() {
  const { has } = auth();
  const isAdmin = await has({ permission: 'admin' });
  
  return (
    <div>
      <h1>Dashboard</h1>
      {/* 共享UI部分 */}
      <CommonDashboardMetrics />
      
      {/* 管理员特有功能，条件渲染 */}
      {isAdmin && <AdminControls />}
    </div>
  );
}
```

**优势**：

* 避免代码重复
* 单一路由结构，更易于维护
* 遵循最小权限原则，UI根据用户权限自适应

### 2. 使用中间件级别的功能开关

在中间件中注入权限信息到请求头，然后在组件中使用：

```typescript
// middleware.ts
export default clerkMiddleware(async (auth, req) => {
  // 检查用户权限
  const isAdmin = await auth.has({ permission: 'admin' });
  
  // 将权限信息添加到请求头
  const headers = new Headers(req.headers);
  headers.set('x-user-permissions', isAdmin ? 'admin' : 'user');
  
  // 继续请求，但带上修改后的头信息
  return NextResponse.next({
    request: {
      headers
    }
  });
});
```

然后在Server Component中：

```tsx
// app/(protected)/dashboard/page.tsx
import { headers } from 'next/headers'

export default function DashboardPage() {
  const headersList = headers();
  const userPermissions = headersList.get('x-user-permissions');
  const isAdmin = userPermissions === 'admin';

  return (
    <div>
      {/* 根据权限渲染不同功能 */}
    </div>
  );
}
```

### 3. 使用HOC（高阶组件）模式

创建一个权限包装器，用于保护特定功能：

```tsx
// components/withPermission.tsx
'use client'

import { useAuth } from '@clerk/nextjs';
import { ReactNode } from 'react';

interface WithPermissionProps {
  permission: string;
  children: ReactNode;
  fallback?: ReactNode;
}

export function WithPermission({ permission, children, fallback = null }: WithPermissionProps) {
  const { has } = useAuth();
  const hasPermission = has({ permission });

  if (!hasPermission) return fallback;
  return <>{children}</>;
}
```

使用方式：

```tsx
// app/(protected)/dashboard/page.tsx
import { WithPermission } from '@/components/withPermission';

export default function DashboardPage() {
  return (
    <div>
      {/* 所有用户都能看到的内容 */}
      <CommonContent />
      
      {/* 仅管理员可见的功能 */}
      <WithPermission permission="admin">
        <AdminFeature />
      </WithPermission>
    </div>
  );
}
```

### 4. 使用动态路由和重定向

在特定情况下，使用动态路由参数来控制访问，但共享相同的组件：

```tsx
// app/(protected)/[role]/dashboard/page.tsx
import { auth, redirectToSignIn } from '@clerk/nextjs';
import { notFound } from 'next/navigation';

export default async function RoleBasedDashboard({ params }) {
  const { role } = params;
  const { has } = auth();
  
  // 验证用户是否有权访问此角色页面
  if (role === 'admin' && !(await has({ permission: 'admin' }))) {
    return notFound(); // 或重定向到普通用户dashboard
  }
  
  return (
    <DashboardContent isAdmin={role === 'admin'} />
  );
}
```

### Next.js 14最佳实践总结

1.  **使用并行路由与权限槽**：

    ```tsx
    // app/(protected)/dashboard/@adminPanel/page.tsx
    // app/(protected)/dashboard/@userPanel/page.tsx
    // app/(protected)/dashboard/layout.tsx 中决定显示哪个槽
    ```
2.  **利用服务器组件的优势**：

    ```tsx
    // 在服务器组件中进行权限检查更安全
    const { userId, has } = auth();
    if (!userId) redirect('/sign-in');

    const canManageUsers = await has({ permission: 'users:manage' });
    ```
3.  **使用React Context共享权限状态**：

    ```tsx
    // providers/PermissionProvider.tsx
    // 在高层级组件中注入权限状态，避免重复检查
    ```
4.  **路由守卫模式**：

    ```tsx
    // 在用户进入路由前检查权限
    export async function generateMetadata() {
      const { has } = auth();
      const canAccess = await has({ permission: 'required:permission' });
      if (!canAccess) redirect('/unauthorized');
    }
    ```

**实际项目建议**：

* 在大多数情况下，基于角色的条件渲染是最简单有效的方法
* 对于复杂应用，考虑使用特性标志(Feature Flags)和权限矩阵
* 尽量将权限检查逻辑抽象到可重用的hooks或组件中
* 在错误处理方面投入精力，确保用户获得良好体验，了解为什么某些功能不可用

使用这些模式，您可以避免重复组件的问题，同时保持清晰的权限边界和良好的代码组织。
