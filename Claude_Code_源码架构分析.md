# Claude Code v2.1.88 源码逻辑结构分析文档

> **项目**: `@anthropic-ai/claude-code` 2.1.88 源码还原  
> **技术栈**: TypeScript + React/Ink (终端UI) + Bun (运行时/打包)  
> **分析日期**: 2026年4月8日

---

## 目录

- [一、全局架构总览](#一全局架构总览)
- [二、核心模块详解](#二核心模块详解)
  - [1. 入口与启动流程](#1️⃣-入口与启动流程)
  - [2. 状态管理系统](#2️⃣-状态管理系统)
  - [3. AI 对话核心循环](#3️⃣-ai-对话核心循环)
  - [4. 工具系统](#4️⃣-工具系统-40-工具)
  - [5. 命令系统](#5️⃣-命令系统-60-斜杠命令)
  - [6. 服务层](#6️⃣-服务层)
  - [7. UI 层](#7️⃣-ui-层)
  - [8. 特殊子系统](#8️⃣-特殊子系统)
- [三、数据流全景](#三数据流全景)
- [四、关键设计模式](#四关键设计模式)
- [五、目录结构速查](#五目录结构速查)

---

## 一、全局架构总览

Claude Code 是 Anthropic 出品的 CLI AI 编程助手。整体架构是一个 **事件驱动的对话循环**，围绕"用户输入 → API 调用 → 工具执行 → 结果返回"的主循环设计。

```
┌─────────────────────────────────────────────────────────────────────┐
│                         CLI 入口层                                  │
│  entrypoints/cli.tsx → main.tsx (Commander 参数解析)                │
│       ↓ fast-path: --version / --dump-system-prompt / --mcp        │
│       ↓ normal-path: init() → 启动 REPL 或 Headless/SDK 模式       │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
         ┌──────────────────────┼──────────────────────┐
         ▼                      ▼                      ▼
┌─────────────────┐  ┌──────────────────┐  ┌────────────────────┐
│  交互模式 (REPL) │  │  Headless / SDK  │  │  MCP Server 模式   │
│  replLauncher   │  │  QueryEngine     │  │  entrypoints/mcp   │
│  → ReplScreen   │  │  (编程式调用)     │  │  (MCP 协议服务端)   │
└────────┬────────┘  └────────┬─────────┘  └────────────────────┘
         │                    │
         └────────┬───────────┘
                  ▼
       ┌─────────────────────┐
       │   query() 核心循环   │  ← 所有模式共享的 AI 对话引擎
       │   (query.ts ~1730行) │
       └──────────┬──────────┘
                  │
    ┌─────────────┼─────────────┬────────────────┐
    ▼             ▼             ▼                ▼
┌─────────┐ ┌──────────┐ ┌──────────┐   ┌──────────────┐
│Claude   │ │  Tools   │ │  Auto    │   │  Slash       │
│API 调用  │ │  执行引擎  │ │  Compact │   │  Commands    │
│(claude  │ │(toolExec)│ │  压缩引擎  │   │  命令处理     │
│ .ts)    │ │          │ │          │   │              │
└─────────┘ └────┬─────┘ └──────────┘   └──────────────┘
                 │
    ┌────────────┼─────────────────┐
    ▼            ▼                 ▼
┌────────┐  ┌──────────┐   ┌──────────────┐
│文件操作  │  │Shell执行  │   │多Agent/Team  │
│工具群   │  │工具群     │   │工具群         │
└────────┘  └──────────┘   └──────┬───────┘
                                  │
                           ┌──────┴───────┐
                           │  Task System │
                           │  后台任务系统  │
                           └──────────────┘
```

### 三种运行模式

| 模式 | 入口 | 说明 |
|------|------|------|
| **交互模式 (REPL)** | `replLauncher.tsx` → `ReplScreen` | 终端交互式对话，React/Ink 渲染 UI |
| **Headless / SDK** | `QueryEngine.ts` | 编程式调用，无 UI，适合自动化集成 |
| **MCP Server** | `entrypoints/mcp.ts` | 作为 MCP 协议服务端对外提供能力 |

---

## 二、核心模块详解

### 1️⃣ 入口与启动流程

```
entrypoints/cli.tsx          ← Bun 编译入口，fast-path 快速响应
        │
        ▼
    main.tsx (~4684行)       ← Commander 解析所有 CLI 参数
        │
        ├──→ entrypoints/init.ts  ← 初始化：配置、TLS、代理、遥测、graceful shutdown
        │
        ├──→ setup.ts             ← 运行时设置：Git worktree、hooks 快照、UDS server
        │
        ├──→ replLauncher.tsx     ← 启动 Ink React 渲染，挂载 ReplScreen
        │
        └──→ QueryEngine.ts      ← SDK/headless 模式入口
```

**关键文件说明**：

| 文件 | 行数 | 职责 |
|------|------|------|
| `entrypoints/cli.tsx` | ~303 | CLI 最外层入口，fast-path 拦截（`--version`、`--mcp` 等） |
| `main.tsx` | ~4684 | Commander 参数定义、子命令路由、初始化编排 |
| `entrypoints/init.ts` | ~341 | 配置加载、环境变量、TLS/mTLS、代理、遥测、graceful shutdown |
| `setup.ts` | - | Git worktree 创建、hooks 配置快照、UDS 消息服务器 |
| `replLauncher.tsx` | - | 启动 Ink React 渲染树，挂载 ReplScreen 组件 |

**启动性能优化设计**：
- 入口文件顶部立即并行启动 3 个预取操作：
  - `startMdmRawRead()` — MDM 子进程预读（plutil/reg query）
  - `startKeychainPrefetch()` — macOS Keychain 预取（OAuth + 旧版 API key）
  - `profileCheckpoint()` — 启动性能打点
- `--version` 快速路径零模块加载，直接输出版本号返回

---

### 2️⃣ 状态管理系统

```
state/
  ├── store.ts              ← 自研轻量 Store（get/set/subscribe ~30行）
  ├── AppState.tsx           ← React Context Provider + useAppState hook
  └── appStateDefinition.ts  ← AppState 巨型类型定义 (~570行)
```

**Store 架构**（类 Zustand 极简设计）：

```typescript
// store.ts 核心 ~30 行
createStore<T>() → { get(), set(partial), subscribe(selector, callback) }
// 基于 lodash/isEqual 做浅比较，防止多余 React 渲染
```

**AppState 核心字段**：

| 字段分类 | 关键字段 | 说明 |
|---------|---------|------|
| **权限** | `permissionContext` | 权限上下文 |
| **模型** | `mainLoopModel` | 当前使用的 AI 模型 |
| **MCP** | `mcpClients`, `mcpTools`, `mcpCommands` | MCP 连接、工具、命令 |
| **后台任务** | `backgroundTasks` | 后台任务映射表 |
| **插件** | `plugins` | 插件加载状态 |
| **推测执行** | `speculativeExecution` | 推测执行状态 |
| **远程** | `replBridge*` | Bridge 远程控制状态 |
| **多Agent** | `swarmContext` | 多Agent (Swarm) 协作上下文 |
| **待办** | `todoList` | 待办列表 |
| **通知** | `inbox` | 消息收件箱 |
| **语音** | `voiceMode` | 语音模式状态 |

**连接关系**：`AppState` 是全局单一数据源，所有 UI 组件和业务逻辑都通过 `useAppState()` / `store.get()` 访问。

---

### 3️⃣ AI 对话核心循环

这是整个系统最核心的部分，负责管理与 Claude API 的完整对话生命周期。

```
QueryEngine.ts (~1296行)        query.ts (~1730行)
┌───────────────────┐          ┌─────────────────────────────────┐
│  QueryEngine 类    │          │  query() 异步生成器              │
│                   │  调用     │                                 │
│ .processInput()───┼─────────→│ 1. 消息规范化                    │
│  (用户输入→模型输出  │          │ 2. 构建 system prompt            │
│   →工具调用→循环)   │          │ 3. 调用 Claude API (流式)         │
│                   │          │ 4. yield 流式 token               │
│ 管理:             │          │ 5. 工具调用 → toolExecutor 执行    │
│  - fileStateCache │          │ 6. 自动压缩检查                   │
│  - tokenUsage     │          │ 7. max_output_tokens 恢复重试     │
│  - permDenials    │          │ 8. Token 预算管理                 │
│  - structuredJSON │          │ 9. prompt_too_long 错误恢复       │
└───────────────────┘          └─────────────────────────────────┘
```

#### QueryEngine（对话管理器）

- 每个 conversation 创建一个 `QueryEngine` 实例
- `processInput()` 是异步生成器，处理完整的用户输入 → 模型回复 → 工具调用 → 结果反馈循环
- 管理：文件状态缓存（`fileStateCache`）、token 用量、权限拒绝追踪
- 支持结构化输出（JSON Schema 强制执行）

#### query()（单轮查询执行）

- 核心中的核心——实际执行 API 调用的异步生成器函数
- 处理流程：消息规范化 → API 调用 → 流式处理 → 工具执行 → 自动压缩
- `max_output_tokens` 恢复循环（最多 3 次重试）
- Token 预算管理（+500K 自动续接）
- 反应式压缩 + 上下文折叠

---

### 4️⃣ 工具系统 (40+ 工具)

#### 类型定义 (`Tool.ts` ~793行)

```typescript
interface Tool {
  name: string
  inputSchema: ToolInputJSONSchema
  call(input, context: ToolUseContext): Promise<ToolResult>
  prompt?: string          // 注入到 system prompt 的工具说明
  isReadOnly?: boolean     // 是否为只读工具
  isEnabled?(ctx): boolean // 动态启用/禁用
  // ...
}
```

**`ToolUseContext`** 是一个巨型接口，为工具执行提供完整上下文：
- 所有工具/命令/MCP 客户端引用
- abort 控制器、文件状态缓存
- 应用状态存取函数
- JSX 渲染回调、通知系统
- 子代理上下文、嵌套记忆
- 权限系统回调

#### 工具注册中心 (`tools.ts` ~390行)

| 函数 | 说明 |
|------|------|
| `getTools()` | 返回所有内建工具（含条件编译） |
| `getToolsForPermMode()` | 按权限模式过滤工具 |
| `getAllTools()` | 合并内建工具 + MCP 工具，去重 |

#### 工具分类清单

| 类别 | 工具 | 功能 |
|------|------|------|
| **📁 文件操作** | `FileReadTool` | 读取文件 |
| | `FileEditTool` | 编辑文件（查找替换） |
| | `FileWriteTool` | 写入新文件 |
| | `GlobTool` | 文件名搜索 |
| | `GrepTool` | 文件内容搜索 |
| | `NotebookEditTool` | Notebook 编辑 |
| **🖥️ 命令执行** | `BashTool` | Bash 命令执行 |
| | `PowerShellTool` | PowerShell (Windows) |
| | `REPLTool` | REPL 执行 (ant-only) |
| **🌐 Web/搜索** | `WebFetchTool` | 网页内容抓取 |
| | `WebSearchTool` | 网络搜索 |
| | `WebBrowserTool` | 浏览器控制 |
| **🤖 多Agent/团队** | `AgentTool` | 子代理任务派发 |
| | `TeamCreateTool` | 创建团队成员 |
| | `TeamDeleteTool` | 删除团队成员 |
| | `SendMessageTool` | 团队消息发送 |
| **📋 任务管理** | `TaskCreateTool` | 创建后台任务 |
| | `TaskGetTool` | 获取任务详情 |
| | `TaskListTool` | 列出所有任务 |
| | `TaskUpdateTool` | 更新任务状态 |
| | `TaskStopTool` | 停止任务 |
| | `TaskOutputTool` | 获取任务输出 |
| **🔗 MCP 集成** | `MCPTool` | MCP 工具代理执行 |
| | `ListMcpResourcesTool` | 列出 MCP 资源 |
| | `ReadMcpResourceTool` | 读取 MCP 资源 |
| **📐 规划模式** | `EnterPlanModeTool` | 进入规划模式 |
| | `ExitPlanModeTool` | 退出规划模式 |
| | `EnterWorktreeTool` | 进入 Git 工作树 |
| | `ExitWorktreeTool` | 退出 Git 工作树 |
| **🎯 其他** | `SkillTool` | 技能调用 |
| | `AskUserQuestionTool` | 询问用户 |
| | `TodoWriteTool` | 待办事项写入 |
| | `ConfigTool` | 配置管理 |
| | `LSPTool` | 语言服务协议集成 |
| | `BriefTool` | 对话摘要 |
| | `SleepTool` | 定时等待 |
| | `ToolSearchTool` | 工具搜索 |
| | `TungstenTool` | 代码分析工具 |

---

### 5️⃣ 命令系统 (60+ 斜杠命令)

#### 注册中心 (`commands.ts` ~755行)

命令分为两种类型：

```typescript
// 本地命令 — 在客户端直接执行
interface LocalCommand {
  type: 'local'
  call(args, context): Promise<LocalCommandResult>
}

// Prompt 命令 — 注入 AI prompt，让 Claude 处理（技能式命令）
interface PromptCommand {
  type: 'prompt'
  promptContent: string
  allowedTools?: string[]
}
```

#### 命令分类

| 分类 | 命令 | 说明 |
|------|------|------|
| **📝 Git** | `/commit` | 生成 commit 消息并提交 |
| | `/commit-push-pr` | 提交 + 推送 + 创建 PR |
| | `/review` | 代码审查 |
| | `/diff` | 查看差异 |
| **⚙️ 配置** | `/config` | 配置管理 |
| | `/skills` | 技能管理 |
| | `/mcp` | MCP 服务器管理 |
| | `/keybindings` | 快捷键 |
| **💬 会话** | `/resume` | 恢复会话 |
| | `/session` | 会话管理 |
| | `/compact` | 手动压缩对话 |
| | `/clear` | 清除对话 |
| **🔐 认证** | `/login` | 登录 |
| | `/logout` | 登出 |
| **🎨 UI** | `/theme` | 主题切换 |
| | `/color` | 颜色设置 |
| | `/vim` | Vim 模式 |
| | `/status` | 状态栏 |
| **📦 扩展** | `/install` | 安装插件 |
| | `/desktop` | 桌面版 |
| | `/mobile` | 移动端 |
| **🌉 远程** | `/bridge` | 远程桥接 |
| | `/teleport` | 会话传送 |
| **🔍 分析** | `/doctor` | 诊断检查 |
| | `/cost` | 费用查看 |
| | `/usage` | 用量统计 |
| **🐛 调试** | `/bughunter` | Bug 猎手 |
| | `/security-review` | 安全审查 |
| **📋 任务** | `/tasks` | 任务管理 |
| | `/ultraplan` | 超级规划 |
| **🔧 其他** | `/help`, `/copy`, `/share`, `/init`, `/memory`, `/rename`... | 杂项 |

#### Feature-gated 命令（条件编译）

通过 `feature()` 宏在编译期开关：

| 特性开关 | 命令 |
|---------|------|
| `KAIROS` | `/assistant`, `/proactive`, `/brief` |
| `BRIDGE_MODE` | `/bridge` |
| `VOICE_MODE` | `/voice` |
| `DAEMON` | `/remote-control-server` |
| `WORKFLOW_SCRIPTS` | `/workflows` |
| `HISTORY_SNIP` | `/force-snip` |

---

### 6️⃣ 服务层

```
services/
  ├── api/              ← Claude API 客户端
  ├── mcp/              ← MCP 协议完整实现
  ├── compact/          ← 对话压缩引擎
  ├── toolExecution/    ← 工具执行编排
  ├── analytics/        ← 分析与遥测
  ├── oauth/            ← OAuth 认证
  ├── plugins/          ← 插件市场
  ├── policyLimits/     ← 企业策略
  ├── remoteManagedSettings/ ← 远程托管设置
  ├── lsp/              ← LSP 语言服务
  ├── SessionMemory/    ← 会话记忆
  ├── voiceRecorder/    ← 语音录制
  └── tips/             ← 提示建议
```

#### 6.1 API 服务 (`services/api/`)

| 文件 | 行数 | 说明 |
|------|------|------|
| `claude.ts` | ~3420 | **API 调用核心**：流式请求、token 计数、模型选择、beta 参数、工具 schema 转换 |
| `errors.ts` | - | 错误分类（`prompt_too_long` 等） |
| `retryBackoff.ts` | - | 重试/回退策略 |
| `bootstrap.ts` | - | 启动数据预取 |
| `filesApi.ts` | - | 文件上传/下载 |

#### 6.2 MCP 服务 (`services/mcp/`, 24个文件)

**Model Context Protocol 的完整客户端实现**：

| 文件 | 说明 |
|------|------|
| `types.ts` | MCP 配置 schema（支持 stdio/sse/http/ws/sdk 传输） |
| `client.ts` | MCP 客户端管理 |
| `McpConnection.tsx` | React 组件式连接管理 |
| `channelPermissions.ts` | 权限处理 |
| `elicitationHandler.ts` | 交互处理 |
| `officialRegistry.ts` | 官方 MCP 服务器注册表 |

#### 6.3 压缩服务 (`services/compact/`)

| 文件 | 说明 |
|------|------|
| `autoCompact.ts` | 自动压缩（上下文窗口超阈值时触发） |
| `compact.ts` | 对话压缩核心算法 |
| `microCompact.ts` | 微型压缩（轻量级） |
| `sessionMemoryCompact.ts` | 会话记忆压缩 |

#### 6.4 工具执行编排 (`services/toolExecution/`)

| 文件 | 说明 |
|------|------|
| `toolExecutor.ts` | **流式工具并发执行器**：并发安全的工具并行执行，非并发的独占执行 |
| `toolOrchestration.ts` | 工具编排逻辑 |
| `toolHooks.ts` | 工具生命周期钩子（PreToolUse / PostToolUse） |

#### 6.5 分析服务 (`services/analytics/`)

| 文件 | 说明 |
|------|------|
| `index.ts` | 事件日志 API（零依赖设计，启动前队列缓冲） |
| `growthbook.ts` | GrowthBook A/B 测试 / 特性门控 |
| `datadog.ts` | Datadog 监控集成 |
| `sink.ts`, `metadata.ts` | 事件路由与元数据 |

---

### 7️⃣ UI 层

#### 屏幕 (`screens/`)

| 文件 | 行数 | 说明 |
|------|------|------|
| `ReplScreen.tsx` | ~5006 | **项目最大文件** — 主交互界面，编排所有组件、管理对话循环 |
| `Doctor.tsx` | - | 诊断屏幕 |
| `ResumeConversation.tsx` | - | 会话恢复屏幕 |

#### 组件 (`components/`, 150+ 文件)

```
components/
  ├── 🏗️ 框架组件
  │   ├── CoreApp.tsx          ← 顶层: FPS → Stats → AppState Provider
  │   ├── MessageList.tsx      ← 消息列表 (虚拟滚动)
  │   ├── VirtualMessageList   ← 虚拟滚动实现
  │   └── InputPrompt.tsx      ← 输入框 (Vim/补全/语音)
  │
  ├── 🔒 权限组件
  │   └── PermissionRequest.tsx← 权限请求对话框
  │
  ├── 📊 显示组件
  │   ├── DiffView.tsx         ← 差异显示
  │   ├── StructuredDiff.tsx   ← 结构化 diff 渲染
  │   ├── Spinner.tsx          ← 加载动画
  │   └── StatusLine.tsx       ← 状态栏
  │
  ├── 🎛️ 功能组件
  │   ├── MCP/                 ← MCP 审批/导入对话框
  │   ├── Swarm/               ← 多 Agent UI
  │   ├── Skills/              ← 技能 UI
  │   ├── Tasks/               ← 任务面板
  │   ├── Settings/            ← 设置界面
  │   └── HelpV2/              ← 帮助界面
  │
  └── 🧱 基础组件
      └── designSystem/        ← 设计系统基础组件
```

#### Hooks (`hooks/`, 80+ 文件)

| Hook | 说明 |
|------|------|
| `useCanUseTool.ts` | **核心** — 工具权限检查门控 |
| `useMergedTools.ts` | 合并内置 + MCP + 插件工具 |
| `useMergedCommands.ts` | 合并内置 + MCP + 插件命令 |
| `useReplBridge.ts` | Bridge 远程控制连接管理 |
| `useVoiceMode.ts` | 语音模式集成 |
| `useSwarmInit.ts` | 多 Agent (Swarm) 初始化 |
| `useMessageQueue.ts` | 消息队列处理 |
| `useSettings.ts` | 设置变更监听 |
| `useTerminalSize.ts` | 终端尺寸监听 |
| `useFileHistory.ts` | 文件历史快照 |
| `useVimInput.ts` | Vim 模式输入处理 |
| `useExitOnCtrlCD.ts` | Ctrl+C/D 退出处理 |
| `useApiKeyValidation.ts` | API Key 验证 |
| `useDiffData.ts` | 差异数据处理 |
| `useScheduledTasks.ts` | 定时任务 |

#### React 上下文 (`context/`)

| 文件 | 说明 |
|------|------|
| `fpsMetrics.tsx` | FPS 性能指标 |
| `mailbox.tsx` | 邮箱消息上下文（团队通信） |
| `notifications.tsx` | 通知队列 |
| `voiceContext.tsx` | 语音模式 Provider |
| `stats.tsx` | 统计数据 |
| `overlayContext.tsx` | 覆盖层管理 |
| `modalContext.tsx` | 模态对话框管理 |
| `QueuedMessageContext.tsx` | 消息队列上下文 |

---

### 8️⃣ 特殊子系统

#### 8.1 Bridge 远程控制系统 (`bridge/`, ~31文件)

**功能**：将本地 Claude Code 暴露到 claude.ai 网页端，实现远程控制。

```
claude.ai 网页端
       │
       │  JWT / 心跳 / 轮询
       ▼
┌──────────────────────────┐
│  bridgeMain.ts (~3000行)  │
│                          │
│  1. 注册环境到 claude.ai  │
│  2. 轮询接收工作          │
│  3. Spawn 子进程执行      │
│  4. 结果回传              │
│  5. 心跳保活              │
└──────────────────────────┘
       │
       ▼
  本地 Claude Code 子进程
```

| 文件 | 说明 |
|------|------|
| `types.ts` (~263行) | 协议类型：`WorkSecret`、`BridgeConfig`、`BridgeSpawnMode` |
| `bridgeMain.ts` (~3000行) | Bridge 主循环：注册→轮询→派发→心跳→清理 |
| `bridgeApi.ts` | API 客户端 |
| `bridgeMessaging.ts` | 消息传输层 |
| `bridgePermissionCallbacks.ts` | 权限回调 |
| `replBridge.ts` | REPL 内的 Bridge 集成 |
| `sessionRunner.ts` | 会话进程管理 |
| `jwtUtils.ts` | JWT 令牌刷新 |
| `trustedDevice.ts` | 可信设备令牌 |

**三种 Spawn 模式**：
- `single-session` — 单会话
- `worktree` — Git 工作树隔离
- `same-dir` — 同目录

#### 8.2 Coordinator 协调器模式 (`coordinator/`)

**功能**：让 Claude 充当任务分配者，将工作委派给 Worker 子代理。

```
┌──────────────────────────────┐
│  主 Agent (协调者)             │
│  - 分析任务、制定计划           │
│  - 通过 AgentTool 派发子任务   │
└──────────┬───────────────────┘
           │  派发
    ┌──────┼──────┐
    ▼      ▼      ▼
┌──────┐┌──────┐┌──────┐
│Worker││Worker││Worker│   ← 受限工具集
│  1   ││  2   ││  3   │   (Bash, FileRead, FileEdit...)
└──────┘└──────┘└──────┘
```

- 通过 `CLAUDE_CODE_COORDINATOR_MODE` 环境变量激活
- Worker 只能使用受限工具集
- 支持 session 恢复时的模式匹配切换

#### 8.3 Task 后台任务系统 (`Task.ts` + `tasks/`)

```typescript
// 任务类型
type TaskType = 'bash' | 'agent' | 'remote_agent' | 'teammate' | 'dream' | 'cron'

// 任务状态
type TaskStatus = 'queued' → 'running' → 'completed' | 'failed' | 'killed'

// 任务 ID 前缀区分类型
// bsh_xxx = bash, agt_xxx = agent, tm_xxx = teammate
```

| 实现 | 说明 |
|------|------|
| `LocalShellTask/` | 本地 Shell 命令任务 |
| `LocalAgentTask/` | 本地 Agent 子任务 |
| `RemoteAgentTask/` | 远程 Agent 任务 |
| `InProcessTeammateTask/` | 进程内团队成员 |
| `DreamTask/` | "Dream" 自主探索模式 |

#### 8.4 Skills 技能系统 (`skills/`)

**功能**：将 Markdown 文件转化为可调用的 AI 技能（slash commands）。

```
技能来源（多层级）
  ├── 用户级: ~/.claude/skills/*.md
  ├── 项目级: .claude/skills/*.md
  ├── 内建: skills/bundled/ (17个)
  └── MCP: mcpSkillBuilders.ts
         │
         ▼
  frontmatter 解析
  (name, description, allowedTools, model, effort, hooks...)
         │
         ▼
  注册为 PromptCommand → SkillTool 触发执行
```

**内建技能**：`debug`, `verify`, `loop`, `remember`, `simplify` 等 17 个

#### 8.5 Plugins 插件系统 (`plugins/`)

```
plugins/
  ├── bundled/
  │   └── index.ts          ← 内建插件注册 (PluginManifest 格式)
  └── init.ts               ← 插件初始化入口
```

- 插件 ID 格式：`publisher/plugin-name`
- 可提供：skills + hooks + MCP servers
- 通过 `/install` 命令安装，设置界面启用/禁用
- AppState 中 `plugins` 字段跟踪完整状态

#### 8.6 Voice 语音系统 (`voice/` + `services/voiceRecorder/`)

```
门控层: GrowthBook kill-switch + OAuth 认证检查
            │
            ▼
录制层 (降级链):
  1. Native cpal (macOS/Linux/Windows) ← 首选
  2. SoX rec                           ← 后备
  3. ALSA arecord                      ← Linux 后备
            │
            ▼
转写层: STT 流式转录 → 文本注入 InputPrompt
```

#### 8.7 Vim 模式 (`vim/`)

完整的 Vim 模式状态机实现：

| 文件 | 说明 |
|------|------|
| `types.ts` (~200行) | 类型定义：INSERT / NORMAL 模式、操作符状态机 |
| `motions.ts` | 移动命令（w/e/b/h/l/0/$ 等） |
| `operators.ts` | 操作符（d/c/y/>/<  等） |
| `textObjects.ts` | 文本对象（iw/aw/i"/a( 等） |
| `transitions.ts` | 状态转换逻辑 |

#### 8.8 类型系统 (`types/`)

| 文件 | 行数 | 说明 |
|------|------|------|
| `permissions.ts` | ~442 | 权限系统完整类型：6种模式 × 3种行为 × 匹配规则 |
| `hooks.ts` | ~291 | Hook 事件类型、Prompt 协议、返回值 schema |
| `commands.ts` | - | 命令类型（LocalCommand / PromptCommand） |
| `ids.ts` | - | 类型安全的 branded ID types |
| `plugins.ts` | - | 插件清单和加载类型 |
| `message.ts` | - | 消息类型定义 |
| `textInputTypes.ts` | - | 输入模式类型 |

#### 8.9 Server 直连服务器 (`server/`)

| 文件 | 说明 |
|------|------|
| `types.ts` | 配置类型（端口、host、auth、idle 超时） |
| `serverClient.ts` | WebSocket 客户端管理器 |
| `createSession.ts` | HTTP 会话创建 |

---

## 三、数据流全景

```
用户输入 (键盘 / 语音 / Bridge 远程)
     │
     ▼
┌──────────────────┐     ┌───────────────────────┐
│  InputPrompt     │────→│  Slash Command?       │
│  (React 组件)     │     │  ┌─ Yes → commands.ts │
└──────────────────┘     │  └─ No  ↓             │
                         └───────┬───────────────┘
                                 ▼
                    ┌────────────────────────┐
                    │  context.ts            │
                    │  构建系统/用户上下文      │
                    │  ┌─ getSystemContext()  │
                    │  │  (Git 状态快照)      │
                    │  └─ getUserContext()    │
                    │     (CLAUDE.md + 日期)  │
                    └───────────┬────────────┘
                                ▼
                    ┌────────────────────────┐
                    │  query() 核心循环       │
                    │  1. System Prompt 组装  │
                    │  2. Claude API 流式调用  │
                    │  3. 解析 tool_use 块    │
                    └───────────┬────────────┘
                                ▼
                    ┌────────────────────────┐
                    │  toolExecutor          │
                    │  1. 并行/串行工具执行    │
                    │  2. 权限检查            │
                    │     (useCanUseTool)    │
                    │  3. Tool.call() 执行   │
                    └───────────┬────────────┘
                                ▼
                    ┌────────────────────────┐
                    │  结果处理               │
                    │  1. 追加到消息历史       │
                    │  2. 检查是否需要压缩     │
                    │  3. 继续下一轮 or 结束   │
                    └───────────┬────────────┘
                                ▼
                    ┌────────────────────────┐
                    │  UI 渲染               │
                    │  ReplScreen → 组件树    │
                    │  MessageList 展示消息   │
                    │  Spinner 展示工具进度   │
                    └────────────────────────┘
```

---

## 四、关键设计模式

| 设计模式 | 应用位置 | 说明 |
|---------|---------|------|
| **编译期特性消除 (DCE)** | `feature('BRIDGE_MODE')` 等 | Bun 打包时裁剪 ant-only 代码，减小外部构建体积 |
| **懒加载/动态导入** | `require()` / `import()` | 打破循环依赖 + 减少启动时模块加载开销 |
| **异步生成器** | `query()` | `yield` 流式返回 AI 响应 token，支持实时渲染 |
| **极简 Store** | `state/store.ts` | 30行实现 get/set/subscribe，配合 `lodash/isEqual` 防重渲染 |
| **Memoize + 缓存失效** | `context.ts` | 上下文 memoize 缓存 + 提供手动 clear 能力 |
| **权限分层** | `types/permissions.ts` | 来源(6级) × 行为(allow/deny/ask) × 匹配规则 |
| **并行预取** | 入口顶部 | MDM / Keychain / API / MCP 同时预取，优化冷启动 |
| **降级链** | 语音录制 | Native → SoX → ALSA 逐级降级，保证跨平台兼容 |
| **条件编译** | 全局 | `feature()` + `process.env.USER_TYPE` 实现代码裁剪 |
| **类型即文档** | `types/` 目录 | 类型定义本身就是完整的架构文档 |

---

## 五、目录结构速查

```
src/
├── main.tsx                    # 主入口 (~4684行)，Commander 参数解析
├── commands.ts                 # 命令注册表 (~755行)
├── tools.ts                    # 工具注册表 (~390行)
├── Tool.ts                     # 工具类型定义 (~793行)
├── query.ts                    # 查询核心循环 (~1730行)
├── QueryEngine.ts              # 查询引擎类 (~1296行)
├── context.ts                  # 上下文构建 (~190行)
├── setup.ts                    # 运行时设置
├── replLauncher.tsx            # REPL 启动器
├── Task.ts                     # 任务类型定义
├── tasks.ts                    # 任务管理
├── history.ts                  # 会话历史
├── ink.ts                      # Ink 渲染基础
│
├── entrypoints/                # 入口点
│   ├── cli.tsx                 #   CLI 入口 (~303行)
│   ├── init.ts                 #   初始化 (~341行)
│   ├── mcp.ts                  #   MCP Server 入口
│   └── sdk/                    #   SDK 入口
│
├── state/                      # 状态管理
│   ├── store.ts                #   极简 Store (~30行)
│   ├── AppState.tsx            #   React Context Provider
│   └── appStateDefinition.ts   #   AppState 类型 (~570行)
│
├── tools/                      # 40+ 工具实现
│   ├── BashTool/
│   ├── FileEditTool/
│   ├── FileReadTool/
│   ├── AgentTool/
│   ├── TaskCreateTool/
│   └── ...
│
├── commands/                   # 60+ 斜杠命令
│   ├── commit.ts
│   ├── review.ts
│   ├── mcp/
│   ├── config/
│   └── ...
│
├── services/                   # 服务层
│   ├── api/                    #   Claude API 客户端
│   ├── mcp/                    #   MCP 协议实现 (24文件)
│   ├── compact/                #   对话压缩
│   ├── toolExecution/          #   工具执行编排
│   ├── analytics/              #   分析遥测
│   ├── oauth/                  #   OAuth 认证
│   ├── plugins/                #   插件市场
│   └── voiceRecorder/          #   语音录制
│
├── components/                 # 150+ React/Ink UI 组件
│   ├── CoreApp.tsx
│   ├── MessageList.tsx
│   ├── InputPrompt.tsx
│   ├── PermissionRequest.tsx
│   └── ...
│
├── hooks/                      # 80+ React Hooks
│   ├── useCanUseTool.ts
│   ├── useMergedTools.ts
│   ├── useReplBridge.ts
│   └── ...
│
├── screens/                    # 顶级屏幕
│   └── ReplScreen.tsx          #   主交互界面 (~5006行)
│
├── bridge/                     # 远程控制桥接 (31文件)
│   ├── types.ts
│   ├── bridgeMain.ts           #   Bridge 主循环 (~3000行)
│   └── ...
│
├── coordinator/                # 协调器模式
│   └── coordinatorMode.ts
│
├── skills/                     # 技能系统
│   ├── bundled/                #   内建技能 (17个)
│   ├── loadSkills.ts           #   技能加载器 (~1087行)
│   └── mcpSkillBuilders.ts
│
├── plugins/                    # 插件系统
│   └── bundled/
│
├── tasks/                      # 任务实现
│   ├── LocalShellTask/
│   ├── LocalAgentTask/
│   ├── RemoteAgentTask/
│   └── InProcessTeammateTask/
│
├── types/                      # 集中类型定义
│   ├── permissions.ts
│   ├── hooks.ts
│   ├── message.ts
│   └── ...
│
├── context/                    # React Context
├── vim/                        # Vim 模式状态机
├── voice/                      # 语音系统
├── server/                     # 直连服务器
├── assistant/                  # 助手模式 (Kairos)
├── buddy/                      # 伴侣精灵
├── utils/                      # 工具函数集合
├── constants/                  # 常量定义
├── schemas/                    # 数据校验 schema
├── migrations/                 # 数据迁移
├── outputStyles/               # 输出样式
├── keybindings/                # 快捷键
├── ink/                        # Ink 渲染基础设施
├── native-ts/                  # 原生 TypeScript 绑定
├── remote/                     # 远程功能
├── memdir/                     # 内存目录
├── moreright/                  # 扩展权限
├── upstreamproxy/              # 上游代理
└── cli/                        # CLI 输出/IO 处理
```

---

> **文档生成说明**：本文档基于对 Claude Code v2.1.88 源码还原项目的静态分析生成，涵盖了所有主要子系统的架构、职责和互联关系。
