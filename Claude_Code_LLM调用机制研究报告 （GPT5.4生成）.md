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
