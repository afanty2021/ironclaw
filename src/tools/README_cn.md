# 工具系统

## 添加新工具

### 内置工具（Rust）

1. 创建 `src/tools/builtin/my_tool.rs`
2. 实现 `Tool` trait
3. 在 `src/tools/builtin/mod.rs` 中添加 `mod my_tool;` 和 `pub use`
4. 在 `registry.rs` 中的 `ToolRegistry::register_builtin_tools()` 中注册
5. 添加测试

### WASM 工具（推荐）

WASM 工具是添加新功能的首选方式。它们在具有明确功能的沙箱环境中运行。

1. 在 `tools-src/<name>/` 中创建新包
2. 实现 WIT 接口（`wit/tool.wit`）
3. 创建 `<name>.capabilities.json` 声明所需权限
4. 使用 `cargo build --target wasm32-wasip2 --release` 构建
5. 使用 `ironclaw tool install path/to/tool.wasm` 安装

有关示例，请参阅 `tools-src/`。

## 工具架构原则

**关键：将特定于工具的逻辑排除在主代理代码库之外。**

主代理提供通用基础设施；工具是通过 capabilities 文件声明其要求的自包含单元。

### 工具中的内容（capabilities.json）

- 工具所需的 API 端点（HTTP 允许列表）
- 所需凭证（秘密名称、注入位置）
- 速率限制和超时
- 身份验证设置说明（见下文）
- 工具可以读取的工作空间路径

### 主代理中不包括的内容

- 特定于服务的身份验证流程（Notion、Slack 等的 OAuth）
- 特定于服务的 CLI 命令（`auth notion`、`auth slack`）
- 特定于服务的配置处理
- 硬编码的 API URL 或令牌格式

### 工具身份验证

工具在 `<tool>.capabilities.json` 的 `auth` 部分中声明其身份验证要求。支持两种方法：

#### OAuth（基于浏览器的登录）

对于支持 OAuth 的服务，用户只需点击浏览器登录：

```json
{
  "auth": {
    "secret_name": "notion_api_token",
    "display_name": "Notion",
    "oauth": {
      "authorization_url": "https://api.notion.com/v1/oauth/authorize",
      "token_url": "https://api.notion.com/v1/oauth/token",
      "client_id_env": "NOTION_OAUTH_CLIENT_ID",
      "client_secret_env": "NOTION_OAUTH_CLIENT_SECRET",
      "scopes": [],
      "use_pkce": false,
      "extra_params": { "owner": "user" }
    },
    "env_var": "NOTION_TOKEN"
  }
}
```

要为工具启用 OAuth：
1. 向服务注册公共 OAuth 应用程序（例如，notion.so/my-integrations）
2. 配置重定向 URI：`http://localhost:9876/callback` 到 `http://localhost:9886/callback`
3. 为 client_id 和 client_secret 设置环境变量

#### 手动令牌输入（回退）

对于没有 OAuth 的服务或未配置 OAuth 时：

```json
{
  "auth": {
    "secret_name": "openai_api_key",
    "display_name": "OpenAI",
    "instructions": "从 platform.openai.com/api-keys 获取您的 API 密钥",
    "setup_url": "https://platform.openai.com/api-keys",
    "token_hint": "以 'sk-' 开头",
    "env_var": "OPENAI_API_KEY"
  }
}
```

#### 身份验证流程优先级

运行 `ironclaw tool auth <tool>` 时：

1. 检查 `env_var` - 如果在环境中设置，直接使用
2. 检查 `oauth` - 如果配置，打开浏览器进行 OAuth 流程
3. 回退到 `instructions` + 手动令牌输入

代理从工具的 capabilities 文件中读取身份验证配置并提供适当的流程。主代理中没有特定于服务的代码。

### WASM 工具与 MCP 服务器：何时使用哪个

两者都是扩展系统中的第一等公民（`ironclaw tool install` 处理两者），但它们具有不同的优势。

**WASM 工具（IronClaw 原生）**

- 沙箱化：燃料计量、内存限制、除了允许列表之外无访问权限
- 凭证由主机运行时注入，工具代码永远不会看到实际令牌
- 在返回 LLM 之前扫描输出是否泄露秘密
- 在 `capabilities.json` 中声明的身份验证（OAuth/手动），代理处理流程
- 单个二进制文件，无进程管理，可离线工作
- 成本：必须自己用 Rust 构建，无生态系统，仅同步

**MCP 服务器（模型上下文协议）**

- 预构建服务器的不断增长的生态系统（GitHub、Notion、Postgres 等）
- 任何语言（TypeScript/Python 最常见）
- 可以做 websockets、流式传输、后台轮询
- 成本：具有完全系统访问权限的外部进程（无沙箱），管理自己的凭证，IronClaw 无法防止泄露

**决策指南：**

| 场景 | 使用 |
|----------|-----|
| 已存在良好的 MCP 服务器 | **MCP** |
| 处理敏感凭证（电子邮件发送、银行） | **WASM** |
| 快速原型或一次性集成 | **MCP** |
| 您将长期维护的核心功能 | **WASM** |
| 需要后台连接（websockets、轮询） | **MCP** |
| 多个工具共享一个 OAuth 令牌（例如，Google 套件） | **WASM** |

LLM 面向的接口对于两者是相同的（工具名称、模式、execute），因此在代理眼中在它们之间交换是透明的。
