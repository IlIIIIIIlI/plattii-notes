# 基于角色的动态 Onboarding 最佳实践 - Next.js RBAC 实现指南

在构建现代 Web 应用程序时，提供个性化的用户引导流程（Onboarding）可以显著提升用户体验。特别是在具有多种用户角色的系统中，不同角色通常需要不同的引导内容。本文将深入探讨如何在 Next.js 14 项目中实现基于角色的访问控制（RBAC）动态 Onboarding 流程。

### 问题背景

在企业级应用中，不同角色（如开发者、设计师、管理者）往往需要完成不同的引导流程，以收集特定信息并设置相应权限。传统的静态 Onboarding 流程难以满足这一需求，而硬编码多个流程又会导致代码维护困难。

### 解决方案架构

我们将通过以下架构组件来构建一个灵活且类型安全的 RBAC Onboarding 系统：

1. **角色与问卷配置映射**
2. **动态问卷加载机制**
3. **状态管理与持久化**
4. **条件渲染组件**
5. **路由保护与重定向**
6. **角色特定的后处理逻辑**

### 详细实现

#### 1. 角色与问卷配置关联

首先，我们需要建立角色和问卷配置之间的映射关系：

```typescript
// config/questionnaires.ts
import { QuestionnaireConfig } from "@/types/onboardingQuestion"
import { developerQuestionnaire } from "./developer-questionnaire"
import { designerQuestionnaire } from "./designer-questionnaire"
import { managerQuestionnaire } from "./manager-questionnaire"

// 角色到问卷的映射
export const roleQuestionnaireMap: Record<string, QuestionnaireConfig> = {
  "developer": developerQuestionnaire,
  "designer": designerQuestionnaire,
  "manager": managerQuestionnaire,
  // 可以添加更多角色
}

// 默认问卷，当角色未匹配时使用
export const defaultQuestionnaire = developerQuestionnaire
```

这种方式使得我们可以灵活地为每个角色定义专属的问卷内容，同时保持配置的模块化。

#### 2. 动态问卷加载

修改主页面组件，根据用户角色动态加载适当的问卷配置：

```tsx
"use client"

import { useEffect } from "react"
import { AnimatePresence } from "framer-motion"
import { useQuestionnaireStore } from "@/store/question-store"
import { useAuth } from "@/hooks/useAuth" 
import { roleQuestionnaireMap, defaultQuestionnaire } from "@/config/questionnaires"
import StepContainer from "@/components/Onboarding/StepContainer"
import NavigationControls from "@/components/Onboarding/NavigationControls"
import ProgressIndicator from "@/components/Onboarding/ProgressIndicator"
import SuccessMessage from "@/components/Onboarding/SuccessMessage"
import { Alert, AlertDescription, AlertTitle } from "@/components/ui/alert"
import { AlertCircle } from "lucide-react"

export default function Home() {
    const { currentStep, responses, isComplete, submitError, submitSuccess, initQuestionnaire } = useQuestionnaireStore()
    const { user } = useAuth()
    
    // 根据用户角色加载相应的问卷
    useEffect(() => {
        const userRole = user?.role || "default"
        const questionnaire = roleQuestionnaireMap[userRole] || defaultQuestionnaire
        initQuestionnaire(questionnaire)
    }, [user, initQuestionnaire])
    
    const activeQuestionnaire = useQuestionnaireStore(state => state.questionnaire)
    
    // 以下省略渲染逻辑...
}
```

#### 3. 扩展状态管理

使用 Zustand 实现动态问卷的状态管理：

```typescript
// store/question-store.ts
import { create } from 'zustand'
import { QuestionnaireConfig } from '@/types/onboardingQuestion'

interface QuestionnaireState {
  questionnaire: QuestionnaireConfig | null
  currentStep: number
  responses: Record<string, any>
  isComplete: boolean
  submitError: string | null
  submitSuccess: boolean
  
  // 新增初始化方法
  initQuestionnaire: (config: QuestionnaireConfig) => void
  nextStep: () => void
  prevStep: () => void
  setResponse: (questionId: string, value: any) => void
  submitQuestionnaire: () => Promise<void>
}

export const useQuestionnaireStore = create<QuestionnaireState>((set, get) => ({
  questionnaire: null,
  currentStep: 0,
  responses: {},
  isComplete: false,
  submitError: null,
  submitSuccess: false,
  
  initQuestionnaire: (config) => {
    set({ 
      questionnaire: config,
      currentStep: 0,
      responses: {},
      isComplete: false,
      submitError: null,
      submitSuccess: false
    })
  },
  
  // 其他方法实现...
}))
```

#### 4. 增强问卷类型定义

为了支持角色特定的问题和步骤，我们扩展了问卷的类型定义：

```typescript
// types/onboardingQuestion.ts
export type QuestionType = 'text' | 'single' | 'multi' | 'trueFalse' | 'matching' | 'roleSpecific'

export interface BaseQuestion {
  id: string
  type: QuestionType
  prompt: string
  optional?: boolean
  validation?: QuestionValidation
  roleVisibility?: string[] // 指定哪些角色可以看到此问题
}

// ... 其他问题类型定义 ...

export interface RoleSpecificQuestion extends BaseQuestion {
  type: 'roleSpecific'
  roleQuestions: Record<string, BaseQuestion> // 每个角色的特定问题
}

export interface QuestionnaireStep {
  id: string
  title: string
  description: string
  label: string
  confirmationMessage?: string
  questions: Question[]
  roleVisibility?: string[] // 指定哪些角色可以看到此步骤
}

export interface QuestionnaireConfig {
  id: string
  title: string
  description: string
  submitButtonText: string
  defaultRequiredMessage: string
  defaultErrorMessage: string
  steps: QuestionnaireStep[]
  roleSpecific?: boolean // 标记此问卷是否为角色特定的
}
```

#### 5. 条件渲染组件

增强 `StepContainer` 组件以支持基于角色的条件渲染：

```tsx
// components/Onboarding/StepContainer.tsx
"use client"

import { motion } from "framer-motion"
import { QuestionnaireStep, Question } from "@/types/onboardingQuestion"
import QuestionRenderer from "./QuestionRenderer"
import { useAuth } from "@/hooks/useAuth"

interface StepContainerProps {
  step: QuestionnaireStep
  responses: Record<string, any>
  stepIndex: number
}

const StepContainer = ({ step, responses, stepIndex }: StepContainerProps) => {
  const { user } = useAuth()
  const userRole = user?.role || "default"
  
  // 检查此步骤是否应对当前角色可见
  if (step.roleVisibility && !step.roleVisibility.includes(userRole)) {
    return null
  }
  
  // 过滤出对当前角色可见的问题
  const visibleQuestions = step.questions.filter(question => {
    return !question.roleVisibility || question.roleVisibility.includes(userRole)
  })
  
  return (
    <motion.div
      key={step.id}
      initial={{ opacity: 0, y: 20 }}
      animate={{ opacity: 1, y: 0 }}
      exit={{ opacity: 0, y: -20 }}
      transition={{ duration: 0.3 }}
      className="bg-card rounded-lg p-6 shadow-sm"
    >
      {/* 渲染逻辑... */}
    </motion.div>
  )
}
```

#### 6. 后端 API 处理

创建一个能够处理不同角色提交的 API 路由：

```typescript
// app/api/submit-questionnaire/route.ts
import { NextRequest, NextResponse } from 'next/server'
import { getServerSession } from 'next-auth/next'
import { authOptions } from '@/lib/auth'
import { roleQuestionnaireMap } from '@/config/questionnaires'

export async function POST(request: NextRequest) {
  try {
    // 获取用户会话和角色
    const session = await getServerSession(authOptions)
    if (!session || !session.user) {
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
    }
    
    const userRole = session.user.role || 'default'
    
    // 获取请求体
    const { questionnaireId, responses } = await request.json()
    
    // 验证问卷 ID 与用户角色匹配
    const expectedQuestionnaire = roleQuestionnaireMap[userRole]
    if (!expectedQuestionnaire || expectedQuestionnaire.id !== questionnaireId) {
      return NextResponse.json({ error: 'Invalid questionnaire for user role' }, { status: 400 })
    }
    
    // 执行角色特定的处理逻辑
    switch (userRole) {
      case 'developer':
        await setupDeveloperOnboarding(session.user.id, responses)
        break
      case 'designer':
        await setupDesignerOnboarding(session.user.id, responses)
        break
      // 其他角色处理...
    }
    
    // 更新用户的 onboarding 状态
    await updateUserOnboardingStatus(session.user.id, userRole)
    
    return NextResponse.json({ success: true })
  } catch (error: any) {
    console.error('Onboarding submission error:', error)
    return NextResponse.json({ error: error.message || 'Failed to process onboarding' }, { status: 500 })
  }
}
```

#### 7. 路由保护与流程跟踪

使用中间件确保用户完成适当的 onboarding 步骤：

```typescript
// middleware.ts
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'
import { getToken } from 'next-auth/jwt'

export async function middleware(request: NextRequest) {
  // 获取 token 和用户信息
  const token = await getToken({ req: request })
  
  // 如果用户未登录，重定向到登录页面
  if (!token) {
    return NextResponse.redirect(new URL('/auth/login', request.url))
  }
  
  // 检查用户是否已完成 onboarding
  const hasCompletedOnboarding = token.user?.hasCompletedOnboarding
  
  // 当前请求的路径
  const path = request.nextUrl.pathname
  
  // 如果访问非 onboarding 页面但未完成 onboarding，重定向到 onboarding
  if (!hasCompletedOnboarding && !path.startsWith('/onboarding')) {
    return NextResponse.redirect(new URL('/onboarding', request.url))
  }
  
  // 如果已完成 onboarding 但尝试访问 onboarding 页面，重定向到仪表板
  if (hasCompletedOnboarding && path.startsWith('/onboarding')) {
    return NextResponse.redirect(new URL('/dashboard', request.url))
  }
  
  return NextResponse.next()
}
```

#### 8. 数据库模型

为了跟踪用户的 onboarding 进度，我们需要在数据库中建立相应的模型：

```prisma
// prisma/schema.prisma
model User {
  id              String    @id @default(cuid())
  name            String?
  email           String?   @unique
  emailVerified   DateTime?
  image           String?
  role            String    @default("user")
  onboardingStatus OnboardingStatus?
}

model OnboardingStatus {
  id              String    @id @default(cuid())
  userId          String    @unique
  user            User      @relation(fields: [userId], references: [id], onDelete: Cascade)
  isCompleted     Boolean   @default(false)
  completedAt     DateTime?
  currentStep     Int       @default(0)
  questionnaireId String
  responses       Json?     // 存储用户的问卷回答
  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt
}
```

### 最佳实践总结

在实现基于角色的动态 Onboarding 流程时，以下是关键的最佳实践：

1. **配置分离**：将问卷配置与业务逻辑分离，使每个角色的问卷配置独立维护。
2. **动态加载**：根据用户角色动态加载相应的问卷配置，避免硬编码多个流程。
3. **类型安全**：使用 TypeScript 确保问卷配置的类型安全，减少运行时错误。
4. **条件渲染**：在问卷和问题级别支持基于角色的条件渲染，实现精细控制。
5. **状态跟踪**：在数据库中跟踪每个用户的 onboarding 状态，支持断点续传。
6. **角色特定处理**：根据角色执行不同的 onboarding 后处理逻辑，如设置特定权限。
7. **路由保护**：使用中间件确保用户按照角色要求完成 onboarding 流程。
8. **可扩展设计**：设计允许轻松添加新角色和新的问卷类型，适应未来需求变化。

### 结论

基于角色的动态 Onboarding 流程可以显著提升用户体验，同时简化代码维护。通过将问卷配置与业务逻辑分离，并利用 TypeScript 的类型系统，我们可以构建一个灵活、安全且可扩展的 onboarding 系统。

Next.js 14 结合现代状态管理工具如 Zustand，为实现这样的系统提供了强大的基础。通过本文介绍的架构和最佳实践，你可以为不同角色的用户提供个性化的引导体验，从而提升用户满意度和系统采用率。
