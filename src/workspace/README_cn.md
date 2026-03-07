# 工作空间与记忆系统

受 [OpenClaw](https://github.com/openclaw/openclaw) 启发，工作空间为代理提供持久化记忆，具有灵活的类文件系统结构。

## 关键原则

1. **"记忆是数据库，而不是 RAM"** - 如果您想记住某些内容，请明确写入
2. **灵活的结构** - 创建您需要的任何目录/文件层次结构
3. **自我记录** - 使用 README.md 文件描述目录结构
4. **混合搜索** - 通过倒数排名融合结合 FTS（关键词）+ 向量（语义）

## 文件系统结构

```
workspace/
├── README.md              <- 根运行手册/索引
├── MEMORY.md              <- 长期策划的记忆
├── HEARTBEAT.md           <- 周期性检查清单
├── IDENTITY.md            <- 代理名称、性质、氛围
├── SOUL.md                <- 核心价值观
├── AGENTS.md              <- 行为指令
├── USER.md                <- 用户上下文
├── TOOLS.md               <- 环境特定工具说明
├── BOOTSTRAP.md           <- 首次运行仪式（入门后删除）
├── context/               <- 身份相关文档
│   ├── vision.md
│   └── priorities.md
├── daily/                 <- 每日日志
│   ├── 2024-01-15.md
│   └── 2024-01-16.md
├── projects/              <- 任意结构
│   └── alpha/
│       ├── README.md
│       └── notes.md
└── ...
```

## 使用工作空间

```rust
use crate::workspace::{Workspace, OpenAiEmbeddings, paths};

// 为用户创建工作空间
let workspace = Workspace::new("user_123", pool)
    .with_embeddings(Arc::new(OpenAiEmbeddings::new(api_key)));

// 读取/写入任何路径
let doc = workspace.read("projects/alpha/notes.md").await?;
workspace.write("context/priorities.md", "# 优先级\n\n1. 功能 X").await?;
workspace.append("daily/2024-01-15.md", "完成任务 X").await?;

// 众所周知的文件的便利方法
workspace.append_memory("用户偏好暗黑模式").await?;
workspace.append_daily_log("会话说明").await?;

// 列出目录内容
let entries = workspace.list("projects/").await?;

// 搜索（混合 FTS + 向量）
let results = workspace.search("暗黑模式偏好", 5).await?;

// 从身份文件获取系统提示
let prompt = workspace.system_prompt().await?;
```

## 记忆工具

四个供 LLM 使用的工具：

- **`memory_search`** - 混合搜索，必须在回答有关先前工作的问题之前调用
- **`memory_write`** - 写入任何路径（记忆、daily_log 或自定义路径）
- **`memory_read`** - 按路径读取任何文件
- **`memory_tree`** - 将工作空间结构视为树（深度参数，默认 1）

## 混合搜索（RRF）

使用倒数排名融合结合全文搜索和向量相似度：

```
score(d) = Σ 1/(k + rank(d)) for each method where d appears
```

默认 k=60。来自两种方法的结果被合并，在两种方法中出现的文档获得提升的分数。

**后端差异：**
- **PostgreSQL：** `ts_rank_cd` 用于 FTS，pgvector 余弦距离用于向量，完整 RRF
- **libSQL：** FTS5 仅用于关键词搜索（通过 `libsql_vector_idx` 的向量搜索尚未连接）

## 心跳系统

主动周期性执行（默认：30 分钟）：

1. 读取 `HEARTBEAT.md` 检查清单
2. 使用检查清单提示运行代理轮次
3. 如果发现发现，通过通道通知
4. 如果无内容，代理回复 "HEARTBEAT_OK"（无通知）

```rust
use crate::agent::{HeartbeatConfig, spawn_heartbeat};

let config = HeartbeatConfig::default()
    .with_interval(Duration::from_secs(60 * 30))
    .with_notify("user_123", "telegram");

spawn_heartbeat(config, workspace, llm, response_tx);
```

## 分块策略

文档被分块以进行搜索索引：
- 默认：每块 800 个单词（英语大约 800 个令牌）
- 块之间 15% 重叠以保留上下文
- 最小块大小：50 个单词（微小的尾随块与前一个合并）
