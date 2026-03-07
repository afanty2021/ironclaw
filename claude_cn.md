# IronClaw 开发指南

## 项目概述

**IronClaw** 是一款安全的个人 AI 助手，保护您的数据并动态扩展其能力。

### 核心理念
- **用户安全至上** - 您的数据始终属于您，加密并本地存储
- **自我扩展** - 无需依赖供应商，动态构建新工具
- **纵深防御** - 多层安全防护，抵御提示注入和数据泄露
- **始终可用** - 多通道访问，支持主动后台执行

### 功能特性
- **多通道输入**：TUI（Ratatui）、HTTP Webhook、WASM 通道（Telegram、Slack）、Web 网关
- **并行任务执行**：配备状态机和卡死任务自修复机制
- **沙箱执行**：Docker 容器隔离，配备网络代理和凭证注入
- **Claude Code 模式**：将任务委托给容器内的 Claude CLI
- **技能系统**：SKILL.md 提示扩展，配备信任模型、工具衰减和 ClawHub 注册表
- **例行任务**：定时（cron）和响应式（事件、webhook）任务执行
- **Web 网关**：浏览器 UI，支持 SSE/WebSocket 实时流式传输
- **扩展管理**：安装、认证、激活 MCP/WASM 扩展
- **可扩展工具**：内置工具、WASM 沙箱、MCP 客户端、动态构建器
- **持久化记忆**：工作空间，支持混合搜索（FTS + 向量 RRF）
- **提示注入防御**：清理器、验证器、策略规则、泄露检测、Shell 环境清理
- **多提供商 LLM**：NEAR AI、OpenAI、Anthropic、Ollama、OpenAI 兼容、Tinfoil 私有推理
- **设置向导**：7 步交互式首次运行配置
- **心跳系统**：主动周期性执行，配备检查清单

## 构建与测试

```bash
# 格式化代码
cargo fmt

# 代码检查（提交前修复所有警告，包括预先存在的警告）
cargo clippy --all --benches --tests --examples --all-features

# 运行所有测试
cargo test

# 运行特定测试
cargo test test_name

# 带日志运行
RUST_LOG=ironclaw=debug cargo run

# 运行集成测试（可能需要运行服务/数据库）
cargo test --test workspace_integration
cargo test --test ws_gateway_integration
cargo test --test heartbeat_integration

# 运行 E2E 测试（Python/Playwright — 需要运行中的 ironclaw 实例）
# 完整设置说明参见 tests/e2e/CLAUDE.md
cd tests/e2e
python -m venv .venv && source .venv/bin/activate  # Windows: .venv\Scripts\activate
pip install -e .
playwright install chromium
pytest scenarios/                    # 所有场景
pytest scenarios/test_chat.py        # 特定场景
```

### 测试层级

| 层级 | 命令 | 运行内容 | 外部依赖 |
|------|------|----------|----------|
| 单元测试 | `cargo test` | 所有 `mod tests` + 自包含集成测试 | 无 |
| 集成测试 | `cargo test --features integration` | + PostgreSQL 依赖测试 | 运行中的 PostgreSQL |
| 实时测试 | `cargo test --features integration -- --ignored` | + LLM 依赖测试 | PostgreSQL + LLM API 密钥 |

运行 `bash scripts/check-boundaries.sh` 验证测试分层和其他架构规则。

## 项目结构

```
src/
├── lib.rs              # 库根，模块声明
├── main.rs             # 入口点，CLI 参数，启动
├── app.rs              # 应用启动编排（通道连接，数据库初始化）
├── bootstrap.rs        # 基目录解析（~/.ironclaw），早期 .env 加载
├── settings.rs         # 用户设置持久化（~/.ironclaw/settings.json）
├── service.rs          # 操作系统服务管理（launchd/systemd 守护进程安装）
├── tracing_fmt.rs      # 自定义追踪格式化器
├── util.rs             # 共享工具
├── config/             # 环境变量配置（按子系统拆分）
│   ├── mod.rs          # 重导出所有配置类型；顶层 Config 结构
│   ├── agent.rs, llm.rs, channels.rs, database.rs, sandbox.rs, skills.rs
│   ├── heartbeat.rs, routines.rs, safety.rs, embeddings.rs, wasm.rs
│   ├── tunnel.rs       # 隧道提供商配置（TUNNEL_PROVIDER, TUNNEL_URL 等）
│   └── secrets.rs, hygiene.rs, builder.rs, helpers.rs
├── error.rs            # 错误类型（thiserror）
│
├── agent/              # 核心代理循环、调度器、会话 — 见 src/agent/CLAUDE.md
│
├── channels/           # 多通道输入
│   ├── channel.rs      # Channel trait, IncomingMessage, OutgoingResponse
│   ├── manager.rs      # ChannelManager 合并流
│   ├── cli/            # 完整 TUI（Ratatui）
│   │   ├── mod.rs      # TuiChannel 实现
│   │   ├── app.rs      # 应用状态
│   │   ├── render.rs   # UI 渲染
│   │   ├── events.rs   # 输入处理
│   │   ├── overlay.rs  # 批准覆盖层
│   │   └── composer.rs # 消息编写
│   ├── http.rs         # HTTP webhook（axum），密钥验证
│   ├── webhook_server.rs # 统一 HTTP 服务器，组合所有 webhook 路由
│   ├── repl.rs         # 简单 REPL（用于测试）
│   ├── web/            # Web 网关（浏览器 UI）— 见 src/channels/web/CLAUDE.md
│   └── wasm/           # WASM 通道运行时
│       ├── mod.rs
│       ├── bundled.rs  # 打包通道发现
│       ├── capabilities.rs # 通道特定能力（HTTP 端点，发送速率）
│       ├── error.rs    # WASM 通道错误类型
│       ├── runtime.rs  # WASM 通道执行运行时
│       └── wrapper.rs  # WASM 模块的 Channel trait 包装器
│
├── cli/                # CLI 子命令（clap）
│   ├── mod.rs          # Cli 结构，Command 枚举（run/onboard/config/tool/registry/mcp/memory/pairing/service/doctor/status/completion）
│   ├── config.rs       # config list/get/set 子命令
│   ├── tool.rs         # tool install/list/remove 子命令
│   ├── registry.rs     # registry list/install 子命令
│   ├── mcp.rs          # mcp add/auth/list/test 子命令
│   ├── memory.rs       # memory search/read/write 子命令
│   ├── pairing.rs      # pairing list/approve 子命令
│   ├── service.rs      # service install/start/stop 子命令
│   ├── doctor.rs       # 主动健康诊断
│   ├── status.rs       # 系统健康/状态显示
│   ├── completion.rs   # Shell 补全脚本生成
│   └── oauth_defaults.rs # 默认 OAuth 重定向 URI
│
├── registry/           # 扩展注册表目录
│   ├── mod.rs          # 公共 API；重导出 RegistryCatalog, RegistryInstaller, 清单类型
│   ├── manifest.rs     # ExtensionManifest, ArtifactSpec, BundleDefinition 类型
│   ├── catalog.rs      # RegistryCatalog：从文件系统和嵌入的 JSON 加载
│   ├── installer.rs    # RegistryInstaller：下载、验证、安装 WASM 制品
│   ├── artifacts.rs    # 制品下载和缓存
│   └── embedded.rs     # 编译时嵌入二进制文件的目录（通过 build.rs）
│
├── hooks/              # 拦截代理操作的生命周期钩子
│   ├── mod.rs          # 6 个 HookPoints：BeforeInbound, BeforeToolCall, BeforeOutbound, OnSessionStart, OnSessionEnd, TransformResponse
│   ├── hook.rs         # Hook trait, HookContext, HookEvent, HookOutcome, HookFailureMode
│   ├── registry.rs     # HookRegistry：注册、优先级排序、执行钩子
│   └── bundled.rs      # 内置钩子：基于规则的过滤器、webhook 转发器、HookBundleConfig
│
├── tunnel/             # 公共互联网暴露的隧道抽象
│   ├── mod.rs          # Tunnel trait, TunnelProviderConfig, create_tunnel() 工厂
│   ├── cloudflare.rs   # CloudflareTunnel（cloudflared 二进制）
│   ├── ngrok.rs        # NgrokTunnel
│   ├── tailscale.rs    # TailscaleTunnel（serve/funnel 模式）
│   ├── custom.rs       # CustomTunnel（带有 {host}/{port} 的任意命令）
│   └── none.rs         # NoneTunnel（仅本地，无暴露）
│
├── observability/      # 可插拔事件/指标记录
│   ├── mod.rs          # create_observer() 工厂，ObservabilityConfig
│   ├── traits.rs       # Observer trait, ObserverEvent, ObserverMetric
│   ├── noop.rs         # NoopObserver（零开销，默认）
│   ├── log.rs          # LogObserver（基于追踪）
│   └── multi.rs        # MultiObserver（多后端扇出）
│
├── orchestrator/       # 沙箱容器的内部 HTTP API
│   ├── mod.rs
│   ├── api.rs          # Axum 端点（LLM 代理、事件、提示）
│   ├── auth.rs         # 每任务 bearer 令牌存储
│   └── job_manager.rs  # 容器生命周期（创建、停止、清理）
│
├── worker/             # 在 Docker 容器内运行
│   ├── mod.rs
│   ├── runtime.rs      # Worker 执行循环（工具调用、LLM）
│   ├── claude_bridge.rs # Claude Code 网桥（生成 claude CLI）
│   ├── api.rs          # 编排器的 HTTP 客户端
│   └── proxy_llm.rs    # 通过编排器代理的 LlmProvider
│
├── safety/             # 提示注入防御
│   ├── sanitizer.rs    # 模式检测、内容转义
│   ├── validator.rs    # 输入验证（长度、编码、模式）
│   ├── policy.rs       # PolicyRule 系统，带有严重性/操作
│   ├── leak_detector.rs # 秘密检测（API 密钥、令牌等）
│   └── credential_detect.rs # HTTP 请求凭证检测（标头、URL 参数）
│
├── llm/                # 多提供商 LLM 集成 — 见 src/llm/CLAUDE.md
│
├── tools/              # 可扩展工具系统
│   ├── tool.rs         # Tool trait, ToolOutput, ToolError
│   ├── registry.rs     # ToolRegistry 发现
│   ├── sandbox.rs      # 基于进程的沙箱（存根，被 wasm/ 取代）
│   ├── rate_limiter.rs # 内置和 WASM 工具的共享滑动窗口速率限制器
│   ├── builtin/        # 内置工具
│   │   ├── echo.rs, time.rs, json.rs, http.rs
│   │   ├── web_fetch.rs # GET URL → 清洁 Markdown（readability + html-to-md 转换）
│   │   ├── file.rs     # ReadFile, WriteFile, ListDir, ApplyPatch
│   │   ├── shell.rs    # Shell 命令执行
│   │   ├── memory.rs   # 记忆工具（search, write, read, tree）
│   │   ├── message.rs  # MessageTool：代理在任何通道上主动向用户发送消息
│   │   ├── job.rs      # CreateJob, ListJobs, JobStatus, CancelJob
│   │   ├── routine.rs  # routine_create/list/update/delete/history
│   │   ├── extension_tools.rs # 扩展 install/auth/activate/remove
│   │   ├── skill_tools.rs # skill_list/search/install/remove 工具
│   │   ├── secrets_tools.rs # secret_list/secret_delete（零暴露：不暴露值）
│   │   ├── html_converter.rs # HTML→Markdown（readability + html-to-markdown-rs）
│   │   ├── path_utils.rs # 共享路径验证/规范化助手
│   │   └── marketplace.rs, ecommerce.rs, taskrabbit.rs, restaurant.rs（存根）
│   ├── builder/        # 动态工具构建
│   │   ├── core.rs     # BuildRequirement, SoftwareType, Language
│   │   ├── templates.rs # 项目脚手架
│   │   ├── testing.rs  # 测试工具集成
│   │   └── validation.rs # WASM 验证
│   ├── mcp/            # Model Context Protocol
│   │   ├── client.rs   # HTTP 上的 MCP 客户端
│   │   ├── protocol.rs # JSON-RPC 类型
│   │   └── session.rs  # MCP 会话管理（Mcp-Session-Id 标头，每服务器状态）
│   └── wasm/           # 完整 WASM 沙箱（wasmtime）
│       ├── runtime.rs  # 模块编译和缓存
│       ├── wrapper.rs  # WASM 模块的 Tool trait 包装器
│       ├── host.rs     # 主机函数（日志、时间、工作空间）
│       ├── limits.rs   # 燃料计量和内存限制
│       ├── allowlist.rs # 网络端点允许列表
│       ├── credential_injector.rs # 安全凭证注入
│       ├── loader.rs   # 从文件系统发现 WASM 工具
│       ├── rate_limiter.rs # 每工具速率限制
│       ├── error.rs    # WASM 特定错误类型
│       └── storage.rs  # 线性内存持久化
│
├── db/                 # 双后端持久化（PostgreSQL + libSQL）— 见 src/db/CLAUDE.md
│
├── workspace/          # 持久化记忆系统（OpenClaw 启发）
│   ├── mod.rs          # Workspace 结构，记忆操作
│   ├── document.rs     # MemoryDocument, MemoryChunk, WorkspaceEntry
│   ├── chunker.rs      # 文档分块（800 令牌，15% 重叠）
│   ├── embeddings.rs   # EmbeddingProvider trait，OpenAI 实现
│   ├── search.rs       # 混合搜索，RRF 算法
│   └── repository.rs   # PostgreSQL CRUD 和搜索操作
│
├── context/            # 任务上下文隔离
│   ├── state.rs        # JobState 枚举，JobContext，状态机
│   ├── memory.rs       # ActionRecord, ConversationMemory
│   └── manager.rs      # ContextManager 并发任务管理
│
├── estimation/         # 成本/时间/价值估算
│   ├── cost.rs         # CostEstimator
│   ├── time.rs         # TimeEstimator
│   ├── value.rs        # ValueEstimator（利润率）
│   └── learner.rs      # 指数移动平均学习
│
├── evaluation/         # 成功评估
│   ├── success.rs      # SuccessEvaluator trait，RuleBasedEvaluator，LlmEvaluator
│   └── metrics.rs      # MetricsCollector，QualityMetrics
│
├── sandbox/            # Docker 执行沙箱
│   ├── mod.rs          # 公共 API，默认允许列表
│   ├── config.rs       # SandboxConfig，SandboxPolicy 枚举
│   ├── manager.rs      # SandboxManager 编排
│   ├── container.rs    # ContainerRunner，Docker 生命周期
│   ├── error.rs        # SandboxError 类型
│   └── proxy/          # 容器的网络代理
│       ├── mod.rs      # NetworkProxyBuilder
│       ├── http.rs     # HttpProxy，CredentialResolver trait
│       ├── policy.rs   # NetworkPolicyDecider trait
│       └── allowlist.rs # DomainAllowlist 验证
│
├── secrets/            # 秘密管理
│   ├── mod.rs          # SecretsStore trait，公共 API
│   ├── types.rs        # 核心类型（Secret, SecretRef, SecretMetadata）
│   ├── crypto.rs       # AES-256-GCM 加密
│   ├── keychain.rs     # 操作系统钥匙串集成（macOS Keychain，GNOME Keyring）用于主密钥
│   └── store.rs        # 加密秘密存储
│
├── setup/              # 入门向导（规范：src/setup/README.md）
│   ├── mod.rs          # 入口点，check_onboard_needed()
│   ├── wizard.rs       # 7 步交互式向导
│   ├── channels.rs     # 通道设置助手
│   └── prompts.rs      # 终端提示（select，confirm，secret）
│
├── skills/             # SKILL.md 提示扩展系统
│   ├── mod.rs          # 核心类型（SkillTrust，LoadedSkill）
│   ├── registry.rs     # SkillRegistry：发现、安装、移除
│   ├── selector.rs     # 确定性评分预过滤器
│   ├── attenuation.rs  # 基于信任的工具上限
│   ├── gating.rs       # 需求检查（bins，env，config）
│   ├── parser.rs       # SKILL.md frontmatter + markdown 解析器
│   └── catalog.rs      # ClawHub 注册表客户端
│
└── history/            # 持久化
    ├── store.rs        # PostgreSQL 存储库
    └── analytics.rs    # 聚合查询（JobStats，ToolStats）

tests/
├── *.rs                # 集成测试（workspace，heartbeat，WS 网关，pairing 等）
├── test-pages/         # HTML→Markdown 转换固定装置（CNN，Medium，Yahoo）
└── e2e/                # Python/Playwright E2E 场景（见 tests/e2e/CLAUDE.md）
```

## 关键模式

### 架构

设计新功能或系统时，始终优先考虑通用/可扩展架构，而不是硬编码特定集成。在实现之前，请询问有关所需抽象级别的问题。

### 错误处理
- 在 `error.rs` 中使用 `thiserror` 定义错误类型
- 永远不要在生产代码中使用 `.unwrap()` 或 `.expect()`（测试可以）
- 使用上下文映射错误：`.map_err(|e| SomeError::Variant { reason: e.to_string() })?`
- 提交前，在更改的文件中 grep `.unwrap()` 和 `.expect(`，以机械方式捕获违规

### 异步
- 所有 I/O 都是使用 tokio 的异步
- 对任务间的共享状态使用 `Arc<T>`
- 对并发读/写访问使用 `RwLock`

### 可扩展性 Trait
- `Database` - 添加新的数据库后端（必须实现所有约 78 个方法）
- `Channel` - 添加新的输入源
- `Tool` - 添加新功能
- `LlmProvider` - 添加新的 LLM 后端
- `SuccessEvaluator` - 自定义评估逻辑
- `EmbeddingProvider` - 添加嵌入后端（工作空间搜索）
- `NetworkPolicyDecider` - 沙箱容器的自定义网络访问策略
- `Hook` - 6 个拦截点的生命周期钩子（BeforeInbound，BeforeToolCall，BeforeOutbound，OnSessionStart，OnSessionEnd，TransformResponse）
- `Observer` - 可观察性后端（noop/log/multi；未来：OpenTelemetry，Prometheus）
- `Tunnel` - 公共互联网暴露的隧道提供商

### 工具实现
```rust
#[async_trait]
impl Tool for MyTool {
    fn name(&self) -> &str { "my_tool" }
    fn description(&self) -> &str { "执行有用的操作" }
    fn parameters_schema(&self) -> serde_json::Value {
        serde_json::json!({
            "type": "object",
            "properties": {
                "param": { "type": "string", "description": "一个参数" }
            },
            "required": ["param"]
        })
    }

    async fn execute(&self, params: serde_json::Value, ctx: &JobContext)
        -> Result<ToolOutput, ToolError>
    {
        let start = std::time::Instant::now();
        // ... 执行工作 ...
        Ok(ToolOutput::text("result", start.elapsed()))
    }

    fn requires_sanitization(&self) -> bool { true } // 外部数据
}
```

### 状态转换
任务状态遵循 `context/state.rs` 中定义的状态机：
```
Pending -> InProgress -> Completed -> Submitted -> Accepted
                     \-> Failed
                     \-> Stuck -> InProgress（恢复）
                              \-> Failed
```

### 代码风格

- 使用 `crate::` 导入，而不是 `super::`
- 除非向下游消费者公开，否则不进行 `pub use` 重导出
- 优先使用强类型而不是字符串（枚举、newtypes）
- 保持函数专注，在重用逻辑时提取助手
- 仅对非显而易见的逻辑添加注释

### 审查与修复纪律

代码审查中吸取的教训 — 在修复 bug 或处理审查反馈时遵循这些规则。

**修复模式，而不仅仅是实例：** 当审查者标记一个 bug（例如，INSERT + SELECT-back 中的 TOCTOU 竞争）时，搜索整个代码库中所有相同模式的实例。在 `SecretsStore::create()` 中的修复如果不同时修复 `WasmToolStore::store()`，那就是一半的修复。

**将架构修复传播到卫星类型：** 如果核心类型更改其并发模型（例如，`LibSqlBackend` 切换到每操作连接），每个从旧模型接收资源的类型（例如，持有单个 `Connection` 的 `LibSqlSecretsStore`，`LibSqlWasmToolStore`）也必须更新。在整个代码库中 grep 旧类型。

**模式转换不仅仅是 DDL：** 在后端之间转换数据库模式时（PostgreSQL 到 libSQL 等），检查：
- **索引** -- 对比两个模式之间的 `CREATE INDEX` 语句
- **种子数据** -- 检查迁移中的 `INSERT INTO`（例如，`leak_detection_patterns`）
- **语义差异** -- 记录 SQL 函数行为不同的地方（例如，`json_patch` vs `jsonb_set`）

**功能标志测试：** 添加功能门控代码时，测试每个功能的单独编译：
```bash
cargo check                                          # 默认功能
cargo check --no-default-features --features libsql  # 仅 libsql
cargo check --all-features                           # 所有功能
```
错误的 `#[cfg]` 门后的死代码只会在使用单个功能构建时显示。

**每次修复都进行回归测试：** 每个 bug 修复都必须包含能够捕获该 bug 的测试。添加一个重现原始失败的 `#[test]` 或 `#[tokio::test]`。豁免：仅限于 `src/channels/web/static/` 或 `.md` 文件的更改。如果确实不可行，请在提交消息或 PR 标签中使用 `[skip-regression-check]`。`commit-msg` 钩子和 CI 工作流自动强制执行此操作。

**零 clippy 警告策略：** 提交前修复所有 clippy 警告，包括您未更改的文件中的预先存在的警告。永远不要留下警告 — 将 `cargo clippy` 输出视为零容忍门槛。

**提交前的机械验证：** 在提交之前对更改的文件运行这些检查：
- `cargo clippy --all --benches --tests --examples --all-features` -- 零警告
- `grep -rnE '\.unwrap\(|\.expect\(' <files>` -- 生产代码中没有 panic
- `grep -rn 'super::' <files>` -- 使用 `crate::` 导入
- 如果修复了模式 bug，在 `src/` 中 grep 该模式的其他实例
- 修复提交必须包含回归测试（由 `commit-msg` 钩子强制执行；使用 `[skip-regression-check]` 绕过）

## 配置

环境变量（参见 `.env.example`）：
```bash
# 数据库后端（默认：postgres）
DATABASE_BACKEND=postgres               # 或 "libsql" / "turso"
DATABASE_URL=postgres://user:pass@localhost/ironclaw
LIBSQL_PATH=~/.ironclaw/ironclaw.db    # libSQL 本地路径（默认）
# LIBSQL_URL=libsql://xxx.turso.io    # Turso 云（可选）
# LIBSQL_AUTH_TOKEN=xxx                # 使用 LIBSQL_URL 时需要

# NEAR AI（当 LLM_BACKEND=nearai 时，默认）
# 两种认证模式：会话令牌（默认）或 API 密钥
# 会话令牌认证（默认）：首次运行时使用浏览器 OAuth
NEARAI_SESSION_TOKEN=sess_...           # 托管提供商：设置此项
NEARAI_BASE_URL=https://private.near.ai
# API 密钥认证：设置 NEARAI_API_KEY，基本 URL 默认为 cloud-api.near.ai
# NEARAI_API_KEY=...                    # 来自 cloud.near.ai 的 API 密钥
NEARAI_MODEL=claude-3-5-sonnet-20241022

# 代理设置
AGENT_NAME=ironclaw
MAX_PARALLEL_JOBS=5

# 嵌入（语义记忆搜索）
OPENAI_API_KEY=sk-...                   # 用于 OpenAI 嵌入
# 或使用 NEAR AI 嵌入：
# EMBEDDING_PROVIDER=nearai
# EMBEDDING_ENABLED=true
EMBEDDING_MODEL=text-embedding-3-small  # 或 text-embedding-3-large

# 心跳（主动周期性执行）
HEARTBEAT_ENABLED=true
HEARTBEAT_INTERVAL_SECS=1800            # 30 分钟
HEARTBEAT_NOTIFY_CHANNEL=tui
HEARTBEAT_NOTIFY_USER=default

# Web 网关
GATEWAY_ENABLED=true
GATEWAY_HOST=127.0.0.1
GATEWAY_PORT=3001
GATEWAY_AUTH_TOKEN=changeme           # API 访问所需
GATEWAY_USER_ID=default

# Docker 沙箱
SANDBOX_ENABLED=true
SANDBOX_IMAGE=ironclaw-worker:latest
SANDBOX_MEMORY_LIMIT_MB=512
SANDBOX_TIMEOUT_SECS=1800
SANDBOX_CPU_LIMIT=1.0                  # 每容器 CPU 核心数
SANDBOX_NETWORK_PROXY=true             # 为容器启用网络代理
SANDBOX_PROXY_PORT=8080                # 代理监听端口
SANDBOX_DEFAULT_POLICY=workspace_write # ReadOnly, WorkspaceWrite, FullAccess

# Claude Code 模式（在沙箱容器内运行）
CLAUDE_CODE_ENABLED=false
CLAUDE_CODE_MODEL=claude-sonnet-4-20250514
CLAUDE_CODE_MAX_TURNS=50
CLAUDE_CODE_CONFIG_DIR=/home/worker/.claude

# 例行任务（定时/响应式执行）
ROUTINES_ENABLED=true
ROUTINES_CRON_INTERVAL=60            # 计时间隔（秒）
ROUTINES_MAX_CONCURRENT=3

# 技能系统
SKILLS_ENABLED=true
SKILLS_MAX_TOKENS=4000                 # 每轮最大提示预算
SKILLS_CATALOG_URL=https://clawhub.dev # ClawHub 注册表 URL
SKILLS_AUTO_DISCOVER=true              # 启动时扫描技能目录

# Tinfoil 私有推理
TINFOIL_API_KEY=...                    # 当 LLM_BACKEND=tinfoil 时需要
TINFOIL_MODEL=kimi-k2-5               # 默认模型

# 隧道（webhook 的公共互联网暴露）
TUNNEL_URL=https://abc123.ngrok.io     # 静态公共 URL（手动隧道）
# 或使用托管隧道提供商：
TUNNEL_PROVIDER=none                   # none（默认），cloudflare，tailscale，ngrok，custom
TUNNEL_CF_TOKEN=...                    # TUNNEL_PROVIDER=cloudflare 时需要
TUNNEL_NGROK_TOKEN=...                 # TUNNEL_PROVIDER=ngrok 时需要
# TUNNEL_NGROK_DOMAIN=...             # 自定义域（付费 ngrok 计划）
# TUNNEL_TS_FUNNEL=true               # 使用 tailscale funnel（公共）vs serve（tailnet）
TUNNEL_CUSTOM_COMMAND=...              # 自定义提供商的带有 {host}/{port} 的命令

# 可观察性后端
OBSERVABILITY_BACKEND=none             # none/noop（默认）或 log
```

### LLM 提供商

后端：`nearai`（默认）、`openai`、`anthropic`、`ollama`、`openai_compatible`、`tinfoil` — 通过 `LLM_BACKEND` 设置。有关每个提供商的认证和配置详细信息，请参阅 [src/llm/CLAUDE.md](src/llm/CLAUDE.md)。

## 数据库

双后端持久化（PostgreSQL + libSQL/Turso）。**所有新的持久化功能必须支持两个后端** — 有关模式、SQL 方言差异、添加操作和 libSQL 限制，请参阅 [src/db/CLAUDE.md](src/db/CLAUDE.md)。

在 `src/db/postgres.rs` 和 `src/db/libsql/mod.rs` 中实现每个新操作。单独测试：
```bash
cargo check                                          # postgres（默认）
cargo check --no-default-features --features libsql  # 仅 libsql
cargo check --all-features                           # 两者
```

数据库配置：参见上面的配置部分。

## 安全层

所有外部工具输出都通过 `SafetyLayer`：
1. **清理器** - 检测注入模式，转义危险内容
2. **验证器** - 检查长度、编码、禁止模式
3. **策略** - 规则，带有严重性（Critical/High/Medium/Low）和操作（Block/Warn/Review/Sanitize）
4. **泄露检测器** - 扫描 15+ 种秘密模式（API 密钥、令牌、私钥、连接字符串），在两个点：工具输出到达 LLM 之前，以及 LLM 响应到达用户之前。每种模式的操作：Block（完全拒绝）、Redact（掩盖秘密）或 Warn（标记但允许）

工具输出在到达 LLM 之前被包装：
```xml
<tool_output name="search" sanitized="true">
[转义的内容]
</tool_output>
```

### Shell 环境清理

shell 工具（`src/tools/builtin/shell.rs`）在执行命令之前清理敏感环境变量，防止秘密通过 `env`、`printenv` 或 `$VAR` 扩展泄露。清理器（`src/safety/sanitizer.rs`）还检测命令注入模式（链式命令、子 shell、路径遍历），并根据策略规则阻止或转义它们。

## 技能系统

技能是 SKILL.md 文件，使用领域特定指令扩展代理的提示。每个技能都是 YAML frontmatter 块（元数据、激活标准、所需工具），后跟 markdown 正文，在技能激活时注入到 LLM 上下文中。

### 信任模型

| 信任级别 | 来源 | 工具访问 |
|---------|------|----------|
| **受信任** | 用户放置在 `~/.ironclaw/skills/` 或工作空间 `skills/` 中 | 代理可用的所有工具 |
| **已安装** | 从 ClawHub 注册表下载 | 仅只读工具（无 shell、文件写入、HTTP） |

### SKILL.md 格式

```yaml
---
name: my-skill
version: 0.1.0
description: 执行有用的操作
activation:
  patterns:
    - "部署到.*生产"
  keywords:
    - "部署"
  max_context_tokens: 2000
metadata:
  openclaw:
    requires:
      bins: [docker, kubectl]
      env: [KUBECONFIG]
---

# 部署技能

当此技能激活时代理的指令...
```

### 选择流程

1. **门控** -- 检查二进制/环境/配置需求；跳过先决条件缺失的技能
2. **评分** -- 使用关键词、标签和正则表达式模式对消息内容进行确定性评分
3. **预算** -- 选择适合 `SKILLS_MAX_TOKENS` 提示预算的最高评分技能
4. **衰减** -- 应用基于信任的工具上限；已安装的技能失去对危险工具的访问

### 技能工具

四个用于运行时管理技能的内置工具：
- **`skill_list`** -- 列出所有发现的技能及其信任级别和状态
- **`skill_search`** -- 搜索 ClawHub 注册表的可用技能
- **`skill_install`** -- 从 ClawHub 下载并安装技能
- **`skill_remove`** -- 移除已安装的技能

### 技能目录

- `~/.ironclaw/skills/` -- 用户的全局技能（受信任）
- `<workspace>/skills/` -- 每工作空间技能（受信任）
- `~/.ironclaw/installed_skills/` -- 注册表安装的技能（已安装信任）

### 测试技能

- `skills/web-ui-test/` -- 通过 Claude for Chrome 扩展进行的 Web 网关 UI 手动测试清单。涵盖连接、聊天、技能搜索/安装/移除和其他标签页。

技能配置：参见上面的配置部分。

## Docker 沙箱

`src/sandbox/` 模块提供基于 Docker 的任务执行隔离，配备控制出站访问和注入凭证的网络代理。

### 沙箱策略

| 策略 | 文件系统 | 网络 | 用例 |
|------|----------|------|------|
| **ReadOnly** | 只读工作空间挂载 | 仅允许列表域 | 分析、代码审查 |
| **WorkspaceWrite** | 读写工作空间挂载 | 仅允许列表域 | 代码生成、文件编辑 |
| **FullAccess** | 完整文件系统 | 无限制 | 受信任的管理任务 |

### 网络代理

容器通过主机侧代理（`src/sandbox/proxy/`）路由所有 HTTP/HTTPS 流量：
- **域允许列表** -- 只有允许列表中的域是可达的（默认：包注册表、文档站点、GitHub、常见 API）
- **凭证注入** -- `CredentialResolver` trait 将认证标头注入到代理请求中，因此秘密永远不会进入容器环境
- **CONNECT 隧道** -- HTTPS 流量使用 CONNECT 方法；代理在建立隧道之前根据允许列表验证目标域
- **策略决策** -- `NetworkPolicyDecider` trait 允许每个请求的允许/拒绝/注入决策的自定义逻辑

### 零暴露凭证模型

秘密（API 密钥、令牌）在主机上加密存储，并由代理在传输时注入到 HTTP 请求中。容器进程永远无法访问原始凭证值，即使容器代码受损也能防止 exfiltration。

沙箱配置：参见上面的配置部分。

## 测试

测试位于每个文件底部的 `mod tests {}` 块中。运行特定模块测试：
```bash
cargo test safety::sanitizer::tests
cargo test tools::registry::tests
```

关键测试模式：
- 纯函数的单元测试
- 使用 `#[tokio::test]` 的异步测试
- 无模拟，优先使用真实实现或存根

## 当前限制 / 待办事项

1. **特定域的工具** - `marketplace.rs`、`restaurant.rs`、`taskrabbit.rs`、`ecommerce.rs` 返回占位符响应；需要真实的 API 集成
2. **集成测试** - 需要 PostgreSQL 的 testcontainers 设置
3. **MCP stdio 传输** - 仅实现 HTTP 传输
4. **WIT bindgen 集成** - 从 WASM 模块自动提取工具描述/模式（存根）
5. **工具构建后的能力授予** - 构建的工具获得空能力；需要授予 HTTP/秘密访问的 UX
6. **工具版本控制工作流** - 动态构建的工具没有版本跟踪或回滚
7. **完整通道状态视图** - 网关状态小部件存在，但没有每通道连接仪表板
8. **可观察性后端** - 仅实现 `log` 和 `noop`；尚未支持 OpenTelemetry/Prometheus

## 工具架构

**将特定于工具的逻辑排除在主代理代码库之外。** 主代理提供通用基础设施；工具是通过 `capabilities.json` 文件声明其要求的自包含单元（API 端点、凭证、速率限制、身份验证设置）。特定于服务的身份验证流程、CLI 命令和配置不属于主代理。

工具可以构建为 **WASM**（沙箱化、凭证注入、单个二进制）或 **MCP 服务器**（预构建服务器的生态系统，任何语言，但无沙箱）。两者都是通过 `ironclaw tool install` 的一等公民。在 capabilities 文件中声明身份验证，支持 OAuth 和手动令牌输入。

有关完整工具架构、添加新工具（内置 Rust 和 WASM）、身份验证 JSON 示例以及 WASM 与 MCP 决策指南，请参阅 `src/tools/README.md`。

## 添加新通道

1. 创建 `src/channels/my_channel.rs`
2. 实现 `Channel` trait
3. 在 `src/config/channels.rs` 中添加配置
4. 在 `src/app.rs` 通道设置部分连接

## 调试

```bash
# 详细日志
RUST_LOG=ironclaw=trace cargo run

# 仅代理模块
RUST_LOG=ironclaw::agent=debug cargo run

# 带有 HTTP 请求日志
RUST_LOG=ironclaw=debug,tower_http=debug cargo run
```

## 模块规范

某些模块有一个 `README.md`，作为该模块行为的权威规范。在修改具有规范的模块中的代码时：

1. **首先阅读规范** 然后再进行更改
2. **代码遵循规范**：如果规范说 X，代码必须执行 X
3. **更新双方**：如果更改行为，更新规范以匹配；
   如果实现规范更改，更新代码以匹配
4. **规范是决胜局**：当代码和规范不一致时，规范是正确的
   （除非规范明显过时，在这种情况下先修复规范）

| 模块 | 规范文件 |
|------|----------|
| `src/setup/` | `src/setup/README.md` |
| `src/workspace/` | `src/workspace/README.md` |
| `src/tools/` | `src/tools/README.md` |
| `src/agent/` | `src/agent/CLAUDE.md` |
| `src/channels/web/` | `src/channels/web/CLAUDE.md` |
| `src/db/` | `src/db/CLAUDE.md` |
| `src/llm/` | `src/llm/CLAUDE.md` |
| `tests/e2e/` | `tests/e2e/CLAUDE.md` |

## 工作空间与记忆系统

OpenClaw 启发的持久化记忆，具有灵活的类文件系统结构。原则："记忆是数据库，而不是 RAM" - 如果您想记住某些内容，请明确写入。使用通过倒数排名融合结合的混合搜索（FTS（关键词）+ 向量（语义））。

LLM 使用的四个记忆工具：`memory_search`（混合搜索 -- 在回答有关先前工作的问题之前调用）、`memory_write`、`memory_read`、`memory_tree`。身份文件（AGENTS.md、SOUL.md、USER.md、IDENTITY.md）被注入到 LLM 系统提示中。

心跳系统运行主动周期性执行（默认：30 分钟），读取 `HEARTBEAT.md`，如果发现发现则通过通道通知。

有关完整 API 文档、文件系统结构、混合搜索详细信息、分块策略和心跳系统，请参阅 `src/workspace/README.md`。
