# 设置 / 入门规范

本文档是 IronClaw 入门向导的权威规范。对 `src/setup/` 的任何代码更改**必须**保持本文档同步。如果未来的贡献者或编码代理修改设置行为，请首先更新此文件，然后调整代码以匹配。

---

## 入口点

```
ironclaw onboard [--skip-auth] [--channels-only]
```

显式调用。加载 `.env` 文件，运行向导，退出。

```
ironclaw          (首次运行，未配置数据库)
```

通过 `main.rs` 中的 `check_onboard_needed()` 自动检测。当设置 `ONBOARD_COMPLETED` 环境变量时（由向导写入 `~/.ironclaw/.env`）跳过入门。否则在未配置数据库时触发：
- 设置了 `DATABASE_URL` 环境变量
- 设置了 `LIBSQL_PATH` 环境变量
- `~/.ironclaw/ironclaw.db` 存在于磁盘上

`--no-onboard` CLI 标志禁止自动检测。

---

## 启动序列（main.rs）

```
1. 解析 CLI 参数
2. 如果 Command::Onboard → 加载 .env，运行向导，退出
3. 如果 Command::Run 或无命令：
   a. 加载 .env 文件（dotenvy::dotenv() 然后 load_ironclaw_env()）
   b. check_onboard_needed() → 如果需要则运行向导
   c. Config::from_env()     → 从环境变量构建配置
   d. 创建 SessionManager  → 加载会话令牌
   e. ensure_authenticated() → 验证会话（仅 NEAR AI）
   f. ... 代理启动的其余部分
```

**关键排序：** `.env` 文件必须在 `Config::from_env()` 之前加载（步骤 3a 到 3c），因为引导变量如 `DATABASE_BACKEND` 位于 `~/.ironclaw/.env` 中。

---

## 8 步向导

### 概述

```
步骤 1：数据库连接
步骤 2：安全性（主密钥）
步骤 3：推理提供商          ← 如果 --skip-auth 则跳过
步骤 4：模型选择
步骤 5：嵌入
步骤 6：通道配置
步骤 7：扩展（工具）
步骤 8：后台任务（心跳）
       ↓
   save_and_summarize()
```

`--channels-only` 模式仅运行步骤 6，跳过其他所有内容。

---

### 步骤 1：数据库连接

**模块：** `wizard.rs` → `step_database()`

**目标：** 选择后端，建立连接，运行迁移。

**决策树：**

```
两个功能都已编译？
├─ 是 → 设置了 DATABASE_BACKEND 环境变量？
│  ├─ 是 → 使用该后端
│  └─ 否  → 交互式选择（PostgreSQL vs libSQL）
├─ 仅 postgres 功能 → step_database_postgres()
└─ 仅 libsql 功能  → step_database_libsql()
```

**PostgreSQL 路径**（`step_database_postgres`）：
1. 从环境或设置检查 `DATABASE_URL`
2. 测试连接（创建 `deadpool_postgres::Pool`）
3. 可选运行 refinery 迁移
4. 在 `self.db_pool` 中存储池

**libSQL 路径**（`step_database_libsql`）：
1. 提供本地路径（默认：`~/.ironclaw/ironclaw.db`）
2. 可选 Turso 云同步（URL + 认证令牌）
3. 测试连接（创建 `LibSqlBackend`）
4. 始终运行迁移（幂等 CREATE IF NOT EXISTS）
5. 在 `self.db_backend` 中存储后端

**不变量：** 步骤 1 之后，`self.db_pool` 或 `self.db_backend` 中恰好有一个是 `Some`。这是 `save_and_summarize()` 中的设置持久化所必需的。

---

### 步骤 2：安全性（主密钥）

**模块：** `wizard.rs` → `step_security()`

**目标：** 为 API 令牌和秘密配置加密。

**决策树：**

```
设置了 SECRETS_MASTER_KEY 环境变量？
├─ 是 → 使用环境变量，完成
└─ 否  → 尝试从 OS 钥匙串获取 get_master_key()
   ├─ Ok(bytes) → 缓存在 self.secrets_crypto，询问"使用现有？"
   │  ├─ 是 → 完成（钥匙串）
   │  └─ 否  → 清除缓存，回退到选项
   └─ Err   → 回退到选项
              ├─ OS 钥匙串：生成 + 存储 + 构建 SecretsCrypto
              ├─ 环境变量：生成 + 打印导出命令
              └─ 跳过：禁用秘密功能
```

**关键警告：macOS 钥匙串对话框**

在 macOS 上，`security_framework::get_generic_password()` 可以触发两个系统对话框：
1. "输入密码以解锁钥匙串"（钥匙串已锁定）
2. "允许 ironclaw 访问此钥匙串项目"（每应用程序授权）

这是我们无法阻止的操作系统级行为。为了最大限度地减少痛苦：

- **使用 `get_master_key()` 而不是 `has_master_key()`** 在步骤 2 中。两者调用相同的基础 API，但 `get_master_key()` 返回密钥字节，因此我们可以缓存它们。`has_master_key()` 会丢弃它们，导致以后的第二次钥匙串访问。

- **急切构建 `SecretsCrypto`。** 当从钥匙串检索密钥时，立即构造 `SecretsCrypto` 并存储在 `self.secrets_crypto` 中。以后对 `init_secrets_context()` 的调用首先检查此字段，避免冗余的钥匙串探测。

- **永远不要在只读命令中探测钥匙串**（例如，`ironclaw status`）。status 命令报告"未设置 env（可能配置了钥匙串）"，而不是触发系统对话框。

**不变量：** 步骤 2 之后，如果用户选择钥匙串或生成了新密钥，`self.secrets_crypto` 是 `Some`。如果用户选择 env-var 模式或跳过秘密，它可能是 `None`。

---

### 步骤 3：推理提供商

**模块：** `wizard.rs` → `step_inference_provider()`

**目标：** 选择 LLM 后端并认证。

**提供商：**

| 提供商 | 认证方法 | 秘密名称 | 环境变量 |
|----------|-------------|-------------|---------|
| NEAR AI Chat | 浏览器 OAuth 或会话令牌 | - | `NEARAI_SESSION_TOKEN` |
| NEAR AI Cloud | API 密钥 | `llm_nearai_api_key` | `NEARAI_API_KEY` |
| Anthropic | API 密钥 | `anthropic_api_key` | `ANTHROPIC_API_KEY` |
| OpenAI | API 密钥 | `openai_api_key` | `OPENAI_API_KEY` |
| Ollama | 无 | - | - |
| OpenRouter¹ | API 密钥 | `llm_compatible_api_key` | `LLM_API_KEY` |
| OpenAI 兼容¹ | 可选 API 密钥 | `llm_compatible_api_key` | `LLM_API_KEY` |

¹ OpenRouter 和 OpenAI 兼容共享相同的秘密名称和环境变量，因为 OpenRouter 存储为 `llm_backend = "openai_compatible"`。在它们之间切换会覆盖相同的凭证槽。

**OpenRouter**（`setup_openrouter`）：
- 预配置的 OpenAI 兼容预设，基本 URL 为 `https://openrouter.ai/api/v1`
- 委托给 `setup_api_key_provider()` 并带有显示名称覆盖（"OpenRouter"）
- 自动设置 `llm_backend = "openai_compatible"` 和 `openai_compatible_base_url`
- 清除 `selected_model`，因此步骤 4 提示输入模型名称（手动文本输入，无基于 API 的模型获取）

**API 密钥提供商**（`setup_api_key_provider`）：
1. 检查环境变量 → 如果设置，询问重用，持久化到秘密存储
2. 否则通过 `secret_input()` 提示输入密钥
3. 通过 `init_secrets_context()` 加密存储在秘密中
4. **在 `self.llm_api_key` 中缓存密钥** 用于步骤 4 中的模型获取

**NEAR AI**（`setup_nearai`）：
- 调用 `session_manager.ensure_authenticated()`，显示认证菜单：
  - 选项 1-2（GitHub/Google）：浏览器 OAuth → **NEAR AI Chat** 模式
    （位于 `private.near.ai` 的 Responses API，会话令牌认证）
  - 选项 4：NEAR AI Cloud API 密钥 → **NEAR AI Cloud** 模式
    （位于 `cloud-api.near.ai` 的 Chat Completions API，API 密钥认证）
- **NEAR AI Chat** 路径：会话令牌保存到 `~/.ironclaw/session.json`。
  托管提供商可以直接设置 `NEARAI_SESSION_TOKEN` 环境变量（优先于基于文件的令牌）。
- **NEAR AI Cloud** 路径：`NEARAI_API_KEY` 保存到 `~/.ironclaw/.env`
  （引导）和加密的秘密存储（`llm_nearai_api_key`）。
  `LlmConfig::resolve()` 在存在 API 密钥时自动选择 `ChatCompletions` 模式。

**`self.llm_api_key` 缓存：** 向导将 API 密钥缓存为 `Option<SecretString>`，以便步骤 4（模型获取）和步骤 5（嵌入）可以使用它，而无需从秘密存储重新读取或改变环境变量。

---

### 步骤 4：模型选择

**模块：** `wizard.rs` → `step_model_selection()`

**目标：** 选择要使用的模型。

**流程：**
1. 如果已设置模型 → 提议保留它
2. 从提供商 API 获取模型（5 秒超时）
3. 在超时或错误时 → 使用静态回退列表
4. 展示列表 + "自定义模型 ID" 逃生舱
5. 存储在 `self.settings.selected_model` 中

**模型获取器显式传递缓存的 API 密钥：**
```rust
let cached = self.llm_api_key.as_ref().map(|k| k.expose_secret().to_string());
let models = fetch_anthropic_models(cached.as_deref()).await;
```

这避免了改变环境变量。获取器首先检查显式密钥，然后回退到标准环境变量。

---

### 步骤 5：嵌入

**模块：** `wizard.rs` → `step_embeddings()`

**目标：** 为工作空间记忆配置语义搜索。

**流程：**
1. 询问"启用语义搜索？"（默认：是）
2. 检测可用的提供商：
   - NEAR AI：如果后端是 `nearai` 或存在有效会话
   - OpenAI：如果 `OPENAI_API_KEY` 在环境中或（后端是 `openai` 且缓存密钥）
3. 如果两者都可用 → 让用户选择
4. 如果只有一个 → 使用它
5. 如果都没有 → 禁用嵌入

**默认模型：** `text-embedding-3-small`（两个提供商）

---

### 步骤 6：通道配置

**模块：** `wizard.rs` → `step_channels()`，委托给 `channels.rs`

**目标：** 启用输入通道（TUI、HTTP、Telegram 等）。

**子步骤：**

```
6a. 隧道设置（如果需要 webhook 通道）
6b. 从 ~/.ironclaw/channels/ 发现 WASM 通道
6c. 构建通道选项：已发现 + 打包 + 注册表目录
6d. 多选：CLI/TUI、HTTP、所有可用通道
6e. 安装缺失的打包通道（复制 WASM 二进制文件）
6f. 安装缺失的注册表通道（下载制品，回退到源构建）
6g. 初始化 SecretsContext（用于令牌存储）
6h. 设置 HTTP webhook（如果选择）
6i. 设置每个 WASM 通道（秘密、所有者绑定）
```

**通道来源**（安装优先级）：
1. 已安装在 `~/.ironclaw/channels/` 中
2. 打包通道（在 `channels-src/` 中预编译）
3. 注册表通道（`registry/channels/*.json`，下载优先源回退）

**隧道设置**（`setup_tunnel`）：
- 选项：ngrok、Cloudflare Tunnel、localtunnel、自定义 URL
- 验证 HTTPS 要求
- 存储在 `self.settings.tunnel.public_url` 中

**WASM 通道设置**（`setup_wasm_channel`）：
- 读取 `capabilities.json` 用于 `setup.required_secrets`
- 对于每个秘密：检查现有、提示或自动生成、验证正则
- 通过 `SecretsContext` 保存每个秘密

**Telegram 特殊情况**（`setup_telegram`）：
- 通过 Telegram `getMe` API 验证机器人令牌
- 所有者绑定：轮询 `getUpdates` 120s 以捕获发送者的用户 ID
- 可选 webhook 密钥生成

**SecretsContext 创建**（`init_secrets_context`）：
1. 检查 `self.secrets_crypto`（在步骤 2 中设置）→ 如果可用则使用
2. 否则尝试 `SECRETS_MASTER_KEY` 环境变量
3. 否则尝试从钥匙串 `get_master_key()`（仅在 `channels_only` 模式）
4. 创建适当后端的秘密存储（尊重所选的数据库后端）

---

### 步骤 7：扩展（工具）

**模块：** `wizard.rs` → `step_extensions()`

**目标：** 从扩展注册表安装 WASM 工具。

**流程：**
1. 从 `registry/` 目录加载 `RegistryCatalog`
2. 如果未找到注册表，打印信息并跳过
3. 列出来自目录的所有工具清单
4. 在 `~/.ironclaw/tools/` 中发现已安装的工具
5. 多选：显示所有注册表工具，带有显示名称、认证方法和描述。预先检查标记为 `"default"` 的工具和已安装的工具。
6. 对于每个选定的尚未安装的工具，通过 `RegistryInstaller::install_with_source_fallback()` 安装（下载优先，源回退）
7. 打印合并的认证提示（按提供商去重，例如，所有共享 `google_oauth_token` 的 Google 工具的一个提示）

**注册表查找**（`load_registry_catalog`）：
按顺序搜索 `registry/` 目录：
1. 当前工作目录
2. 可执行文件旁边
3. `CARGO_MANIFEST_DIR`（编译时，开发构建）

---

### 步骤 8：心跳

**模块：** `wizard.rs` → `step_heartbeat()`

**目标：** 配置周期性后台执行。

**流程：**
1. 询问"启用心跳？"（默认：否）
2. 如果是：间隔分钟（默认：30），通知通道
3. 存储在 `self.settings.heartbeat` 中

---

## 设置持久化

### 两层架构

设置在两个地方持久化：

**第 1 层：`~/.ironclaw/.env`**（引导变量）

仅包含数据库连接之前所需的设置。由 `bootstrap.rs` 中的 `save_bootstrap_env()` 写入。

```env
DATABASE_BACKEND="libsql"
LIBSQL_PATH="/Users/name/.ironclaw/ironclaw.db"
LLM_BACKEND="openai_compatible"
LLM_BASE_URL="http://my-vllm:8000/v1"
```

或对于 PostgreSQL + NEAR AI：
```env
DATABASE_BACKEND="postgres"
DATABASE_URL="postgres://user:pass@localhost/ironclaw"
LLM_BACKEND="nearai"
```

或对于 Ollama：
```env
LLM_BACKEND="ollama"
OLLAMA_BASE_URL="http://localhost:11434"
```

**为什么分开？** 蛋生鸡问题：您需要 `DATABASE_BACKEND` 来知道连接到哪个数据库，`LLM_BACKEND` 来知道是否尝试 NEAR AI 会话认证 — 两者都不能存储在数据库中。

**第 2 层：数据库设置表**（其他所有内容）

所有其他设置作为键值对存储在 `settings` 表中，由 `(user_id, key)` 键控。由 `set_all_settings()` 写入。

设置通过 `Settings::to_db_map()` 序列化为点分路径：
```
database_backend = "libsql"
llm_backend = "nearai"
selected_model = "anthropic/claude-sonnet-4-5"
embeddings.enabled = "true"
embeddings.provider = "nearai"
channels.http_enabled = "true"
heartbeat.enabled = "true"
heartbeat.interval_secs = "300"
```

### 增量持久化

设置在**每个成功步骤后**持久化，而不仅仅是在结束时。这可以防止如果后续步骤失败时的数据丢失（例如，用户在步骤 3 中输入 API 密钥但步骤 5 崩溃 — 他们不需要重新输入）。

**`persist_after_step()`** 在 `run()` 中的每个步骤之后调用，并：
1. 通过 `write_bootstrap_env()` 将引导变量写入 `~/.ironclaw/.env`
2. 通过 `persist_settings()` 将所有当前设置写入数据库
3. 静默忽略错误（例如，如果在建立 DB 之前调用）

**`try_load_existing_settings()`** 在步骤 1 建立数据库连接后调用。它使用 `get_all_settings("default")` → `Settings::from_db_map()` → `merge_from()` 从数据库加载任何以前保存的设置。这从以前的部分向导运行中恢复进度。

**步骤 1 后的排序至关重要：**

```
step_database()                        → 在 self.settings 中设置 DB 字段
let step1 = self.settings.clone()      → 快照步骤 1 选择
try_load_existing_settings()           → 将 DB 值合并到 self.settings
self.settings.merge_from(&step1)       → 重新应用步骤 1（新鲜胜过陈旧）
persist_after_step()                   → 保存合并状态
```

此排序确保：
- 先前进度（来自以前部分运行的步骤 2-7）被恢复
- 新鲜步骤 1 选择覆盖陈旧 DB 值（而不是相反）
- 第一次 DB 持久化不会用默认值破坏以前的设置

### save_and_summarize()

向导的最后一步：

```
1. 标记 onboard_completed = true
2. 调用 persist_settings() 进行最终写入（幂等 — 确保
   保存了 onboard_completed 标志）
3. 调用 write_bootstrap_env() 进行最终 .env 写入（幂等）
4. 打印配置摘要
```

写入 `~/.ironclaw/.env` 的引导变量：
- `DATABASE_BACKEND`（始终）
- `DATABASE_URL`（如果 postgres）
- `LIBSQL_PATH`（如果 libsql）
- `LIBSQL_URL`（如果 turso 同步）
- `LLM_BACKEND`（始终，当设置时）
- `LLM_BASE_URL`（如果 openai_compatible）
- `OLLAMA_BASE_URL`（如果 ollama）
- `NEARAI_API_KEY`（如果 API 密钥认证路径）
- `ONBOARD_COMPLETED`（始终，"true"）

**不变量：** 必须写入第 1 层和第 2 层。如果数据库写入失败，向导返回错误，并且不写入 `.env` 文件。

### 遗留迁移

`bootstrap.rs` 处理从旧配置格式的一次性升级：
- `bootstrap.json` → 提取 `DATABASE_URL`，写入 `.env`，重命名为 `.migrated`
- `settings.json` → 通过 `migrate_disk_to_db()` 迁移到数据库

---

## Settings 结构

**模块：** `settings.rs`

```rust
pub struct Settings {
    // 元
    pub onboard_completed: bool,

    // 步骤 1：数据库
    pub database_backend: Option<String>,    // "postgres" | "libsql"
    pub database_url: Option<String>,
    pub libsql_path: Option<String>,
    pub libsql_url: Option<String>,

    // 步骤 2：安全性
    pub secrets_master_key_source: KeySource, // Keychain | Env | None

    // 步骤 3：推理
    pub llm_backend: Option<String>,         // "nearai" | "anthropic" | "openai" | "ollama" | "openai_compatible"
    pub ollama_base_url: Option<String>,
    pub openai_compatible_base_url: Option<String>,

    // 步骤 4：模型
    pub selected_model: Option<String>,

    // 步骤 5：嵌入
    pub embeddings: EmbeddingsSettings,      // enabled, provider, model

    // 步骤 6：通道
    pub tunnel: TunnelSettings,              // provider, public_url
    pub channels: ChannelSettings,           // http 配置、telegram 所有者等

    // 步骤 7：心跳
    pub heartbeat: HeartbeatSettings,        // enabled, interval, notify

    // 高级（不在向导中，通过 `ironclaw config set` 设置）
    pub agent: AgentSettings,
    pub wasm: WasmSettings,
    pub sandbox: SandboxSettings,
    pub safety: SafetySettings,
    pub builder: BuilderSettings,
}
```

**KeySource 枚举：** `Keychain | Env | None`

---

## 秘密流程

### SecretsContext

设置时秘密操作的薄包装器：

```rust
pub struct SecretsContext {
    store: Arc<dyn SecretsStore>,
    user_id: String,
}
```

由 `init_secrets_context()` 创建，它：
1. 从 `self.secrets_crypto` 或从钥匙串/env 加载 `SecretsCrypto`
2. 创建适当的后端存储：
   - 如果两个功能都已编译：尊重 `self.settings.database_backend`
   - 首先尝试选定的后端，回退到另一个
3. 返回包装存储的 `SecretsContext`

### 秘密存储

秘密使用主密钥通过 AES-256-GCM 加密，然后存储在数据库 `secrets` 表中。向导写入的秘密如下：

```
telegram_bot_token    → 加密的机器人令牌
telegram_webhook_secret → 加密的 webhook HMAC 秘密
anthropic_api_key     → 加密的 API 密钥
```

---

## 提示工具

**模块：** `prompts.rs`

| 函数 | 描述 |
|----------|-------------|
| `select_one(label, options)` | 编号单选菜单 |
| `select_many(label, options, defaults)` | 复选框多选（原始终端模式） |
| `input(label)` | 单行文本输入 |
| `optional_input(label, hint)` | 可为空的文本输入 |
| `secret_input(label)` | 隐藏输入（每字符显示 `*`），返回 `SecretString` |
| `confirm(label, default)` | `[Y/n]` 或 `[y/N]` 提示 |
| `print_header(text)` | 带下划线的粗体部分标题 |
| `print_step(n, total, text)` | `[1/7] 步骤名称` |
| `print_success(text)` | 绿色 `✓` 前缀（ANSI 颜色），消息用默认颜色 |
| `print_error(text)` | 红色 `✗` 前缀（ANSI 颜色），消息用默认颜色 |
| `print_info(text)` | 蓝色 `ℹ` 前缀（ANSI 颜色），消息用默认颜色 |

`select_many` 使用 `crossterm` 原始模式进行箭头键导航。必须在所有退出路径上正确恢复终端状态。

---

## 平台注意事项

### macOS 钥匙串

- `get_generic_password()` 触发系统对话框（解锁 + 授权）
- 每次调用两个对话框是正常的，不是 bug
- 首次访问后缓存结果以避免重复提示
- 永远不要在只读命令中探测钥匙串（`status`、`--help`）
- 服务名称：`"ironclaw"`，帐户：`"master_key"`

### Linux Secret Service

- 通过 `secret-service` crate 使用 GNOME Keyring 或 KWallet
- 可能需要运行 `gnome-keyring` 守护进程
- 集合解锁可能会提示输入密码

### 远程服务器认证

在远程/VPS 服务器上，NEAR AI 的基于浏览器的 OAuth 流程可能无法工作，因为用户的本地浏览器无法访问 `http://127.0.0.1:9876`。

**解决方案：**

1. **NEAR AI Cloud API 密钥（认证菜单中的选项 4）：** 从 `https://cloud.near.ai` 获取 API 密钥并将其粘贴到终端中。不需要本地监听器。密钥保存到 `~/.ironclaw/.env` 和加密的秘密存储。使用 OpenAI 兼容的 ChatCompletions API 模式。

2. **自定义回调 URL：** 设置 `IRONCLAW_OAUTH_CALLBACK_URL` 为公共可访问的 URL（例如，通过 SSH 隧道或反向代理），该 URL 转发到服务器上的端口 9876：
   ```bash
   export IRONCLAW_OAUTH_CALLBACK_URL=https://myserver.example.com:9876
   ```

`oauth_defaults.rs` 中的 `callback_url()` 函数检查此环境变量并回退到 `http://127.0.0.1:{OAUTH_CALLBACK_PORT}`。

### URL 密码

- `#` 在 URL 编码密码中很常见（`%23` 解码）
- `.env` 值必须双引号以保留 `#`
- 显示被屏蔽：`postgres://user:****@host/db`

### Telegram API

- 机器人令牌格式：`123456:ABC-DEF...`
- 令牌进入 URL 路径：`https://api.telegram.org/bot{TOKEN}/method`
- Webhook 秘密标头：`X-Telegram-Bot-Api-Secret-Token`
- 所有者绑定轮询 `getUpdates`（必须先删除 webhook）

---

## 测试

测试位于每个文件底部的 `mod tests {}` 中。

**修改设置时测试的内容：**

- 设置往返：`to_db_map()` 然后 `from_db_map()` 保留值
- 引导 `.env`：dotenvy 可以解析 `save_bootstrap_env()` 写入的内容
- 模型获取器：当 API 不可达时静态回退有效
- 通道发现：处理缺失目录、无效 JSON、去重
- 提示函数：未测试（交互式 I/O），但确保错误路径不会 panic

**运行设置测试：**
```bash
cargo test --lib -- setup
cargo test --lib -- bootstrap
```

---

## 修改清单

更改入门流程时：

1. 首先使用预期行为更改更新此 README
2. 如果添加新的向导步骤：
   - 在 `run()` 中添加到步骤枚举，调整 `total_steps`
   - 将相应的设置字段添加到 `Settings`
   - 添加 `to_db_map` / `from_db_map` 序列化
   - 如果设置在 DB 连接之前需要，添加到 `save_bootstrap_env()`
3. 如果添加新的提供商或通道：
   - 在适当步骤的选择菜单中添加
   - 添加认证流程（API 密钥或 OAuth）
   - 添加带有静态回退 + 5s 超时的模型获取器
4. 如果接触钥匙串：
   - 缓存结果，永远不要调用 `get_master_key()` 两次
   - 在 macOS 上测试（对话框行为与 Linux 不同）
5. 如果接触秘密：
   - 确保 `init_secrets_context()` 尊重选定的数据库后端
   - 同时测试 postgres 和 libsql 功能
6. 运行完整的发布清单：
   ```bash
   cargo fmt
   cargo clippy --all --benches --tests --examples --all-features -- -D warnings
   cargo test --lib -- setup bootstrap
   ```
7. 测试全新的入门：`rm -rf ~/.ironclaw && cargo run`
