# 多链 Rugpull 与蜜罐风险扫描仪 — 设计文档

> 单网页（Single HTML）架构，底层调用 mimo-v2.5-pro 模型

**日期**: 2026-06-15
**状态**: 设计已确认，待实现

---

## 1. 整体架构

### 1.1 文件结构

单一 `rugpull-scanner.html`，内部分为 6 个功能区块，用 IIFE 隔离作用域。

```
┌─────────────────────────────────────────────┐
│  <head>                                     │
│    ├─ <link> highlight.js github-dark 主题   │
│    ├─ <script src="tailwindcss Play CDN">   │
│    ├─ <script src="echarts@5.5.0">          │
│    ├─ <script src="marked/marked.min.js">   │
│    ├─ <script src="highlight.js core">      │
│    └─ <script src="highlightjs-solidity">   │
├─────────────────────────────────────────────┤
│  <body>   UI 结构                            │
│    ├─ Header: 标题 + API Key 配置区          │
│    ├─ Input Panel: 合约地址/源码输入         │
│    ├─ Chain Selector: 多链选择               │
│    ├─ Scan Button + Progress Indicator       │
│    ├─ Results Panel                          │
│    │   ├─ ECharts 雷达图 + 仪表盘            │
│    │   ├─ 红牌警告 / 绿牌通过卡片            │
│    │   └─ 拼装的 Markdown 审计报告           │
│    ├─ 安全警告（红色醒目提示）               │
│    └─ Footer: 免责声明                       │
├─────────────────────────────────────────────┤
│  <script>  应用逻辑                          │
│    ├─ [1] Config & Constants                 │
│    ├─ [2] Storage + MessageState             │
│    ├─ [3] API Client (Xiaomi + Etherscan)    │
│    ├─ [4] Contract Parser (import 解析)      │
│    ├─ [5] AI Analyzer (prompt + 调用)        │
│    └─ [6] UI Renderer (ECharts + Markdown)   │
└─────────────────────────────────────────────┘
```

### 1.2 数据流

```
用户点击扫描 → 防抖+按钮禁用 → 获取合约源码(自动/手动)
  → 解析 import 依赖(三色拓扑排序) → 合并为单一源码
  → token 估算检查 → 组装 prompt(System锁死+User动态)
  → 调用 mimo API(fetchWithRetry) → 解析 AI 响应
  → Layer 1: JSON.parse → Layer 2: 字段校验 → Layer 3: Markdown降级
  → ECharts 渲染(雷达图+仪表盘) + 红绿卡片(hljs高亮) + 拼装报告
```

### 1.3 CDN 依赖清单

```html
<head>
  <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/highlight.js@11.9.0/styles/github-dark.min.css">
  <script src="https://cdn.tailwindcss.com"></script>
  <script src="https://cdn.jsdelivr.net/npm/echarts@5.5.0/dist/echarts.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/marked/marked.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/highlight.js@11.9.0/lib/core.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/highlightjs-solidity@2.0.6/dist/solidity.min.js"></script>
</head>
```

- Tailwind CSS 使用 Play CDN（`<script>` 而非 `<link>`，运行时动态生成 CSS）
- marked.js 负责 Markdown 渲染
- highlight.js + highlightjs-solidity 负责 Solidity 语法高亮
- ECharts 负责雷达图和仪表盘

---

## 2. API 调用与 Key 安全存储

### 2.1 小米 MiMo API

- 前端直连小米 API（`https://token-plan-cn.xiaomimimo.com/v1`），小米网关已配置 CORS 策略支持 Preflight 预检请求
- API Key（`tp-` 前缀）仅存储在用户浏览器的 `localStorage` 中，绝不上传任何第三方服务器
- 浏览器会自动发送 OPTIONS 预检请求（因自定义请求头 `api-key`），小米网关已支持

### 2.2 Etherscan API

- 支持免费注册的 API Key，同样存入 localStorage
- 可选配置：无 Key 时使用匿名模式，但会触发严格限流（~1 req/s），UI 同步提示用户

### 2.3 安全警告（硬编码在 UI 中）

> ⚠️ 安全警告：API Key 存储在本地浏览器中，请仅在本地双击运行、localhost 或您完全私有的独立域名下使用。切勿在公共多用户共享域名下运行，以防您的 Credit 资产被 XSS 脚本窃取！

### 2.4 localStorage 持久化

```javascript
const STORAGE_KEYS = {
  MIMO_KEY: 'rugscan_mimo_key',
  ETHERSCAN_KEY: 'rugscan_etherscan_key',
  CHAIN: 'rugscan_default_chain'
};
```

---

## 3. 多链支持与 Etherscan 集成

### 3.1 支持的区块链

| Chain ID | 链 | 浏览器 API 端点 |
|----------|------|-----------------|
| 1 | Ethereum | api.etherscan.io |
| 56 | BSC | api.bscscan.com |
| 137 | Polygon | api.polygonscan.com |
| 42161 | Arbitrum | api.arbiscan.io |
| 8453 | Base | api.basescan.org |
| 43114 | Avalanche | api.snowtrace.io |

CHAIN_CONFIG 使用数字 Chain ID 作为 Key，与 UI 和网络协议一致。

### 3.2 合约源码获取（双模式）

**自动拉取模式**：
1. 用户选择链 + 输入合约地址 + 可选 Etherscan API Key
2. 调用 Etherscan `getsourcecode` API
3. 解析返回格式：
   - 单文件合约：直接使用 `SourceCode` 字符串
   - 多文件合约（Standard JSON Input）：`{{...}}` 格式，解析后提取 `parsed.sources[path].content`

**手动粘贴模式**：
- 用户直接粘贴 Solidity 源码
- 支持多文件，用 `=== File: xxx.sol ===` 分隔
- 支持 "+ 添加更多文件" 按钮

### 3.3 import 解析（安全版）

```javascript
function parseImports(source) {
  const cleanSource = source
    .replace(/\/\/.*/g, '')           // 移除单行注释
    .replace(/\/\*[\s\S]*?\*\//g, ''); // 移除多行注释
  // 然后用正则提取 import 路径
}
```

必须先剔除注释，否则注释中的 import 语句会污染依赖图。

### 3.4 拓扑排序（三色标记法）

```javascript
function topologicalSort(files) {
  const order = [];
  const state = {}; // 0=未访问, 1=正在访问, 2=已访问

  function visit(file) {
    if (state[file] === 1) throw new Error(`检测到循环依赖: ${file}`);
    if (state[file] === 2) return;
    state[file] = 1;
    for (const imp of parseImports(files[file] || '')) {
      const resolved = resolveImportPath(imp, Object.keys(files));
      if (resolved) visit(resolved);
    }
    state[file] = 2;
    order.push(file);
  }

  for (const file in files) {
    if (!state[file]) visit(file);
  }
  return order;
}
```

检测到循环依赖时立即抛出清晰错误，不会卡死浏览器。

### 3.5 Token 估算（Solidity 专用）

```javascript
function estimateTokens(text) {
  return Math.ceil(text.length / 2.5); // Solidity 高密字符特征，1 token ≈ 2.5 字符
}
```

超过 900K token 时警告用户分析可能不完整。

---

## 4. AI Prompt 工程

### 4.1 System Prompt（字符级锁死，永不变化）

System Prompt 不含任何动态内容，确保小米 Prompt Cache 100% 命中（300 Credits → 2.5 Credits，120 倍成本差距）。

内容包含：
- 角色定义：专业智能合约安全审计师
- 思维链强制指令：在输出 JSON 前，必须对合约的每一行代码、每一个外部调用路径、每一个修饰符进行最少 3 轮对抗性攻击模拟
- 知识截止日期（固定值）
- 输出 JSON 格式定义
- 评分公式约束
- 禁止事项

### 4.2 输出格式（严格 JSON）

```json
{
  "risk_score": "0-100 综合风险评分",
  "risk_level": "CRITICAL|HIGH|MEDIUM|LOW|SAFE",
  "summary": "一句话结论",
  "red_flags": [{
    "title": "风险标题",
    "severity": "CRITICAL|HIGH|MEDIUM|LOW",
    "description": "详细说明",
    "code_snippet": "相关代码片段",
    "recommendation": "修复建议"
  }],
  "green_passes": [{
    "title": "安全项标题",
    "description": "说明"
  }],
  "category_scores": {
    "honeypot_risk": "0-100 蜜罐风险",
    "rugpull_risk": "0-100 Rugpull风险",
    "owner_privilege": "0-100 所有者权限",
    "liquidity_risk": "0-100 流动性风险",
    "code_quality": "0-100 代码质量"
  }
}
```

**注意**：JSON 中不包含 `audit_report_markdown` 字段。审计报告由前端利用 marked.js 将 `red_flags` + `green_passes` + `summary` 动态拼装渲染，既保证 100% JSON 解析成功率，又减轻 AI 输出 Token 负担。

### 4.3 评分公式约束

System Prompt 中硬编码数学公式约束 5 维评分与发现的一致性：
- `honeypot_risk`: 发现蜜罐逻辑 → 强制 100 分
- `owner_privilege`: 发现隐藏所有权/无限增发/动态黑名单 → 强制 ≥80 分
- 5 维评分必须与 red_flags 中的发现严格一致，禁止自相矛盾

### 4.4 User Message（动态内容）

```
请审计以下智能合约源码：

=== 文件: Contract.sol (主合约) ===
[source code]

=== 文件: Lib.sol (依赖) ===
[source code]
```

### 4.5 MiMo 思维链回传协议

MiMo-V2.5-Pro 在思考模式下会产生 `reasoning_content` 字段。多轮对话时必须完整回传上一轮的 `reasoning_content`，否则接口返回 400。

```javascript
messages.push({
  role: "assistant",
  content: apiResponse.content,
  reasoning_content: apiResponse.reasoning_content || ""
});
```

---

## 5. ECharts 可视化与异常熔断

### 5.1 三张图表

| 图表 | 数据源 | 作用 |
|------|--------|------|
| 雷达图 (Radar) | `category_scores` 5 维 | 各维度风险分布 |
| 仪表盘 (Gauge) | `risk_score` 0-100 | 综合评分，红黄绿三色 |
| 红绿卡片 (DOM) | `red_flags` + `green_passes` | 风险项+安全项，代码 hljs 高亮 |

### 5.2 ECharts 安全初始化（防内存泄漏）

```javascript
function initCharts() {
  try {
    ['radar-chart', 'gauge-chart'].forEach(id => {
      const dom = document.getElementById(id);
      if (echarts.getInstanceByDom(dom)) echarts.dispose(dom);
    });
    return {
      radar: echarts.init(document.getElementById('radar-chart'), 'dark'),
      gauge: echarts.init(document.getElementById('gauge-chart'), 'dark')
    };
  } catch (e) {
    console.error('ECharts 初始化失败，降级为纯文本模式', e);
    return null;
  }
}
```

### 5.3 Resize 防抖

```javascript
function debounce(fn, delay = 150) {
  let timer;
  return (...args) => { clearTimeout(timer); timer = setTimeout(() => fn(...args), delay); };
}
window.addEventListener('resize', debounce(() => {
  charts?.radar?.resize();
  charts?.gauge?.resize();
}));
```

### 5.4 Marked + Highlight.js 全局钩子

```javascript
function configureMarked() {
  marked.use({
    renderer: {
      code(code, infostring) {
        const lang = infostring || 'solidity';
        let highlighted;
        try {
          highlighted = hljs.getLanguage(lang)
            ? hljs.highlight(code, { language: lang }).value
            : hljs.highlightAuto(code).value;
        } catch (e) { highlighted = code; }
        return `<pre class="hljs"><code class="language-${lang}">${highlighted}</code></pre>`;
      }
    }
  });
}
```

### 5.5 三层熔断链

```
Layer 1: JSON.parse(aiResponse)
  ├─ 成功 → Layer 2: 校验必要字段 (risk_score, category_scores, red_flags)
  │   ├─ 完整 → 正常渲染（雷达图 + 仪表盘 + 红绿卡片 + 拼装报告）
  │   └─ 缺失 → 填充默认值 + 黄色警告条提示
  └─ 失败 → Layer 3: 纯 Markdown 降级
      ├─ marked.parse(aiResponse) 渲染为 HTML（带 hljs 高亮）
      ├─ 雷达图: FALLBACK_RADAR_OPTION（灰色虚线 50 分中性）
      ├─ 仪表盘: FALLBACK_GAUGE_OPTION（灰色 "?" 标识）
      └─ 黄条: "⚠️ AI 返回格式异常，以下为原始分析报告"
```

### 5.6 降级默认图表配置

```javascript
const FALLBACK_RADAR_OPTION = {
  title: { text: '⚠️ 风险评分不可用', left: 'center' },
  radar: {
    indicator: [
      { name: '蜜罐风险', max: 100 }, { name: 'Rugpull', max: 100 },
      { name: '所有者权限', max: 100 }, { name: '流动性风险', max: 100 },
      { name: '代码质量', max: 100 }
    ]
  },
  series: [{
    type: 'radar',
    data: [{ value: [50, 50, 50, 50, 50], name: '数据不可用' }],
    areaStyle: { opacity: 0.3 },
    lineStyle: { type: 'dashed', color: '#999' }
  }]
};

const FALLBACK_GAUGE_OPTION = {
  title: { text: '综合评分不可用', left: 'center', bottom: '10%',
           textStyle: { color: '#94a3b8', fontSize: 14 } },
  series: [{
    type: 'gauge', min: 0, max: 100, radius: '80%',
    axisLine: { lineStyle: { width: 20, color: [[1, '#475569']] } },
    pointer: { show: true, itemStyle: { color: '#64748b' } },
    axisTick: { show: false }, splitLine: { show: false },
    axisLabel: { show: false },
    detail: { formatter: '?', color: '#94a3b8', offsetCenter: [0, '-10%'], fontSize: 32 },
    data: [{ value: 50 }]
  }]
};
```

---

## 6. 状态管理与错误处理

### 6.1 应用状态对象

```javascript
const AppState = {
  config: {
    mimoApiKey: '',          // tp- 开头
    etherscanKey: '',        // 可选
    selectedChain: 1,        // 默认 Ethereum
  },
  scan: {
    status: 'idle',          // idle | fetching | parsing | analyzing | done | error
    progress: '',            // 当前步骤描述
    contractFiles: {},       // { "path.sol": "source..." }
    mergedSource: '',        // 合并后的完整源码
    aiResponse: null,        // AI 原始文本
    parsedResult: null,      // JSON.parse 后的结构化数据
    error: null,
    renderMode: 'json'       // json | markdown
  },
  charts: null
};
```

### 6.2 状态机

```
idle → fetching → parsing → analyzing → done
  │       │          │          │         │
  └───────┴──────────┴──────────┴─────────┴→ error
```

### 6.3 fetchWithRetry（指数退避 + 错误区分）

```javascript
async function fetchWithRetry(url, options = {}, retries = 3, delay = 1000) {
  try {
    const response = await fetch(url, options);
    if (response.status === 429 && retries > 0) {
      console.warn(`触发限流(429)，${delay}ms 后重试...`);
      await new Promise(r => setTimeout(r, delay));
      return fetchWithRetry(url, options, retries - 1, delay * 2);
    }
    return response;
  } catch (error) {
    if (error instanceof TypeError && error.message === 'Failed to fetch') {
      throw new Error('网络连接失败，请检查网络状态、小米 API 网关地址或代理软件');
    }
    if (retries > 0) {
      await new Promise(r => setTimeout(r, delay));
      return fetchWithRetry(url, options, retries - 1, delay * 2);
    }
    throw error;
  }
}
```

### 6.4 前端报告拼装

```javascript
function assembleReport(data) {
  let md = `# 🔍 智能合约安全审计报告\n\n`;
  md += `## 综合评分: ${data.risk_score}/100 (${data.risk_level})\n\n`;
  md += `> ${data.summary}\n\n`;
  md += `## 🚩 红牌警告 (${data.red_flags.length})\n`;
  data.red_flags.forEach(f => {
    md += `### ${f.title}\n**严重性**: ${f.severity}\n${f.description}\n`;
    md += `\`\`\`solidity\n${f.code_snippet}\n\`\`\`\n`;
    md += `**修复建议**: ${f.recommendation}\n\n`;
  });
  md += `## ✅ 绿牌通过 (${data.green_passes.length})\n`;
  data.green_passes.forEach(p => { md += `### ${p.title}\n${p.description}\n\n`; });
  return md;
}
```

---

## 7. UI 设计

### 7.1 布局结构

- 暗色主题（Tailwind dark 类名）
- 顶部：标题 + API Key 配置入口
- 中部：合约输入区（Tab 切换：自动拉取 / 手动粘贴）+ 链选择器 + 扫描按钮
- 下部：结果区（左：ECharts 双图表，右：红绿卡片墙，下方：拼装的 Markdown 报告）
- 页脚：安全警告 + 免责声明

### 7.2 自动拉取模式 UI

```
┌─────────────────────────────────────────────────┐
│  [ 自动拉取 ▼ ]  [ 手动粘贴 ]     ← Tab 切换   │
├─────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌──────────────────────────┐ │
│  │ 选择链 ▼    │  │ 合约地址 0x...           │ │
│  └─────────────┘  └──────────────────────────┘ │
│  ┌──────────────────────────────────────────┐  │
│  │ Etherscan API Key (可选)                 │  │
│  └──────────────────────────────────────────┘  │
└─────────────────────────────────────────────────┘
```

### 7.3 手动粘贴模式 UI

```
┌─────────────────────────────────────────────────┐
│  ┌──────────────────────────────────────────┐  │
│  │ 粘贴 Solidity 源码...                    │  │
│  │ (支持多文件，用 === File: xxx.sol ===     │  │
│  │  分隔)                                   │  │
│  └──────────────────────────────────────────┘  │
│  [ + 添加更多文件 ]                              │
└─────────────────────────────────────────────────┘
```

---

## 8. 设计决策汇总

| 维度 | 决策 |
|------|------|
| 架构 | 单 HTML 纯前端，双击即用 |
| API 调用 | 前端直连小米 API，无 CORS 代理 |
| Key 存储 | localStorage，含安全警告 |
| 合约获取 | 混合模式（Etherscan 自动 + 手动粘贴） |
| 上下文组装 | 前端拓扑排序合并为单一源码 |
| 缓存优化 | System Prompt 字符级锁死，120 倍成本节省 |
| 结果渲染 | JSON → ECharts，失败降级 Markdown + 默认图表 |
| Markdown | marked.js + highlight.js + highlightjs-solidity |
| 429 处理 | fetchWithRetry 指数退避 + 按钮防抖 |
| 循环依赖 | 三色标记法 DFS，检测到立即抛错 |
| Token 估算 | Solidity 专用：1 token ≈ 2.5 字符 |
