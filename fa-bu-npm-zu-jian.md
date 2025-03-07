# 发布NPM - 组件

首先，你需要将组件库从应用程序中分离出来：

```
Copy/
├── src/
│   ├── components/    # 所有UI组件
│   ├── lib/           # 工具函数和store
│   ├── types/         # 类型定义
│   └── index.ts       # 主导出文件
├── examples/          # 示例用法
├── .npmignore         # npm发布忽略文件
├── tsconfig.json      # TypeScript配置
├── package.json       # 包配置
└── README.md          # 文档
```

#### 2. 配置构建工具

对于React组件库，可以使用tsup、Rollup或esbuild进行构建。以tsup为例：

```
import { defineConfig } from 'tsup';

export default defineConfig({
  entry: ['src/index.ts'],
  format: ['cjs', 'esm'],
  dts: true,
  splitting: false,
  sourcemap: true,
  clean: true,
  external: [
    'react', 
    'react-dom',
    'next', 
    'next/font',
    '@radix-ui/*',
    'framer-motion',
    'lucide-react',
    'sonner',
    'zustand',
    'tailwindcss'
  ],
  treeshake: true,
  minify: true
});
```

#### 3. 编写package.json

```
{
  "name": "react-questionnaire-ui",
  "version": "0.1.0",
  "description": "A customizable questionnaire UI component library for React",
  "author": "Your Name",
  "license": "MIT",
  "main": "dist/index.js",
  "module": "dist/index.mjs",
  "types": "dist/index.d.ts",
  "exports": {
    ".": {
      "require": "./dist/index.js",
      "import": "./dist/index.mjs",
      "types": "./dist/index.d.ts"
    },
    "./styles.css": "./dist/styles.css"
  },
  "files": [
    "dist/",
    "tailwind.config.js"
  ],
  "sideEffects": false,
  "publishConfig": {
    "access": "public"
  },
  "scripts": {
    "build": "tsup && pnpm build:css",
    "build:css": "tailwindcss -i ./src/styles.css -o ./dist/styles.css",
    "dev": "tsup --watch",
    "lint": "eslint \"src/**/*.{ts,tsx}\"",
    "clean": "rm -rf dist",
    "prepublishOnly": "pnpm build"
  },
  "peerDependencies": {
    "react": "^18.0.0",
    "react-dom": "^18.0.0",
    "next": ">=13.0.0",
    "tailwindcss": "^3.3.0",
    "framer-motion": "^10.0.0"
  },
  "dependencies": {
    "@radix-ui/react-accordion": "^1.1.2",
    "@radix-ui/react-alert-dialog": "^1.0.5",
    "@radix-ui/react-checkbox": "^1.0.4",
    "@radix-ui/react-dialog": "^1.0.5",
    "@radix-ui/react-label": "^2.0.2",
    "@radix-ui/react-popover": "^1.0.7",
    "@radix-ui/react-progress": "^1.0.3",
    "@radix-ui/react-radio-group": "^1.1.3",
    "@radix-ui/react-select": "^1.2.2",
    "class-variance-authority": "^0.7.0",
    "clsx": "^2.0.0",
    "lucide-react": "^0.291.0",
    "sonner": "^1.0.0",
    "tailwind-merge": "^1.14.0",
    "zustand": "^4.4.1"
  },
  "devDependencies": {
    "@types/node": "^20.5.7",
    "@types/react": "^18.2.21",
    "@types/react-dom": "^18.2.7",
    "autoprefixer": "^10.4.15",
    "eslint": "^8.48.0",
    "postcss": "^8.4.29",
    "tailwindcss": "^3.3.3",
    "tsup": "^7.2.0",
    "typescript": "^5.2.2"
  },
  "keywords": [
    "react",
    "nextjs",
    "form",
    "questionnaire",
    "survey",
    "ui",
    "components",
    "tailwindcss"
  ]
}
```

#### 4. 创建主导出文件

创建一个`src/index.ts`文件，导出所有公共组件和工具：

```
// 导出组件
export * from './components/NavigationControls';
export * from './components/ProgressIndicator';
export * from './components/QuestionContainer';
export * from './components/QuestionRenderer';
export * from './components/StepConfirmation';
export * from './components/StepContainer';
export * from './components/SubmitButton';
export * from './components/SuccessMessage';

// 导出问题类型组件
export * from './components/questions/SingleChoice';
export * from './components/questions/MultiChoice';
export * from './components/questions/TextInput';
export * from './components/questions/TrueFalse'; 
export * from './components/questions/MatchingQuestion';

// 导出UI组件
export * from './components/ui/alert';
export * from './components/ui/alert-dialog';
export * from './components/ui/button';
export * from './components/ui/checkbox';
export * from './components/ui/progress';
export * from './components/ui/radio-group';
export * from './components/ui/select';

// 导出hooks和工具
export * from './lib/store';
export * from './lib/utils';
export * from './hooks/use-toast';

// 导出类型定义
export * from './types/question';
```

#### 5. 添加一个简单的README

````
# React Questionnaire UI

A modern, customizable questionnaire and survey UI component library for React and Next.js applications, built with TailwindCSS.

## Features

- 🧩 Modular component design
- 📱 Fully responsive UI
- 🎨 Customizable with TailwindCSS
- 🔄 Step-by-step navigation
- ✅ Built-in validation
- 💾 Progress persistence
- 🌙 Light/dark mode support
- 📊 Various question types supported

## Installation

```bash
# npm
npm install react-questionnaire-ui

# yarn
yarn add react-questionnaire-ui

# pnpm
pnpm add react-questionnaire-ui
```

## Setup

### 1. Add tailwind config

Add the component paths to your `tailwind.config.js`:

```js
/** @type {import('tailwindcss').Config} */
module.exports = {
  content: [
    // ...
    "./node_modules/react-questionnaire-ui/**/*.{js,ts,jsx,tsx}",
  ],
  // ...
}
```

### 2. Import the styles

Import the base styles in your main CSS file:

```css
@import 'react-questionnaire-ui/styles.css';
/* Your other styles */
```

## Quick Start

```jsx
import { useQuestionnaireStore } from 'react-questionnaire-ui';
import { StepContainer, NavigationControls, ProgressIndicator } from 'react-questionnaire-ui';

// Define your questionnaire configuration
const questionnaire = {
  id: "my-questionnaire",
  title: "User Survey",
  description: "Help us improve our product",
  steps: [
    {
      id: "step-1",
      title: "Basic Information",
      questions: [
        {
          id: "name",
          type: "text",
          prompt: "What is your name?",
        },
        // More questions...
      ]
    },
    // More steps...
  ]
};

function SurveyForm() {
  const { currentStep, responses } = useQuestionnaireStore();
  const step = questionnaire.steps[currentStep];
  
  return (
    <div className="max-w-2xl mx-auto p-6">
      <h1 className="text-2xl font-bold">{questionnaire.title}</h1>
      <p className="text-gray-500">{questionnaire.description}</p>
      
      <ProgressIndicator />
      
      <StepContainer 
        step={step}
        responses={responses}
        stepIndex={currentStep}
      />
      
      <NavigationControls />
    </div>
  );
}
```

## Documentation

For detailed documentation and examples, visit [the documentation site](https://your-docs-site.com).

## License

MIT
````

#### 6. 添加一个示例配置文件，用于用户自定义问卷

```
import { QuestionnaireConfig } from 'react-questionnaire-ui';

export const customQuestionnaire: QuestionnaireConfig = {
  id: "customer-feedback",
  title: "客户满意度调查",
  description: "感谢您参与我们的调查，您的反馈对我们非常重要！",
  submitButtonText: "提交反馈",
  defaultRequiredMessage: "请填写此项",
  defaultErrorMessage: "输入有误，请检查",
  steps: [
    {
      id: "personal-info",
      title: "个人信息",
      description: "请告诉我们一些关于您的信息",
      label: "信息",
      confirmationMessage: "您确定要继续吗？未填写的信息将无法修改。",
      questions: [
        {
          id: "user-age",
          type: "single",
          prompt: "您的年龄段是？",
          options: ["18岁以下", "18-24岁", "25-34岁", "35-44岁", "45-54岁", "55岁以上"],
          validation: {
            errorMessage: "请选择您的年龄段",
          },
        },
        {
          id: "purchase-frequency",
          type: "single",
          prompt: "您多久购买一次我们的产品？",
          options: ["每周", "每月", "每季度", "每年", "从未购买"],
          optional: true,
        }
      ],
    },
    {
      id: "product-feedback",
      title: "产品反馈",
      description: "请对我们的产品进行评价",
      label: "产品",
      questions: [
        {
          id: "product-rating",
          type: "single",
          prompt: "您对我们的产品总体评价如何？",
          options: ["非常满意", "满意", "一般", "不满意", "非常不满意"],
          validation: {
            errorMessage: "请选择您的评价",
          },
        },
        {
          id: "improvement-areas",
          type: "multi",
          prompt: "您认为我们需要改进的地方有哪些？",
          options: ["产品质量", "价格", "客户服务", "产品多样性", "用户体验", "交付时间"],
          optional: true,
        },
      ],
    },
    {
      id: "comments",
      title: "其他建议",
      description: "请提供您的任何其他反馈或建议",
      label: "建议",
      questions: [
        {
          id: "feedback-text",
          type: "text",
          prompt: "您有什么想告诉我们的？",
          validation: {
            minLength: 10,
            maxLength: 500,
            errorMessage: "请提供至少10个字符的反馈",
          },
        },
        {
          id: "contact-permission",
          type: "trueFalse",
          prompt: "我们可以联系您进一步了解您的反馈吗？",
          optional: true,
        },
      ],
    },
  ],
};

export default customQuestionnaire;
```

#### 7. 添加TailwindCSS配置文件，供用户接入自己的项目

```
/** @type {import('tailwindcss').Config} */
module.exports = {
  darkMode: ['class'],
  content: [
    './src/components/**/*.{js,ts,jsx,tsx,mdx}',
  ],
  theme: {
    extend: {
      colors: {
        background: 'hsl(var(--background))',
        foreground: 'hsl(var(--foreground))',
        card: {
          DEFAULT: 'hsl(var(--card))',
          foreground: 'hsl(var(--card-foreground))',
        },
        popover: {
          DEFAULT: 'hsl(var(--popover))',
          foreground: 'hsl(var(--popover-foreground))',
        },
        primary: {
          DEFAULT: 'hsl(var(--primary))',
          foreground: 'hsl(var(--primary-foreground))',
        },
        secondary: {
          DEFAULT: 'hsl(var(--secondary))',
          foreground: 'hsl(var(--secondary-foreground))',
        },
        muted: {
          DEFAULT: 'hsl(var(--muted))',
          foreground: 'hsl(var(--muted-foreground))',
        },
        accent: {
          DEFAULT: 'hsl(var(--accent))',
          foreground: 'hsl(var(--accent-foreground))',
        },
        destructive: {
          DEFAULT: 'hsl(var(--destructive))',
          foreground: 'hsl(var(--destructive-foreground))',
        },
        border: 'hsl(var(--border))',
        input: 'hsl(var(--input))',
        ring: 'hsl(var(--ring))',
        chart: {
          '1': 'hsl(var(--chart-1))',
          '2': 'hsl(var(--chart-2))',
          '3': 'hsl(var(--chart-3))',
          '4': 'hsl(var(--chart-4))',
          '5': 'hsl(var(--chart-5))',
        },
      },
      borderRadius: {
        lg: 'var(--radius)',
        md: 'calc(var(--radius) - 2px)',
        sm: 'calc(var(--radius) - 4px)',
      },
      keyframes: {
        'accordion-down': {
          from: { height: '0' },
          to: { height: 'var(--radix-accordion-content-height)' },
        },
        'accordion-up': {
          from: { height: 'var(--radix-accordion-content-height)' },
          to: { height: '0' },
        },
      },
      animation: {
        'accordion-down': 'accordion-down 0.2s ease-out',
        'accordion-up': 'accordion-up 0.2s ease-out',
      },
    },
  },
  plugins: [require('tailwindcss-animate')],
}
```

#### 8. 创建样式文件

```
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer base {
  :root {
    --background: 0 0% 100%;
    --foreground: 0 0% 3.9%;
    --card: 0 0% 100%;
    --card-foreground: 0 0% 3.9%;
    --popover: 0 0% 100%;
    --popover-foreground: 0 0% 3.9%;
    --primary: 0 0% 9%;
    --primary-foreground: 0 0% 98%;
    --secondary: 0 0% 96.1%;
    --secondary-foreground: 0 0% 9%;
    --muted: 0 0% 96.1%;
    --muted-foreground: 0 0% 45.1%;
    --accent: 0 0% 96.1%;
    --accent-foreground: 0 0% 9%;
    --destructive: 0 84.2% 60.2%;
    --destructive-foreground: 0 0% 98%;
    --border: 0 0% 89.8%;
    --input: 0 0% 89.8%;
    --ring: 0 0% 3.9%;
    --chart-1: 12 76% 61%;
    --chart-2: 173 58% 39%;
    --chart-3: 197 37% 24%;
    --chart-4: 43 74% 66%;
    --chart-5: 27 87% 67%;
    --radius: 0.5rem;
  }

  .dark {
    --background: 0 0% 3.9%;
    --foreground: 0 0% 98%;
    --card: 0 0% 3.9%;
    --card-foreground: 0 0% 98%;
    --popover: 0 0% 3.9%;
    --popover-foreground: 0 0% 98%;
    --primary: 0 0% 98%;
    --primary-foreground: 0 0% 9%;
    --secondary: 0 0% 14.9%;
    --secondary-foreground: 0 0% 98%;
    --muted: 0 0% 14.9%;
    --muted-foreground: 0 0% 63.9%;
    --accent: 0 0% 14.9%;
    --accent-foreground: 0 0% 98%;
    --destructive: 0 62.8% 30.6%;
    --destructive-foreground: 0 0% 98%;
    --border: 0 0% 14.9%;
    --input: 0 0% 14.9%;
    --ring: 0 0% 83.1%;
    --chart-1: 220 70% 50%;
    --chart-2: 160 60% 45%;
    --chart-3: 30 80% 55%;
    --chart-4: 280 65% 60%;
    --chart-5: 340 75% 55%;
  }
}

@layer base {
  * {
    @apply border-border;
  }
  body {
    @apply bg-background text-foreground;
  }
}
```

#### 9. 添加.npmignore文件

```
# 源代码
src/
examples/

# 配置文件
tsconfig.json
tsup.config.ts
.eslintrc.json
.prettierrc
.editorconfig

# 工具和缓存
node_modules/
.turbo/
.next/
.cache/
coverage/
.nyc_output/

# 测试
test/
__tests__/
**/*.test.ts
**/*.test.tsx
**/*.spec.ts
**/*.spec.tsx

# 日志和临时文件
*.log
npm-debug.log*
yarn-debug.log*
yarn-error.log*
.DS_Store
*.tsbuildinfo

# 其他
.github/
.vscode/
.idea/
.git/
.gitignore
*.md
!README.md
!LICENSE.md
```

````
# 使用指南

本文档将指导您如何自定义和使用React Questionnaire UI组件库。

## 目录

1. [基本设置](#基本设置)
2. [自定义问卷](#自定义问卷)
3. [主题定制](#主题定制)
4. [添加自定义问题类型](#添加自定义问题类型)
5. [数据处理](#数据处理)
6. [常见问题](#常见问题)

## 基本设置

### 安装依赖

确保项目中安装了必要的依赖:

```bash
pnpm add react-questionnaire-ui
pnpm add tailwindcss-animate # 可选，用于动画效果
```

### 配置TailwindCSS

在您的`tailwind.config.js`中添加组件库路径:

```js
/** @type {import('tailwindcss').Config} */
module.exports = {
  content: [
    // 其他路径...
    "./node_modules/react-questionnaire-ui/**/*.{js,ts,jsx,tsx}",
  ],
  // ...其他配置
}
```

### 导入样式

在您的主CSS文件中导入基础样式:

```css
@import 'react-questionnaire-ui/styles.css';
/* 您的其他样式 */
```

## 自定义问卷

### 创建问卷配置

您需要创建一个问卷配置对象:

```tsx
import { QuestionnaireConfig } from 'react-questionnaire-ui';

export const myQuestionnaire: QuestionnaireConfig = {
  id: "my-survey",
  title: "客户调查",
  description: "帮助我们改进服务",
  submitButtonText: "提交反馈",
  steps: [
    {
      id: "step-1",
      title: "基本信息",
      description: "请告诉我们一些关于您的信息",
      label: "基础", // 显示在进度条上的标签
      questions: [
        {
          id: "name",
          type: "text",
          prompt: "您的名字是?",
          // 没有optional标记表示该字段必填
          validation: {
            minLength: 2,
            errorMessage: "请输入您的名字"
          }
        },
        {
          id: "age-group",
          type: "single",
          prompt: "您的年龄段是?",
          options: ["18-24", "25-34", "35-44", "45+"],
          optional: true // 标记为可选字段
        }
      ]
    },
    // 添加更多步骤...
  ]
};
```

### 创建问卷组件

然后创建您的问卷组件:

```tsx
import React from 'react';
import {
  useQuestionnaireStore,
  StepContainer,
  NavigationControls,
  ProgressIndicator,
  SuccessMessage
} from 'react-questionnaire-ui';
import { myQuestionnaire } from './my-questionnaire-config';

export default function MySurvey() {
  const { currentStep, responses, isComplete, submitSuccess } = useQuestionnaireStore();
  
  // 当问卷完成并成功提交时显示成功页面
  if (isComplete && submitSuccess) {
    return <SuccessMessage />;
  }
  
  const currentStepData = myQuestionnaire.steps[currentStep];
  
  return (
    <main className="min-h-screen bg-background p-8">
      <div className="max-w-2xl mx-auto space-y-8">
        <div className="space-y-2">
          <h1 className="text-3xl font-bold">{myQuestionnaire.title}</h1>
          <p className="text-muted-foreground">{myQuestionnaire.description}</p>
        </div>
        
        <ProgressIndicator steps={myQuestionnaire.steps} />
        
        {currentStepData && (
          <StepContainer
            step={currentStepData}
            responses={responses}
            stepIndex={currentStep}
          />
        )}
        
        <NavigationControls questionnaire={myQuestionnaire} />
      </div>
    </main>
  );
}
```

## 主题定制

### 自定义颜色和主题

您可以通过修改CSS变量来自定义颜色:

```css
:root {
  /* 修改主色调 */
  --primary: 210 100% 50%; /* 蓝色主题 */
  --primary-foreground: 0 0% 100%;
  
  /* 修改强调色 */
  --accent: 280 100% 50%; /* 紫色强调 */
  --accent-foreground: 0 0% 100%;
  
  /* 自定义边框圆角 */
  --radius: 0.75rem;
}
```

### 自定义组件样式

您可以通过添加自定义类名来覆盖默认样式:

```tsx
<StepContainer
  step={currentStepData}
  responses={responses}
  stepIndex={currentStep}
  className="bg-blue-50 p-6 rounded-xl shadow-lg"
/>
```

## 添加自定义问题类型

您可以创建自定义问题类型:

1. 创建自定义问题组件:
   
```tsx
// 创建自定义评分问题组件
import React from 'react';
import { cn } from 'react-questionnaire-ui';

interface RatingQuestionProps {
  value?: number;
  onChange: (value: number) => void;
  required?: boolean;
  error?: boolean;
  maxRating?: number;
}

export function RatingQuestion({
  value = 0,
  onChange,
  required,
  error,
  maxRating = 5
}: RatingQuestionProps) {
  return (
    <div className="flex gap-2">
      {Array.from({ length: maxRating }).map((_, index) => (
        <button
          key={index}
          type="button"
          className={cn(
            "p-2 rounded-full w-10 h-10 flex items-center justify-center",
            "border transition-colors",
            index < value 
              ? "bg-primary text-primary-foreground" 
              : "bg-background hover:bg-muted",
            error && "border-destructive"
          )}
          onClick={() => onChange(index + 1)}
        >
          {index + 1}
        </button>
      ))}
    </div>
  );
}
```

2. 扩展问题类型定义:

```tsx
// 扩展types/question.ts
import { QuestionType } from 'react-questionnaire-ui';

// 扩展问题类型
declare module 'react-questionnaire-ui' {
  export interface QuestionTypeMap {
    rating: number; // 增加评分类型
  }
}

// 在问卷配置中使用
const myQuestionnaire = {
  // ...
  questions: [
    {
      id: "satisfaction",
      type: "rating" as QuestionType, // 使用扩展的类型
      prompt: "请评价您的满意度",
      // ...
    }
  ]
};
```

3. 在问题渲染器中添加支持:

```tsx
// 扩展QuestionRenderer组件
import { QuestionRenderer } from 'react-questionnaire-ui';
import { RatingQuestion } from './RatingQuestion';

function CustomQuestionRenderer(props) {
  const { question, onAnswer, currentAnswer, error } = props;
  
  // 处理自定义问题类型
  if (question.type === 'rating') {
    return (
      <RatingQuestion
        value={currentAnswer}
        onChange={(value) => onAnswer(question.id, value)}
        required={!question.optional}
        error={!!error}
      />
    );
  }
  
  // 默认使用库提供的问题渲染器
  return <QuestionRenderer {...props} />;
}
```

## 数据处理

### 获取和保存响应数据

问卷库使用Zustand管理状态，您可以直接访问响应数据:

```tsx
import { useQuestionnaireStore } from 'react-questionnaire-ui';

function SaveResponses() {
  const { responses } = useQuestionnaireStore();
  
  const handleSave = async () => {
    try {
      // 发送到您的API
      const result = await fetch('/api/save-responses', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(responses)
      });
      
      // 处理结果
      if (result.ok) {
        console.log('响应已保存');
      }
    } catch (error) {
      console.error('保存失败', error);
    }
  };
  
  return (
    <button onClick={handleSave}>
      保存当前进度
    </button>
  );
}
```

### 预填响应数据

您可以预填问卷的响应数据:

```tsx
import { useQuestionnaireStore } from 'react-questionnaire-ui';
import { useEffect } from 'react';

function PrefilledQuestionnaire() {
  const { setResponse } = useQuestionnaireStore();
  
  useEffect(() => {
    // 预填数据
    setResponse('name', '张三');
    setResponse('age-group', '25-34');
    // ...其他预填数据
  }, [setResponse]);
  
  // 渲染问卷...
}
```

## 常见问题

### 问题: 如何重置问卷?

```tsx
import { useQuestionnaireStore } from 'react-questionnaire-ui';

function ResetButton() {
  const { resetQuestionnaire } = useQuestionnaireStore();
  
  return (
    <button onClick={resetQuestionnaire}>
      重新开始
    </button>
  );
}
```

### 问题: 如何自定义提交逻辑?

```tsx
import { useQuestionnaireStore } from 'react-questionnaire-ui';
import { toast } from 'sonner';

function CustomSubmitButton() {
  const { responses, validateCurrentStep } = useQuestionnaireStore();
  
  const handleSubmit = async () => {
    // 验证最后一步
    if (!validateCurrentStep()) {
      toast.error('请填写所有必填项');
      return;
    }
    
    try {
      // 自定义提交逻辑
      const result = await fetch('/api/my-custom-endpoint', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          responses,
          submittedAt: new Date().toISOString(),
          // 其他自定义字段
          source: 'website',
          version: '1.0.0'
        })
      });
      
      if (result.ok) {
        toast.success('提交成功!');
        // 其他成功后的操作...
      } else {
        toast.error('提交失败，请重试');
      }
    } catch (error) {
      toast.error('发生错误，请稍后再试');
      console.error(error);
    }
  };
  
  return (
    <button 
      onClick={handleSubmit}
      className="bg-green-600 hover:bg-green-700 text-white px-4 py-2 rounded"
    >
      提交我的回答
    </button>
  );
}
```

### 问题: 如何添加页面跳转逻辑?

```tsx
import { useQuestionnaireStore } from 'react-questionnaire-ui';

function ConditionalNavigation() {
  const { responses, goToStep } = useQuestionnaireStore();
  
  const handleNext = () => {
    // 基于特定问题的回答决定跳转
    if (responses['user-type'] === 'customer') {
      goToStep(3); // 跳转到客户特定部分
    } else if (responses['user-type'] === 'partner') {
      goToStep(5); // 跳转到合作伙伴特定部分
    } else {
      goToStep(2); // 正常流程
    }
  };
  
  return (
    <button onClick={handleNext}>
      继续
    </button>
  );
}
```
````

### 发布到NPM的完整步骤

1. **准备项目结构**
   * 整理源代码，将组件库代码放入`src`目录
   * 确保导出所有必要的组件、工具和类型
2.  **安装开发依赖**

    ```bash
    bashCopypnpm add -D tsup typescript @types/react @types/node
    ```
3. **配置项目文件**
   * 我们已经创建了`package.json`、`tsup.config.ts`、`tailwind.config.js`等配置文件
   * 确保`tsconfig.json`正确配置
4.  **构建库**

    ```bash
    bashCopypnpm build
    ```
5.  **本地测试** 使用`pnpm link`测试组件库是否可以正常工作：

    ```bash
    bashCopycd 你的组件库目录
    pnpm link --global

    # 然后在测试项目中
    cd 你的测试项目
    pnpm link --global react-questionnaire-ui
    ```
6.  **发布到NPM**

    ```bash
    bashCopy# 登录到NPM
    npm login

    # 发布包
    npm publish
    ```

### 组件库结构

我们已经为你准备了以下文件：

* **配置文件**：`package.json`、`tsup.config.ts`、`tailwind.config.js`
* **核心源代码**：`src/index.ts`（主导出文件）
* **样式**：`src/styles.css`
* **文档**：`README.md`、`docs/USAGE.md`
* **配置示例**：`examples/custom-questionnaire.ts`

### 用户使用流程

使用你的组件库，用户需要：

1. 安装库：`pnpm add react-questionnaire-ui`
2. 配置TailwindCSS
3. 导入样式
4. 创建自定义问卷配置
5. 使用提供的组件构建问卷界面

### 后续优化建议

1. **添加测试**：为组件添加单元测试和集成测试
2. **创建演示网站**：展示组件库功能和自定义选项
3. **版本管理**：使用semantic-release自动化版本发布
4. **完善文档**：添加TypeDoc生成API文档
5. **分发CDN版本**：为不使用构建工具的用户提供CDN版本

这套配置为用户提供了高度可定制的问卷UI库，同时保持了使用简便性。用户可以自定义问题类型、主题、样式，并且能够灵活处理表单数据
