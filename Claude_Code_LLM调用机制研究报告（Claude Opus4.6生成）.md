# Claude Code LLM 调用机制研究报告

> 基于 Claude Code v2.1.88 源码深度分析
> 分析日期：2026-04-09

---

## 目录

1. [概述与架构总览](#1-概述与架构总览)
2. [API Provider 多后端架构](#2-api-provider-多后端架构)
3. [客户端创建与认证体系](#3-客户端创建与认证体系)
4. [模型系统与选择机制](#4-模型系统与选择机制)
5. [核心查询引擎 QueryEngine](#5-核心查询引擎-queryengine)
6. [查询循环 query.ts 详解](#6-查询循环-queryts-详解)
7. [API 请求构造与参数体系 claude.ts](#7-api-请求构造与参数体系-claudets)
8. [流式响应处理机制](#8-流式响应处理机制)
9. [重试与容错机制 withRetry](#9-重试与容错机制-withretry)
10. [Prompt Cache 与成本优化](#10-prompt-cache-与成本优化)
11. [Thinking（思考链）配置体系](#11-thinking思考链配置体系)
12. [Beta 特性与动态头管理](#12-beta-特性与动态头管理)
13. [Token 预算与上下文管理](#13-token-预算与上下文管理)
14. [Tool Schema 构建与序列化](#14-tool-schema-构建与序列化)
15. [模型能力检测与上下文窗口](#15-模型能力检测与上下文窗口)
16. [环境变量完整参考](#16-环境变量完整参考)
17. [自定义调用其他大模型指南](#17-自定义调用其他大模型指南)
18. [架构设计启示](#18-架构设计启示)

---

## 1. 概述与架构总览

### 1.1 调用链路全景

Claude Code 的 LLM 调用是一套完整的多层架构，从用户输入到最终收到模型响应，经过以下核心层：

```
用户输入
  │
  ▼
┌─────────────────────────────────┐
│  QueryEngine (src/QueryEngine.ts)│  ← 会话级管理：模型选择、工具注册、上下文组装
│  submitMessage() → query()       │
└──────────────┬──────────────────┘
               │
               ▼
┌─────────────────────────────────┐
│  queryLoop (src/query.ts)        │  ← 多轮循环：自动压缩、工具执行、token 预算
│  while(true) { ... }             │
└──────────────┬──────────────────┘
               │
               ▼
┌─────────────────────────────────┐
│  queryModel (services/api/claude.ts) │  ← 请求构造：system prompt、messages、tools、betas
│  → paramsFromContext()                │
└──────────────┬──────────────────┘
               │
               ▼
┌─────────────────────────────────┐
│  withRetry (services/api/withRetry.ts) │  ← 重试/容错：529/429处理、模型降级
│  → getAnthropicClient()                │
└──────────────┬──────────────────┘
               │
               ▼
┌─────────────────────────────────┐
│  getAnthropicClient (services/api/client.ts) │  ← 客户端工厂：根据 Provider 创建对应 SDK
│  → Anthropic / AnthropicBedrock / ...        │
└──────────────┬──────────────────┘
               │
               ▼
┌─────────────────────────────────┐
│  Anthropic SDK                   │  ← 底层 HTTP：请求发送、流式响应接收
│  anthropic.beta.messages.create()│
└─────────────────────────────────┘
```

### 1.2 核心源文件清单

| 文件路径 | 代码行数 | 核心职责 |
|---------|---------|---------|
| `src/services/api/claude.ts` | 3420 | API 请求构造、流式处理、响应解析 |
| `src/query.ts` | 1730 | 多轮查询循环、工具执行、自动压缩 |
| `src/QueryEngine.ts` | 1296 | 会话引擎、配置管理、提交消息 |
| `src/services/api/client.ts` | 390 | SDK 客户端工厂（多 Provider） |
| `src/services/api/withRetry.ts` | 823 | 重试逻辑、容错降级 |
| `src/utils/model/model.ts` | 619 | 模型选择优先级链 |
| `src/utils/model/configs.ts` | ~130 | 模型 ID 跨 Provider 映射表 |
| `src/utils/model/modelStrings.ts` | ~170 | 运行时模型字符串解析 |
| `src/utils/model/providers.ts` | ~50 | Provider 类型定义与检测 |
| `src/utils/api.ts` | 719 | Tool schema 序列化、消息规范化 |
| `src/utils/model/modelCapabilities.ts` | ~150 | 模型能力缓存与检测 |

---

## 2. API Provider 多后端架构

### 2.1 Provider 类型定义

```typescript
// src/utils/model/providers.ts
type APIProvider = 'firstParty' | 'bedrock' | 'vertex' | 'foundry'
```

四种 Provider 对应四种不同的 Anthropic API 接入方式：

| Provider | 说明 | SDK 包 | 环境变量触发 |
|---------|------|--------|-------------|
| `firstParty` | Anthropic 直连 API | `@anthropic-ai/sdk` | 默认 |
| `bedrock` | AWS Bedrock 托管 | `@anthropic-ai/bedrock-sdk` | `CLAUDE_CODE_USE_BEDROCK=1` |
| `vertex` | Google Cloud Vertex AI | `@anthropic-ai/vertex-sdk` | `CLAUDE_CODE_USE_VERTEX=1` |
| `foundry` | Azure Foundry 平台 | `@anthropic-ai/foundry-sdk` | `CLAUDE_CODE_USE_FOUNDRY=1` |

### 2.2 Provider 检测逻辑

```typescript
// src/utils/model/providers.ts
export function getAPIProvider(): APIProvider {
  if (isEnvTruthy(process.env.CLAUDE_CODE_USE_BEDROCK)) return 'bedrock'
  if (isEnvTruthy(process.env.CLAUDE_CODE_USE_VERTEX)) return 'vertex'
  if (isEnvTruthy(process.env.CLAUDE_CODE_USE_FOUNDRY)) return 'foundry'
  return 'firstParty'
}

export function isFirstPartyAnthropicBaseUrl(): boolean {
  const baseUrl = process.env.ANTHROPIC_BASE_URL
  return !baseUrl || baseUrl.includes('api.anthropic.com')
}
```

> **关键设计**：Provider 通过环境变量单一切换，互斥选择。`isFirstPartyAnthropicBaseUrl()` 用于判断是否使用自定义代理（如 LiteLLM），影响部分 beta 头发送策略。

### 2.3 模型 ID 跨 Provider 映射

每个模型在不同 Provider 下有不同的 ID 字符串：

```typescript
// src/utils/model/configs.ts
export const CLAUDE_OPUS_4_6_CONFIG = {
  firstParty: 'claude-opus-4-6',
  bedrock:    'us.anthropic.claude-opus-4-6-v1',
  vertex:     'claude-opus-4-6',
  foundry:    'claude-opus-4-6',
} as const satisfies ModelConfig

export const CLAUDE_SONNET_4_6_CONFIG = {
  firstParty: 'claude-sonnet-4-6',
  bedrock:    'us.anthropic.claude-sonnet-4-6',
  vertex:     'claude-sonnet-4-6',
  foundry:    'claude-sonnet-4-6',
} as const satisfies ModelConfig
```

完整模型注册表 `ALL_MODEL_CONFIGS`：

| 内部 Key | 首方 ID | Bedrock ID | Vertex ID |
|----------|---------|------------|-----------|
| `opus46` | `claude-opus-4-6` | `us.anthropic.claude-opus-4-6-v1` | `claude-opus-4-6` |
| `opus45` | `claude-opus-4-5-20251101` | `us.anthropic.claude-opus-4-5-20251101-v1:0` | `claude-opus-4-5@20251101` |
| `sonnet46` | `claude-sonnet-4-6` | `us.anthropic.claude-sonnet-4-6` | `claude-sonnet-4-6` |
| `sonnet45` | `claude-sonnet-4-5-20250929` | `us.anthropic.claude-sonnet-4-5-20250929-v1:0` | `claude-sonnet-4-5@20250929` |
| `haiku45` | `claude-haiku-4-5-20251001` | `us.anthropic.claude-haiku-4-5-20251001-v1:0` | `claude-haiku-4-5@20251001` |
| `haiku35` | `claude-3-5-haiku-20241022` | `us.anthropic.claude-3-5-haiku-20241022-v1:0` | `claude-3-5-haiku@20241022` |

---

## 3. 客户端创建与认证体系

### 3.1 客户端工厂函数

`getAnthropicClient()` 是唯一的客户端创建入口，根据当前 Provider 返回不同的 SDK 实例：

```typescript
// src/services/api/client.ts
export async function getAnthropicClient({
  apiKey, maxRetries, model, fetchOverride, source
}: ClientOptions): Promise<Anthropic> {
  // 公共参数
  const ARGS = {
    maxRetries: maxRetries ?? 0,
    fetch: buildFetch(fetchOverride, source),
    defaultHeaders: {
      ...getCustomHeaders(),
      ...(await getProxyFetchOptions()).headers,
    },
  }

  // Provider 路由
  if (isEnvTruthy(process.env.CLAUDE_CODE_USE_BEDROCK)) {
    return createBedrockClient(ARGS, model)
  }
  if (isEnvTruthy(process.env.CLAUDE_CODE_USE_FOUNDRY)) {
    return createFoundryClient(ARGS)
  }
  if (isEnvTruthy(process.env.CLAUDE_CODE_USE_VERTEX)) {
    return createVertexClient(ARGS, model)
  }
  return createFirstPartyClient(ARGS, apiKey)
}
```

### 3.2 各 Provider 认证方式

#### FirstParty（直连 Anthropic）

```typescript
const clientConfig = {
  apiKey: isClaudeAISubscriber() ? null : apiKey || getAnthropicApiKey(),
  authToken: isClaudeAISubscriber()
    ? getClaudeAIOAuthTokens()?.accessToken
    : undefined,
}
return new Anthropic(clientConfig)
```

- **API Key 模式**：通过 `ANTHROPIC_API_KEY` 环境变量
- **OAuth 模式**：Claude AI 订阅用户自动使用 OAuth token，支持自动刷新
- **Bearer Token 模式**：通过 `ANTHROPIC_AUTH_TOKEN` 环境变量设置自定义认证头
- **自定义 Base URL**：通过 `ANTHROPIC_BASE_URL` 可指向代理服务器

#### Bedrock（AWS）

```typescript
const { AnthropicBedrock } = await import('@anthropic-ai/bedrock-sdk')
return new AnthropicBedrock({
  ...ARGS,
  awsRegion: process.env.AWS_REGION,
  // 认证优先级：
  // 1. AWS_SESSION_TOKEN (临时凭证)
  // 2. AWS_BEARER_TOKEN_BEDROCK (Bearer Token)
  // 3. AWS Credentials Provider (默认凭证链)
})
```

- 支持 AWS 默认凭证链（IAM Role、环境变量、配置文件等）
- 支持 `AWS_REGION` 指定区域
- 支持 `ANTHROPIC_SMALL_FAST_MODEL_AWS_REGION` 为 Haiku 指定独立区域
- Bedrock Inference Profile 自动发现（通过 `ListInferenceProfiles` API）

#### Vertex（GCP）

```typescript
const [{ AnthropicVertex }, { GoogleAuth }] = await Promise.all([
  import('@anthropic-ai/vertex-sdk'),
  import('google-auth-library'),
])
const googleAuth = new GoogleAuth({
  scopes: ['https://www.googleapis.com/auth/cloud-platform'],
  projectId: process.env.ANTHROPIC_VERTEX_PROJECT_ID, // 降级方案
})
return new AnthropicVertex({
  ...ARGS,
  region: getVertexRegionForModel(model),
  googleAuth,
})
```

- 使用 Google Application Default Credentials
- 项目 ID 查找顺序：`GCLOUD_PROJECT` → `GOOGLE_CLOUD_PROJECT` → 凭证文件 → GCE 元数据 → `ANTHROPIC_VERTEX_PROJECT_ID`（防止 12 秒元数据服务器超时）
- 支持 `CLAUDE_CODE_SKIP_VERTEX_AUTH` 跳过认证（用于代理场景）

#### Foundry（Azure）

```typescript
const { AnthropicFoundry } = await import('@anthropic-ai/foundry-sdk')
return new AnthropicFoundry({
  ...ARGS,
  azureADTokenProvider, // Azure AD 令牌提供者
  // 或 ANTHROPIC_FOUNDRY_API_KEY
})
```

- 优先使用 Azure AD 认证（`DefaultAzureCredential`）
- 也支持 `ANTHROPIC_FOUNDRY_API_KEY` API Key 认证

### 3.3 自定义请求头与代理

```typescript
// 自定义请求头
function getCustomHeaders(): Record<string, string> {
  const customHeadersEnv = process.env.ANTHROPIC_CUSTOM_HEADERS
  // 格式: "Header-Name: value\nAnother-Header: value"
  // 按换行分割，每行 "Name: Value" 格式
}

// 请求 fetch 包装
function buildFetch(fetchOverride, source): ClientOptions['fetch'] {
  return (input, init) => {
    const headers = new Headers(init?.headers)
    // 注入 x-client-request-id (仅 firstParty)
    if (injectClientRequestId) {
      headers.set('x-client-request-id', randomUUID())
    }
    return inner(input, { ...init, headers })
  }
}
```

---

## 4. 模型系统与选择机制

### 4.1 模型选择优先级链

Claude Code 通过 5 级优先级链确定运行时使用的模型：

```
/model 命令 (交互切换)
    ↓ 未设置
--model 命令行参数
    ↓ 未设置
ANTHROPIC_MODEL 环境变量
    ↓ 未设置
Settings 配置文件 (model 字段)
    ↓ 未设置
内置默认值 (按订阅等级)
```

### 4.2 内置默认模型

```typescript
// src/utils/model/model.ts
function getDefaultForSubscriptionType(): ModelName {
  if (isMaxSubscriber() || isTeamPremiumSubscriber()) {
    return getDefaultOpusModel()   // → claude-opus-4-6
  }
  return getDefaultSonnetModel()   // → claude-sonnet-4-6
}
```

| 订阅等级 | 默认主模型 | 默认小模型 |
|---------|----------|----------|
| Max / Team Premium | Opus 4.6 | Haiku 4.5 |
| Pro / PAYG / Enterprise | Sonnet 4.6 | Haiku 4.5 |

### 4.3 模型别名系统

```typescript
// src/utils/model/aliases.ts
export const MODEL_ALIASES = [
  'sonnet',      // → getDefaultSonnetModel()
  'opus',        // → getDefaultOpusModel()
  'haiku',       // → getDefaultHaikuModel()
  'best',        // → getBestModel()
  'sonnet[1m]',  // → sonnet + 1M 上下文
  'opus[1m]',    // → opus + 1M 上下文
  'opusplan',    // → sonnet 默认, opus 在 plan mode
]
```

### 4.4 模型解析函数

```typescript
// src/utils/model/model.ts
export function parseUserSpecifiedModel(modelInput: string): ModelName {
  // 1. 标准化输入（小写、去空格）
  // 2. 检测 [1m] 后缀
  // 3. 别名匹配 → 返回具体模型 ID
  // 4. 旧版 Opus 自动重映射（4.0/4.1 → 当前默认）
  // 5. Ant 内部模型解析
  // 6. 保持原始输入不变 (自定义模型名)
}
```

### 4.5 环境变量覆盖

```typescript
// 可覆盖各版本默认模型
ANTHROPIC_DEFAULT_OPUS_MODEL    // 覆盖 'opus' 别名指向
ANTHROPIC_DEFAULT_SONNET_MODEL  // 覆盖 'sonnet' 别名指向
ANTHROPIC_DEFAULT_HAIKU_MODEL   // 覆盖 'haiku' 别名指向
ANTHROPIC_SMALL_FAST_MODEL      // 覆盖小型快速模型（用于分类等）

// 示例：指向自定义模型
ANTHROPIC_MODEL=my-custom-model-v1
```

### 4.6 ModelOverrides（设置覆盖）

```typescript
// src/utils/model/modelStrings.ts
function applyModelOverrides(ms: ModelStrings): ModelStrings {
  const overrides = getInitialSettings().modelOverrides
  // overrides 的 key 是 canonical 模型 ID（如 "claude-opus-4-6"）
  // value 是任意 provider 特定字符串（如 Bedrock ARN）
  for (const [canonicalId, override] of Object.entries(overrides)) {
    const key = CANONICAL_ID_TO_KEY[canonicalId]
    if (key && override) {
      out[key] = override
    }
  }
}
```

---

## 5. 核心查询引擎 QueryEngine

### 5.1 QueryEngine 配置

```typescript
// src/QueryEngine.ts
type QueryEngineConfig = {
  cwd: string
  tools: Tools
  commands: Command[]
  mcpClients: McpClient[]
  agents: AgentDefinition[]
  canUseTool: CanUseTool
  getAppState: () => AppState
  setAppState: (state: AppState) => void
  customSystemPrompt?: string
  appendSystemPrompt?: string
  userSpecifiedModel?: string
  fallbackModel?: string
  thinkingConfig: ThinkingConfig
  maxTurns?: number
  maxBudgetUsd?: number
  taskBudget?: { total: number; remaining?: number }
  jsonSchema?: object        // 结构化输出
  outputFormat?: string
}
```

### 5.2 submitMessage 核心流程

```typescript
class QueryEngine {
  async* submitMessage(userInput: string): AsyncGenerator<...> {
    // 1. 构建消息列表
    const messages = this.buildMessages(userInput)

    // 2. 组装系统提示词
    const systemPrompt = this.buildSystemPrompt()

    // 3. 如果是 Coordinator 模式，注入协调上下文
    if (this.coordinatorContext) {
      userContext = getCoordinatorUserContext(...)
    }

    // 4. 调用 query() 进入查询循环
    const terminal = yield* query({
      messages,
      systemPrompt,
      userContext,
      systemContext,
      canUseTool: this.canUseTool,
      toolUseContext: this.buildToolUseContext(),
      fallbackModel: this.config.fallbackModel,
      querySource: this.querySource,
      maxTurns: this.config.maxTurns,
      taskBudget: this.config.taskBudget,
    })

    // 5. 更新会话状态
    this.updateSessionState(terminal)
  }
}
```

---

## 6. 查询循环 query.ts 详解

### 6.1 QueryParams 类型

```typescript
type QueryParams = {
  messages: Message[]
  systemPrompt: SystemPrompt
  userContext: string
  systemContext: string
  canUseTool: CanUseTool
  toolUseContext: ToolUseContext
  fallbackModel?: string
  querySource: QuerySource
  maxOutputTokensOverride?: number
  maxTurns?: number
  taskBudget?: { total: number; remaining?: number }
  skipCacheWrite?: boolean
  deps?: QueryDeps  // 可注入的依赖
}
```

### 6.2 查询循环主流程

```typescript
async function* queryLoop(params, consumedCommandUuids) {
  // 初始化循环状态
  let state: State = {
    messages: params.messages,
    toolUseContext: params.toolUseContext,
    turnCount: 1,
    autoCompactTracking: undefined,
    hasAttemptedReactiveCompact: false,
    // ...
  }
  const budgetTracker = createBudgetTracker()
  const config = buildQueryConfig()  // 快照一次性配置

  while (true) {
    // ═══ 第一阶段：预处理 ═══
    // 1. Skill Discovery 预取
    // 2. 工具结果预算控制 (applyToolResultBudget)
    // 3. History Snip（历史剪裁）
    // 4. Microcompact（微压缩）
    // 5. Context Collapse（上下文折叠）
    // 6. 自动压缩 (autocompact)

    // ═══ 第二阶段：API 调用 ═══
    // 7. Token 预算检查（阻塞限制）
    // 8. 调用 deps.callModel() → queryModelWithStreaming()
    //    流式接收 assistant 消息

    // ═══ 第三阶段：工具执行 ═══
    // 9. 流式工具执行 (StreamingToolExecutor)
    // 10. 工具权限检查
    // 11. 实际工具调用
    // 12. 收集 tool_result 消息

    // ═══ 第四阶段：循环控制 ═══
    // 13. 检查是否需要继续
    //     - 有 tool_use → 继续（需执行工具并回传结果）
    //     - stop_reason = 'end_turn' → 结束
    //     - stop_reason = 'max_tokens' → 尝试恢复
    //     - 达到 maxTurns → 结束
    // 14. 更新 state, 进入下一轮
  }
}
```

### 6.3 继续条件（Continue）判断

```typescript
// 查询循环继续的原因
type Continue =
  | { type: 'tool_use' }              // 有工具调用需要执行
  | { type: 'max_tokens_recovery' }    // max_tokens 恢复
  | { type: 'stop_hook_retry' }        // stop hook 触发重试
  | { type: 'reactive_compact' }       // 响应式压缩后重试
```

### 6.4 自动压缩触发

在每次循环迭代中，在 API 调用前运行自动压缩检查：

```
消息列表 messagesForQuery
    ↓
applyToolResultBudget()   ← 控制单条工具结果大小
    ↓
snipCompactIfNeeded()     ← 历史剪裁（删除旧对话片段）
    ↓
microcompact()            ← 微压缩（缩减冗余内容）
    ↓
applyCollapsesIfNeeded()  ← 上下文折叠（将旧工具调用折叠为摘要）
    ↓
autocompact()             ← 全量压缩（当 token 超过阈值时触发）
    ↓
最终消息列表 → 发送给 API
```

---

## 7. API 请求构造与参数体系 claude.ts

### 7.1 核心请求函数

`claude.ts` 中的 `queryModel()` 是实际构造 API 请求的核心异步生成器：

```typescript
async function* queryModel(
  messages: Message[],
  systemPrompt: SystemPrompt,
  thinkingConfig: ThinkingConfig,
  tools: Tools,
  signal: AbortSignal,
  options: Options,
): AsyncGenerator<StreamEvent | AssistantMessage | SystemAPIErrorMessage> {
  // ... 3000+ 行核心逻辑
}
```

### 7.2 Options 类型定义

```typescript
type Options = {
  getToolPermissionContext: () => Promise<ToolPermissionContext>
  model: string
  toolChoice?: BetaToolChoiceTool | BetaToolChoiceAuto
  isNonInteractiveSession: boolean
  extraToolSchemas?: BetaToolUnion[]
  maxOutputTokensOverride?: number
  fallbackModel?: string
  onStreamingFallback?: () => void
  querySource: QuerySource
  agents: AgentDefinition[]
  allowedAgentTypes?: string[]
  hasAppendSystemPrompt: boolean
  fetchOverride?: ClientOptions['fetch']
  enablePromptCaching?: boolean
  skipCacheWrite?: boolean
  temperatureOverride?: number
  effortValue?: EffortValue
  mcpTools: Tools
  hasPendingMcpServers?: boolean
  queryTracking?: QueryChainTracking
  agentId?: AgentId
  outputFormat?: BetaJSONOutputFormat
  fastMode?: boolean
  advisorModel?: string
  addNotification?: (notif: Notification) => void
  taskBudget?: { total: number; remaining?: number }
}
```

### 7.3 请求参数构造 paramsFromContext

这是一个闭包函数，每次 API 调用（含重试）都会调用：

```typescript
const paramsFromContext = (retryContext: RetryContext) => {
  return {
    // ── 基础参数 ──
    model: normalizeModelStringForAPI(options.model),
    messages: addCacheBreakpoints(messagesForAPI, ...),
    system: buildSystemPromptBlocks(systemPrompt, enablePromptCaching, ...),
    tools: allTools,
    tool_choice: options.toolChoice,
    max_tokens: maxOutputTokens,

    // ── Thinking 配置 ──
    thinking: {
      type: 'adaptive',  // 或 { type: 'enabled', budget_tokens: N }
    },

    // ── 温度 ──
    temperature: !hasThinking ? (temperatureOverride ?? 1) : undefined,

    // ── Beta 特性 ──
    betas: betasParams,        // 动态构建的 beta 头列表

    // ── 元数据 ──
    metadata: getAPIMetadata(), // { user_id: JSON }

    // ── 上下文管理 ──
    context_management: contextManagement,

    // ── 输出配置 ──
    output_config: {
      effort: effortValue,      // 'high' | 'medium' | 'low'
      task_budget: { type: 'tokens', total, remaining },
      format: outputFormat,     // JSON schema
    },

    // ── 速度模式 ──
    speed: isFastMode ? 'fast' : undefined,

    // ── 额外参数 ──
    ...extraBodyParams,  // CLAUDE_CODE_EXTRA_BODY 环境变量
  }
}
```

### 7.4 实际 API 调用

```typescript
// 流式调用
const result = await anthropic.beta.messages
  .create(
    { ...params, stream: true },
    {
      signal,
      headers: { [CLIENT_REQUEST_ID_HEADER]: clientRequestId },
    },
  )
  .withResponse()

stream = result.data
streamRequestId = result.request_id
streamResponse = result.response
```

### 7.5 消息规范化

在发送给 API 前，消息经过多步规范化处理：

```
原始 messages
    ↓
normalizeMessagesForAPI()     ← 转换为 API 兼容格式
    ↓
ensureToolResultPairing()     ← 修复 tool_use/tool_result 配对
    ↓
stripAdvisorBlocks()          ← 移除不支持的 advisor 块
    ↓
stripExcessMediaItems(100)    ← 限制媒体项数量 ≤100
    ↓
addCacheBreakpoints()         ← 添加缓存控制标记
    ↓
prependUserContext()          ← 注入用户上下文
    ↓
最终 messagesForAPI
```

### 7.6 系统提示词构建

```typescript
systemPrompt = [
  getAttributionHeader(fingerprint),    // 归因标识
  getCLISyspromptPrefix({...}),         // CLI 前缀（交互/非交互差异）
  ...systemPrompt,                       // 用户/系统提示词
  ...(advisorModel ? [ADVISOR_TOOL_INSTRUCTIONS] : []),
]

const system = buildSystemPromptBlocks(systemPrompt, enablePromptCaching, {
  skipGlobalCacheForSystemPrompt: needsToolBasedCacheMarker,
  querySource,
})
```

### 7.7 Extra Body 扩展

```typescript
// CLAUDE_CODE_EXTRA_BODY 环境变量，可注入任意请求参数
export function getExtraBodyParams(betaHeaders?: string[]): JsonObject {
  const extraBodyStr = process.env.CLAUDE_CODE_EXTRA_BODY
  let result = safeParseJSON(extraBodyStr) as JsonObject
  // 合并 beta headers
  // 注入 anti_distillation（仅 CLI + 1P）
  return result
}

// 示例
// CLAUDE_CODE_EXTRA_BODY='{"temperature": 0.5, "top_p": 0.9}'
```

---

## 8. 流式响应处理机制

### 8.1 流式 vs 非流式入口

```typescript
// 流式（默认路径）
export async function* queryModelWithStreaming({...}): AsyncGenerator<...> {
  yield* withStreamingVCR(messages, async function* () {
    yield* queryModel(messages, systemPrompt, thinkingConfig, tools, signal, options)
  })
}

// 非流式（用于 API Key 验证等简单场景）
export async function queryModelWithoutStreaming({...}): Promise<AssistantMessage> {
  let assistantMessage: AssistantMessage | undefined
  for await (const message of withStreamingVCR(messages, async function* () {
    yield* queryModel(...)
  })) {
    if (message.type === 'assistant') assistantMessage = message
  }
  return assistantMessage!
}
```

### 8.2 流式事件处理

`queryModel()` 内部使用原始 Stream 而非 `BetaMessageStream`（避免 O(n²) 的 `partialParse()`）：

```typescript
for await (const part of stream) {
  resetStreamIdleTimer()  // 重置空闲超时

  switch (part.type) {
    case 'message_start':
      // 初始化 partialMessage、usage、TTFB 计时
      partialMessage = part.message
      ttftMs = Date.now() - start
      usage = updateUsage(usage, part.message?.usage)
      break

    case 'content_block_start':
      // 初始化 contentBlock（text/tool_use/thinking/server_tool_use）
      switch (part.content_block.type) {
        case 'tool_use':
          contentBlocks[part.index] = { ...part.content_block, input: '' }
          break
        case 'thinking':
          contentBlocks[part.index] = { ...part.content_block, thinking: '', signature: '' }
          break
        // ...
      }
      break

    case 'content_block_delta':
      // 累加增量内容
      switch (delta.type) {
        case 'input_json_delta':  contentBlock.input += delta.partial_json; break
        case 'text_delta':        contentBlock.text += delta.text; break
        case 'thinking_delta':    contentBlock.thinking += delta.thinking; break
        case 'signature_delta':   contentBlock.signature = delta.signature; break
      }
      break

    case 'content_block_stop':
      // 构造 AssistantMessage 并 yield
      const m: AssistantMessage = {
        message: { ...partialMessage, content: normalizeContentFromAPI([contentBlock], tools, agentId) },
        requestId: streamRequestId,
        type: 'assistant',
        uuid: randomUUID(),
        timestamp: new Date().toISOString(),
      }
      newMessages.push(m)
      yield m
      break

    case 'message_delta':
      // 更新 usage、stop_reason、计算成本
      usage = updateUsage(usage, part.usage)
      stopReason = part.delta.stop_reason
      costUSD += calculateUSDCost(resolvedModel, usage)
      break
  }

  // yield 原始流事件供 UI 渲染
  yield { type: 'stream_event', event: part }
}
```

### 8.3 流式空闲超时看门狗

```typescript
// 可配置的流空闲超时
const STREAM_IDLE_TIMEOUT_MS =
  parseInt(process.env.CLAUDE_STREAM_IDLE_TIMEOUT_MS || '', 10) || 90_000

// 启用：CLAUDE_ENABLE_STREAM_WATCHDOG=1
// 功能：如果 N 秒无新 chunk，强制中止流并触发非流式降级
```

### 8.4 流式到非流式降级

当流式请求出错时（非用户中止），自动降级为非流式：

```typescript
// 捕获流式错误后
catch (streamingError) {
  if (streamingError instanceof APIUserAbortError) throw streamingError

  // 非流式降级
  const fallbackResult = yield* executeNonStreamingRequest(
    clientOptions,
    retryOptions,
    paramsFromContext,
    onAttempt,
    captureRequest,
    streamRequestId,
  )
  // 用 fallbackResult 构造 AssistantMessage
}
```

---

## 9. 重试与容错机制 withRetry

### 9.1 重试生成器

```typescript
// src/services/api/withRetry.ts
export async function* withRetry<T>(
  getClient: () => Promise<Anthropic>,
  operation: (client: Anthropic, attempt: number, context: RetryContext) => Promise<T>,
  options: RetryOptions,
): AsyncGenerator<SystemAPIErrorMessage, T> {
  const maxRetries = getMaxRetries(options)  // 默认 10
  let consecutive529Errors = options.initialConsecutive529Errors ?? 0

  for (let attempt = 1; attempt <= maxRetries + 1; attempt++) {
    try {
      client ??= await getClient()
      return await operation(client, attempt, retryContext)
    } catch (error) {
      // 错误分类处理
      if (error instanceof APIUserAbortError) throw error

      if (is529Error(error)) {
        // 容量过载 → 有限重试
        consecutive529Errors++
        if (consecutive529Errors > MAX_529_RETRIES) {
          // 超过阈值 → 尝试模型降级
          if (options.fallbackModel) {
            throw new FallbackTriggeredError(options.model, options.fallbackModel)
          }
          throw new CannotRetryError(error, retryContext)
        }
      }

      if (error.status === 429) {
        // 速率限制 → 读取 Retry-After 头
        const retryAfter = getRetryAfterMs(error)
        await sleep(retryAfter || calculateBackoff(attempt))
      }

      if (error.status === 401) {
        // 认证失败 → 清除凭证缓存、重新获取
        clearApiKeyHelperCache()
        clearAwsCredentialsCache()
        client = null  // 强制重新创建客户端
      }

      if (isStaleConnectionError(error)) {
        // 陈旧连接 → 禁用 Keep-Alive 重试
        disableKeepAlive()
        client = null
      }

      // 指数退避
      await sleep(calculateBackoff(attempt))
    }
  }
}
```

### 9.2 退避策略

```typescript
const BASE_DELAY_MS = 500

function calculateBackoff(attempt: number): number {
  // 指数退避 + 随机抖动
  const delay = BASE_DELAY_MS * Math.pow(2, attempt - 1)
  const jitter = Math.random() * BASE_DELAY_MS
  return Math.min(delay + jitter, MAX_BACKOFF_MS)
}
```

### 9.3 529 重试源过滤

只有前台交互查询才重试 529（容量过载），后台任务直接失败：

```typescript
const FOREGROUND_529_RETRY_SOURCES = new Set([
  'repl_main_thread',
  'sdk',
  'agent:custom', 'agent:default', 'agent:builtin',
  'compact',
  'hook_agent', 'hook_prompt',
  'verification_agent',
  'side_question',
  'auto_mode',
])
```

### 9.4 持久重试模式

```typescript
// CLAUDE_CODE_UNATTENDED_RETRY: 无人值守模式
// 429/529 无限重试，高退避上限（5 分钟），每 30 秒心跳
const PERSISTENT_MAX_BACKOFF_MS = 5 * 60 * 1000
const PERSISTENT_RESET_CAP_MS = 6 * 60 * 60 * 1000
const HEARTBEAT_INTERVAL_MS = 30_000
```

### 9.5 模型降级（Fallback）

```typescript
// FallbackTriggeredError → 切换到 fallbackModel 重试
class FallbackTriggeredError extends Error {
  constructor(
    public readonly originalModel: string,
    public readonly fallbackModel: string,
  ) {}
}

// 触发条件：
// 1. 连续 529 超过阈值 (MAX_529_RETRIES = 3)
// 2. 有配置 fallbackModel
// 3. 在 query.ts 中捕获该错误，使用 fallbackModel 重新调用
```

---

## 10. Prompt Cache 与成本优化

### 10.1 Prompt Cache 控制

```typescript
export function getPromptCachingEnabled(model: string): boolean {
  if (isEnvTruthy(process.env.DISABLE_PROMPT_CACHING)) return false
  if (isEnvTruthy(process.env.DISABLE_PROMPT_CACHING_HAIKU) && model === smallFastModel) return false
  if (isEnvTruthy(process.env.DISABLE_PROMPT_CACHING_SONNET) && model === defaultSonnet) return false
  if (isEnvTruthy(process.env.DISABLE_PROMPT_CACHING_OPUS) && model === defaultOpus) return false
  return true
}
```

### 10.2 Cache Control 标记

```typescript
export function getCacheControl({ scope, querySource }) {
  return {
    type: 'ephemeral',
    ...(should1hCacheTTL(querySource) && { ttl: '1h' }),  // 1 小时缓存
    ...(scope === 'global' && { scope }),                   // 全局缓存范围
  }
}
```

### 10.3 1 小时 Cache TTL

- Bedrock 用户：通过 `ENABLE_PROMPT_CACHING_1H_BEDROCK=1` 启用
- 1P 用户：Ant 用户或未超额的订阅用户自动启用
- 通过 GrowthBook 配置的 allowlist 控制哪些 querySource 可用

### 10.4 Global Cache Scope

通过 `prompt_caching_scope` beta 头启用全局（跨用户）缓存，当无 MCP 工具时可使用。

### 10.5 Cache Break 检测

```typescript
// src/services/api/promptCacheBreakDetection.ts
// 追踪 prompt 状态变化，检测缓存命中率异常
recordPromptState({ system, toolSchemas, model, betas, ... })
checkResponseForCacheBreak(querySource, cache_read, cache_creation, ...)
```

### 10.6 Cached Microcompact（缓存编辑）

避免压缩操作破坏 prompt cache — 通过 `cache_edits` 在服务端编辑缓存而非重建。

---

## 11. Thinking（思考链）配置体系

### 11.1 ThinkingConfig 类型

```typescript
// src/utils/thinking.ts
type ThinkingConfig =
  | { type: 'disabled' }
  | { type: 'enabled'; budgetTokens?: number }
```

### 11.2 Thinking 参数构造

```typescript
// claude.ts → paramsFromContext()
if (hasThinking && modelSupportsThinking(model)) {
  if (modelSupportsAdaptiveThinking(model)) {
    // 自适应思考（无需预设 budget）
    thinking = { type: 'adaptive' }
  } else {
    // 传统思考（需要 budget）
    let thinkingBudget = getMaxThinkingTokensForModel(model)
    if (thinkingConfig.budgetTokens !== undefined) {
      thinkingBudget = thinkingConfig.budgetTokens
    }
    thinkingBudget = Math.min(maxOutputTokens - 1, thinkingBudget)
    thinking = { type: 'enabled', budget_tokens: thinkingBudget }
  }
}
```

### 11.3 相关环境变量

| 变量 | 作用 |
|------|------|
| `CLAUDE_CODE_DISABLE_THINKING` | 完全禁用思考链 |
| `CLAUDE_CODE_DISABLE_ADAPTIVE_THINKING` | 禁用自适应思考，回退到 budget 模式 |

---

## 12. Beta 特性与动态头管理

### 12.1 Beta 头构建

```typescript
const betas = getMergedBetas(options.model, { isAgenticQuery })
// 包含：
// - 模型基础 beta（如 extended-thinking）
// - Advisor beta
// - Tool Search beta
// - Prompt Caching Scope beta
// - Context Management beta
// - Effort beta
// - Task Budgets beta
// - Fast Mode beta
// - AFK Mode beta
// - Cache Editing beta
// - Structured Outputs beta
// - 1M Context beta
```

### 12.2 Sticky-on Latches（粘滞锁存）

部分 beta 头使用"锁存"机制，一旦首次启用就在整个会话中保持，防止切换导致缓存失效：

```typescript
// 示例：Fast Mode header latch
let fastModeHeaderLatched = getFastModeHeaderLatched() === true
if (!fastModeHeaderLatched && isFastMode) {
  fastModeHeaderLatched = true
  setFastModeHeaderLatched(true)  // 持久化到 bootstrap state
}
// 后续所有请求都会带上该头，即使 fastMode 被关闭
```

涉及的锁存头：`fastMode`、`afkMode`、`cacheEditing`、`thinkingClear`

---

## 13. Token 预算与上下文管理

### 13.1 Task Budget

API 侧 token 预算，让模型自行控制节奏：

```typescript
type TaskBudgetParam = {
  type: 'tokens'
  total: number      // 总预算
  remaining?: number // 剩余预算（压缩后更新）
}

// 通过 output_config.task_budget 发送
configureTaskBudgetParams(taskBudget, outputConfig, betas)
```

### 13.2 Max Output Tokens

```typescript
const maxOutputTokens =
  retryContext.maxTokensOverride ||     // 重试上下文覆盖
  options.maxOutputTokensOverride ||    // 用户配置覆盖
  getMaxOutputTokensForModel(model)     // 模型默认值

// 可通过环境变量配置：
// CLAUDE_CODE_MAX_OUTPUT_TOKENS
```

### 13.3 上下文窗口管理

```typescript
// API 侧上下文管理策略
const contextManagement = getAPIContextManagement({
  hasThinking,
  isRedactThinkingActive,
  clearAllThinking: thinkingClearLatched,
})
// 通过 context_management 参数发送给 API
```

### 13.4 非流式降级 Token 限制

```typescript
const MAX_NON_STREAMING_TOKENS = 3000  // 非流式请求最大输出
function adjustParamsForNonStreaming(params, maxTokens) {
  // 降低 max_tokens 以减少非流式请求的延迟
}
```

---

## 14. Tool Schema 构建与序列化

### 14.1 Tool → API Schema

```typescript
// src/utils/api.ts
export async function toolToAPISchema(
  tool: Tool,
  options: { getToolPermissionContext, tools, agents, model, deferLoading, cacheControl },
): Promise<BetaToolUnion> {
  let input_schema = tool.inputJSONSchema || zodToJsonSchema(tool.inputSchema)

  // 过滤 Swarm 字段（非 EAP 用户）
  if (!isAgentSwarmsEnabled()) {
    input_schema = filterSwarmFieldsFromSchema(tool.name, input_schema)
  }

  const base = {
    name: tool.name,
    description: await tool.prompt({...}),
    input_schema,
    strict: tool.strict && modelSupportsStructuredOutputs(model),
    defer_loading: deferLoading,       // 延迟加载（tool search）
    eager_input_streaming: enabled,     // 细粒度工具流（仅 1P）
    cache_control: cacheControl,       // 缓存控制
  }
  return base
}
```

### 14.2 Tool Schema 缓存

```typescript
// 对话内复用缓存：防止 GrowthBook 翻转导致 tool schema 变化
const cache = getToolSchemaCache()
let base = cache.get(cacheKey)
if (!base) {
  base = buildSchema(tool)
  cache.set(cacheKey, base)
}
```

---

## 15. 模型能力检测与上下文窗口

### 15.1 动态能力刷新

```typescript
// src/utils/model/modelCapabilities.ts
export async function refreshModelCapabilities(): Promise<void> {
  // 仅 ant 用户 + firstParty 启用
  const anthropic = await getAnthropicClient({ maxRetries: 1 })
  for await (const entry of anthropic.models.list({ betas })) {
    parsed.push({ id, max_input_tokens, max_tokens })
  }
  // 缓存到 ~/.claude/cache/model-capabilities.json
}

export function getModelCapability(model: string): ModelCapability | undefined {
  // 精确匹配 → 子串匹配（最长 ID 优先）
}
```

### 15.2 能力检查函数

```typescript
modelSupportsThinking(model)           // 是否支持思考链
modelSupportsAdaptiveThinking(model)   // 是否支持自适应思考
modelSupportsEffort(model)             // 是否支持 effort 参数
modelSupportsStructuredOutputs(model)  // 是否支持结构化输出
modelSupportsAdvisor(model)            // 是否支持 Advisor 工具
modelSupports1M(model)                 // 是否支持 1M 上下文
isFastModeSupportedByModel(model)      // 是否支持快速模式
```

---

## 16. 环境变量完整参考

### 16.1 Provider 选择

| 环境变量 | 类型 | 说明 |
|---------|------|------|
| `CLAUDE_CODE_USE_BEDROCK` | boolean | 启用 AWS Bedrock 后端 |
| `CLAUDE_CODE_USE_VERTEX` | boolean | 启用 Google Vertex AI 后端 |
| `CLAUDE_CODE_USE_FOUNDRY` | boolean | 启用 Azure Foundry 后端 |

### 16.2 认证

| 环境变量 | 类型 | 说明 |
|---------|------|------|
| `ANTHROPIC_API_KEY` | string | Anthropic API Key |
| `ANTHROPIC_AUTH_TOKEN` | string | Bearer Token 认证 |
| `ANTHROPIC_BASE_URL` | string | 自定义 API 基础 URL（代理） |
| `ANTHROPIC_CUSTOM_HEADERS` | string | 自定义请求头（换行分隔） |
| `AWS_REGION` | string | Bedrock 区域 |
| `AWS_SESSION_TOKEN` | string | AWS 临时凭证 |
| `AWS_BEARER_TOKEN_BEDROCK` | string | Bedrock Bearer Token |
| `ANTHROPIC_VERTEX_PROJECT_ID` | string | Vertex 项目 ID |
| `ANTHROPIC_FOUNDRY_API_KEY` | string | Foundry API Key |
| `CLAUDE_CODE_SKIP_VERTEX_AUTH` | boolean | 跳过 Vertex 认证 |

### 16.3 模型选择

| 环境变量 | 类型 | 说明 |
|---------|------|------|
| `ANTHROPIC_MODEL` | string | 主模型名称或别名 |
| `ANTHROPIC_SMALL_FAST_MODEL` | string | 小型快速模型 |
| `ANTHROPIC_DEFAULT_OPUS_MODEL` | string | Opus 别名指向 |
| `ANTHROPIC_DEFAULT_SONNET_MODEL` | string | Sonnet 别名指向 |
| `ANTHROPIC_DEFAULT_HAIKU_MODEL` | string | Haiku 别名指向 |
| `ANTHROPIC_SMALL_FAST_MODEL_AWS_REGION` | string | Haiku Bedrock 区域 |

### 16.4 请求参数

| 环境变量 | 类型 | 说明 |
|---------|------|------|
| `CLAUDE_CODE_MAX_OUTPUT_TOKENS` | number | 最大输出 token 数 |
| `CLAUDE_CODE_EXTRA_BODY` | JSON | 注入额外请求参数 |
| `CLAUDE_CODE_EXTRA_METADATA` | JSON | 注入额外元数据 |
| `CLAUDE_CODE_DISABLE_THINKING` | boolean | 禁用思考链 |
| `CLAUDE_CODE_DISABLE_ADAPTIVE_THINKING` | boolean | 禁用自适应思考 |
| `API_TIMEOUT_MS` | number | API 请求超时（毫秒） |

### 16.5 缓存控制

| 环境变量 | 类型 | 说明 |
|---------|------|------|
| `DISABLE_PROMPT_CACHING` | boolean | 全局禁用 prompt 缓存 |
| `DISABLE_PROMPT_CACHING_HAIKU` | boolean | 禁用 Haiku 缓存 |
| `DISABLE_PROMPT_CACHING_SONNET` | boolean | 禁用 Sonnet 缓存 |
| `DISABLE_PROMPT_CACHING_OPUS` | boolean | 禁用 Opus 缓存 |
| `ENABLE_PROMPT_CACHING_1H_BEDROCK` | boolean | Bedrock 启用 1h 缓存 |

### 16.6 重试与流控制

| 环境变量 | 类型 | 说明 |
|---------|------|------|
| `CLAUDE_CODE_UNATTENDED_RETRY` | boolean | 无人值守持久重试 |
| `CLAUDE_CODE_DISABLE_NONSTREAMING_FALLBACK` | boolean | 禁用非流式降级 |
| `CLAUDE_ENABLE_STREAM_WATCHDOG` | boolean | 启用流空闲看门狗 |
| `CLAUDE_STREAM_IDLE_TIMEOUT_MS` | number | 流空闲超时（默认 90s） |
| `CLAUDE_CODE_DISABLE_LEGACY_MODEL_REMAP` | boolean | 禁用旧模型自动重映射 |

---

## 17. 自定义调用其他大模型指南

### 17.1 方案一：使用 ANTHROPIC_BASE_URL（推荐）

**最简单的方法**：通过兼容 Anthropic API 格式的代理中间层（如 LiteLLM、OpenRouter、One API）来接入其他模型。

```bash
# 设置代理地址
export ANTHROPIC_BASE_URL="http://localhost:4000"  # LiteLLM 代理
export ANTHROPIC_API_KEY="your-proxy-key"

# 指定模型（使用代理支持的模型名）
export ANTHROPIC_MODEL="gpt-4o"          # 通过 LiteLLM
export ANTHROPIC_MODEL="deepseek-v3"     # 通过 LiteLLM

# 禁用不兼容的特性
export CLAUDE_CODE_DISABLE_THINKING=1    # 非 Claude 模型不支持
export DISABLE_PROMPT_CACHING=1          # 非 Claude 模型不支持
```

**原理**：Claude Code 检测到 `ANTHROPIC_BASE_URL` 不是 `api.anthropic.com` 时：
- `isFirstPartyAnthropicBaseUrl()` 返回 false
- 不发送 firstParty-only 的 beta 头
- 不注入 `x-client-request-id`
- 但仍使用 Anthropic SDK 的请求格式

**适用模型**：任何能通过代理层转换为 Anthropic API 格式的模型。

### 17.2 方案二：使用 Bedrock 接入

```bash
# 启用 Bedrock
export CLAUDE_CODE_USE_BEDROCK=1
export AWS_REGION="us-east-1"

# AWS 凭证（使用标准 AWS 认证链）
export AWS_ACCESS_KEY_ID="..."
export AWS_SECRET_ACCESS_KEY="..."
# 或使用 AWS 配置文件、IAM Role 等

# 可选：指定 Bedrock 上的特定模型
export ANTHROPIC_MODEL="us.anthropic.claude-opus-4-6-v1"
```

### 17.3 方案三：使用 Vertex AI 接入

```bash
# 启用 Vertex
export CLAUDE_CODE_USE_VERTEX=1
export ANTHROPIC_VERTEX_PROJECT_ID="my-gcp-project"
export GOOGLE_CLOUD_PROJECT="my-gcp-project"

# GCP 认证：使用 Application Default Credentials
gcloud auth application-default login
# 或设置 GOOGLE_APPLICATION_CREDENTIALS

# 可选：指定区域
# 通过 getVertexRegionForModel() 自动选择或通过设置覆盖
```

### 17.4 方案四：使用 Foundry (Azure) 接入

```bash
# 启用 Foundry
export CLAUDE_CODE_USE_FOUNDRY=1

# 认证方式一：Azure AD
# 使用 DefaultAzureCredential 自动认证

# 认证方式二：API Key
export ANTHROPIC_FOUNDRY_API_KEY="your-azure-key"
```

### 17.5 方案五：深度定制（修改源码）

如果需要接入完全不同 API 格式的模型（如 OpenAI 原生、本地模型），需要修改核心文件：

#### 步骤 1：扩展 Provider 类型

```typescript
// src/utils/model/providers.ts
type APIProvider = 'firstParty' | 'bedrock' | 'vertex' | 'foundry' | 'openai' | 'local'

export function getAPIProvider(): APIProvider {
  if (isEnvTruthy(process.env.CLAUDE_CODE_USE_OPENAI)) return 'openai'
  if (isEnvTruthy(process.env.CLAUDE_CODE_USE_LOCAL)) return 'local'
  // ...原有逻辑
}
```

#### 步骤 2：添加客户端工厂

```typescript
// src/services/api/client.ts
if (isEnvTruthy(process.env.CLAUDE_CODE_USE_OPENAI)) {
  // 创建一个 Anthropic API 兼容的适配器
  // 将 Anthropic 请求格式转换为 OpenAI 格式
  return createOpenAIAdapter(ARGS)
}
```

#### 步骤 3：添加模型配置

```typescript
// src/utils/model/configs.ts
export const CUSTOM_MODEL_CONFIG = {
  firstParty: 'custom-model-name',
  bedrock: 'custom-model-name',
  vertex: 'custom-model-name',
  foundry: 'custom-model-name',
  openai: 'gpt-4o',            // 新增
  local: 'llama-3.3-70b',      // 新增
} as const satisfies ModelConfig
```

#### 步骤 4：处理能力差异

关键差异点需要处理：

| 特性 | Claude 支持 | 需处理 |
|------|-----------|--------|
| Thinking（思考链） | ✅ | 非 Claude 模型需禁用 |
| Prompt Caching | ✅ | 非 Claude 需禁用 |
| Tool Use (Beta) | ✅ | 转换为目标模型的 function calling |
| Streaming | ✅ | 转换 SSE 事件格式 |
| cache_control | ✅ | 移除 |
| content_block 粒度事件 | ✅ | 转换为对应事件 |
| signature (思考签名) | ✅ | 移除 |

### 17.6 方案六：通过 CLAUDE_CODE_EXTRA_BODY 注入参数

```bash
# 注入自定义参数到每个 API 请求
export CLAUDE_CODE_EXTRA_BODY='{"anthropic_version":"2023-06-01","custom_param":"value"}'
```

### 17.7 关键注意事项

1. **Tool Use 格式**：Claude Code 大量依赖 Anthropic 的 `tool_use`/`tool_result` 格式，切换模型时必须确保代理层正确转换
2. **Thinking 支持**：大部分非 Claude 模型不支持 `thinking` 参数，务必设置 `CLAUDE_CODE_DISABLE_THINKING=1`
3. **Beta 头**：非标准 API 可能拒绝未知的 beta 头，代理层需要过滤
4. **Streaming 格式**：Claude Code 期望 Anthropic 格式的 SSE 事件（`message_start`、`content_block_start` 等），代理层需要正确转换
5. **模型能力**：部分高级功能（Advisor、Fast Mode、Effort）仅部分模型支持，切换模型后自动降级

---

## 18. 架构设计启示

### 18.1 分层解耦

Claude Code 的 LLM 调用架构体现了优秀的分层设计：

```
╔════════════════════════════════════════╗
║  应用层 (QueryEngine)                  ║  会话管理、用户交互
╠════════════════════════════════════════╣
║  业务层 (query.ts)                     ║  多轮循环、工具执行、预算控制
╠════════════════════════════════════════╣
║  协议层 (claude.ts)                    ║  请求构造、消息规范化、流处理
╠════════════════════════════════════════╣
║  容错层 (withRetry.ts)                 ║  重试、降级、认证刷新
╠════════════════════════════════════════╣
║  传输层 (client.ts)                    ║  SDK 客户端、多 Provider 适配
╚════════════════════════════════════════╝
```

### 18.2 核心设计模式

1. **AsyncGenerator 贯穿全链路**：从 `QueryEngine.submitMessage()` 到 `queryModel()`，全部使用 AsyncGenerator 实现流式传输，优雅地支持了背压（backpressure）控制

2. **Provider 策略模式**：通过 `getAPIProvider()` + 客户端工厂实现多后端透明切换，上层代码无需感知底层差异

3. **环境变量驱动配置**：几乎所有行为都可通过环境变量控制，无需修改代码即可定制

4. **Sticky Latch 防缓存抖动**：Beta 头锁存机制防止会话中途的配置翻转导致 prompt cache 失效

5. **优雅降级链**：流式 → 非流式 → 模型降级 → 错误报告，多级容错确保可用性

6. **依赖注入**：`QueryParams.deps` 允许注入自定义的 `callModel`、`autocompact`、`microcompact` 等函数，便于测试和定制

### 18.3 性能优化亮点

- **Prompt Cache**：多级缓存策略（ephemeral / 1h TTL / global scope）减少重复计费
- **Cache Editing**：服务端编辑缓存避免压缩导致的缓存失效
- **Tool Schema 缓存**：会话内复用，防止 flag 翻转导致的字节变化
- **fine-grained tool streaming**：避免大型 tool input 的长时间卡顿
- **Streaming 空闲看门狗**：检测并恢复僵死连接

---

> 本报告基于 Claude Code v2.1.88 源码分析，涵盖了 LLM 调用机制的核心架构、多 Provider 支持、请求构造、流式处理、重试容错、缓存优化等关键模块，并提供了自定义接入其他大模型的 6 种方案。
# Claude Code 大模型调用机制研究报告

## 1. 研究目标

本文基于当前工程源码，系统梳理 Claude Code 在运行时如何完成以下工作：

1. 会话请求如何从 `QueryEngine` 进入主查询循环。
2. 请求如何被组织成 Anthropic Messages API 所需的数据结构。
3. 模型、Provider、Beta Header、Thinking、Tool Schema、Prompt Caching 如何参与请求构建。
4. 流式响应、非流式回退、重试、Fallback、限流恢复如何实现。
5. 当前架构下，如何“自定义调用其他大模型”。
6. 如果要真正扩展到非 Anthropic 协议模型，需要改哪些核心点。

---

## 2. 总体结论

这套系统本质上是一个**围绕 Anthropic Messages API 构建的 Agent Runtime**，其关键特征不是“简单发个 prompt”，而是：

- 以 `query.ts` 为主循环，驱动多轮对话、工具调用、压缩、恢复、预算控制。
- 以 `src/services/api/claude.ts` 为 LLM 请求装配中心，负责把内部消息、系统提示、工具、thinking、betas、缓存策略拼成最终 API 请求。
- 以 `src/services/api/client.ts` 为 Provider 客户端工厂，按环境变量切到 first-party / Bedrock / Vertex / Foundry。
- 以 `src/services/api/withRetry.ts` 为统一重试与故障恢复层，处理 429/529、认证刷新、context overflow、模型 fallback 等复杂场景。
- 以 `src/utils/model/*` 为模型解析层，统一管理别名、默认模型、Provider 映射、能力判断、用户覆盖。

因此，**Claude Code 并不是“随便替换一个 base URL 就能兼容任意大模型”的架构**。它对以下 Anthropic 语义存在深度耦合：

- `messages` / `system` 的请求结构
- `tool_use` / `tool_result` 内容块协议
- `thinking` / `redacted_thinking`
- `betas` 头与 `output_config`
- 流式事件类型（如 `message_start`、`content_block_delta`、`message_delta`）
- prompt caching / cache_control / context_management 等 Anthropic 专属能力

所以，“自定义调用其他大模型”有三种可行层级：

1. **最小改造**：接入 Anthropic 兼容网关或代理。
2. **中等改造**：在现有 Provider 体系中新增一种 Anthropic 风格 Provider。
3. **深度改造**：抽象出通用 LLM Adapter，把当前 Anthropic 协议从 Runtime 中解耦。

---

## 3. 核心调用链路

### 3.1 顶层入口

调用链的主干如下：

```text
QueryEngine.submit/query
  -> query.ts: query()/queryLoop()
    -> deps.callModel(...)
      -> src/services/api/claude.ts: queryModelWithStreaming/queryModelWithoutStreaming
        -> queryModel(...)
          -> getAnthropicClient(...)
          -> anthropic.beta.messages.create(...)
```

这条链路可以拆成四层：

1. **会话层**：`QueryEngine.ts`
   - 负责组合会话上下文、系统提示、工具集、MCP、Agent 定义、预算等。
   - 负责 transcript 持久化、resume、命令模式、附加上下文装配。

2. **主循环层**：`query.ts`
   - 负责一轮轮推进对话。
   - 负责 auto-compact、reactive compact、context collapse、tool execution、token budget continuation。
   - 负责处理模型响应后的 tool_use，然后把 tool_result 重新送回模型。

3. **API 编排层**：`claude.ts`
   - 负责将内部 `Message[]` 规范化为 API 请求。
   - 负责构建 `system`、`tools`、`thinking`、`betas`、`output_config`、`metadata`、`cache_control`。
   - 负责消费流式事件并回写成内部 `AssistantMessage`。

4. **Provider 客户端层**：`client.ts`
   - 负责根据运行环境选择具体 SDK 客户端。
   - 负责认证、请求头、代理、fetch 包装、请求 ID 注入。

---

## 4. QueryEngine：会话级调度入口

`src/QueryEngine.ts` 是会话运行的组织者，`QueryEngineConfig` 中定义了大量运行期参数，例如：

- `cwd`
- `tools`
- `commands`
- `mcpClients`
- `agents`
- `customSystemPrompt`
- `appendSystemPrompt`
- `userSpecifiedModel`
- `fallbackModel`
- `thinkingConfig`
- `maxTurns`
- `maxBudgetUsd`
- `taskBudget`
- `jsonSchema`

它的职责不是直接发请求，而是先把“当前会话应该怎么跑”定下来：

- 使用哪个主模型
- 是否允许工具
- 当前工具权限模式是什么
- 有没有附加系统提示
- 是否在 coordinator / subagent / hook 场景下运行
- 是否有结构化输出要求

可以把 `QueryEngine` 理解为**Agent 执行上下文的组装器**。

---

## 5. query.ts：真正的主循环引擎

`src/query.ts` 是整个系统最核心的控制循环。

### 5.1 QueryParams 的角色

`QueryParams` 携带的不是单个 prompt，而是一整套运行上下文，包括：

- `messages`
- `systemPrompt`
- `userContext`
- `systemContext`
- `toolUseContext`
- `fallbackModel`
- `querySource`
- `maxOutputTokensOverride`
- `maxTurns`
- `taskBudget`
- `deps`

这说明系统从设计上就是**面向“多轮 Agent 任务”而不是单轮问答**。

### 5.2 queryLoop 的状态机

循环状态 `State` 里包含：

- 当前消息序列 `messages`
- 工具上下文 `toolUseContext`
- auto compact 跟踪信息
- `maxOutputTokensRecoveryCount`
- 是否尝试过 reactive compact
- `maxOutputTokensOverride`
- `pendingToolUseSummary`
- `stopHookActive`
- 当前 turn 计数
- 上一轮继续原因 `transition`

主循环每一轮大致做这些事：

1. 预取 memory / skill
2. 对历史消息做预算治理：tool result budget、snip、microcompact、context collapse、autocompact
3. 构造系统 prompt
4. 判断是否达到 blocking limit
5. 发起模型请求
6. 流式接收 assistant 消息
7. 如果包含 tool_use，则执行工具
8. 把 tool_result 回填到消息序列，进入下一轮
9. 如果达到停止条件，则返回 terminal reason

### 5.3 Token 与上下文控制是内建能力

Claude Code 的主循环并不是“上下文满了就报错”，而是内建了多层恢复策略：

- `snip`
- `microcompact`
- `context collapse`
- `autocompact`
- `reactive compact`
- `max_output_tokens` 恢复
- `taskBudget.remaining` 跨压缩边界续传

这意味着它是一个**会主动维护上下文窗口健康度的运行时**。

---

## 6. client.ts：Provider 客户端工厂

`src/services/api/client.ts` 是模型调用真正下沉到 SDK 的地方。

### 6.1 Provider 选择方式

Provider 类型由 `src/utils/model/providers.ts` 决定：

- `firstParty`
- `bedrock`
- `vertex`
- `foundry`

选择逻辑来自环境变量：

- `CLAUDE_CODE_USE_BEDROCK`
- `CLAUDE_CODE_USE_VERTEX`
- `CLAUDE_CODE_USE_FOUNDRY`

都不启用时，默认是 `firstParty`。

### 6.2 first-party 模式

默认使用 `@anthropic-ai/sdk`。

认证来源：

- `ANTHROPIC_API_KEY`
- 或 Claude AI OAuth token（订阅用户）

额外特征：

- 支持 `ANTHROPIC_BASE_URL` 自定义 API 地址
- 支持 `ANTHROPIC_CUSTOM_HEADERS`
- 支持代理 fetch
- 自动注入 `x-client-request-id`

### 6.3 Bedrock 模式

使用 `@anthropic-ai/bedrock-sdk`。

关键点：

- 可自动刷新 AWS 凭证
- 可通过 `AWS_REGION` / `AWS_DEFAULT_REGION` 指定区域
- 可通过 `AWS_BEARER_TOKEN_BEDROCK` 使用 Bearer Token
- 可通过 `ANTHROPIC_BEDROCK_BASE_URL` 指向自定义 Bedrock endpoint
- 对 inference profile 有专门支持

### 6.4 Vertex 模式

使用 `@anthropic-ai/vertex-sdk` + `google-auth-library`。

关键点：

- 默认走 GoogleAuth
- 可跳过认证：`CLAUDE_CODE_SKIP_VERTEX_AUTH`
- 会优先尝试环境变量 / keyfile / projectId，尽量避免 metadata server 的 12 秒超时
- 区域由模型映射函数决定

### 6.5 Foundry 模式

使用 `@anthropic-ai/foundry-sdk`。

关键点：

- 可用 `ANTHROPIC_FOUNDRY_API_KEY`
- 或 Azure AD Token Provider
- deployment/model id 与 Anthropic public model id 不必一致

### 6.6 fetch 包装

`buildFetch()` 会做两件非常重要的事：

1. 注入客户端 request ID，方便跟踪 timeout / server log。
2. 记录请求路径和 source，用于调试和遥测。

因此，SDK 调用并不是直接裸发，而是带有一层**工程化可观测性包装**。

---

## 7. 模型系统：别名、默认值、Provider 映射

模型系统集中在 `src/utils/model/*`。

### 7.1 Provider 映射

`configs.ts` 维护 canonical model config，例如：

- `firstParty: claude-sonnet-4-6`
- `bedrock: us.anthropic.claude-sonnet-4-6`
- `vertex: claude-sonnet-4-6`
- `foundry: claude-sonnet-4-6`

`modelStrings.ts` 负责按当前 Provider 输出一套“运行期模型字符串”。

### 7.2 模型选择优先级

从源码看，主模型解析优先级大致为：

1. `/model` 命令显式选择
2. `--model` 参数
3. `ANTHROPIC_MODEL`
4. settings 中的模型设置
5. 内建默认模型

### 7.3 模型别名

`aliases.ts` 中定义了：

- `sonnet`
- `opus`
- `haiku`
- `best`
- `sonnet[1m]`
- `opus[1m]`
- `opusplan`

`parseUserSpecifiedModel()` 会把这些别名解析成最终模型 ID。

### 7.4 自定义覆盖点

已有源码中最实用的自定义能力有两个：

1. `modelOverrides`（settings）
   - 用 canonical first-party model id -> provider-specific string 做覆盖
   - 特别适合 Bedrock inference profile ARN

2. `ANTHROPIC_DEFAULT_*_MODEL`
   - 可覆盖默认 Opus / Sonnet / Haiku 模型

这意味着在**不改代码**的前提下，已经可以做一部分“换模型”能力。

---

## 8. claude.ts：请求构建中心

`src/services/api/claude.ts` 是本项目里最关键的 LLM 调用文件。

它做的事情非常多，可以理解为“Anthropic Request Compiler + Stream Runtime”。

### 8.1 主要职责

1. 构造 metadata
2. 构造 system prompt blocks
3. 规范化 messages
4. 计算 betas
5. 构造 tool schema
6. 注入 prompt caching 标记
7. 注入 thinking / effort / output_config / task_budget
8. 发起 streaming request
9. 解析流式事件
10. 在失败时回退到 non-streaming
11. 记录 usage / cost / telemetry / request id

### 8.2 getExtraBodyParams：额外请求体扩展点

`CLAUDE_CODE_EXTRA_BODY` 是一个非常重要的扩展点。

它允许用户传入 JSON 对象，并被浅拷贝后合并进 API 请求体。

支持场景：

- 注入额外 Anthropic body 参数
- 对接某些代理层的扩展字段
- Bedrock 下附带 beta body 参数

这是**低侵入扩展请求参数**的最佳入口之一。

### 8.3 metadata

`getAPIMetadata()` 会把以下信息打包到 `metadata.user_id`：

- `device_id`
- `account_uuid`
- `session_id`
- 以及 `CLAUDE_CODE_EXTRA_METADATA` 中的额外字段

这主要用于归因和遥测。

### 8.4 消息转换

内部消息会通过：

- `userMessageToMessageParam()`
- `assistantMessageToMessageParam()`
- `normalizeMessagesForAPI()`

被转成最终的 API 结构。

同时会处理：

- cache_control 标记
- tool_use / tool_result 配对修复
- tool_reference 清理
- advisor block 清理
- 超限媒体剥离

说明这个系统在 API 发出前就做了大量“消息面清洗”。

### 8.5 system prompt 构建

系统提示最终来自多段拼接：

- attribution header
- CLI sysprompt prefix
- 原始 system prompt
- advisor 指令
- chrome tool search 指令
- system context 拼接内容

之后再通过 `buildSystemPromptBlocks()` 标上缓存策略。

### 8.6 tool schema 构建

工具不会直接原样发给 API，而是经过 `toolToAPISchema()` 转换。

转换时会考虑：

- 工具描述 prompt
- input schema（zod/json schema）
- strict mode
- defer_loading
- eager_input_streaming
- global/org cache scope
- agent swarms 字段过滤

这说明“工具协议”也是深度跟 Anthropic tool use 模式绑定的。

---

## 9. Thinking / Effort / Task Budget / Structured Output

Claude Code 调用模型时，不只是传 messages。

它还会叠加多个运行时控制项。

### 9.1 Thinking

来自 `src/utils/thinking.ts` 与 `claude.ts` 中的配置逻辑。

支持三种模式：

- `disabled`
- `enabled` + `budget_tokens`
- `adaptive`

核心逻辑：

- 若模型支持 adaptive thinking，则优先使用 adaptive
- 否则用 `budget_tokens`
- budget 不可超过 `max_tokens - 1`

环境变量与控制项：

- `MAX_THINKING_TOKENS`
- `CLAUDE_CODE_DISABLE_THINKING`
- `CLAUDE_CODE_DISABLE_ADAPTIVE_THINKING`

### 9.2 Effort

`configureEffortParams()` 会把 effort 写入 `output_config` 或 internal body 字段。

适用场景：

- 指定模型推理强度
- ant 内部场景下可做数值化 override

### 9.3 Task Budget

`configureTaskBudgetParams()` 会把 `task_budget` 写入 `output_config`。

这和 query loop 里的 token budget 不是一回事：

- query loop token budget：客户端侧控制，决定要不要继续下一轮
- API-side task budget：告诉模型还剩多少任务预算，让模型自己节制输出

### 9.4 Structured Output

如果 `options.outputFormat` 存在，则写入 `output_config.format`。

同时要求模型支持 structured outputs，并自动补 beta header。

这意味着 Claude Code 已经具备“约束 JSON 输出”的基础设施。

---

## 10. Prompt Caching 与 Cache Control

Prompt caching 是该架构的一个关键优化点。

### 10.1 开关逻辑

`getPromptCachingEnabled(model)` 会综合判断：

- `DISABLE_PROMPT_CACHING`
- `DISABLE_PROMPT_CACHING_HAIKU`
- `DISABLE_PROMPT_CACHING_SONNET`
- `DISABLE_PROMPT_CACHING_OPUS`

### 10.2 Cache Control

通过 `getCacheControl()` 构造：

- `type: ephemeral`
- `ttl: 1h`（符合条件时）
- `scope: global`

### 10.3 1 小时缓存策略

`should1hCacheTTL(querySource)` 会考虑：

- 用户是否有资格
- querySource 是否在 allowlist 中
- Bedrock 是否开启 `ENABLE_PROMPT_CACHING_1H_BEDROCK`

### 10.4 Cached Microcompact

系统支持通过 cache_edits 方式减少 prompt 变化，避免缓存击穿。

这说明 Claude Code 的性能优化并不只是 token 计费层面，而是已经深入到“请求字节级稳定性”层面。

---

## 11. Streaming 响应处理机制

`queryModel()` 默认走 streaming。

### 11.1 流式调用方式

通过：

- `anthropic.beta.messages.create({ ...params, stream: true }).withResponse()`

返回原始流和 response。

### 11.2 解析事件

主要处理的事件有：

- `message_start`
- `content_block_start`
- `content_block_delta`
- `content_block_stop`
- `message_delta`
- `message_stop`

系统会自己累积：

- 文本块
- thinking 块
- tool_use input JSON
- advisor server tool 块

最终在 `content_block_stop` 时，转成内部 `AssistantMessage` 并 yield 出去。

### 11.3 流式健壮性控制

为避免 SSE 长时间卡死，专门做了：

- stream idle watchdog
- stall detection
- timeout telemetry
- clean / error 两种退出路径区分

这部分体现出源码作者对“长任务代理场景”的工程经验非常强。

---

## 12. 非流式回退机制

当 streaming 出错时，不一定直接失败。

系统会尝试走 `executeNonStreamingRequest()`。

### 12.1 回退触发条件

源码中可见几类典型场景：

- 流式过程出错
- 流式创建阶段 404
- 流式结束但没有形成有效 assistant message
- watchdog 主动中断

### 12.2 回退过程

1. 重新构造请求
2. 调用非流式 `anthropic.beta.messages.create(...)`
3. 限制 `MAX_NON_STREAMING_TOKENS`
4. 同步调整 thinking budget
5. 把结果重新转回内部 `AssistantMessage`

### 12.3 关闭回退的方式

- `CLAUDE_CODE_DISABLE_NONSTREAMING_FALLBACK`
- 或 feature gate `tengu_disable_streaming_to_non_streaming_fallback`

这说明回退不是永远强制启用，而是一个可控策略。

---

## 13. withRetry.ts：统一重试与故障恢复框架

`src/services/api/withRetry.ts` 是另一个核心文件。

### 13.1 它处理什么问题

- 429 / 529 限流
- 网络连接错误
- stale keep-alive 连接
- OAuth 401/403 刷新
- AWS / GCP 凭证刷新
- fast mode 降级
- context overflow 缩减 max tokens
- fallback model 切换
- unattended 无限重试模式

### 13.2 重试上下文

`RetryContext` 持有：

- `maxTokensOverride`
- `model`
- `thinkingConfig`
- `fastMode`

这个上下文会被传回 `paramsFromContext()`，使每次 retry 能动态修正请求参数。

### 13.3 429 / 529 策略

源码中把前台 query source 和后台 query source 区分开。

只有前台阻塞用户的 query source，才会对 529 做重试。这样做是为了避免容量故障时后台任务雪崩式放大。

### 13.4 context overflow 自动收缩

如果 API 返回 prompt 太长，系统会尝试解析错误里的 token 信息，并降低 `max_tokens` 重试。

这不是 query loop 的 compact，而是**请求侧的输出 token 自救**。

### 13.5 FallbackTriggeredError

若连续 529 达阈值，且配置了 `fallbackModel`，则抛出 `FallbackTriggeredError`。

之后 query loop 会：

- 清空当前 attempt 的 assistant / tool execution 临时状态
- 切换模型
- 再来一轮请求

这是一种“模型级故障切换”而不是“请求重试”。

---

## 14. 错误系统与日志系统

### 14.1 errors.ts

`src/services/api/errors.ts` 负责把底层 SDK / API 错误翻译成用户可见或系统可消费的错误消息。

涵盖：

- 认证失败
- 余额问题
- 组织禁用
- OAuth token 吊销
- timeout
- 图片/PDF/媒体大小拒绝
- 重复 529
- prompt too long

其中 `isPromptTooLongError()`、`extractPromptTooLongDetails()` 对 reactive compact 很关键。

### 14.2 logging.ts

`src/services/api/logging.ts` 负责：

- 记录 API query 发起
- 记录 API success / duration / usage / cost
- 记录 API error
- 检测 gateway / proxy
- OTLP / tracing 上报

这意味着如果你要接第三方模型网关，最好保证响应头或 base URL 能被这层识别，否则可观测性会下降。

---

## 15. betas.ts：Beta Header 体系

Claude Code 对 beta header 的管理非常成熟。

会根据以下因素自动决定是否发送：

- provider
- model 能力
- user type
- feature gate
- interactive / non-interactive 模式
- tool search / context management / structured output 等运行时特征

这意味着：

1. API 调用不是一个固定协议。
2. 同一个模型在不同会话条件下，可能发出不同的 beta headers。
3. 如果要适配其他模型或代理，必须确认它是否能接受这些 header/body 扩展。

---

## 16. modelCapabilities.ts：能力发现与缓存

该模块会在符合条件时调用 `anthropic.models.list()`，并把模型能力缓存到本地。

可发现能力包括：

- `max_input_tokens`
- `max_tokens`

用途：

- 为运行时能力判断提供数据
- 减少硬编码依赖

但要注意：当前实现仍然是 **Anthropic Models API 风格**，并不是通用模型发现机制。

---

## 17. 为什么当前架构对 Anthropic 深度耦合

要回答“如何接其他大模型”，必须先明确当前耦合点。

### 17.1 请求协议耦合

请求构造完全基于 Anthropic Messages API：

- `system`
- `messages`
- `tools`
- `tool_choice`
- `betas`
- `thinking`
- `output_config`
- `context_management`
- `metadata`

如果目标模型是 OpenAI Responses API、Chat Completions API、Gemini GenerateContent API、Qwen 原生协议，这一层都不能直接复用。

### 17.2 响应流协议耦合

流式事件解析使用的是 Anthropic 原始事件：

- `message_start`
- `content_block_delta`
- `message_delta`

其他模型服务通常不会返回这一套事件。

### 17.3 工具语义耦合

内部工具运行时依赖的就是 `tool_use` / `tool_result` block 语义。

如果目标模型的 function calling 语义不同，就必须做协议转换。

### 17.4 Thinking / Redacted Thinking / Context Management 耦合

这些都不是通用 LLM 标准能力。

如果迁移到其他模型，必须决定：

- 伪装兼容
- 降级忽略
- 还是重新抽象统一能力层

---

## 18. 如何自定义调用其他大模型：三条路径

下面给出最实用的三种方案。

### 方案 A：最小改造，接入 Anthropic 兼容网关

适用场景：

- 你要接的是一个代理层
- 它能模拟 Anthropic Messages API
- 能接受 `tools` / streaming / betas 中至少核心子集

#### 做法

1. 设置 `ANTHROPIC_BASE_URL`
2. 设置 `ANTHROPIC_API_KEY` 或自定义认证头
3. 如有需要，设置 `ANTHROPIC_CUSTOM_HEADERS`
4. 用 `ANTHROPIC_MODEL` 或 settings/modelOverrides 指向目标模型名

#### 优点

- 基本不用改源码
- query loop、tool runtime、compact、retry 都可复用

#### 缺点

- 你的网关必须高度兼容 Anthropic 协议
- thinking / beta / cache_control / context_management 不一定可用
- 如果兼容性不完整，可能在流式或工具阶段出错

#### 适合对象

- LiteLLM 的 Anthropic 路由
- 自建 Anthropic-compatible proxy
- 某些云平台的 Anthropic 兼容层

---

### 方案 B：中等改造，新增一个 Provider

适用场景：

- 目标服务仍然“基本像 Anthropic”
- 但认证、region、endpoint、model id 映射有独特要求

#### 需要改的核心点

1. `src/utils/model/providers.ts`
   - 新增 provider 类型，例如 `custom`
   - 扩展 `getAPIProvider()`

2. `src/services/api/client.ts`
   - 新增 `if (isEnvTruthy(process.env.CLAUDE_CODE_USE_CUSTOM)) { ... }`
   - 创建对应 SDK/client
   - 处理认证、headers、fetch、region

3. `src/utils/model/configs.ts`
   - 为每个 canonical model 增加 `custom` 映射

4. `src/utils/model/modelStrings.ts`
   - 支持 custom provider 的 model string 解析

5. `src/utils/model/model.ts`
   - 校验模型别名 / canonical name / display name 时补齐 custom provider 逻辑

6. `src/utils/betas.ts`
   - 确定 custom provider 支持哪些 beta，哪些要禁掉

#### 优点

- 对现有架构侵入较小
- 保留 Provider 选择机制的一致性
- 适合接“Anthropic 变体”服务

#### 缺点

- 仍然假设底层协议接近 Anthropic
- 对完全不同的模型协议不适用

---

### 方案 C：深度改造，抽象通用 LLM Adapter

适用场景：

- 目标是 OpenAI、Gemini、Qwen、DeepSeek、GLM 等原生非 Anthropic 协议模型
- 你希望 Claude Code Runtime 真正成为多模型 Agent 框架

#### 建议抽象层

建议新增统一接口，例如：

```ts
interface LLMAdapter {
  buildRequest(input: InternalLLMInput): ProviderRequest
  stream(request: ProviderRequest): AsyncGenerator<ProviderStreamEvent>
  complete(request: ProviderRequest): Promise<ProviderResponse>
  normalizeResponse(events: ProviderStreamEvent[]): InternalAssistantMessage[]
  supports(feature: LLMCapability): boolean
}
```

#### 需要解耦的模块

1. `claude.ts`
   - 从“Anthropic 实现”改为“AnthropicAdapter”

2. `query.ts`
   - 不再直接依赖 Anthropic 特定 stop reason / block type
   - 改为依赖内部统一消息模型

3. `utils/api.ts`
   - tool schema 转换从 Anthropic 专用改为 Adapter 下发

4. `withRetry.ts`
   - 把 Anthropic 特定错误解析下沉到 adapter-specific retry policy

5. `errors.ts`
   - 变成 provider-agnostic error normalization

6. `modelCapabilities.ts`
   - 抽象成统一 capability discovery

#### 优点

- 真正支持多模型
- 后续扩展 OpenAI/Gemini/Qwen 更规范

#### 缺点

- 改造量大
- 要重新定义工具调用与 thinking 能力映射
- 测试成本高

---

## 19. 推荐的实际落地路线

如果你的目标是“尽快接入其他模型”，建议按下面顺序做。

### 第一步：先验证 Anthropic 兼容代理路径

优先尝试：

- `ANTHROPIC_BASE_URL`
- `ANTHROPIC_CUSTOM_HEADERS`
- `ANTHROPIC_MODEL`
- settings 中 `modelOverrides`

原因：

- 改动最小
- 最快验证 query loop/tool runtime 是否能继续工作
- 能快速暴露协议兼容缺口

### 第二步：若兼容不够，再新增 Provider

如果你接的目标服务只是认证或 endpoint 逻辑不同，但协议仍接近 Anthropic，就走 Provider 扩展。

这是**性价比最高**的源码改造方式。

### 第三步：只有在明确需要支持多家原生协议时，才做 Adapter 抽象

否则容易把一个成熟稳定的 Anthropic Runtime 重构成长期不稳定分支。

---

## 20. 自定义接入的关键注意事项

### 20.1 不是只有请求兼容就够了

你至少要验证：

- streaming 事件兼容
- tool use 兼容
- JSON/schema 输出兼容
- stop reason 兼容
- 错误码兼容
- retry-after / 429 / 529 语义兼容

### 20.2 很多“高级能力”可能需要关闭

接第三方模型时，建议先禁用这些高级特性做最小可用验证：

- thinking
- structured outputs
- tool search
- context management
- prompt caching
- cache editing
- fast mode
- advisor

否则你很可能在请求构造层就踩到协议不兼容。

### 20.3 模型能力与别名映射要单独维护

如果第三方服务的模型名不是 Claude 官方命名，建议：

- 继续保留 canonical first-party id
- 用 `modelOverrides` 做 provider-specific 映射
- 不要把内部业务逻辑直接写死到第三方模型名上

这样后续切换底层服务时成本最低。

---

## 21. 推荐的最小验证方案

如果你的目的是先证明“这套代码能跑其他模型”，建议按如下顺序验证。

### 方案 1：无代码改动验证

1. 配置 `ANTHROPIC_BASE_URL`
2. 配置 `ANTHROPIC_API_KEY`
3. 配置 `ANTHROPIC_MODEL`
4. 关闭复杂能力：thinking / prompt caching / structured outputs / advisor
5. 先跑无工具纯对话
6. 再跑单工具调用
7. 最后再验证流式与长上下文

### 方案 2：轻代码改动验证

如果目标服务对部分字段敏感，可在 `claude.ts` 中临时裁剪：

- 不发 `betas`
- 不发 `output_config`
- 不发 `context_management`
- 不发 `cache_control`

只保留最小：

- `model`
- `messages`
- `system`
- `tools`
- `tool_choice`
- `max_tokens`
- `temperature`

如果这样能跑通，再逐步恢复高级能力。

---

## 22. 从源码视角看，最佳扩展点在哪里

如果你准备正式改造代码，优先关注以下文件：

1. `src/services/api/client.ts`
   - 入口最清晰，适合新增 provider/client

2. `src/utils/model/providers.ts`
   - Provider 选择开关定义点

3. `src/utils/model/configs.ts`
   - canonical 模型与 provider 模型映射中心

4. `src/services/api/claude.ts`
   - 请求体装配、流式解析、fallback 的主战场

5. `src/services/api/withRetry.ts`
   - 失败恢复与 fallback 策略

6. `src/utils/api.ts`
   - tool schema / message normalize / system prompt block 生成

如果只是“新增一种 Anthropic 风格 Provider”，改动集中在 1~4 即可。

如果是“支持完全不同协议模型”，1~6 全都要动。

---

## 23. 环境变量与自定义点速查

### Provider / Endpoint

- `CLAUDE_CODE_USE_BEDROCK`
- `CLAUDE_CODE_USE_VERTEX`
- `CLAUDE_CODE_USE_FOUNDRY`
- `ANTHROPIC_BASE_URL`
- `ANTHROPIC_API_KEY`
- `ANTHROPIC_CUSTOM_HEADERS`

### Model

- `ANTHROPIC_MODEL`
- `ANTHROPIC_SMALL_FAST_MODEL`
- `ANTHROPIC_DEFAULT_OPUS_MODEL`
- `ANTHROPIC_DEFAULT_SONNET_MODEL`
- `ANTHROPIC_DEFAULT_HAIKU_MODEL`
- settings: `modelOverrides`

### Thinking / Output

- `MAX_THINKING_TOKENS`
- `CLAUDE_CODE_DISABLE_THINKING`
- `CLAUDE_CODE_DISABLE_ADAPTIVE_THINKING`
- `CLAUDE_CODE_MAX_OUTPUT_TOKENS`

### Prompt Caching

- `DISABLE_PROMPT_CACHING`
- `DISABLE_PROMPT_CACHING_HAIKU`
- `DISABLE_PROMPT_CACHING_SONNET`
- `DISABLE_PROMPT_CACHING_OPUS`
- `ENABLE_PROMPT_CACHING_1H_BEDROCK`

### Retry / Fallback

- `API_TIMEOUT_MS`
- `CLAUDE_CODE_DISABLE_NONSTREAMING_FALLBACK`
- `CLAUDE_ENABLE_STREAM_WATCHDOG`
- `CLAUDE_STREAM_IDLE_TIMEOUT_MS`
- `CLAUDE_CODE_UNATTENDED_RETRY`

### Provider-specific Auth

- `AWS_REGION`
- `AWS_DEFAULT_REGION`
- `AWS_BEARER_TOKEN_BEDROCK`
- `CLAUDE_CODE_SKIP_BEDROCK_AUTH`
- `CLAUDE_CODE_SKIP_VERTEX_AUTH`
- `ANTHROPIC_VERTEX_PROJECT_ID`
- `ANTHROPIC_FOUNDRY_API_KEY`

---

## 24. 最终结论

从源码实现看，Claude Code 的大模型调用体系具备以下特点：

1. **功能非常强**：不是单纯聊天，而是完整的 Agent 运行时。
2. **工程化很深**：重试、fallback、stream watchdog、token budget、prompt caching、tool runtime 都做得很完整。
3. **Anthropic 耦合很深**：当前实现不是通用 LLM abstraction，而是 Anthropic-first architecture。
4. **扩展仍然可行**：如果目标模型能兼容 Anthropic Messages API，则可用较小代价接入。
5. **真正多模型化需要重构**：要支持 OpenAI/Gemini/Qwen 等原生协议，建议抽象独立 Adapter 层。

所以，回答“如何自定义调用其他大模型”，最准确的结论是：

- **短期最优解**：优先走 Anthropic 兼容网关 / 自定义 base URL / modelOverrides。
- **中期最优解**：新增 Provider，复用现有 query loop 与 tool runtime。
- **长期最优解**：抽象统一 LLM Adapter，把 Anthropic 协议从核心运行时中解耦。

---

## 25. 建议的后续工作

如果下一步要继续深入，我建议按以下顺序推进：

1. 先输出一份“自定义 Provider 改造设计方案”。
2. 再输出一份“把 OpenAI/Gemini/Qwen 接进来所需的 Adapter 抽象设计”。
3. 最后再做最小可运行 PoC，实现一个 `custom provider` 分支。

这样风险最低，也最符合当前源码结构。
