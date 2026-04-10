# Claude Code 自定义接入其他大模型 — 完整研究报告

> **基于 Claude Code v2.1.88 源码深度逆向分析**
> 
> 本报告聚焦于"如何灵活地将 Claude Code 接入非 Anthropic 大模型（如 OpenAI GPT-4o、DeepSeek-V3、Google Gemini、Llama 等）"，从协议绑定分析、扩展点识别、网关中转实操、源码级 Provider 注入到完整的排故矩阵，为开发者提供一站式参考。

---

## 目录

- [1. 系统协议绑定性分析](#1-系统协议绑定性分析)
- [2. 五大灵活接入路线](#2-五大灵活接入路线)
- [3. 路线A：API 网关中转法（零代码，最推荐）](#3-路线aapi-网关中转法零代码最推荐)
- [4. 路线B：环境变量 + 配置深度调优](#4-路线b环境变量--配置深度调优)
- [5. 路线C：Settings 覆写 + 模型别名扩展](#5-路线csettings-覆写--模型别名扩展)
- [6. 路线D：源码级新增 Provider 分支](#6-路线d源码级新增-provider-分支)
- [7. 路线E：协议适配层重构（Adapter 架构）](#7-路线e协议适配层重构adapter-架构)
- [8. 协议差异对照：Anthropic vs OpenAI](#8-协议差异对照anthropic-vs-openai)
- [9. 已知兼容网关检测机制](#9-已知兼容网关检测机制)
- [10. 针对目标模型的接入实操指南](#10-针对目标模型的接入实操指南)
- [11. 故障排除矩阵](#11-故障排除矩阵)
- [12. 环境变量完整索引](#12-环境变量完整索引)

---

## 1. 系统协议绑定性分析

Claude Code **不是**一个通用 LLM 客户端，而是一个深度耦合 Anthropic Messages API 协议的 **Agent Runtime**。要理解接入其他模型的难度和策略，必须首先明确其协议绑定点。

### 1.1 六大协议绑定点

| 绑定点 | 涉及源码 | 绑定程度 | 对非 Anthropic 模型影响 |
|--------|---------|---------|----------------------|
| **流式事件格式** | `claude.ts` 流解析状态机 | 🔴 强绑定 | 必须返回 `BetaRawMessageStreamEvent` 格式 |
| **工具调用协议** | `tool_use` / `input_json_delta` | 🔴 强绑定 | 与 OpenAI Function Calling 格式完全不同 |
| **Prompt Caching** | `cache_control: {type:'ephemeral'}` | 🟡 中等绑定 | 可通过 `DISABLE_PROMPT_CACHING=1` 完全禁用 |
| **Extended Thinking** | `thinking: {type:'adaptive'}` | 🟡 中等绑定 | 可通过 `CLAUDE_CODE_DISABLE_THINKING=1` 完全禁用 |
| **Beta Headers** | `betas: [...]` 数组 | 🟢 弱绑定 | 非 1P 端点自动跳过实验性 Beta |
| **结构化输出** | `output_config: {format: 'json_schema'}` | 🟢 弱绑定 | 仅特定场景启用，可被网关忽略 |

### 1.2 流式事件状态机详解

代码在 `src/services/api/claude.ts` 中维护了一套精密的 SSE 事件解析状态机，期望的事件序列为：

```text
message_start  →  content_block_start  →  content_block_delta (×N)  →  content_block_stop
                                                                        ↓
                  content_block_start  →  content_block_delta (×N)  →  content_block_stop
                                                                        ↓
                                                              message_delta  →  message_stop
```

**必须处理的 6 种事件类型**：

| 事件 | 必须字段 | 说明 |
|------|---------|------|
| `message_start` | `message.usage.input_tokens` | 初始化消息和 token 计数 |
| `content_block_start` | `index`, `content_block.type` (`text`/`tool_use`/`thinking`) | 按索引创建内容块 |
| `content_block_delta` | `index`, `delta.type` (`text_delta`/`input_json_delta`/`thinking_delta`) | 增量累加到对应块 |
| `content_block_stop` | `index` | 完成一个内容块，yield AssistantMessage |
| `message_delta` | `delta.stop_reason`, `usage` | 更新停止原因和 token 统计 |
| `message_stop` | (无) | 流结束 |

### 1.3 工具调用协议差异（核心难点）

**Anthropic Messages API 的 tool_use 格式**（Claude Code 期望的）：
```json
{
  "type": "tool_use",
  "id": "toolu_01A09q9...",
  "name": "bash_tool",
  "input": {"command": "ls -la"}
}
```
流式传输时，`input` 以 `input_json_delta` 类型的 `partial_json` 分片到达，需要累加后整体 JSON.parse。

**OpenAI Chat Completions API 的 function_call 格式**：
```json
{
  "tool_calls": [{
    "id": "call_abc123",
    "type": "function",
    "function": {
      "name": "bash_tool",
      "arguments": "{\"command\": \"ls -la\"}"
    }
  }]
}
```

> **关键差异**：Anthropic 将工具调用平铺为 `content[]` 数组的一个元素，OpenAI 将其放在独立的 `tool_calls[]` 字段中。流式增量格式也完全不同。**这就是为什么必须依赖网关做协议转译**。

---

## 2. 五大灵活接入路线

按**改动成本从低到高**排列：

| 路线 | 代码改动 | 灵活性 | 稳定性 | 适用场景 |
|------|---------|--------|--------|---------|
| **A. API 网关中转** | 零代码 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | 快速试用、多模型混用、团队部署 |
| **B. 环境变量调优** | 零代码 | ⭐⭐⭐ | ⭐⭐⭐⭐ | 对接 Anthropic 兼容端点 |
| **C. Settings 覆写** | 改配置文件 | ⭐⭐⭐ | ⭐⭐⭐⭐ | 企业内部固定路由 |
| **D. 新增 Provider** | 中等改码 | ⭐⭐⭐⭐ | ⭐⭐⭐ | 需要深度集成某一厂商 SDK |
| **E. Adapter 重构** | 大量改码 | ⭐⭐⭐⭐⭐ | ⭐⭐ | 完全厂商无关化 |

---

## 3. 路线A：API 网关中转法（零代码，最推荐）

### 3.1 原理

通过在 Claude Code 与目标 LLM 之间部署一个**协议转译网关**，让网关接收 Anthropic Messages API 格式的请求，转换为目标模型的原生格式（如 OpenAI Chat Completions），再将响应反向转译回 Anthropic 格式。

```text
Claude Code ──[Anthropic Protocol]──> 网关(LiteLLM/OneAPI)
                                          │
                                          ├──[OpenAI Protocol]──> GPT-4o
                                          ├──[OpenAI Protocol]──> DeepSeek-V3  
                                          ├──[Gemini Protocol]──> Gemini 2.0
                                          └──[Ollama Protocol]──> Llama 3.3
```

### 3.2 推荐网关方案对比

| 网关 | 支持模型数 | Anthropic 协议兼容性 | 工具调用转译 | 流式支持 | 部署难度 |
|------|----------|-------------------|------------|---------|---------|
| **LiteLLM** | 100+ | ✅ 原生支持 `/anthropic/messages` | ✅ 自动 | ✅ SSE | 低 |
| **OneAPI / New API** | 50+ | ✅ 适配器模式 | ✅ 自动 | ✅ SSE | 低 |
| **OpenRouter** | 200+ | ✅ 云托管 | ✅ 自动 | ✅ SSE | 极低（SaaS） |
| **Portkey** | 50+ | ✅ 企业级 | ✅ 自动 | ✅ SSE | 低 |
| **自建 Nginx + 脚本** | 自定义 | ❌ 需手写 | ❌ 需手写 | ❓ 取决于实现 | 高 |

### 3.3 LiteLLM 实操部署

#### 步骤 1：安装 LiteLLM
```bash
pip install litellm[proxy]
```

#### 步骤 2：创建配置文件 `litellm_config.yaml`
```yaml
model_list:
  # OpenAI GPT-4o
  - model_name: gpt-4o
    litellm_params:
      model: openai/gpt-4o
      api_key: sk-xxx
      
  # DeepSeek V3
  - model_name: deepseek-v3
    litellm_params:
      model: deepseek/deepseek-chat
      api_key: sk-xxx
      api_base: https://api.deepseek.com/v1
      
  # Google Gemini 2.0
  - model_name: gemini-2.0-flash
    litellm_params:
      model: gemini/gemini-2.0-flash
      api_key: AIza-xxx
      
  # 本地 Ollama
  - model_name: llama3.3
    litellm_params:
      model: ollama/llama3.3
      api_base: http://localhost:11434

  # Anthropic 原生（保留直连能力）
  - model_name: claude-sonnet-4-6
    litellm_params:
      model: anthropic/claude-sonnet-4-6
      api_key: sk-ant-xxx

litellm_settings:
  drop_params: true        # 自动丢弃目标模型不支持的参数
  set_verbose: false
```

#### 步骤 3：启动代理服务器
```bash
litellm --config litellm_config.yaml --port 4000
```

#### 步骤 4：配置 Claude Code 环境变量
```powershell
# 核心路由配置
$env:ANTHROPIC_BASE_URL = "http://127.0.0.1:4000/v1"
$env:ANTHROPIC_API_KEY = "sk-litellm-master-key"
$env:ANTHROPIC_MODEL = "gpt-4o"   # 使用 LiteLLM 中定义的 model_name

# ⚠️ 必须禁用的 Anthropic 专有特性（目标模型不支持）
$env:CLAUDE_CODE_DISABLE_THINKING = "1"
$env:DISABLE_PROMPT_CACHING = "1"

# 可选：增加超时时间（某些模型响应较慢）
$env:API_TIMEOUT_MS = "900000"       # 15分钟
```

#### 步骤 5：正常启动 Claude Code
```bash
claude
```

### 3.4 OpenRouter 云托管方案（最简单）

无需部署任何本地服务：
```powershell
$env:ANTHROPIC_BASE_URL = "https://openrouter.ai/api/v1"
$env:ANTHROPIC_API_KEY = "sk-or-v1-xxx"    # OpenRouter API Key
$env:ANTHROPIC_MODEL = "openai/gpt-4o"     # OpenRouter 模型标识
$env:CLAUDE_CODE_DISABLE_THINKING = "1"
$env:DISABLE_PROMPT_CACHING = "1"
```

### 3.5 网关方案的局限性

| 问题 | 原因 | 影响 |
|------|------|------|
| 工具调用质量下滑 | 非 Claude 模型对复杂 JSON Schema tool_use 的遵循度较低 | 可能生成格式错误的工具参数 |
| 上下文窗口不匹配 | Claude Code 按 Claude 的 200k/1M 窗口计算 token 预算 | 小窗口模型可能频繁触发 context overflow |
| 停止原因不对应 | `stop_reason: 'tool_use'` 需要网关正确映射 | 可能导致 Agent 循环异常退出 |
| 嵌套工具并发 | Claude Code 可能同时下发 5+ 个工具调用 | 部分模型不支持多工具并行返回 |

---

## 4. 路线B：环境变量 + 配置深度调优

当你使用**已经兼容 Anthropic 协议的端点**（如 AWS Bedrock、Azure 上的 Claude、或企业私有化部署的 Anthropic 实例）时，仅需环境变量即可完成路由。

### 4.1 完整环境变量操控矩阵

#### 基础路由控制
```powershell
# 端点 & 认证
$env:ANTHROPIC_BASE_URL = "https://your-proxy.company.com/v1"
$env:ANTHROPIC_API_KEY = "your-key"
$env:ANTHROPIC_AUTH_TOKEN = "Bearer xxx"    # 替代 API Key 的 Bearer Token

# 自定义请求头（支持换行符分隔多个）
$env:ANTHROPIC_CUSTOM_HEADERS = "X-Team-Id: engineering`nX-Project: claude-code"

# 模型指定
$env:ANTHROPIC_MODEL = "claude-sonnet-4-6"  # 或自定义模型 ID
```

#### 特性开关精细控制
```powershell
# 禁用可能不兼容的特性
$env:CLAUDE_CODE_DISABLE_THINKING = "1"          # 关闭扩展思维
$env:DISABLE_PROMPT_CACHING = "1"                # 关闭提示缓存标记
$env:CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS = "1" # 关闭所有实验性 Beta

# 超时调整
$env:API_TIMEOUT_MS = "900000"                   # API 请求超时 (默认 600000ms)

# 注入自定义请求参数
$env:CLAUDE_CODE_EXTRA_BODY = '{"temperature": 0.7, "custom_routing_key": "team-a"}'
```

### 4.2 `CLAUDE_CODE_EXTRA_BODY` 深度用法

这是一个极其强大的扩展点。它允许你向 API 请求体中注入任意 JSON 字段，这些字段会在最终请求组装时**浅合并**到请求参数中。

**源码位置**：`src/services/api/claude.ts` L266-295

**合并位置**：在构建最终 API 请求后、发送前，`extraBodyParams` 被展开合并：
```typescript
return {
  model, messages, system, tools, tool_choice,
  betas, metadata, max_tokens, thinking, temperature,
  ...extraBodyParams,    // ← 你的自定义参数在此注入
  output_config, speed
}
```

**实用场景**：
```powershell
# 场景1：为网关传递路由标签
$env:CLAUDE_CODE_EXTRA_BODY = '{"x-route-to": "us-east-cluster"}'

# 场景2：覆盖采样参数
$env:CLAUDE_CODE_EXTRA_BODY = '{"temperature": 0.3, "top_p": 0.95}'

# 场景3：注入 Bedrock 特定参数
$env:CLAUDE_CODE_EXTRA_BODY = '{"anthropic_beta": ["extended-thinking-2025-04-01"]}'

# 场景4：传递网关级 metadata
$env:CLAUDE_CODE_EXTRA_BODY = '{"helicone_properties": {"session_id": "abc"}}'
```

> ⚠️ **安全提示**：`CLAUDE_CODE_EXTRA_BODY` 的内容会经过 `safeParseJSON` 校验，必须是合法的 JSON 对象（不能是数组或标量值）。

---

## 5. 路线C：Settings 覆写 + 模型别名扩展

Claude Code 的 settings 配置系统(`~/.claude/settings.json` 或项目级 `.claude/settings.json`) 提供了丰富的模型覆写能力。

### 5.1 模型覆写（modelOverrides）

允许将内部的 Canonical Model ID 映射到任意自定义字符串：

**源码位置**：`src/utils/model/modelStrings.ts` L63

```json
// ~/.claude/settings.json
{
  "modelOverrides": {
    "claude-sonnet-4-6": "my-company-sonnet-proxy",
    "claude-opus-4-6": "arn:aws:bedrock:us-west-2:123456:inference-profile/opus-custom",
    "claude-haiku-4-5": "deepseek-v3-via-proxy"
  }
}
```

**原理**：当代码内部请求 `claude-sonnet-4-6` 时，经过 `applyModelOverrides()` 函数，实际发送的 `model` 参数会被替换为你指定的字符串。

### 5.2 可用模型白名单（availableModels）

```json
{
  "availableModels": ["opus", "sonnet", "gpt-4o", "deepseek-v3"]
}
```

验证逻辑 (`src/utils/model/modelAllowlist.ts`) 支持：
- **完全匹配**：`"gpt-4o"` 精确匹配
- **前缀/包含匹配**：`"opus"` 匹配所有含 `opus` 的模型 ID
- **家族别名**：`"sonnet"` 匹配所有 Sonnet 变体

### 5.3 模型选择优先级全链路

```text
/model 命令 (会话级)
    ↓ (未设置时)
--model 启动参数
    ↓ (未设置时)
$env:ANTHROPIC_MODEL 环境变量
    ↓ (未设置时)
settings.json → model 字段
    ↓ (未设置时)
硬编码默认值 (claude-sonnet-4-6)
```

每一层的值都会经过 `parseUserSpecifiedModel()` 处理，支持别名解析（`sonnet` → `claude-sonnet-4-6`）和长上下文后缀（`opus[1m]`）。

### 5.4 模型验证机制

当你使用自定义模型 ID 时，系统会进行以下验证 (`src/utils/model/validateModel.ts`)：

1. **白名单检查**：如果设了 `availableModels`，先检查是否在白名单内
2. **别名检查**：预定义别名（`sonnet`, `opus`, `haiku`, `best`）始终有效
3. **缓存检查**：之前验证通过的模型 ID 会被缓存
4. **实时 API 验证**：发起一个最小化的 API 调用（`max_tokens: 1`）测试模型是否可达

> **关键发现**：系统**允许传递任意模型 ID**，只要目标端点（`ANTHROPIC_BASE_URL`）能够识别该 ID 即可。这意味着通过网关中转时，你可以使用 `gpt-4o`、`deepseek-v3` 等非 Anthropic 的模型名称。

---

## 6. 路线D：源码级新增 Provider 分支

如果你想像 Bedrock/Vertex/Foundry 那样，为某个特定厂商（如 OpenAI）添加一等公民级的支持，需要修改以下文件。

### 6.1 以 Foundry 为模板的改造清单

**Foundry 是最晚加入的 Provider**，其集成模式最适合作为模板。

#### 第 1 步：扩展 Provider 枚举

**文件**：`src/utils/model/providers.ts`
```typescript
// 修改前
export type APIProvider = 'firstParty' | 'bedrock' | 'vertex' | 'foundry'

// 修改后
export type APIProvider = 'firstParty' | 'bedrock' | 'vertex' | 'foundry' | 'openai'

export function getAPIProvider(): APIProvider {
  return isEnvTruthy(process.env.CLAUDE_CODE_USE_BEDROCK)
    ? 'bedrock'
    : isEnvTruthy(process.env.CLAUDE_CODE_USE_VERTEX)
      ? 'vertex'
      : isEnvTruthy(process.env.CLAUDE_CODE_USE_FOUNDRY)
        ? 'foundry'
        : isEnvTruthy(process.env.CLAUDE_CODE_USE_OPENAI)  // 新增
          ? 'openai'
          : 'firstParty'
}
```

#### 第 2 步：创建 SDK 适配客户端

**文件**：`src/services/api/client.ts`

需要创建一个**伪装层**，将 OpenAI SDK 的调用接口包装成与 `@anthropic-ai/sdk` 相同的签名：

```typescript
// 在 getAnthropicClient() 中新增 openai 分支
if (provider === 'openai') {
  return createOpenAICompatClient({
    apiKey: process.env.OPENAI_API_KEY,
    baseURL: process.env.OPENAI_BASE_URL || 'https://api.openai.com/v1',
    timeout: parseInt(process.env.API_TIMEOUT_MS || '600000', 10),
  })
}
```

> ⚠️ **核心难点**：`createOpenAICompatClient` 必须返回一个符合 `Anthropic` SDK 接口的对象，其内部需要做请求格式转换（Anthropic Messages → OpenAI Chat Completions）和响应格式转换（OpenAI SSE → Anthropic BetaRawMessageStreamEvent）。这相当于在代码内嵌了一个精简版 LiteLLM。

#### 第 3 步：注册模型配置

**文件**：`src/utils/model/configs.ts`
```typescript
export const GPT4O_CONFIG = {
  firstParty: 'gpt-4o',     // 不会被使用
  bedrock: 'gpt-4o',        // 不会被使用
  vertex: 'gpt-4o',         // 不会被使用
  foundry: 'gpt-4o',        // 不会被使用
  openai: 'gpt-4o',         // 实际使用
} as const satisfies ModelConfig
```

#### 第 4 步：更新 Beta 支持检测

**文件**：`src/utils/betas.ts`
```typescript
export function shouldIncludeFirstPartyOnlyBetas(): boolean {
  const provider = getAPIProvider()
  // OpenAI 不支持任何 Anthropic Beta
  return (provider === 'firstParty' || provider === 'foundry') &&
         !isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS)
}
```

### 6.2 改动评估

| 改动项 | 复杂度 | 文件数 |
|--------|--------|--------|
| Provider 枚举扩展 | 低 | 1 |
| 客户端适配层（含协议转换） | 🔴 极高 | 1-2（但代码量大） |
| 模型配置注册 | 低 | 1 |
| Beta/能力检测更新 | 中 | 2-3 |
| 错误码映射 | 中 | 1 |
| **总计** | **高** | **5-8 文件** |

> **建议**：除非你有非常明确的需求（如企业安全合规要求不能使用外部网关），否则路线 A 的网关方案远比手动实现 Provider 更加高效。

---

## 7. 路线E：协议适配层重构（Adapter 架构）

这是最彻底的方案——从根本上解耦代码中所有 Anthropic 专有协议依赖。

### 7.1 目标架构

```typescript
// 定义通用 LLM 适配器接口
interface LLMAdapter {
  // 请求转换：内部统一格式 → 厂商原生格式
  buildRequest(params: UnifiedRequestParams): unknown
  
  // 工具定义转换：统一 Schema → 厂商格式
  buildToolDefinitions(tools: UnifiedTool[]): unknown
  
  // 流式响应转换：厂商 SSE → 统一 AssistantMessage
  parseStream(stream: AsyncIterator<unknown>): AsyncGenerator<AssistantMessage>
  
  // 错误映射：厂商错误 → 统一错误类型
  classifyError(error: unknown): UnifiedAPIError
  
  // 能力声明
  capabilities: {
    supportsThinking: boolean
    supportsPromptCaching: boolean
    supportsStructuredOutput: boolean
    supportsParallelToolCalls: boolean
    maxContextTokens: number
  }
}

// 厂商实现
class AnthropicAdapter implements LLMAdapter { ... }
class OpenAIAdapter implements LLMAdapter { ... }
class GeminiAdapter implements LLMAdapter { ... }
```

### 7.2 需要改动的核心文件

| 文件 | 当前职责 | 改造内容 |
|------|---------|---------|
| `src/services/api/claude.ts` | API 请求组装 + 流解析 | 拆为适配器调度层 + 各厂商实现 |
| `src/services/api/client.ts` | SDK 实例化 | 改为适配器工厂 |
| `src/services/api/errors.ts` | 错误分类 | 扩展错误映射表 |
| `src/query.ts` | 主循环 | 使用统一 `AssistantMessage` 接口 |
| `src/utils/api.ts` | 工具 Schema 构建 | 拆分为通用定义 + 厂商转换 |
| `src/utils/thinking.ts` | 思考配置 | 按适配器能力有条件启用 |
| `src/utils/betas.ts` | Beta 头管理 | 按适配器能力动态生成 |

### 7.3 成本评估

- **预计改动行数**：2000-4000 行
- **涉及文件**：15-25 个
- **测试覆盖**：每个适配器需独立集成测试
- **风险**：可能破坏现有 Anthropic 调用的稳定性

---

## 8. 协议差异对照：Anthropic vs OpenAI

这是网关转译或 Adapter 开发时最核心的参考表：

### 8.1 请求格式对照

| 特性 | Anthropic Messages API | OpenAI Chat Completions API |
|------|----------------------|---------------------------|
| **端点** | `POST /v1/messages` | `POST /v1/chat/completions` |
| **System Prompt** | 独立 `system` 字段（支持数组） | `messages[0].role = "system"` |
| **消息角色** | `user`, `assistant` | `system`, `user`, `assistant`, `tool` |
| **内容块** | `content: [{type:'text', text:'...'}, {type:'image', ...}]` | `content: "string"` 或 `content: [{type:'text',...}]` |
| **工具定义** | `tools: [{type:'tool', name, input_schema}]` | `tools: [{type:'function', function:{name, parameters}}]` |
| **工具结果** | `role:'user'`, `content:[{type:'tool_result', tool_use_id, content}]` | `role:'tool'`, `tool_call_id`, `content` |
| **max tokens** | `max_tokens` (必填) | `max_tokens` (可选) |
| **温度** | `temperature` (仅 thinking 关闭时) | `temperature` (始终可用) |
| **流式** | `stream: true` → SSE with typed events | `stream: true` → SSE with `data: {...}` |

### 8.2 流式响应事件对照

| Anthropic SSE 事件 | 对应 OpenAI SSE 块 |
|--------------------|------------------|
| `event: message_start` | `data: {"choices":[{"delta":{"role":"assistant"}}]}` |
| `event: content_block_start` (text) | 首个 `data: {"choices":[{"delta":{"content":"..."}}]}` |
| `event: content_block_delta` (text_delta) | `data: {"choices":[{"delta":{"content":"chunk"}}]}` |
| `event: content_block_start` (tool_use) | `data: {"choices":[{"delta":{"tool_calls":[{"index":0,"id":"...","function":{"name":"..."}}]}}]}` |
| `event: content_block_delta` (input_json_delta) | `data: {"choices":[{"delta":{"tool_calls":[{"index":0,"function":{"arguments":"chunk"}}]}}]}` |
| `event: message_delta` (stop_reason) | `data: {"choices":[{"finish_reason":"stop"}]}` |
| `event: message_stop` | `data: [DONE]` |

### 8.3 停止原因映射

| Anthropic `stop_reason` | OpenAI `finish_reason` |
|------------------------|----------------------|
| `end_turn` | `stop` |
| `tool_use` | `tool_calls` |
| `max_tokens` | `length` |
| `refusal` | `content_filter` |

---

## 9. 已知兼容网关检测机制

Claude Code 内置了一套网关指纹识别系统（`src/services/api/logging.ts`），通过**响应头特征**和**主机名后缀**自动识别已知网关，用于日志标记和错误处理优化。

### 9.1 已识别网关列表

| 网关名称 | 识别方式 | 响应头前缀/主机名 |
|---------|---------|----------------|
| **LiteLLM** | 响应头 | `x-litellm-*` |
| **Helicone** | 响应头 | `helicone-*` |
| **Portkey** | 响应头 | `x-portkey-*` |
| **Cloudflare AI Gateway** | 响应头 | `cf-aig-*` |
| **Kong** | 响应头 | `x-kong-*` |
| **Braintrust** | 响应头 | `x-bt-*` |
| **Databricks** | 主机名 | `.cloud.databricks.com`, `.azuredatabricks.net`, `.gcp.databricks.com` |

### 9.2 非官方端点的 Beta 行为

当检测到 `ANTHROPIC_BASE_URL` 不是 `api.anthropic.com` 时：

```typescript
// src/utils/model/providers.ts
export function isFirstPartyAnthropicBaseUrl(): boolean {
  const baseUrl = process.env.ANTHROPIC_BASE_URL
  if (!baseUrl) return true  // 未设置 = 官方
  const host = new URL(baseUrl).host
  return ['api.anthropic.com'].includes(host)
}
```

**影响**：
- 实验性 Beta Headers（如 `interleaved-thinking`、`structured-outputs`）**不会发送**给非 1P 端点
- 仅通用 Beta（如 `context-1m`）会保留
- 这实际上**有助于兼容性**——非 Anthropic 端点不会收到不理解的 Beta 标记

---

## 10. 针对目标模型的接入实操指南

### 10.1 OpenAI GPT-4o / GPT-4.5

```powershell
# 方案1：通过 LiteLLM
$env:ANTHROPIC_BASE_URL = "http://127.0.0.1:4000/v1"
$env:ANTHROPIC_API_KEY = "sk-litellm-key"
$env:ANTHROPIC_MODEL = "gpt-4o"
$env:CLAUDE_CODE_DISABLE_THINKING = "1"
$env:DISABLE_PROMPT_CACHING = "1"
$env:API_TIMEOUT_MS = "600000"
```

**注意事项**：
- GPT-4o 的工具调用质量很高，网关转译后基本稳定
- 上下文窗口 128k，低于 Claude 的 200k，可能触发 context overflow 更频繁
- 不支持 `tool_use` 的 `eager_input_streaming`，网关应丢弃该字段

### 10.2 DeepSeek-V3 / DeepSeek-R1

```powershell
$env:ANTHROPIC_BASE_URL = "http://127.0.0.1:4000/v1"
$env:ANTHROPIC_API_KEY = "sk-litellm-key"
$env:ANTHROPIC_MODEL = "deepseek-v3"
$env:CLAUDE_CODE_DISABLE_THINKING = "1"
$env:DISABLE_PROMPT_CACHING = "1"
$env:API_TIMEOUT_MS = "1200000"         # DeepSeek 较慢，建议20分钟
$env:CLAUDE_CODE_EXTRA_BODY = '{"temperature": 0.5}'  # DeepSeek 建议较低温度
```

**注意事项**：
- DeepSeek-V3 对大量工具定义（Claude Code 通常注入 20+ 个工具）的遵循度不如 Claude
- JSON 参数生成可能包含尾随逗号
- 建议在 LiteLLM 配置中启用 `drop_params: true`

### 10.3 Google Gemini 2.0 Flash / Pro

```powershell
$env:ANTHROPIC_BASE_URL = "http://127.0.0.1:4000/v1"
$env:ANTHROPIC_API_KEY = "sk-litellm-key"
$env:ANTHROPIC_MODEL = "gemini-2.0-flash"
$env:CLAUDE_CODE_DISABLE_THINKING = "1"
$env:DISABLE_PROMPT_CACHING = "1"
```

**注意事项**：
- Gemini 有自己的工具调用格式（`functionCall`），LiteLLM 的转译基本可靠
- Gemini 2.0 Flash 的 1M 上下文窗口非常充裕
- 安全过滤器可能误触发导致 `refusal` 类型异常

### 10.4 本地模型（Ollama + Llama/Qwen）

```powershell
# 先启动 Ollama
# ollama serve

# LiteLLM 配置中加入 Ollama 后端
$env:ANTHROPIC_BASE_URL = "http://127.0.0.1:4000/v1"
$env:ANTHROPIC_API_KEY = "sk-litellm-key"
$env:ANTHROPIC_MODEL = "llama3.3"
$env:CLAUDE_CODE_DISABLE_THINKING = "1"
$env:DISABLE_PROMPT_CACHING = "1"
$env:API_TIMEOUT_MS = "1800000"         # 本地推理可能很慢
```

**注意事项**：
- 本地模型的工具调用能力通常较弱，建议使用 70B+ 参数规模
- Claude Code 的系统提示非常长（~8000 tokens），小模型可能被"淹没"
- 上下文窗口通常只有 8k-32k，会频繁触发 Context Collapse

---

## 11. 故障排除矩阵

| 症状 | 可能原因 | 解决方案 |
|------|---------|---------|
| `400 Bad Request` | 网关未正确转译 `cache_control` 或 `thinking` 参数 | 设置 `DISABLE_PROMPT_CACHING=1` 和 `CLAUDE_CODE_DISABLE_THINKING=1` |
| `404 Not Found` | 模型 ID 在目标端点不存在 | 检查 `ANTHROPIC_MODEL` 是否匹配网关中定义的 model_name |
| 流卡死无响应 | 网关未正确传递 SSE 流，或目标模型不返回流 | 增加 `API_TIMEOUT_MS`；检查网关日志；确认目标模型支持流式 |
| `Unexpected end of stream` | 目标模型响应提前终止 | 降低 `max_tokens`；检查目标模型的 token 限制 |
| 工具调用后陷入死循环 | 模型返回的 `tool_use` 参数格式不正确，导致工具执行失败后反复重试 | 换用工具调用能力更强的模型；或减少暴露给模型的工具数量 |
| `Model not allowed` | `settings.json` 中配置了 `availableModels` 白名单 | 在 `availableModels` 中添加你的自定义模型 ID |
| `529 Overloaded` 大量重试 | 网关或目标模型过载 | Claude Code 已内置 529 重试机制（最多 3 次 + fallback）；检查网关负载 |
| `context_too_long` 报错 | 对话超出目标模型的上下文窗口 | Claude Code 会自动尝试 context collapse；也可开新会话 |
| 非流式 fallback 后工具重复执行 | 流式调用失败后 fallback 到非流式，但之前的工具已部分执行 | 设置 `CLAUDE_CODE_DISABLE_NONSTREAMING_FALLBACK=1` 禁用 fallback |
| `temperature` 相关错误 | 某些模型不接受 `temperature: 1` | 通过 `CLAUDE_CODE_EXTRA_BODY='{"temperature":0.7}'` 覆盖 |

---

## 12. 环境变量完整索引

### 核心路由

| 变量 | 默认值 | 说明 |
|------|-------|------|
| `ANTHROPIC_BASE_URL` | `https://api.anthropic.com` | API 端点 |
| `ANTHROPIC_API_KEY` | (无) | API 密钥 |
| `ANTHROPIC_AUTH_TOKEN` | (无) | Bearer Token（替代 API Key） |
| `ANTHROPIC_MODEL` | (无) | 强制覆盖模型 ID |
| `ANTHROPIC_CUSTOM_HEADERS` | (无) | 自定义请求头（`\n`分隔） |
| `CLAUDE_CODE_EXTRA_BODY` | (无) | 注入任意 JSON 到请求体 |

### Provider 选择

| 变量 | 默认值 | 说明 |
|------|-------|------|
| `CLAUDE_CODE_USE_BEDROCK` | (无) | 启用 AWS Bedrock |
| `CLAUDE_CODE_USE_VERTEX` | (无) | 启用 Google Vertex AI |
| `CLAUDE_CODE_USE_FOUNDRY` | (无) | 启用 Azure Foundry |

### 特性开关

| 变量 | 默认值 | 说明 |
|------|-------|------|
| `CLAUDE_CODE_DISABLE_THINKING` | (无) | 禁用 Extended Thinking |
| `DISABLE_PROMPT_CACHING` | (无) | 禁用 Prompt Caching |
| `CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS` | (无) | 禁用全部实验性 Beta |
| `CLAUDE_CODE_DISABLE_NONSTREAMING_FALLBACK` | (无) | 禁用非流式降级 |

### 超时与重试

| 变量 | 默认值 | 说明 |
|------|-------|------|
| `API_TIMEOUT_MS` | `600000` (10min) | API 请求超时 |
| `CLAUDE_STREAM_IDLE_TIMEOUT_MS` | `90000` (90s) | 流闲置超时 |
| `CLAUDE_CODE_UNATTENDED_RETRY` | (无) | 无人值守无限重试模式 |

### AWS Bedrock 专用

| 变量 | 说明 |
|------|------|
| `AWS_REGION` / `AWS_DEFAULT_REGION` | AWS 区域 |
| `AWS_BEARER_TOKEN_BEDROCK` | Bedrock API Key |
| `ANTHROPIC_SMALL_FAST_MODEL_AWS_REGION` | Haiku 模型区域 |
| `CLAUDE_CODE_SKIP_BEDROCK_AUTH` | 跳过认证（测试用） |

### Google Vertex 专用

| 变量 | 说明 |
|------|------|
| `ANTHROPIC_VERTEX_PROJECT_ID` | GCP 项目 ID |
| `CLOUD_ML_REGION` | GCP 区域 |
| `VERTEX_REGION_CLAUDE_*` | 模型特定区域 |
| `CLAUDE_CODE_SKIP_VERTEX_AUTH` | 跳过认证（测试用） |

### Azure Foundry 专用

| 变量 | 说明 |
|------|------|
| `ANTHROPIC_FOUNDRY_RESOURCE` | Azure 资源名称 |
| `ANTHROPIC_FOUNDRY_BASE_URL` | Foundry 端点 URL |
| `ANTHROPIC_FOUNDRY_API_KEY` | Azure API 密钥 |
| `CLAUDE_CODE_SKIP_FOUNDRY_AUTH` | 跳过认证（测试用） |

---

## 总结：推荐决策树

```text
需要接入其他大模型？
│
├── 只是想试试看 / 快速验证
│   └── 路线A：LiteLLM 本地网关 或 OpenRouter 云服务
│       └── 5分钟部署，零代码改动
│
├── 企业环境，需要稳定运行
│   ├── 目标是 Anthropic 系列（不同部署渠道）
│   │   └── 路线B：环境变量直配 Bedrock/Vertex/Foundry
│   │
│   └── 目标是非 Anthropic 模型
│       └── 路线A：LiteLLM/Portkey 企业版 + 路线C Settings 覆写
│
├── 需要深度集成某单一厂商
│   └── 路线D：源码级新增 Provider（参考 Foundry 模板）
│
└── 要做一个通用的多模型 Agent 框架
    └── 路线E：Adapter 架构重构（注意：工作量极大）
```

> **核心结论**：对于绝大多数场景，**路线 A（API 网关中转）+ 路线 B（环境变量调优）的组合**是最佳实践。通过 LiteLLM 等成熟网关处理协议转译，通过环境变量禁用不兼容特性，即可在零代码改动的前提下将 Claude Code 连接到几乎任何大模型。