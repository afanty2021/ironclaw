# 代理模块

核心代理逻辑。这是最复杂的子系统 — 在 `src/agent/` 中工作之前请阅读此内容。

## 模块映射

| 文件 | 角色 |
|------|------|
| `agent_loop.rs` | `Agent` 结构，`AgentDeps`，主 `run()` 事件循环。委托给同级模块。 |
| `dispatcher.rs` | 对话轮次的代理循环：LLM 调用 → 工具执行 → 重复。注入技能上下文。返回 `Response` 或 `NeedApproval`。 |
| `thread_ops.rs` | 线程/会话操作：`process_user_input`、undo/redo、批准、认证模式拦截、数据库水合、压缩。 |
| `commands.rs` | 系统命令处理器（`/help`、`/model`、`/status`、`/skills` 等）和任务意图处理器。 |
| `session.rs` | 数据模型：`Session` → `Thread` → `Turn`。线程和轮次的状态机。 |
| `session_manager.rs` | 生命周期：创建/查找会话、将外部线程 ID 映射到内部 UUID、修剪陈旧会话、管理 undo 管理器。 |
| `router.rs` | 将显式 `/commands` 路由到 `MessageIntent`。自然语言完全绕过路由器。 |
| `scheduler.rs` | 并行任务调度。维护 `jobs` 映射（完整 LLM 驱动）和 `subtasks` 映射（工具执行/后台）。 |
| `worker.rs` | 后台调度器任务的每任务执行：调用 LLM、运行工具、处理推理循环。与 `dispatcher.rs` 不同。 |
| `compaction.rs` | 上下文窗口管理：总结旧轮次、写入工作空间每日日志、修剪上下文。三种策略。 |
| `context_monitor.rs` | 检测内存压力。根据使用级别建议 `CompactionStrategy`。 |
| `self_repair.rs` | 检测卡死的任务和损坏的工具，尝试恢复。 |
| `heartbeat.rs` | 主动周期性执行。读取 `HEARTBEAT.md`，如果发现发现则通过通道通知。 |
| `submission.rs` | 在路由之前将所有用户提交解析为类型化变体。 |
| `undo.rs` | 基于检查点的轮次 undo/redo。检查点存储消息列表（默认最多 20 个）。 |
| `routine.rs` | `Routine` 类型：`Trigger`（cron/事件/webhook/手动）+ `RoutineAction`（轻量级/full_job）+ `RoutineGuardrails`。 |
| `routine_engine.rs` | Cron 计时器和事件匹配器。在触发器匹配时触发例程。轻量级内联运行；full_job 分派到 `Scheduler`。 |
| `task.rs` | 调度器的任务类型：`Job`、`ToolExec`、`Background`。由 `spawn_subtask` 和 `spawn_batch` 使用。 |
| `cost_guard.rs` | LLM 支出和操作速率执行。跟踪每日预算（美分）和每小时调用速率。位于 `AgentDeps` 中。 |
| `job_monitor.rs` | 订阅 SSE 广播，将 Claude Code（容器）输出作为 `IncomingMessage` 注入回代理循环。 |

## 会话 / 线程 / 轮次模型

```
Session（每用户）
└── Thread（每次对话 — 可以有多个）
    └── Turn（每对请求/响应）
        ├── user_input: String
        ├── response: Option<String>
        ├── tool_calls: Vec<ToolCall>
        └── state: TurnState（Pending | Running | Complete | Failed）
```

- 一个会话一次有一个**活动线程**；可以切换线程。
- 轮次是仅追加的。Undo 通过恢复先前的检查点（消息列表，而不是完整的线程快照）来回滚。
- `UndoManager` 是每个线程的，存储在 `SessionManager` 中，而不是在 `Session` 本身上。最多 20 个检查点（超过时丢弃最旧的）。
- 群聊检测：如果 `metadata.chat_type` 是 `group`/`channel`/`supergroup`，`MEMORY.md` 从系统提示中排除，以防止泄露个人上下文。
- **认证模式**：如果线程设置了 `pending_auth`（例如，从返回 `awaiting_token` 的 `tool_auth`），下一条用户消息在任何轮次创建、日志记录或安全验证之前被拦截，并直接发送到凭证存储。任何控制提交（undo、interrupt 等）取消认证模式。
- `ThreadState` 值：`Idle`、`Processing`、`AwaitingApproval`、`Completed`、`Interrupted`。
- `SessionManager` 将 `(user_id, channel, external_thread_id)` 映射到内部 UUID。每 10 分钟修剪一次空闲会话（在 1000 个会话时警告）。

## 代理循环（dispatcher.rs）

`dispatcher.rs` 模块处理**直接对话轮次**（由主代理内联处理的用户消息）。后台调度器任务使用 `worker.rs` — 这是两个单独的执行路径。

```
run_agentic_loop()  [dispatcher.rs — 对话轮次]
  1. 加载工作空间系统提示（身份文件：AGENTS.md、SOUL.md 等）
  2. 从元数据检测群聊；如果是群聊则排除 MEMORY.md
  3. 选择活动技能（针对消息内容的关键词/模式评分）
  4. 构建技能上下文块（在用户消息之前注入）
  5. LLM 调用 → 文本响应 或 工具调用
  6. 如果有工具调用：
     a. 检查工具批准（会话自动批准、待批准队列）
     b. 执行工具（通过 JoinSet 并行）
     c. 通过 SafetyLayer 清理结果
     d. 将反馈输送到 → 转到 5
  7. 返回 AgenticLoopResult::Response 或 NeedApproval
```

**工具批准：** 标记为 `requires_approval` 的工具暂停循环并返回 `NeedApproval`。Web 网关将 `PendingApproval` 存储在会话状态中并发送 `approval_needed` SSE 事件。用户的批准/拒绝恢复循环。

**worker.rs 与 dispatcher.rs：** `dispatcher.rs` 为用户发起的对话轮次运行代理循环（保持会话锁、跟踪轮次）。`worker.rs` 由 `Scheduler` 为通过 `CreateJob` / `/job` 创建的后台任务生成 — 它独立于会话运行，并有自己的 LLM 推理循环，支持规划（`use_planning` 标志）。

## 命令路由（router.rs）

`Router` 处理显式 `/commands`（前缀 `/`）。它将它们解析为 `MessageIntent` 变体：`CreateJob`、`CheckJobStatus`、`CancelJob`、`ListJobs`、`HelpJob`、`Command`。自然语言消息完全绕过路由器 — 它们通过 `process_user_input` 直接进入 `dispatcher.rs`。注意：大多数面向用户的命令（undo、compact 等）由 `SubmissionParser` 在路由器运行之前处理，因此 `Router` 只看到未被 `submission.rs` 声明的未识别的 `/xxx` 模式。

## 压缩

由 `ContextMonitor` 在令牌使用接近模型的上下文限制时触发。

**令牌估算：** 字数 × 1.3 + 每条消息 4 个开销。默认上下文限制：100,000 个令牌。压缩阈值：80%（可配置）。

三种策略，由 `ContextMonitor.suggest_compaction()` 根据使用率选择：
- **MoveToWorkspace** — 将完整轮次记录写入工作空间每日日志，保留最近 10 个轮次。当使用率为 80-85% 时使用（中等）。如果没有工作空间，则回退到 `Truncate(5)`。
- **Summarize**（`keep_recent: N`）— LLM 生成旧轮次的摘要，将其写入工作空间每日日志（`daily/YYYY-MM-DD.md`），删除旧轮次。当使用率为 85-95% 时使用。
- **Truncate**（`keep_recent: N`）— 删除最旧的轮次而不进行摘要（快速路径）。当使用率 >95% 时使用（严重）。

如果用于摘要的 LLM 调用失败，错误会传播 — 轮次在失败时**不会**被截断。

手动触发：用户发送 `/compact`（由 `submission.rs` 解析）。

## 调度器

`Scheduler` 在 `Arc<RwLock<HashMap>>` 下维护两个映射：
- `jobs` — 完整的 LLM 驱动任务，每个都有 `Worker` 和一个用于 `WorkerMessage`（`Start`、`Stop`、`Ping`、`UserMessage`）的 `mpsc` 通道
- `subtasks` — 通过 `spawn_subtask()` / `spawn_batch()` 生成的轻量级 `ToolExec` 或 `Background` 任务

**首选入口点：** `dispatch_job()` — 创建上下文、可选设置元数据、持久化到数据库（以便来自 `job_actions`/`llm_calls` 的 FK 引用立即有效），然后调用 `schedule()`。除非您已经持久化，否则不要直接调用 `schedule()`。

检查插入在单个写锁下完成，以防止 TOCTOU 竞争。清理任务每秒轮询一次任务完成，并从映射中删除该条目。

`spawn_subtask()` 返回一个 `oneshot::Receiver` — 调用者必须等待它才能获得结果。`spawn_batch()` 并发运行所有任务并按输入顺序返回结果。

## 自修复

`DefaultSelfRepair` 在 `repair_check_interval`（来自 `AgentConfig`）上运行。它：
1. 调用 `ContextManager::find_stuck_jobs()` 查找处于 `JobState::Stuck` 的任务
2. 尝试 `ctx.attempt_recovery()`（转换回 `InProgress`）
3. 如果 `repair_attempts >= max_repair_attempts`，返回 `ManualRequired`
4. 通过 `store.get_broken_tools(5)` 检测损坏的工具（阈值：5 次失败）。需要调用 `with_store()`；否则返回空。
5. 尝试通过 `SoftwareBuilder` 重建损坏的工具。需要调用 `with_builder()`；否则返回 `ManualRequired`。

注意：`stuck_threshold` 持续时间已存储但当前未使用（标记为 `#[allow(dead_code)]`）。卡住检测依赖于由状态机设置的 `JobState::Stuck`，而不是挂钟时间比较。

修复结果：`Success`、`Retry`、`Failed`、`ManualRequired`。`Retry` **不会**通知用户（以避免垃圾邮件）。

## 关键不变量

- 永远不要调用 `.unwrap()` 或 `.expect()` — 使用带有适当错误映射的 `?`
- `Session`/`Thread` 上的所有状态突变都在 `Arc<Mutex<Session>>` 锁下发生
- 代理循环每个线程是单线程的；并行执行发生在任务/调度器级别
- 技能是**确定性**选择的（无 LLM 调用）— 参见 `skills/selector.rs`
- 工具结果在返回 LLM 之前通过 `SafetyLayer`（清理器 → 验证器 → 策略 → 泄露检测器）
- `SessionManager` 使用双重检查锁定进行会话创建。首先读锁（快速路径），然后写锁重新检查以防止重复会话
- `Scheduler.schedule()` 在整个检查插入序列中保持写锁 — 调用时不要持有任何其他锁
- `AgentDeps` 中的 `cheap_llm` 用于心跳和其他轻量级任务。如果 `None`，则回退到主 `llm`。使用 `agent.cheap_llm()` 访问器，而不是直接使用 `deps.cheap_llm`
- 必须在 LLM 调用**之前**调用 `CostGuard.check_allowed()`；必须**在之后**调用 `record_llm_call()`。两个调用是分开的 — 守卫不会自动记录
- `BeforeInbound` 和 `BeforeOutbound` 钩子分别为每条用户消息和代理响应运行。钩子可以修改内容或拒绝。钩子错误被记录但**失败开放**（处理继续）
- `SessionManager` 使用双重检查锁定进行会话创建。首先读锁（快速路径），然后写锁重新检查以防止重复会话
- `Scheduler.schedule()` 在整个检查插入序列中保持写锁 — 调用时不要持有任何其他锁

## 完整提交命令参考

由 `SubmissionParser::parse()` 解析的所有命令：

| 输入 | 变体 | 注意 |
|-------|---------|-------|
| `/undo` | `Undo` | |
| `/redo` | `Redo` | |
| `/interrupt`、`/stop` | `Interrupt` | |
| `/compact` | `Compact` | |
| `/clear` | `Clear` | |
| `/heartbeat` | `Heartbeat` | |
| `/summarize`、`/summary` | `Summarize` | |
| `/suggest` | `Suggest` | |
| `/new`、`/thread new` | `NewThread` | |
| `/thread <uuid>` | `SwitchThread` | 必须是有效的 UUID |
| `/resume <uuid>` | `Resume` | 必须是有效的 UUID |
| `/status [id]`、`/progress [id]`、`/list` | `JobStatus` | `/list` = 所有任务 |
| `/cancel <id>` | `JobCancel` | |
| `/quit`、`/exit`、`/shutdown` | `Quit` | |
| `yes/y/approve/ok` 及别名 | `ApprovalResponse { approved: true, always: false }` | |
| `always/a` 及别名 | `ApprovalResponse { approved: true, always: true }` | |
| `no/n/deny/reject/cancel` 及别名 | `ApprovalResponse { approved: false }` | |
| JSON `ExecApproval{...}` | `ExecApproval` | 来自 Web 网关批准端点 |
| `/help`、`/?` | `SystemCommand { "help" }` | 绕过线程状态检查 |
| `/version` | `SystemCommand { "version" }` | |
| `/tools` | `SystemCommand { "tools" }` | |
| `/skills [search <q>]` | `SystemCommand { "skills" }` | |
| `/ping` | `SystemCommand { "ping" }` | |
| `/debug` | `SystemCommand { "debug" }` | |
| `/model [name]` | `SystemCommand { "model" }` | |
| 其他所有 | `UserInput` | 开始新的代理轮次 |

**`SystemCommand` 与 control：** `SystemCommand` 变体完全绕过线程状态检查（无会话锁、无轮次创建）。`Quit` 从 `handle_message` 返回 `Ok(None)`，这会中断主循环。

## 添加新提交命令

提交是在代理循环运行之前在 `submission.rs` 中解析的特殊消息。要添加新的：
1. 在 `submission.rs` 中添加变体到 `Submission` 枚举
2. 在 `SubmissionParser::parse()` 中添加解析
3. 在 `agent_loop.rs` 中匹配 `SubmissionResult` 的位置处理（`handle_message` 中的 `match submission { ... }` 块）
4. 实现处理程序方法（通常在 `thread_ops.rs` 中用于会话操作，或在 `commands.rs` 中用于系统命令）
