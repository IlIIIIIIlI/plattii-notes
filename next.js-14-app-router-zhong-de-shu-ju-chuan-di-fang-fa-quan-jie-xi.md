# Next.js 14 App Router中的数据传递方法全解析

### 引言

Next.js 14的App Router模式带来了许多强大的功能，包括更灵活的路由系统、服务器组件和客户端组件的混合使用，以及更简洁的数据获取方式。在构建现代Web应用时，不同场景下如何在前后端间高效传递数据是一个核心问题。本文将全面介绍Next.js 14 App Router中可用的各种数据传递方法，并通过实际代码示例说明每种方法的实现和适用场景。

### 1. 查询参数 (Query Parameters)

查询参数是URL中`?`后面的键值对，适合传递可选且非敏感的数据。

#### 客户端实现

```tsx
// app/products/page.tsx (客户端组件)
'use client'

import { useSearchParams } from 'next/navigation'

export default function ProductsPage() {
  const searchParams = useSearchParams()
  
  // 获取查询参数
  const category = searchParams.get('category') || 'all'
  const page = parseInt(searchParams.get('page') || '1')
  
  return (
    <div>
      <h1>产品列表</h1>
      <p>类别: {category}</p>
      <p>页码: {page}</p>
      {/* 产品列表组件 */}
    </div>
  )
}
```

#### 服务器组件实现

```tsx
// app/products/page.tsx (服务器组件)
export default function ProductsPage({
  searchParams,
}: {
  searchParams: { [key: string]: string | string[] | undefined }
}) {
  // 获取查询参数
  const category = searchParams.category || 'all'
  const page = parseInt(searchParams.page as string || '1')
  
  return (
    <div>
      <h1>产品列表</h1>
      <p>类别: {category}</p>
      <p>页码: {page}</p>
      {/* 产品列表组件 */}
    </div>
  )
}
```

#### 实现带参数的链接

```tsx
// 使用Link组件构建带参数的链接
import Link from 'next/link'

function CategoryNavigation() {
  return (
    <nav>
      <Link href="/products?category=electronics">电子产品</Link>
      <Link href="/products?category=clothing&page=2">服装 (第2页)</Link>
      
      {/* 动态构建链接 */}
      <Link href={`/products?category=furniture&sort=price&order=asc`}>
        家具 (按价格升序)
      </Link>
    </nav>
  )
}
```

#### 从API路由获取查询参数

```tsx
// app/api/products/route.ts
import { NextRequest, NextResponse } from 'next/server'

export async function GET(request: NextRequest) {
  const { searchParams } = new URL(request.url)
  const category = searchParams.get('category') || 'all'
  const page = parseInt(searchParams.get('page') || '1')
  
  // 根据参数从数据库获取产品
  // const products = await db.getProducts({ category, page })
  
  return NextResponse.json({ 
    category, 
    page, 
    products: [] // 示例返回
  })
}
```

#### 使用场景与最佳实践

* **适用于**：筛选、排序、分页、非敏感数据传递
* **优点**：易于分享、可添加到书签、SEO友好（如果使用静态生成）
* **最佳实践**：
  * 保持查询参数简短明确
  * 使用默认值处理缺失参数
  * 对数值型参数进行类型转换和验证

### 2. 路径参数 (Path Parameters)

路径参数是URL路径的一部分，适合表示资源标识符。

#### 定义动态路由

```tsx
// app/users/[id]/page.tsx
export default function UserProfile({ params }: { params: { id: string } }) {
  return (
    <div>
      <h1>用户资料</h1>
      <p>用户ID: {params.id}</p>
      {/* 用户详细资料 */}
    </div>
  )
}
```

#### 多段动态路由

```tsx
// app/dashboard/[orgId]/[projectId]/page.tsx
export default function ProjectPage({ 
  params 
}: { 
  params: { orgId: string, projectId: string } 
}) {
  return (
    <div>
      <h1>项目详情</h1>
      <p>组织ID: {params.orgId}</p>
      <p>项目ID: {params.projectId}</p>
      {/* 项目详情内容 */}
    </div>
  )
}
```

#### 捕获所有路径段

```tsx
// app/docs/[...slug]/page.tsx
export default function DocPage({ params }: { params: { slug: string[] } }) {
  const path = params.slug.join('/')
  
  return (
    <div>
      <h1>文档页面</h1>
      <p>路径: {path}</p>
      {/* 文档内容 */}
    </div>
  )
}
```

#### API路由中的路径参数

```tsx
// app/api/users/[id]/route.ts
import { NextRequest, NextResponse } from 'next/server'

export async function GET(
  request: NextRequest,
  { params }: { params: { id: string } }
) {
  const userId = params.id
  
  // 根据ID获取用户
  // const user = await db.getUser(userId)
  
  return NextResponse.json({ 
    id: userId,
    name: 'John Doe' // 示例数据
  })
}
```

#### 使用场景与最佳实践

* **适用于**：资源标识、分层数据结构、RESTful API设计
* **优点**：语义清晰、URL简洁、SEO友好
* **最佳实践**：
  * 使用有意义的参数名称
  * 考虑参数验证与类型转换
  * 为非法参数提供友好的错误处理

### 3. HTTP请求头 (Headers)

HTTP头适合传递元数据、认证信息等，不适合包含在URL中的数据。

#### 客户端发送自定义头

```tsx
// app/components/DataFetcher.tsx
'use client'

import { useState, useEffect } from 'react'

export default function DataFetcher() {
  const [data, setData] = useState(null)
  
  useEffect(() => {
    async function fetchData() {
      const response = await fetch('/api/protected-data', {
        headers: {
          'X-Custom-Header': 'custom-value',
          'Authorization': 'Bearer your-token'
        }
      })
      
      const result = await response.json()
      setData(result)
    }
    
    fetchData()
  }, [])
  
  return (
    <div>
      <h2>获取的数据</h2>
      <pre>{JSON.stringify(data, null, 2)}</pre>
    </div>
  )
}
```

#### 服务器端读取头信息

```tsx
// app/api/protected-data/route.ts
import { NextRequest, NextResponse } from 'next/server'

export async function GET(request: NextRequest) {
  // 读取请求头
  const customHeader = request.headers.get('X-Custom-Header')
  const authHeader = request.headers.get('Authorization')
  
  // 验证Authorization头
  if (!authHeader || !authHeader.startsWith('Bearer ')) {
    return NextResponse.json({ error: '未授权访问' }, { status: 401 })
  }
  
  const token = authHeader.split(' ')[1]
  
  // 验证token (实际应用中应使用适当的认证库)
  // const user = await verifyToken(token)
  
  return NextResponse.json({
    message: '受保护的数据',
    receivedHeader: customHeader
  })
}
```

#### 从服务器组件读取头信息

```tsx
// app/headers-demo/page.tsx
import { headers } from 'next/headers'

export default function HeadersDemo() {
  const headersList = headers()
  const userAgent = headersList.get('user-agent')
  const referer = headersList.get('referer')
  
  return (
    <div>
      <h1>HTTP头信息</h1>
      <p>User Agent: {userAgent}</p>
      <p>Referer: {referer || 'None'}</p>
    </div>
  )
}
```

#### 使用场景与最佳实践

* **适用于**：认证、多语言支持、客户端信息传递
* **优点**：不污染URL、适合传递元数据
* **最佳实践**：
  * 自定义头部使用`X-`前缀
  * 敏感信息使用HTTPS传输
  * 考虑CORS策略对头信息的影响

### 4. 请求体 (Request Body)

请求体适合发送大量数据或复杂数据结构，通常用于POST、PUT等非GET请求。

#### 客户端发送表单数据

```tsx
// app/components/UserForm.tsx
'use client'

import { useState } from 'react'

export default function UserForm() {
  const [name, setName] = useState('')
  const [email, setEmail] = useState('')
  const [status, setStatus] = useState('')
  
  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault()
    
    try {
      const response = await fetch('/api/users', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({ name, email })
      })
      
      if (response.ok) {
        setStatus('用户创建成功！')
        setName('')
        setEmail('')
      } else {
        const error = await response.text()
        setStatus(`错误: ${error}`)
      }
    } catch (error) {
      setStatus(`提交失败: ${error instanceof Error ? error.message : String(error)}`)
    }
  }
  
  return (
    <form onSubmit={handleSubmit}>
      <div>
        <label htmlFor="name">姓名:</label>
        <input
          id="name"
          value={name}
          onChange={(e) => setName(e.target.value)}
          required
        />
      </div>
      <div>
        <label htmlFor="email">邮箱:</label>
        <input
          id="email"
          type="email"
          value={email}
          onChange={(e) => setEmail(e.target.value)}
          required
        />
      </div>
      <button type="submit">创建用户</button>
      {status && <p>{status}</p>}
    </form>
  )
}
```

#### 服务器端处理JSON数据

```tsx
// app/api/users/route.ts
import { NextRequest, NextResponse } from 'next/server'

export async function POST(request: NextRequest) {
  try {
    // 解析请求体
    const { name, email } = await request.json()
    
    // 验证数据
    if (!name || !email) {
      return NextResponse.json(
        { error: '姓名和邮箱为必填项' },
        { status: 400 }
      )
    }
    
    // 在实际应用中，这里会保存到数据库
    // const newUser = await db.users.create({ name, email })
    
    // 返回成功响应
    return NextResponse.json(
      { id: 'new-user-id', name, email },
      { status: 201 }
    )
  } catch (error) {
    console.error('创建用户时出错:', error)
    return NextResponse.json(
      { error: '服务器错误' },
      { status: 500 }
    )
  }
}
```

#### 处理FormData

```tsx
// app/api/upload/route.ts
import { NextRequest, NextResponse } from 'next/server'
import { writeFile } from 'fs/promises'
import { join } from 'path'

export async function POST(request: NextRequest) {
  try {
    const formData = await request.formData()
    const file = formData.get('file') as File
    const name = formData.get('name') as string
    
    if (!file) {
      return NextResponse.json(
        { error: '没有提供文件' },
        { status: 400 }
      )
    }
    
    // 转换文件为Buffer
    const bytes = await file.arrayBuffer()
    const buffer = Buffer.from(bytes)
    
    // 保存文件
    const path = join(process.cwd(), 'public', 'uploads', file.name)
    await writeFile(path, buffer)
    
    return NextResponse.json({ 
      message: '文件上传成功',
      fileName: file.name,
      name: name || '未命名'
    })
  } catch (error) {
    console.error('文件上传错误:', error)
    return NextResponse.json(
      { error: '文件上传失败' },
      { status: 500 }
    )
  }
}
```

#### 使用场景与最佳实践

* **适用于**：表单提交、文件上传、复杂数据结构
* **优点**：可传递大量数据、支持多种数据格式
* **最佳实践**：
  * 始终验证和净化输入数据
  * 设置适当的Content-Type头
  * 对大型请求体设置超时和大小限制

### 5. Cookie与客户端存储

Cookie适合在请求间保持状态，LocalStorage/SessionStorage适合客户端数据存储。

#### 设置和读取Cookie

```tsx
// app/api/login/route.ts
import { NextRequest, NextResponse } from 'next/server'
import { cookies } from 'next/headers'

export async function POST(request: NextRequest) {
  const { username, password } = await request.json()
  
  // 在实际应用中验证用户凭据
  if (username === 'admin' && password === 'password') {
    // 创建会话
    const response = NextResponse.json({ 
      success: true,
      message: '登录成功' 
    })
    
    // 设置Cookie (安全设置)
    response.cookies.set({
      name: 'session',
      value: 'session-token-value',
      httpOnly: true,
      secure: process.env.NODE_ENV === 'production',
      sameSite: 'strict',
      maxAge: 60 * 60 * 24 * 7, // 7天
      path: '/'
    })
    
    return response
  }
  
  return NextResponse.json(
    { success: false, message: '用户名或密码错误' },
    { status: 401 }
  )
}
```

#### 读取服务器端Cookie

```tsx
// app/profile/page.tsx
import { cookies } from 'next/headers'
import { redirect } from 'next/navigation'

export default function ProfilePage() {
  const cookieStore = cookies()
  const sessionCookie = cookieStore.get('session')
  
  // 检查用户是否已认证
  if (!sessionCookie) {
    redirect('/login')
  }
  
  // 在实际应用中会验证会话Token并获取用户数据
  // const user = await getUserFromSession(sessionCookie.value)
  
  return (
    <div>
      <h1>用户资料</h1>
      <p>已认证会话: {sessionCookie.value}</p>
      {/* 用户资料内容 */}
    </div>
  )
}
```

#### 客户端中使用LocalStorage

```tsx
// app/components/ThemeToggle.tsx
'use client'

import { useState, useEffect } from 'react'

export default function ThemeToggle() {
  const [theme, setTheme] = useState('light')
  
  // 从LocalStorage初始化主题
  useEffect(() => {
    const savedTheme = localStorage.getItem('theme') || 'light'
    setTheme(savedTheme)
    document.documentElement.setAttribute('data-theme', savedTheme)
  }, [])
  
  // 切换主题
  function toggleTheme() {
    const newTheme = theme === 'light' ? 'dark' : 'light'
    setTheme(newTheme)
    localStorage.setItem('theme', newTheme)
    document.documentElement.setAttribute('data-theme', newTheme)
  }
  
  return (
    <button onClick={toggleTheme}>
      当前主题: {theme} (点击切换)
    </button>
  )
}
```

#### 使用场景与最佳实践

* **适用于**：用户偏好、会话管理、持久化状态
* **优点**：无需每次请求传递相同数据
* **最佳实践**：
  * Cookie设置httpOnly、secure和SameSite属性
  * LocalStorage不存储敏感信息
  * 考虑存储大小限制

### 6. Hash片段 (URL Fragment)

Hash片段是URL中#号后的部分，不会发送到服务器，适合客户端路由和状态保存。

#### 客户端处理Hash片段

```tsx
// app/components/TabSystem.tsx
'use client'

import { useState, useEffect } from 'react'

export default function TabSystem() {
  const [activeTab, setActiveTab] = useState('overview')
  
  // 从URL hash初始化标签
  useEffect(() => {
    const hash = window.location.hash.substring(1)
    if (hash && ['overview', 'features', 'pricing'].includes(hash)) {
      setActiveTab(hash)
    }
    
    // 处理hash变化事件
    const handleHashChange = () => {
      const newHash = window.location.hash.substring(1)
      if (newHash && ['overview', 'features', 'pricing'].includes(newHash)) {
        setActiveTab(newHash)
      }
    }
    
    window.addEventListener('hashchange', handleHashChange)
    return () => window.removeEventListener('hashchange', handleHashChange)
  }, [])
  
  // 更新hash
  const changeTab = (tab: string) => {
    setActiveTab(tab)
    window.location.hash = tab
  }
  
  return (
    <div>
      <div className="tabs">
        <button 
          onClick={() => changeTab('overview')}
          className={activeTab === 'overview' ? 'active' : ''}
        >
          概览
        </button>
        <button 
          onClick={() => changeTab('features')}
          className={activeTab === 'features' ? 'active' : ''}
        >
          功能
        </button>
        <button 
          onClick={() => changeTab('pricing')}
          className={activeTab === 'pricing' ? 'active' : ''}
        >
          价格
        </button>
      </div>
      
      <div className="tab-content">
        {activeTab === 'overview' && <div>概览内容...</div>}
        {activeTab === 'features' && <div>功能列表...</div>}
        {activeTab === 'pricing' && <div>价格详情...</div>}
      </div>
    </div>
  )
}
```

#### 带Hash的链接

```tsx
// app/components/ProductLinks.tsx
import Link from 'next/link'

export default function ProductLinks() {
  return (
    <nav>
      <Link href="/product#overview">产品概览</Link>
      <Link href="/product#features">产品功能</Link>
      <Link href="/product#pricing">价格方案</Link>
    </nav>
  )
}
```

#### 使用场景与最佳实践

* **适用于**：页内导航、客户端状态保存、SPA路由
* **优点**：不会触发页面刷新、不发送到服务器
* **最佳实践**：
  * 用于UI状态而非关键数据传递
  * 考虑初始页面加载时的默认选项
  * 结合useEffect处理hash变化

### 7. 会话状态 (Session State)

Next.js没有内置会话管理，但可以结合第三方库或自定义实现。

#### 使用Iron Session实现

首先安装依赖：

```bash
npm install iron-session
```

创建会话配置：

```tsx
// lib/session.ts
import { SessionOptions } from 'iron-session'

export interface SessionData {
  userId?: string
  isLoggedIn: boolean
}

export const sessionOptions: SessionOptions = {
  password: process.env.SESSION_PASSWORD as string,
  cookieName: 'next-app-session',
  cookieOptions: {
    secure: process.env.NODE_ENV === 'production'
  }
}

// 声明模块扩展以添加类型支持
declare module 'iron-session' {
  interface IronSessionData {
    user?: {
      id: string
      isLoggedIn: boolean
    }
  }
}
```

创建登录API：

```tsx
// app/api/session/login/route.ts
import { NextRequest, NextResponse } from 'next/server'
import { getIronSession } from 'iron-session'
import { sessionOptions } from '@/lib/session'

export async function POST(request: NextRequest) {
  const { username, password } = await request.json()
  
  // 验证用户凭据 (实际应用中应查询数据库)
  const isValid = username === 'user' && password === 'password'
  
  if (!isValid) {
    return NextResponse.json(
      { success: false, message: '无效的凭据' },
      { status: 401 }
    )
  }
  
  // 创建新会话
  const session = await getIronSession(request, NextResponse, sessionOptions)
  
  // 设置会话数据
  session.user = {
    id: 'user-123',
    isLoggedIn: true
  }
  
  // 保存会话
  await session.save()
  
  return NextResponse.json({ success: true, message: '登录成功' })
}
```

获取用户会话：

```tsx
// app/profile/page.tsx
import { cookies } from 'next/headers'
import { redirect } from 'next/navigation'
import { getIronSession } from 'iron-session'
import { sessionOptions } from '@/lib/session'

export default async function ProfilePage() {
  const session = await getIronSession(cookies(), {}, sessionOptions)
  
  // 检查用户是否已认证
  if (!session.user?.isLoggedIn) {
    redirect('/login')
  }
  
  return (
    <div>
      <h1>用户资料</h1>
      <p>用户ID: {session.user.id}</p>
      {/* 用户资料内容 */}
    </div>
  )
}
```

#### 使用场景与最佳实践

* **适用于**：用户认证、权限管理、跨请求状态维护
* **优点**：安全性高、数据不暴露给客户端
* **最佳实践**：
  * 使用强密码和加密
  * 设置适当的会话过期时间
  * 考虑分布式系统中的会话共享

### 8. Matrix参数 (Matrix URL Parameters)

Matrix参数在Next.js中没有内置支持，但可以通过路径参数+自定义解析实现。

#### 实现Matrix参数解析

```tsx
// app/blog/[...slug]/page.tsx
export default function BlogPage({ params }: { params: { slug: string[] } }) {
  // 解析Matrix参数
  const path = params.slug.join('/')
  const pathSegments = path.split('/')
  
  const matrixParams: Record<string, Record<string, string>> = {}
  
  pathSegments.forEach((segment, index) => {
    if (segment.includes(';')) {
      const [basePath, ...paramParts] = segment.split(';')
      
      matrixParams[index] = { basePath }
      
      paramParts.forEach(part => {
        if (part.includes('=')) {
          const [key, value] = part.split('=')
          matrixParams[index][key] = value
        } else {
          matrixParams[index][part] = 'true'
        }
      })
    }
  })
  
  return (
    <div>
      <h1>博客文章</h1>
      <p>路径: {path}</p>
      <h2>解析的Matrix参数:</h2>
      <pre>{JSON.stringify(matrixParams, null, 2)}</pre>
    </div>
  )
}
```

#### 使用场景与最佳实践

* **适用于**：分层参数、特殊路由需求
* **优点**：比查询参数更语义化
* **最佳实践**：
  * 由于Next.js没有内置支持，应谨慎使用
  * 考虑用路径参数或查询参数的替代方案

### 总结与选择指南

在Next.js 14的App Router中，我们有多种数据传递方式可供选择，每种方式都有其特定的用例和优势：

| 传递方式     | 适用场景           | Next.js特性支持        |
| -------- | -------------- | ------------------ |
| 查询参数     | 筛选、排序、分页、可选参数  | 完全支持，易于实现          |
| 路径参数     | 资源标识、层次结构、必需参数 | 完全支持，核心路由功能        |
| HTTP头    | 认证、元数据传递       | 支持，需区分客户端/服务器端实现   |
| 请求体      | 表单提交、复杂数据、文件上传 | 支持所有主要数据格式         |
| Cookie   | 用户偏好、会话标识符     | 内置API支持读写          |
| 客户端存储    | UI状态、用户偏好      | 仅客户端使用，需use client |
| Hash片段   | 页内导航、客户端路由     | 仅客户端可用，需处理未匹配      |
| 会话状态     | 用户认证、安全数据存储    | 需第三方库如iron-session |
| Matrix参数 | 特殊路由结构         | 无内置支持，需自定义解析       |

#### 选择建议

1. **对于公开内容**：
   * 使用路径参数标识主要资源
   * 使用查询参数进行筛选和排序
2. **对于用户操作**：
   * 表单提交和数据创建使用请求体
   * 简单操作和筛选使用查询参数
3. **对于安全需求**：
   * 认证信息使用HTTP头或加密Cookie
   * 敏感数据使用服务器会话存储
4. **对于UI状态**：
   * 使用Hash片段或客户端存储
   * 需要分享的状态使用查询参数

通过合理组合这些数据传递方法，可以构建既安全又用户友好的Next.js应用，充分利用App Router提供的强大功能。在选择方法时，始终考虑安全性、用户体验和性能的平衡。
