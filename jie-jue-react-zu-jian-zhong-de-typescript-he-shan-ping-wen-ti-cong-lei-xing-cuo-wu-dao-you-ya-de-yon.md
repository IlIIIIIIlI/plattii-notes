# 解决React组件中的TypeScript和闪屏问题：从类型错误到优雅的用户体验

## 解决React组件中的TypeScript和闪屏问题：从类型错误到优雅的用户体验

在开发现代React应用时，TypeScript和良好的用户体验都是不可或缺的。然而，当使用自定义UI组件库时，我们常常会遇到类型兼容性问题和渲染闪烁问题。本文记录了我们如何一步步解决一个项目中的这两类问题，希望能为遇到类似问题的开发者提供参考。

### 问题背景

我们在使用自定义UI组件库（`@plattii/ui`）构建登录页面时，遇到了两个主要问题：

1. **TypeScript类型错误**：组件属性定义不匹配导致的编译错误
2. **页面闪屏**：组件在刷新时从左上角瞬间闪动到正确位置

这些问题都源于我们的组件库架构和React的渲染机制。组件库定义了严格的类型和自定义属性，而不是使用标准的HTML属性。

### 第一阶段：解决属性类型错误

最初的代码中，我们在`OakBox`组件上使用了flex相关的属性：

```tsx
<OakBox $minHeight="100vh" $display="flex" $alignItems="center" $justifyContent="center">
  {/* 内容 */}
</OakBox>
```

这导致了TypeScript错误：

```
Property '$alignItems' does not exist on type 'IntrinsicAttributes & { children?: ReactNode; ... }'
```

**问题原因**：`OakBox`组件不支持flex相关的属性，这些属性只存在于`OakFlex`组件中。

**解决方案**：将`OakBox`替换为`OakFlex`：

```tsx
<OakFlex $minHeight="100vh" $alignItems="center" $justifyContent="center">
  {/* 内容 */}
</OakFlex>
```

### 第二阶段：解决嵌套组件的类型错误

解决了顶层组件的问题后，我们发现内部组件也存在类似的类型错误：

```
Property '$flex' does not exist on type 'IntrinsicAttributes & { children?: ReactNode; ... }'
```

我们将所有使用flex属性的`OakBox`组件都替换为`OakFlex`组件：

```tsx
<OakFlex $flexGrow="1" $flexBasis="0">
  {/* 表单内容 */}
</OakFlex>

<OakFlex $flexGrow="1" $flexBasis="0" $justifyContent="center" $alignItems="center">
  {/* 图片内容 */}
</OakFlex>
```

### 第三阶段：尝试解决闪屏问题

解决了类型错误后，我们注意到页面刷新时有闪烁现象。我们尝试使用CSS过渡和opacity属性来解决这个问题：

```tsx
<OakFlex 
    $minHeight="100vh" 
    $alignItems="center" 
    $justifyContent="center"
    $opacity={isMounted ? "1" : "0"}
    $transition="opacity 0.3s"
>
```

然而，这又导致了新的类型错误：

```
Type '"0" | "1"' is not assignable to type 'ResponsiveValues<"transparent" | "semi-transparent" | "semi-opaque" | "opaque"> | undefined'
```

这是因为我们的UI库使用了枚举值而不是原生CSS值。

### 第四阶段：尝试使用内联样式

接下来，我们尝试使用React的内联样式来解决问题：

```tsx
<OakFlex 
    $minHeight="100vh" 
    $alignItems="center" 
    $justifyContent="center"
    style={{ 
        opacity: isMounted ? 1 : 0,
        transition: 'opacity 0.3s'
    }}
>
```

但是我们的组件库不支持标准的`style`属性：

```
Property 'style' does not exist on type 'IntrinsicAttributes & DisplayStyleProps & ...'
```

### 最终解决方案：条件渲染

经过多次尝试，我们找到了最优雅的解决方案 - 条件渲染：

{% code lineNumbers="true" %}
```tsx
const SignUpSignInLayout = ({ children, loaded }: Readonly<SignUpSignInLayoutProps>) => {
    const [isMounted, setIsMounted] = useState(false)
    
    useEffect(() => {
        setIsMounted(true)
    }, [])
    
    if (!isMounted) {
        // 返回占位符，保持高度一致以防止布局移动
        return <div style={{ height: '100vh' }}></div>
    }
    
    return (
        <HeroContainer>
            <OakFlex 
                $minHeight="100vh" 
                $alignItems="center" 
                $justifyContent="center"
            >
                {/* 组件内容 */}
            </OakFlex>
        </HeroContainer>
    )
}
```
{% endcode %}

这个方案的原理是：

1. 组件初始渲染一个简单的占位符div，高度与最终高度相同
2. 当组件完成挂载后（所有样式加载完成），切换到显示实际内容
3. 由于完整内容是在样式加载后才显示的，因此不会出现内容从默认位置闪动到样式指定位置的现象

### 技术要点与经验总结

通过这个案例，我们学到了几个重要的技术经验：

#### 1. 了解自定义组件库的类型定义

当使用自定义UI库时，必须了解每个组件支持的属性和类型。在我们的案例中，`OakBox`和`OakFlex`有不同的属性支持，混用会导致类型错误。

#### 2. 遵循组件设计意图

每个组件都有其设计目的。`OakBox`用于基本布局，`OakFlex`专门用于Flexbox布局。根据需要选择正确的组件可以避免许多问题。

#### 3. 解决闪屏问题的策略

处理客户端渲染的闪屏问题有多种方法：

* **CSS过渡**：通过opacity和transition控制内容的显示（但可能受组件库限制）
* **条件渲染**：只在组件完全挂载后才渲染内容
* **占位符技术**：先渲染占位符，再替换为实际内容

#### 4. 优雅降级的思路

当遇到复杂问题时，逐步简化解决方案往往比强行应用复杂方案更有效。我们最终的解决方案非常简单，但完全解决了问题。

### 完整代码示例

最终的解决方案代码：

{% code lineNumbers="true" %}
```tsx
"use client"

import { Transition } from "@headlessui/react"
import { OakBox, OakFlex, OakMaxWidth } from "@plattii/ui"
import Image from "next/image"
import { useEffect, useState } from "react"

import jigsaw from "@/assets/svg/illustration/jigsaw.svg"
import HeroContainer from "./HeroContainer"

type SignUpSignInLayoutProps = {
    children: React.ReactNode
    loaded: boolean
}

const SignUpSignInLayout = ({ children, loaded }: Readonly<SignUpSignInLayoutProps>) => {
    const [isMounted, setIsMounted] = useState(false)
    
    useEffect(() => {
        setIsMounted(true)
    }, [])
    
    if (!isMounted) {
        return <div style={{ height: '100vh' }}></div>
    }
    
    return (
        <HeroContainer>
            <OakFlex 
                $minHeight="100vh" 
                $alignItems="center" 
                $justifyContent="center"
            >
                <OakMaxWidth>
                    <OakFlex 
                        $flexDirection={["column", "row"]} 
                        $alignItems="center" 
                        $justifyContent={["center", "space-between"]}
                        $gap="all-spacing-4"
                    >
                        <OakFlex 
                            $flexGrow="1"
                            $width={["100%", "auto"]}
                        >
                            <Transition
                                show={Boolean(loaded)}
                                appear={true}
                                enter="transition-opacity duration-500"
                                enterFrom="opacity-0"
                                enterTo="opacity-100"
                                leave="transition-opacity duration-500"
                                leaveFrom="opacity-100"
                                leaveTo="opacity-0"
                            >
                                {children}
                            </Transition>
                        </OakFlex>

                        <OakFlex 
                            $flexGrow="1" 
                            $justifyContent="center" 
                            $alignItems="center"
                            $display={["none", "flex"]}
                        >
                            <Image 
                                src={jigsaw} 
                                alt="Magic Carpet" 
                                width={400} 
                                height={400} 
                                priority 
                            />
                        </OakFlex>
                    </OakFlex>
                </OakMaxWidth>
            </OakFlex>
        </HeroContainer>
    )
}

export default SignUpSignInLayout
```
{% endcode %}

### 结论

在现代前端开发中，类型安全和良好的用户体验同样重要。通过深入理解组件库的类型设计和React的渲染机制，我们可以解决复杂的类型错误和用户体验问题。有时候，最简单的解决方案可能是最好的 - 条件渲染这种基本的React模式解决了我们的复杂问题，既保持了类型安全，又提供了良好的用户体验。

希望这篇文章能帮助您在遇到类似问题时找到解决思路。记住，前端开发中的很多挑战都可以通过回归基础原则来解决。
