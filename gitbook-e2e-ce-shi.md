# Gitbook e2e测试

这是 GitBook 团队用来测试他们**客户网站**的visual testing配置文件。让我解释一下：

### 这个文件的用途

这是一个**客户网站监控测试**，GitBook用它来：

1. **监控客户网站的视觉回归** - 确保GitBook平台的更新不会破坏客户的网站外观
2. **质量保证** - 在发布新版本前，测试真实的客户网站是否正常工作
3. **客户服务** - 主动发现客户网站的问题

### 文件中的网站类型

这些都是使用GitBook平台的**真实客户网站**：

#### 知名公司/产品文档：

* **Snyk** (`docs.snyk.io`) - 代码安全平台
* **Portainer** (`docs.portainer.io`) - Docker容器管理
* **NordVPN** (`meshnet.nordvpn.com`) - VPN服务
* **Bubble** (`manual.bubble.io`) - 无代码开发平台
* **Character.ai** (`book.character.ai`) - AI聊天平台
* **Umbraco** (`docs.umbraco.com`) - CMS内容管理系统

#### 加密货币/区块链项目：

* **PancakeSwap** (`docs.pancakeswap.finance`) - DeFi交易所
* **Raydium** (`docs.raydium.io`) - Solana生态DeFi
* **Hyperliquid** (`hyperliquid.gitbook.io`) - 去中心化交易所

#### AI/技术公司：

* **Dify** (`docs.dify.ai`) - AI应用开发平台
* **GMGN** (`docs.gmgn.ai`) - AI相关服务

### 测试结构解析

```typescript
{
    name: 'Snyk',                                    // 客户名称
    contentBaseURL: 'https://docs.snyk.io',         // 客户网站URL
    tests: [
        { name: 'Home', url: '/', run: waitForCookiesDialog },     // 测试首页
        { name: 'OpenAPI', url: '/snyk-api/reference/apps' },      // 测试API文档页
    ],
}
```

### 特殊配置说明

* `waitForCookiesDialog` - 等待cookie同意弹窗
* `screenshot: { waitForTOCScrolling: false }` - 不等待目录滚动完成
* `skip: process.env.ARGOS_BUILD_NAME !== 'customers-v2'` - 只在特定构建中运行

### 对你的项目的启发

你可以借鉴这种结构为你的项目创建类似的测试：

```typescript
const testCases: TestsCase[] = [
    {
        name: 'Plattii Frontend',
        contentBaseURL: 'https://plattii-frontend.vercel.app',
        tests: [
            { name: 'Home', url: '/' },
            { name: 'Chat', url: '/chat' },
            { name: 'Dashboard', url: '/dashboard' },
        ],
    },
];
```

这是一个很好的visual testing实践，确保了GitBook平台的稳定性和客户满意度！
