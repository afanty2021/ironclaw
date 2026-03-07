# IronClaw E2E 测试

Python/Playwright 测试套件，针对运行的 ironclaw 实例。在 PR #553（"轨迹基准和 e2e 跟踪测试装备"）中添加。

## 设置

```bash
cd tests/e2e

# 创建 virtualenv（一次性）
python -m venv .venv
source .venv/bin/activate   # Windows 上为 .venv\Scripts\activate

# 安装依赖
pip install -e .

# 安装浏览器二进制文件（一次性）
playwright install chromium
```

依赖项：`pytest`、`pytest-asyncio`、`pytest-playwright`、`pytest-timeout`、`playwright`、`aiohttp`、`httpx`。可选：`anthropic`（vision extras）。需要 Python >= 3.11。

## 运行测试

```bash
# 首先激活 venv
source .venv/bin/activate

# 运行所有场景（conftest.py 自动构建二进制文件并启动所有服务器）
pytest scenarios/

# 运行特定场景
pytest scenarios/test_chat.py
pytest scenarios/test_sse_reconnect.py

# 详细输出运行
pytest scenarios/ -v

# 使用特定超时运行（默认每个测试 120s，在 pyproject.toml 中设置）
pytest scenarios/ --timeout=60

# 使用有头浏览器运行（对调试有用）
HEADED=1 pytest scenarios/
```

## 测试场景

| 文件 | 测试内容 |
|------|----------|
| `test_connection.py` | 网关可达性、选项卡导航、认证拒绝（无令牌显示认证屏幕） |
| `test_chat.py` | 通过浏览器 UI 发送消息，验证来自模拟 LLM 的流式响应；还测试空消息抑制 |
| `test_html_injection.py` | 通过 `page.evaluate("addMessage('assistant', ...)")` 直接注入的 XSS 向量被 `renderMarkdown` 清理；用户消息显示为转义的纯文本 |
| `test_skills.py` | 技能选项卡 UI 可见性、ClawHub 搜索（如果注册表不可达则跳过）、安装 + 移除生命周期 |
| `test_sse_reconnect.py` | SSE 在程序化 `eventSource.close()` + `connectSSE()` 后重新连接；重新连接后重新加载历史 |
| `test_tool_approval.py` | 批准卡出现、批准/拒绝时按钮禁用、参数切换；所有通过 `page.evaluate("showApproval(...)")` 触发 — 不需要真正的工具调用 |

## `helpers.py`

每个测试文件和 `conftest.py` 导入的共享常量和工具。

- **`SEL`** — 所有 DOM 元素的 CSS/ID 选择器字典（聊天输入、消息气泡、批准卡、选项卡按钮、技能搜索等）。当前端 HTML 更改时更新此字典；测试从此导入选择器，而不是硬编码它们。
- **`TABS`** — 有序选项卡名称列表：`["chat", "memory", "jobs", "routines", "extensions", "skills"]`。
- **`AUTH_TOKEN`** — 硬编码为 `"e2e-test-token"`。`conftest.py` 在启动服务器时使用它（`GATEWAY_AUTH_TOKEN`），`page` fixture 在导航时使用它（`/?token=e2e-test-token`）。
- **`wait_for_ready(url, timeout, interval)`** — 轮询 URL 直到 HTTP 200 或超时；用于等待网关和模拟 LLM 变得可用。
- **`wait_for_port_line(process, pattern, timeout)`** — 逐行读取子进程的 stdout 直到正则匹配；用于从 `MOCK_LLM_PORT=XXXX` 提取动态分配的模拟 LLM 端口。

## `conftest.py` 和 Fixtures

所有 fixtures 都在 `tests/e2e/conftest.py` 中定义。从 `tests/e2e/` 目录运行 `pytest scenarios/` 会自动选择此 conftest（它位于 `scenarios/` 上方一级）。

### 会话范围的 fixtures（每次 `pytest` 调用运行一次）

| Fixture | 功能 |
|---------|-------------|
| `ironclaw_binary` | 检查 `target/debug/ironclaw`；如果不存在，运行 `cargo build --no-default-features --features libsql`（超时 600s）。 |
| `mock_llm_server` | 启动 `mock_llm.py --port 0`，从 stdout 读取分配的端口，等待 `/v1/models` 返回 200。产生基本 URL。 |
| `ironclaw_server` | 使用最小环境启动 ironclaw 二进制文件（见下文），等待 `/api/health`（超时 60s）。产生基本 URL。在拆解时发送 **SIGINT**（不是 SIGTERM），以便 tokio ctrl_c 处理程序触发优雅关闭并刷新 LLVM 覆盖数据。 |
| `browser` | 启动单个 Chromium 实例（默认无头；设置 `HEADED=1` 为有头）。在所有测试之间共享。 |

### 函数范围的 fixtures

| Fixture | 功能 |
|---------|-------------|
| `page` | 每个测试创建一个新浏览器**上下文**（视口 1280×720）和**页面**，导航到 `/?token=e2e-test-token`，并在产生之前等待 `#auth-screen` 被隐藏。在每个测试后关闭上下文。 |

函数范围的 `page` fixture 意味着**每个测试都有一个干净的浏览器上下文**（cookie、存储等），但重用同一个 ironclaw 服务器和浏览器进程。需要直接访问服务器 URL 的测试（例如，`test_auth_rejection`）接受 `ironclaw_server` 作为附加参数。

### 测试中传递给 ironclaw 的环境

`ironclaw_server` fixture 注入最小的确定性环境：

```
GATEWAY_ENABLED=true, GATEWAY_HOST=127.0.0.1, GATEWAY_PORT=<dynamic>
GATEWAY_AUTH_TOKEN=e2e-test-token, GATEWAY_USER_ID=e2e-tester
CLI_ENABLED=false
LLM_BACKEND=openai_compatible, LLM_BASE_URL=<mock_llm_url>, LLM_MODEL=mock-model
DATABASE_BACKEND=libsql, LIBSQL_PATH=<tmpdir>/e2e.db
SANDBOX_ENABLED=false, ROUTINES_ENABLED=false, HEARTBEAT_ENABLED=false
EMBEDDING_ENABLED=false, SKILLS_ENABLED=true
ONBOARD_COMPLETED=true   # 防止设置向导
```

二进制文件也以 `--no-onboard` 启动。覆盖环境变量（`CARGO_LLVM_COV*`、`LLVM_*`、`CARGO_ENCODED_RUSTFLAGS`、`CARGO_INCREMENTAL`）在存在时从外部环境转发。

## 模拟 LLM（`mock_llm.py`）

`aiohttp` 基于 OpenAI 兼容服务器，用于需要确定性 LLM 响应而无需访问真实提供商的测试。

```bash
# 手动启动（端口自动选择，打印为 MOCK_LLM_PORT=XXXX）
python mock_llm.py --port 0
```

它提供 `POST /v1/chat/completions`（流式 + 非流式）和 `GET /v1/models`。响应从 `CANNED_RESPONSES` 中针对最后一条用户消息进行模式匹配。不匹配的消息返回 `"I understand your request."`。报告的模型名称始终是 `"mock-model"`。

要添加新的罐头响应：
```python
# 在 mock_llm.py 中
CANNED_RESPONSES = [
    (re.compile(r"your pattern", re.IGNORECASE), "Your response"),
    ...
]
```

## 配置

`conftest.py` 自动处理所有服务器启动 — 您不需要在运行 `pytest` 之前手动启动 ironclaw。conftest 构建二进制文件（libsql 功能），启动模拟 LLM，并使用每个 `pytest` 调用的新临时数据库启动 ironclaw。

如果您需要针对手动启动的 ironclaw 进行测试，可以通过 `--co`（仅收集）运行 pytest 来理解将要运行的内容，或者直接调用 httpx/REST 助手而不使用 `page` fixture。

## 编写新场景

1. 创建 `scenarios/test_my_feature.py`。
2. 所有异步函数都自动识别为测试 — `asyncio_mode = "auto"` 在 `pyproject.toml` 中全局设置。**不要**添加 `@pytest.mark.asyncio`；它是多余的并引发警告。
3. 将 `page` fixture 用于浏览器测试（函数范围，每个测试都有新上下文）。直接使用 `ironclaw_server` 进行纯 HTTP 测试。
4. 从 `helpers.SEL` 和 `helpers.AUTH_TOKEN` 导入选择器 — 不要内联硬编码选择器或令牌。
5. 使用 `httpx.AsyncClient` 进行 REST 调用；`aiohttp` 用于 SSE 流式传输。
6. 尽可能保持新 fixtures 会话范围；服务器启动很昂贵。函数范围的 fixtures（如 `page`）对于每个测试必须干净的浏览器状态是可以的。

```python
import httpx
from helpers import AUTH_TOKEN

async def test_my_endpoint(ironclaw_server):
    headers = {"Authorization": f"Bearer {AUTH_TOKEN}"}
    async with httpx.AsyncClient() as client:
        r = await client.get(f"{ironclaw_server}/api/health", headers=headers)
        assert r.status_code == 200
```

对于浏览器测试：
```python
from helpers import SEL

async def test_my_ui_feature(page):
    # 页面已经导航并经过身份验证
    chat_input = page.locator(SEL["chat_input"])
    await chat_input.wait_for(state="visible", timeout=5000)
    # ... 与页面交互 ...
```

### 陷阱

- **`asyncio_default_fixture_loop_scope = "session"`** — 所有异步 fixtures 共享一个事件循环。不要在 fixtures 内使用 `asyncio.run()`；直接使用 `await`。
- **`page` fixture 使用 `/?token=e2e-test-token` 导航并等待 `#auth-screen` 被隐藏。** 测试接收一个已经通过认证屏幕并连接了 SSE 的页面。
- **`test_skills.py` 对 ClawHub 进行真实的网络调用。** 如果注册表不可达，测试跳过（而不是失败），通过 `pytest.skip()`。
- **`test_html_injection.py` 和 `test_tool_approval.py` 通过 `page.evaluate(...)` 注入状态。** 它们测试浏览器端渲染管道，不依赖于 LLM 或后端工具执行。
- **浏览器仅限 Chromium。** `conftest.py` 使用 `p.chromium.launch()`；没有 Firefox 或 WebKit 变体。
- **默认超时为 120 秒**（pyproject.toml）。测试内的单个 `wait_for` 调用使用更短的超时（5-20s）以获得更快的失败消息。
- **libsql 数据库是一个临时目录**，每次 `pytest` 调用时新创建；测试不在运行之间共享状态。

## CI 集成

E2E 测试在 CI 中运行，使用 `cargo-llvm-cov` 进行覆盖收集。CI 工作流（"fix(ci): persist all cargo-llvm-cov env vars for E2E coverage" — PR #559）在生成 ironclaw 二进制文件之前设置 `LLVM_PROFILE_FILE` 和相关变量，以便捕获来自 E2E 运行的覆盖。
