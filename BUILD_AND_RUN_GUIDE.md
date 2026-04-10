# Claude Code 2.1.88 源码编译与运行指南

> **适用平台**: Windows 10/11 | macOS | Linux  
> **难度等级**: ⭐⭐⭐⭐ (需要中高级 Node.js/TypeScript 经验)  
> **预计耗时**: 路线一 5 分钟 / 路线二 2-4 小时  
> **最后更新**: 2026-04-10

---

## 📋 目录

1. [项目背景](#1-项目背景)
2. [两条运行路线对比](#2-两条运行路线对比)
3. [路线一：直接安装官方包（推荐）](#3-路线一直接安装官方包推荐)
   - [3.5 使用第三方模型（DeepSeek/Gemini/GPT 等）](#35-使用第三方模型deepseekgeminigpt-等)
4. [路线二：从源码编译运行](#4-路线二从源码编译运行)
   - [4.1 环境准备](#41-环境准备)
   - [4.2 创建 package.json](#42-创建-packagejson)
   - [4.3 创建 tsconfig.json](#43-创建-tsconfigjson)
   - [4.4 处理 bun:bundle 宏](#44-处理-bunbundle-宏)
   - [4.5 处理 MACRO 全局常量](#45-处理-macro-全局常量)
   - [4.6 创建构建脚本](#46-创建构建脚本)
   - [4.7 安装依赖](#47-安装依赖)
   - [4.8 编译和运行](#48-编译和运行)
5. [三大编译阻碍详解](#5-三大编译阻碍详解)
6. [环境变量配置](#6-环境变量配置)
7. [项目架构速查](#7-项目架构速查)
8. [常见问题排查](#8-常见问题排查)
9. [进阶：各子系统说明](#9-进阶各子系统说明)
10. [进阶：使用网页端 LLM 作为输入/输出端](#10-进阶使用网页端-llm-作为输入输出端)
    - [10.1 架构原理与挑战](#101-架构原理与挑战)
    - [10.2 三条实现路径对比](#102-三条实现路径对比)
    - [10.3 路径一：浏览器自动化代理](#103-路径一浏览器自动化代理)
    - [10.4 路径二：拦截网页端内部 API](#104-路径二拦截网页端内部-api)
    - [10.5 路径三：Prompt 工具模拟（推荐）](#105-路径三prompt-工具模拟推荐)
    - [10.6 完整链路实操指南](#106-完整链路实操指南)
    - [10.7 重要限制与风险](#107-重要限制与风险)

---

## 1. 项目背景

本项目是 Claude Code v2.1.88 的源码还原版本，从 npm 包中附带的 `cli.js.map` (Source Map) 提取而来。

**关键技术栈**：

| 技术 | 说明 |
|------|------|
| **TypeScript** | 500+ 源文件，完整类型系统 |
| **React 19 + 自定义 Ink** | 终端 UI 框架 (src/ink/ 下 46 个文件) |
| **Bun** | 原始构建工具链 (编译时宏系统) |
| **Commander.js** | CLI 命令解析框架 |
| **@anthropic-ai/sdk** | Claude API 调用 |
| **MCP** | Model Context Protocol 集成 |

**核心挑战**：原始代码使用 Bun 作为构建工具链，其 `feature()` 编译时宏和 `MACRO.*` 常量注入在标准 Node.js/tsc 环境中无法直接使用，需要创建 shim/polyfill 来替代。

---

## 2. 两条运行路线对比

| 维度 | 路线一：安装官方包 | 路线二：从源码编译 |
|------|------------------|------------------|
| **耗时** | 5 分钟 | 2-4 小时 |
| **难度** | ⭐ | ⭐⭐⭐⭐ |
| **能否修改源码** | ❌ 不能 | ✅ 可以 |
| **适用场景** | 日常使用 Claude Code | 研究架构、二次开发、定制功能 |
| **需要处理 Bun 宏** | 不需要 | 需要 |
| **完整功能** | ✅ 全功能 | ⚠️ 部分内部功能不可用 |

---

## 3. 路线一：直接安装官方包（推荐）

### 3.1 安装

```powershell
# 方式 A：最新稳定版 (推荐)
npm install -g @anthropic-ai/claude-code

# 方式 B：指定 2.1.88 版本（腾讯云镜像缓存，可能已失效）
npm install -g https://mirrors.cloud.tencent.com/npm/@anthropic-ai/claude-code/-/claude-code-2.1.88.tgz
```

### 3.2 配置 API Key

```powershell
# 设置环境变量
$env:ANTHROPIC_API_KEY = "sk-ant-xxxxx"

# 或写入用户环境变量（永久生效）
[Environment]::SetEnvironmentVariable("ANTHROPIC_API_KEY", "sk-ant-xxxxx", "User")
```

### 3.3 运行

```powershell
claude --help        # 查看帮助
claude --version     # 查看版本
claude               # 启动交互式终端
claude "你好"        # 单次提问
```

### 3.4 使用云端模型后端（可选）

```powershell
# AWS Bedrock
$env:CLAUDE_CODE_USE_BEDROCK = "1"
$env:AWS_REGION = "us-east-1"

# Google Vertex AI
$env:CLAUDE_CODE_USE_VERTEX = "1"
$env:CLOUD_ML_REGION = "us-east5"
$env:ANTHROPIC_VERTEX_PROJECT_ID = "your-project-id"

# Azure Foundry
$env:CLAUDE_CODE_USE_FOUNDRY = "1"
$env:ANTHROPIC_FOUNDRY_BASE_URL = "https://your-resource.azure.com"
```

### 3.5 使用第三方模型（DeepSeek/Gemini/GPT 等）

Claude Code **严格绑定 Anthropic Messages API 协议**，无法直接调用 DeepSeek/Gemini/OpenAI 等原生 API。但可以通过**协议转译网关**实现——网关接收 Anthropic 格式请求，自动转换后转发给目标模型。

```
Claude Code → Anthropic 格式请求 → [网关转译] → DeepSeek/Gemini/OpenAI 原生 API
```

**原因**：源码中 API 请求格式、工具调用（tool_use）、流式响应格式均为 Anthropic 专有格式，仅有 firstParty/Bedrock/Vertex/Foundry 四个硬编码提供商，无插件扩展机制。

#### 方案一：OpenRouter（最简单，无需本地部署）

[OpenRouter](https://openrouter.ai) 是云端 API 网关，支持 100+ 模型，自动转译协议。

```powershell
# 设置环境变量
$env:ANTHROPIC_BASE_URL = "https://openrouter.ai/api/v1"
$env:ANTHROPIC_API_KEY = "sk-or-v1-xxxxx"   # OpenRouter API Key

# 使用 DeepSeek
claude --model "deepseek/deepseek-chat-v3-0324"

# 使用 Gemini
claude --model "google/gemini-2.5-pro-preview-05-06"

# 使用 GPT-4o
claude --model "openai/gpt-4o"

# 使用 Llama
claude --model "meta-llama/llama-4-maverick"
```

**优点**：零部署、模型丰富、按量付费  
**缺点**：需注册、数据经第三方、有延迟加成

#### 方案二：LiteLLM（本地代理，最灵活）

[LiteLLM](https://github.com/BerriAI/litellm) 是开源的 API 协议转译代理，在本地运行，支持 Anthropic ↔ OpenAI ↔ 其他格式互转。

**安装：**

```powershell
pip install litellm[proxy]
```

**创建配置文件 `litellm_config.yaml`：**

```yaml
model_list:
  # DeepSeek V3
  - model_name: deepseek-v3
    litellm_params:
      model: deepseek/deepseek-chat
      api_key: sk-xxxxx                    # DeepSeek API Key
      api_base: https://api.deepseek.com/v1

  # DeepSeek R1 (推理模型)
  - model_name: deepseek-r1
    litellm_params:
      model: deepseek/deepseek-reasoner
      api_key: sk-xxxxx
      api_base: https://api.deepseek.com/v1

  # Gemini
  - model_name: gemini-pro
    litellm_params:
      model: gemini/gemini-2.5-pro-preview-05-06
      api_key: AIzaSyxxxxx                 # Google AI Studio Key

  # GPT-4o
  - model_name: gpt-4o
    litellm_params:
      model: openai/gpt-4o
      api_key: sk-xxxxx                    # OpenAI Key

  # 本地 Ollama 模型
  - model_name: qwen3
    litellm_params:
      model: ollama/qwen3:32b
      api_base: http://localhost:11434

  # 硅基流动 (SiliconFlow)
  - model_name: qwen-turbo
    litellm_params:
      model: openai/Qwen/Qwen2.5-72B-Instruct
      api_key: sk-xxxxx
      api_base: https://api.siliconflow.cn/v1
```

**启动代理并使用：**

```powershell
# 启动 LiteLLM 代理
litellm --config litellm_config.yaml --port 4000

# 配置 Claude Code
$env:ANTHROPIC_BASE_URL = "http://127.0.0.1:4000/v1"
$env:ANTHROPIC_API_KEY = "sk-1234"   # LiteLLM 的 master key，任意字符串

# 使用各种模型
claude --model deepseek-v3
claude --model gemini-pro
claude --model qwen3
```

**优点**：数据不离本地、完全可控、支持 Ollama 本地模型  
**缺点**：需要本地运行代理进程

#### 方案三：One API（自建多模型管理平台）

[One API](https://github.com/songquanpeng/one-api) 是开源的 API 管理和分发系统，支持统一密钥管理、用量统计、多渠道负载均衡。

```powershell
# Docker 部署
docker run -d -p 3000:3000 -e TZ=Asia/Shanghai justsong/one-api

# 在 Web 界面 http://localhost:3000 配置渠道后：
$env:ANTHROPIC_BASE_URL = "http://127.0.0.1:3000/v1"
$env:ANTHROPIC_API_KEY = "sk-one-api-xxxxx"   # One API 生成的令牌
claude --model deepseek-v3
```

#### 方案对比

| 维度 | OpenRouter | LiteLLM | One API |
|------|-----------|---------|--------|
| **部署** | 无需部署 | 本地 pip | Docker/二进制 |
| **数据隐私** | ⚠️ 经第三方 | ✅ 本地 | ✅ 本地 |
| **支持模型数** | 100+ | 100+ | 取决于配置 |
| **Ollama 本地模型** | ❌ | ✅ | ✅ |
| **用量管理/计费** | ✅ 内置 | ⚠️ 基础 | ✅ 完善 |
| **上手难度** | ⭐ | ⭐⭐ | ⭐⭐⭐ |
| **推荐场景** | 快速体验 | 个人开发 | 团队/企业 |

#### 注意事项

1. **工具调用兼容性**：Claude Code 大量使用 tool_use（工具调用），部分模型支持不完善：
   - ✅ DeepSeek V3/R1、GPT-4o、Gemini 2.5 Pro — 工具调用较好
   - ⚠️ 小模型 / 量化模型 — 工具调用容易出错，体验下降明显

2. **Extended Thinking**：Claude Code 的扩展思考特性仅 Anthropic 原生模型支持，第三方模型自动降级为普通模式

3. **永久配置**（避免每次设置环境变量）：
   ```powershell
   [Environment]::SetEnvironmentVariable("ANTHROPIC_BASE_URL", "http://127.0.0.1:4000/v1", "User")
   [Environment]::SetEnvironmentVariable("ANTHROPIC_API_KEY", "sk-1234", "User")
   ```

4. **交互中切换模型**：
   ```
   claude /model deepseek-v3    # 在 Claude Code 交互界面中切换
   ```

> ✅ **路线一到此结束。** 以下内容仅针对路线二（源码编译）。

---

## 4. 路线二：从源码编译运行

### 4.1 环境准备

#### 必需工具

| 工具 | 版本要求 | 安装命令 | 用途 |
|------|---------|---------|------|
| **Node.js** | ≥ 18.0.0 | [nodejs.org](https://nodejs.org) 下载 | 运行时 |
| **npm** | ≥ 9.x (随 Node 自带) | - | 包管理 |
| **Git** | 任意版本 | `winget install Git.Git` | 版本管理 |
| **Python** | 3.x (可选，node-gyp 需要) | `winget install Python.Python.3` | 编译原生模块 |
| **Visual Studio Build Tools** | 2022 (Windows 可选) | [下载](https://visualstudio.microsoft.com/visual-cpp-build-tools/) | 编译原生 .node 模块 |

#### 验证环境

```powershell
node --version       # 应 >= 18.0.0
npm --version        # 应 >= 9.x
git --version
python --version     # 可选
```

### 4.2 创建 package.json

在项目根目录创建 `package.json`：

```json
{
  "name": "claude-code-src",
  "version": "2.1.88",
  "description": "Claude Code v2.1.88 source recovery - build from source",
  "type": "module",
  "main": "dist/entrypoints/cli.js",
  "bin": {
    "claude-dev": "dist/entrypoints/cli.js"
  },
  "scripts": {
    "prebuild": "node scripts/patch-bun-macros.mjs",
    "build": "tsc --project tsconfig.json",
    "postbuild": "node scripts/inject-macros.mjs",
    "start": "node dist/entrypoints/cli.js",
    "dev": "node --import tsx src/entrypoints/cli.tsx",
    "clean": "rimraf dist",
    "typecheck": "tsc --noEmit"
  },
  "engines": {
    "node": ">=18.0.0"
  },
  "dependencies": {
    "@anthropic-ai/sdk": "^0.52.0",
    "@anthropic-ai/bedrock-sdk": "^0.15.0",
    "@anthropic-ai/vertex-sdk": "^0.8.0",
    "@commander-js/extra-typings": "^13.1.0",
    "@growthbook/sdk": "^1.4.1",
    "@modelcontextprotocol/sdk": "^1.12.1",
    "@opentelemetry/api": "^1.9.0",
    "@opentelemetry/exporter-trace-otlp-http": "^0.57.2",
    "@opentelemetry/resources": "^1.30.1",
    "@opentelemetry/sdk-trace-base": "^1.30.1",
    "@opentelemetry/sdk-trace-node": "^1.30.1",
    "@opentelemetry/semantic-conventions": "^1.28.0",
    "@anthropic-ai/foundry-sdk": "^0.1.0",
    "axios": "^1.8.4",
    "chalk": "^5.4.1",
    "chokidar": "^4.0.3",
    "commander": "^13.1.0",
    "diff": "^7.0.0",
    "eventsource": "^3.0.7",
    "execa": "^9.5.2",
    "fast-xml-parser": "^5.0.0",
    "google-auth-library": "^9.15.1",
    "ignore": "^7.0.4",
    "jsonc-parser": "^3.3.1",
    "jsonwebtoken": "^9.0.2",
    "lodash-es": "^4.17.21",
    "marked": "^15.0.9",
    "open": "^10.1.2",
    "plist": "^3.1.0",
    "proper-lockfile": "^4.1.2",
    "qrcode": "^1.5.4",
    "react": "^19.1.0",
    "semver": "^7.7.1",
    "sharp": "^0.34.1",
    "shell-quote": "^1.8.2",
    "tree-kill": "^1.2.2",
    "turndown": "^7.2.0",
    "uuid": "^11.1.0",
    "vscode-jsonrpc": "^9.0.0-next.6",
    "ws": "^8.18.1",
    "yaml": "^2.7.1",
    "zod": "^3.24.2",
    "zod-to-json-schema": "^3.24.5"
  },
  "devDependencies": {
    "@types/diff": "^7.0.2",
    "@types/jsonwebtoken": "^9.0.9",
    "@types/lodash-es": "^4.17.12",
    "@types/node": "^22.13.14",
    "@types/proper-lockfile": "^4.1.4",
    "@types/qrcode": "^1.5.5",
    "@types/react": "^19.1.0",
    "@types/semver": "^7.7.0",
    "@types/shell-quote": "^1.7.5",
    "@types/turndown": "^5.0.5",
    "@types/uuid": "^10.0.0",
    "@types/ws": "^8.18.0",
    "rimraf": "^6.0.1",
    "tsx": "^4.19.0",
    "typescript": "^5.8.3"
  }
}
```

### 4.3 创建 tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "lib": ["ES2022"],
    "outDir": "./dist",
    "rootDir": "./src",
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "strict": true,
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    "forceConsistentCasingInFileNames": true,
    "skipLibCheck": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "react-jsx",
    "paths": {
      "bun:bundle": ["./src/shims/bun-bundle.ts"],
      "src/*": ["./src/*"]
    },
    "baseUrl": "."
  },
  "include": ["src/**/*.ts", "src/**/*.tsx"],
  "exclude": ["node_modules", "dist"]
}
```

### 4.4 处理 bun:bundle 宏

这是最关键的步骤。`bun:bundle` 是 Bun 构建器的编译时模块，其 `feature()` 函数在编译时被替换为 `true`/`false` 常量，用于 dead code elimination。

#### 方案 A：创建 shim 模块（推荐）

创建目录和文件 `src/shims/bun-bundle.ts`：

```typescript
/**
 * bun:bundle shim — 替代 Bun 编译时宏
 *
 * 原始行为: Bun 在编译时将 feature('FLAG') 替换为 true/false 常量
 * Shim 行为: 运行时根据环境变量或默认值返回布尔值
 *
 * 使用环境变量 CLAUDE_CODE_FEATURES 启用特性:
 *   CLAUDE_CODE_FEATURES=BRIDGE_MODE,DAEMON,BUDDY
 */

// 外部构建默认开启的特性（非 Ant 内部特性）
const DEFAULT_ENABLED_FEATURES = new Set<string>([
  // 核心功能 — 大部分用户需要
  'WORKFLOW_SCRIPTS',
  'BG_SESSIONS',
  'TEMPLATES',
  'HISTORY_SNIP',
  'EXTRACT_MEMORIES',
  'FILE_PERSISTENCE',
  'TOKEN_BUDGET',
  'STREAMLINED_OUTPUT',
  'COMMIT_ATTRIBUTION',
  'WEB_BROWSER_TOOL',
  'FORK_SUBAGENT',
  'MCP_SKILLS',
]);

// 从环境变量加载额外启用的特性
const envFeatures = new Set(
  (process.env.CLAUDE_CODE_FEATURES ?? '')
    .split(',')
    .map(f => f.trim().toUpperCase())
    .filter(Boolean)
);

/**
 * 运行时特性门控。
 *
 * @param flag - 特性标志名称 (如 'BRIDGE_MODE', 'BUDDY' 等)
 * @returns 是否启用该特性
 */
export function feature(flag: string): boolean {
  return DEFAULT_ENABLED_FEATURES.has(flag) || envFeatures.has(flag);
}
```

#### 方案 B：全文替换为 true（快速但粗暴）

如果只想尽快跑起来，可以用脚本将所有 `feature('xxx')` 替换为 `true`：

```javascript
// scripts/patch-features-all-true.mjs
import { readFileSync, writeFileSync } from 'fs';
import { globSync } from 'fs';

const files = globSync('src/**/*.{ts,tsx}');
for (const file of files) {
  let content = readFileSync(file, 'utf-8');
  // 移除 bun:bundle 导入
  content = content.replace(/import\s*\{[^}]*\}\s*from\s*['"]bun:bundle['"];?\n?/g, '');
  // 替换所有 feature() 调用为 true
  content = content.replace(/feature\(['"][^'"]+['"]\)/g, 'true');
  writeFileSync(file, content);
}
console.log(`Patched ${files.length} files`);
```

> ⚠️ **注意**：方案 B 会启用所有特性（包括 Ant 内部特性），可能导致运行时错误。推荐使用方案 A。

### 4.5 处理 MACRO 全局常量

源码中使用了 7 个编译时注入的 MACRO 常量：

| 常量 | 出现次数 | 用途 |
|------|---------|------|
| `MACRO.VERSION` | 58+ | 版本号字符串 |
| `MACRO.PACKAGE_URL` | 17+ | npm 包名/URL |
| `MACRO.BUILD_TIME` | 4 | ISO 构建时间戳 |
| `MACRO.FEEDBACK_CHANNEL` | 6+ | 反馈频道 (/bug) |
| `MACRO.ISSUES_EXPLAINER` | 2 | issue 跟踪器 URL |
| `MACRO.NATIVE_PACKAGE_URL` | 10 | 原生包 URL |
| `MACRO.VERSION_CHANGELOG` | 3 | 版本更新日志 |

#### 创建 MACRO 全局定义

创建 `src/shims/macro.ts`：

```typescript
/**
 * MACRO 全局常量定义
 * 原始行为: Bun 在编译时将 MACRO.XXX 内联为常量值
 * Shim 行为: 运行时提供相同的值
 */

const MACRO = {
  VERSION: '2.1.88-dev',
  PACKAGE_URL: '@anthropic-ai/claude-code',
  BUILD_TIME: new Date().toISOString(),
  FEEDBACK_CHANNEL: '/bug',
  ISSUES_EXPLAINER: 'https://github.com/anthropics/claude-code/issues',
  NATIVE_PACKAGE_URL: '@anthropic-ai/claude-code',
  VERSION_CHANGELOG: '',
} as const;

// 注入到全局作用域
(globalThis as any).MACRO = MACRO;

export default MACRO;
```

然后在入口文件最前面导入：

```typescript
// 在 src/entrypoints/cli.tsx 最顶部添加
import '../shims/macro.js';
```

#### 注入脚本（编译后处理）

创建 `scripts/inject-macros.mjs`：

```javascript
/**
 * 编译后处理: 在 dist/ 输出中替换 MACRO 引用
 * 如果使用全局 shim 方式则不需要此脚本
 */

import { readFileSync, writeFileSync } from 'fs';
import { glob } from 'glob';

const VERSION = '2.1.88-dev';
const BUILD_TIME = new Date().toISOString();
const PACKAGE_URL = '@anthropic-ai/claude-code';

const macros = {
  'MACRO.VERSION': JSON.stringify(VERSION),
  'MACRO.PACKAGE_URL': JSON.stringify(PACKAGE_URL),
  'MACRO.BUILD_TIME': JSON.stringify(BUILD_TIME),
  'MACRO.FEEDBACK_CHANNEL': JSON.stringify('/bug'),
  'MACRO.ISSUES_EXPLAINER': JSON.stringify('https://github.com/anthropics/claude-code/issues'),
  'MACRO.NATIVE_PACKAGE_URL': JSON.stringify(PACKAGE_URL),
  'MACRO.VERSION_CHANGELOG': JSON.stringify(''),
};

const files = await glob('dist/**/*.js');
let patchCount = 0;

for (const file of files) {
  let content = readFileSync(file, 'utf-8');
  let modified = false;

  for (const [macro, value] of Object.entries(macros)) {
    if (content.includes(macro)) {
      content = content.replaceAll(macro, value);
      modified = true;
    }
  }

  if (modified) {
    writeFileSync(file, content);
    patchCount++;
  }
}

console.log(`✅ Injected MACROs into ${patchCount} files`);
```

### 4.6 创建构建脚本

创建 `scripts/patch-bun-macros.mjs` ——预编译宏处理：

```javascript
/**
 * 预编译处理脚本
 * 1. 将 import { feature } from 'bun:bundle' 替换为 shim 导入
 * 2. 确保 MACRO 全局可用
 */

import { readFileSync, writeFileSync, existsSync, mkdirSync } from 'fs';
import { glob } from 'glob';
import { dirname, relative, join } from 'path';

// Step 1: 确保 shim 文件存在
const shimsDir = 'src/shims';
if (!existsSync(shimsDir)) {
  mkdirSync(shimsDir, { recursive: true });
}

// Step 2: 替换 bun:bundle 导入
const files = await glob('src/**/*.{ts,tsx}');
let patchCount = 0;

for (const file of files) {
  let content = readFileSync(file, 'utf-8');
  let modified = false;

  // 替换 bun:bundle 导入为相对路径 shim
  if (content.includes("'bun:bundle'") || content.includes('"bun:bundle"')) {
    const fileDir = dirname(file);
    const shimPath = relative(fileDir, join(shimsDir, 'bun-bundle.js'))
      .replace(/\\/g, '/');
    const shimImport = shimPath.startsWith('.') ? shimPath : `./${shimPath}`;

    content = content.replace(
      /import\s*\{\s*feature\s*\}\s*from\s*['"]bun:bundle['"]/g,
      `import { feature } from '${shimImport}'`
    );
    modified = true;
  }

  // 替换 src/ 绝对导入为相对导入 (约 20+ 文件)
  const srcImportRegex = /from\s*['"](src\/[^'"]+)['"]/g;
  let match;
  while ((match = srcImportRegex.exec(content)) !== null) {
    const importPath = match[1];
    const fileDir = dirname(file);
    const targetPath = importPath; // src/xxx/yyy.js
    let relativePath = relative(fileDir, targetPath).replace(/\\/g, '/');
    if (!relativePath.startsWith('.')) {
      relativePath = './' + relativePath;
    }
    content = content.replace(
      new RegExp(`from\\s*['"]${importPath.replace(/[.*+?^${}()|[\]\\]/g, '\\$&')}['"]`),
      `from '${relativePath}'`
    );
    modified = true;
  }

  if (modified) {
    writeFileSync(file, content);
    patchCount++;
  }
}

console.log(`✅ Pre-build: patched ${patchCount} files`);
```

### 4.7 安装依赖

```powershell
cd "F:\SynologyDrive\000\AI\SKILL及claude code\CC_src-master"

# 安装所有依赖
npm install

# 如果 sharp 安装失败 (需要编译原生模块)
npm install --ignore-scripts
npm rebuild sharp
```

### 4.8 编译和运行

#### 完整编译流程

```powershell
# Step 1: 预处理宏 (替换 bun:bundle 导入)
npm run prebuild

# Step 2: TypeScript 编译
npm run build

# Step 3: 后处理 MACRO 注入
npm run postbuild

# Step 4: 运行
npm start -- --version
npm start -- --help
npm start
```

#### 快速开发模式（使用 tsx，跳过编译）

```powershell
# tsx 能直接运行 TypeScript，但需要先处理 bun:bundle
node scripts/patch-bun-macros.mjs
npx tsx src/entrypoints/cli.tsx --version
npx tsx src/entrypoints/cli.tsx --help
npx tsx src/entrypoints/cli.tsx
```

> ⚠️ tsx 模式下仍需要先运行预处理脚本替换 `bun:bundle` 导入。

---

## 5. 三大编译阻碍详解

### B1: `bun:bundle` 编译时宏 🔴

**问题**：100+ 个文件包含 `import { feature } from 'bun:bundle'`

**原理**：Bun 的 bundler 在编译时将 `feature('FLAG')` 替换为 `true/false` 布尔常量，然后 tree-shaker 删除 `if (false) {...}` 代码块。这实现了编译时特性裁剪。

**发现的 30+ 个特性门控标志**：

| 类别 | 标志 |
|------|------|
| **助手模式** | `KAIROS`, `KAIROS_BRIEF`, `KAIROS_CHANNELS`, `KAIROS_GITHUB_WEBHOOKS` |
| **远程/桥接** | `BRIDGE_MODE`, `CCR_MIRROR`, `CCR_AUTO_CONNECT`, `DIRECT_CONNECT`, `SSH_REMOTE` |
| **AI 分类器** | `BASH_CLASSIFIER`, `TRANSCRIPT_CLASSIFIER`, `PROACTIVE` |
| **内部工具** | `BUDDY`, `COORDINATOR_MODE`, `VOICE_MODE`, `DAEMON` |
| **技能/脚本** | `WORKFLOW_SCRIPTS`, `EXPERIMENTAL_SKILL_SEARCH`, `MCP_SKILLS`, `FORK_SUBAGENT` |
| **功能特性** | `BG_SESSIONS`, `TEMPLATES`, `HISTORY_SNIP`, `EXTRACT_MEMORIES`, `AGENT_TRIGGERS` |
| **集成** | `CHICAGO_MCP`, `UDS_INBOX` |
| **部署** | `BYOC_ENVIRONMENT_RUNNER`, `SELF_HOSTED_RUNNER` |
| **构建选项** | `WEB_BROWSER_TOOL`, `LODESTONE`, `CACHED_MICROCOMPACT` |

**解决方案**：见 [4.4 节](#44-处理-bunbundle-宏)

### B2: MACRO 全局常量 🟡

**问题**：80+ 处使用 `MACRO.VERSION`、`MACRO.BUILD_TIME` 等全局常量

**原理**：Bun 在编译时将 `MACRO.XXX` 内联替换为字符串常量

**解决方案**：见 [4.5 节](#45-处理-macro-全局常量)

### B3: ESM 导入路径 🟡

**问题**：所有 TypeScript 文件中的导入都使用 `.js` 扩展名 + 部分文件使用 `src/` 绝对路径

```typescript
// .js 扩展名 — TypeScript ESM 标准做法，tsc 不做转换
import { something } from './utils/helper.js';

// src/ 绝对路径 — 需要 tsconfig paths 或预处理
import { logEvent } from 'src/services/analytics/index.js';
```

**解决方案**：
- `.js` 扩展名：`tsconfig.json` 使用 `"module": "NodeNext"` + `"moduleResolution": "NodeNext"` 即可
- `src/` 路径：在预处理脚本中转为相对路径，或在 `tsconfig.json` 配置 `"paths": { "src/*": ["./src/*"] }`

---

## 6. 环境变量配置

### 必需

| 变量 | 说明 | 示例 |
|------|------|------|
| `ANTHROPIC_API_KEY` | Claude API 密钥 | `sk-ant-api03-xxxx` |

### 可选 — 模型后端

| 变量 | 说明 |
|------|------|
| `CLAUDE_CODE_USE_BEDROCK=1` | 切换到 AWS Bedrock |
| `AWS_REGION` | Bedrock 区域 |
| `CLAUDE_CODE_USE_VERTEX=1` | 切换到 Google Vertex |
| `ANTHROPIC_VERTEX_PROJECT_ID` | GCP 项目 ID |
| `CLOUD_ML_REGION` | Vertex AI 区域 |
| `CLAUDE_CODE_USE_FOUNDRY=1` | 切换到 Azure Foundry |
| `ANTHROPIC_FOUNDRY_BASE_URL` | Foundry 端点 URL |

### 可选 — 运行时行为

| 变量 | 说明 |
|------|------|
| `ANTHROPIC_MODEL` | 覆盖默认模型名 |
| `ANTHROPIC_BASE_URL` | 自定义 API base URL（用于代理） |
| `CLAUDE_CODE_SIMPLE=1` | 简化/裸模式 (--bare) |
| `CLAUDE_CODE_REMOTE=true` | 远程执行环境标志 |
| `DISABLE_PROMPT_CACHING=1` | 禁用提示词缓存 |
| `CLAUDE_CODE_FEATURES` | 逗号分隔的特性标志，如 `BRIDGE_MODE,DAEMON` |
| `CLAUDE_CODE_PROFILE_STARTUP=1` | 输出启动性能分析 |

---

## 7. 项目架构速查

### 目录结构

```
CC_src-master/
├── src/
│   ├── entrypoints/          # 入口点
│   │   ├── cli.tsx           # 🔑 主入口 — 快速路径 + main.tsx 加载
│   │   ├── init.ts           # 配置/网络/遥测初始化
│   │   ├── mcp.ts            # MCP 服务器入口
│   │   └── sdk/              # SDK/Headless 模式
│   │
│   ├── main.tsx              # 🔑 完整 CLI 主模块 (~4684 行)
│   │                         #     Commander.js 参数 + 60 命令 + 40 工具
│   ├── setup.ts              # 运行时设置 (Node版本检查/Session ID等)
│   │
│   ├── commands/             # 60+ 个命令 (commit, review, init, mcp...)
│   ├── tools/                # 40+ 个工具 (Read, Write, Bash, Grep...)
│   ├── components/           # React/Ink 终端 UI 组件
│   ├── ink/                  # 自定义 Ink 渲染引擎 (46 文件)
│   │
│   ├── services/
│   │   ├── api/              # API 层 (4 后端: Direct/Bedrock/Vertex/Foundry)
│   │   │   ├── client.ts     # 客户端创建 (多后端切换)
│   │   │   └── claude.ts     # LLM 请求/流式响应
│   │   ├── analytics/        # 遥测/分析
│   │   └── ...
│   │
│   ├── bridge/               # 远程控制系统 (30 文件)
│   ├── coordinator/          # 多 Agent 协调器
│   ├── skills/               # 内置技能系统
│   ├── plugins/              # 插件系统
│   ├── hooks/                # React Hooks (终端状态管理)
│   ├── state/                # 应用状态管理
│   ├── constants/            # 常量定义 (21 文件)
│   ├── types/                # TypeScript 类型
│   └── utils/                # 工具函数
│
├── vendor/                   # 原生模块源码 (audio/image/modifiers/url)
├── .claude/skills/           # 自定义 SKILL 文件
├── node_modules/             # 依赖 (已存在)
│
├── package.json              # ← 需要创建
├── tsconfig.json             # ← 需要创建
└── scripts/                  # ← 需要创建
    ├── patch-bun-macros.mjs  # 预处理宏
    └── inject-macros.mjs     # 后处理 MACRO
```

### 启动流程

```
用户执行 claude
        │
        ▼
  cli.tsx (入口)
        │
        ├── --version → 输出版本 → 退出
        ├── --dump-system-prompt → 内部调试
        ├── remote-control/rc → Bridge 模式
        ├── daemon → 后台进程
        ├── ps/logs/attach → 进程管理
        │
        └── (标准路径) → 加载 main.tsx
                              │
                              ▼
                        init.ts (初始化)
                        ├── 配置加载
                        ├── 网络检查
                        ├── 遥测启动
                        └── Graceful Shutdown
                              │
                              ▼
                        main.tsx (CLI 主体)
                        ├── Commander.js 参数定义
                        ├── 60+ 命令注册
                        ├── 40+ 工具注册
                        ├── 特性门控检查
                        └── React/Ink 渲染启动
                              │
                              ▼
                        交互式 REPL 循环
                        ├── 用户输入
                        ├── LLM API 调用 (services/api/)
                        ├── 工具执行
                        └── 终端 UI 更新
```

### API 调用链

```
用户消息 → QueryEngine.ts → claude.ts → client.ts → Anthropic SDK
                                                         │
                                          ┌──────────────┤
                                          ▼              ▼
                                    Direct API      Bedrock/Vertex/Foundry
                                          │              │
                                          └──────┬───────┘
                                                 ▼
                                          流式响应处理
                                                 │
                                          ┌──────┼──────┐
                                          ▼      ▼      ▼
                                       文本    工具调用  思考
                                       输出    执行     展示
```

---

## 8. 常见问题排查

### Q1: `Cannot find module 'bun:bundle'`

**原因**：未运行预处理脚本  
**解决**：
```powershell
node scripts/patch-bun-macros.mjs
```

### Q2: `MACRO is not defined`

**原因**：未注入 MACRO 全局常量  
**解决**：确保 `src/shims/macro.ts` 被创建，且在 `src/entrypoints/cli.tsx` 顶部导入

### Q3: `Cannot find module 'src/xxx'`

**原因**：部分文件使用 `src/` 绝对路径导入  
**解决**：
1. 确认 `tsconfig.json` 中配置了 `"paths": { "src/*": ["./src/*"] }`
2. 或运行预处理脚本自动转换

### Q4: `sharp` 安装失败

**原因**：sharp 需要编译原生模块  
**解决**：
```powershell
npm install --ignore-scripts
npx node-gyp rebuild
# 或使用预编译版本
npm install sharp --platform=win32
```

### Q5: TSX 组件编译报错

**原因**：JSX 配置问题  
**解决**：确认 `tsconfig.json` 中 `"jsx": "react-jsx"`

### Q6: `node_modules` 中缺少某些包

**原因**：现有 node_modules 可能不完整  
**解决**：
```powershell
# 删除现有 node_modules 重新安装
Remove-Item -Recurse -Force node_modules
npm install
```

### Q7: 运行后提示 API Key 无效

**原因**：未设置 `ANTHROPIC_API_KEY`  
**解决**：
```powershell
$env:ANTHROPIC_API_KEY = "sk-ant-api03-xxxx"
```

---

## 9. 进阶：各子系统说明

### 9.1 自定义 Ink 渲染引擎 (`src/ink/`)

46 个文件，实现了完整的终端 UI 渲染：

| 模块 | 文件 | 功能 |
|------|------|------|
| **核心渲染** | ink.tsx, root.ts, reconciler.ts | React Fiber 到终端的映射 |
| **布局** | layout/*, measure-element.ts | Yoga (Flexbox) 布局引擎 |
| **输出** | render-to-screen.ts, output.ts, screen.ts | ANSI 转义码生成 |
| **组件** | components/* | Box, Text, Spinner, Button, Link 等 |
| **事件** | events/*, focus.ts, parse-keypress.ts | 键盘输入处理 |

### 9.2 Bridge 远程控制 (`src/bridge/`)

30 个文件，实现 `claude remote-control` / `claude rc`：

- JWT 认证 + 可信设备
- WebSocket 实时通信
- 权限回调 (远程端需要本地确认)
- 会话管理 (创建/恢复/同步)

### 9.3 命令系统 (`src/commands/`)

60+ 个命令，分为：

| 类型 | 示例 |
|------|------|
| **核心** | commit, review, init, login, mcp |
| **代码分析** | security-review, insights, advisor |
| **工作流** | ultraplan, autofix-pr, bughunter |
| **配置** | config, color, compact, clear |
| **调试** | ant-trace, break-cache, cost |

### 9.4 工具系统 (`src/tools/`)

40+ 个工具，通过 MCP 协议暴露给 LLM：

```
Read, Write, Edit, Bash, Grep, Glob,
WebFetch, WebSearch, CreateFile, ListDir,
Screenshot, ComputerUse, ...
```

### 9.5 MCP 集成

- 内置 MCP 服务器 (`src/entrypoints/mcp.ts`)
- MCP 客户端 (连接外部 MCP 服务器)
- 支持 stdio 和 SSE 两种传输

---

## 附录 A：文件统计

| 类别 | 文件数 | 说明 |
|------|--------|------|
| TypeScript (.ts) | ~400+ | 核心逻辑 |
| TSX (.tsx) | ~100+ | 终端 UI 组件 |
| 自定义 Ink | 46 | 渲染引擎 |
| 命令 | 60+ | CLI 命令 |
| 工具 | 40+ | LLM 工具 |
| 常量文件 | 21 | 配置常量 |
| 自定义 SKILL | 8 | .claude/skills/ |

## 附录 B：完整特性标志参考

```
# 用逗号分隔，设置到 CLAUDE_CODE_FEATURES 环境变量
# 示例: CLAUDE_CODE_FEATURES=BRIDGE_MODE,DAEMON,BUDDY

# 助手模式
KAIROS                      # Claude AI 助手模式
KAIROS_BRIEF               # 简报功能
KAIROS_CHANNELS             # 频道功能
KAIROS_GITHUB_WEBHOOKS      # GitHub Webhook 集成

# 远程/桥接
BRIDGE_MODE                 # 远程控制模式
CCR_MIRROR                  # CCR 镜像
CCR_AUTO_CONNECT           # 自动连接
DIRECT_CONNECT             # 直连模式
SSH_REMOTE                 # SSH 远程

# 智能功能
BASH_CLASSIFIER             # Bash 命令分类器
TRANSCRIPT_CLASSIFIER       # 对话分类器
PROACTIVE                   # 主动建议

# 内部工具
BUDDY                       # 伴侣动画
COORDINATOR_MODE            # 协调器模式
VOICE_MODE                  # 语音模式
DAEMON                      # 守护进程

# 技能/脚本
WORKFLOW_SCRIPTS             # 工作流脚本
EXPERIMENTAL_SKILL_SEARCH    # 实验性技能搜索
MCP_SKILLS                  # MCP 技能
FORK_SUBAGENT               # Fork 子代理

# 功能开关
BG_SESSIONS                 # 后台会话
TEMPLATES                   # 模板系统
HISTORY_SNIP                # 历史裁剪
EXTRACT_MEMORIES            # 记忆提取
AGENT_TRIGGERS              # 代理触发器
FILE_PERSISTENCE            # 文件持久化
TOKEN_BUDGET                # Token 预算
STREAMLINED_OUTPUT          # 精简输出
COMMIT_ATTRIBUTION          # 提交归属
WEB_BROWSER_TOOL            # 网页浏览工具
LODESTONE                   # Lodestone 集成
CACHED_MICROCOMPACT         # 缓存微压缩

# 部署
BYOC_ENVIRONMENT_RUNNER     # 自带环境运行器
SELF_HOSTED_RUNNER          # 自托管运行器
```

---

> **文档版本**: v1.1 | **适用代码版本**: Claude Code 2.1.88 | **日期**: 2026-04-10

---

## 10. 进阶：使用网页端 LLM 作为输入/输出端

> **难度**: ⭐⭐⭐⭐⭐ | **稳定性**: ⚠️ 中低 | **推荐程度**: 仅在无法获取 API Key 时考虑

本章介绍如何将 ChatGPT/Claude.ai/DeepSeek/Gemini 等网页端对话框作为 Claude Code 的 LLM 后端。

### 10.1 架构原理与挑战

#### 当前 API 调用链

```
用户输入
    ↓
QueryEngine.submitMessage()     [src/QueryEngine.ts]
    ↓
query()                         [src/query.ts]
    ↓
queryModelWithStreaming()        [src/services/api/claude.ts]
    ↓
anthropic.beta.messages.stream() ← 核心 API 调用点
    ↓
BetaRawMessageStreamEvent SSE 流
    ↓
┌──────────────────┬──────────────────┐
▼                  ▼                  ▼
text_delta      tool_use          thinking
    ↓               ↓
终端显示      执行工具 → tool_result → 再次调用 API
```

#### 核心挑战

Claude Code **严格依赖 Anthropic Messages API 协议**，包括：

| 协议层 | 格式 | 问题 |
|--------|------|------|
| **请求格式** | Anthropic `messages` 格式（含 `tool_use` 块） | 网页端无法接受程序化结构化输入 |
| **工具调用** | `tool_use` / `tool_result` 结构化块 | 网页端无原生 tool_use 支持 |
| **流式响应** | Anthropic 专有 SSE 事件格式 | 网页端不暴露 SSE 接口 |
| **工具循环** | 自动化多轮: 调用→执行→返回结果→再调用 | 网页端为人类交互设计，不支持程序驱动 |

**本质矛盾**：Claude Code 的工具调用是**完全自动化**的（每次调用 API → 解析工具调用指令 → 执行工具 → 把结果喂回 → 再调用，循环几十次无人干预），而网页端对话框是为人类单轮交互设计的。

---

### 10.2 三条实现路径对比

| 路径 | 原理 | 复杂度 | 稳定性 | 推荐程度 |
|------|------|--------|--------|---------|
| **路径一** 浏览器自动化 | Playwright 操控 DOM 输入/读取 | ⭐⭐⭐⭐⭐ | ❌ 极低 | 🚫 不推荐 |
| **路径二** 拦截内部 API | 用网页端 Session Token 调内部 API | ⭐⭐⭐⭐ | ⚠️ 低 | ⚠️ 谨慎 |
| **路径三** Prompt 工具模拟 | 将工具定义嵌 Prompt，文本解析结果 | ⭐⭐⭐ | ⚠️ 中 | ✅ 最可行 |

---

### 10.3 路径一：浏览器自动化代理

用 Playwright 自动化控制网页聊天界面，本地包装为 HTTP API 端点。

```
Claude Code → [HTTP 请求 Anthropic 格式]
      ↓
本地代理服务器 :4000
      ↓
请求 → 纯文本 prompt（将工具定义序列化为文字）
      ↓
Playwright 浏览器
  1. 在网页输入框填入文本
  2. 点击发送按钮
  3. 监听 DOM 变化，实时捕获流式输出
  4. 检测"生成完毕"标志（如停止按钮消失）
      ↓
响应 → 解析纯文本 → 伪造 Anthropic SSE 流
      ↓
Claude Code ← SSE 响应
```

**致命缺陷**：

- 网页端不支持结构化 `tool_use` 输出，必须教模型输出特定格式文本再解析，可靠性极低
- 任何网页 UI 变动（按钮改名、输入框 ID 变化）都会导致自动化脚本失效
- 速率限制严苛（GPT-4 约每 3 小时 40 条消息）
- 工具调用循环会在数分钟内耗尽额度

---

### 10.4 路径二：拦截网页端内部 API

大多数 LLM 网页端底层实际调用自己的内部 HTTP API，可用浏览器 Cookie/Session Token 直接调用，跳过 UI 层。

```
Claude Code → [Anthropic 格式请求]
      ↓
LiteLLM / 自建代理
  Anthropic 格式 → OpenAI 格式转换
      ↓
网页端"逆向"代理（用 Session Token 调内部 API）
      ↓
网站内部 API（ChatGPT / DeepSeek / Gemini 等）
      ↓
响应 → OpenAI 格式 → Anthropic SSE 格式
      ↓
Claude Code ← SSE 响应
```

#### 开源逆向工具参考

| 工具 | 支持平台 | GitHub | 原理 |
|------|---------|--------|------|
| **chat2api** | ChatGPT 网页 | [lanqian528/chat2api](https://github.com/lanqian528/chat2api) | Access Token 调内部 API |
| **fuclaude** | Claude.ai 网页 | [wozulong/fuclaude](https://github.com/wozulong/fuclaude) | Session Key 镜像代理 |
| **gemini-openai-proxy** | Gemini 网页 | [zhu327/gemini-openai-proxy](https://github.com/zhu327/gemini-openai-proxy) | Cookie 调内部 API |
| **coze2openai** | Coze/豆包 | [fatwang2/coze2openai](https://github.com/fatwang2/coze2openai) | API Token 转换 |

> ⚠️ 上述工具均以逆向工程方式运行，稳定性依赖于平台内部 API 不发生变化。平台检测到自动化访问时会触发风控或封号。

---

### 10.5 路径三：Prompt 工具模拟（推荐）

**最务实的方案**：改造 Claude Code 的工具调用协议层，让不支持原生 `tool_use` 的模型通过 prompt 工程模拟。

```
原始流程：
messages + tools → API (tool_use 原生) → tool_use block → 执行工具

改造流程：
messages + tools → [适配层]
    ↓
将工具定义序列化进 system prompt（纯文本格式）
    ↓
任意 LLM（含网页端/本地部署）
    ↓
纯文本回复（含特定标记格式的工具调用）
    ↓
[适配层] 解析文本 → 伪造 tool_use block
    ↓
Claude Code 正常处理工具循环
```

#### 改造层一：工具定义 → 文本 Prompt

将结构化的 Anthropic tool schema 转换为纯文本指令嵌入 system prompt：

```
原始 Anthropic tool 格式：
{
  "tools": [{
    "name": "Read",
    "description": "读取文件内容",
    "input_schema": {
      "properties": {
        "file_path":   { "type": "string", "description": "文件路径" },
        "start_line":  { "type": "number" },
        "end_line":    { "type": "number" }
      },
      "required": ["file_path"]
    }
  }]
}

转换后的 system prompt 追加段：
---
你有以下工具可用。需要调用工具时，必须严格输出以下 XML 格式（在正文之后）：

<tool_call>
{"name": "工具名", "input": {参数JSON}}
</tool_call>

可用工具：
1. Read - 读取文件内容
   参数: file_path (string, 必需), start_line (number, 可选), end_line (number, 可选)

2. Write - 写入文件
   参数: file_path (string, 必需), content (string, 必需)

3. Bash - 执行 shell 命令
   参数: command (string, 必需), timeout (number, 可选)
...（40+ 工具全部序列化）
---
```

#### 改造层二：响应解析 → tool_use block

```typescript
// 在适配层中解析 LLM 纯文本回复
function parseToolCallsFromText(responseText: string): {
  textContent: string,
  toolCalls: Array<{ name: string, input: object, id: string }>
} {
  const toolCallRegex = /<tool_call>\s*([\s\S]*?)\s*<\/tool_call>/g;
  const toolCalls = [];
  let textContent = responseText;

  let match;
  while ((match = toolCallRegex.exec(responseText)) !== null) {
    try {
      const parsed = JSON.parse(match[1]);
      toolCalls.push({
        name: parsed.name,
        input: parsed.input ?? {},
        id: 'toolu_' + crypto.randomUUID().replace(/-/g, '')
      });
      // 从文本内容中移除工具调用块
      textContent = textContent.replace(match[0], '');
    } catch {
      // JSON 解析失败 → 模型格式错误，跳过
    }
  }

  return { textContent: textContent.trim(), toolCalls };
}
```

#### 改造层三：伪造 Anthropic SSE 事件流

```typescript
// 将解析结果包装为 Claude Code 期望的 BetaRawMessageStreamEvent 格式
async function* fakeAnthropicSSEStream(
  textContent: string,
  toolCalls: Array<{ name: string, input: object, id: string }>
): AsyncGenerator<object> {
  const msgId = 'msg_fake_' + Date.now();

  // 1. message_start
  yield {
    type: 'message_start',
    message: {
      id: msgId, type: 'message', role: 'assistant',
      content: [], model: 'custom-web-llm',
      stop_reason: null, stop_sequence: null,
      usage: { input_tokens: 0, output_tokens: 0,
               cache_creation_input_tokens: 0, cache_read_input_tokens: 0 }
    }
  };

  // 2. 文本内容块
  if (textContent) {
    yield { type: 'content_block_start', index: 0,
            content_block: { type: 'text', text: '' } };
    yield { type: 'content_block_delta', index: 0,
            delta: { type: 'text_delta', text: textContent } };
    yield { type: 'content_block_stop', index: 0 };
  }

  // 3. 工具调用块
  const baseIndex = textContent ? 1 : 0;
  for (let i = 0; i < toolCalls.length; i++) {
    const tc = toolCalls[i];
    const idx = baseIndex + i;
    yield { type: 'content_block_start', index: idx,
            content_block: { type: 'tool_use', id: tc.id, name: tc.name, input: {} } };
    yield { type: 'content_block_delta', index: idx,
            delta: { type: 'input_json_delta',
                     partial_json: JSON.stringify(tc.input) } };
    yield { type: 'content_block_stop', index: idx };
  }

  // 4. 结束事件
  const stopReason = toolCalls.length > 0 ? 'tool_use' : 'end_turn';
  yield { type: 'message_delta',
          delta: { stop_reason: stopReason, stop_sequence: null },
          usage: { output_tokens: Math.ceil(textContent.length / 4),
                   cache_creation_input_tokens: 0, cache_read_input_tokens: 0 } };
  yield { type: 'message_stop' };
}
```

#### 关键改造点定位

| 改造点 | 文件 | 代码位置 | 说明 |
|--------|------|---------|------|
| **客户端替换** | `src/services/api/client.ts` | `getAnthropicClient()` | 返回自定义 mock client |
| **流式调用替换** | `src/services/api/claude.ts` | `anthropic.beta.messages.stream()` | 接入自定义流生成器 |
| **工具 schema 转换** | `src/utils/api.ts` | `toolToAPISchema()` | 可选：转为其他格式 |
| **消息格式转换** | `src/services/api/claude.ts` | `userMessageToMessageParam()` | 已内置，无需改动 |
| **响应事件消费** | `src/query.ts` | 事件循环 | 已内置，无需改动 |

---

### 10.6 完整链路实操指南

**推荐组合**：网页端逆向工具 + LiteLLM + Claude Code

#### Step 1：网页端 → OpenAI 兼容 API

选择对应你使用的网页端的工具，在本地启动代理：

```powershell
# ChatGPT 网页 → OpenAI API（使用 chat2api）
docker run -p 3040:3040 -e CHATGPT_COOKIE="your_cookie" lanqian528/chat2api

# Claude.ai 网页 → Anthropic API 镜像（使用 fuclaude）
docker run -p 8080:8080 wozulong/fuclaude

# DeepSeek 网页（抓取 Bearer Token 后直调，DeepSeek API 本身已支持 OpenAI 格式）
# 直接在 LiteLLM 中配置，无需额外工具
```

#### Step 2：OpenAI 兼容 API → Anthropic 协议（LiteLLM）

创建 `litellm_web_config.yaml`：

```yaml
model_list:
  # ChatGPT 网页版（通过 chat2api 转换）
  - model_name: chatgpt-web
    litellm_params:
      model: openai/gpt-4o
      api_key: any-string          # chat2api 不验证 key
      api_base: http://127.0.0.1:3040/v1

  # Claude.ai 网页版（通过 fuclaude 镜像）
  - model_name: claude-web
    litellm_params:
      model: anthropic/claude-opus-4-6
      api_key: your-session-key
      api_base: http://127.0.0.1:8080

  # DeepSeek 网页版（DeepSeek 官方 API，价格极低）
  - model_name: deepseek-web
    litellm_params:
      model: deepseek/deepseek-chat
      api_key: sk-xxxxx
      api_base: https://api.deepseek.com/v1
```

```powershell
# 启动 LiteLLM
litellm --config litellm_web_config.yaml --port 4000
```

#### Step 3：配置 Claude Code

```powershell
# 永久写入用户环境变量
[Environment]::SetEnvironmentVariable("ANTHROPIC_BASE_URL", "http://127.0.0.1:4000/v1", "User")
[Environment]::SetEnvironmentVariable("ANTHROPIC_API_KEY", "sk-any-dummy-key", "User")

# 使用指定模型运行
claude --model chatgpt-web
claude --model deepseek-web
```

#### 完整链路示意图

```
claude (Anthropic Messages API 格式)
      ↓
LiteLLM :4000  ── Anthropic ↔ OpenAI 协议转换
      ↓
chat2api / fuclaude :3040  ── OpenAI API ↔ 网页端 Session
      ↓
ChatGPT / Claude.ai / DeepSeek 网页账号
```

---

### 10.7 重要限制与风险

| 限制 | 详情 | 严重程度 |
|------|------|---------|
| **工具调用可靠性** | 网页端模型无原生 tool_use，靠 prompt 模拟；小模型频繁格式错误 | 🔴 高 |
| **速率限制** | ChatGPT 免费版约每 3 小时 40 条；工具调用循环快速耗尽额度 | 🔴 高 |
| **违反服务条款** | 多数平台明确禁止自动化调用网页端 | 🔴 高 |
| **Session 稳定性** | Cookie/Token 会过期（通常 1-7 天），需定期刷新 | 🟡 中 |
| **上下文截断** | 网页端可能静默截断长对话，导致工具循环丢失上下文 | 🟡 中 |
| **Extended Thinking** | Claude Code 的扩展思考特性仅 Anthropic 原生支持，第三方自动降级 | 🟡 中 |
| **响应格式漂移** | 网页端 UI 变化可能导致 DOM 自动化脚本失效 | 🟡 中 |

#### 替代建议

如果目标是**降低成本**而非必须使用网页端，以下方案更稳定：

| 方案 | 价格估算 | 稳定性 | 工具调用 |
|------|---------|--------|---------|
| **DeepSeek API** | ~Claude 的 1/50 | ✅ 高 | ✅ 支持 |
| **硅基流动 Qwen** | 极低甚至免费额度 | ✅ 高 | ✅ 支持 |
| **本地 Ollama** | 免费（需 GPU） | ✅ 高 | ⚠️ 取决于模型 |
| **OpenRouter 低价模型** | 按量，可低至 $0.1/M tokens | ✅ 高 | ✅ 多数支持 |

以上方案均可通过 [3.5 节](#35-使用第三方模型deepseekgeminigpt-等) 的 LiteLLM 方式对接 Claude Code，无需任何源码改动。
