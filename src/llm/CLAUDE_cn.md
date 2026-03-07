# LLM 模块

多提供商 LLM 集成，配备断路器、重试、故障转移和响应缓存。

## 文件映射

| 文件 | 角色 |
|------|------|
| `mod.rs` | 提供商工厂（`create_llm_provider`、`build_provider_chain`）；`LlmBackend` 枚举 |
| `provider.rs` | `LlmProvider` trait、`ChatMessage`、`ToolCall`、`CompletionRequest`、`sanitize_tool_messages` |
| `nearai_chat.rs` | NEAR AI Chat Completions 提供商（双重认证：会话令牌或 API 密钥） |
| `reasoning.rs` | `Reasoning` 结构、`ReasoningContext`、`RespondResult`、`ActionPlan`、`ToolSelection`；思维标签剥离；`SILENT_REPLY_TOKEN` |
| `session.rs` | 带有磁盘 + DB 持久化的 NEAR AI 会话令牌管理，OAuth 登录流程 |
| `circuit_breaker.rs` | 断路器：Closed → Open → HalfOpen 状态机 |
| `retry.rs` | 指数退避重试包装器；`is_retryable()` 分类 |
| `failover.rs` | `FailoverProvider` — 按顺序尝试提供商，每个提供商有冷却期 |
| `response_cache.rs` | 带有 TTL 和 LRU 驱逐的内存中 LLM 响应缓存（通过 SHA-256 键控） |
| `costs.rs` | 每模型成本静态表（OpenAI、Anthropic、本地/Ollama 启发式） |
| `rig_adapter.rs` | 适配器桥接 rig-core `CompletionModel` → `LlmProvider`；由 OpenAI、Anthropic、Ollama、Tinfoil 使用 |
| `smart_routing.rs` | `SmartRoutingProvider` — 13 维复杂度评分器路由廉价 vs 主模型 |
| `recording.rs` | `RecordingLlm` — 用于 E2E 重放测试的跟踪捕获（`IRONCLAW_RECORD_TRACE`） |

## 提供商选择

通过 `LLM_BACKEND` 环境变量设置：

| 值 | 提供商 | 关键环境变量 |
|-------|----------|-------------|
| `nearai`（默认） | NEAR AI Chat Completions | `NEARAI_SESSION_TOKEN` 或 `NEARAI_API_KEY` |
| `openai` | OpenAI | `OPENAI_API_KEY` |
| `anthropic` | Anthropic | `ANTHROPIC_API_KEY` |
| `ollama` | Ollama 本地 | `OLLAMA_BASE_URL` |
| `openai_compatible` | 任何 OpenAI 兼容端点 | `LLM_BASE_URL`、`LLM_API_KEY`、`LLM_MODEL` |
| `tinfoil` | Tinfoil TEE 推理 | `TINFOIL_API_KEY`、`TINFOIL_MODEL` |

## NEAR AI 提供商陷阱

**双重认证模式：**
- **会话令牌**（默认）：`NEARAI_SESSION_TOKEN=sess_...`，基本 URL = `https://private.near.ai`。令牌持久化到 `~/.ironclaw/session.json`（模式 0600）并可选地到数据库 `settings` 表（`nearai.session_token`）。在 401 响应中，当正文包含 "session" + "expired"/"invalid" 时，`NearAiChatProvider` 调用 `session.handle_auth_failure()`，这会触发交互式 OAuth 登录流程并重试一次。纯 `AuthFailed` 401 不会重试。
- **API 密钥**：设置 `NEARAI_API_KEY`（来自 `cloud.near.ai`），基本 URL 默认为 `https://cloud-api.near.ai`。使用 API 密钥认证的 401 立即返回为 `LlmError::AuthFailed` — 无续期。

**会话续期是交互式的：** 当 `SessionExpired` 触发续期时，它会阻塞并在终端中提示用户（GitHub/Google OAuth 或手动 API 密钥输入）。这不适合无头/托管部署 — 改为设置 `NEARAI_SESSION_TOKEN` 环境变量。

**工具消息扁平化：** NEAR AI 的 API 不支持标准格式的 `role: "tool"` 消息。`nearai_chat.rs` 默认 `flatten_tool_messages = true`，将工具结果转换为带有 `[Tool result from <name>]: <content>` 格式的用户消息。对于兼容端点，使用 `NearAiChatProvider::new_with_flatten(..., false)` 禁用。

**定价自动获取：** 启动时，`NearAiChatProvider` 触发后台任务以从 `/v1/model/list` 获取每模型定价。如果获取失败，它会静默回退到 `costs::model_cost()` / `costs::default_cost()`。定价仅存储在内存中。

**HTTP 请求超时：** NEAR AI HTTP 客户端每个请求有 120 秒超时。速率限制 `Retry-After` 标头被解析（delay-seconds 和 HTTP-date 格式）并作为 `LlmError::RateLimited { retry_after }` 转发，以便 `RetryProvider` 遵守。

## 断路器

`circuit_breaker.rs` 中的状态机：
```
Closed（正常）
  → Open（在 failure_threshold 连续瞬态故障后；默认：5）
    → HalfOpen（在 recovery_timeout 后；默认：30s）
      → Closed（在 half_open_successes_needed 探测成功后；默认：2）
      → Open（如果任何探测失败）
```

**瞬态与非瞬态错误：** 只有 `RequestFailed`、`RateLimited`、`InvalidResponse`、`SessionExpired`、`SessionRenewalFailed`、`Http` 和 `Io` 计入阈值。`AuthFailed`、`ContextLengthExceeded`、`ModelNotAvailable` 和 `Json` 错误永远不会触发断路器 — 它们表示调用者问题，而不是后端降级。

通过 `NearAiConfig` 字段配置：`circuit_breaker_threshold`（None = 禁用）、`circuit_breaker_recovery_secs`（默认：30）。

断路器包装整个提供商链。当打开时，它立即返回 `LlmError::RequestFailed` 并包含剩余冷却秒数的消息。坐在外部的 `FailoverProvider` 然后可以尝试备用模型。

## 故障转移链

`failover.rs` 中的 `FailoverProvider` 包装 `LlmProvider` 实例列表。在可重试错误时，它尝试列表中的下一个提供商。反复失败的提供商进入冷却期并被跳过（除非所有提供商都在冷却中，在这种情况下尝试最近最少冷却的提供商）。

**冷却默认值：** `failure_threshold = 3` 连续可重试故障 → 冷却 `cooldown_duration = 300s`。通过 `NearAiConfig` 字段配置：`failover_cooldown_secs`、`failover_cooldown_threshold`。

**当前连接：** 故障转移设置在主模型和 `NEARAI_FALLBACK_MODEL`（同一 NEAR AI 后端上的不同模型名称）之间，而不是跨不同的 LLM 提供商类型。跨提供商故障转移（例如，NEAR AI → Anthropic）需要手动构造。

## 重试

`retry.rs` 中的 `RetryProvider` 使用指数退避包装任何 `LlmProvider`。重试：`RequestFailed`、`RateLimited`、`InvalidResponse`、`SessionRenewalFailed`、`Http`、`Io`。**不**重试：`AuthFailed`、`SessionExpired`、`ContextLengthExceeded`、`ModelNotAvailable`、`Json`。

**退避计划：** 基数 1s，每次尝试加倍，±25% 抖动，最小下限 100ms。尝试 0：~1s，尝试 1：~2s，尝试 2：~4s。对于 `RateLimited`，使用错误中的 `retry_after` 持续时间（提供商提供）而不是退避。

通过 `NearAiConfig.max_retries` 配置（环境变量：`NEARAI_MAX_RETRIES`；默认：3）。设置为 0 禁用。

## LlmProvider Trait

完整的 trait（必须实现所有方法或依赖默认值）：

```rust
#[async_trait]
pub trait LlmProvider: Send + Sync {
    // 必需
    fn model_name(&self) -> &str;
    fn cost_per_token(&self) -> (Decimal, Decimal);  // (input, output) 每令牌
    async fn complete(&self, request: CompletionRequest) -> Result<CompletionResponse, LlmError>;
    async fn complete_with_tools(&self, request: ToolCompletionRequest) -> Result<ToolCompletionResponse, LlmError>;

    // 可选（有默认值）
    async fn list_models(&self) -> Result<Vec<String>, LlmError> { Ok(vec![]) }
    async fn model_metadata(&self) -> Result<ModelMetadata, LlmError> { /* 仅名称 */ }
    fn effective_model_name(&self, requested_model: Option<&str>) -> String { /* 使用 active */ }
    fn active_model_name(&self) -> String { self.model_name().to_string() }
    fn set_model(&self, _model: &str) -> Result<(), LlmError> { /* Err: 不支持 */ }
    fn calculate_cost(&self, input_tokens: u32, output_tokens: u32) -> Decimal { /* 使用 cost_per_token */ }
}
```

关键说明：
- `model_name()` 返回配置的模型名称；`active_model_name()` 返回当前活动的模型（如果调用了 `set_model()` 则可能不同 — 只有 `NearAiChatProvider` 支持此功能）
- `cost_per_token()` 使用 `rust_decimal` 返回 `(Decimal, Decimal)`。在构造函数中通过 `costs::model_cost()` 查找；对于未知对象回退到 `costs::default_cost()`
- `RigAdapter` 忽略每请求模型覆盖（记录警告）。只有 `NearAiChatProvider` 通过 `CompletionRequest::model` 支持每请求模型覆盖
- `complete_with_tools()` 永远不会被缓存（工具调用可能有副作用）— `CachedProvider` 始终传递它们

要添加新提供商：
1. 创建实现 `LlmProvider` 的 `src/llm/myprovider.rs`
2. 在 `mod.rs` 中添加变体到 `LlmBackend`
3. 在 `mod.rs` 的工厂匹配中连接
4. 在 `config/llm.rs` 和 `.env.example` 中添加环境变量

## 响应缓存

`response_cache.rs` 中的 `CachedProvider` 缓存 `complete()` 响应。`complete_with_tools()` 永远不会被缓存（副作用）。缓存键是 `(model_name, messages_json, max_tokens, temperature, stop_sequences)` 的 SHA-256。当达到 `max_entries` 时 LRU 驱逐；访问时基于 TTL 到期。

**默认值：** TTL = 1 小时，最多条目 = 1000。通过 `NearAiConfig` 字段配置：`response_cache_enabled`（环境变量：`NEARAI_RESPONSE_CACHE_ENABLED`）、`response_cache_ttl_secs`、`response_cache_max_entries`。缓存仅在内存中 — 重启时驱逐。

## OpenAI 兼容自定义标头

设置 `LLM_EXTRA_HEADERS=Key:Value,Key2:Value2` 将标头注入到每个请求中。对于 OpenRouter 归属（`HTTP-Referer`、`X-Title`）很有用。无效的标头名称/值被跳过并带有警告（不是致命错误）。

## 提供商链构造

`mod.rs` 中的 `build_provider_chain()` 是组装装饰器的单一真实来源。链是：

```
Raw provider
  → RetryProvider           (每提供商退避；包装主和备用)
  → SmartRoutingProvider    (当设置 NEARAI_CHEAP_MODEL 时廉价/主拆分)
  → FailoverProvider        (备用模型；仅当设置 NEARAI_FALLBACK_MODEL 时)
  → CircuitBreakerProvider  (快速失败；仅当设置 NEARAI_CIRCUIT_BREAKER_THRESHOLD 时)
  → CachedProvider          (响应缓存；仅当 NEARAI_RESPONSE_CACHE_ENABLED=true 时)
  → RecordingLlm            (跟踪捕获；仅当设置 IRONCLAW_RECORD_TRACE 时)
```

`build_provider_chain()` 还返回一个单独的独立廉价 LLM 提供商（用于心跳/评估任务 — 不属于装饰器链）。

## reasoning.rs 内容

`reasoning.rs` **不**包含 `IntentClassifier`。它包含：
- `Reasoning` 结构 — 代理工作器使用的主要推理引擎；调用 `complete_with_tools()` 并处理工具分派
- `ReasoningContext` — 将消息、可用工具、任务描述和元数据携带到推理调用中
- `RespondResult`、`ActionPlan`、`ToolSelection` — 推理引擎的输出类型
- `TokenUsage` — 输入/输出令牌计数
- `SILENT_REPLY_TOKEN`（`"NO_REPLY"`）和 `is_silent_reply()` — 由调度器用于在群聊中抑制空响应
- 思维标签剥离 — 基于正则的表达式从模型响应中删除 `<thinking>`、`<reflection>`、`<scratchpad>`、`<|think|>`、`<final>` 等，然后返回给用户

## costs.rs 详情

`costs.rs` 提供静态查找表（`model_cost(model_id)`），返回每令牌的 `(input_cost, output_cost)` 作为 `rust_decimal::Decimal`。提供商前缀如 `"openai/gpt-4o"` 在查找之前被剥离。对于未知模型返回 `None` — 调用者应回退到 `default_cost()`（大致 GPT-4o 定价）。本地模型启发式（`is_local_model()`）为 Ollama 风格标识符（llama*、mistral*、`:latest`、`:instruct` 等）返回零成本。

## rig_adapter.rs 详情

`RigAdapter<M>` 将任何 rig-core `CompletionModel` 桥接到 `LlmProvider`。它为所有非 NEAR AI 提供商（OpenAI、Anthropic、Ollama、Tinfoil、OpenAI 兼容）在生产环境中积极使用。关键行为：
- **每请求模型覆盖被静默忽略**（记录警告）；模型在构造时烘焙
- **OpenAI 严格模式模式规范化**应用于所有工具定义：`additionalProperties: false`，所有属性添加到 `required`，可选字段通过 `"type": ["T", "null"]` 可为空。这在提供商边界透明地发生。
- **系统消息**被提取到 rig-core `preamble` 字段中（如果有多个则用换行符连接）
- **工具调用 ID**被生成（`generated_tool_call_{seed}`），如果提供商返回空/空白 ID
- **工具名称规范化**：如果匹配已知工具，则剥离 `proxy_` 前缀（处理某些代理实现）
- **OpenAI 使用 Chat Completions API**（`completions_api()`），而不是较新的 Responses API — 当工具结果发送回时，Responses API 路径会 panic（rig-core 没有通过 `ToolCall` 线程 `call_id`）

## 流式支持

无流式支持。所有提供商使用非流式（阻塞）Chat Completions 请求。只有在完整响应可用时，`complete()` 和 `complete_with_tools()` 方法才返回。

## 跟踪记录

设置 `IRONCLAW_RECORD_TRACE=1` 通过 `RecordingLlm` 启用实时跟踪记录。跟踪是包含以下内容的 JSON 文件：内存快照、来自工具的 HTTP 交换以及 LLM 步骤（用户输入、文本响应、工具调用响应）。在 E2E 测试中通过 `TraceLlm` 重放这些。使用 `IRONCLAW_TRACE_OUTPUT` 配置输出路径（默认：`trace_{timestamp}.json`）。
