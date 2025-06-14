# Gitbook e2e测试

```tsx
import { type TestsCase, runTestCases, waitForCookiesDialog } from './util';

/** A list of test cases to run on the customers' docs sites. */
const testCases: TestsCase[] = [
    {
        name: 'Snyk',
        contentBaseURL: 'https://docs.snyk.io',
        tests: [
            { name: 'Home', url: '/', run: waitForCookiesDialog },
            { name: 'OpenAPI', url: '/snyk-api/reference/apps', run: waitForCookiesDialog },
        ],
    },
    // {
    //     name: 'Nexthink',
    //     contentBaseURL: 'https://docs.nexthink.com',
    //     tests: [
    //         {
    //             name: 'Home',
    //             url: '/',
    //             screenshot: { waitForTOCScrolling: false },
    //             run: waitForCookiesDialog,
    //         },
    //     ],
    // },
    {
        name: 'asiksupport-stg.dto.kemkes.go.id',
        contentBaseURL: 'https://asiksupport-stg.dto.kemkes.go.id',
        tests: [{ name: 'Home', url: '/' }],
    },
    {
        name: 'jasons-tutorials.gitbook.io',
        contentBaseURL: 'https://jasons-tutorials.gitbook.io',
        tests: [{ name: 'Home', url: '/' }],
    },
    {
        name: 'faq.deltaemulator.com',
        contentBaseURL: 'https://faq.deltaemulator.com',
        tests: [{ name: 'Home', url: '/' }],
    },
    {
        name: 'docs.dify.ai',
        contentBaseURL: 'https://docs.dify.ai',
        tests: [{ name: 'Home', url: '/', run: waitForCookiesDialog }],
    },
    {
        name: 'seeddao.gitbook.io',
        contentBaseURL: 'https://seeddao.gitbook.io',
        tests: [{ name: 'Home', url: '/' }],
    },
    {
        name: 'faq.altstore.io',
        contentBaseURL: 'https://faq.altstore.io',
        tests: [{ name: 'Home', url: '/' }],
    },
    {
        name: 'support.audacityteam.org',
        contentBaseURL: 'https://support.audacityteam.org',
        tests: [{ name: 'Home', url: '/' }],
    },
    {
        name: 'docs.gmgn.ai',
        contentBaseURL: 'https://docs.gmgn.ai',
        tests: [{ name: 'Home', url: '/' }],
    },
    {
        name: 'docs.spicychat.ai',
        contentBaseURL: 'https://docs.spicychat.ai',
        tests: [{ name: 'Home', url: '/' }],
    },
    {
        name: 'docs.portainer.io',
        contentBaseURL: 'https://docs.portainer.io',
        tests: [{ name: 'Home', url: '/', run: waitForCookiesDialog }],
    },
    {
        name: 'docs.chirptoken.io',
        contentBaseURL: 'https://docs.chirptoken.io',
        tests: [{ name: 'Home', url: '/', run: waitForCookiesDialog }],
    },
    {
        name: 'docs.dexscreener.com',
        contentBaseURL: 'https://docs.dexscreener.com',
        tests: [{ name: 'Home', url: '/', run: waitForCookiesDialog }],
    },
    {
        name: 'docs.pancakeswap.finance',
        contentBaseURL: 'https://docs.pancakeswap.finance',
        tests: [{ name: 'Home', url: '/', run: waitForCookiesDialog }],
    },
    {
        name: 'book.character.ai',
        contentBaseURL: 'https://book.character.ai',
        tests: [{ name: 'Home', url: '/', run: waitForCookiesDialog }],
    },
    {
        name: 'azcoiner.gitbook.io',
        contentBaseURL: 'https://azcoiner.gitbook.io',
        tests: [{ name: 'Home', url: '/' }],
    },
    {
        name: 'docs.midas.app',
        contentBaseURL: 'https://docs.midas.app',
        tests: [{ name: 'Home', url: '/' }],
    },
    {
        name: 'docs.keeper.io',
        contentBaseURL: 'https://docs.keeper.io',
        tests: [{ name: 'Home', url: '/', run: waitForCookiesDialog }],
    },
    {
        name: 'adiblar.gitbook.io',
        contentBaseURL: 'https://adiblar.gitbook.io',
        tests: [{ name: 'Home', url: '/' }],
    },
    {
        name: 'docs.gradient.network',
        contentBaseURL: 'https://docs.gradient.network',
        tests: [{ name: 'Home', url: '/' }],
    },
    {
        name: 'mygate-network.gitbook.io',
        contentBaseURL: 'https://mygate-network.gitbook.io',
        tests: [{ name: 'Home', url: '/' }],
    },
    {
        name: 'treasurenft.gitbook.io',
        contentBaseURL: 'https://treasurenft.gitbook.io',
        tests: [{ name: 'Home', url: '/' }],
    },
    {
        name: 'browndust2.gitbook.io',
        contentBaseURL: 'https://browndust2.gitbook.io',
        tests: [{ name: 'Home', url: '/', screenshot: { waitForTOCScrolling: false } }],
    },
    {
        name: 'junookyo.gitbook.io',
        contentBaseURL: 'https://junookyo.gitbook.io',
        tests: [{ name: 'Home', url: '/' }],
    },
    {
        name: 'meshnet.nordvpn.com',
        contentBaseURL: 'https://meshnet.nordvpn.com',
        tests: [{ name: 'Home', url: '/', run: waitForCookiesDialog }],
    },
    {
        name: 'manual.bubble.io',
        contentBaseURL: 'https://manual.bubble.io',
        tests: [{ name: 'Home', url: '/', run: waitForCookiesDialog }],
    },
    {
        name: 'docs.tickettool.xyz',
        contentBaseURL: 'https://docs.tickettool.xyz',
        tests: [{ name: 'Home', url: '/' }],
    },
    {
        name: 'wiki.redmodding.org',
        contentBaseURL: 'https://wiki.redmodding.org',
        tests: [{ name: 'Home', url: '/' }],
    },
    // {
    //     name: 'docs.cherry-ai.com',
    //     contentBaseURL: 'https://docs.cherry-ai.com',
    //     tests: [{ name: 'Home', url: '/', run: waitForCookiesDialog }],
    // },
    {
        name: 'docs.snyk.io',
        contentBaseURL: 'https://docs.snyk.io',
        tests: [{ name: 'Home', url: '/', run: waitForCookiesDialog }],
    },
    {
        name: 'docs.realapp.link',
        contentBaseURL: 'https://docs.realapp.link',
        tests: [{ name: 'Home', url: '/' }],
    },
    {
        name: 'docs.plaza.finance',
        contentBaseURL: 'https://docs.plaza.finance',
        tests: [{ name: 'Home', url: '/' }],
    },
    {
        name: 'docs.publicai.io',
        contentBaseURL: 'https://docs.publicai.io',
        tests: [{ name: 'Home', url: '/' }],
    },
    {
        name: 'hyperliquid.gitbook.io',
        contentBaseURL: 'https://hyperliquid.gitbook.io',
        tests: [{ name: 'Home', url: '/' }],
    },
    {
        name: 'docs.umbraco.com',
        contentBaseURL: 'https://docs.umbraco.com',
        tests: [
            {
                name: 'Home',
                url: '/welcome',
                run: waitForCookiesDialog,
                screenshot: { waitForTOCScrolling: false },
            },
        ],
    },
    {
        name: 'sosovalue-white-paper.gitbook.io',
        contentBaseURL: 'https://sosovalue-white-paper.gitbook.io',
        tests: [{ name: 'Home', url: '/' }],
    },
    // {
    //     name: 'docs.revrobotics.com',
    //     contentBaseURL: 'https://docs.revrobotics.com',
    //     tests: [{ name: 'Home', url: '/', run: waitForCookiesDialog }],
    // },
    {
        name: 'chartschool.stockcharts.com',
        contentBaseURL: 'https://chartschool.stockcharts.com',
        tests: [{ name: 'Home', url: '/', run: waitForCookiesDialog }],
    },
    {
        name: 'docs.soniclabs.com',
        contentBaseURL: 'https://docs.soniclabs.com',
        tests: [{ name: 'Home', url: '/' }],
    },
    {
        name: 'docs.meshchain.ai',
        contentBaseURL: 'https://docs.meshchain.ai',
        tests: [{ name: 'Home', url: '/' }],
    },
    {
        name: 'docs.thousandeyes.com',
        contentBaseURL: 'https://docs.thousandeyes.com',
        tests: [{ name: 'Home', url: '/', run: waitForCookiesDialog }],
    },
    {
        name: 'docs.raydium.io',
        contentBaseURL: 'https://docs.raydium.io',
        tests: [{ name: 'Home', url: '/' }],
    },
    {
        name: 'docs.fluentbit.io',
        contentBaseURL: 'https://docs.fluentbit.io',
        tests: [{ name: 'Home', url: '/', run: waitForCookiesDialog }],
    },
    {
        name: 'run-ai-docs.nvidia.com',
        contentBaseURL: 'https://run-ai-docs.nvidia.com',
        skip: process.env.ARGOS_BUILD_NAME !== 'customers-v2',
        tests: [
            { name: 'Home', url: '/' },
            { name: 'OG Image', url: '/~gitbook/ogimage/h17zQIFwy3MaafVNmItO', mode: 'image' },
        ],
    },
];

runTestCases(testCases);
```

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
