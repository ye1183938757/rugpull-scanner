# 多链 Rugpull 与蜜罐风险扫描仪 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 构建一个单 HTML 纯前端的多链 Rugpull 与蜜罐风险扫描仪，调用 mimo-v2.5-pro 模型分析 Solidity 合约安全。

**Architecture:** 单一 `rugpull-scanner.html` 文件，内含 6 个 IIFE 隔离的功能模块。前端直连小米 API，Etherscan API 自动拉取合约源码，拓扑排序合并后发送给 AI 分析，ECharts 渲染风险雷达图和仪表盘，三层熔断链保证异常时不崩溃。

**Tech Stack:** Tailwind CSS (Play CDN), ECharts 5.5, marked.js, highlight.js + highlightjs-solidity, mimo-v2.5-pro API

**Spec:** `docs/superpowers/specs/2026-06-15-rugpull-scanner-design.md`

---

## File Structure

本项目仅创建一个文件：

| 文件 | 职责 |
|------|------|
| `rugpull-scanner.html` | 完整应用：HTML 结构 + CSS 样式 + 6 个 JS 模块 |

---

### Task 1: HTML 骨架 + CDN 依赖 + Tailwind 暗色主题布局

**Files:**
- Create: `rugpull-scanner.html`

- [ ] **Step 1: 创建 HTML 文件，写入 CDN 依赖和基础骨架**

```html
<!DOCTYPE html>
<html lang="zh-CN" class="dark">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>多链 Rugpull 与蜜罐风险扫描仪</title>

  <!-- highlight.js 暗黑主题 -->
  <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/highlight.js@11.9.0/styles/github-dark.min.css">

  <!-- Tailwind CSS Play CDN（运行时动态生成 CSS，非 link 标签） -->
  <script src="https://cdn.tailwindcss.com"></script>
  <script>
    tailwind.config = {
      darkMode: 'class',
      theme: {
        extend: {
          colors: {
            surface: { DEFAULT: '#0f172a', 50: '#1e293b', 100: '#334155' }
          }
        }
      }
    }
  </script>

  <!-- ECharts -->
  <script src="https://cdn.jsdelivr.net/npm/echarts@5.5.0/dist/echarts.min.js"></script>
  <!-- marked.js (Markdown 渲染) -->
  <script src="https://cdn.jsdelivr.net/npm/marked/marked.min.js"></script>
  <!-- highlight.js core + Solidity 语法 -->
  <script src="https://cdn.jsdelivr.net/npm/highlight.js@11.9.0/lib/core.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/highlightjs-solidity@2.0.6/dist/solidity.min.js"></script>

  <style>
    body { background-color: #0f172a; color: #e2e8f0; font-family: 'Segoe UI', system-ui, sans-serif; }
    .card { background-color: #1e293b; border: 1px solid #334155; border-radius: 0.75rem; padding: 1.5rem; }
    .card-red { border-left: 4px solid #ef4444; }
    .card-green { border-left: 4px solid #22c55e; }
    #results-panel { display: none; }
    .progress-bar { transition: width 0.3s ease; }
    .btn-primary { background: linear-gradient(135deg, #3b82f6, #8b5cf6); }
    .btn-primary:hover { background: linear-gradient(135deg, #2563eb, #7c3aed); }
    .btn-primary:disabled { opacity: 0.5; cursor: not-allowed; }
    .tab-active { border-bottom: 2px solid #3b82f6; color: #3b82f6; }
    .tab-inactive { border-bottom: 2px solid transparent; color: #94a3b8; }
    .tab-inactive:hover { color: #e2e8f0; }
    /* Markdown 报告样式 */
    .report-content h1 { font-size: 1.5rem; font-weight: 700; margin: 1rem 0; }
    .report-content h2 { font-size: 1.25rem; font-weight: 600; margin: 0.75rem 0; color: #93c5fd; }
    .report-content h3 { font-size: 1.1rem; font-weight: 600; margin: 0.5rem 0; }
    .report-content blockquote { border-left: 3px solid #3b82f6; padding-left: 1rem; color: #94a3b8; margin: 0.5rem 0; }
    .report-content pre { border-radius: 0.5rem; padding: 1rem; overflow-x: auto; margin: 0.5rem 0; }
    .report-content ul, .report-content ol { padding-left: 1.5rem; margin: 0.5rem 0; }
    .report-content li { margin: 0.25rem 0; }
    .report-content p { margin: 0.5rem 0; line-height: 1.7; }
    .report-content strong { color: #fbbf24; }
    /* 安全警告 */
    .security-warning { border: 1px solid rgba(239, 68, 68, 0.3); background: rgba(239, 68, 68, 0.05); }
  </style>
</head>
<body class="min-h-screen">
  <!-- ===== HEADER ===== -->
  <header class="border-b border-gray-800 px-6 py-4">
    <div class="max-w-7xl mx-auto flex items-center justify-between">
      <div>
        <h1 class="text-2xl font-bold bg-gradient-to-r from-blue-400 to-purple-500 bg-clip-text text-transparent">
          🔍 多链 Rugpull 与蜜罐风险扫描仪
        </h1>
        <p class="text-sm text-gray-500 mt-1">Powered by MiMo-V2.5-Pro · 支持 ETH / BSC / Polygon / Arbitrum / Base / Avalanche</p>
      </div>
      <button id="btn-settings" class="text-gray-400 hover:text-white transition" title="API Key 设置">
        <svg class="w-6 h-6" fill="none" stroke="currentColor" viewBox="0 0 24 24">
          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M10.325 4.317c.426-1.756 2.924-1.756 3.35 0a1.724 1.724 0 002.573 1.066c1.543-.94 3.31.826 2.37 2.37a1.724 1.724 0 001.066 2.573c1.756.426 1.756 2.924 0 3.35a1.724 1.724 0 00-1.066 2.573c.94 1.543-.826 3.31-2.37 2.37a1.724 1.724 0 00-2.573 1.066c-.426 1.756-2.924 1.756-3.35 0a1.724 1.724 0 00-2.573-1.066c-1.543.94-3.31-.826-2.37-2.37a1.724 1.724 0 00-1.066-2.573c-1.756-.426-1.756-2.924 0-3.35a1.724 1.724 0 001.066-2.573c-.94-1.543.826-3.31 2.37-2.37.996.608 2.296.07 2.572-1.065z"/>
          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15 12a3 3 0 11-6 0 3 3 0 016 0z"/>
        </svg>
      </button>
    </div>
  </header>

  <!-- ===== SETTINGS MODAL ===== -->
  <div id="settings-modal" class="fixed inset-0 bg-black/60 backdrop-blur-sm z-50 hidden flex items-center justify-center">
    <div class="card w-full max-w-md mx-4">
      <h2 class="text-lg font-semibold mb-4">⚙️ API Key 配置</h2>
      <div class="space-y-4">
        <div>
          <label class="block text-sm text-gray-400 mb-1">小米 MiMo API Key <span class="text-red-400">*</span></label>
          <input id="input-mimo-key" type="password" placeholder="tp-..." class="w-full bg-surface-50 border border-gray-700 rounded-lg px-3 py-2 text-sm focus:outline-none focus:border-blue-500">
        </div>
        <div>
          <label class="block text-sm text-gray-400 mb-1">Etherscan API Key <span class="text-gray-600">(可选)</span></label>
          <input id="input-etherscan-key" type="password" placeholder="免费注册: etherscan.io/apis" class="w-full bg-surface-50 border border-gray-700 rounded-lg px-3 py-2 text-sm focus:outline-none focus:border-blue-500">
        </div>
        <div class="security-warning rounded-lg p-3 text-sm text-red-400">
          ⚠️ 安全警告：API Key 存储在本地浏览器中，请仅在本地双击运行、localhost 或您完全私有的独立域名下使用。切勿在公共多用户共享域名下运行，以防您的 Credit 资产被 XSS 脚本窃取！
        </div>
      </div>
      <div class="flex justify-end gap-3 mt-6">
        <button id="btn-settings-cancel" class="px-4 py-2 text-sm text-gray-400 hover:text-white transition">取消</button>
        <button id="btn-settings-save" class="btn-primary px-4 py-2 text-sm text-white rounded-lg">保存</button>
      </div>
    </div>
  </div>

  <!-- ===== MAIN CONTENT ===== -->
  <main class="max-w-7xl mx-auto px-6 py-8">

    <!-- Input Panel -->
    <div class="card mb-6">
      <!-- Tabs -->
      <div class="flex gap-6 mb-4 border-b border-gray-700 pb-2">
        <button class="tab-active pb-2 text-sm font-medium" data-tab="auto">🔗 自动拉取</button>
        <button class="tab-inactive pb-2 text-sm font-medium" data-tab="manual">📋 手动粘贴</button>
      </div>

      <!-- Auto Fetch Tab -->
      <div id="tab-auto">
        <div class="flex gap-4 mb-4">
          <select id="select-chain" class="bg-surface-50 border border-gray-700 rounded-lg px-3 py-2 text-sm focus:outline-none focus:border-blue-500">
            <option value="1">Ethereum</option>
            <option value="56">BSC</option>
            <option value="137">Polygon</option>
            <option value="42161">Arbitrum</option>
            <option value="8453">Base</option>
            <option value="43114">Avalanche</option>
          </select>
          <input id="input-address" type="text" placeholder="输入合约地址 0x..." class="flex-1 bg-surface-50 border border-gray-700 rounded-lg px-3 py-2 text-sm focus:outline-none focus:border-blue-500">
        </div>
      </div>

      <!-- Manual Paste Tab -->
      <div id="tab-manual" class="hidden">
        <textarea id="input-source" rows="12" placeholder="粘贴 Solidity 源码...&#10;&#10;支持多文件，用 === File: xxx.sol === 分隔" class="w-full bg-surface-50 border border-gray-700 rounded-lg px-3 py-2 text-sm font-mono focus:outline-none focus:border-blue-500 resize-y"></textarea>
        <button id="btn-add-file" class="mt-2 text-sm text-blue-400 hover:text-blue-300 transition">+ 添加更多文件</button>
      </div>

      <!-- Scan Button -->
      <div class="mt-4 flex items-center gap-4">
        <button id="btn-scan" class="btn-primary px-6 py-2.5 text-sm font-medium text-white rounded-lg">
          🚀 开始扫描
        </button>
        <div id="scan-progress" class="flex-1 hidden">
          <div class="flex items-center gap-3">
            <div class="flex-1 bg-gray-800 rounded-full h-2 overflow-hidden">
              <div id="progress-bar" class="progress-bar h-full bg-gradient-to-r from-blue-500 to-purple-500 rounded-full" style="width: 0%"></div>
            </div>
            <span id="progress-text" class="text-xs text-gray-400 whitespace-nowrap"></span>
          </div>
        </div>
      </div>
    </div>

    <!-- Results Panel -->
    <div id="results-panel">
      <!-- Warning Banner (shown on fallback) -->
      <div id="warning-banner" class="hidden bg-yellow-500/10 border border-yellow-500/30 rounded-lg px-4 py-3 mb-6 text-sm text-yellow-400">
        ⚠️ <span id="warning-text"></span>
      </div>

      <!-- Charts Row -->
      <div class="grid grid-cols-1 lg:grid-cols-2 gap-6 mb-6">
        <div class="card">
          <div id="radar-chart" style="width: 100%; height: 350px;"></div>
        </div>
        <div class="card">
          <div id="gauge-chart" style="width: 100%; height: 350px;"></div>
        </div>
      </div>

      <!-- Red/Green Cards -->
      <div id="cards-container" class="grid grid-cols-1 md:grid-cols-2 gap-4 mb-6"></div>

      <!-- Markdown Report -->
      <div class="card">
        <h2 class="text-lg font-semibold mb-4">📄 详细审计报告</h2>
        <div id="report-content" class="report-content prose prose-invert max-w-none"></div>
      </div>
    </div>
  </main>

  <!-- ===== FOOTER ===== -->
  <footer class="border-t border-gray-800 px-6 py-6 mt-12">
    <div class="max-w-7xl mx-auto">
      <div class="security-warning rounded-lg p-3 text-sm text-red-400 mb-4">
        ⚠️ 安全警告：API Key 存储在本地浏览器中，请仅在本地双击运行、localhost 或您完全私有的独立域名下使用。切勿在公共多用户共享域名下运行，以防您的 Credit 资产被 XSS 脚本窃取！
      </div>
      <p class="text-xs text-gray-600 text-center">
        免责声明：本工具仅供参考，不构成任何投资建议。智能合约审计结果可能存在误判，请结合专业安全团队的人工审计。
      </p>
    </div>
  </footer>

  <!-- ===== APPLICATION LOGIC ===== -->
  <script>
  // 模块代码将在后续 Task 中逐步填入
  </script>
</body>
</html>
```

- [ ] **Step 2: 在浏览器中双击打开 `rugpull-scanner.html`，验证页面正常渲染**

Expected: 页面显示暗色主题，Header 标题可见，输入面板显示两个 Tab（自动拉取/手动粘贴），齿轮图标可点击。

- [ ] **Step 3: 验证 CDN 资源加载**

在浏览器 F12 Console 中运行：

```javascript
console.log('Tailwind:', typeof tailwind !== 'undefined');
console.log('ECharts:', typeof echarts !== 'undefined');
console.log('marked:', typeof marked !== 'undefined');
console.log('hljs:', typeof hljs !== 'undefined');
```

Expected: 全部输出 `true`。

- [ ] **Step 4: Commit**

```bash
git add rugpull-scanner.html
git commit -m "feat: create HTML skeleton with CDN dependencies and dark theme layout"
```

---

### Task 2: Config & Storage 模块（localStorage 持久化）

**Files:**
- Modify: `rugpull-scanner.html` (在 `<script>` 标签内)

- [ ] **Step 1: 在 `<script>` 标签内写入 Config & Storage 模块**

将 `<script>` 标签中的占位注释替换为以下代码：

```javascript
(function() {
  'use strict';

  // ===== [1] Config & Constants =====
  const CONFIG = {
    MIMO_API_BASE: 'https://token-plan-cn.xiaomimimo.com/v1',
    MIMO_MODEL: 'mimo-v2.5-pro',
    MAX_RETRIES: 3,
    RETRY_DELAY: 1000,
    TOKEN_WARN_THRESHOLD: 900000
  };

  const CHAIN_CONFIG = {
    1:     { host: 'api.etherscan.io',     name: 'Ethereum' },
    56:    { host: 'api.bscscan.com',      name: 'BSC' },
    137:   { host: 'api.polygonscan.com',  name: 'Polygon' },
    42161: { host: 'api.arbiscan.io',      name: 'Arbitrum' },
    8453:  { host: 'api.basescan.org',     name: 'Base' },
    43114: { host: 'api.snowtrace.io',     name: 'Avalanche' }
  };

  const STORAGE_KEYS = {
    MIMO_KEY: 'rugscan_mimo_key',
    ETHERSCAN_KEY: 'rugscan_etherscan_key',
    CHAIN: 'rugscan_default_chain'
  };

  // ===== [2] Storage Module =====
  const Storage = {
    save(key, value) {
      try { localStorage.setItem(key, value); }
      catch (e) { console.warn('localStorage 写入失败:', e); }
    },
    load(key, fallback = '') {
      try { return localStorage.getItem(key) || fallback; }
      catch (e) { return fallback; }
    },
    saveConfig(config) {
      this.save(STORAGE_KEYS.MIMO_KEY, config.mimoApiKey);
      this.save(STORAGE_KEYS.ETHERSCAN_KEY, config.etherscanKey);
      this.save(STORAGE_KEYS.CHAIN, String(config.selectedChain));
    },
    loadConfig() {
      return {
        mimoApiKey: this.load(STORAGE_KEYS.MIMO_KEY),
        etherscanKey: this.load(STORAGE_KEYS.ETHERSCAN_KEY),
        selectedChain: parseInt(this.load(STORAGE_KEYS.CHAIN, '1'))
      };
    }
  };

  // ===== App State =====
  const AppState = {
    config: Storage.loadConfig(),
    scan: {
      status: 'idle',
      progress: '',
      contractFiles: {},
      mergedSource: '',
      aiResponse: null,
      parsedResult: null,
      error: null,
      renderMode: 'json'
    },
    charts: null
  };

  // 暴露到全局供后续模块使用
  window.__SCANNER = { CONFIG, CHAIN_CONFIG, STORAGE_KEYS, Storage, AppState };
})();
```

- [ ] **Step 2: 在浏览器 Console 中验证 Storage 模块**

```javascript
const { Storage, AppState } = window.__SCANNER;
// 测试保存和读取
Storage.save('rugscan_test', 'hello');
console.log('存取测试:', Storage.load('rugscan_test') === 'hello' ? 'PASS' : 'FAIL');
// 测试 config 加载
console.log('Config:', JSON.stringify(AppState.config));
```

Expected: 存取测试输出 `PASS`，Config 输出包含 `mimoApiKey`、`etherscanKey`、`selectedChain` 字段。

- [ ] **Step 3: 验证设置弹窗的打开/关闭逻辑**

在 `<script>` 标签末尾（`window.__SCANNER` 之后，`})();` 之前）追加：

```javascript
  // ===== Settings Modal =====
  const settingsModal = document.getElementById('settings-modal');
  const btnSettings = document.getElementById('btn-settings');
  const btnCancel = document.getElementById('btn-settings-cancel');
  const btnSave = document.getElementById('btn-settings-save');
  const inputMimoKey = document.getElementById('input-mimo-key');
  const inputEtherscanKey = document.getElementById('input-etherscan-key');

  btnSettings.addEventListener('click', () => {
    inputMimoKey.value = AppState.config.mimoApiKey;
    inputEtherscanKey.value = AppState.config.etherscanKey;
    settingsModal.classList.remove('hidden');
  });

  btnCancel.addEventListener('click', () => {
    settingsModal.classList.add('hidden');
  });

  btnSave.addEventListener('click', () => {
    AppState.config.mimoApiKey = inputMimoKey.value.trim();
    AppState.config.etherscanKey = inputEtherscanKey.value.trim();
    Storage.saveConfig(AppState.config);
    settingsModal.classList.add('hidden');
  });

  // 点击背景关闭
  settingsModal.addEventListener('click', (e) => {
    if (e.target === settingsModal) settingsModal.classList.add('hidden');
  });
```

在浏览器中验证：点击齿轮图标 → 弹窗出现 → 输入测试 Key → 点击保存 → 弹窗关闭 → 刷新页面 → 再次打开弹窗 → Key 仍在。

- [ ] **Step 4: Commit**

```bash
git add rugpull-scanner.html
git commit -m "feat: add config/storage module with localStorage persistence"
```

---

### Task 3: API Client 模块（fetchWithRetry + 小米 API + Etherscan）

**Files:**
- Modify: `rugpull-scanner.html` (IIFE 内部，Storage 模块之后)

- [ ] **Step 1: 在 IIFE 内部追加 API Client 模块**

在 `window.__SCANNER = { ... };` 之后、`})();` 之前追加：

```javascript
  // ===== [3] API Client =====
  const API = {
    // 带指数退避的 fetch，区分 429 限流与网络/CORS 错误
    async fetchWithRetry(url, options = {}, retries = CONFIG.MAX_RETRIES, delay = CONFIG.RETRY_DELAY) {
      try {
        const response = await fetch(url, options);
        if (response.status === 429 && retries > 0) {
          console.warn(`触发限流(429)，${delay}ms 后重试...`);
          await new Promise(r => setTimeout(r, delay));
          return this.fetchWithRetry(url, options, retries - 1, delay * 2);
        }
        return response;
      } catch (error) {
        if (error instanceof TypeError && error.message === 'Failed to fetch') {
          throw new Error('网络连接失败，请检查网络状态、小米 API 网关地址或代理软件');
        }
        if (retries > 0) {
          await new Promise(r => setTimeout(r, delay));
          return this.fetchWithRetry(url, options, retries - 1, delay * 2);
        }
        throw error;
      }
    },

    // 调用小米 MiMo API
    async callMiMo(messages) {
      const resp = await this.fetchWithRetry(
        `${CONFIG.MIMO_API_BASE}/chat/completions`,
        {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            'api-key': AppState.config.mimoApiKey
          },
          body: JSON.stringify({
            model: CONFIG.MIMO_MODEL,
            messages: messages,
            temperature: 0.1,
            max_tokens: 16384
          })
        }
      );
      if (!resp.ok) {
        const errText = await resp.text().catch(() => '');
        throw new Error(`MiMo API 请求失败 (${resp.status}): ${errText}`);
      }
      const data = await resp.json();
      return {
        content: data.choices?.[0]?.message?.content || '',
        reasoning_content: data.choices?.[0]?.message?.reasoning_content || ''
      };
    },

    // 通过 Etherscan 获取合约源码
    async fetchContractSource(chainId, address) {
      const chain = CHAIN_CONFIG[chainId];
      if (!chain) throw new Error(`不支持的链: ${chainId}`);
      const etherscanKey = AppState.config.etherscanKey;
      if (!etherscanKey) {
        UI.updateProgress('正在尝试匿名提取源码...（提示：匿名提取极易限流，如报错请配置免费 Etherscan API Key）');
      }
      const url = `https://${chain.host}/api?module=contract&action=getsourcecode`
                + `&address=${address}&apikey=${etherscanKey || ''}`;
      const resp = await this.fetchWithRetry(url);
      const json = await resp.json();
      if (json.status !== '1' || !json.result?.[0]?.SourceCode) {
        throw new Error(`获取合约源码失败: ${json.message || json.result || '未知错误'}`);
      }
      const source = json.result[0].SourceCode;
      // 多文件合约：Standard JSON Input 格式 {{...}}
      if (source.startsWith('{{') && source.endsWith('}}')) {
        const parsed = JSON.parse(source.slice(1, -1));
        const files = {};
        if (parsed.sources) {
          for (const [filePath, fileObj] of Object.entries(parsed.sources)) {
            files[filePath] = fileObj.content || '';
          }
        }
        return files;
      }
      // 单文件合约
      return { 'Contract.sol': source };
    }
  };

  window.__SCANNER.API = API;
```

- [ ] **Step 2: 在浏览器 Console 中验证 fetchWithRetry 存在**

```javascript
console.log('API module:', typeof window.__SCANNER.API.fetchWithRetry);
console.log('callMiMo:', typeof window.__SCANNER.API.callMiMo);
console.log('fetchContractSource:', typeof window.__SCANNER.API.fetchContractSource);
```

Expected: 全部输出 `function`。

- [ ] **Step 3: Commit**

```bash
git add rugpull-scanner.html
git commit -m "feat: add API client with fetchWithRetry, Xiaomi API, and Etherscan integration"
```

---

### Task 4: Contract Parser 模块（import 解析 + 三色拓扑排序 + 合并）

**Files:**
- Modify: `rugpull-scanner.html` (IIFE 内部，API 模块之后)

- [ ] **Step 1: 在 IIFE 内部追加 Contract Parser 模块**

```javascript
  // ===== [4] Contract Parser =====
  const Parser = {
    // 从 Solidity 源码中提取 import 路径（安全过滤注释版）
    parseImports(source) {
      const cleanSource = source
        .replace(/\/\/.*/g, '')            // 移除单行注释
        .replace(/\/\*[\s\S]*?\*\//g, ''); // 移除多行注释
      const imports = [];
      const regex = /import\s+(?:{[^}]+}\s+from\s+)?["']([^"']+)["']/g;
      let match;
      while ((match = regex.exec(cleanSource)) !== null) {
        imports.push(match[1]);
      }
      return imports;
    },

    // 解析 import 路径为 files 字典中的 key
    resolveImportPath(importPath, fileKeys) {
      const fileName = importPath.split('/').pop();
      return fileKeys.find(k => k.endsWith('/' + fileName) || k === fileName);
    },

    // 三色标记法拓扑排序（0=未访问, 1=正在访问, 2=已访问）
    topologicalSort(files) {
      const order = [];
      const state = {};
      const fileKeys = Object.keys(files);

      const visit = (file) => {
        if (state[file] === 1) {
          throw new Error(`检测到 Solidity 循环依赖：${file} 存在循环导入，请检查合约源码！`);
        }
        if (state[file] === 2) return;
        state[file] = 1;
        for (const imp of this.parseImports(files[file] || '')) {
          const resolved = this.resolveImportPath(imp, fileKeys);
          if (resolved) visit(resolved);
        }
        state[file] = 2;
        order.push(file);
      };

      for (const file of fileKeys) {
        if (!state[file]) visit(file);
      }
      return order;
    },

    // 合并为单一源码字符串
    mergeFiles(files) {
      const order = this.topologicalSort(files);
      let merged = '';
      for (const filePath of order) {
        merged += `\n// ===== File: ${filePath} =====\n`;
        merged += files[filePath] + '\n';
      }
      return merged.trim();
    },

    // Token 估算（Solidity 专用：1 token ≈ 2.5 字符）
    estimateTokens(text) {
      return Math.ceil(text.length / 2.5);
    },

    // 解析手动粘贴的多文件输入
    parseManualInput(text) {
      const files = {};
      const regex = /===\s*File:\s*(.+?)\s*===/g;
      const parts = text.split(regex);
      if (parts.length <= 1) {
        // 没有分隔符，视为单文件
        return { 'Contract.sol': text };
      }
      // parts: [前置内容, filename1, content1, filename2, content2, ...]
      for (let i = 1; i < parts.length; i += 2) {
        const fileName = parts[i].trim();
        const content = (parts[i + 1] || '').trim();
        if (fileName && content) {
          files[fileName] = content;
        }
      }
      return Object.keys(files).length > 0 ? files : { 'Contract.sol': text };
    }
  };

  window.__SCANNER.Parser = Parser;
```

- [ ] **Step 2: 在浏览器 Console 中验证拓扑排序和循环依赖检测**

```javascript
const { Parser } = window.__SCANNER;

// 正常排序测试
const files1 = {
  'A.sol': 'import "./B.sol"; contract A {}',
  'B.sol': 'import "./C.sol"; contract B {}',
  'C.sol': 'contract C {}'
};
console.log('拓扑排序:', Parser.topologicalSort(files1));
// Expected: ['C.sol', 'B.sol', 'A.sol']

// 循环依赖检测
const files2 = {
  'A.sol': 'import "./B.sol"; contract A {}',
  'B.sol': 'import "./A.sol"; contract B {}'
};
try {
  Parser.topologicalSort(files2);
  console.log('循环依赖检测: FAIL（未抛出错误）');
} catch (e) {
  console.log('循环依赖检测:', e.message.includes('循环依赖') ? 'PASS' : 'FAIL');
}

// 注释过滤测试
const source = '// import "./Fake.sol";\nimport "./Real.sol";';
console.log('注释过滤:', Parser.parseImports(source).length === 1 ? 'PASS' : 'FAIL');

// Token 估算
console.log('Token估算:', Parser.estimateTokens('a'.repeat(2500)));
// Expected: 1000
```

Expected: 拓扑排序输出 `['C.sol', 'B.sol', 'A.sol']`，循环依赖检测输出 `PASS`，注释过滤输出 `PASS`，Token 估算输出 `1000`。

- [ ] **Step 3: Commit**

```bash
git add rugpull-scanner.html
git commit -m "feat: add contract parser with topological sort and cycle detection"
```

---

### Task 5: AI Analyzer 模块（System Prompt + User Message + MiMo 调用）

**Files:**
- Modify: `rugpull-scanner.html` (IIFE 内部，Parser 模块之后)

- [ ] **Step 1: 在 IIFE 内部追加 AI Analyzer 模块**

```javascript
  // ===== [5] AI Analyzer =====
  const Analyzer = {
    // System Prompt（字符级锁死，永不变化，确保 120 倍缓存命中）
    SYSTEM_PROMPT: `你是一名专业的智能合约安全审计师，精通 Solidity 语言和 EVM 虚拟机。

【思维链强制指令】
在输出任何 JSON 字段之前，你必须使用思维链（Thinking）将合约的每一行代码、每一个外部调用路径、每一个修饰符进行最少 3 轮的对抗性攻击模拟（模拟黑客如何利用重入、利用所有权漏洞、利用闪电贷等）。在思维链推演完毕后，再将最终确凿的审计结论序列化为 JSON 格式输出。

【知识截止日期】当前日期：2026年6月15日，知识截止：2024年12月

【输出格式】严格输出以下 JSON，不要包含任何其他文本、markdown 代码块标记或解释：

{
  "risk_score": 0-100的整数,
  "risk_level": "CRITICAL或HIGH或MEDIUM或LOW或SAFE",
  "summary": "一句话结论",
  "red_flags": [
    {
      "title": "风险标题",
      "severity": "CRITICAL或HIGH或MEDIUM或LOW",
      "description": "详细说明",
      "code_snippet": "相关代码片段",
      "recommendation": "修复建议"
    }
  ],
  "green_passes": [
    {
      "title": "安全项标题",
      "description": "说明"
    }
  ],
  "category_scores": {
    "honeypot_risk": 0-100,
    "rugpull_risk": 0-100,
    "owner_privilege": 0-100,
    "liquidity_risk": 0-100,
    "code_quality": 0-100
  }
}

【评分公式约束】
overall_score = min(100, Σ(wi * si))
- honeypot_risk: 发现蜜罐逻辑（如无法卖出、动态税率、隐藏后门）→ 强制 100 分
- owner_privilege: 发现隐藏所有权转移、无限增发、动态黑白名单 → 强制 ≥80 分
- rugpull_risk: 发现抽 Rugpull 机制（如 owner 可随时提走流动性）→ 强制 ≥80 分
- 5 维评分必须与 red_flags 中的发现严格一致，禁止自相矛盾
- risk_score 应等于 5 维 category_scores 的加权平均

【禁止事项】
- 不要在 JSON 中包含任何 Markdown 审计报告字段
- 不要输出 markdown 代码块标记（如 \`\`\`json）
- 不要在 JSON 之外输出任何额外文本`,

    // 组装 User Message（动态内容）
    buildUserMessage(mergedSource) {
      return `请审计以下智能合约源码：\n\n${mergedSource}`;
    },

    // 解析 AI 响应（三层熔断链）
    parseResponse(rawText) {
      // Layer 1: JSON.parse
      let parsed;
      try {
        // 尝试提取被 ```json ``` 包裹的内容
        const jsonMatch = rawText.match(/```json\s*([\s\S]*?)\s*```/) ||
                          rawText.match(/```\s*([\s\S]*?)\s*```/) ||
                          [null, rawText];
        parsed = JSON.parse(jsonMatch[1].trim());
      } catch (e) {
        return { mode: 'markdown', data: null, raw: rawText, error: 'JSON 解析失败' };
      }

      // Layer 2: 字段校验
      const required = ['risk_score', 'risk_level', 'summary', 'red_flags', 'green_passes', 'category_scores'];
      const missing = required.filter(k => parsed[k] === undefined);
      if (missing.length > 0) {
        return { mode: 'markdown', data: null, raw: rawText, error: `JSON 缺失字段: ${missing.join(', ')}` };
      }

      // 填充 category_scores 缺失维度的默认值
      const defaultScores = { honeypot_risk: 50, rugpull_risk: 50, owner_privilege: 50, liquidity_risk: 50, code_quality: 50 };
      parsed.category_scores = { ...defaultScores, ...parsed.category_scores };

      return { mode: 'json', data: parsed, raw: rawText, error: null };
    },

    // 执行完整扫描流程
    async scan(mergedSource) {
      const messages = [
        { role: 'system', content: this.SYSTEM_PROMPT },
        { role: 'user', content: this.buildUserMessage(mergedSource) }
      ];
      const response = await API.callMiMo(messages);
      // 保存 reasoning_content 用于潜在的多轮对话回传
      AppState.lastReasoningContent = response.reasoning_content;
      return this.parseResponse(response.content);
    }
  };

  window.__SCANNER.Analyzer = Analyzer;
```

- [ ] **Step 2: 在浏览器 Console 中验证 Prompt 结构和解析逻辑**

```javascript
const { Analyzer } = window.__SCANNER;

// 验证 System Prompt 不含动态内容
console.log('System Prompt 长度:', Analyzer.SYSTEM_PROMPT.length);
console.log('无动态日期:', !Analyzer.SYSTEM_PROMPT.includes('${'));

// 验证 JSON 解析（正常路径）
const testJson = '{"risk_score":85,"risk_level":"HIGH","summary":"test","red_flags":[],"green_passes":[],"category_scores":{"honeypot_risk":50,"rugpull_risk":50,"owner_privilege":50,"liquidity_risk":50,"code_quality":50}}';
const result1 = Analyzer.parseResponse(testJson);
console.log('JSON解析:', result1.mode === 'json' ? 'PASS' : 'FAIL');

// 验证 JSON 解析（降级路径）
const result2 = Analyzer.parseResponse('这不是JSON，是普通文本');
console.log('Markdown降级:', result2.mode === 'markdown' ? 'PASS' : 'FAIL');

// 验证字段缺失降级
const result3 = Analyzer.parseResponse('{"risk_score":50}');
console.log('字段缺失降级:', result3.mode === 'markdown' ? 'PASS' : 'FAIL');
```

Expected: 三个测试全部输出 `PASS`。

- [ ] **Step 3: Commit**

```bash
git add rugpull-scanner.html
git commit -m "feat: add AI analyzer with locked system prompt and three-layer fallback"
```

---

### Task 6: ECharts 可视化模块（雷达图 + 仪表盘 + 降级配置）

**Files:**
- Modify: `rugpull-scanner.html` (IIFE 内部，Analyzer 模块之后)

- [ ] **Step 1: 在 IIFE 内部追加 ECharts 模块**

```javascript
  // ===== [6] ECharts Visualization =====
  const Charts = {
    // 安全初始化（防内存泄漏：检测旧实例 → dispose → 重建）
    init() {
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
    },

    // 安全渲染
    safeRender(chart, option) {
      try {
        chart.setOption(option, true);
      } catch (e) {
        console.error('ECharts 渲染失败', e);
        chart.clear();
        chart.setOption({ title: { text: '图表渲染失败', subtext: '请刷新页面重试', left: 'center' } });
      }
    },

    // 渲染正常雷达图
    renderRadar(chart, scores) {
      const option = {
        title: { text: '风险维度雷达图', left: 'center', textStyle: { fontSize: 14 } },
        tooltip: {},
        radar: {
          indicator: [
            { name: '蜜罐风险', max: 100 },
            { name: 'Rugpull', max: 100 },
            { name: '所有者权限', max: 100 },
            { name: '流动性风险', max: 100 },
            { name: '代码质量', max: 100 }
          ],
          shape: 'polygon'
        },
        series: [{
          type: 'radar',
          data: [{
            value: [
              scores.honeypot_risk || 0,
              scores.rugpull_risk || 0,
              scores.owner_privilege || 0,
              scores.liquidity_risk || 0,
              scores.code_quality || 0
            ],
            name: '风险评分',
            areaStyle: { opacity: 0.2 },
            lineStyle: { width: 2 }
          }]
        }]
      };
      this.safeRender(chart, option);
    },

    // 渲染正常仪表盘
    renderGauge(chart, score, level) {
      const colorMap = {
        CRITICAL: '#ef4444', HIGH: '#f97316', MEDIUM: '#eab308',
        LOW: '#22c55e', SAFE: '#10b981'
      };
      const color = colorMap[level] || '#3b82f6';
      const option = {
        title: { text: '综合风险评分', left: 'center', textStyle: { fontSize: 14 } },
        series: [{
          type: 'gauge', min: 0, max: 100, radius: '80%',
          axisLine: {
            lineStyle: {
              width: 20,
              color: [[0.3, '#10b981'], [0.6, '#eab308'], [0.8, '#f97316'], [1, '#ef4444']]
            }
          },
          pointer: { itemStyle: { color: 'auto' } },
          axisTick: { distance: -20, length: 6, lineStyle: { color: '#fff', width: 1 } },
          splitLine: { distance: -24, length: 16, lineStyle: { color: '#fff', width: 2 } },
          axisLabel: { color: 'inherit', distance: 30, fontSize: 11 },
          detail: {
            valueAnimation: true, formatter: '{value}',
            color: color, offsetCenter: [0, '-10%'], fontSize: 36, fontWeight: 'bold'
          },
          title: { offsetCenter: [0, '25%'], fontSize: 14, color: '#94a3b8' },
          data: [{ value: score, name: level }]
        }]
      };
      this.safeRender(chart, option);
    },

    // 降级雷达图（灰色虚线 50 分中性）
    FALLBACK_RADAR_OPTION: {
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
    },

    // 降级仪表盘（灰色 "?" 标识）
    FALLBACK_GAUGE_OPTION: {
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
    },

    // 渲染降级图表
    renderFallback(charts) {
      this.safeRender(charts.radar, this.FALLBACK_RADAR_OPTION);
      this.safeRender(charts.gauge, this.FALLBACK_GAUGE_OPTION);
    }
  };

  // Resize 防抖
  function debounce(fn, delay = 150) {
    let timer;
    return (...args) => { clearTimeout(timer); timer = setTimeout(() => fn(...args), delay); };
  }
  window.addEventListener('resize', debounce(() => {
    if (AppState.charts) {
      AppState.charts.radar?.resize();
      AppState.charts.gauge?.resize();
    }
  }));

  window.__SCANNER.Charts = Charts;
```

- [ ] **Step 2: 在浏览器 Console 中验证 ECharts 初始化**

```javascript
const { Charts, AppState } = window.__SCANNER;
AppState.charts = Charts.init();
console.log('ECharts 初始化:', AppState.charts ? 'PASS' : 'FAIL');
console.log('Radar 实例:', AppState.charts.radar ? '存在' : '缺失');
console.log('Gauge 实例:', AppState.charts.gauge ? '存在' : '缺失');
```

Expected: 初始化输出 `PASS`，两个实例均输出 `存在`。

- [ ] **Step 3: 在浏览器 Console 中验证正常渲染和降级渲染**

```javascript
// 正常渲染
Charts.renderRadar(AppState.charts.radar, {
  honeypot_risk: 80, rugpull_risk: 60, owner_privilege: 40,
  liquidity_risk: 70, code_quality: 30
});
Charts.renderGauge(AppState.charts.gauge, 75, 'HIGH');
console.log('正常渲染: 已在页面上显示雷达图和仪表盘');

// 降级渲染
Charts.renderFallback(AppState.charts);
console.log('降级渲染: 已显示灰色虚线降级图表');
```

Expected: 页面上依次看到正常彩色图表，然后切换为灰色降级图表。

- [ ] **Step 4: Commit**

```bash
git add rugpull-scanner.html
git commit -m "feat: add ECharts visualization with radar, gauge, and fallback configs"
```

---

### Task 7: UI Renderer 模块（Markdown 渲染 + 红绿卡片 + 报告拼装）

**Files:**
- Modify: `rugpull-scanner.html` (IIFE 内部，Charts 模块之后)

- [ ] **Step 1: 在 IIFE 内部追加 UI Renderer 模块**

```javascript
  // ===== [7] UI Renderer =====
  const UI = {
    // 配置 marked.js + highlight.js 全局钩子
    configureMarked() {
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
    },

    // 更新进度条
    updateProgress(text, percent) {
      const progressContainer = document.getElementById('scan-progress');
      const progressBar = document.getElementById('progress-bar');
      const progressText = document.getElementById('progress-text');
      progressContainer.classList.remove('hidden');
      progressBar.style.width = `${percent || 0}%`;
      progressText.textContent = text;
    },

    // 隐藏进度条
    hideProgress() {
      document.getElementById('scan-progress').classList.add('hidden');
    },

    // 显示警告横幅
    showWarning(text) {
      const banner = document.getElementById('warning-banner');
      document.getElementById('warning-text').textContent = text;
      banner.classList.remove('hidden');
    },

    // 隐藏警告横幅
    hideWarning() {
      document.getElementById('warning-banner').classList.add('hidden');
    },

    // 渲染红绿卡片
    renderCards(data) {
      const container = document.getElementById('cards-container');
      container.innerHTML = '';

      // 红牌警告
      data.red_flags.forEach(flag => {
        const card = document.createElement('div');
        card.className = 'card card-red';
        const severityColors = { CRITICAL: 'text-red-400', HIGH: 'text-orange-400', MEDIUM: 'text-yellow-400', LOW: 'text-blue-400' };
        card.innerHTML = `
          <div class="flex items-center justify-between mb-2">
            <h3 class="font-semibold text-red-300">🚩 ${flag.title}</h3>
            <span class="text-xs px-2 py-0.5 rounded ${severityColors[flag.severity] || 'text-gray-400'} bg-red-500/10">${flag.severity}</span>
          </div>
          <p class="text-sm text-gray-300 mb-2">${flag.description}</p>
          ${flag.code_snippet ? `<pre class="hljs"><code class="language-solidity">${this.highlightCode(flag.code_snippet)}</code></pre>` : ''}
          <p class="text-xs text-green-400 mt-2">💡 ${flag.recommendation}</p>
        `;
        container.appendChild(card);
      });

      // 绿牌通过
      data.green_passes.forEach(pass => {
        const card = document.createElement('div');
        card.className = 'card card-green';
        card.innerHTML = `
          <h3 class="font-semibold text-green-300 mb-2">✅ ${pass.title}</h3>
          <p class="text-sm text-gray-300">${pass.description}</p>
        `;
        container.appendChild(card);
      });
    },

    // 单独高亮代码片段
    highlightCode(code) {
      try {
        return hljs.getLanguage('solidity')
          ? hljs.highlight(code, { language: 'solidity' }).value
          : hljs.highlightAuto(code).value;
      } catch (e) { return code.replace(/</g, '&lt;').replace(/>/g, '&gt;'); }
    },

    // 前端拼装 Markdown 审计报告
    assembleReport(data) {
      let md = `# 🔍 智能合约安全审计报告\n\n`;
      md += `## 综合评分: ${data.risk_score}/100 (${data.risk_level})\n\n`;
      md += `> ${data.summary}\n\n`;
      md += `## 🚩 红牌警告 (${data.red_flags.length})\n`;
      data.red_flags.forEach(f => {
        md += `### ${f.title}\n`;
        md += `**严重性**: ${f.severity}\n\n`;
        md += `${f.description}\n\n`;
        if (f.code_snippet) md += `\`\`\`solidity\n${f.code_snippet}\n\`\`\`\n\n`;
        md += `**修复建议**: ${f.recommendation}\n\n`;
      });
      md += `## ✅ 绿牌通过 (${data.green_passes.length})\n`;
      data.green_passes.forEach(p => {
        md += `### ${p.title}\n${p.description}\n\n`;
      });
      return md;
    },

    // 渲染 Markdown 到 DOM
    renderReport(markdown) {
      const container = document.getElementById('report-content');
      container.innerHTML = marked.parse(markdown);
      // 对报告中的代码块执行高亮
      container.querySelectorAll('pre code').forEach(block => {
        try { hljs.highlightElement(block); } catch (e) {}
      });
    },

    // 显示结果面板
    showResults() {
      document.getElementById('results-panel').style.display = 'block';
    },

    // 设置按钮状态
    setScanButtonState(disabled, text) {
      const btn = document.getElementById('btn-scan');
      btn.disabled = disabled;
      if (text) btn.innerHTML = text;
    }
  };

  // 初始化 marked + hljs 钩子
  UI.configureMarked();

  window.__SCANNER.UI = UI;
```

- [ ] **Step 2: 在浏览器 Console 中验证报告拼装**

```javascript
const { UI } = window.__SCANNER;
const testData = {
  risk_score: 85,
  risk_level: 'HIGH',
  summary: '发现多个高危漏洞',
  red_flags: [{
    title: '重入攻击风险',
    severity: 'CRITICAL',
    description: 'withdraw 函数存在重入漏洞',
    code_snippet: 'function withdraw() { msg.call{value:bal}(""); }',
    recommendation: '使用 ReentrancyGuard'
  }],
  green_passes: [{
    title: '使用 SafeMath',
    description: '合约使用了 SafeMath 库防止溢出'
  }],
  category_scores: { honeypot_risk: 90, rugpull_risk: 80, owner_privilege: 70, liquidity_risk: 60, code_quality: 40 }
};
const report = UI.assembleReport(testData);
console.log('报告拼装:', report.length > 100 ? 'PASS' : 'FAIL');
console.log('包含红牌:', report.includes('重入攻击风险') ? 'PASS' : 'FAIL');
console.log('包含绿牌:', report.includes('SafeMath') ? 'PASS' : 'FAIL');
```

Expected: 三个测试全部输出 `PASS`。

- [ ] **Step 3: Commit**

```bash
git add rugpull-scanner.html
git commit -m "feat: add UI renderer with marked+hljs integration, cards, and report assembly"
```

---

### Task 8: 主控逻辑（Tab 切换 + 扫描流程 + 状态机串联）

**Files:**
- Modify: `rugpull-scanner.html` (IIFE 内部，UI 模块之后，`})();` 之前)

- [ ] **Step 1: 在 IIFE 内部追加主控逻辑，串联完整扫描流程**

```javascript
  // ===== [8] Main Controller =====
  const Controller = {
    // Tab 切换
    initTabs() {
      const tabs = document.querySelectorAll('[data-tab]');
      const tabAuto = document.getElementById('tab-auto');
      const tabManual = document.getElementById('tab-manual');
      tabs.forEach(tab => {
        tab.addEventListener('click', () => {
          tabs.forEach(t => { t.classList.remove('tab-active'); t.classList.add('tab-inactive'); });
          tab.classList.remove('tab-inactive');
          tab.classList.add('tab-active');
          if (tab.dataset.tab === 'auto') {
            tabAuto.classList.remove('hidden');
            tabManual.classList.add('hidden');
          } else {
            tabAuto.classList.add('hidden');
            tabManual.classList.remove('hidden');
          }
        });
      });
    },

    // 获取输入的合约源码文件
    async getContractFiles() {
      const activeTab = document.querySelector('[data-tab].tab-active').dataset.tab;
      if (activeTab === 'auto') {
        const chainId = parseInt(document.getElementById('select-chain').value);
        const address = document.getElementById('input-address').value.trim();
        if (!address) throw new Error('请输入合约地址');
        if (!/^0x[a-fA-F0-9]{40}$/.test(address)) throw new Error('合约地址格式无效，应为 0x 开头的 42 位十六进制');
        return API.fetchContractSource(chainId, address);
      } else {
        const source = document.getElementById('input-source').value.trim();
        if (!source) throw new Error('请粘贴 Solidity 源码');
        return Parser.parseManualInput(source);
      }
    },

    // 防抖标记
    _scanning: false,

    // 主扫描流程
    async startScan() {
      if (this._scanning) return;
      this._scanning = true;

      const btn = document.getElementById('btn-scan');
      UI.setScanButtonState(true, '⏳ 扫描中...');
      UI.hideWarning();

      try {
        // Step 1: 获取合约源码
        UI.updateProgress('正在获取合约源码...', 10);
        AppState.scan.status = 'fetching';
        const files = await this.getContractFiles();
        AppState.scan.contractFiles = files;

        // Step 2: 拓扑排序 + 合并
        UI.updateProgress('正在解析依赖关系...', 30);
        AppState.scan.status = 'parsing';
        const merged = Parser.mergeFiles(files);
        AppState.scan.mergedSource = merged;

        // Step 3: Token 估算检查
        const tokens = Parser.estimateTokens(merged);
        UI.updateProgress(`源码已合并 (${tokens.toLocaleString()} tokens)，正在调用 AI 分析...`, 50);
        if (tokens > CONFIG.TOKEN_WARN_THRESHOLD) {
          UI.showWarning(`合约源码约 ${tokens.toLocaleString()} tokens，接近上下文限制，分析结果可能不完整`);
        }

        // Step 4: 调用 AI 分析
        AppState.scan.status = 'analyzing';
        UI.updateProgress('AI 正在深度审计合约...（可能需要 30-120 秒）', 60);
        const result = await Analyzer.scan(merged);
        AppState.scan.renderMode = result.mode;
        AppState.scan.parsedResult = result.data;
        AppState.scan.aiResponse = result.raw;

        // Step 5: 渲染结果
        UI.updateProgress('正在渲染分析结果...', 90);
        AppState.charts = Charts.init();

        if (result.mode === 'json' && result.data) {
          // 正常渲染路径
          Charts.renderRadar(AppState.charts.radar, result.data.category_scores);
          Charts.renderGauge(AppState.charts.gauge, result.data.risk_score, result.data.risk_level);
          UI.renderCards(result.data);
          const report = UI.assembleReport(result.data);
          UI.renderReport(report);
        } else {
          // Markdown 降级路径
          Charts.renderFallback(AppState.charts);
          UI.showWarning('AI 返回格式异常，以下为原始分析报告');
          UI.renderReport(result.raw);
          document.getElementById('cards-container').innerHTML = '';
        }

        // 完成
        UI.updateProgress('扫描完成！', 100);
        AppState.scan.status = 'done';
        UI.showResults();

        // 滚动到结果
        document.getElementById('results-panel').scrollIntoView({ behavior: 'smooth' });

      } catch (error) {
        AppState.scan.status = 'error';
        AppState.scan.error = error.message;
        UI.showWarning(`扫描失败: ${error.message}`);
        UI.showResults();
        console.error('扫描错误:', error);
      } finally {
        UI.setScanButtonState(false, '🚀 开始扫描');
        UI.hideProgress();
        this._scanning = false;
      }
    },

    // 初始化
    init() {
      // 初始化 Tab 切换
      this.initTabs();

      // 恢复链选择
      document.getElementById('select-chain').value = String(AppState.config.selectedChain);
      document.getElementById('select-chain').addEventListener('change', (e) => {
        AppState.config.selectedChain = parseInt(e.target.value);
        Storage.saveConfig(AppState.config);
      });

      // 扫描按钮事件（带防抖）
      document.getElementById('btn-scan').addEventListener('click', () => this.startScan());

      // 添加更多文件按钮（手动模式）
      document.getElementById('btn-add-file').addEventListener('click', () => {
        const textarea = document.getElementById('input-source');
        textarea.value += '\n\n=== File: NewContract.sol ===\n\n';
        textarea.focus();
        textarea.setSelectionRange(textarea.value.length, textarea.value.length);
      });

      // 检查 API Key
      if (!AppState.config.mimoApiKey) {
        console.warn('未配置 MiMo API Key，请点击齿轮图标设置');
      }
    }
  };

  // 启动应用
  Controller.init();
```

- [ ] **Step 2: 在浏览器中验证 Tab 切换**

点击「自动拉取」和「手动粘贴」Tab，确认两个面板正确切换显示/隐藏。

Expected: 点击 Tab 后对应面板可见，另一个隐藏，Tab 下划线颜色跟随切换。

- [ ] **Step 3: 验证输入校验（不填入任何内容直接点击扫描）**

点击「开始扫描」按钮，不输入任何内容。

Expected: 页面显示黄色警告横幅，提示「请输入合约地址」。

- [ ] **Step 4: 配置 API Key 后进行端到端测试**

1. 点击齿轮图标，输入有效的 MiMo API Key
2. 在「手动粘贴」Tab 中粘贴一个简单的 Solidity 合约：

```solidity
pragma solidity ^0.8.0;

contract SimpleStorage {
    uint256 private value;
    address public owner;

    constructor() {
        owner = msg.sender;
    }

    function setValue(uint256 _value) public {
        require(msg.sender == owner);
        value = _value;
    }

    function getValue() public view returns (uint256) {
        return value;
    }
}
```

3. 点击「开始扫描」

Expected: 进度条推进 → AI 分析中 → 结果面板显示雷达图、仪表盘、红绿卡片、Markdown 审计报告。

- [ ] **Step 5: Commit**

```bash
git add rugpull-scanner.html
git commit -m "feat: wire up main controller with tab switching, scan pipeline, and state machine"
```

---

### Task 9: 最终验证与收尾

**Files:**
- Modify: `rugpull-scanner.html` (如有需要的微调)

- [ ] **Step 1: 完整功能验证清单**

在浏览器中依次验证：

1. ✅ 双击打开 HTML，暗色主题正常渲染
2. ✅ CDN 全部加载成功（Console 无报错）
3. ✅ 齿轮图标打开设置弹窗，输入 Key 后保存到 localStorage
4. ✅ 刷新页面后 Key 仍在
5. ✅ Tab 切换正常（自动拉取 / 手动粘贴）
6. ✅ 空输入点击扫描 → 显示校验错误
7. ✅ 手动粘贴简单合约 → 完整扫描流程 → 结果面板显示
8. ✅ ECharts 雷达图和仪表盘正常渲染
9. ✅ 红绿卡片正确显示，Solidity 代码有语法高亮
10. ✅ Markdown 报告正确渲染
11. ✅ 点击背景关闭设置弹窗
12. ✅ 安全警告在页脚可见

- [ ] **Step 2: 最终 Commit**

```bash
git add rugpull-scanner.html
git commit -m "feat: complete rugpull scanner with all 6 modules integrated"
```
