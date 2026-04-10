# Claude Code Agent 子系统实现原理详解

> 基于 Claude Code v2.1.88 源码分析 · 2026-04-09

---

## 目录

1. [架构总览](#1-架构总览)
2. [Agent 类型体系](#2-agent-类型体系)
3. [Agent 定义与加载机制](#3-agent-定义与加载机制)
4. [AgentTool — 核心入口](#4-agenttool--核心入口)
5. [Agent 执行引擎 (runAgent)](#5-agent-执行引擎-runagent)
6. [同步 vs 异步执行模型](#6-同步-vs-异步执行模型)
7. [Fork 子代理机制](#7-fork-子代理机制)
8. [Coordinator 协调者模式](#8-coordinator-协调者模式)
9. [多 Agent 团队 (Swarms)](#9-多-agent-团队-swarms)
10. [工具过滤与权限控制](#10-工具过滤与权限控制)
11. [Agent 记忆与快照系统](#11-agent-记忆与快照系统)
12. [任务管理框架](#12-任务管理框架)
13. [通信与消息路由](#13-通信与消息路由)
14. [模型选择策略](#14-模型选择策略)
15. [Agent 上下文隔离](#15-agent-上下文隔离)
16. [会话恢复 (Resume)](#16-会话恢复-resume)
17. [后台摘要生成](#17-后台摘要生成)
18. [Worktree 隔离](#18-worktree-隔离)
19. [安全与分类](#19-安全与分类)
20. [关键数据流图](#20-关键数据流图)

---

## 1. 架构总览

Claude Code 的 Agent 子系统是一个完整的**多层级自主代理调度框架**，允许主会话（main REPL）将复杂任务委派给子代理执行。其核心设计原则包括：

- **工具即代理**：Agent 对模型暴露为一个工具（`Agent` tool），模型通过工具调用来创建子代理
- **异步优先**：子代理支持前台同步、后台异步、以及运行时前→后台切换
- **缓存敏感**：Fork 机制确保子代理与父代理共享 Anthropic API 的提示缓存
- **类型多态**：支持内置 Agent、自定义 Agent（用户/项目/策略）、插件 Agent 三种来源
- **多级隔离**：通过 AsyncLocalStorage、独立 AbortController、可选的 Git Worktree 实现上下文隔离

### 文件拓扑

```
src/tools/AgentTool/          ← Agent 工具核心
├── AgentTool.tsx             ← Tool 定义 & call() 入口 (1398行)
├── runAgent.ts               ← 执行引擎 (974行)
├── forkSubagent.ts           ← Fork 机制 (211行)
├── agentToolUtils.ts         ← 工具过滤/最终化/异步生命周期 (687行)
├── loadAgentsDir.ts          ← Agent 定义加载 (756行)
├── builtInAgents.ts          ← 内置 Agent 注册表
├── prompt.ts                 ← Agent 工具的提示生成 (288行)
├── resumeAgent.ts            ← 会话恢复
├── agentMemory.ts            ← 持久化记忆
├── agentMemorySnapshot.ts    ← 记忆快照同步
├── agentColorManager.ts      ← 颜色分配
├── agentDisplay.ts           ← 显示工具
├── constants.ts              ← 常量
├── UI.tsx                    ← Ink UI 组件
└── built-in/                 ← 内置 Agent 实现
    ├── generalPurposeAgent.ts
    ├── exploreAgent.ts
    ├── planAgent.ts
    ├── verificationAgent.ts
    ├── claudeCodeGuideAgent.ts
    └── statuslineSetup.ts

src/coordinator/              ← 协调者模式
└── coordinatorMode.ts        ← Coordinator 系统提示 & 工具限制

src/tasks/                    ← 任务管理框架
├── LocalAgentTask/           ← 本地后台 Agent 任务
├── RemoteAgentTask/          ← 远程 CCR Agent 任务
├── InProcessTeammateTask/    ← 进程内 Teammate 任务
└── types.ts                  ← 任务状态联合类型

src/utils/
├── agentContext.ts            ← AsyncLocalStorage 上下文
├── agentSwarmsEnabled.ts      ← Swarm 功能开关
├── agentId.ts                 ← 确定性 Agent ID 生成
├── model/agent.ts             ← 模型选择逻辑
└── forkedAgent.ts             ← Fork 辅助 & 子代理上下文创建

src/tools/SendMessageTool/    ← Agent 间通信
src/services/AgentSummary/    ← 后台摘要
```

---

## 2. Agent 类型体系

Agent 系统使用 TypeScript 联合类型定义三类 Agent，共享 `BaseAgentDefinition` 基础结构：

### 2.1 BaseAgentDefinition（公共字段）

```typescript
type BaseAgentDefinition = {
  agentType: string              // 标识符 (如 "general-purpose", "Explore")
  whenToUse: string              // 工具描述 — 告诉模型何时选用此 Agent
  tools?: string[]               // 工具白名单 (["*"] = 全部)
  disallowedTools?: string[]     // 工具黑名单
  skills?: string[]              // 预加载技能
  mcpServers?: AgentMcpServerSpec[]  // Agent 专属 MCP 服务器
  hooks?: HooksSettings          // 会话级别钩子
  color?: AgentColorName         // UI 颜色标识
  model?: string                 // 模型覆盖 ("inherit" | "sonnet" | ...)
  effort?: EffortValue           // 推理深度
  permissionMode?: PermissionMode // 权限模式
  maxTurns?: number              // 最大执行轮次
  background?: boolean           // 始终后台运行
  memory?: AgentMemoryScope      // 持久化记忆范围
  isolation?: 'worktree' | 'remote'  // 隔离模式
  omitClaudeMd?: boolean         // 省略 CLAUDE.md (只读 Agent 优化)
  // ... 其他
}
```

### 2.2 三种具体类型

| 类型 | TypeScript 类型 | Source 字段 | 系统提示 | 典型来源 |
|------|----------------|------------|---------|---------|
| **内置** | `BuiltInAgentDefinition` | `'built-in'` | 动态函数 `getSystemPrompt({toolUseContext})` | 代码中硬编码 |
| **自定义** | `CustomAgentDefinition` | `'userSettings'` / `'projectSettings'` / `'policySettings'` | 闭包 `getSystemPrompt()` | Markdown/JSON 文件 |
| **插件** | `PluginAgentDefinition` | `'plugin'` | 闭包 `getSystemPrompt()` | 插件系统加载 |

### 2.3 内置 Agent 清单

| Agent | 文件 | 用途 | 模型 | 特点 |
|-------|------|------|------|------|
| **general-purpose** | `generalPurposeAgent.ts` | 通用子代理 | inherit (默认) | 全工具 `['*']` |
| **Explore** | `exploreAgent.ts` | 代码库搜索探索 | haiku (外部) / inherit (内部) | 只读、省略 CLAUDE.md |
| **Plan** | `planAgent.ts` | 架构设计与实现规划 | inherit | 只读、输出"关键文件列表" |
| **Verification** | `verificationAgent.ts` | 验证/破坏性测试 | inherit | 后台运行、对抗性验证 |
| **claude-code-guide** | `claudeCodeGuideAgent.ts` | Claude Code 使用指南 | — | 非 SDK 入口 |
| **statusline-setup** | `statuslineSetup.ts` | 状态栏配置 | — | — |

---

## 3. Agent 定义与加载机制

### 3.1 加载入口 — `loadAgentsDir()`

`loadAgentsDir.ts` 中的 `loadAgentsDir()` 是 memoized 的核心加载函数，它合并所有来源的 Agent 定义：

```
加载流程:
  1. getBuiltInAgents()         → 内置 Agent 列表
  2. loadPluginAgents()         → 插件 Agent
  3. loadMarkdownFilesForSubdir("agents")  → 用户/项目/策略 Markdown Agent
  4. 合并 & 去重 (同名后定义覆盖先定义)
  5. 颜色初始化 (为有 color 属性的 Agent 分配颜色)
  6. Agent 记忆快照初始化
```

### 3.2 优先级（低→高）

```
built-in < plugin < policySettings < projectSettings < userSettings
```

同名 Agent 中高优先级会覆盖低优先级。`agentDisplay.ts` 的 `annotateOverrides()` 负责标注被覆盖的 Agent。

### 3.3 Markdown Agent 解析

Markdown 格式的 Agent 文件通过 frontmatter 定义配置：

```markdown
---
name: my-agent
description: 何时使用此 Agent
model: sonnet
tools: ["Bash", "Read", "Write"]
color: blue
permissionMode: bubble
maxTurns: 50
memory: project
hooks:
  SubagentStop:
    - command: echo "Agent done"
---

这是 Agent 的系统提示正文...
```

支持的 frontmatter 字段通过 `AgentJsonSchema` Zod schema 验证，包括：name、description、color、model、background、memory、isolation、effort、permissionMode、maxTurns、tools、disallowedTools、skills、initialPrompt、mcpServers、hooks。

### 3.4 MCP 服务器规范

Agent 可以声明专属 MCP 服务器，有两种方式：

```typescript
type AgentMcpServerSpec =
  | string                          // 引用已有服务器名 (如 "slack")
  | { [name: string]: McpServerConfig }  // 内联定义
```

内联定义的 MCP 服务器在 Agent 结束时自动清理；引用的共享服务器不会被清理。

---

## 4. AgentTool — 核心入口

`AgentTool.tsx` (1398 行) 是整个 Agent 子系统的入口点，作为 `buildTool()` 构建的标准工具暴露给模型。

### 4.1 输入 Schema

```typescript
const fullInputSchema = z.object({
  description: z.string(),     // 3-5 词任务描述
  prompt: z.string(),          // 完整任务提示
  subagent_type: z.string().optional(),  // Agent 类型选择
  model: z.enum(['sonnet', 'opus', 'haiku']).optional(),
  run_in_background: z.boolean().optional(),
  name: z.string().optional(), // Swarm 中的可寻址名称
  team_name: z.string().optional(),
  mode: permissionModeSchema().optional(),
  isolation: z.enum(['worktree', 'remote']).optional(),
  cwd: z.string().optional(),  // 工作目录覆盖
})
```

Schema 会根据 Feature Flag 动态裁剪：
- `FORK_SUBAGENT` 关闭时：`subagent_type` 为必填
- `KAIROS` 关闭时：移除 `cwd` 字段
- 后台任务禁用时：移除 `run_in_background`

### 4.2 输出 Schema

```typescript
const outputSchema = z.union([
  // 同步完成
  agentToolResultSchema().extend({ status: z.literal('completed'), prompt: z.string() }),
  // 异步启动
  z.object({
    status: z.literal('async_launched'),
    agentId: z.string(),
    description: z.string(),
    prompt: z.string(),
    outputFile: z.string(),
    canReadOutputFile: z.boolean().optional()
  })
])
```

另外还有两种内部专用输出类型（不导出到 schema）：
- `TeammateSpawnedOutput` — Swarm teammate 创建成功
- `RemoteLaunchedOutput` — 远程 CCR Agent 启动

### 4.3 call() 核心路由流程

```
call(input, toolUseContext, canUseTool, assistantMessage, onProgress)
  │
  ├─── team_name && name → spawnTeammate()          [Swarm 路径]
  │
  ├─── subagent_type 未指定 && FORK 开启 → Fork 路径
  │     └── selectedAgent = FORK_AGENT
  │         └── 检查递归 fork 防护
  │
  ├─── subagent_type 指定 → 查找 Agent 定义
  │     └── 权限过滤 → MCP 依赖检查
  │
  ├─── isolation === 'remote' → teleportToRemote()   [远程路径]
  │
  ├─── 构建 systemPrompt + promptMessages
  │     ├── Fork 路径: 继承父系统提示 + buildForkedMessages()
  │     └── 普通路径: Agent 自身系统提示 + 用户消息
  │
  ├─── shouldRunAsync === true → 异步路径
  │     ├── registerAsyncAgent()
  │     ├── void runAsyncAgentLifecycle()  [fire-and-forget]
  │     └── return { status: 'async_launched' }
  │
  └─── shouldRunAsync === false → 同步路径
        ├── registerAgentForeground()
        ├── while(true) { race(nextMessage, backgroundSignal) }
        │     ├── 消息处理: 进度追踪 + UI 更新
        │     └── background 信号: 转为异步继续
        └── return { status: 'completed', ...agentResult }
```

### 4.4 异步触发条件

以下任一条件为 true 时，Agent 以异步模式运行：

```typescript
const shouldRunAsync =
  run_in_background === true       // 用户显式指定
  || selectedAgent.background      // Agent 定义中声明
  || isCoordinator                 // 协调者模式
  || forceAsync                    // FORK_SUBAGENT 实验
  || assistantForceAsync           // KAIROS 助手模式
  || proactiveModule?.isProactiveActive()
```

---

## 5. Agent 执行引擎 (runAgent)

`runAgent.ts` (974 行) 实现了 Agent 的实际执行循环，作为 **AsyncGenerator** 返回消息流。

### 5.1 函数签名

```typescript
async function* runAgent({
  agentDefinition,       // Agent 定义
  promptMessages,        // 初始消息
  toolUseContext,        // 工具执行上下文
  canUseTool,            // 工具权限检查
  isAsync,               // 是否异步
  forkContextMessages,   // Fork 上下文消息 (可选)
  querySource,           // 查询来源标识
  override,              // 覆盖参数 (systemPrompt, abortController, agentId)
  model,                 // 模型覆盖
  maxTurns,              // 最大轮次
  availableTools,        // 预计算的工具池
  allowedTools,          // 工具权限规则
  useExactTools,         // Fork 路径: 使用完全相同的工具集
  worktreePath,          // Worktree 隔离路径
  ...
}): AsyncGenerator<Message, void>
```

### 5.2 执行流程

```
runAgent()
  │
  ├── 1. 模型解析
  │     getAgentModel(定义模型, 主循环模型, 工具指定模型, 权限模式)
  │
  ├── 2. 上下文构建
  │     ├── userContext (可选省略 CLAUDE.md)
  │     ├── systemContext (可选省略 gitStatus)
  │     └── 合并成 agentToolUseContext
  │
  ├── 3. 权限模式覆盖
  │     ├── Agent 定义的 permissionMode
  │     ├── 异步 Agent → shouldAvoidPermissionPrompts
  │     ├── bubble 模式 → 始终显示提示
  │     └── 工具权限范围限定 (allowedTools)
  │
  ├── 4. 工具解析
  │     ├── Fork 路径: useExactTools → 直接使用父工具池
  │     └── 普通路径: resolveAgentTools() 过滤
  │
  ├── 5. 系统提示构建
  │     ├── Fork: 使用父的系统提示 (byte-exact)
  │     └── 普通: getAgentSystemPrompt() + enhanceSystemPromptWithEnvDetails()
  │
  ├── 6. 钩子与技能预加载
  │     ├── SubagentStart hooks → 额外上下文
  │     ├── frontmatter hooks 注册
  │     └── skills 预加载 → 初始消息
  │
  ├── 7. MCP 服务器初始化
  │     initializeAgentMcpServers() → 合并工具
  │
  ├── 8. 子代理上下文创建
  │     createSubagentContext() → 隔离的 toolUseContext
  │
  ├── 9. 查询循环
  │     for await (message of query({...}))
  │       ├── stream_event → 转发 TTFT 指标
  │       ├── attachment (max_turns_reached) → break
  │       ├── recordable → 记录 transcript + yield
  │       └── 检查 abort 信号
  │
  └── 10. 清理 (finally)
        ├── MCP 服务器清理
        ├── 会话钩子清理
        ├── 缓存追踪清理
        ├── 文件状态缓存释放
        ├── Perfetto 追踪注销
        ├── todos 清理
        └── 后台 shell 任务终止
```

### 5.3 关键设计：上下文共享 vs 隔离

| 资源 | 同步 Agent | 异步 Agent |
|------|-----------|-----------|
| `setAppState` | 共享父级 | 隔离 (no-op) |
| `setResponseLength` | 共享 | 共享 |
| `abortController` | 共享父级 | 新建独立 |
| `readFileState` | Fork 克隆 / 新建 | Fork 克隆 / 新建 |
| `thinkingConfig` | Fork 继承 / 禁用 | Fork 继承 / 禁用 |

---

## 6. 同步 vs 异步执行模型

### 6.1 同步执行

```
主线程调用 call()
  │
  ├── registerAgentForeground()   ← 注册前台任务（可被 Ctrl+B 后台化）
  │
  ├── while(true) {
  │     race(agentIterator.next(), backgroundSignal)
  │     │
  │     ├── 消息到达 → 处理
  │     │     ├── 进度追踪 (ProgressTracker)
  │     │     ├── onProgress 回调 (UI 更新)
  │     │     ├── 令牌计数累积
  │     │     └── bash_progress 转发
  │     │
  │     └── 后台化信号 → wasBackgrounded = true
  │           ├── agentIterator.return()  ← 关闭前台迭代器
  │           ├── 创建新的 runAgent() 实例  ← 后台继续
  │           └── return async_launched
  │   }
  │
  └── finalizeAgentTool() → return { status: 'completed' }
```

**前台→后台切换**：用户按 Ctrl+B 或超过 `autoBackgroundMs` 时间阈值后自动触发。核心挑战：
1. 前台迭代器必须正确关闭（释放 MCP 连接、session hooks 等）
2. 后台继续使用**新的 runAgent 实例**，继承已有消息和进度
3. 进度追踪器从已有消息重新构建

### 6.2 异步执行

```
主线程调用 call()
  │
  ├── registerAsyncAgent()  ← 注册异步任务
  │     ├── 创建独立 AbortController
  │     ├── 初始化 LocalAgentTaskState
  │     └── 输出文件符号链接
  │
  ├── void runAsyncAgentLifecycle()  ← fire-and-forget
  │     │
  │     ├── for await (msg of runAgent())
  │     │     ├── 进度追踪 & AppState 更新
  │     │     └── 可选：后台摘要生成
  │     │
  │     ├── try: finalizeAgentTool() → completeAsyncAgent()
  │     │     ├── classifyHandoffIfNeeded()
  │     │     ├── cleanupWorktreeIfNeeded()
  │     │     └── enqueueAgentNotification()
  │     │
  │     ├── catch AbortError: killAsyncAgent()
  │     │
  │     └── catch other: failAsyncAgent()
  │
  └── return { status: 'async_launched', agentId, outputFile }
```

### 6.3 生命周期状态机

```
              ┌──────────┐
              │ spawning │
              └────┬─────┘
                   │
          ┌────────▼────────┐
          │    running      │
          └──┬─────────┬────┘
             │         │
    ┌────────▼───┐ ┌───▼────────┐
    │ completed  │ │  stopped   │
    └────────────┘ └────────────┘
                       │
                  ┌────▼────┐
                  │ failed  │
                  └─────────┘
```

---

## 7. Fork 子代理机制

Fork 是 `FORK_SUBAGENT` Feature Flag 下的实验性功能，核心理念：**子代理继承父的完整对话上下文和系统提示，最大化 Anthropic API 的提示缓存命中率**。

### 7.1 触发条件

```typescript
// 用户省略 subagent_type 且 FORK_SUBAGENT 开启 → Fork 路径
const effectiveType = subagent_type ?? (isForkSubagentEnabled() ? undefined : 'general-purpose')
const isForkPath = effectiveType === undefined
```

### 7.2 FORK_AGENT 定义

```typescript
const FORK_AGENT = {
  agentType: 'fork',
  tools: ['*'],           // 完全继承父工具池
  model: 'inherit',       // 继承父模型
  permissionMode: 'bubble', // 权限提示冒泡到父终端
  source: 'built-in',
}
```

### 7.3 缓存共享原理

Anthropic API 的缓存键由以下部分组成：
```
缓存键 = hash(system_prompt + tools + model + messages_prefix + thinking_config)
```

Fork 子代理的策略：

1. **系统提示**：直接使用父的 `renderedSystemPrompt` 字节（不重新生成，避免 GrowthBook 状态变化导致差异）
2. **工具定义**：使用 `useExactTools: true` 传递父的完全相同的工具数组
3. **消息前缀**：`buildForkedMessages()` 构建字节相同的前缀

```
buildForkedMessages(directive, assistantMessage):
  1. 保留完整的父 assistant 消息 (所有 tool_use blocks)
  2. 构建 user 消息: 所有 tool_use 的 placeholder tool_result + 每子代理独有的 directive

结果: [...history, assistant(all_tool_uses), user(placeholder_results..., directive)]
       ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^  ←── 所有 fork 子代理相同
                                                                          ^^^^^^^^^  ←── 每子代理不同

只有最后的 directive 文本块不同 → 最大化缓存命中
```

### 7.4 递归防护

Fork 子代理保留 `Agent` 工具以保证工具定义一致，但通过两重防护防止递归 fork：

```typescript
// 方法1: querySource 检查 (主要 — 抗 autocompact)
if (toolUseContext.options.querySource === 'agent:builtin:fork') → 拒绝

// 方法2: 消息扫描 (回退 — 检测 fork 标记)
if (isInForkChild(messages)) → 拒绝  // 检查 FORK_BOILERPLATE_TAG
```

### 7.5 Fork vs 普通子代理对比

| 维度 | Fork | 普通子代理 |
|------|------|----------|
| 上下文 | 继承父全部对话 | 仅接收 prompt 参数 |
| 提示缓存 | 共享父缓存 | 独立缓存 |
| 系统提示 | 父的渲染字节 | Agent 自己的提示 |
| 工具池 | 完全相同 | 独立解析 |
| Thinking | 继承父配置 | 禁用 |
| Prompt 风格 | Directive (做什么) | Briefing (什么+为什么+背景) |
| 适用场景 | 不需要保留中间工具输出 | 需要完整独立执行 |

---

## 8. Coordinator 协调者模式

Coordinator 模式将 Claude Code 转变为一个纯粹的任务编排器，不直接执行操作，只通过 Worker 完成工作。

### 8.1 开启方式

```bash
CLAUDE_CODE_COORDINATOR_MODE=1 claude
```

### 8.2 协调者的工具集

协调者只能使用以下工具：
- `Agent` — 创建 Worker
- `SendMessage` — 向 Worker 发送后续消息
- `TaskStop` — 停止 Worker
- `subscribe_pr_activity` / `unsubscribe_pr_activity` — PR 事件订阅

### 8.3 Worker 能力

Worker 拥有标准工具集（`ASYNC_AGENT_ALLOWED_TOOLS`），排除了团队管理工具（TeamCreate、TeamDelete、SendMessage、SyntheticOutput）。

### 8.4 工作流

协调者遵循四阶段工作流：

```
Research → Synthesis → Implementation → Verification
   ↑                      ↓
   └──────── Worker 执行 ──┘
```

**关键原则**：
- 只读任务可并行
- 写入任务串行执行
- 验证可以在不同文件区域并行
- **综合是协调者最重要的职责** — 不能把理解委派给 Worker

### 8.5 通知机制

Worker 完成后，结果以 `<task-notification>` XML 格式注入到协调者的用户消息流：

```xml
<task-notification>
  <task-id>{agentId}</task-id>
  <status>completed|failed|killed</status>
  <summary>Agent "Investigate auth bug" completed</summary>
  <result>Found null pointer in src/auth/validate.ts:42...</result>
  <usage>
    <total_tokens>N</total_tokens>
    <tool_uses>N</tool_uses>
    <duration_ms>N</duration_ms>
  </usage>
</task-notification>
```

---

## 9. 多 Agent 团队 (Swarms)

Swarm 系统允许多个 Agent 作为 Teammate 并行运行，每个 Teammate 是一个独立的 Claude Code 实例。

### 9.1 启用条件

```typescript
function isAgentSwarmsEnabled(): boolean {
  // 内部构建: 始终启用
  // 外部构建: 需要环境变量 + GrowthBook gate
  return ENV.ENABLE_AGENT_SWARMS || CLI_FLAG || growthbook('tengu_amber_flint')
}
```

### 9.2 Teammate 创建

当 `team_name` 和 `name` 同时提供时，触发 Teammate 创建：

```typescript
// AgentTool.tsx 中的路由
if (teamName && name) {
  const result = await spawnTeammate({
    name, prompt, description,
    team_name: teamName,
    use_splitpane: true,
    plan_mode_required: spawnMode === 'plan',
    model, agent_type: subagent_type
  }, toolUseContext)
  return { status: 'teammate_spawned', ...result.data }
}
```

### 9.3 Teammate 后端

支持三种运行后端：
- **tmux** — 每个 Teammate 在独立 tmux pane 中运行
- **iTerm2** — macOS iTerm2 分屏
- **In-process** — 进程内轻量级 Teammate

### 9.4 Agent ID 系统

Swarm 使用**确定性 Agent ID**，格式为 `{name}@{teamName}`：

```typescript
function generateAgentId(name: string, team: string): string {
  return `${name}@${team}`
}
```

优势：
- **可重现**：相同名称+团队 = 相同 ID → 支持崩溃后重连
- **可读性**：`tester@my-project` 比 UUID 更易调试
- **可预测**：Team lead 可直接计算 teammate 的 ID

### 9.5 约束

- Teammate 不能创建嵌套 Teammate（花名册是扁平的）
- 进程内 Teammate 不能创建后台 Agent
- Agent 名称不能包含 `@` 字符

---

## 10. 工具过滤与权限控制

### 10.1 三层过滤

```
全局禁止 (ALL_AGENT_DISALLOWED_TOOLS)
  → 自定义禁止 (CUSTOM_AGENT_DISALLOWED_TOOLS)
    → 异步限制 (ASYNC_AGENT_ALLOWED_TOOLS)
      → Agent 定义的白名单/黑名单
```

### 10.2 resolveAgentTools()

```typescript
function resolveAgentTools(agentDef, availableTools, isAsync): ResolvedAgentTools {
  // 1. 解析 tools 列表 ("*" → 全部; 具体名称 → 匹配)
  // 2. 应用 disallowedTools 黑名单
  // 3. 解析 Agent(x,y) 元数据 → allowedAgentTypes
  // 4. filterToolsForAgent() 三层过滤
  return { resolvedTools, validTools, invalidTools, hasWildcard, allowedAgentTypes }
}
```

### 10.3 Agent(x,y) 工具元数据

Agent 定义中可以通过 `Agent(Explore,Plan)` 语法限制子代理可以创建哪些 Agent 类型：

```markdown
tools: ["Bash", "Read", "Agent(Explore,Plan)"]
```

这会被解析为：保留 `Agent` 工具，但 `allowedAgentTypes = ['Explore', 'Plan']`。

### 10.4 权限模式

| 模式 | 行为 |
|------|------|
| `acceptEdits` | 自动接受编辑，其他工具需确认 |
| `bypassPermissions` | 跳过所有权限检查 |
| `bubble` | 权限提示冒泡到父终端 |
| `plan` | 需要计划审批 |
| `auto` | 自动分类器决定 |

Agent 的权限模式继承规则：
- 父级 `bypassPermissions`/`acceptEdits`/`auto` 时 → 保持父级模式
- 否则 → 使用 Agent 定义的 `permissionMode`

---

## 11. Agent 记忆与快照系统

### 11.1 记忆范围

| 范围 | 存储路径 | 用途 |
|------|---------|------|
| `user` | `~/.claude/agent-memory/<agentType>/` | 跨项目的个人记忆 |
| `project` | `<cwd>/.claude/agent-memory/<agentType>/` | 项目级记忆 (可 git) |
| `local` | `<cwd>/.claude/agent-memory-local/<agentType>/` | 项目级但不提交 |

### 11.2 快照同步

```
项目记忆 ←→ 快照 ←→ 其他开发者的本地记忆

snapshot.json        → { snapshotTimestamp: "..." }    快照版本
.snapshot-synced.json → { syncedTimestamp: "..." }     本地同步标记
```

`checkAgentMemorySnapshot()` 检查三种状态：
- `'none'` — 无快照存在
- `'initialize'` — 本地无记忆，需要从快照初始化
- `'prompt-update'` — 快照比本地更新，提示用户更新

### 11.3 记忆加载

`loadAgentMemoryPrompt()` 在 Agent 启动时读取所有 `.md` 文件并生成记忆提示，注入到系统提示中。

---

## 12. 任务管理框架

### 12.1 三种任务类型

| 类型 | 文件 | 存储位置 | 通信方式 |
|------|------|---------|---------|
| `LocalAgentTask` | `LocalAgentTask.tsx` | 本进程内 | `<task-notification>` XML |
| `RemoteAgentTask` | `RemoteAgentTask.tsx` | 云端 CCR | 轮询检查 |
| `InProcessTeammateTask` | `InProcessTeammateTask.tsx` | 本进程内 | Mailbox |

### 12.2 LocalAgentTaskState

```typescript
type LocalAgentTaskState = TaskStateBase & {
  type: 'local_agent'
  agentId: string
  prompt: string
  selectedAgent?: AgentDefinition
  agentType: string
  model?: string
  abortController?: AbortController
  error?: string
  result?: AgentToolResult
  progress?: AgentProgress     // { toolUseCount, tokenCount, lastActivity }
  isBackgrounded: boolean       // 前台 vs 后台标记
  pendingMessages: string[]     // SendMessage 待处理消息队列
  retain: boolean               // UI 是否持有此任务
  diskLoaded: boolean           // 是否已从磁盘加载 transcript
  evictAfter?: number           // 面板可见性截止时间
}
```

### 12.3 进度追踪

```typescript
type ProgressTracker = {
  toolUseCount: number
  latestInputTokens: number          // 累积 (API 每轮覆盖)
  cumulativeOutputTokens: number     // 累加 (每轮新增)
  recentActivities: ToolActivity[]   // 最近 5 条活动
}
```

Token 计算：`total = latestInputTokens + cumulativeOutputTokens`

---

## 13. 通信与消息路由

### 13.1 SendMessageTool

```typescript
// 输入
{ to: string, message: string | StructuredMessage }

// 路由目标
├── teammate name → mailbox 写入
├── "*" → 广播到所有 teammates
├── agentId → LocalAgentTask.queuePendingMessage()
├── peer → 本地 peer
└── remote peer → Remote Control
```

### 13.2 通知入队

```typescript
function enqueueAgentNotification({
  taskId, description, status, finalMessage, usage, worktreePath
}) {
  // 1. 原子性检查 notified 标志 (防重复)
  // 2. 构建 XML: <task-notification>...</task-notification>
  // 3. enqueuePendingNotification() → 注入到主会话消息流
  // 4. abortSpeculation() → 取消当前推测
}
```

### 13.3 消息流向图

```
                    ┌─────────────┐
                    │ Main REPL   │
                    │ (user ↔ AI) │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
        ┌─────▼─────┐ ┌───▼────┐ ┌────▼────┐
        │ Agent A   │ │Agent B │ │Agent C  │
        │(sync/bg)  │ │(async) │ │(fork)   │
        └─────┬─────┘ └───┬────┘ └────┬────┘
              │            │            │
              └────────────┼────────────┘
                           │
                  <task-notification>
                  注入主会话消息流
```

---

## 14. 模型选择策略

### 14.1 优先级链

```
CLAUDE_CODE_SUBAGENT_MODEL 环境变量       ← 最高优先级
  → tool 指定的 model 参数
    → Agent 定义的 model 字段
      → getDefaultSubagentModel()         ← 最低优先级 (inherit)
```

### 14.2 inherit 语义

当模型为 `inherit` 时，子代理继承父的 `mainLoopModel`。

### 14.3 Bedrock 区域前缀继承

```typescript
// 父模型: "eu.anthropic.claude-sonnet-4-20250514-v1:0"
// Agent 定义: model: "sonnet"
// 解析结果: "eu.anthropic.claude-sonnet-4-20250514-v1:0" (继承区域前缀)

// 逻辑: sonnet 是父模型的同级别 → 直接继承父模型全名
// 原因: IAM 权限绑定到特定区域前缀
```

### 14.4 getAgentModel() 实现

```typescript
function getAgentModel(
  agentModel: string | undefined,      // Agent 定义的模型
  mainLoopModel: string,               // 父循环模型
  toolModel: string | undefined,       // 工具参数指定的模型
  permissionMode: PermissionMode
): string {
  // 1. 环境变量覆盖
  // 2. tool 参数覆盖 (仅非 coordinator 模式)
  // 3. Agent 定义模型 → resolveInherit() 处理 'inherit'
  // 4. 默认: inherit → mainLoopModel
  // 5. Bedrock 前缀继承
}
```

---

## 15. Agent 上下文隔离

### 15.1 AsyncLocalStorage

```typescript
// agentContext.ts
const agentContextStorage = new AsyncLocalStorage<AgentContext>()

type SubagentContext = {
  agentId: string
  parentSessionId?: string
  agentType: 'subagent'
  subagentName: string
  isBuiltIn: boolean
  invokingRequestId?: string
  invocationKind: 'spawn' | 'resume'
  invocationEmitted: boolean
}

type TeammateContext = {
  agentType: 'teammate'
  agentName: string
  teamName: string
  agentColor?: string
  planModeRequired?: boolean
}
```

**为什么用 AsyncLocalStorage 而非 AppState**？
当 Agent 后台化（Ctrl+B）后，多个 Agent 可在同一进程并发运行。AppState 是共享单例会被覆盖，而 AsyncLocalStorage 隔离每条异步执行链。

### 15.2 createSubagentContext()

```typescript
function createSubagentContext(parent: ToolUseContext, overrides): ToolUseContext {
  return {
    ...parent,
    options: overrides.options,          // Agent 专属配置
    agentId: overrides.agentId,
    messages: overrides.messages,        // 独立消息历史
    readFileState: overrides.readFileState,  // 独立文件缓存
    abortController: overrides.abortController,
    getAppState: overrides.getAppState,
    // 同步 Agent 共享 setAppState，异步 Agent 隔离
    setAppState: overrides.shareSetAppState ? parent.setAppState : noOp,
  }
}
```

---

## 16. 会话恢复 (Resume)

`resumeAgent.ts` 实现了后台 Agent 的持久化恢复，允许在会话重启后继续执行。

### 16.1 恢复流程

```
resumeAgentBackground(agentId)
  │
  ├── 1. 读取 transcript (JSONL) + metadata
  │
  ├── 2. 消息过滤
  │     ├── 移除空白 assistant 消息
  │     ├── 移除孤立 thinking blocks
  │     └── 移除未解析的工具调用
  │
  ├── 3. 重建 contentReplacementState
  │     (确保工具结果替换的一致性，维护缓存稳定性)
  │
  ├── 4. Worktree 验证
  │     ├── 路径存在 → 更新 mtime (防止 stale 清理)
  │     └── 路径不存在 → 回退到父 cwd
  │
  ├── 5. Agent 定义恢复
  │     ├── metadata.agentType → 查找 AgentDefinition
  │     └── Fork agent → 重建父系统提示
  │
  ├── 6. 注册 + 启动
  │     ├── registerAsyncAgent()
  │     └── runAsyncAgentLifecycle()
  │
  └── return { agentId, description, outputFile }
```

### 16.2 缓存稳定性

恢复时必须重建 `contentReplacementState`，确保工具结果的 replacement 与原始执行一致：

```
原始执行: tool_result "abc" → 替换为 "replaced_abc"
恢复执行: tool_result "abc" → 必须同样替换为 "replaced_abc" (否则缓存失效)
```

---

## 17. 后台摘要生成

`agentSummary.ts` 为协调者模式提供 Agent 进度的实时摘要。

### 17.1 机制

```
每 30 秒触发:
  1. 从 toolUseContext 读取当前消息
  2. Fork 子 Agent 对话 (共享缓存参数)
  3. 在 canUseTool 中拒绝所有工具使用 (但保留工具定义)
  4. 系统提示: "1 sentence status of what the agent is doing"
  5. 提取 3-5 词进度描述 (现在进行时)
  6. 更新 task.progress.summary
```

### 17.2 提示设计

```
要求: 3-5 个词, 现在进行时 (-ing)
好的: "Refactoring auth middleware"
坏的: "Working on feature/auth-refactor" ← 分支名不有用
```

---

## 18. Worktree 隔离

### 18.1 触发

```typescript
if (isolation === 'worktree') {
  const slug = `agent-${earlyAgentId.slice(0, 8)}`
  worktreeInfo = await createAgentWorktree(slug)
}
```

### 18.2 生命周期

```
创建 → Agent 执行 (cwd = worktreePath) → 完成检查 → 清理/保留

完成后:
  ├── 无变更 (HEAD 一致) → removeAgentWorktree()
  └── 有变更 → 保留, 日志记录, 通知中包含 worktreePath + worktreeBranch
```

### 18.3 Fork + Worktree

Fork 子代理在 worktree 中运行时，会注入额外通知消息 `buildWorktreeNotice()`，告知子代理路径翻译规则和文件可能过期。

---

## 19. 安全与分类

### 19.1 Handoff 分类

`classifyHandoffIfNeeded()` 在 `TRANSCRIPT_CLASSIFIER` Feature Flag 和 `auto` 权限模式下运行：

```typescript
async function classifyHandoffIfNeeded({
  agentMessages, tools, toolPermissionContext,
  abortSignal, subagentType, totalToolUseCount
}): Promise<string | undefined> {
  // 1. 构建 transcript
  // 2. YOLO 分类器审查子 Agent 行为
  // 3. 如果检测到违反安全策略 → 返回警告字符串
  // 4. 警告注入到 tool_result 前面
}
```

### 19.2 权限自动分类

在 `auto` 模式下，Agent 工具本身需要通过权限检查：

```typescript
async checkPermissions(input, context): Promise<PermissionResult> {
  if (mode === 'auto') {
    return { behavior: 'passthrough', message: 'Agent tool requires permission...' }
  }
  return { behavior: 'allow' }
}
```

---

## 20. 关键数据流图

### 20.1 完整调用链

```
用户输入 "帮我搜索所有 TODO 注释"
    │
    ▼
主循环 query()
    │
    ▼
模型决策: tool_use { name: "Agent", input: { subagent_type: "Explore", prompt: "..." } }
    │
    ▼
AgentTool.call()
    ├── 查找 AgentDefinition (Explore)
    ├── 解析模型 (haiku)
    ├── 构建系统提示
    ├── 过滤工具 (只读工具集)
    ├── shouldRunAsync = true (coordinator mode)
    │
    ▼
registerAsyncAgent() → LocalAgentTaskState
    │
    ▼
void runAsyncAgentLifecycle()
    │
    ▼
runAgent() ← AsyncGenerator
    ├── createSubagentContext()
    ├── initializeAgentMcpServers()
    ├── for await (msg of query({
    │     messages: [userMessage(prompt)],
    │     systemPrompt: exploreAgentPrompt,
    │     tools: readOnlyTools,
    │     model: 'haiku'
    │   }))
    │     ├── assistant: tool_use(Grep, "TODO")
    │     ├── user: tool_result(matches...)
    │     ├── assistant: tool_use(Read, file1.ts)
    │     ├── user: tool_result(content...)
    │     └── assistant: "Found 15 TODOs across 8 files..."
    │
    ▼
finalizeAgentTool() → AgentToolResult
    │
    ▼
completeAsyncAgent() → 状态更新
    │
    ▼
enqueueAgentNotification()
    │
    ▼
<task-notification> XML 注入主会话
    │
    ▼
主循环收到通知 → 模型生成回复
    │
    ▼
"我找到了 15 个 TODO 注释分布在 8 个文件中..."
```

### 20.2 模块依赖关系

```
                        ┌──────────────┐
                        │  AgentTool   │ ← 入口
                        │  (call)      │
                        └──────┬───────┘
                               │
                 ┌─────────────┼─────────────┐
                 │             │             │
          ┌──────▼──────┐ ┌───▼────┐ ┌──────▼──────┐
          │  runAgent   │ │ fork   │ │ spawn       │
          │  (引擎)     │ │Subagent│ │ Teammate    │
          └──────┬──────┘ └───┬────┘ └──────┬──────┘
                 │            │             │
          ┌──────▼──────┐    │      ┌──────▼──────┐
          │   query()   │◄───┘      │ InProcess   │
          │  (API 调用) │           │ Teammate    │
          └──────┬──────┘           └─────────────┘
                 │
          ┌──────▼──────┐
          │ 工具执行    │
          │ Bash/Read/  │
          │ Write/...   │
          └─────────────┘
                 ↓
         ┌──────────────┐
         │ LocalAgent   │ ← 任务状态
         │ TaskState    │
         └──────┬───────┘
                │
         ┌──────▼──────┐
         │ Notification │ ← 结果回传
         │ (XML)        │
         └──────────────┘
```

---

## 附录 A：关键常量

| 常量 | 值 | 用途 |
|------|-----|------|
| `AGENT_TOOL_NAME` | `'Agent'` | 工具名 |
| `LEGACY_AGENT_TOOL_NAME` | `'Task'` | 向后兼容别名 |
| `PROGRESS_THRESHOLD_MS` | `2000` | 显示后台提示延迟 |
| `MAX_RECENT_ACTIVITIES` | `5` | 进度追踪活动上限 |
| `PANEL_GRACE_MS` | — | 任务面板保留时间 |
| `READ_FILE_STATE_CACHE_SIZE` | — | 文件缓存大小 |
| `SUMMARY_INTERVAL_MS` | `30000` | 摘要生成间隔 |

## 附录 B：Feature Flags

| Flag | 功能 |
|------|------|
| `FORK_SUBAGENT` | Fork 子代理实验 |
| `COORDINATOR_MODE` | 协调者模式 |
| `KAIROS` | 助手/守护进程模式 |
| `BUILTIN_EXPLORE_PLAN_AGENTS` | Explore/Plan 内置 Agent |
| `VERIFICATION_AGENT` | 验证 Agent |
| `TRANSCRIPT_CLASSIFIER` | 安全分类器 |
| `PROMPT_CACHE_BREAK_DETECTION` | 缓存中断检测 |
| `PROACTIVE` | 主动代理 |
| `MONITOR_TOOL` | 监控工具 |

## 附录 C：GrowthBook Gates

| Gate | 功能 |
|------|------|
| `tengu_amber_stoat` | Explore/Plan Agent 启用 |
| `tengu_amber_flint` | Swarm 功能杀手开关 |
| `tengu_hive_evidence` | Verification Agent 启用 |
| `tengu_auto_background_agents` | 自动后台化 |
| `tengu_agent_list_attach` | Agent 列表注入为附件 |
| `tengu_slim_subagent_claudemd` | 子代理省略 CLAUDE.md |
| `tengu_scratch` | Scratchpad 目录 |

---

*文档基于 Claude Code v2.1.88 源码静态分析生成*
