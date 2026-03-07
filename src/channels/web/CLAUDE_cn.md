# Web 网关模块

面向浏览器的 HTTP API 和 SSE/WebSocket 实时流式传输。基于 Axum，单用户配备 bearer 令牌认证。

## 文件映射

| 文件 | 角色 |
|------|------|
| `mod.rs` | 网关构建器、启动、`WebChannel` 实现、`with_*` 构建器方法 |
| `server.rs` | `GatewayState`、`start_server()`、所有 Axum 路由注册、内联处理程序 |
| `types.rs` | 请求/响应 DTO 和 `SseEvent` 枚举（SSE 协定的真实来源） |
| `sse.rs` | `SseManager` — 将 `SseEvent` 扇出到所有连接的 SSE 客户端的广播通道 |
| `ws.rs` | WebSocket 处理程序（`handle_ws_connection`）+ `WsConnectionTracker` |
| `auth.rs` | Bearer 令牌中间件（`Authorization: Bearer <GATEWAY_AUTH_TOKEN>`） |
| `log_layer.rs` | 将日志行输出到 `/api/logs/events` SSE 流的追踪层 |
| `handlers/` | 按域拆分的处理程序函数：`chat`、`extensions`、`jobs`、`memory`、`routines`、`settings`、`skills`、`static_files` |
| `openai_compat.rs` | OpenAI 兼容代理（`/v1/chat/completions`、`/v1/models`） |
| `util.rs` | 共享助手（`build_turns_from_db_messages`、`truncate_preview`） |
| `static/` | 单页应用（HTML/CSS/JS）— 编译时通过 `include_str!`/`include_bytes!` 嵌入 |

## API 路由

### 公共（无需认证）
| 方法 | 路径 | 描述 |
|------|------|-------------|
| GET | `/api/health` | 健康检查 |
| GET | `/oauth/callback` | 扩展认证的 OAuth 回调 |

### 聊天
| 方法 | 路径 | 描述 |
|------|------|-------------|
| POST | `/api/chat/send` | 发送消息 → 排队到代理循环 |
| GET | `/api/chat/events` | 代理事件的 SSE 流 |
| GET | `/api/chat/ws` | WebSocket 替代 SSE |
| GET | `/api/chat/history` | 线程的分页轮次历史 |
| GET | `/api/chat/threads` | 列出线程（返回 `assistant_thread` + 常规线程） |
| POST | `/api/chat/thread/new` | 创建新线程 |
| POST | `/api/chat/approval` | 批准/拒绝/始终批准待处理的工具调用 |
| POST | `/api/chat/auth-token` | 提交扩展的认证令牌 |
| POST | `/api/chat/auth-cancel` | 取消待处理的认证流程 |

### 记忆
| 方法 | 路径 | 描述 |
|------|------|-------------|
| GET | `/api/memory/tree` | 工作空间目录树 |
| GET | `/api/memory/list` | 列出路径的文件 |
| GET | `/api/memory/read` | 读取工作空间文件 |
| POST | `/api/memory/write` | 写入工作空间文件 |
| POST | `/api/memory/search` | 混合 FTS + 向量搜索 |

### 任务（沙箱）
| 方法 | 路径 | 描述 |
|------|------|-------------|
| GET | `/api/jobs` | 列出沙箱任务 |
| GET | `/api/jobs/summary` | 聚合统计 |
| GET | `/api/jobs/{id}` | 任务详情 |
| POST | `/api/jobs/{id}/cancel` | 取消运行中的任务 |
| POST | `/api/jobs/{id}/restart` | 重启失败的任务 |
| POST | `/api/jobs/{id}/prompt` | 向 Claude Code 网桥发送后续提示 |
| GET | `/api/jobs/{id}/events` | 特定任务的 SSE 流 |
| GET | `/api/jobs/{id}/files/list` | 列出任务工作空间的文件 |
| GET | `/api/jobs/{id}/files/read` | 从任务工作空间读取文件 |

### 技能
| 方法 | 路径 | 描述 |
|------|------|-------------|
| GET | `/api/skills` | 列出已安装的技能 |
| POST | `/api/skills/search` | 搜索 ClawHub 注册表 + 本地技能 |
| POST | `/api/skills/install` | 从 ClawHub 或通过 URL/内容安装技能 |
| DELETE | `/api/skills/{name}` | 移除已安装的技能 |

### 扩展
| 方法 | 路径 | 描述 |
|------|------|-------------|
| GET | `/api/extensions` | 已安装的扩展 |
| GET | `/api/extensions/tools` | 所有已注册的工具（来自工具注册表） |
| POST | `/api/extensions/install` | 安装扩展 |
| GET | `/api/extensions/registry` | 来自注册表清单的可用扩展 |
| POST | `/api/extensions/{name}/activate` | 激活已安装的扩展 |
| POST | `/api/extensions/{name}/remove` | 移除扩展 |
| GET/POST | `/api/extensions/{name}/setup` | 扩展设置向导 |

### 例行任务
| 方法 | 路径 | 描述 |
|------|------|-------------|
| GET | `/api/routines` | 列出例行任务 |
| GET | `/api/routines/summary` | 聚合统计（总计/已启用/已禁用/失败/今日运行） |
| GET | `/api/routines/{id}` | 例行任务详情及最近运行历史 |
| POST | `/api/routines/{id}/trigger` | 手动触发例行任务 |
| POST | `/api/routines/{id}/toggle` | 启用/禁用例行任务 |
| DELETE | `/api/routines/{id}` | 删除例行任务 |
| GET | `/api/routines/{id}/runs` | 列出特定例行任务的运行 |

### 设置
| 方法 | 路径 | 描述 |
|------|------|-------------|
| GET | `/api/settings` | 列出所有设置 |
| GET | `/api/settings/export` | 将所有设置导出为映射 |
| POST | `/api/settings/import` | 从映射批量导入设置 |
| GET | `/api/settings/{key}` | 获取单个设置 |
| PUT | `/api/settings/{key}` | 设置单个设置 |
| DELETE | `/api/settings/{key}` | 删除设置 |

### 其他
| 方法 | 路径 | 描述 |
|------|------|-------------|
| GET | `/api/logs/events` | 实时日志流（SSE） |
| GET/PUT | `/api/logs/level` | 运行时获取/设置日志级别 |
| GET | `/api/pairing/{channel}` | 列出待处理的配对请求 |
| POST | `/api/pairing/{channel}/approve` | 批准配对请求 |
| GET | `/api/gateway/status` | 服务器正常运行时间、连接的客户端、配置 |
| POST | `/v1/chat/completions` | OpenAI 兼容的 LLM 代理 |
| GET | `/v1/models` | OpenAI 兼容的模型列表 |

### 静态 / 项目文件
| 方法 | 路径 | 描述 |
|------|------|-------------|
| GET | `/` | 单页应用 HTML |
| GET | `/style.css` | 应用样式表 |
| GET | `/app.js` | 应用 JavaScript |
| GET | `/favicon.ico` | 收藏夹图标（缓存 1 天） |
| GET | `/projects/{project_id}/` | 任务工作空间浏览器（重定向） |
| GET | `/projects/{project_id}/{*path}` | 从任务工作空间提供文件（需要认证） |

## SSE 事件类型（`types.rs` 中的 `SseEvent`）

SSE 协定 — 每个字段都是 `#[serde(tag = "type")]`：

| 类型 | 发出时间 |
|------|-------------|
| `response` | 来自代理的最终文本响应 |
| `stream_chunk` | 流式令牌（部分响应） |
| `thinking` | 推理期间的代理状态更新 |
| `tool_started` | 工具调用开始 |
| `tool_completed` | 工具调用完成（包括成功/错误） |
| `tool_result` | 工具输出预览 |
| `status` | 通用状态消息 |
| `job_started` | 沙箱任务创建 |
| `job_message` | 来自沙箱工作器的消息 |
| `job_tool_use` | 沙箱内调用的工具 |
| `job_tool_result` | 来自沙箱的工具结果 |
| `job_status` | 沙箱任务状态更新 |
| `job_result` | 沙箱任务的最终结果 |
| `approval_needed` | 工具需要用户批准（暂停代理） |
| `auth_required` | 扩展需要认证凭证 |
| `auth_completed` | 扩展认证流程完成 |
| `extension_status` | WASM 通道激活状态已更改 |
| `error` | 来自代理或网关的错误 |
| `heartbeat` | SSE 保活（空负载） |

**SSE 序列化：** 事件使用 `#[serde(tag = "type")]` — 线格式是 `{"type":"<variant>", ...fields}`。SSE 帧的 `event:` 字段设置为与 `type` 相同的字符串，以便在浏览器中轻松使用 `addEventListener`。

**WebSocket 信封：** 通过 WebSocket，SSE 事件被包装为 `{"type":"event","event_type":"<variant>","data":{...}}`。Ping/pong 使用 `{"type":"ping"}` / `{"type":"pong"}`。客户端到服务器的消息（`message`、`approval`、`auth_token`、`auth_cancel`）在 `types.rs` 的 `WsClientMessage` 中定义。

**要添加新的 SSE 事件：** 使用 `add-sse-event` 技能（`/add-sse-event`）。它搭建 Rust 变体、序列化、广播调用和前端处理程序。还要在 `types.rs` 的 `WsServerMessage::from_sse_event()` 中添加匹配的 arm。

## 认证

所有受保护的路由都需要 `Authorization: Bearer <GATEWAY_AUTH_TOKEN>`。令牌通过 `GATEWAY_AUTH_TOKEN` 环境变量设置。缺少/错误的令牌 → 401。`Bearer` 前缀不区分大小写比较（RFC 6750）。

**查询字符串令牌认证（`?token=xxx`）：** 因为 `EventSource` 和 WebSocket 升级无法从浏览器设置自定义标头，三个端点也接受令牌作为 URL 查询参数：`/api/chat/events`、`/api/logs/events` 和 `/api/chat/ws`。所有其他端点拒绝查询字符串令牌。如果添加新的 SSE 或 WebSocket 端点，请在 `auth.rs` 的 `allows_query_token_auth()` 中注册其路径。

**如果未配置 `GATEWAY_AUTH_TOKEN`**，则在启动时生成一个随机的 32 个字符的字母数字令牌并打印到控制台。

速率限制：聊天发送端点上限为**每 60 秒 30 条消息**（滑动窗口，而非每 IP）。

## GatewayState

共享状态结构（`server.rs`）持有对所有子系统的引用。字段是 `Option<Arc<T>>`，因此即使禁用了可选子系统（工作空间、沙箱、技能），网关也能启动。在使用之前始终进行空值检查。

关键字段：
- `msg_tx` — `RwLock<Option<mpsc::Sender<IncomingMessage>>>` — 向代理循环发送消息；在调用 `Channel` 上的 `start()` 时设置
- `sse` — `SseManager` — 广播中心；从任何处理程序调用 `state.sse.broadcast(event)`
- `ws_tracker` — `Option<Arc<WsConnectionTracker>>` — 单独跟踪 WS 连接计数与 SSE
- `chat_rate_limiter` — `RateLimiter` — 所有聊天发送调用者共享的 30 req/60 s 滑动窗口
- `scheduler` — `Option<SchedulerSlot>` — 用于将后续消息注入到运行中的代理任务
- `cost_guard` — `Option<Arc<CostGuard>>` — 在状态端点中暴露令牌使用/成本总计
- `startup_time` — `Instant` — 用于在网关状态响应中计算正常运行时间
- `registry_entries` — `Vec<RegistryEntry>` — 启动时从注册表清单加载一次；由可用扩展 API 使用，而无需访问网络

子系统通过 `GatewayChannel`（`mod.rs`）上的 `with_*` 构建器方法连接。每次调用都会重建 `Arc<GatewayState>` — 在 `start()` 之前调用是安全的，而不是之后。

## SSE / WebSocket 连接限制

SSE 和 WebSocket 共享同一个 `SseManager` 广播通道。主要特征：

- **广播缓冲区：** 256 个事件。落后太多的客户端将错过事件 — `BroadcastStream` 会静默删除滞后的活动。SSE 客户端应该重新连接并重新获取历史。
- **最大连接数：** 总共 100 个（SSE + WebSocket 组合）。超出限制的连接将收到 503 / 被立即删除。
- **SSE 保活：** Axum 的 `KeepAlive` 每**30 秒**发送一个空事件以防止代理超时。
- **WebSocket：** 每个连接两个任务 — 一个发送器任务（广播 → WS 帧）和一个接收器循环（WS 帧 → 代理）。当客户端断开连接时，发送器被中止，并且 SSE 连接计数器和 WS 跟踪器计数器都递减。

## CORS 和安全标头

CORS 限制为网关自己的源（同一 IP+端口和 `localhost`+端口）。允许的方法：GET、POST、PUT、DELETE。允许的标头：`Content-Type`、`Authorization`。允许凭证。

所有响应都包括：
- `X-Content-Type-Options: nosniff`
- `X-Frame-Options: DENY`

**请求正文限制：** 1 MB（`DefaultBodyLimit::max(1024 * 1024)`）。较大的负载返回 413。

## 待批准

工具批准状态**仅限内存**（未持久化到数据库）。服务器重启清除所有待批准。`HistoryResponse` 中的 `pending_approval` 字段在从内存状态线程切换时重新填充。

## 添加新的 API 端点

1. 在 `types.rs` 中定义请求/响应类型
2. 在适当的 `handlers/*.rs` 文件中实现处理程序（或在 `server.rs` 中内联用于简单处理程序）
3. 在 `server.rs` 的 `start_server()` 中在正确的路由器（`public`、`protected` 或 `statics`）下注册路由
4. 如果是 SSE 或 WebSocket 端点，将其路径添加到 `auth.rs` 的 `allows_query_token_auth()`
5. 如果需要新的 `GatewayState` 字段，请将其添加到结构以及 `GatewayChannel::new()` 初始值设定项和 `mod.rs` 中的 `rebuild_state()`，然后添加 `with_*` 构建器方法
