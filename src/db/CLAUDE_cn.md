# 数据库模块

双后端持久化层。**所有新的持久化功能必须支持两个后端。**

## 快速参考

```bash
# 默认构建（PostgreSQL）
cargo build

# libSQL/Turso 构建
cargo build --no-default-features --features libsql

# 两个后端
cargo build --features "postgres,libsql"

# 单独测试每个后端
cargo check                                           # postgres（默认）
cargo check --no-default-features --features libsql   # 仅 libsql
cargo check --all-features                            # 两者
```

## 文件

| 文件 | 角色 |
|------|------|
| `mod.rs` | `Database` 超级 trait + 7 个子 trait（约 78 个异步方法）— 首先在此添加新操作 |
| `postgres.rs` | PostgreSQL 后端 — 委托给 `history/` 中的 `Store` + `Repository` |
| `libsql/mod.rs` | libSQL/Turso 后端结构、连接助手、行解析工具 |
| `libsql/conversations.rs` | `ConversationStore` 实现 |
| `libsql/jobs.rs` | `JobStore` 实现 |
| `libsql/sandbox.rs` | `SandboxStore` 实现 |
| `libsql/routines.rs` | `RoutineStore` 实现 |
| `libsql/settings.rs` | `SettingsStore` 实现 |
| `libsql/tool_failures.rs` | `ToolFailureStore` 实现 |
| `libsql/workspace.rs` | `WorkspaceStore` 实现（FTS5 + 向量搜索） |
| `libsql_migrations.rs` | 合并的 libSQL 模式（CREATE IF NOT EXISTS，无 ALTER TABLE） |
| `tls.rs` | PostgreSQL 的 TLS 连接器工厂（`rustls` + 系统根证书） |

PostgreSQL 模式：`migrations/V1__initial.sql` 到 `V9__flexible_embedding_dimension.sql`（由 `refinery` 管理）。V1 是基础模式；后续迁移添加表、列，并将 `claude_code_events` 重命名为 `job_events`。

## Trait 结构

`Database` 超级 trait 由七个子 trait 组成。叶消费者可以依赖他们需要的最窄子 trait，而不是完整的 `Database`：

| 子 trait | 方法数 | 覆盖 |
|----------|--------|--------|
| `ConversationStore` | 12 | 对话、消息 |
| `JobStore` | 13 | 代理任务、操作、LLM 调用、估算 |
| `SandboxStore` | 13 | 沙箱任务、任务事件 |
| `RoutineStore` | 15 | 例行任务、例行任务运行 |
| `ToolFailureStore` | 4 | 自修复跟踪 |
| `SettingsStore` | 8 | 每用户键值设置 |
| `WorkspaceStore` | 13 | 记忆文档、块、混合搜索 |

`Database` 添加 `run_migrations()` 并结合所有子 trait。

## 添加新的持久化操作

1. 确定方法属于哪个子 trait，或创建新的子 trait
2. 在 `mod.rs` 中为该子 trait 添加异步方法签名
3. 在 `postgres.rs` 中实现（委托给 `Store` 或 `Repository`）
4. 在 `libsql/<module>.rs` 中实现（SQLite 方言 SQL，每操作使用 `self.connect().await?`）
5. 如需要，添加迁移：
   - PostgreSQL：新的 `migrations/VN__description.sql`
   - libSQL：将 `CREATE TABLE IF NOT EXISTS` 添加到 `libsql_migrations.rs`

## SQL 方言差异

| 功能 | PostgreSQL | libSQL |
|---------|-----------|--------|
| UUID | `UUID` 类型 | `TEXT` |
| 时间戳 | `TIMESTAMPTZ` | `TEXT`（ISO-8601 RFC 3339，毫秒精度） |
| JSON | `JSONB` | `TEXT` |
| 数字/小数 | `NUMERIC` | `TEXT`（保留 `rust_decimal` 精度） |
| 数组 | `TEXT[]` | `TEXT`（JSON 编码数组） |
| 布尔值 | `BOOLEAN` | `INTEGER`（0/1） |
| 向量嵌入 | `VECTOR`（任何维度，V9 移除固定 1536） | 通过 `libsql_vector_idx` 的 `F32_BLOB(1536)` |
| 全文搜索 | `tsvector` + `ts_rank_cd` | FTS5 虚拟表 + 同步触发器 |
| JSON 路径更新 | `jsonb_set(col, '{key}', val)` | `json_patch(col, '{"key": val}')` |
| PL/pgSQL | 函数 | 触发器（SQLite 中没有存储过程） |
| 连接模型 | `deadpool-postgres` 连接池 | 每操作新连接（`self.connect()`） |
| 并发 | 基于池，完全并发 | WAL 模式 + 5 秒繁忙超时；写入序列化 |
| 自动时间戳 | `DEFAULT NOW()` | `DEFAULT (datetime('now'))` |
| 时间戳解析 | 原生类型 | `parse_timestamp()` 中的多格式回退 |

**JSON 合并补丁陷阱：** libSQL 对元数据更新使用 RFC 7396 JSON Merge Patch（`json_patch`）。这会完全替换顶级键 — 它**不能**做部分嵌套更新。PostgreSQL 使用路径定向的 `jsonb_set`。如果您需要 libSQL 兼容性，请不要依赖部分嵌套元数据更新。

**布尔存储：** libSQL 将布尔值存储为整数。读取时，使用 `get_i64(row, idx) != 0`；写入时，传递 `1i64`/`0i64`。永远不要直接传递 Rust `bool`。

**时间戳写入格式：** 始终使用 `fmt_ts(dt)` 写入时间戳（RFC 3339，毫秒精度）。使用 `get_ts()` / `get_opt_ts()` 读取，它们也处理遗留的朴素格式。

**向量维度：** PostgreSQL V9 迁移将列更改为无界 `vector`（移除 HNSW 索引）。libSQL 仍使用 `F32_BLOB(1536)` — 如果您使用不同维度的嵌入模型，libSQL 模式也需要更新。

**每操作连接：** `LibSqlBackend::connect()` 为每个操作创建一个新连接，设置 `PRAGMA busy_timeout = 5000`，并在 `Connection` 被删除时关闭它。这是有意的 — libSQL SDK 不提供池。避免在 `await` 点之间保持连接打开。

## 模式：关键表

**核心：**
- `conversations` — 多通道对话跟踪
- `conversation_messages` — 对话中的单条消息
- `agent_jobs` — 任务元数据和状态
- `job_actions` — 事件源工具执行
- `job_events` — 沙箱任务流式事件（在 V7 中从 `claude_code_events` 重命名）
- `dynamic_tools` — 代理构建的工具
- `llm_calls` — 成本/令牌跟踪
- `estimation_snapshots` — 学习数据
- `repair_attempts` — 自修复操作日志（尚未通过 Database trait 公开）

**工作空间/记忆：**
- `memory_documents` — 灵活的基于路径的文件
- `memory_chunks` — 带有 FTS + 向量索引的分块内容
- `memory_chunks_fts` — FTS5 虚拟表（libSQL）/ `tsvector` 列（PostgreSQL）
- `heartbeat_state` — 周期性执行跟踪

**安全/扩展：**
- `secrets` — AES-256-GCM 加密凭证
- `wasm_tools` — 已安装的 WASM 工具二进制文件
- `tool_capabilities` — 每工具 HTTP 允许列表、秘密访问、速率限制
- `leak_detection_patterns` — 秘密正则模式（两个后端的种子数据）
- `leak_detection_events` — 检测到的泄露的审计日志
- `secret_usage_log` — 每请求凭证注入审计跟踪
- `tool_rate_limit_state` — 滑动窗口速率限制计数器

**其他：**
- `routines`、`routine_runs` — 定时/响应式执行
- `settings` — 每用户键值
- `tool_failures` — 用于自修复的损坏工具跟踪
- `_migrations` — libSQL 仅内部迁移版本跟踪

## libSQL 当前限制

- **秘密存储** — 仍需要 `PostgresSecretsStore`；`LibSqlSecretsStore` 存在但未通过主启动路径
- **设置重新加载** — `Config::from_db` 跳过（需要 `Store`）
- **无增量迁移** — 模式是幂等 CREATE IF NOT EXISTS；无 ALTER TABLE 支持；列添加需要新的版本化方法
- **无静态加密** — 仅秘密（API 令牌）使用 AES-256-GCM 加密；所有其他数据是明文 SQLite
- **混合搜索** — FTS5 和向量搜索（`libsql_vector_idx`）都已实现；但是，向量索引固定为 `F32_BLOB(1536)`，而 PostgreSQL 在 V9 中切换到无界 `vector`
- **写入序列化** — WAL 模式允许多个并发读取器，但一次只能有一个写入器；繁忙超时为 5 秒，这可能会在高写入并发下导致超时

## 本地运行 libSQL

```bash
# 使用本地 SQLite 文件（默认）
DATABASE_BACKEND=libsql LIBSQL_PATH=~/.ironclaw/test.db cargo run

# 使用 Turso 云（嵌入副本将本地文件同步到云）
DATABASE_BACKEND=libsql LIBSQL_URL=libsql://xxx.turso.io LIBSQL_AUTH_TOKEN=xxx cargo run

# 内存中（仅测试 — 进程退出时数据丢失）
# 直接在测试代码中使用 LibSqlBackend::new_memory()
```

## 测试 libSQL 后端

在单元测试中使用 `LibSqlBackend::new_memory()` — 无需文件，无需清理：

```rust
#[tokio::test]
async fn test_my_feature() {
    let backend = LibSqlBackend::new_memory().await.unwrap();
    backend.run_migrations().await.unwrap();
    // backend 实现 Database — 调用任何 trait 方法
}
```

对于需要多个连接共享状态的并发测试，请使用带有 `tempfile::tempdir()` 的 `LibSqlBackend::new_local(&tmp_path)`。内存数据库不在连接之间共享状态。

## 共享 libSQL 数据库句柄

`LibSqlBackend::shared_db()` 返回一个 `Arc<LibSqlDatabase>`，用于传递给卫星存储（例如，`LibSqlSecretsStore`、`LibSqlWasmToolStore`），这些存储需要自己的每操作连接，但应共享同一个底层数据库文件。这些存储自己调用 `.connect()` 共享句柄。这是正确的模式 — 不要将实时 `Connection` 传递给卫星存储。

## 模式：修复模式，而不仅仅是实例

在修复一个后端的 SQL 中的 bug 时，始终 grep 另一个后端中的相同模式。`postgres.rs` 中的修复如果不也修复 libSQL 模块（例如，`libsql/jobs.rs`），那就是一半的修复。对于 `LibSqlSecretsStore` 或 `LibSqlWasmToolStore` 等卫星类型也是如此。
