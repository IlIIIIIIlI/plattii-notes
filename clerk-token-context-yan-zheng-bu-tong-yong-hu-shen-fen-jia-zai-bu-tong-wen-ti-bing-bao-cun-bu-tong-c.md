# clerk token context 验证不同用户身份加载不同问题并保存不同context

You've identified critical issues with my approach. Let me redesign the solution to address these concerns:

### Improved Server-Side Approach

#### 1. Create a Server API Endpoint

First, let's create a server endpoint to securely determine the user's role:

```typescript
// app/api/user-questionnaire/route.ts
import { auth } from "@clerk/nextjs/server";
import { NextResponse } from "next/server";
import { roleQuestionnaireMap, defaultQuestionnaire } from "@/config/questionnaire";

export async function GET() {
  try {
    const { userId, sessionClaims } = await auth();
    
    if (!userId) {
      return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
    }
    
    // Extract role from the session claims
    const orgRole = sessionClaims?.org_role as string || null;
    
    // Select the appropriate questionnaire type
    const questionnaireType = orgRole && roleQuestionnaireMap[orgRole.toLowerCase()] 
      ? orgRole.toLowerCase() 
      : 'developer'; // Default
    
    return NextResponse.json({ 
      questionnaireType,
      userId
    });
  } catch (error) {
    console.error("Error fetching user questionnaire:", error);
    return NextResponse.json(
      { error: "Failed to determine questionnaire type" }, 
      { status: 500 }
    );
  }
}
```

#### 2. Update the Store to Handle Questionnaire Type

Let's modify the question store to include the questionnaire type and handle persistence:

```typescript
// store/question-store.ts (modified)
export interface QuestionnaireState {
  // ... existing fields
  
  // Add questionnaire type tracking
  questionnaireType: string;
  
  // Add method to set questionnaire type
  setQuestionnaireType: (type: string) => void;
}

export const useQuestionnaireStore = create<QuestionnaireState>()(
  persist(
    (set, get) => ({
      // ... existing state
      
      questionnaireType: 'developer', // Default
      
      setQuestionnaireType: (type: string) => {
        set({ 
          questionnaireType: type,
          // If changing questionnaire type mid-process, we might need to reset responses
          // or apply a migration function if the questionnaires have different structures
        });
      },
      
      // Modify resetQuestionnaire to preserve the questionnaire type
      resetQuestionnaire: () => {
        const currentType = get().questionnaireType;
        
        set({
          currentStep: 0,
          responses: {},
          isComplete: false,
          submitError: null,
          submitSuccess: false,
          confirmationOpen: false,
          interactedQuestions: [],
          // Keep the questionnaire type
          questionnaireType: currentType,
        });
      },
      
      // ... rest of the store
    }),
    {
      name: "questionnaire-storage",
      partialize: state => ({
        // Add questionnaireType to persisted state
        responses: state.responses,
        currentStep: state.currentStep,
        isComplete: state.isComplete,
        debounceTime: state.debounceTime,
        questionnaireType: state.questionnaireType,
      }),
    }
  )
);
```

#### 3. Create a Module to Safely Load the Questionnaire

```typescript
// lib/questionnaire-loader.ts
import { roleQuestionnaireMap, defaultQuestionnaire } from "@/config/questionnaire";
import type { QuestionnaireConfig } from "@/types/onboardingQuestion";

// Get questionnaire config by type safely
export function getQuestionnaireByType(type: string): QuestionnaireConfig {
  if (!type) return defaultQuestionnaire;
  
  const questionnaire = roleQuestionnaireMap[type.toLowerCase()];
  return questionnaire || defaultQuestionnaire;
}
```

#### 4. Update the Onboarding Page

```typescript
// app/onboarding/page.tsx
"use client"

import { useEffect, useState } from "react"
import { useQuestionnaireStore } from "@/store/question-store"
import { AnimatePresence } from "framer-motion"
import StepContainer from "@/components/Onboarding/StepContainer"
import NavigationControls from "@/components/Onboarding/NavigationControls"
import ProgressIndicator from "@/components/Onboarding/ProgressIndicator"
import SuccessMessage from "@/components/Onboarding/SuccessMessage"
import { Alert, AlertDescription, AlertTitle } from "@/components/ui/alert"
import { AlertCircle } from "lucide-react"
import { getQuestionnaireByType } from "@/lib/questionnaire-loader"

export default function OnboardingPage() {
    const { 
      currentStep, 
      responses, 
      isComplete, 
      submitError, 
      submitSuccess,
      questionnaireType,
      setQuestionnaireType
    } = useQuestionnaireStore()
    
    const [isLoading, setIsLoading] = useState(true)
    const [error, setError] = useState<string | null>(null)
    const [activeQuestionnaireConfig, setActiveQuestionnaireConfig] = useState(getQuestionnaireByType(questionnaireType))
    
    // Fetch the user's questionnaire type from the server
    useEffect(() => {
        async function loadQuestionnaireType() {
            try {
                setIsLoading(true)
                
                const response = await fetch('/api/user-questionnaire')
                
                if (!response.ok) {
                    throw new Error('Failed to fetch questionnaire type')
                }
                
                const data = await response.json()
                
                // Only update if different to avoid unnecessary rerenders
                if (data.questionnaireType !== questionnaireType) {
                    setQuestionnaireType(data.questionnaireType)
                }
                
                // Get the actual questionnaire config
                const config = getQuestionnaireByType(data.questionnaireType)
                setActiveQuestionnaireConfig(config)
                
                setIsLoading(false)
            } catch (error) {
                console.error("Error loading questionnaire:", error)
                setError("Failed to load your onboarding process. Please try again.")
                setIsLoading(false)
            }
        }

        loadQuestionnaireType()
    }, [questionnaireType, setQuestionnaireType])
    
    // Ensure we re-get the questionnaire config when type changes
    useEffect(() => {
        setActiveQuestionnaireConfig(getQuestionnaireByType(questionnaireType))
    }, [questionnaireType])
    
    // Loading state
    if (isLoading) {
        return (
            <main className="min-h-screen bg-background p-[2rem] flex items-center justify-center">
                <div className="animate-spin rounded-full h-8 w-8 border-b-2 border-primary"></div>
            </main>
        )
    }
    
    // Error state
    if (error) {
        return (
            <main className="min-h-screen bg-background p-[2rem] flex items-center justify-center">
                <Alert variant="destructive" className="max-w-md">
                    <AlertCircle className="h-4 w-4" />
                    <AlertTitle>Error</AlertTitle>
                    <AlertDescription>{error}</AlertDescription>
                </Alert>
            </main>
        )
    }
    
    // Success state
    if (isComplete && submitSuccess) {
        return (
            <main className="min-h-screen bg-background p-[2rem] flex items-center justify-center">
                <SuccessMessage />
            </main>
        )
    }
    
    const currentStepData = activeQuestionnaireConfig.steps[currentStep]

    return (
        <main className="min-h-screen bg-background p-[2rem]">
            <div className="max-w-2xl mx-auto space-y-[2rem]">
                <div className="space-y-[0.5rem]">
                    <h1 className="text-3xl font-bold">{activeQuestionnaireConfig.title}</h1>
                    <p className="text-muted-foreground">{activeQuestionnaireConfig.description}</p>
                </div>

                <ProgressIndicator activeQuestionnaire={activeQuestionnaireConfig} />

                {submitError && (
                    <Alert variant="destructive">
                        <AlertCircle className="h-[1rem] w-[1rem]" />
                        <AlertTitle>Submission Error</AlertTitle>
                        <AlertDescription>{submitError}</AlertDescription>
                    </Alert>
                )}

                <AnimatePresence mode="wait">
                    {currentStepData && (
                        <StepContainer 
                            key={currentStepData.id} 
                            step={currentStepData} 
                            responses={responses} 
                            stepIndex={currentStep} 
                        />
                    )}
                </AnimatePresence>

                <NavigationControls activeQuestionnaire={activeQuestionnaireConfig} />
            </div>
        </main>
    )
}
```

#### 5. Update the Components to Accept the Questionnaire Config

We need to modify the components that previously accessed the global `activeQuestionnaire` to accept it as a prop:

```typescript
// components/Onboarding/ProgressIndicator.tsx
export default function ProgressIndicator({ activeQuestionnaire }) {
  // Use the passed questionnaire config instead of importing it
  const { currentStep, goToStep, validateCurrentStep } = useQuestionnaireStore()
  const steps = activeQuestionnaire.steps
  // ...rest of component
}

// components/Onboarding/NavigationControls.tsx
export default function NavigationControls({ activeQuestionnaire }) {
  const {
    currentStep,
    nextStep,
    previousStep,
    // ...other state
  } = useQuestionnaireStore()
  
  const totalSteps = activeQuestionnaire.steps.length
  const isLastStep = currentStep === totalSteps - 1
  const currentStepData = activeQuestionnaire.steps[currentStep]
  // ...rest of component
}
```

#### 6. Handle Role Changes During Onboarding

Add a function to manage questionnaire migrations when roles change:

```typescript
// lib/questionnaire-migration.ts
import { QuestionnaireConfig } from "@/types/onboardingQuestion";

// Maps common question IDs between different questionnaire types
const commonQuestionMap: Record<string, Record<string, string>> = {
  // Map from developer role to administrator role
  "developer:administrator": {
    "company-name": "company-name",
    "country-size": "country-size",
    // ...other mappings
  },
  // Add other role combinations as needed
}

export function migrateResponses(
  sourceType: string, 
  targetType: string,
  responses: Record<string, any>
): Record<string, any> {
  // If the same type, no migration needed
  if (sourceType === targetType) return responses;
  
  const migrationKey = `${sourceType}:${targetType}`;
  const mapping = commonQuestionMap[migrationKey];
  
  // If no mapping exists, return original responses
  if (!mapping) return responses;
  
  const migratedResponses: Record<string, any> = {};
  
  // Map responses based on the mapping
  Object.entries(responses).forEach(([questionId, answer]) => {
    const targetQuestionId = mapping[questionId];
    if (targetQuestionId) {
      migratedResponses[targetQuestionId] = answer;
    }
  });
  
  return migratedResponses;
}
```

Then update the `setQuestionnaireType` function in the store:

```typescript
// store/question-store.ts
import { migrateResponses } from "@/lib/questionnaire-migration";

// Inside the store:
setQuestionnaireType: (type: string) => {
  set(state => {
    // Only migrate if the type is changing
    if (type === state.questionnaireType) return { questionnaireType: type };
    
    // Migrate responses to the new questionnaire type
    const migratedResponses = migrateResponses(
      state.questionnaireType, 
      type, 
      state.responses
    );
    
    return {
      questionnaireType: type,
      responses: migratedResponses,
      // Reset to first step when changing questionnaire type
      currentStep: 0,
      // Reset interaction state
      interactedQuestions: []
    };
  });
},
```

This improved solution:

1. ✅ Securely determines the user's role on the server side
2. ✅ Persists the questionnaire type between page refreshes
3. ✅ Adds support for handling role changes during the onboarding process
4. ✅ Properly passes the questionnaire configuration to components rather than relying on global imports

The implementation now properly handles the concerns you mentioned and should work correctly with Clerk's auth system.





***



如果你想换成Python后端验证role，这套方案非常容易修改。你只需要更改API调用部分，而前端逻辑基本不变。下面是具体修改步骤：

### 1. 配置Python API端点

首先，在你的环境配置中添加Python API的URL：

```typescript
// .env.local
NEXT_PUBLIC_QUESTIONNAIRE_API_URL=http://your-python-api.com/api/user-questionnaire
```

### 2. 创建API服务模块

```typescript
// lib/api-service.ts
export async function fetchUserQuestionnaireType() {
  const apiUrl = process.env.NEXT_PUBLIC_QUESTIONNAIRE_API_URL || '/api/user-questionnaire';
  
  // 获取Clerk令牌 - 这部分保持不变
  const clerkToken = await window.Clerk?.session?.getToken();
  
  const response = await fetch(apiUrl, {
    headers: {
      'Authorization': `Bearer ${clerkToken}`,
      'Content-Type': 'application/json'
    }
  });
  
  if (!response.ok) {
    throw new Error(`API error: ${response.status}`);
  }
  
  return response.json();
}
```

### 3. 修改onboarding页面中的API调用

```typescript
// app/onboarding/page.tsx - 只更改loadQuestionnaireType函数
async function loadQuestionnaireType() {
  try {
    setIsLoading(true);
    
    // 使用服务函数而不是直接fetch
    const data = await fetchUserQuestionnaireType();
    
    // 其余逻辑保持不变
    if (data.questionnaireType !== questionnaireType) {
      setQuestionnaireType(data.questionnaireType);
    }
    
    const config = getQuestionnaireByType(data.questionnaireType);
    setActiveQuestionnaireConfig(config);
    
    setIsLoading(false);
  } catch (error) {
    console.error("Error loading questionnaire:", error);
    setError("Failed to load your onboarding process. Please try again.");
    setIsLoading(false);
  }
}
```

### 4. Python后端实现示例

这里是一个简单的Python FastAPI实现示例，你可以根据你的需求调整：

```python
# app.py
from fastapi import FastAPI, Header, HTTPException, Depends
import jwt
import requests
from typing import Optional

app = FastAPI()

# 你的Clerk API密钥配置
CLERK_API_KEY = "your_clerk_api_key"
CLERK_JWT_PUBLIC_KEY = "your_clerk_jwt_public_key"

def verify_token(authorization: str = Header(...)):
    """验证Clerk JWT令牌"""
    if not authorization.startswith("Bearer "):
        raise HTTPException(status_code=401, detail="Invalid authorization header")
    
    token = authorization.replace("Bearer ", "")
    
    try:
        # 解码JWT令牌 (实际实现可能需要用到公钥验证)
        payload = jwt.decode(
            token, 
            CLERK_JWT_PUBLIC_KEY, 
            algorithms=["RS256"],
            audience="your_audience"
        )
        
        # 返回解码后的payload
        return payload
    except jwt.PyJWTError as e:
        raise HTTPException(status_code=401, detail=f"Invalid token: {str(e)}")

@app.get("/api/user-questionnaire")
async def get_user_questionnaire(payload = Depends(verify_token)):
    """根据用户角色返回问卷类型"""
    user_id = payload.get("sub")
    
    if not user_id:
        raise HTTPException(status_code=401, detail="User ID not found in token")
    
    # 从payload中获取组织角色
    org_role = payload.get("org_role")
    
    # 如果token中没有角色，可以从Clerk API获取更多信息
    if not org_role:
        # 调用Clerk API获取用户信息
        response = requests.get(
            f"https://api.clerk.dev/v1/users/{user_id}",
            headers={"Authorization": f"Bearer {CLERK_API_KEY}"}
        )
        
        if response.status_code != 200:
            raise HTTPException(status_code=500, detail="Failed to fetch user data")
            
        user_data = response.json()
        # 根据用户数据确定角色
        org_memberships = user_data.get("organization_memberships", [])
        if org_memberships:
            org_role = org_memberships[0].get("role")
    
    # 确定问卷类型
    questionnaire_type = org_role.lower() if org_role else "developer"
    
    # 确保类型有效
    valid_types = ["developer", "administrator"]
    if questionnaire_type not in valid_types:
        questionnaire_type = "developer"  # 默认值
    
    return {
        "questionnaireType": questionnaire_type,
        "userId": user_id
    }
```

### 5. 进一步改进 - 错误处理和重试逻辑

```typescript
// lib/api-service.ts - 增强版
export async function fetchUserQuestionnaireType(retries = 3) {
  const apiUrl = process.env.NEXT_PUBLIC_QUESTIONNAIRE_API_URL || '/api/user-questionnaire';
  
  let attempts = 0;
  
  while (attempts < retries) {
    try {
      attempts++;
      
      // 获取Clerk令牌
      const clerkToken = await window.Clerk?.session?.getToken();
      
      if (!clerkToken) {
        throw new Error("Authentication token not available");
      }
      
      const response = await fetch(apiUrl, {
        headers: {
          'Authorization': `Bearer ${clerkToken}`,
          'Content-Type': 'application/json'
        }
      });
      
      if (!response.ok) {
        const errorText = await response.text();
        throw new Error(`API error (${response.status}): ${errorText}`);
      }
      
      return await response.json();
    } catch (error) {
      console.error(`API attempt ${attempts} failed:`, error);
      
      // 最后一次尝试失败则抛出错误
      if (attempts >= retries) {
        throw error;
      }
      
      // 等待1秒后重试
      await new Promise(resolve => setTimeout(resolve, 1000));
    }
  }
  
  // 默认返回开发者问卷类型（这里不应该执行到，但为了类型安全）
  return { questionnaireType: "developer" };
}
```

### 优势总结

这种方案的优势在于:

1. **松耦合**: 前端和后端通过明确的API契约交互，使得后端实现可以灵活替换
2. **可配置**: 通过环境变量配置API端点，便于在不同环境切换
3. **JWT处理**: 前端自动获取和传递Clerk令牌，Python后端只需处理JWT验证
4. **错误处理**: 增加了重试机制和详细的错误处理
5. **几乎不需要改动前端逻辑**: 只需更改API调用部分，其余代码保持不变

切换到Python后端时，你只需要:

1. 配置`NEXT_PUBLIC_QUESTIONNAIRE_API_URL`指向Python API
2. 在Python中实现相应的JWT验证和角色确定逻辑
3. 确保Python API返回与前端期望一致的JSON格式

前端其余部分的逻辑完全不需要变动，这使得后端实现可以无缝切换。
