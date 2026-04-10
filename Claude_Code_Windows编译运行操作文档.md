# Claude Code 源码在 Windows 上编译运行 — 完整操作文档

> **基于 Claude Code v2.1.88 恢复源码的深度逆向工程分析**
>
> 本文档详细说明如何将从 npm source map 恢复的 Claude Code TypeScript 源码，在 Windows 系统上构建、编译并运行。包含环境准备、依赖处理、构建配置、编译阻碍排除和运行调试的全流程操作指南。

---

## 目录

- [1. 项目现状分析](#1-项目现状分析)
- [2. 方案总览：两条路线](#2-方案总览两条路线)
- [3. 路线一：直接安装官方 npm 包运行（最简单）](#3-路线一直接安装官方-npm-包运行最简单)
- [4. 路线二：从源码编译运行（完整重建）](#4-路线二从源码编译运行完整重建)
- [5. 阶段一：Windows 环境准备](#5-阶段一windows-环境准备)
- [6. 阶段二：项目骨架重建](#6-阶段二项目骨架重建)
- [7. 阶段三：解决 Bun 构建宏](#7-阶段三解决-bun-构建宏)
- [8. 阶段四：TypeScript 编译配置](#8-阶段四typescript-编译配置)
- [9. 阶段五：构建脚本与编译](#9-阶段五构建脚本与编译)
- [10. 阶段六：原生模块处理](#10-阶段六原生模块处理)
- [11. 阶段七：运行与调试](#11-阶段七运行与调试)
- [12. 阶段八：功能验证清单](#12-阶段八功能验证清单)
- [13. 常见问题与排障](#13-常见问题与排障)
- [14. 进阶：使用 Bun 原生构建](#14-进阶使用-bun-原生构建)

---

## 1. 项目现状分析

### 1.1 你拿到的是什么

这是从 `@anthropic-ai/claude-code@2.1.88` npm 包中的 `cli.js.map` (Source Map) 还原出来的**纯源代码**。

| 项目 | 状态 |
|------|------|
| TypeScript 源码 (`src/`) | ✅ 完整（500+ 文件） |
| `package.json` | ❌ **不存在** |
| `tsconfig.json` | ❌ **不存在** |
| 构建脚本 | ❌ **不存在** |
| 编译产物 (`dist/`, `cli.js`) | ❌ **不存在** |
| `node_modules/` | ⚠️ 有目录结构但可能不完整 |
| 原生二进制 (`.node` 文件) | ❌ 需要编译或获取 |

### 1.2 三大核心阻碍

| 阻碍编号 | 问题 | 难度 |
|---------|------|------|
| **B1** | `import { feature } from 'bun:bundle'` — Bun 专有的编译时特性门控宏，50+ 个文件使用 | 🔴 高 |
| **B2** | `MACRO.VERSION` / `MACRO.BUILD_TIME` / `MACRO.PACKAGE_URL` — 构建时注入的全局常量 | 🟡 中 |
| **B3** | `.ts`/`.tsx` 源码需要编译为 `.js`，且所有 import 路径以 `.js` 结尾（ESM 风格） | 🟡 中 |

### 1.3 入口点结构

```text
src/entrypoints/cli.tsx    ← 主引导入口（零模块快速路径 + 动态加载主模块）
src/entrypoints/init.ts    ← 配置/网络/遥测初始化
src/entrypoints/mcp.ts     ← MCP 服务器入口
src/main.tsx               ← 完整 CLI 主模块（Commander.js 命令注册）
```

---

## 2. 方案总览：两条路线

```text
┌─────────────────────────────────────────────────────┐
│  路线一：直安装运行 (5 分钟)                          │
│  npm install 官方编译好的包，直接运行 cli.js          │
│  适合：想用 Claude Code 的人                         │
├─────────────────────────────────────────────────────┤
│  路线二：源码编译运行 (2-4 小时)                      │
│  重建构建系统，处理宏替换，编译 TypeScript             │
│  适合：想研究/修改源码、定制功能的开发者               │
└─────────────────────────────────────────────────────┘
```

---

## 3. 路线一：直接安装官方 npm 包运行（最简单）

如果你只是想在 Windows 上**运行** Claude Code（而非修改源码），直接安装官方或镜像包即可。

### 3.1 安装

```powershell
# 方法1：腾讯云镜像（官方 npm 已下架 2.1.88）
npm install -g https://mirrors.cloud.tencent.com/npm/@anthropic-ai/claude-code/-/claude-code-2.1.88.tgz

# 方法2：直接安装最新版
npm install -g @anthropic-ai/claude-code
```

### 3.2 配置并运行

```powershell
# 设置 API Key
$env:ANTHROPIC_API_KEY = "sk-ant-xxx"

# 运行
claude
```

### 3.3 验证安装位置

```powershell
# 查看全局包位置
npm root -g
# 通常在 C:\Users\<用户名>\AppData\Roaming\npm\node_modules\@anthropic-ai\claude-code\

# 查看编译后的入口文件
Get-ChildItem "$(npm root -g)\@anthropic-ai\claude-code\cli.js" | Select-Object Length
# 这个 cli.js 就是 Bun 编译打包后的单文件产物（数 MB 大小）
```

> **关键认知**：官方的 `cli.js` 是 Bun 打包出来的单个 JavaScript 文件，已经解决了所有 `feature()`、`MACRO.*`、TypeScript 编译等问题。本仓库的 `src/` 目录是从 `cli.js.map` 反向还原的源码。

---

## 4. 路线二：从源码编译运行（完整重建）

以下是从源码到可运行程序的全流程。

---

## 5. 阶段一：Windows 环境准备

### 5.1 必装软件

```powershell
# 1. Node.js >= 18（推荐 20 LTS 或更高）
winget install OpenJS.NodeJS.LTS
# 验证
node --version    # 需要 >= 18.0.0

# 2. Git
winget install Git.Git
git --version

# 3. Python 3.x（用于编译原生模块）
winget install Python.Python.3.12

# 4. Visual Studio Build Tools（C++ 编译工具链）
winget install Microsoft.VisualStudio.2022.BuildTools
# 安装完成后，打开 Visual Studio Installer，勾选：
# - "使用 C++ 的桌面开发" 工作负载
# - Windows 10/11 SDK

# 5. (可选但推荐) Bun 运行时 —— 原生支持 feature() 宏
powershell -c "irm bun.sh/install.ps1 | iex"
bun --version
```

### 5.2 配置 npm 原生模块编译环境

```powershell
# 设置 Python 路径
npm config set python "C:\Users\$env:USERNAME\AppData\Local\Programs\Python\Python312\python.exe"

# 或使用 windows-build-tools（自动化配置）
npm install -g windows-build-tools
```

### 5.3 启用 Windows 长路径支持

```powershell
# 以管理员运行
reg add "HKLM\SYSTEM\CurrentControlSet\Control\FileSystem" /v LongPathsEnabled /t REG_DWORD /d 1 /f
```

---

## 6. 阶段二：项目骨架重建

### 6.1 创建 `package.json`

在项目根目录创建：

```json
{
  "name": "claude-code-source",
  "version": "2.1.88",
  "private": true,
  "type": "module",
  "bin": {
    "claude": "./dist/entrypoints/cli.js"
  },
  "engines": {
    "node": ">=18.0.0"
  },
  "scripts": {
    "prebuild": "node scripts/replace-macros.mjs",
    "build": "tsc -p tsconfig.json",
    "start": "node --enable-source-maps dist/entrypoints/cli.js",
    "dev": "tsx src/entrypoints/cli.tsx",
    "clean": "rimraf dist"
  },
  "dependencies": {
    "@anthropic-ai/sdk": "^0.52.0",
    "@anthropic-ai/bedrock-sdk": "^0.15.0",
    "@anthropic-ai/vertex-sdk": "^0.8.0",
    "@commander-js/extra-typings": "^13.1.0",
    "@growthbook/sdk": "^1.4.0",
    "@modelcontextprotocol/sdk": "^1.12.1",
    "@opentelemetry/api": "^1.9.0",
    "@opentelemetry/sdk-metrics": "^1.30.0",
    "@opentelemetry/sdk-trace-base": "^1.30.0",
    "axios": "^1.8.4",
    "chalk": "^5.4.1",
    "chokidar": "^4.0.3",
    "commander": "^13.1.0",
    "diff": "^7.0.0",
    "eventsource": "^3.0.6",
    "execa": "^9.6.0",
    "fuse.js": "^7.1.0",
    "google-auth-library": "^9.15.1",
    "ignore": "^7.0.4",
    "jsonc-parser": "^3.3.1",
    "jsonwebtoken": "^9.0.2",
    "lodash-es": "^4.17.21",
    "marked": "^15.0.11",
    "node-fetch": "^3.3.2",
    "open": "^10.1.0",
    "parse5": "^7.2.1",
    "pkce-challenge": "^5.0.0",
    "proper-lockfile": "^4.1.2",
    "qrcode": "^1.5.4",
    "react": "^19.1.0",
    "semver": "^7.7.1",
    "sharp": "^0.34.1",
    "shell-quote": "^1.8.2",
    "tree-kill": "^1.2.2",
    "turndown": "^7.2.0",
    "undici": "^7.8.0",
    "uuid": "^11.1.0",
    "vscode-jsonrpc": "^8.2.1",
    "ws": "^8.18.0",
    "xss": "^1.0.15",
    "yaml": "^2.7.1",
    "zod": "^3.24.3",
    "zod-to-json-schema": "^3.24.5"
  },
  "devDependencies": {
    "@types/diff": "^7.0.2",
    "@types/jsonwebtoken": "^9.0.9",
    "@types/lodash-es": "^4.17.12",
    "@types/node": "^22.15.3",
    "@types/proper-lockfile": "^4.1.4",
    "@types/qrcode": "^1.5.5",
    "@types/react": "^19.1.0",
    "@types/semver": "^7.7.0",
    "@types/shell-quote": "^1.7.5",
    "@types/turndown": "^5.0.5",
    "@types/uuid": "^10.0.0",
    "@types/ws": "^8.18.0",
    "rimraf": "^6.0.1",
    "tsx": "^4.19.4",
    "typescript": "^5.8.3"
  }
}
```

> ⚠️ **注意**：版本号是根据 `node_modules/` 目录中已有的包推断的。某些包可能需要微调版本。如果已有 `node_modules/` 目录且包含依赖，可以跳过 `npm install` 或仅安装 devDependencies。

### 6.2 检查并补充依赖

```powershell
# 如果 node_modules 已存在且基本完整，只安装 devDependencies
npm install --save-dev typescript tsx rimraf @types/node @types/react

# 如果需要全新安装（删除旧 node_modules 后）
npm install
```

---

## 7. 阶段三：解决 Bun 构建宏

这是最关键的一步。源码中有两类 Bun 专有构造需要处理：

### 7.1 `bun:bundle` 的 `feature()` 函数

**原理**：`feature('FLAG_NAME')` 在 Bun 构建时被替换为 `true` 或 `false` 常量，然后 Bun 的 Tree Shaker 将 `if (false) {...}` 的代码块完全删除（Dead Code Elimination）。

**策略**：创建一个 shimmodule，让所有 `feature()` 默认返回 `false`（禁用内部实验特性），或者选择性地开启部分功能。

#### 创建 `src/shims/bun-bundle.ts`

```typescript
/**
 * Shim for `bun:bundle` — replaces Bun's compile-time feature() macro.
 * In the original build, `feature('FLAG')` is resolved to true/false at
 * compile time and dead code is eliminated. This shim evaluates at runtime.
 *
 * Set CLAUDE_CODE_FEATURES=FLAG1,FLAG2 to enable specific features.
 */

const enabledFeatures = new Set(
  (process.env.CLAUDE_CODE_FEATURES || '').split(',').map(s => s.trim()).filter(Boolean)
);

// The public release had these features ENABLED:
const DEFAULT_ENABLED: string[] = [
  // 'DUMP_SYSTEM_PROMPT',     // Ant-only, disabled in external builds
  // 'ABLATION_BASELINE',      // Research-only
  // 'COORDINATOR_MODE',       // Multi-worker mode
  // 'KAIROS',                 // Assistant mode
  // 'CHICAGO_MCP',            // Computer use MCP
  // 'DAEMON',                 // Long-running supervisor
  // 'BRIDGE_MODE',            // Remote control
  // 'BG_SESSIONS',            // Background sessions
  // 'TEMPLATES',              // Template jobs
  // 'BYOC_ENVIRONMENT_RUNNER',// BYOC headless
  // 'SELF_HOSTED_RUNNER',     // Self-hosted runner
];

for (const f of DEFAULT_ENABLED) {
  enabledFeatures.add(f);
}

export function feature(name: string): boolean {
  return enabledFeatures.has(name);
}
```

### 7.2 `MACRO.*` 全局常量

**原理**：`MACRO.VERSION` 等在源码中直接作为全局标识符引用（不是对象属性查找，而是编译时文本替换）。

#### 创建 `src/shims/macros.ts`

```typescript
/**
 * Runtime shim for build-time MACRO.* constants.
 * In the original Bun build, these are inlined as literals.
 */

// Declare as global namespace to match source usage pattern
declare global {
  const MACRO: {
    VERSION: string;
    PACKAGE_URL: string;
    BUILD_TIME: string;
    ISSUES_EXPLAINER: string;
    FEEDBACK_CHANNEL: string;
    NATIVE_PACKAGE_URL: string | undefined;
  };
}

// Inject into globalThis
(globalThis as any).MACRO = {
  VERSION: '2.1.88',
  PACKAGE_URL: '@anthropic-ai/claude-code',
  BUILD_TIME: new Date().toISOString(),
  ISSUES_EXPLAINER: 'https://github.com/anthropics/claude-code/issues',
  FEEDBACK_CHANNEL: '/bug',
  NATIVE_PACKAGE_URL: undefined,
};
```

### 7.3 创建预处理脚本

#### 创建 `scripts/replace-macros.mjs`

这个脚本在编译前自动处理所有宏引用：

```javascript
/**
 * Pre-build script: replaces bun:bundle imports and MACRO references
 * across all TypeScript source files.
 *
 * Run with: node scripts/replace-macros.mjs
 */
import { readFileSync, writeFileSync, readdirSync, statSync } from 'fs';
import { join, relative, dirname } from 'path';
import { fileURLToPath } from 'url';

const __dirname = dirname(fileURLToPath(import.meta.url));
const srcDir = join(__dirname, '..', 'src');
const vendorDir = join(__dirname, '..', 'vendor');

let filesModified = 0;
let importsReplaced = 0;

function processFile(filePath) {
  const ext = filePath.split('.').pop();
  if (!['ts', 'tsx'].includes(ext)) return;

  let content = readFileSync(filePath, 'utf-8');
  let modified = false;

  // 1. Replace `import { feature } from 'bun:bundle'` with shim import
  if (content.includes("from 'bun:bundle'") || content.includes('from "bun:bundle"')) {
    const relPath = relative(dirname(filePath), join(srcDir, 'shims', 'bun-bundle.js'))
      .replace(/\\/g, '/')
      .replace(/\.ts$/, '.js');
    const importPath = relPath.startsWith('.') ? relPath : './' + relPath;

    content = content.replace(
      /import\s*\{\s*feature\s*\}\s*from\s*['"]bun:bundle['"]\s*;?/g,
      `import { feature } from '${importPath}';`
    );
    modified = true;
    importsReplaced++;
  }

  if (modified) {
    writeFileSync(filePath, content, 'utf-8');
    filesModified++;
  }
}

function walkDir(dir) {
  for (const entry of readdirSync(dir)) {
    const fullPath = join(dir, entry);
    const stat = statSync(fullPath);
    if (stat.isDirectory() && entry !== 'node_modules' && entry !== 'dist' && entry !== '.git') {
      walkDir(fullPath);
    } else if (stat.isFile()) {
      processFile(fullPath);
    }
  }
}

console.log('🔧 Processing bun:bundle imports...');
walkDir(srcDir);
walkDir(vendorDir);
console.log(`✅ Modified ${filesModified} files, replaced ${importsReplaced} bun:bundle imports`);
```

### 7.4 执行预处理

```powershell
# 先备份源码（重要！）
Copy-Item -Path src -Destination src_backup -Recurse

# 执行宏替换
node scripts/replace-macros.mjs
```

---

## 8. 阶段四：TypeScript 编译配置

### 8.1 创建 `tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "lib": ["ES2022"],
    "outDir": "./dist",
    "rootDir": ".",
    "sourceMap": true,
    "declaration": false,
    "strict": true,
    "skipLibCheck": true,
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "react-jsx",
    "paths": {
      "src/*": ["./src/*"]
    },
    "baseUrl": "."
  },
  "include": [
    "src/**/*.ts",
    "src/**/*.tsx",
    "vendor/**/*.ts"
  ],
  "exclude": [
    "node_modules",
    "dist",
    "src_backup",
    "scripts"
  ]
}
```

### 8.2 关于 `src/*` 路径别名

源码中存在 `import ... from 'src/services/...'` 形式的路径引用。TypeScript 的 `paths` 映射只在编译检查时有效，运行时需要额外处理：

#### 方案 A：使用 `tsx` 运行（开发阶段推荐）

`tsx` 自动处理路径别名和 TypeScript 编译：

```powershell
# 安装
npm install -g tsx

# 直接运行 TypeScript 源码（无需编译）
tsx --tsconfig tsconfig.json src/entrypoints/cli.tsx --version
```

#### 方案 B：使用 `tsc-alias` 后处理（生产构建）

```powershell
npm install --save-dev tsc-alias

# 在 package.json scripts 中修改 build 命令
# "build": "tsc -p tsconfig.json && tsc-alias -p tsconfig.json"
```

### 8.3 处理 `react/compiler-runtime` 导入

源码中有些文件使用了 React Compiler Runtime：
```typescript
import { c as _c } from "react/compiler-runtime";
```

如果你使用的 React 版本没有这个模块，需要创建 shim：

#### 创建 `src/shims/react-compiler-runtime.ts`
```typescript
// Shim for react/compiler-runtime (React Compiler optimization)
// In production builds this is provided by the React Compiler Babel plugin
export function c(size: number) {
  return new Array(size);
}
```

然后在 `tsconfig.json` 的 `paths` 中添加映射：
```json
{
  "paths": {
    "src/*": ["./src/*"],
    "react/compiler-runtime": ["./src/shims/react-compiler-runtime.ts"]
  }
}
```

---

## 9. 阶段五：构建脚本与编译

### 9.1 完整构建流程

```powershell
# Step 1: 确保 shim 文件就位
# （参见阶段三中创建的文件）

# Step 2: 全局 MACRO 注入 — 在入口文件最顶部引入
# 在 src/entrypoints/cli.tsx 第 1 行前插入：
# import '../shims/macros.js';

# Step 3: 预处理 bun:bundle 导入
node scripts/replace-macros.mjs

# Step 4: TypeScript 编译
npx tsc -p tsconfig.json

# Step 5: 路径别名解析（如果使用 tsc-alias）
npx tsc-alias -p tsconfig.json

# Step 6: 验证产物
Test-Path dist/src/entrypoints/cli.js
```

### 9.2 快速开发模式（跳过编译）

```powershell
# 使用 tsx 直接运行 TypeScript（最方便的开发方式）
npx tsx src/entrypoints/cli.tsx --version

# 带环境变量运行
$env:ANTHROPIC_API_KEY = "sk-ant-xxx"
npx tsx src/entrypoints/cli.tsx
```

### 9.3 预期的编译错误与解决方案

首次 `tsc` 编译时可能遇到数百个类型错误。以下是常见分类和处理策略：

| 错误类型 | 示例 | 解决方案 |
|---------|------|---------|
| 找不到模块 `bun:bundle` | `Cannot find module 'bun:bundle'` | 已通过预处理脚本解决 |
| 找不到 `MACRO` | `Cannot find name 'MACRO'` | 确保 `macros.ts` 的 `declare global` 在 `tsconfig` include 范围内 |
| 缺少类型声明 | `Could not find declaration file for module 'xxx'` | `npm install @types/xxx` 或在 `src/shims/` 中创建 `.d.ts` |
| 路径别名解析失败 | `Cannot find module 'src/xxx'` | 检查 `tsconfig.json` 的 `paths` 和 `baseUrl` |
| React/JSX 类型错误 | `Cannot find namespace 'JSX'` | 确保安装了 `@types/react` |
| 缺少内部模块 | 某些 `import` 指向不存在的 `.js` 文件 | 这些可能是未还原的源码，需要创建 stub |

**实用策略**：先用宽松模式跑通，再逐步修复：

```json
// tsconfig.json 中临时添加（开发阶段）
{
  "compilerOptions": {
    "strict": false,
    "noImplicitAny": false,
    "skipLibCheck": true,
    "noEmit": false
  }
}
```

---

## 10. 阶段六：原生模块处理

### 10.1 原生模块清单

源码中引用了以下 `.node` 原生二进制：

| 模块 | 源码位置 | 平台 | Windows 必需？ |
|------|---------|------|--------------|
| `audio-capture.node` | `vendor/audio-capture-src/` | macOS/Linux/Win | ❌ 可选（语音功能） |
| `image-processor.node` | `vendor/image-processor-src/` | macOS only | ❌ Windows 不支持 |
| `modifiers-napi.node` | `vendor/modifiers-napi-src/` | 跨平台 | ❌ 可选 |

### 10.2 处理策略：优雅降级

这些原生模块都使用了**懒加载 + try-catch** 模式，加载失败时不会崩溃。例如 `audio-capture-src/index.ts`：

```typescript
function loadNativeModule(): NativeModule | null {
  try {
    return require(modulePath);
  } catch {
    return null;  // Graceful fallback — feature just won't be available
  }
}
```

因此 Windows 上可以**直接忽略**这些原生模块的编译。核心 CLI 功能（对话、工具调用、代码编辑）不依赖它们。

### 10.3 `sharp` 图像处理库

`sharp` 是一个需要平台原生二进制的 npm 包。安装时会自动下载预编译的二进制：

```powershell
# sharp 通常自动安装，如果失败：
npm install sharp --platform=win32
```

---

## 11. 阶段七：运行与调试

### 11.1 首次运行检查清单

```powershell
# 1. 设置必要的环境变量
$env:ANTHROPIC_API_KEY = "sk-ant-api03-xxx"

# 2. 使用 tsx 直接运行（推荐开发方式）
npx tsx src/entrypoints/cli.tsx --version
# 期望输出：2.1.88 (Claude Code)

# 3. 测试基本对话
npx tsx src/entrypoints/cli.tsx -p "Hello, what is 2+2?"

# 4. 交互模式
npx tsx src/entrypoints/cli.tsx
```

### 11.2 环境变量速查

```powershell
# === 必需 ===
$env:ANTHROPIC_API_KEY = "sk-ant-xxx"

# === 推荐设置（避免高级特性报错）===
$env:CLAUDE_CODE_DISABLE_THINKING = "1"       # 如果模型不支持 thinking
$env:DISABLE_PROMPT_CACHING = "1"             # 如果用代理
$env:CLAUDE_CODE_SIMPLE = "1"                 # 简化模式，减少依赖

# === 调试用 ===
$env:CLAUDE_CODE_DEBUG = "1"                  # 启用调试日志
$env:CLAUDE_CODE_PROFILE_STARTUP = "1"        # 启动性能分析
$env:NODE_OPTIONS = "--enable-source-maps"    # 源码映射（调试堆栈）
```

### 11.3 使用编译后的 JS 运行

```powershell
# 如果 tsc 编译成功产出了 dist/
node --enable-source-maps dist/src/entrypoints/cli.js --version
```

### 11.4 调试技巧

```powershell
# VS Code 调试配置（.vscode/launch.json）
```

创建 `.vscode/launch.json`：
```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Debug Claude Code (tsx)",
      "type": "node",
      "request": "launch",
      "runtimeExecutable": "npx",
      "runtimeArgs": ["tsx"],
      "args": ["src/entrypoints/cli.tsx"],
      "cwd": "${workspaceFolder}",
      "env": {
        "ANTHROPIC_API_KEY": "sk-ant-xxx",
        "CLAUDE_CODE_DEBUG": "1"
      },
      "console": "integratedTerminal",
      "skipFiles": ["<node_internals>/**"]
    }
  ]
}
```

---

## 12. 阶段八：功能验证清单

按以下顺序逐步验证：

### 阶段 A：最小可行验证

| # | 测试项 | 命令 | 期望结果 |
|---|--------|------|---------|
| A1 | 版本输出 | `tsx cli.tsx --version` | `2.1.88 (Claude Code)` |
| A2 | 帮助信息 | `tsx cli.tsx --help` | 显示命令列表 |
| A3 | 单次提问 | `tsx cli.tsx -p "Hi"` | 返回 AI 响应 |

### 阶段 B：核心功能验证

| # | 测试项 | 命令 | 期望结果 |
|---|--------|------|---------|
| B1 | 交互对话 | `tsx cli.tsx` | 进入 REPL 界面 |
| B2 | 工具调用 | 在对话中让它 "list files in current dir" | 调用 BashTool/ListFilesTool |
| B3 | 文件编辑 | 让它 "create a hello.py file" | 使用 WriteFileTool |
| B4 | 模型切换 | `/model sonnet` | 切换到 Sonnet |

### 阶段 C：高级功能验证

| # | 测试项 | 命令 | 期望结果 |
|---|--------|------|---------|
| C1 | MCP 服务器 | `tsx mcp.ts` | 启动 MCP Server |
| C2 | 自定义命令 | `/compact` `/clear` | 内置命令可用 |
| C3 | 上下文管理 | 长对话后观察 token 计数 | 自动 compact 触发 |

---

## 13. 常见问题与排障

### Q1: `Cannot find module 'bun:bundle'`

**原因**：预处理脚本未运行或遗漏了某些文件。

**解决**：
```powershell
# 重新运行预处理
node scripts/replace-macros.mjs

# 检查是否有遗漏
Select-String -Path "src\**\*.ts","src\**\*.tsx" -Pattern "bun:bundle" -Recurse
```

### Q2: `MACRO is not defined`

**原因**：全局 MACRO 对象未注入。

**解决**：确保 `src/shims/macros.ts` 被最早执行。在 `cli.tsx` 的第一行添加：
```typescript
import './shims/macros.js';  // 必须是第一个 import
```

或者使用 Node.js 的 `--require` 预加载：
```powershell
node --require ./dist/src/shims/macros.js dist/src/entrypoints/cli.js
```

### Q3: `ERR_MODULE_NOT_FOUND` — 找不到 `.js` 文件

**原因**：TypeScript 源码中的 `import` 使用 `.js` 扩展名（ESM 规范），但实际文件是 `.ts`。

**解决**：使用 `tsx` 而非 `node` 运行，`tsx` 自动处理 `.ts` ↔ `.js` 映射。

### Q4: `Error: Cannot find module 'src/services/xxx'`

**原因**：`src/` 前缀的路径别名未被正确解析。

**解决**：
```powershell
# 方案1：使用 tsx 的 tsconfig paths 支持
npx tsx --tsconfig tsconfig.json src/entrypoints/cli.tsx

# 方案2：安装路径别名运行时支持
npm install tsconfig-paths
node -r tsconfig-paths/register dist/src/entrypoints/cli.js
```

### Q5: `react/compiler-runtime` 模块不存在

**原因**：需要 React Compiler 的运行时，但未安装。

**解决**：使用前面创建的 shim，或安装 React Compiler：
```powershell
npm install react-compiler-runtime
```

### Q6: Ink/React 终端界面乱码或无输出

**原因**：Windows Terminal 对 ANSI 转义序列的支持差异。

**解决**：
```powershell
# 使用 Windows Terminal（而非 cmd.exe）
# 或设置环境变量强制 ANSI 支持
$env:FORCE_COLOR = "1"

# 如果使用 PowerShell 7+，通常自动支持
```

### Q7: `sharp` 安装失败

```powershell
# 清理缓存后重新安装
npm cache clean --force
npm install sharp

# 如果仍然失败，手动指定平台
npm install @img/sharp-win32-x64
```

### Q8: TypeScript 编译错误太多，无从下手

**策略**：分层修复。

```powershell
# 1. 先只编译入口文件看有多少直接依赖链的错误
npx tsc --noEmit src/entrypoints/cli.tsx 2>&1 | Select-Object -First 50

# 2. 使用 --skipLibCheck 跳过第三方库的类型错误
# 3. 将 strict 设为 false
# 4. 逐个解决剩余错误
```

---

## 14. 进阶：使用 Bun 原生构建

如果你安装了 Bun（Windows 上已支持），可以更接近官方的构建方式：

### 14.1 Bun 原生运行（最简方案）

```powershell
# 安装 Bun for Windows
powershell -c "irm bun.sh/install.ps1 | iex"

# Bun 原生支持 TypeScript + JSX + 路径别名
# 但 bun:bundle 的 feature() 需要通过 Bun 的 build API 使用

# 直接运行（Bun 会处理 .ts/.tsx 编译）
bun run src/entrypoints/cli.tsx --version
```

### 14.2 Bun Build 打包

创建 `build.ts`：

```typescript
import { build } from 'bun';

await build({
  entrypoints: ['src/entrypoints/cli.tsx'],
  outdir: './dist',
  target: 'node',
  format: 'esm',
  sourcemap: 'linked',
  define: {
    'MACRO.VERSION': JSON.stringify('2.1.88'),
    'MACRO.PACKAGE_URL': JSON.stringify('@anthropic-ai/claude-code'),
    'MACRO.BUILD_TIME': JSON.stringify(new Date().toISOString()),
    'MACRO.ISSUES_EXPLAINER': JSON.stringify('https://github.com/anthropics/claude-code/issues'),
    'MACRO.FEEDBACK_CHANNEL': JSON.stringify('/bug'),
    'MACRO.NATIVE_PACKAGE_URL': 'undefined',
  },
  // Feature flags — set to true/false to include/exclude code paths
  conditions: [
    // 'ABLATION_BASELINE',
    // 'BRIDGE_MODE',
    // 'BG_SESSIONS',
    // etc.
  ],
});

console.log('✅ Build complete');
```

```powershell
bun run build.ts
node dist/cli.js --version
```

> ⚠️ **注意**：Bun for Windows 仍在快速迭代中，`bun:bundle` 的 `feature()` 宏支持可能需要特定版本才能正常工作。

### 14.3 Bun compile 生成独立可执行文件

```powershell
# 生成 Windows 原生 .exe（不依赖 Node.js/Bun 运行时）
bun build src/entrypoints/cli.tsx --compile --outfile claude.exe
```

---

## 附录 A：完整文件创建清单

| 文件 | 用途 | 必须？ |
|------|------|--------|
| `package.json` | 项目依赖 & 脚本 | ✅ |
| `tsconfig.json` | TypeScript 编译配置 | ✅ |
| `src/shims/bun-bundle.ts` | `feature()` 函数运行时替身 | ✅ |
| `src/shims/macros.ts` | `MACRO.*` 全局常量注入 | ✅ |
| `src/shims/react-compiler-runtime.ts` | React Compiler 运行时 shim | ⚠️ 视情况 |
| `scripts/replace-macros.mjs` | 预处理脚本（替换 bun:bundle 导入） | ✅ |
| `.vscode/launch.json` | VS Code 调试配置 | 可选 |

## 附录 B：快速启动脚本

创建 `start.ps1` 一键启动脚本：

```powershell
#!/usr/bin/env pwsh
# Claude Code 源码快速启动脚本

$ErrorActionPreference = "Stop"

# 检查环境
if (-not (Get-Command node -ErrorAction SilentlyContinue)) {
    Write-Host "❌ 未找到 Node.js，请先安装" -ForegroundColor Red
    exit 1
}

$nodeVersion = (node --version).TrimStart('v').Split('.')[0]
if ([int]$nodeVersion -lt 18) {
    Write-Host "❌ Node.js 版本需要 >= 18，当前: $(node --version)" -ForegroundColor Red
    exit 1
}

# 检查 API Key
if (-not $env:ANTHROPIC_API_KEY) {
    Write-Host "⚠️  未设置 ANTHROPIC_API_KEY" -ForegroundColor Yellow
    $key = Read-Host "请输入你的 Anthropic API Key"
    $env:ANTHROPIC_API_KEY = $key
}

# 检查 tsx
if (-not (Get-Command npx -ErrorAction SilentlyContinue)) {
    Write-Host "❌ 未找到 npx" -ForegroundColor Red
    exit 1
}

# 检查是否已运行预处理
$bunBundleCount = (Select-String -Path "src\entrypoints\cli.tsx" -Pattern "bun:bundle" -Quiet)
if ($bunBundleCount) {
    Write-Host "🔧 检测到未处理的 bun:bundle 导入，执行预处理..." -ForegroundColor Cyan
    node scripts/replace-macros.mjs
}

# 运行
Write-Host "🚀 启动 Claude Code..." -ForegroundColor Green
npx tsx src/entrypoints/cli.tsx @args
```

使用方式：
```powershell
.\start.ps1              # 交互模式
.\start.ps1 --version    # 查看版本
.\start.ps1 -p "Hello"   # 单次提问
```

---

## 附录 C：推荐学习路径

如果你是为了**研究源码**而非运行：

1. **先装官方包**（路线一），确保你有一个能跑的基准
2. **对比阅读**：`cli.js`（编译后单文件）vs `src/`（源码）
3. **聚焦模块**：
   - 想看 Agent 循环 → `src/query.ts` + `src/QueryEngine.ts`
   - 想看工具系统 → `src/Tool.ts` + `src/tools/`
   - 想看 API 调用 → `src/services/api/claude.ts`
   - 想看命令系统 → `src/commands.ts` + `src/commands/`
   - 想看 UI 渲染 → `src/ink/` + `src/components/`
4. **不必编译**也能阅读理解——TypeScript 源码可读性非常高

---

> **总结**：推荐 **`tsx` + shim 文件** 的方案作为 Windows 上最快的源码运行路径。`tsx` 无需编译步骤，直接运行 TypeScript，配合 `bun-bundle` shim 和 `MACRO` 全局注入，即可绕过所有 Bun 专有构建依赖。完整的 `tsc` 编译则适合生产构建或需要优化启动速度的场景。
