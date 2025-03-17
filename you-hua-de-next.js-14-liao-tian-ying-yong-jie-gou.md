# 优化的Next.js 14聊天应用结构

### 聊天应用结构

```
src/
├── app/                           # Next.js App Router
│   ├── layout.tsx                 # 根布局
│   ├── providers.tsx              # 全局Context providers
│   ├── globals.css                # 全局样式
│   │
│   └── chat/                      # 聊天功能路由
│       ├── layout.tsx             # 聊天页面布局
│       ├── page.tsx               # 默认欢迎页
│       │
│       └── [[...id]]/             # 捕获式动态路由(可选参数,支持/chat和/chat/session1)
│           ├── layout.tsx         # 带三栏布局的布局组件
│           ├── loading.tsx        # 加载状态
│           ├── page.tsx           # 会话详情页面
│           │
│           ├── @session/          # 会话列表(平行路由)
│           │   └── default.tsx    # 会话组件入口
│           │
│           ├── @conversation/     # 对话内容(平行路由)
│           │   └── default.tsx    # 对话组件入口
│           │
│           └── @topic/            # 话题面板(平行路由)
│               └── default.tsx    # 话题组件入口
│
├── components/                    # 共享UI组件
│   ├── ui/                        # 基础UI组件
│   │   ├── button.tsx
│   │   ├── input.tsx
│   │   └── ...
│   │
│   ├── chat/                      # 聊天相关组件
│   │   ├── session/               # 会话相关组件
│   │   │   ├── SessionList.tsx
│   │   │   ├── SessionItem.tsx
│   │   │   └── SessionHeader.tsx
│   │   │
│   │   ├── conversation/          # 对话相关组件
│   │   │   ├── MessageList.tsx
│   │   │   ├── MessageItem.tsx
│   │   │   └── ChatInput.tsx
│   │   │
│   │   └── topic/                 # 话题相关组件
│   │       ├── TopicList.tsx
│   │       ├── TopicItem.tsx
│   │       └── TopicHeader.tsx
│   │
│   └── layout/                    # 布局组件
│       ├── ThreeColumnLayout.tsx  # 三栏布局
│       └── ...
│
├── hooks/                         # 自定义Hooks
│   ├── useSession.ts              # 会话相关hook
│   ├── useConversation.ts         # 对话相关hook
│   └── useTopic.ts                # 话题相关hook
│
├── lib/                           # 工具库和辅助函数
│   ├── api.ts                     # API请求函数
│   ├── utils.ts                   # 通用工具函数
│   └── ...
│
├── store/                         # 状态管理
│   ├── session-store.ts           # 会话状态
│   ├── message-store.ts           # 消息状态
│   └── topic-store.ts             # 话题状态
│
└── types/                         # 类型定义
    └── index.ts                   # 类型声明
```



### 架构亮点

这个优化的架构方案有以下优势：

#### 1. 更灵活的路由结构

```
/**
 * 使用捕获式动态路由 [[...id]] 的优势:
 * 
 * 1. 支持可选参数 - 可以同时匹配 /chat 和 /chat/session123
 * 2. 更灵活的URL结构 - 支持 /chat/session123/thread456 这样的嵌套路由
 * 3. 统一的布局 - 无论是首页还是特定会话页面都共享相同的三栏布局
 */

// app/chat/[[...id]]/layout.tsx
// 三栏布局组件，支持任何会话ID

import { ReactNode } from "react";
import ThreeColumnLayout from "@/components/layout/ThreeColumnLayout";

export default function ChatLayout({
  children,
  session,
  conversation,
  topic
}: {
  children: ReactNode;
  session: ReactNode;
  conversation: ReactNode;
  topic: ReactNode;
}) {
  return (
    <ThreeColumnLayout
      session={session}
      conversation={conversation}
      topic={topic}
    >
      {children}
    </ThreeColumnLayout>
  );
}

// app/chat/[[...id]]/page.tsx
// 智能决定显示欢迎页面还是特定会话内容

import { notFound } from "next/navigation";
import WelcomePage from "@/components/chat/WelcomePage";
import { getSessionById } from "@/lib/api";

export default async function ChatPage({ params }: { params: { id?: string[] } }) {
  // 如果没有会话ID参数，显示欢迎页面
  if (!params.id || params.id.length === 0) {
    return <WelcomePage />;
  }
  
  // 提取会话ID (第一个参数)
  const sessionId = params.id[0];
  
  // 获取会话数据
  const session = await getSessionById(sessionId);
  
  // 如果会话不存在，返回404
  if (!session) {
    return notFound();
  }
  
  // 显示会话内容
  return (
    <div className="h-full w-full">
      <div className="text-center">
        <h1 className="text-2xl font-semibold mb-4">{session.title}</h1>
        <p className="text-muted-foreground">{session.description}</p>
      </div>
    </div>
  );
}
```



#### 2. 集中式状态管理

```
/**
 * 使用 Zustand 进行状态管理
 * 
 * 优势:
 * 1. 轻量级 - 相比Redux更轻量简洁
 * 2. 易集成 - 与React和TypeScript无缝集成
 * 3. 易于使用 - 简单API，支持不可变更新
 * 4. 无Provider - 不需要Context Provider包装
 */

// store/session-store.ts
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

export type Session = {
  id: string;
  title: string;
  description: string;
  avatar?: string;
  lastUpdated: number;
};

type SessionState = {
  sessions: Session[];
  activeSessionId: string | null;
  loading: boolean;
  error: string | null;
  
  // 操作
  setActiveSession: (id: string) => void;
  addSession: (session: Omit<Session, 'id' | 'lastUpdated'>) => void;
  updateSession: (id: string, updates: Partial<Session>) => void;
  deleteSession: (id: string) => void;
  fetchSessions: () => Promise<void>;
};

export const useSessionStore = create<SessionState>()(
  persist(
    (set, get) => ({
      sessions: [],
      activeSessionId: null,
      loading: false,
      error: null,
      
      setActiveSession: (id) => set({ activeSessionId: id }),
      
      addSession: (sessionData) => {
        const newSession = {
          id: `session-${Date.now()}`,
          lastUpdated: Date.now(),
          ...sessionData,
        };
        
        set((state) => ({
          sessions: [...state.sessions, newSession],
          activeSessionId: newSession.id,
        }));
      },
      
      updateSession: (id, updates) => {
        set((state) => ({
          sessions: state.sessions.map((session) =>
            session.id === id
              ? { ...session, ...updates, lastUpdated: Date.now() }
              : session
          ),
        }));
      },
      
      deleteSession: (id) => {
        set((state) => {
          const newSessions = state.sessions.filter((s) => s.id !== id);
          
          // 如果删除的是活跃会话，选择新的活跃会话
          let newActiveId = state.activeSessionId;
          if (state.activeSessionId === id) {
            newActiveId = newSessions.length > 0 ? newSessions[0].id : null;
          }
          
          return {
            sessions: newSessions,
            activeSessionId: newActiveId,
          };
        });
      },
      
      fetchSessions: async () => {
        set({ loading: true, error: null });
        
        try {
          // 这里应该是实际的API调用
          const response = await fetch('/api/sessions');
          const data = await response.json();
          
          set({ sessions: data, loading: false });
        } catch (error) {
          console.error('Failed to fetch sessions:', error);
          set({ 
            error: 'Failed to load sessions. Please try again.',
            loading: false
          });
        }
      },
    }),
    {
      name: 'chat-sessions', // 持久化存储的键名
      partialize: (state) => ({ 
        sessions: state.sessions,
        activeSessionId: state.activeSessionId 
      }), // 只持久化部分状态
    }
  )
);

// store/message-store.ts
import { create } from 'zustand';

export type Message = {
  id: string;
  sessionId: string;
  content: string;
  role: 'user' | 'assistant';
  timestamp: number;
};

type MessageState = {
  messages: Record<string, Message[]>; // 以会话ID为键的消息映射
  loading: boolean;
  error: string | null;
  streaming: boolean;
  
  // 操作
  sendMessage: (sessionId: string, content: string) => Promise<void>;
  fetchMessages: (sessionId: string) => Promise<void>;
  clearMessages: (sessionId: string) => void;
  stopStreaming: () => void;
};

export const useMessageStore = create<MessageState>((set, get) => ({
  messages: {},
  loading: false,
  error: null,
  streaming: false,
  
  sendMessage: async (sessionId, content) => {
    // 创建用户消息
    const userMessage: Message = {
      id: `msg-${Date.now()}`,
      sessionId,
      content,
      role: 'user',
      timestamp: Date.now(),
    };
    
    // 更新状态，添加用户消息
    set((state) => ({
      messages: {
        ...state.messages,
        [sessionId]: [
          ...(state.messages[sessionId] || []),
          userMessage,
        ],
      },
      streaming: true,
    }));
    
    try {
      // 调用API发送消息并获取回复
      const response = await fetch('/api/chat', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ 
          sessionId, 
          message: content 
        }),
      });
      
      const data = await response.json();
      
      // 创建助手消息
      const assistantMessage: Message = {
        id: `msg-${Date.now() + 1}`,
        sessionId,
        content: data.response,
        role: 'assistant',
        timestamp: Date.now(),
      };
      
      // 更新状态，添加助手消息
      set((state) => ({
        messages: {
          ...state.messages,
          [sessionId]: [
            ...(state.messages[sessionId] || []),
            assistantMessage,
          ],
        },
        streaming: false,
      }));
    } catch (error) {
      console.error('Failed to send message:', error);
      set({ 
        error: 'Failed to send message. Please try again.',
        streaming: false,
      });
    }
  },
  
  fetchMessages: async (sessionId) => {
    // 如果已有消息，不重复加载
    if (get().messages[sessionId]?.length > 0) return;
    
    set({ loading: true, error: null });
    
    try {
      // 调用API获取消息
      const response = await fetch(`/api/messages?sessionId=${sessionId}`);
      const data = await response.json();
      
      set((state) => ({
        messages: {
          ...state.messages,
          [sessionId]: data,
        },
        loading: false,
      }));
    } catch (error) {
      console.error('Failed to fetch messages:', error);
      set({ 
        error: 'Failed to load messages. Please try again.',
        loading: false,
      });
    }
  },
  
  clearMessages: (sessionId) => {
    set((state) => {
      const newMessages = { ...state.messages };
      delete newMessages[sessionId];
      
      return { messages: newMessages };
    });
  },
  
  stopStreaming: () => {
    // 实际实现中，这里应该调用中断API请求的逻辑
    set({ streaming: false });
  },
}));
```



#### 3. 组件库和布局优化

```
// components/layout/ThreeColumnLayout.tsx
// 可复用的三栏布局组件

import { ReactNode, useEffect, useState } from "react";
import { useSessionStore } from "@/store/session-store";
import { useMessageStore } from "@/store/message-store";
import { useMediaQuery } from "@/hooks/useMediaQuery";
import ChatHeader from "@/components/chat/ChatHeader";

interface ThreeColumnLayoutProps {
  children: ReactNode;
  session: ReactNode;
  conversation: ReactNode;
  topic: ReactNode;
}

export default function ThreeColumnLayout({
  children,
  session,
  conversation,
  topic
}: ThreeColumnLayoutProps) {
  // 响应式布局状态
  const isMobile = useMediaQuery("(max-width: 768px)");
  const isTablet = useMediaQuery("(max-width: 1024px)");
  
  // 面板展开状态
  const [sessionExpanded, setSessionExpanded] = useState(true);
  const [topicExpanded, setTopicExpanded] = useState(!isTablet);
  
  // 从全局状态获取活跃会话ID
  const activeSessionId = useSessionStore(state => state.activeSessionId);
  
  // 根据屏幕大小调整布局
  useEffect(() => {
    if (isMobile) {
      setSessionExpanded(false);
      setTopicExpanded(false);
    } else if (isTablet) {
      setSessionExpanded(true);
      setTopicExpanded(false);
    } else {
      setSessionExpanded(true);
      setTopicExpanded(true);
    }
  }, [isMobile, isTablet]);
  
  return (
    <div className="flex h-screen overflow-hidden">
      {/* 会话列表面板 */}
      <div 
        className={`h-full bg-background border-r transition-all duration-300 ${
          sessionExpanded ? "w-64" : "w-0 overflow-hidden"
        }`}
      >
        {session}
      </div>
      
      {/* 主内容区 */}
      <div className="flex flex-col flex-1 h-full overflow-hidden">
        {/* 顶部导航栏 */}
        <ChatHeader 
          sessionExpanded={sessionExpanded}
          topicExpanded={topicExpanded}
          onToggleSession={() => setSessionExpanded(!sessionExpanded)}
          onToggleTopic={() => setTopicExpanded(!topicExpanded)}
        />
        
        <div className="flex flex-1 overflow-hidden">
          {/* 消息区域 */}
          <div className="flex-1 flex flex-col h-full overflow-hidden">
            {/* 消息列表 */}
            <div className="flex-1 overflow-y-auto">
              {activeSessionId ? conversation : children}
            </div>
          </div>
          
          {/* 话题面板 */}
          <div 
            className={`h-full bg-background border-l transition-all duration-300 ${
              topicExpanded ? "w-64" : "w-0 overflow-hidden"
            }`}
          >
            {topic}
          </div>
        </div>
      </div>
    </div>
  );
}

// components/chat/conversation/MessageList.tsx
// 优化的消息列表组件

import { useEffect, useRef } from "react";
import { useMessageStore } from "@/store/message-store";
import MessageItem from "./MessageItem";
import SkeletonMessage from "./SkeletonMessage";
import WelcomeMessage from "./WelcomeMessage";

interface MessageListProps {
  sessionId: string;
}

export default function MessageList({ sessionId }: MessageListProps) {
  const messages = useMessageStore(state => state.messages[sessionId] || []);
  const loading = useMessageStore(state => state.loading);
  const fetchMessages = useMessageStore(state => state.fetchMessages);
  const endOfMessagesRef = useRef<HTMLDivElement>(null);
  
  // 获取消息和自动滚动到底部
  useEffect(() => {
    if (sessionId) {
      fetchMessages(sessionId);
    }
  }, [sessionId, fetchMessages]);
  
  useEffect(() => {
    endOfMessagesRef.current?.scrollIntoView({ behavior: "smooth" });
  }, [messages.length]);
  
  // 加载状态
  if (loading) {
    return (
      <div className="flex flex-col gap-4 p-4">
        <SkeletonMessage role="user" />
        <SkeletonMessage role="assistant" />
        <SkeletonMessage role="user" />
      </div>
    );
  }
  
  // 空消息列表
  if (messages.length === 0) {
    return <WelcomeMessage sessionId={sessionId} />;
  }
  
  // 渲染消息列表
  return (
    <div className="flex flex-col gap-4 p-4">
      {messages.map(message => (
        <MessageItem 
          key={message.id} 
          message={message}
        />
      ))}
      <div ref={endOfMessagesRef} />
    </div>
  );
}

// components/chat/ChatHeader.tsx
// 智能头部组件

import { useSessionStore } from "@/store/session-store";
import { ActionIcon } from "@lobehub/ui";
import { PanelLeftClose, PanelLeftOpen, PanelRightClose, PanelRightOpen } from "lucide-react";

interface ChatHeaderProps {
  sessionExpanded: boolean;
  topicExpanded: boolean;
  onToggleSession: () => void;
  onToggleTopic: () => void;
}

export default function ChatHeader({
  sessionExpanded,
  topicExpanded,
  onToggleSession,
  onToggleTopic
}: ChatHeaderProps) {
  // 从状态获取当前会话
  const activeSessionId = useSessionStore(state => state.activeSessionId);
  const sessions = useSessionStore(state => state.sessions);
  
  // 查找当前会话信息
  const currentSession = sessions.find(s => s.id === activeSessionId);
  
  return (
    <header className="h-16 border-b flex items-center justify-between px-4">
      <div className="flex items-center gap-2">
        <ActionIcon
          icon={sessionExpanded ? PanelLeftClose : PanelLeftOpen}
          onClick={onToggleSession}
          title={sessionExpanded ? "Close session panel" : "Open session panel"}
        />
        
        {currentSession ? (
          <div>
            <h1 className="text-lg font-medium">{currentSession.title}</h1>
            <p className="text-sm text-muted-foreground">{currentSession.description}</p>
          </div>
        ) : (
          <h1 className="text-lg font-medium">Chat</h1>
        )}
      </div>
      
      <ActionIcon
        icon={topicExpanded ? PanelRightClose : PanelRightOpen}
        onClick={onToggleTopic}
        title={topicExpanded ? "Close topic panel" : "Open topic panel"}
      />
    </header>
  );
}
```



#### 4. 自定义Hooks封装业务逻辑

```
// hooks/useChat.ts
// 聊天功能的核心hook，将所有相关逻辑封装为一个易用的hook

import { useState, useCallback } from 'react';
import { useSessionStore } from '@/store/session-store';
import { useMessageStore } from '@/store/message-store';
import { useTopicStore } from '@/store/topic-store';
import { useRouter } from 'next/navigation';

export function useChat() {
  const router = useRouter();
  const [inputValue, setInputValue] = useState('');
  
  // 从状态获取会话信息
  const {
    sessions,
    activeSessionId,
    setActiveSession,
    addSession,
    updateSession,
    deleteSession,
  } = useSessionStore();
  
  // 从状态获取消息信息
  const {
    messages,
    loading: messagesLoading,
    streaming,
    sendMessage,
    fetchMessages,
    stopStreaming,
  } = useMessageStore();
  
  // 从状态获取话题信息
  const {
    topics,
    activeTopic,
    setActiveTopic,
  } = useTopicStore();
  
  // 当前会话的消息
  const currentMessages = activeSessionId ? messages[activeSessionId] || [] : [];
  
  // 当前会话的话题
  const currentTopics = activeSessionId ? topics[activeSessionId] || [] : [];
  
  // 当前会话信息
  const currentSession = sessions.find(s => s.id === activeSessionId);
  
  // 发送消息
  const handleSendMessage = useCallback(async () => {
    if (!inputValue.trim() || !activeSessionId) return;
    
    try {
      await sendMessage(activeSessionId, inputValue);
      setInputValue('');
      
      // 更新会话的最后更新时间
      updateSession(activeSessionId, { lastUpdated: Date.now() });
    } catch (error) {
      console.error('Failed to send message:', error);
    }
  }, [inputValue, activeSessionId, sendMessage, updateSession]);
  
  // 选择会话
  const handleSelectSession = useCallback((sessionId: string) => {
    setActiveSession(sessionId);
    fetchMessages(sessionId);
    router.push(`/chat/${sessionId}`);
  }, [setActiveSession, fetchMessages, router]);
  
  // 创建新会话
  const handleNewSession = useCallback(() => {
    const newSession = {
      title: 'New Chat',
      description: new Date().toLocaleString(),
    };
    
    addSession(newSession);
    // 新会话ID会由addSession生成并设置为activeSessionId
  }, [addSession]);
  
  // 删除会话
  const handleDeleteSession = useCallback((sessionId: string) => {
    deleteSession(sessionId);
    
    // 如果删除的是当前活跃会话，路由回到首页
    if (sessionId === activeSessionId) {
      router.push('/chat');
    }
  }, [deleteSession, activeSessionId, router]);
  
  return {
    // 会话相关
    sessions,
    currentSession,
    activeSessionId,
    selectSession: handleSelectSession,
    newSession: handleNewSession,
    deleteSession: handleDeleteSession,
    
    // 消息相关
    messages: currentMessages,
    messagesLoading,
    streaming,
    inputValue,
    setInputValue,
    sendMessage: handleSendMessage,
    stopStreaming,
    
    // 话题相关
    topics: currentTopics,
    activeTopic,
    setActiveTopic,
  };
}

// hooks/useResponsive.ts
// 响应式布局hook

import { useEffect, useState } from 'react';

// 预定义断点
type Breakpoint = 'xs' | 'sm' | 'md' | 'lg' | 'xl' | '2xl';

const breakpoints: Record<Breakpoint, number> = {
  xs: 0,
  sm: 640,
  md: 768,
  lg: 1024,
  xl: 1280,
  '2xl': 1536,
};

export function useResponsive() {
  // 初始状态设为undefined，等待客户端初始化
  const [windowSize, setWindowSize] = useState({
    width: undefined as number | undefined,
    height: undefined as number | undefined,
  });
  
  useEffect(() => {
    // 只在客户端运行
    if (typeof window === 'undefined') return;
    
    // 处理窗口大小变化
    function handleResize() {
      setWindowSize({
        width: window.innerWidth,
        height: window.innerHeight,
      });
    }
    
    // 添加resize监听
    window.addEventListener('resize', handleResize);
    
    // 初始化窗口大小
    handleResize();
    
    // 清理监听
    return () => window.removeEventListener('resize', handleResize);
  }, []);
  
  // 生成响应式对象
  const responsive: Record<Breakpoint, boolean> = {
    xs: (windowSize.width ?? 0) >= breakpoints.xs,
    sm: (windowSize.width ?? 0) >= breakpoints.sm,
    md: (windowSize.width ?? 0) >= breakpoints.md,
    lg: (windowSize.width ?? 0) >= breakpoints.lg,
    xl: (windowSize.width ?? 0) >= breakpoints.xl,
    '2xl': (windowSize.width ?? 0) >= breakpoints['2xl'],
  };
  
  // 当前断点
  const currentBreakpoint = Object.entries(breakpoints)
    .sort(([, a], [, b]) => b - a)
    .find(([, size]) => (windowSize.width ?? 0) >= size)?.[0] as Breakpoint | undefined;
  
  return {
    ...responsive,
    windowSize,
    currentBreakpoint,
    isMobile: !responsive.md,
    isTablet: !responsive.lg,
    isDesktop: responsive.lg,
  };
}

// hooks/useMediaQuery.ts
// 媒体查询hook

import { useEffect, useState } from 'react';

export function useMediaQuery(query: string): boolean {
  const [matches, setMatches] = useState(false);
  
  useEffect(() => {
    // 服务器端渲染时，默认为false
    if (typeof window === 'undefined') return;
    
    const mediaQuery = window.matchMedia(query);
    
    // 初始匹配状态
    setMatches(mediaQuery.matches);
    
    // 监听媒体查询变化
    const handler = (event: MediaQueryListEvent) => {
      setMatches(event.matches);
    };
    
    // 添加监听
    mediaQuery.addEventListener('change', handler);
    
    // 清理监听
    return () => {
      mediaQuery.removeEventListener('change', handler);
    };
  }, [query]);
  
  return matches;
}
```



#### 5. 集成API路由

```
// app/api/sessions/route.ts
// 会话相关API

import { NextResponse } from 'next/server';

// 模拟数据库
const sessions = [
  {
    id: 'inbox',
    title: 'Inbox',
    description: 'Default inbox for new messages',
    avatar: '/icons/inbox.png',
    lastUpdated: Date.now(),
  },
  {
    id: 'session1',
    title: 'AI Assistant',
    description: 'General purpose AI assistant',
    avatar: '/icons/assistant.png',
    lastUpdated: Date.now() - 3600000,
  },
  {
    id: 'session2',
    title: 'Code Helper',
    description: 'Specialized in programming assistance',
    avatar: '/icons/code.png',
    lastUpdated: Date.now() - 7200000,
  },
  {
    id: 'session3',
    title: 'Math Tutor',
    description: 'Help with mathematics problems',
    avatar: '/icons/math.png',
    lastUpdated: Date.now() - 10800000,
  },
];

// GET /api/sessions - 获取所有会话
export async function GET() {
  // 在实际应用中，这里应该从数据库获取数据
  return NextResponse.json(sessions);
}

// POST /api/sessions - 创建新会话
export async function POST(request: Request) {
  try {
    const body = await request.json();
    const { title, description, avatar } = body;
    
    // 验证必要字段
    if (!title) {
      return NextResponse.json(
        { error: 'Title is required' },
        { status: 400 }
      );
    }
    
    // 创建新会话
    const newSession = {
      id: `session-${Date.now()}`,
      title,
      description: description || '',
      avatar: avatar || null,
      lastUpdated: Date.now(),
    };
    
    // 在实际应用中，这里应该将新会话保存到数据库
    sessions.push(newSession);
    
    return NextResponse.json(newSession, { status: 201 });
  } catch (error) {
    console.error('Error creating session:', error);
    return NextResponse.json(
      { error: 'Failed to create session' },
      { status: 500 }
    );
  }
}

// app/api/sessions/[id]/route.ts
// 单个会话API

import { NextResponse } from 'next/server';

// 模拟数据库
const sessions = [
  {
    id: 'inbox',
    title: 'Inbox',
    description: 'Default inbox for new messages',
    avatar: '/icons/inbox.png',
    lastUpdated: Date.now(),
  },
  // ...其他会话
];

// GET /api/sessions/:id - 获取指定会话
export async function GET(
  request: Request,
  { params }: { params: { id: string } }
) {
  const id = params.id;
  
  // 在实际应用中，这里应该从数据库查询
  const session = sessions.find(s => s.id === id);
  
  if (!session) {
    return NextResponse.json(
      { error: 'Session not found' },
      { status: 404 }
    );
  }
  
  return NextResponse.json(session);
}

// PATCH /api/sessions/:id - 更新会话
export async function PATCH(
  request: Request,
  { params }: { params: { id: string } }
) {
  try {
    const id = params.id;
    const body = await request.json();
    
    // 在实际应用中，这里应该更新数据库
    const sessionIndex = sessions.findIndex(s => s.id === id);
    
    if (sessionIndex === -1) {
      return NextResponse.json(
        { error: 'Session not found' },
        { status: 404 }
      );
    }
    
    // 更新会话，保留原有字段
    const updatedSession = {
      ...sessions[sessionIndex],
      ...body,
      lastUpdated: Date.now(),
    };
    
    sessions[sessionIndex] = updatedSession;
    
    return NextResponse.json(updatedSession);
  } catch (error) {
    console.error('Error updating session:', error);
    return NextResponse.json(
      { error: 'Failed to update session' },
      { status: 500 }
    );
  }
}

// DELETE /api/sessions/:id - 删除会话
export async function DELETE(
  request: Request,
  { params }: { params: { id: string } }
) {
  const id = params.id;
  
  // 在实际应用中，这里应该从数据库删除
  const sessionIndex = sessions.findIndex(s => s.id === id);
  
  if (sessionIndex === -1) {
    return NextResponse.json(
      { error: 'Session not found' },
      { status: 404 }
    );
  }
  
  sessions.splice(sessionIndex, 1);
  
  return NextResponse.json({ success: true });
}

// app/api/messages/route.ts
// 消息相关API

import { NextResponse } from 'next/server';

// 模拟数据库
const messagesDB: Record<string, any[]> = {
  'inbox': [
    {
      id: 'welcome',
      sessionId: 'inbox',
      content: 'Welcome to your inbox! How can I assist you today?',
      role: 'assistant',
      timestamp: Date.now() - 3600000,
    }
  ],
  'session1': [
    {
      id: 'welcome-1',
      sessionId: 'session1',
      content: 'Hello! I\'m your AI Assistant. How can I help you today?',
      role: 'assistant',
      timestamp: Date.now() - 3600000,
    },
    {
      id: 'user-1',
      sessionId: 'session1',
      content: 'I\'m looking for information about building a chat interface.',
      role: 'user',
      timestamp: Date.now() - 3500000,
    },
    {
      id: 'assistant-1',
      sessionId: 'session1',
      content: 'Building a chat interface involves several key components: message display, input area, and styling to differentiate between user and assistant messages. Would you like me to explain more about any specific aspect?',
      role: 'assistant',
      timestamp: Date.now() - 3400000,
    },
  ],
  // ...其他会话的消息
};

// GET /api/messages?sessionId=xxx - 获取指定会话的消息
export async function GET(request: Request) {
  const { searchParams } = new URL(request.url);
  const sessionId = searchParams.get('sessionId');
  
  if (!sessionId) {
    return NextResponse.json(
      { error: 'Session ID is required' },
      { status: 400 }
    );
  }
  
  // 在实际应用中，这里应该从数据库查询
  const messages = messagesDB[sessionId] || [];
  
  return NextResponse.json(messages);
}

// app/api/chat/route.ts
// 聊天API端点

import { NextResponse } from 'next/server';
import { OpenAIStream, StreamingTextResponse } from 'ai';
import OpenAI from 'openai';

// 初始化OpenAI API
const openai = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY,
});

// 模拟数据库中的消息
const messagesDB: Record<string, any[]> = {
  // ...与之前相同
};

// POST /api/chat - 处理聊天消息
export async function POST(request: Request) {
  try {
    const { sessionId, message } = await request.json();
    
    if (!sessionId || !message) {
      return NextResponse.json(
        { error: 'Session ID and message are required' },
        { status: 400 }
      );
    }
    
    // 保存用户消息
    const userMessage = {
      id: `msg-${Date.now()}`,
      sessionId,
      content: message,
      role: 'user',
      timestamp: Date.now(),
    };
    
    if (!messagesDB[sessionId]) {
      messagesDB[sessionId] = [];
    }
    
    messagesDB[sessionId].push(userMessage);
    
    // 在实际应用中，这里应该调用AI API
    // 以下是一个模拟响应
    
    // 如果有API密钥，使用实际的OpenAI API
    if (process.env.OPENAI_API_KEY) {
      // 为OpenAI准备消息历史
      const history = messagesDB[sessionId].map(msg => ({
        role: msg.role,
        content: msg.content,
      }));
      
      // 调用OpenAI API
      const response = await openai.chat.completions.create({
        model: 'gpt-4',
        messages: history,
        stream: true,
      });
      
      // 创建流式响应
      const stream = OpenAIStream(response);
      return new StreamingTextResponse(stream);
    } else {
      // 模拟响应
      const aiResponses = {
        'inbox': 'I am your inbox assistant. How can I help you today?',
        'session1': 'As your AI Assistant, I\'d be happy to help with that. Can you provide more details?',
        'session2': 'From a coding perspective, I recommend considering the following approach...',
        'session3': 'For this math problem, let\'s break it down step by step...',
      };
      
      // 使用默认响应或根据会话类型响应
      const responseText = aiResponses[sessionId] || 
        `I received your message: "${message}". How can I assist you further?`;
      
      // 模拟AI思考时间
      await new Promise(resolve => setTimeout(resolve, 1000));
      
      // 保存助手回复
      const assistantMessage = {
        id: `msg-${Date.now() + 1}`,
        sessionId,
        content: responseText,
        role: 'assistant',
        timestamp: Date.now(),
      };
      
      messagesDB[sessionId].push(assistantMessage);
      
      return NextResponse.json({ response: responseText });
    }
  } catch (error) {
    console.error('Error processing chat:', error);
    return NextResponse.json(
      { error: 'Failed to process chat message' },
      { status: 500 }
    );
  }
}
```



### 为什么这是更好的结构？

1. **捕获式动态路由**：使用 `[[...id]]` 这样的捕获式动态路由，可以同时处理 `/chat` 和 `/chat/session123` 这样的路径，让布局更加统一。
2. **逻辑分离**：明确地将组件、布局、状态管理和API路由分开，使代码更易维护和扩展。
3. **集中式状态管理**：使用Zustand管理状态，比分散在各个组件的useState更易于管理，且无需Context层层传递。
4. **自定义Hooks**：将业务逻辑封装在自定义Hooks中，让组件保持简洁，只关注渲染。
5. **响应式设计**：通过媒体查询和响应式Hooks，实现在不同设备上的最佳体验。
6. **API路由分离**：使用Next.js内置的API路由功能，实现前后端分离，易于扩展。
7. **代码目录结构清晰**：按功能而非技术类型组织文件，使相关文件靠近，减少跳转查找。

### 实现步骤

1. 根据上述结构重新组织你的项目文件
2. 实现状态管理层和Hooks
3. 创建核心布局和组件
4. 使用上述代码片段构建API路由
5. 连接状态和UI

这种架构不仅解决了当前的问题，还为未来的扩展提供了坚实基础，比如添加多用户支持、持久化存储、插件系统等。同时，这种结构也更符合现代React和Next.js的最佳实践。
