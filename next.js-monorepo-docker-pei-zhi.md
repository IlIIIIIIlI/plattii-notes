# Next.js Monorepo Docker 配置

### 1. 根目录 Dockerfile (Multi-stage build)

```dockerfile
# Dockerfile
FROM node:18-alpine AS base

# Install pnpm
RUN npm install -g pnpm

# Set working directory
WORKDIR /app

# Copy workspace configuration
COPY pnpm-workspace.yaml package.json pnpm-lock.yaml ./

# Install dependencies
FROM base AS deps
COPY packages ./packages
COPY apps ./apps
COPY demos ./demos
RUN pnpm install --frozen-lockfile

# Build stage for frontend
FROM base AS frontend-builder
COPY --from=deps /app/node_modules ./node_modules
COPY --from=deps /app/packages ./packages
COPY --from=deps /app/apps/frontend ./apps/frontend
COPY --from=deps /app/apps/backend ./apps/backend

# Build the frontend application
WORKDIR /app/apps/frontend/web
RUN pnpm build

# Build stage for backend (if needed)
FROM base AS backend-builder
COPY --from=deps /app/node_modules ./node_modules
COPY --from=deps /app/packages ./packages
COPY --from=deps /app/apps/backend ./apps/backend

WORKDIR /app/apps/backend
RUN pnpm build

# Production frontend image
FROM node:18-alpine AS frontend-production
WORKDIR /app

# Install pnpm
RUN npm install -g pnpm

# Copy built application
COPY --from=frontend-builder /app/apps/frontend/web/.next ./.next
COPY --from=frontend-builder /app/apps/frontend/web/public ./public
COPY --from=frontend-builder /app/apps/frontend/web/package.json ./package.json
COPY --from=frontend-builder /app/apps/frontend/web/next.config.js ./next.config.js

# Copy necessary packages
COPY --from=deps /app/packages ./packages
COPY --from=deps /app/node_modules ./node_modules

# Expose port
EXPOSE 3000

# Start the application
CMD ["pnpm", "start"]

# Production backend image
FROM node:18-alpine AS backend-production
WORKDIR /app

# Install pnpm
RUN npm install -g pnpm

# Copy built application
COPY --from=backend-builder /app/apps/backend/dist ./dist
COPY --from=backend-builder /app/apps/backend/package.json ./package.json

# Copy necessary packages and dependencies
COPY --from=deps /app/packages ./packages
COPY --from=deps /app/node_modules ./node_modules

# Expose port (adjust according to your backend port)
EXPOSE 8000

# Start the application
CMD ["node", "dist/main.js"]
```

### 2. 前端专用 Dockerfile

```dockerfile
# apps/frontend/web/Dockerfile
FROM node:18-alpine AS base

# Install pnpm
RUN npm install -g pnpm

WORKDIR /app

# Copy workspace files
COPY ../../../pnpm-workspace.yaml ../../../package.json ../../../pnpm-lock.yaml ./

# Copy all packages and apps for workspace resolution
FROM base AS deps
COPY ../../../packages ../../packages
COPY ../../../apps ../../apps
RUN pnpm install --frozen-lockfile

# Build stage
FROM base AS builder
COPY --from=deps /app/node_modules ./node_modules
COPY --from=deps /app/packages ../../packages
COPY . .

ENV NEXT_TELEMETRY_DISABLED 1
RUN pnpm build

# Production stage
FROM node:18-alpine AS runner
WORKDIR /app

ENV NODE_ENV production
ENV NEXT_TELEMETRY_DISABLED 1

RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

# Copy built application
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/public ./public

# Set permissions
USER nextjs

EXPOSE 3000

ENV PORT 3000

CMD ["node", "server.js"]
```

### 3. Docker Compose 配置

```yaml
# docker-compose.yml
version: '3.8'

services:
  frontend:
    build:
      context: .
      dockerfile: Dockerfile
      target: frontend-production
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=${NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY}
      - CLERK_SECRET_KEY=${CLERK_SECRET_KEY}
      - NEXT_PUBLIC_API_URL=${NEXT_PUBLIC_API_URL}
      - OPENAI_API_KEY=${OPENAI_API_KEY}
    depends_on:
      - backend
    networks:
      - app-network

  backend:
    build:
      context: .
      dockerfile: Dockerfile
      target: backend-production
    ports:
      - "8000:8000"
    environment:
      - NODE_ENV=production
      - DATABASE_URL=${DATABASE_URL}
      - JWT_SECRET=${JWT_SECRET}
    networks:
      - app-network

  # Optional: Add database service
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: plattii
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - app-network

networks:
  app-network:
    driver: bridge

volumes:
  postgres_data:
```

### 4. 开发环境 Docker Compose

```yaml
# docker-compose.dev.yml
version: '3.8'

services:
  frontend-dev:
    build:
      context: .
      dockerfile: Dockerfile
      target: base
    ports:
      - "3000:3000"
    volumes:
      - .:/app
      - /app/node_modules
      - /app/apps/frontend/web/.next
    environment:
      - NODE_ENV=development
      - NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=${NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY}
      - CLERK_SECRET_KEY=${CLERK_SECRET_KEY}
      - NEXT_PUBLIC_API_URL=http://backend-dev:8000
    command: sh -c "cd apps/frontend/web && pnpm dev"
    depends_on:
      - backend-dev
    networks:
      - app-network

  backend-dev:
    build:
      context: .
      dockerfile: Dockerfile
      target: base
    ports:
      - "8000:8000"
    volumes:
      - .:/app
      - /app/node_modules
    environment:
      - NODE_ENV=development
    command: sh -c "cd apps/backend && pnpm dev"
    networks:
      - app-network

networks:
  app-network:
    driver: bridge
```

### 5. .dockerignore 文件

```gitignore
# .dockerignore
node_modules
npm-debug.log*
.next
.nuxt
dist
.git
.gitignore
README.md
.env
.env.local
.env.development.local
.env.test.local
.env.production.local
.nyc_output
coverage
.vscode
.idea
*.log
.DS_Store
Thumbs.db

# Test files
**/*.test.js
**/*.test.ts
**/*.spec.js
**/*.spec.ts
e2e
playwright-report
test-results

# Build artifacts
build
dist
.turbo
```

### 6. 环境变量文件

```env
# .env.docker
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=your_clerk_key
CLERK_SECRET_KEY=your_clerk_secret
NEXT_PUBLIC_API_URL=http://backend:8000
OPENAI_API_KEY=your_openai_key
DATABASE_URL=postgresql://postgres:password@postgres:5432/plattii
POSTGRES_PASSWORD=your_postgres_password
JWT_SECRET=your_jwt_secret
```

### 7. 使用方法

#### 构建和运行生产环境

```bash
# 使用 Docker Compose
docker-compose --env-file .env.docker up --build

# 或者单独构建前端
docker build -t plattii-frontend .
docker run -p 3000:3000 --env-file .env.docker plattii-frontend
```

#### 开发环境

```bash
# 运行开发环境
docker-compose -f docker-compose.dev.yml up --build

# 仅运行前端开发
docker-compose -f docker-compose.dev.yml up frontend-dev
```

#### 单独构建镜像

```bash
# 构建前端镜像
docker build --target frontend-production -t plattii-frontend .

# 构建后端镜像
docker build --target backend-production -t plattii-backend .
```

### 8. 优化建议

#### Next.js 优化配置

在 `next.config.js` 中添加：

```javascript
/** @type {import('next').NextConfig} */
const config = {
  // ... 你现有的配置
  output: 'standalone', // 启用独立输出模式
  experimental: {
    // ... 你现有的实验性配置
    outputFileTracingRoot: path.join(__dirname, '../../..'), // monorepo 根目录
  },
}
```

#### Docker 多阶段构建优化

* 使用 Alpine Linux 基础镜像减小体积
* 利用 Docker 层缓存优化构建速度
* 分离依赖安装和代码构建阶段
* 在生产镜像中只包含必要文件

#### 安全考虑

* 使用非 root 用户运行应用
* 正确设置环境变量
* 不在镜像中包含敏感信息
* 定期更新基础镜像

这个配置支持你的整个 monorepo 结构，包括共享的 packages 和多个应用。你可以根据具体需求调整配置。
