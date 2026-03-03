# Codex 配置整合文档

本文整合以下 4 篇配置相关文章：

1. `config-basic.zh-CN.md`
2. `config-advanced.zh-CN.md`
3. `config-reference.zh-CN.md`
4. `config-sample.zh-CN.md`

## 目录

1. [配置基础知识](#配置基础知识)
2. [高级配置](#高级配置)
3. [配置参考](#配置参考)
4. [配置示例](#配置示例)

---

## 配置基础知识

来源文件：`config-basic.zh-CN.md`


Codex 从多个位置读取配置详细信息。您的个人默认值位于 `~/.codex/config.toml` 中，您可以使用 `.codex/config.toml` 文件添加项目覆盖。为了安全起见，Codex 仅在您信任项目时才加载项目配置文件。

## Codex配置文件

Codex 将用户级配置存储在 `~/.codex/config.toml` 处。要将设置范围限制到特定项目或子文件夹，请在存储库中添加 `.codex/config.toml` 文件。

要从 Codex IDE 扩展打开配置文件，请选择右上角的齿轮图标，然后选择 **Codex Settings > Open config.toml**。

CLI 和 IDE 扩展共享相同的配置层。您可以使用它们来：

- 设置默认模型和提供商。
- 配置[审批策略和沙箱设置](https://developers.openai.com/codex/security#sandbox-and-approvals)。
- 配置 [MCP 服务器](https://developers.openai.com/codex/mcp)。

## 配置优先级

Codex 按以下顺序解析值（优先级最高）：

1. CLI 标志和 `--config` 覆盖
2. [配置档](https://developers.openai.com/codex/config-advanced#profiles) 值（来自 `--profile <name>`）
3. 项目配置文件：`.codex/config.toml`，从项目根目录到当前工作目录（最近的获胜；仅限受信任的项目）
4. 用户配置：`~/.codex/config.toml`
5. 系统配置（如果存在）：Unix 上的 `/etc/codex/config.toml`
6. 内置默认值

使用该优先级在顶层设置共享默认值，并使配置文件重点关注不同的值。

如果您将项目标记为不受信任，Codex 会跳过项目范围的 `.codex/` 层（包括 `.codex/config.toml`）并回退到用户、系统和内置默认值。

对于通过 `-c`/`--config` 进行的一次性覆盖（包括 TOML 引用规则），请参阅 [高级配置](https://developers.openai.com/codex/config-advanced#one-off-overrides-from-the-cli)。

在托管计算机上，您的组织还可以通过以下方式强制实施约束
`requirements.toml`（例如，不允许 `approval_policy = "never"` 或
`sandbox_mode = "danger-full-access"`)。请参阅[托管
配置](https://developers.openai.com/codex/security#managed-configuration) 和 [管理员强制
要求](https://developers.openai.com/codex/enterprise/managed-configuration#admin-enforced-requirements-requirementstoml)。

## 常用配置选项

以下是人们最常更改的一些选项：

#### 默认模型

选择 Codex 在 CLI 和 IDE 中默认使用的模型。

```toml
model = "gpt-5.2"
```

#### 审批提示

控制 Codex 在运行生成的命令之前何时暂停询问。

```toml
approval_policy = "on-request"
```

有关 `untrusted`、`on-request` 和 `never` 之间的行为差异，请参阅 [无需审批提示运行](https://developers.openai.com/codex/security#run-without-approval-prompts) 和 [常见沙箱与审批组合](https://developers.openai.com/codex/security#common-sandbox-and-approval-combinations)。

#### 沙箱级别

调整 Codex 在执行命令时拥有的文件系统和网络访问权限。

```toml
sandbox_mode = "workspace-write"
```

对于逐个模式的行为（包括受保护的 `.git`/`.codex` 路径和网络默认值），请参阅 [沙箱与审批](https://developers.openai.com/codex/security#sandbox-and-approvals)、[可写根中的受保护路径](https://developers.openai.com/codex/security#protected-paths-in-writable-roots) 和 [网络接入](https://developers.openai.com/codex/security#network-access)。

#### Windows 沙箱模式

在 Windows 上本机运行 Codex 时，请将本机沙箱模式设置为 `windows` 表中的 `elevated`。仅当您没有管理员权限或提升的安装失败时才使用 `unelevated`。

```toml
[windows]
sandbox = "elevated"   # Recommended
# sandbox = "unelevated" # Fallback if admin permissions/setup are unavailable
```

#### 网页搜索模式

Codex 默认为本地任务启用 Web 搜索，并提供来自 Web 搜索缓存的结果。缓存是 OpenAI 维护的 Web 结果索引，因此缓存模式返回预先索引的结果，而不是获取实时页面。这可以减少来自任意实时内容的提示注入的风险，但您仍应将 Web 结果视为不可信。如果您使用 `--yolo` 或其他[完全访问沙箱设置](https://developers.openai.com/codex/security#common-sandbox-and-approval-combinations)，网络搜索默认为实时结果。选择带有 `web_search` 的模式：

- `"cached"`（默认）提供来自网络搜索缓存的结果。
- `"live"` 从网络获取最新数据（与 `--search` 相同）。
- `"disabled"` 关闭网络搜索工具。

```toml
web_search = "cached"  # default; serves results from the web search cache
# web_search = "live"  # fetch the most recent data from the web (same as --search)
# web_search = "disabled"
```

#### 推理努力

调整模型在支持时应用的推理工作量。

```toml
model_reasoning_effort = "high"
```

#### 沟通方式

为支持的模型设置默认通信方式。

```toml
personality = "friendly" # or "pragmatic" or "none"
```

您可以稍后在活动会话中使用 `/personality` 或使用应用程序服务器 API 时按线程/回合覆盖此设置。

#### 命令环境

控制 Codex 将哪些环境变量转发给生成的命令。

```toml
[shell_environment_policy]
include_only = ["PATH", "HOME"]
```

#### 日志目录

覆盖 Codex 写入本地日志文件（例如 `codex-tui.log`）的位置。

```toml
log_dir = "/absolute/path/to/codex-logs"
```

对于一次性运行，您还可以从 CLI 进行设置：

```bash
codex -c log_dir=./.codex-log
```

## 功能标志

使用 `config.toml` 中的 `[features]` 表来切换可选和实验功能。

```toml
[features]
shell_snapshot = true           # Speed up repeated commands
```

### 支持的功能

| Key                       | Default | Maturity     | Description                                                                                        |
| ------------------------- | :-----: | ------------ | -------------------------------------------------------------------------------------------------- |
| `apply_patch_freeform`    |  false  | Experimental | Include the freeform `apply_patch` tool                                                            |
| `apps`                    |  错误的  | 实验性的 | 启用 ChatGPT 应用程序/连接器支持                                                             |
| `apps_mcp_gateway`        |  错误的  | 实验性的 | 通过 `https://api.openai.com/v1/connectors/mcp/` 路由应用程序 MCP 呼叫，而不是传统路由 |
| `collaboration_modes`     |  true   | Stable       | Enable collaboration modes such as plan mode                                                       |
| `multi_agent`             |  错误的  | 实验性的 | 启用多代理协作工具                                                             |
| `personality`             |  true   | Stable       | Enable personality selection controls                                                              |
| `remote_models`           |  false  | Experimental | Refresh remote model list before showing readiness                                                 |
| `runtime_metrics`         |  false  | Experimental | Show runtime metrics summaries in TUI turn separators                                              |
| `request_rule`            |  真的   | 稳定的       | 启用智能审批（`prefix_rule` 建议）                                                 |
| `search_tool`             |  错误的  | 实验性的 | 启用 `search_tool_bm25` 以便 Codex 在工具调用之前通过搜索发现 Apps MCP 工具           |
| `shell_snapshot`          |  false  | Beta         | Snapshot your shell environment to speed up repeated commands                                      |
| `shell_tool`              |  true   | Stable       | Enable the default `shell` tool                                                                    |
| `use_linux_sandbox_bwrap` |  错误的  | 实验性的 | 使用基于 bubblewrap 的 Linux 沙箱管道                                                    |
| `unified_exec`            |  错误的  | 贝塔         | 使用统一的 PTY 支持的执行工具                                                               |
| `undo`                    |  真的   | 稳定的       | 通过每轮 git Ghost 快照启用撤消                                                       |
| `web_search`              |  真的   | 已弃用   | 旧版切换；更喜欢顶级 `web_search` 设置                                           |
| `web_search_cached`       |  真的   | 已弃用   | 未设置时映射到 `web_search = "cached"` 的旧版切换开关                                      |
| `web_search_request`      |  真的   | 已弃用   | 未设置时映射到 `web_search = "live"` 的旧版切换开关                                        |

成熟度列使用功能成熟度标签，例如实验、Beta、
和稳定。请参阅 [功能职业度](https://developers.openai.com/codex/feature-maturity) 了解如何
解释这些标签。

省略功能键以保留其默认值。

### 启用功能

- 在 `config.toml` 中，在 `[features]` 下添加 `feature_name = true`。
- 从 CLI 运行 `codex --enable feature_name`。
- 要启用多个功能，请运行 `codex --enable feature_a --enable feature_b`。
- 要禁用某项功能，请将键设置为 `config.toml` 中的 `false`。

---

## 高级配置

来源文件：`config-advanced.zh-CN.md`


当您需要对提供商、策略和集成进行更多控制时，请使用这些选项。如需快速入门，请参阅[配置基础知识](https://developers.openai.com/codex/config-basic)。

有关项目指导、可重用功能、自定义斜杠命令、多代理工作流程和集成的背景信息，请参阅[定制化](https://developers.openai.com/codex/concepts/customization)。有关配置键，请参阅[配置参考](https://developers.openai.com/codex/config-reference)。

## 配置档

配置档允许你保存命名的配置值集并通过 CLI 在它们之间进行切换。

配置文件是实验性的，可能会在未来的版本中更改或删除。

Codex IDE 扩展当前不支持配置文件。

在 `config.toml` 中的 `[profiles.<name>]` 下定义配置文件，然后运行 ​​`codex --profile <name>`：

```toml
model = "gpt-5-codex"
approval_policy = "on-request"
model_catalog_json = "/Users/me/.codex/model-catalogs/default.json"

[profiles.deep-review]
model = "gpt-5-pro"
model_reasoning_effort = "high"
approval_policy = "never"
model_catalog_json = "/Users/me/.codex/model-catalogs/deep-review.json"

[profiles.lightweight]
model = "gpt-4.1"
approval_policy = "untrusted"
```

要使配置文件成为默认配置，请在 `config.toml` 的顶层添加 `profile = "deep-review"`。 Codex 会加载该配置文件，除非您在命令行上覆盖它。

配置文件还可以覆盖 `model_catalog_json`。当顶层和选定的配置文件都设置 `model_catalog_json` 时，Codex 更喜欢配置文件值。

## 从 CLI 一次性覆盖

除了编辑 `~/.codex/config.toml` 之外，您还可以从 CLI 覆盖单次运行的配置：

- 首选专用标志（如果存在）（例如 `--model`）。
- 当您需要覆盖任意键时，请使用 `-c` / `--config` 。

示例：

```shell
# Dedicated flag
codex --model gpt-5.2

# Generic key/value override (value is TOML, not JSON)
codex --config model='"gpt-5.2"'
codex --config sandbox_workspace_write.network_access=true
codex --config 'shell_environment_policy.include_only=["PATH","HOME"]'
```

说明：

- 键可以使用点表示法来设置嵌套值（例如，`mcp_servers.context7.enabled=false`）。
- `--config` 值被解析为 TOML。如有疑问，请引用该值，以便 shell 不会将其分割为空格。
- 如果该值无法解析为 TOML，Codex 会将其视为字符串。

## 配置和状态位置

Codex 将其本地状态存储在 `CODEX_HOME` 下（默认为 `~/.codex`）。

您可能会在那里看到常见文件：

- `config.toml`（您的本地配置）
- `auth.json`（如果您使用基于文件的凭证存储）或您的操作系统钥匙串/钥匙圈
- `history.jsonl`（如果启用历史记录持久化）
- 其他每用户状态，例如日志和缓存

有关身份验证详细信息（包括凭证存储模式），请参阅[身份验证](https://developers.openai.com/codex/auth)。有关配置键的完整列表，请参阅[配置参考](https://developers.openai.com/codex/config-reference)。

有关签入存储库或系统路径的共享默认值、规则和技能，请参阅 [团队配置](https://developers.openai.com/codex/enterprise/admin-setup#team-config)。

如果您只需将内置 OpenAI 提供程序指向 LLM 代理、路由器或启用数据驻留的项目，请设置环境变量 `OPENAI_BASE_URL` 而不是定义新的提供程序。这将覆盖默认的 OpenAI 端点，而不进行 `config.toml` 更改。

```shell
export OPENAI_BASE_URL="https://api.openai.com/v1"
codex
```

## 项目配置文件 (`.codex/config.toml`)

除了您的用户配置之外，Codex 还会从存储库内的 `.codex/config.toml` 文件中读取项目范围的覆盖。 Codex 从项目根目录走到当前工作目录并加载它找到的每个 `.codex/config.toml` 。如果多个文件定义相同的密钥，则最接近您的工作目录的文件获胜。

为了安全起见，Codex 仅在项目受信任时才加载项目范围的配置文件。如果项目不受信任，Codex 会忽略项目中的 `.codex/config.toml` 文件。

项目配置内的相对路径（例如 `experimental_instructions_file`）是相对于包含 `config.toml` 的 `.codex/` 文件夹进行解析的。

## 代理角色（`config.toml` 中的 `[agents]`）

有关多代理角色配置（`config.toml` 中的 `[agents]`），请参阅[多代理](https://developers.openai.com/codex/multi-agent)。

## 项目根检测

Codex 通过从工作目录向上直至到达项目根来发现项目配置（例如，`.codex/` 层和 `AGENTS.md`）。

默认情况下，Codex 将包含 `.git` 的目录视为项目根目录。要自定义此行为，请在 `config.toml` 中设置 `project_root_markers`：

```toml
# Treat a directory as the project root when it contains any of these markers.
project_root_markers = [".git", ".hg", ".sl"]
```

设置 `project_root_markers = []` 跳过搜索父目录并将当前工作目录视为项目根目录。

## 定制模型提供商

模型提供程序定义 Codex 如何连接到模型（基本 URL、wire API 和可选的 HTTP 标头）。

定义其他提供者并将 `model_provider` 指向它们：

```toml
model = "gpt-5.1"
model_provider = "proxy"

[model_providers.proxy]
name = "OpenAI using LLM proxy"
base_url = "http://proxy.example.com"
env_key = "OPENAI_API_KEY"

[model_providers.ollama]
name = "Ollama"
base_url = "http://localhost:11434/v1"

[model_providers.mistral]
name = "Mistral"
base_url = "https://api.mistral.ai/v1"
env_key = "MISTRAL_API_KEY"
```

需要时添加请求标头：

```toml
[model_providers.example]
http_headers = { "X-Example-Header" = "example-value" }
env_http_headers = { "X-Example-Features" = "EXAMPLE_FEATURES" }
```

## OSS模式（本地提供商）

当您传递 `--oss` 时，Codex 可以针对本地“开源”提供程序（例如 Ollama 或 LM Studio）运行。如果您传递 `--oss` 而不指定提供程序，Codex 将使用 `oss_provider` 作为默认值。

```toml
# Default local provider used with `--oss`
oss_provider = "ollama" # or "lmstudio"
```

## Azure 提供商和每个提供商的调整

```toml
[model_providers.azure]
name = "Azure"
base_url = "https://YOUR_PROJECT_NAME.openai.azure.com/openai"
env_key = "AZURE_OPENAI_API_KEY"
query_params = { api-version = "2025-04-01-preview" }
wire_api = "responses"

[model_providers.openai]
request_max_retries = 4
stream_max_retries = 10
stream_idle_timeout_ms = 300000
```

## 使用数据驻留的 ChatGPT 客户

启用[数据驻留](https://help.openai.com/en/articles/9903489-data-residency-and-inference-residency-for-chatgpt) 创建的项目可以创建模型提供程序，以使用[正确的导出](https://platform.openai.com/docs/guides/your-data#which-models-and-features-are-eligible-for-data-residency) 更新 base_url。

```toml
model_provider = "openaidr"
[model_providers.openaidr]
name = "OpenAI Data Residency"
base_url = "https://us.api.openai.com/v1" # Replace 'us' with domain prefix
```

## 模型推理、冗长和限制

```toml
model_reasoning_summary = "none"          # Disable summaries
model_verbosity = "low"                   # Shorten responses
model_supports_reasoning_summaries = true # Force reasoning
model_context_window = 128000             # Context window size
```

`model_verbosity` 仅适用于使用响应 API 的提供商。聊天完成提供商将忽略该设置。

## 审批策略和沙盒模式

选择批准严格性（影响 Codex 暂停时）和沙箱级别（影响文件/网络访问）。

编辑`config.toml`时容易忽略的操作细节，请参见[常见的沙和关节组合](https://developers.openai.com/codex/security#common-sandbox-and-approval-combinations)、[可写根中的受保护路径](https://developers.openai.com/codex/security#protected-paths-in-writable-roots)和[网络接入](https://developers.openai.com/codex/security#network-access)。

您还可以使用精细拒绝策略 (`approval_policy = { reject = { ... } }`) 仅自动拒绝选定的提示类别（沙箱审批、execpolicy 规则提示或 MCP 引发），同时保持其他提示交互。

```toml
approval_policy = "untrusted"   # Other options: on-request, never, or { reject = { ... } }
sandbox_mode = "workspace-write"
allow_login_shell = false       # Optional hardening: disallow login shells for shell tools

[sandbox_workspace_write]
exclude_tmpdir_env_var = false  # Allow $TMPDIR
exclude_slash_tmp = false       # Allow /tmp
writable_roots = ["/Users/YOU/.pyenv/shims"]
network_access = false          # Opt in to outbound network
```

需要完整的密钥列表（包括配置文件范围的覆盖和需求约束）？请参阅[配置参考](https://developers.openai.com/codex/config-reference) 和[托管配置](https://developers.openai.com/codex/security#managed-configuration)。

在工作区写入模式下，某些环境保留 `.git/` 和 `.codex/`
即使工作区的其余部分是可写的，也是只读的。这就是为什么
像 `git commit` 这样的命令可能仍然需要批准才能在
沙箱。如果您希望 Codex 跳过特定命令（例如，阻止 `git
commit` 在沙箱之外），使用
<a href="/codex/rules">规则</a>。

完全禁用沙箱（仅当您的环境已经隔离进程时才使用）：

```toml
sandbox_mode = "danger-full-access"
```

## Shell环境政策

`shell_environment_policy` 控制 Codex 将哪些环境变量传递给它启动的任何子进程（例如，在运行模型建议的工具命令时）。从干净的开始 (`inherit = "none"`) 或修剪集 (`inherit = "core"`) 开始，然后分层排除、包含和覆盖以避免泄露机密，同时仍然提供任务所需的路径、键或标志。

```toml
[shell_environment_policy]
inherit = "none"
set = { PATH = "/usr/bin", MY_FLAG = "1" }
ignore_default_excludes = false
exclude = ["AWS_*", "AZURE_*"]
include_only = ["PATH", "HOME"]
```

模式是不区分大小写的 glob（`*`、`?`、`[A-Z]`）； `ignore_default_excludes = false` 在包含/排除运行之前保留自动 KEY/SECRET/TOKEN 过滤器。

## MCP服务器

有关配置详细信息，请参阅专用的 [MCP 文档](https://developers.openai.com/codex/mcp)。

## 可观测性和遥测

启用 OpenTelemetry (OTel) 日志导出以跟踪 Codex 运行（API 请求、SSE/事件、提示、工具批准/结果）。默认禁用；通过 `[otel]` 选择加入：

```toml
[otel]
environment = "staging"   # defaults to "dev"
exporter = "none"         # set to otlp-http or otlp-grpc to send events
log_user_prompt = false   # redact user prompts unless explicitly enabled
```

选择出口商：

```toml
[otel]
exporter = { otlp-http = {
  endpoint = "https://otel.example.com/v1/logs",
  protocol = "binary",
  headers = { "x-otlp-api-key" = "${OTLP_TOKEN}" }
}}
```

```toml
[otel]
exporter = { otlp-grpc = {
  endpoint = "https://otel.example.com:4317",
  headers = { "x-otlp-meta" = "abc123" }
}}
```

如果 `exporter = "none"` Codex 记录事件但不发送任何内容。导出器异步批处理并在关闭时刷新。事件元数据包括服务名称、CLI 版本、环境标记、对话 ID、模型、沙箱/批准设置和每个事件字段（请参阅 [配置参考](https://developers.openai.com/codex/config-reference)）。

### 发出什么

Codex 发出有关运行和工具使用情况的结构化日志事件。代表性事件类型包括：

- `codex.conversation_starts`（模型、推理设置、沙箱/批准策略）
- `codex.api_request`（尝试、状态/成功、持续时间和错误详细信息）
- `codex.sse_event`（流事件类型、成功/失败、持续时间以及 `response.completed` 上的令牌计数）
- `codex.websocket_request` 和 `codex.websocket_event` （请求持续时间加上每条消息的类型/成功/错误）
- `codex.user_prompt`（长度；内容经过编辑，除非明确启用）
- `codex.tool_decision`（批准/拒绝以及决定是否来自配置与用户）
- `codex.tool_result`（持续时间、成功、输出片段）

### 发布的 OTel 指标

启用 OTel 指标管道后，Codex 会发出 API、流和工具活动的计数器和持续时间直方图。

下面的每个指标还包括默认元数据标签：`auth_mode`、`originator`、`session_source`、`model` 和 `app.version`。

| Metric                                | Type      | Fields              | Description                                                       |
| ------------------------------------- | --------- | ------------------- | ----------------------------------------------------------------- |
| `codex.api_request`                   | 柜台   | `status`、`success` | 按 HTTP 状态和成功/失败划分的 API 请求计数。             |
| `codex.api_request.duration_ms`       | 直方图 | `status`、`success` | API 请求持续时间（以毫秒为单位）。                             |
| `codex.sse_event`                     | 柜台   | `kind`、`success`   | 按事件类型和成功/失败划分的 SSE 事件计数。                |
| `codex.sse_event.duration_ms`         | 直方图 | `kind`、`success`   | SSE 事件处理持续时间（以毫秒为单位）。                    |
| `codex.websocket.request`             | 柜台   | `success`           | WebSocket 请求按成功/失败计数。                       |
| `codex.websocket.request.duration_ms` | 直方图 | `success`           | WebSocket 请求持续时间（以毫秒为单位）。                       |
| `codex.websocket.event`               | 柜台   | `kind`、`success`   | 按类型和成功/失败划分的 WebSocket 消息/事件计数。        |
| `codex.websocket.event.duration_ms`   | 直方图 | `kind`、`success`   | WebSocket 消息/事件处理持续时间（以毫秒为单位）。      |
| `codex.tool.call`                     | 柜台   | `tool`、`success`   | 按工具名称和成功/失败进行工具调用计数。           |
| `codex.tool.call.duration_ms`         | 直方图 | `tool`、`success`   | 按工具名称和结果显示的工具执行持续时间（以毫秒为单位）。 |

有关遥测的更多安全和隐私指南，请参阅 [安全](https://developers.openai.com/codex/security#monitoring-and-telemetry)。

### 指标

默认情况下，Codex 定期将少量匿名使用情况和健康数据发送回 OpenAI。这有助于检测 Codex 何时无法正常工作，并显示正在使用哪些功能和配置选项，以便 Codex 团队可以专注于最重要的事情。这些指标不包含任何个人身份信息 (PII)。指标收集独立于 OTel 日志/跟踪导出。

如果您想在计算机上完全禁用跨 Codex 表面的指标收集，请在配置中设置分析标志：

```toml
[analytics]
enabled = false
```

每个指标都包含其自己的字段以及下面的默认上下文字段。

#### 默认上下文字段（适用于每个事件/指标）

- `auth_mode`：`swic` | `api` | `unknown`。
- `model`：所用模型的名称。
- `app.version`：Codex版本。

#### 指标目录

每个指标都包含必填字段以及上面的默认上下文字段。每个指标都以 `codex.` 为前缀。
如果指标包含 `tool` 字段，则它反映所使用的内部工具（例如 `apply_patch` 或 `shell`），并且不包含 `codex` 尝试应用的实际 shell 命令或补丁。

| Metric                                   | Type      | Fields             | Description                                                                                                                   |
| ---------------------------------------- | --------- | ------------------ | ----------------------------------------------------------------------------------------------------------------------------- |
| `feature.state`                          | 柜台   | `feature`、`value` | 与默认值不同的特征值（每个非默认值发出一行）。                                                      |
| `thread.started`                         | 柜台   | `is_git`           | 新线程已创建。                                                                                                           |
| `thread.fork`                            | 柜台   |                    | 通过分叉现有线程创建的新线程。                                                                             |
| `thread.rename`                          | 柜台   |                    | 线程已重命名。                                                                                                               |
| `task.compact`                           | 柜台   | `type`             | 每种类型的压缩次数（`remote` 或 `local`），包括手动和自动。                                              |
| `task.user_shell`                        | 柜台   |                    | 用户 shell 操作的数量（例如 TUI 中的 `!`）。                                                                    |
| `task.review`                            | 柜台   |                    | 触发的评论数量。                                                                                                  |
| `task.undo`                              | 柜台   |                    | 触发的撤消操作数。                                                                                             |
| `approval.requested`                     | 柜台   | `tool`、`approved` | 工具批准请求结果（`approved`、`approved_with_amendment`、`approved_for_session`、`denied`、`abort`）。              |
| `conversation.turn.count`                | 柜台   |                    | 每个线程的用户/助理轮数，记录在线程末尾。                                                           |
| `turn.e2e_duration_ms`                   | 直方图 |                    | 完整回合的端到端时间。                                                                                              |
| `mcp.call`                               | 柜台   | `status`           | MCP 工具调用结果（`ok` 或错误字符串）。                                                                            |
| `model_warning`                          | 柜台   |                    | 警告已发送至模型。                                                                                                    |
| `tool.call`                              | 柜台   | `tool`、`success`  | 工具调用结果（`success`：`true` 或 `false`）。                                                                        |
| `tool.call.duration_ms`                  | 直方图 | `tool`、`success`  | 工具执行时间。                                                                                                          |
| `remote_models.fetch_update.duration_ms` | 直方图 |                    | 是时候获取远程模型定义了。                                                                                       |
| `remote_models.load_cache.duration_ms`   | 直方图 |                    | 是时候加载远程模型缓存了。                                                                                          |
| `shell_snapshot`                         | 柜台   | `success`          | shell快照是否成功。                                                                                    |
| `shell_snapshot.duration_ms`             | 直方图 | `success`          | 是时候拍摄 shell 快照了。                                                                                                |
| `db.init`                                | 柜台   | `status`           | 状态数据库初始化结果（`opened`、`created`、`open_error`、`init_error`）。                                           |
| `db.backfill`                            | 柜台   | `status`           | 初始状态数据库回填结果（`upserted`、`failed`）。                                                                     |
| `db.backfill.duration_ms`                | 直方图 | `status`           | 初始状态数据库回填的持续时间，标记为 `success`、`failed` 或 `partial_failure`。                             |
| `db.error`                               | 柜台   | `stage`            | 状态 DB 操作期间出现错误（例如 `extract_metadata_from_rollout`、`backfill_sessions`、`apply_rollout_items`）。 |
| `db.compare_error`                       | 柜台   | `stage`、`reason`  | 协调期间检测到的状态数据库差异。                                                                        |

### 反馈控制

默认情况下，Codex 允许用户从 `/feedback` 发送反馈。要禁用计算机上跨 Codex 表面的反馈收集，请更新您的配置：

```toml
[feedback]
enabled = false
```

禁用后，`/feedback` 显示禁用消息，并且 Codex 拒绝反馈提交。

### 隐藏或表面推理事件

如果你想减少嘈杂的“推理”输出（例如在 CI 日志中），你可以抑制它：

```toml
hide_agent_reasoning = true
```

如果您想在模型发出原始推理内容时显示原始推理内容：

```toml
show_raw_agent_reasoning = true
```

仅当您的工作流程可接受时才启用原始推理。某些模型/提供者（如 `gpt-oss`）不会发出原始推理；在这种情况下，此设置不会产生明显的效果。

## 通知

每当 Codex 发出支持的事件时，使用 `notify` 触发外部程序（当前仅 `agent-turn-complete`）。这对于桌面 toast、聊天 webhook、CI 更新或内置 TUI 通知未涵盖的任何旁路警报非常方便。

```toml
notify = ["python3", "/path/to/notify.py"]
```

对 `agent-turn-complete` 做出反应的示例 `notify.py` （已截断）：

```python
#!/usr/bin/env python3
import json, subprocess, sys

def main() -> int:
    notification = json.loads(sys.argv[1])
    if notification.get("type") != "agent-turn-complete":
        return 0
    title = f"Codex: {notification.get('last-assistant-message', 'Turn Complete!')}"
    message = " ".join(notification.get("input-messages", []))
    subprocess.check_output([
        "terminal-notifier",
        "-title", title,
        "-message", message,
        "-group", "codex-" + notification.get("thread-id", ""),
        "-activate", "com.googlecode.iterm2",
    ])
    return 0

if __name__ == "__main__":
    sys.exit(main())
```

该脚本接收单个 JSON 参数。常见字段包括：

- `type`（当前 `agent-turn-complete`）
- `thread-id`（会话标识符）
- `turn-id`（转弯标识符）
- `cwd`（工作目录）
- `input-messages`（导致转向的用户消息）
- `last-assistant-message`（最后一条助理消息文本）

将脚本放置在磁盘上的某个位置并将 `notify` 指向它。

#### `notify` 与 `tui.notifications`

- `notify` 运行外部程序（适用于 webhook、桌面通知程序、CI 挂钩）。
- `tui.notifications` 内置于 TUI 中，可以选择按事件类型进行过滤（例如 `agent-turn-complete` 和 `approval-requested`）。
- `tui.notification_method` 控制 TUI 如何发出终端通知（`auto`、`osc9` 或 `bel`）。

在 `auto` 模式下，Codex 更喜欢 OSC 9 通知（一些终端将其解释为桌面通知的终端转义序列），否则会回退到 BEL (`\x07`)。

有关确切的键，请参阅[配置参考](https://developers.openai.com/codex/config-reference)。

## 历史的坚守

默认情况下，Codex 将本地会话记录保存在 `CODEX_HOME` 下（例如 `~/.codex/history.jsonl`）。要禁用本地历史记录持久性：

```toml
[history]
persistence = "none"
```

要限制历史文件大小，请设置 `history.max_bytes`。当文件超出上限时，Codex 会删除最旧的条目并压缩文件，同时保留最新的记录。

```toml
[history]
max_bytes = 104857600 # 100 MiB
```

## 可点击的引文

如果您使用支持它的终端/编辑器集成，Codex 可以将文件引用呈现为可点击的链接。配置 `file_opener` 以选择 Codex 使用的 URI 方案：

```toml
file_opener = "vscode" # or cursor, windsurf, vscode-insiders, none
```

示例：像 `/home/user/project/main.py:42` 这样的引用可以重写为可点击的 `vscode://file/...:42` 链接。

## 项目指令发现

Codex 读取 `AGENTS.md` （和相关文件），并在第一轮会议中包含有限数量的项目指导。两个旋钮控制其工作原理：

- `project_doc_max_bytes`：从每个 `AGENTS.md` 文件读取多少内容
- `project_doc_fallback_filenames`：当目录级别缺少 `AGENTS.md` 时要尝试的其他文件名

有关详细演练，请参阅 [使用 AGENTS.md 自定义指令](https://developers.openai.com/codex/guides/agents-md)。

## 途易选项

运行不带子命令的 `codex` 会启动交互式终端 UI (TUI)。 Codex 在 `[tui]` 下公开了一些特定于 TUI 的配置，包括：

- `tui.notifications`：启用/禁用通知（或限制特定类型）
- `tui.notification_method`：为终端通知选择 `auto`、`osc9` 或 `bel`
- `tui.animations`：启用/禁用 ASCII 动画和闪光效果
- `tui.alternate_screen`：控制备用屏幕使用（设置为 `never` 以保持终端回滚）
- `tui.show_tooltips`：在欢迎屏幕上显示或隐藏入门工具提示

`tui.notification_method` 默认为 `auto`。在 `auto` 模式下，当终端似乎支持 OSC 9 通知（一些终端将其解释为桌面通知的终端转义序列）时，Codex 更倾向于使用 OSC 9 通知，否则会回退到 BEL (`\x07`)。

请参阅 [配置参考](https://developers.openai.com/codex/config-reference) 了解完整的密钥列表。

---

## 配置参考

来源文件：`config-reference.zh-CN.md`


使用此页面作为 Codex 配置文件的可搜索参考。如需概念指导和示例，请从 [配置基础知识](https://developers.openai.com/codex/config-basic) 和 [高级配置](https://developers.openai.com/codex/config-advanced) 开始。

## `config.toml`

用户级配置位于 `~/.codex/config.toml` 中。您还可以在 `.codex/config.toml` 文件中添加项目范围的覆盖。仅当您信任项目时，Codex 才会加载项目范围的配置文件。

对于沙箱与审批键（`approval_policy`、`sandbox_mode` 和 `sandbox_workspace_write.*`），请将此参考与 [沙箱与审批](https://developers.openai.com/codex/security#sandbox-and-approvals)、[可写根中的受保护路径](https://developers.openai.com/codex/security#protected-paths-in-writable-roots) 和 [网络接入](https://developers.openai.com/codex/security#network-access) 配对。

<ConfigTable
  options={[
    {
      key: "model",
      type: "string",
      description: "要使用的模型（例如 `gpt-5-codex`）。",
    },
    {
      key: "review_model",
      type: "string",
      description:
        "`/review` 使用的可选模型覆盖（默认为当前会话模型）。",
    },
    {
      key: "model_provider",
      type: "string",
      description: "来自 `model_providers` 的提供商 ID（默认值：`openai`）。",
    },
    {
      key: "model_context_window",
      type: "number",
      description: "可用于活动模型的上下文窗口标记。",
    },
    {
      key: "model_auto_compact_token_limit",
      type: "number",
      description:
        "触发自动历史记录压缩的令牌阈值（未设置使用模型默认值）。",
    },
    {
      key: "model_catalog_json",
      type: "string (path)",
      description:
        "启动时加载的 JSON 模型目录的可选路径。配置文件级别 `profiles.<name>.model_catalog_json` 可以根据配置文件覆盖此设置。",
    },
    {
      key: "oss_provider",
      type: "lmstudio | ollama",
      description:
        "使用 `--oss` 运行时使用的默认本地提供程序（如果未设置，则默认提示）。",
    },
    {
      key: "approval_policy",
      type: "untrusted | on-request | never | { reject = { sandbox_approval = bool, rules = bool, mcp_elicitations = bool } }",
      description:
        "控制 Codex 在执行命令之前何时暂停以供批准。您还可以使用 `approval_policy = { reject = { ... } }` 自动拒绝特定提示类别，同时保持其他提示交互。 `on-failure` 已弃用；使用 `on-request` 进行交互式运行，或使用 `never` 进行非交互式运行。",
    },
    {
      key: "approval_policy.reject.sandbox_approval",
      type: "boolean",
      description:
        "当 `true` 时，沙箱升级审批提示将被自动拒绝。",
    },
    {
      key: "approval_policy.reject.rules",
      type: "boolean",
      description:
        "当 `true` 时，由 execpolicy `prompt` 规则触发的审批将被自动拒绝。",
    },
    {
      key: "approval_policy.reject.mcp_elicitations",
      type: "boolean",
      description:
        "当 `true` 时，MCP 诱导提示会自动拒绝，而不是显示给用户。",
    },
    {
      key: "allow_login_shell",
      type: "boolean",
      description:
        "允许基于 shell 的工具使用登录 shell 语义。默认为 `true`；当 `false`、`login = true` 请求被拒绝并省略时，`login` 默认为非登录 shell。",
    },
    {
      key: "sandbox_mode",
      type: "read-only | workspace-write | danger-full-access",
      description:
        "命令执行期间文件系统和网络访问的沙箱策略。",
    },
    {
      key: "sandbox_workspace_write.writable_roots",
      type: "array<string>",
      description:
        '当 `sandbox_mode = "workspace-write"` 时的附加可写根目录。',
    },
    {
      key: "sandbox_workspace_write.network_access",
      type: "boolean",
      description:
        "允许工作区写入沙箱内的出站网络访问。",
    },
    {
      key: "sandbox_workspace_write.exclude_tmpdir_env_var",
      type: "boolean",
      description:
        "在工作区写入模式下从可写根中排除 `$TMPDIR`。",
    },
    {
      key: "sandbox_workspace_write.exclude_slash_tmp",
      type: "boolean",
      description:
        "在工作区写入模式下从可写根中排除 `/tmp`。",
    },
    {
      key: "windows.sandbox",
      type: "unelevated | elevated",
      description:
        "在 Windows 上本机运行 Codex 时仅限 Windows 的本机沙盒模式。",
    },
    {
      key: "notify",
      type: "array<string>",
      description:
        "为通知调用的命令；从 Codex 接收 JSON 有效负载。",
    },
    {
      key: "check_for_update_on_startup",
      type: "boolean",
      description:
        "启动时检查 Codex 更新（仅当更新集中管理时设置为 false）。",
    },
    {
      key: "feedback.enabled",
      type: "boolean",
      description:
        "通过 `/feedback` 跨 Codex 界面启用反馈提交（默认值：true）。",
    },
    {
      key: "instructions",
      type: "string",
      description:
        "保留以供将来使用；更喜欢 `model_instructions_file` 或 `AGENTS.md`。",
    },
    {
      key: "developer_instructions",
      type: "string",
      description:
        "注入会话的其他开发人员说明（可选）。",
    },
    {
      key: "log_dir",
      type: "string (path)",
      description:
        "Codex 写入日志文件的目录（例如 `codex-tui.log`）；默认为 `$CODEX_HOME/log`。",
    },
    {
      key: "compact_prompt",
      type: "string",
      description: "历史压缩提示的内联覆盖。",
    },
    {
      key: "model_instructions_file",
      type: "string (path)",
      description:
        "替换内置指令而不是 `AGENTS.md`。",
    },
    {
      key: "personality",
      type: "none | friendly | pragmatic",
      description:
        "宣传 `supportsPersonality` 的模型的默认通信方式；可以按线程/转或通过 `/personality` 覆盖。",
    },
    {
      key: "experimental_compact_prompt_file",
      type: "string (path)",
      description:
        "从文件加载压缩提示覆盖（实验性）。",
    },
    {
      key: "skills.config",
      type: "array<object>",
      description: "每个技能的启用覆盖存储在 config.toml 中。",
    },
    {
      key: "skills.config.<index>.path",
      type: "string (path)",
      description: "包含 `SKILL.md` 的技能文件夹的路径。",
    },
    {
      key: "skills.config.<index>.enabled",
      type: "boolean",
      description: "启用或禁用引用的技能。",
    },
    {
      key: "apps.<id>.enabled",
      type: "boolean",
      description:
        "通过 id 启用或禁用特定应用程序/连接器（默认值：true）。",
    },
    {
      key: "apps._default.enabled",
      type: "boolean",
      description:
        "所有应用程序的默认应用程序启用状态，除非每个应用程序被覆盖。",
    },
    {
      key: "apps._default.destructive_enabled",
      type: "boolean",
      description:
        "默认允许/拒绝带有 `destructive_hint = true` 的应用工具。",
    },
    {
      key: "apps._default.open_world_enabled",
      type: "boolean",
      description:
        "默认允许/拒绝带有 `open_world_hint = true` 的应用工具。",
    },
    {
      key: "apps.<id>.destructive_enabled",
      type: "boolean",
      description:
        "允许或阻止此应用中宣传 `destructive_hint = true` 的工具。",
    },
    {
      key: "apps.<id>.open_world_enabled",
      type: "boolean",
      description:
        "允许或阻止此应用中宣传 `open_world_hint = true` 的工具。",
    },
    {
      key: "apps.<id>.default_tools_enabled",
      type: "boolean",
      description:
        "除非存在每个工具的覆盖，否则此应用程序中工具的默认启用状态。",
    },
    {
      key: "apps.<id>.default_tools_approval_mode",
      type: "auto | prompt | approve",
      description:
        "除非存在每个工具的覆盖，否则此应用程序中工具的默认批准行为。",
    },
    {
      key: "apps.<id>.tools.<tool>.enabled",
      type: "boolean",
      description:
        "应用程序工具的每个工具启用覆盖（例如 `repos/list`）。",
    },
    {
      key: "apps.<id>.tools.<tool>.approval_mode",
      type: "auto | prompt | approve",
      description: "单个应用程序工具的每个工具审批行为覆盖。",
    },
    {
      key: "features.apps",
      type: "boolean",
      description: "启用 ChatGPT 应用程序/连接器支持（实验性）。",
    },
    {
      key: "features.apps_mcp_gateway",
      type: "boolean",
      description:
        "通过 OpenAI 连接器 MCP 网关 (`https://api.openai.com/v1/connectors/mcp/`) 路由应用程序 MCP 调用，而不是传统路由（实验性）。",
    },
    {
      key: "mcp_servers.<id>.command",
      type: "string",
      description: "MCP stdio 服务器的启动器命令。",
    },
    {
      key: "mcp_servers.<id>.args",
      type: "array<string>",
      description: "传递给 MCP stdio 服务器命令的参数。",
    },
    {
      key: "mcp_servers.<id>.env",
      type: "map<string,string>",
      description: "环境变量转发到 MCP stdio 服务器。",
    },
    {
      key: "mcp_servers.<id>.env_vars",
      type: "array<string>",
      description:
        "用于 MCP stdio 服务器白名单的其他环境变量。",
    },
    {
      key: "mcp_servers.<id>.cwd",
      type: "string",
      description: "MCP stdio 服务器进程的工作目录。",
    },
    {
      key: "mcp_servers.<id>.url",
      type: "string",
      description: "MCP 可流式 HTTP 服务器的端点。",
    },
    {
      key: "mcp_servers.<id>.bearer_token_env_var",
      type: "string",
      description:
        "为 MCP HTTP 服务器提供不记名令牌的环境变量。",
    },
    {
      key: "mcp_servers.<id>.http_headers",
      type: "map<string,string>",
      description: "每个 MCP HTTP 请求中包含静态 HTTP 标头。",
    },
    {
      key: "mcp_servers.<id>.env_http_headers",
      type: "map<string,string>",
      description:
        "从 MCP HTTP 服务器的环境变量填充的 HTTP 标头。",
    },
    {
      key: "mcp_servers.<id>.enabled",
      type: "boolean",
      description: "禁用 MCP 服务器而不删除其配置。",
    },
    {
      key: "mcp_servers.<id>.required",
      type: "boolean",
      description:
        "当为 true 时，如果此启用的 MCP 服务器无法初始化，则启动/恢复失败。",
    },
    {
      key: "mcp_servers.<id>.startup_timeout_sec",
      type: "number",
      description:
        "覆盖 MCP 服务器默认的 10 秒启动超时。",
    },
    {
      key: "mcp_servers.<id>.startup_timeout_ms",
      type: "number",
      description: "`startup_timeout_sec` 的别名（以毫秒为单位）。",
    },
    {
      key: "mcp_servers.<id>.tool_timeout_sec",
      type: "number",
      description:
        "覆盖 MCP 服务器默认的每个工具 60 秒超时。",
    },
    {
      key: "mcp_servers.<id>.enabled_tools",
      type: "array<string>",
      description: "MCP 服务器公开的工具名称的允许列表。",
    },
    {
      key: "mcp_servers.<id>.disabled_tools",
      type: "array<string>",
      description:
        "在 MCP 服务器的 `enabled_tools` 之后应用拒绝列表。",
    },
    {
      key: "agents.max_threads",
      type: "number",
      description:
        "可以同时打开的最大代理线程数。",
    },
    {
      key: "agents.max_depth",
      type: "number",
      description:
        "生成的代理线程允许的最大嵌套深度（根会话从深度 0 开始；默认值：1）。",
    },
    {
      key: "agents.<name>.description",
      type: "string",
      description:
        "选择和生成该代理类型时向 Codex 显示的角色指南。",
    },
    {
      key: "agents.<name>.config_file",
      type: "string (path)",
      description:
        "该角色的 TOML 配置层的路径；相对路径从声明角色的配置文件解析。",
    },
    {
      key: "features.unified_exec",
      type: "boolean",
      description: "使用统一的 PTY 支持的执行工具（测试版）。",
    },
    {
      key: "features.shell_snapshot",
      type: "boolean",
      description:
        "快照 shell 环境可加速重复命令（测试版）。",
    },
    {
      key: "features.apply_patch_freeform",
      type: "boolean",
      description: "公开自由形式 `apply_patch` 工具（实验性）。",
    },
    {
      key: "features.multi_agent",
      type: "boolean",
      description:
        "启用多代理协作工具（`spawn_agent`、`send_input`、`resume_agent`、`wait` 和 `close_agent`）（实验性；默认情况下关闭）。",
    },
    {
      key: "features.personality",
      type: "boolean",
      description:
        "启用个性选择控件（稳定；默认打开）。",
    },
    {
      key: "features.web_search",
      type: "boolean",
      description:
        "已弃用的旧版切换；更喜欢顶级 `web_search` 设置。",
    },
    {
      key: "features.web_search_cached",
      type: "boolean",
      description:
        '已弃用的旧版开关。当 `web_search` 未设置时，`true` 映射为 `web_search = "cached"`。',
    },
    {
      key: "features.web_search_request",
      type: "boolean",
      description:
        '已弃用的旧版开关。当 `web_search` 未设置时，`true` 映射为 `web_search = "live"`。',
    },
    {
      key: "features.shell_tool",
      type: "boolean",
      description:
        "启用默认的 `shell` 工具来运行命令（稳定；默认启用）。",
    },
    {
      key: "features.request_rule",
      type: "boolean",
      description:
        "启用智能审批（针对升级请求的 `prefix_rule` 建议；稳定；默认启用）。",
    },
    {
      key: "features.search_tool",
      type: "boolean",
      description:
        "在调用应用程序 MCP 工具之前启用 `search_tool_bm25` 以进行应用程序工具发现（实验）。",
    },
    {
      key: "features.collaboration_modes",
      type: "boolean",
      description:
        "启用协作模式，例如计划模式（稳定；默认开启）。",
    },
    {
      key: "features.use_linux_sandbox_bwrap",
      type: "boolean",
      description:
        "使用基于 bubblewrap 的 Linux 沙箱管道（实验性；默认情况下关闭）。",
    },
    {
      key: "features.remote_models",
      type: "boolean",
      description:
        "在显示准备就绪之前刷新远程模型列表（实验性）。",
    },
    {
      key: "features.runtime_metrics",
      type: "boolean",
      description:
        "在 TUI 回合分隔符中显示运行时指标摘要（实验）。",
    },
    {
      key: "features.powershell_utf8",
      type: "boolean",
      description: "强制 PowerShell UTF-8 输出（默认为 true）。",
    },
    {
      key: "features.child_agents_md",
      type: "boolean",
      description:
        "即使不存在 AGENTS.md，也附加 AGENTS.md 范围/优先级指南（实验性）。",
    },
    {
      key: "suppress_unstable_features_warning",
      type: "boolean",
      description:
        "抑制启用正在开发的功能标志时出现的警告。",
    },
    {
      key: "model_providers.<id>.name",
      type: "string",
      description: "自定义模型提供者的显示名称。",
    },
    {
      key: "model_providers.<id>.base_url",
      type: "string",
      description: "模型提供者的 API 基本 URL。",
    },
    {
      key: "model_providers.<id>.env_key",
      type: "string",
      description: "提供提供商 API 密钥的环境变量。",
    },
    {
      key: "model_providers.<id>.env_key_instructions",
      type: "string",
      description: "提供商 API 密钥的可选设置指南。",
    },
    {
      key: "model_providers.<id>.experimental_bearer_token",
      type: "string",
      description:
        "提供者的直接不记名令牌（不鼓励；使用 `env_key`）。",
    },
    {
      key: "model_providers.<id>.requires_openai_auth",
      type: "boolean",
      description:
        "提供商使用 OpenAI 身份验证（默认为 false）。",
    },
    {
      key: "model_providers.<id>.wire_api",
      type: "chat | responses",
      description:
        "提供者使用的协议（如果省略，则默认为 `chat`）。",
    },
    {
      key: "model_providers.<id>.query_params",
      type: "map<string,string>",
      description: "附加到提供者请求的额外查询参数。",
    },
    {
      key: "model_providers.<id>.http_headers",
      type: "map<string,string>",
      description: "添加到提供者请求的静态 HTTP 标头。",
    },
    {
      key: "model_providers.<id>.env_http_headers",
      type: "map<string,string>",
      description:
        "HTTP 标头由环境变量填充（如果存在）。",
    },
    {
      key: "model_providers.<id>.request_max_retries",
      type: "number",
      description:
        "向提供商发送 HTTP 请求的重试次数（默认值：4）。",
    },
    {
      key: "model_providers.<id>.stream_max_retries",
      type: "number",
      description: "SSE 流中断的重试计数（默认值：5）。",
    },
    {
      key: "model_providers.<id>.stream_idle_timeout_ms",
      type: "number",
      description:
        "SSE 流的空闲超时以毫秒为单位（默认值：300000）。",
    },
    {
      key: "model_reasoning_effort",
      type: "minimal | low | medium | high | xhigh",
      description:
        "调整支持模型的推理工作（仅限响应 API；`xhigh` 取决于模型）。",
    },
    {
      key: "model_reasoning_summary",
      type: "auto | concise | detailed | none",
      description:
        "选择推理摘要详细信息或完全禁用摘要。",
    },
    {
      key: "model_verbosity",
      type: "low | medium | high",
      description:
        "控制 GPT-5 响应 API 详细程度（默认为 `medium`）。",
    },
    {
      key: "model_supports_reasoning_summaries",
      type: "boolean",
      description: "强制 Codex 发送或不发送推理元数据。",
    },
    {
      key: "shell_environment_policy.inherit",
      type: "all | core | none",
      description:
        "生成子进程时的基线环境继承。",
    },
    {
      key: "shell_environment_policy.ignore_default_excludes",
      type: "boolean",
      description:
        "在其他过滤器运行之前保留包含 KEY/SECRET/TOKEN 的变量。",
    },
    {
      key: "shell_environment_policy.exclude",
      type: "array<string>",
      description:
        "用于在默认值之后删除环境变量的全局模式。",
    },
    {
      key: "shell_environment_policy.include_only",
      type: "array<string>",
      description:
        "模式白名单；设置时仅保留匹配的变量。",
    },
    {
      key: "shell_environment_policy.set",
      type: "map<string,string>",
      description:
        "显式环境覆盖注入到每个子进程中。",
    },
    {
      key: "shell_environment_policy.experimental_use_profile",
      type: "boolean",
      description: "生成子进程时使用用户 shell 配置文件。",
    },
    {
      key: "project_root_markers",
      type: "array<string>",
      description:
        "项目根标记文件名列表；在搜索项目根目录的父目录时使用。",
    },
    {
      key: "project_doc_max_bytes",
      type: "number",
      description:
        "构建项目指令时从 `AGENTS.md` 读取的最大字节数。",
    },
    {
      key: "project_doc_fallback_filenames",
      type: "array<string>",
      description: "缺少 `AGENTS.md` 时要尝试的其他文件名。",
    },
    {
      key: "profile",
      type: "string",
      description:
        "启动时应用的默认配置文件（相当于 `--profile`）。",
    },
    {
      key: "profiles.<name>.*",
      type: "various",
      description:
        "任何受支持的配置键的配置文件范围覆盖。",
    },
    {
      key: "profiles.<name>.include_apply_patch_tool",
      type: "boolean",
      description:
        "用于启用自由格式 apply_patch 的旧名称；更喜欢 `[features].apply_patch_freeform`。",
    },
    {
      key: "profiles.<name>.web_search",
      type: "disabled | cached | live",
      description:
        '配置档级别的 Web 搜索模式覆盖（默认：`"cached"`）。',
    },
    {
      key: "profiles.<name>.personality",
      type: "none | friendly | pragmatic",
      description:
        "支持模型的配置文件范围的通信方式覆盖。",
    },
    {
      key: "profiles.<name>.model_catalog_json",
      type: "string (path)",
      description:
        "配置文件范围的模型目录 JSON 路径覆盖（仅在启动时应用；覆盖该配置文件的顶级 `model_catalog_json`）。",
    },
    {
      key: "profiles.<name>.experimental_use_unified_exec_tool",
      type: "boolean",
      description:
        "用于启用统一执行的旧名称；更喜欢 `[features].unified_exec`。",
    },
    {
      key: "profiles.<name>.experimental_use_freeform_apply_patch",
      type: "boolean",
      description:
        "用于启用自由格式 apply_patch 的旧名称；更喜欢 `[features].apply_patch_freeform`。",
    },
    {
      key: "profiles.<name>.oss_provider",
      type: "lmstudio | ollama",
      description: "用于 `--oss` 会话的配置文件范围的 OSS 提供程序。",
    },
    {
      key: "history.persistence",
      type: "save-all | none",
      description:
        "控制 Codex 是否将会话记录保存到history.jsonl。",
    },
    {
      key: "tool_output_token_limit",
      type: "number",
      description:
        "用于在历史中存储单个工具/功能输出的代币预算。",
    },
    {
      key: "background_terminal_max_timeout",
      type: "number",
      description:
        "空 `write_stdin` 轮询（后台终端轮询）的最大轮询窗口（以毫秒为单位）。默认值：`300000`（5 分钟）。替换旧的 `background_terminal_timeout` 密钥。",
    },
    {
      key: "history.max_bytes",
      type: "number",
      description:
        "如果设置，则通过删除最旧的条目来限制历史文件大小（以字节为单位）。",
    },
    {
      key: "file_opener",
      type: "vscode | vscode-insiders | windsurf | cursor | none",
      description:
        "用于打开 Codex 输出引文的 URI 方案（默认值：`vscode`）。",
    },
    {
      key: "otel.environment",
      type: "string",
      description:
        "应用于发出的 OpenTelemetry 事件的环境标记（默认值：`dev`）。",
    },
    {
      key: "otel.exporter",
      type: "none | otlp-http | otlp-grpc",
      description:
        "选择 OpenTelemetry 导出器并提供任何端点元数据。",
    },
    {
      key: "otel.trace_exporter",
      type: "none | otlp-http | otlp-grpc",
      description:
        "选择 OpenTelemetry 跟踪导出器并提供任何端点元数据。",
    },
    {
      key: "otel.log_user_prompt",
      type: "boolean",
      description:
        "选择使用 OpenTelemetry 日志导出原始用户提示。",
    },
    {
      key: "otel.exporter.<id>.endpoint",
      type: "string",
      description: "OTEL 日志的导出器端点。",
    },
    {
      key: "otel.exporter.<id>.protocol",
      type: "binary | json",
      description: "OTLP/HTTP 导出器使用的协议。",
    },
    {
      key: "otel.exporter.<id>.headers",
      type: "map<string,string>",
      description: "OTEL 导出器请求中包含静态标头。",
    },
    {
      key: "otel.trace_exporter.<id>.endpoint",
      type: "string",
      description: "跟踪 OTEL 日志的导出器端点。",
    },
    {
      key: "otel.trace_exporter.<id>.protocol",
      type: "binary | json",
      description: "OTLP/HTTP 跟踪导出器使用的协议。",
    },
    {
      key: "otel.trace_exporter.<id>.headers",
      type: "map<string,string>",
      description: "OTEL 跟踪导出器请求中包含静态标头。",
    },
    {
      key: "otel.exporter.<id>.tls.ca-certificate",
      type: "string",
      description: "OTEL 导出器 TLS 的 CA 证书路径。",
    },
    {
      key: "otel.exporter.<id>.tls.client-certificate",
      type: "string",
      description: "OTEL 导出器 TLS 的客户端证书路径。",
    },
    {
      key: "otel.exporter.<id>.tls.client-private-key",
      type: "string",
      description: "OTEL 导出器 TLS 的客户端私钥路径。",
    },
    {
      key: "otel.trace_exporter.<id>.tls.ca-certificate",
      type: "string",
      description: "OTEL 跟踪导出器 TLS 的 CA 证书路径。",
    },
    {
      key: "otel.trace_exporter.<id>.tls.client-certificate",
      type: "string",
      description: "OTEL 跟踪导出器 TLS 的客户端证书路径。",
    },
    {
      key: "otel.trace_exporter.<id>.tls.client-private-key",
      type: "string",
      description: "OTEL 跟踪导出器 TLS 的客户端私钥路径。",
    },
    {
      key: "tui",
      type: "table",
      description:
        "TUI 特定选项，例如启用内联桌面通知。",
    },
    {
      key: "tui.notifications",
      type: "boolean | array<string>",
      description:
        "启用 TUI 通知；可选择限制特定事件类型。",
    },
    {
      key: "tui.notification_method",
      type: "auto | osc9 | bel",
      description:
        "未聚焦的终端通知的通知方法（默认：自动）。",
    },
    {
      key: "tui.animations",
      type: "boolean",
      description:
        "启用终端动画（欢迎屏幕、闪烁、旋转）（默认值：true）。",
    },
    {
      key: "tui.alternate_screen",
      type: "auto | always | never",
      description:
        "控制 TUI 的备用屏幕使用（默认值：自动；自动在 Zellij 中跳过它以保留回滚）。",
    },
    {
      key: "tui.show_tooltips",
      type: "boolean",
      description:
        "在 TUI 欢迎屏幕中显示入门工具提示（默认值：true）。",
    },
    {
      key: "tui.status_line",
      type: "array<string> | null",
      description:
        "TUI 页脚状态行项目标识符的有序列表。 `null` 禁用状态行。",
    },
    {
      key: "hide_agent_reasoning",
      type: "boolean",
      description:
        "抑制 TUI 和 `codex exec` 输出中的推理事件。",
    },
    {
      key: "show_raw_agent_reasoning",
      type: "boolean",
      description:
        "当活动模型发出原始推理内容时，将其呈现出来。",
    },
    {
      key: "disable_paste_burst",
      type: "boolean",
      description: "在 TUI 中禁用突发粘贴检测。",
    },
    {
      key: "windows_wsl_setup_acknowledged",
      type: "boolean",
      description: "跟踪 Windows 登录确认（仅限 Windows）。",
    },
    {
      key: "chatgpt_base_url",
      type: "string",
      description: "覆盖 ChatGPT 登录流程中使用的基本 URL。",
    },
    {
      key: "cli_auth_credentials_store",
      type: "file | keyring | auto",
      description:
        "控制 CLI 存储缓存凭据的位置（基于文件的 auth.json 与操作系统钥匙串）。",
    },
    {
      key: "mcp_oauth_credentials_store",
      type: "auto | file | keyring",
      description: "MCP OAuth 凭据的首选存储。",
    },
    {
      key: "mcp_oauth_callback_port",
      type: "integer",
      description:
        "MCP OAuth 登录期间使用的本地 HTTP 回调服务器的可选固定端口。未设置时，Codex 会绑定到操作系统选择的临时端口。",
    },
    {
      key: "mcp_oauth_callback_url",
      type: "string",
      description:
        "MCP OAuth 登录的可选重定向 URI 覆盖（例如，devbox 入口 URL）。 `mcp_oauth_callback_port` 仍然控制回调侦听器端口。",
    },
    {
      key: "experimental_use_unified_exec_tool",
      type: "boolean",
      description:
        "用于启用统一执行的旧名称；更喜欢 `[features].unified_exec` 或 `codex --enable unified_exec`。",
    },
    {
      key: "experimental_use_freeform_apply_patch",
      type: "boolean",
      description:
        "用于启用自由格式 apply_patch 的旧名称；更喜欢 `[features].apply_patch_freeform` 或 `codex --enable apply_patch_freeform`。",
    },
    {
      key: "include_apply_patch_tool",
      type: "boolean",
      description:
        "用于启用自由格式 apply_patch 的旧名称；更喜欢 `[features].apply_patch_freeform`。",
    },
    {
      key: "tools.web_search",
      type: "boolean",
      description:
        "已弃用网络搜索的旧版切换开关；更喜欢顶级 `web_search` 设置。",
    },
    {
      key: "web_search",
      type: "disabled | cached | live",
      description:
        'Web 搜索模式（默认：`"cached"`；`cached` 使用 OpenAI 维护的索引，不会抓取实时页面；若使用 `--yolo` 或其他完全访问沙箱设置，默认改为 `"live"`）。使用 `"live"` 从网络抓取最新数据，或使用 `"disabled"` 关闭该工具。',
    },
    {
      key: "projects.<path>.trust_level",
      type: "string",
      description:
        '将项目或 worktree 标记为受信任或不受信任（`"trusted"` | `"untrusted"`）。不受信任项目会跳过项目范围的 `.codex/` 层。',
    },
    {
      key: "notice.hide_full_access_warning",
      type: "boolean",
      description: "跟踪完全访问警告提示的确认。",
    },
    {
      key: "notice.hide_world_writable_warning",
      type: "boolean",
      description:
        "跟踪 Windows 全局可写目录警告的确认。",
    },
    {
      key: "notice.hide_rate_limit_model_nudge",
      type: "boolean",
      description: "跟踪速率限制模型切换提醒的选择退出。",
    },
    {
      key: "notice.hide_gpt5_1_migration_prompt",
      type: "boolean",
      description: "跟踪 GPT-5.1 迁移提示的确认。",
    },
    {
      key: "notice.hide_gpt-5.1-codex-max_migration_prompt",
      type: "boolean",
      description:
        "跟踪 gpt-5.1-codex-max 迁移提示的确认。",
    },
    {
      key: "notice.model_migrations",
      type: "map<string,string>",
      description: "将已确认的模型迁移跟踪为旧->新映射。",
    },
    {
      key: "forced_login_method",
      type: "chatgpt | api",
      description: "将 Codex 限制为特定的身份验证方法。",
    },
    {
      key: "forced_chatgpt_workspace_id",
      type: "string (uuid)",
      description: "将 ChatGPT 登录限制为特定工作区标识符。",
    },
  ]}
  client:load
/>

您可以找到 `config.toml` [此处](https://developers.openai.com/codex/config-schema.json) 的最新 JSON 架构。

要在 VS Code 或 Cursor 中编辑 `config.toml` 时获得自动完成和诊断功能，您可以安装 [更好的 TOML](https://marketplace.visualstudio.com/items?itemName=tamasfe.even-better-toml) 扩展并将此行添加到 `config.toml` 的顶部：

```toml
#:schema https://developers.openai.com/codex/config-schema.json
```

注意：将 `experimental_instructions_file` 重命名为 `model_instructions_file`。 Codex 弃用旧密钥；将现有配置更新为新名称。

## `requirements.toml`

`requirements.toml` 是管理员强制执行的配置文件，它限制用户无法覆盖的安全敏感设置。详情、地点和示例参见[管理员强制要求](https://developers.openai.com/codex/enterprise/managed-configuration#admin-enforced-requirements-requirementstoml)。

对于 ChatGPT 商业和企业用户，Codex 还可以应用云获取
要求。有关优先级详细信息，请参阅安全页面。

<ConfigTable
  options={[
    {
      key: "allowed_approval_policies",
      type: "array<string>",
      description:
        "`approval_policy` 的允许值（例如 `untrusted`、`on-request`、`never` 和 `reject`）。",
    },
    {
      key: "allowed_sandbox_modes",
      type: "array<string>",
      description: "`sandbox_mode` 的允许值。",
    },
    {
      key: "allowed_web_search_modes",
      type: "array<string>",
      description:
        "`web_search` 的允许值（`disabled`、`cached`、`live`）。 `disabled` 始终是允许的；空列表实际上只允许 `disabled`。",
    },
    {
      key: "mcp_servers",
      type: "table",
      description:
        "可以启用的 MCP 服务器的白名单。服务器名称 (`<id>`) 及其标识必须匹配才能启用 MCP 服务器。任何不在允许列表中（或身份不匹配）的已配置 MCP 服务器都将被禁用。",
    },
    {
      key: "mcp_servers.<id>.identity",
      type: "table",
      description:
        "单个 MCP 服务器的身份规则。设置 `command` (stdio) 或 `url` (流式传输 HTTP)。",
    },
    {
      key: "mcp_servers.<id>.identity.command",
      type: "string",
      description:
        "当 MCP stdio 服务器 `mcp_servers.<id>.command` 与此命令匹配时，允许该服务器。",
    },
    {
      key: "mcp_servers.<id>.identity.url",
      type: "string",
      description:
        "当 MCP 可流式 HTTP 服务器 `mcp_servers.<id>.url` 与此 URL 匹配时，允许该服务器。",
    },
    {
      key: "rules",
      type: "table",
      description:
        "管理员强制执行的命令规则与 `.rules` 文件合并。需求规则必须是限制性的。",
    },
    {
      key: "rules.prefix_rules",
      type: "array<table>",
      description:
        "强制执行的前缀规则列表。每条规则必须包含 `pattern` 和 `decision`。",
    },
    {
      key: "rules.prefix_rules[].pattern",
      type: "array<table>",
      description:
        "以模式标记表示的命令前缀。每个令牌设置 `token` 或 `any_of`。",
    },
    {
      key: "rules.prefix_rules[].pattern[].token",
      type: "string",
      description: "此位置有一个文字标记。",
    },
    {
      key: "rules.prefix_rules[].pattern[].any_of",
      type: "array<string>",
      description: "此位置允许的替代令牌的列表。",
    },
    {
      key: "rules.prefix_rules[].decision",
      type: "prompt | forbidden",
      description:
        "必需的。需求规则只能提示或禁止（不允许）。",
    },
    {
      key: "rules.prefix_rules[].justification",
      type: "string",
      description:
        "可选的非空理由出现在批准提示或拒绝消息中。",
    },
  ]}
  client:load
/>

---

## 配置示例

来源文件：`config-sample.zh-CN.md`


使用此示例配置作为起点。它包括 Codex 从 `config.toml` 读取的大多数键，以及默认值和简短注释。

有关说明和指导，请参阅：

- [配置基础知识](https://developers.openai.com/codex/config-basic)
- [高级配置](https://developers.openai.com/codex/config-advanced)
- [配置参考](https://developers.openai.com/codex/config-reference)
- [沙箱与审批](https://developers.openai.com/codex/security#sandbox-and-approvals)
- [托管配置](https://developers.openai.com/codex/security#managed-configuration)

使用下面的代码片段作为参考。仅将您需要的键和部分复制到 `~/.codex/config.toml` （或项目范围的 `.codex/config.toml`）中，然后调整您的设置的值。

```toml
# Codex example configuration (config.toml)
#
# This file lists all keys Codex reads from config.toml, their default values,
# and concise explanations. Values here mirror the effective defaults compiled
# into the CLI. Adjust as needed.
#
# Notes
# - Root keys must appear before tables in TOML.
# - Optional keys that default to "unset" are shown commented out with notes.
# - MCP servers, profiles, and model providers are examples; remove or edit.

################################################################################
# Core Model Selection
################################################################################

# Primary model used by Codex. Default: "gpt-5.2-codex" on all platforms.
model = "gpt-5.2-codex"

# Default communication style for supported models. Default: "friendly".
# Allowed values: none | friendly | pragmatic
# personality = "friendly"

# Optional model override for /review. Default: unset (uses current session model).
# review_model = "gpt-5.2-codex"

# Provider id selected from [model_providers]. Default: "openai".
model_provider = "openai"

# Default OSS provider for --oss sessions. When unset, Codex prompts. Default: unset.
# oss_provider = "ollama"

# Optional manual model metadata. When unset, Codex auto-detects from model.
# Uncomment to force values.
# model_context_window = 128000       # tokens; default: auto for model
# model_auto_compact_token_limit = 0  # tokens; unset uses model defaults
# tool_output_token_limit = 10000     # tokens stored per tool output; default: 10000 for gpt-5.2-codex
# model_catalog_json = "/absolute/path/to/models.json" # optional startup-only model catalog override
# background_terminal_max_timeout = 300000 # ms; max empty write_stdin poll window (default 5m)
# log_dir = "/absolute/path/to/codex-logs" # directory for Codex logs; default: "$CODEX_HOME/log"

################################################################################
# Reasoning & Verbosity (Responses API capable models)
################################################################################

# Reasoning effort: minimal | low | medium | high | xhigh (default: medium; xhigh on gpt-5.2-codex and gpt-5.2)
model_reasoning_effort = "medium"

# Reasoning summary: auto | concise | detailed | none (default: auto)
# model_reasoning_summary = "auto"

# Text verbosity for GPT-5 family (Responses API): low | medium | high (default: medium)
# model_verbosity = "medium"

# Force enable or disable reasoning summaries for current model
# model_supports_reasoning_summaries = true

################################################################################
# Instruction Overrides
################################################################################

# Additional user instructions are injected before AGENTS.md. Default: unset.
# developer_instructions = ""

# (Ignored) Optional legacy base instructions override (prefer AGENTS.md). Default: unset.
# instructions = ""

# Inline override for the history compaction prompt. Default: unset.
# compact_prompt = ""

# Override built-in base instructions with a file path. Default: unset.
# model_instructions_file = "/absolute/or/relative/path/to/instructions.txt"

# Migration note: experimental_instructions_file was renamed to model_instructions_file (deprecated).

# Load the compact prompt override from a file. Default: unset.
# experimental_compact_prompt_file = "/absolute/or/relative/path/to/compact_prompt.txt"

# Legacy name for apply_patch_freeform. Default: false
include_apply_patch_tool = false


################################################################################
# Notifications
################################################################################

# External notifier program (argv array). When unset: disabled.
# Example: notify = ["notify-send", "Codex"]
notify = [ ]


################################################################################
# Approval & Sandbox
################################################################################

# When to ask for command approval:
# - untrusted: only known-safe read-only commands auto-run; others prompt
# - on-request: model decides when to ask (default)
# - never: never prompt (risky)
# - { reject = { ... } }: auto-reject selected prompt categories
approval_policy = "on-request"
# Example granular auto-reject policy:
# approval_policy = { reject = { sandbox_approval = true, rules = false, mcp_elicitations = false } }

# Allow login-shell semantics for shell-based tools when they request `login = true`.
# Default: true. Set false to force non-login shells and reject explicit login-shell requests.
allow_login_shell = true

# Filesystem/network sandbox policy for tool calls:
# - read-only (default)
# - workspace-write
# - danger-full-access (no sandbox; extremely risky)
sandbox_mode = "read-only"

################################################################################
# Authentication & Login
################################################################################

# Where to persist CLI login credentials: file (default) | keyring | auto
cli_auth_credentials_store = "file"

# Base URL for ChatGPT auth flow (not OpenAI API). Default:
chatgpt_base_url = "https://chatgpt.com/backend-api/"

# Restrict ChatGPT login to a specific workspace id. Default: unset.
# forced_chatgpt_workspace_id = ""

# Force login mechanism when Codex would normally auto-select. Default: unset.
# Allowed values: chatgpt | api
# forced_login_method = "chatgpt"

# Preferred store for MCP OAuth credentials: auto (default) | file | keyring
mcp_oauth_credentials_store = "auto"

# Optional fixed port for MCP OAuth callback: 1-65535. Default: unset.
# mcp_oauth_callback_port = 4321

# Optional redirect URI override for MCP OAuth login (for example, remote devbox ingress).
# Custom callback paths are supported. `mcp_oauth_callback_port` still controls the listener port.
# mcp_oauth_callback_url = "https://devbox.example.internal/callback"

################################################################################
# Project Documentation Controls
################################################################################

# Max bytes from AGENTS.md to embed into first-turn instructions. Default: 32768
project_doc_max_bytes = 32768

# Ordered fallbacks when AGENTS.md is missing at a directory level. Default: []
project_doc_fallback_filenames = []

# Project root marker filenames used when searching parent directories. Default: [".git"]
# project_root_markers = [".git"]

################################################################################
# History & File Opener
################################################################################

# URI scheme for clickable citations: vscode (default) | vscode-insiders | windsurf | cursor | none
file_opener = "vscode"

################################################################################
# UI, Notifications, and Misc
################################################################################

# Suppress internal reasoning events from output. Default: false
hide_agent_reasoning = false

# Show raw reasoning content when available. Default: false
show_raw_agent_reasoning = false

# Disable burst-paste detection in the TUI. Default: false
disable_paste_burst = false

# Track Windows onboarding acknowledgement (Windows only). Default: false
windows_wsl_setup_acknowledged = false

# Check for updates on startup. Default: true
check_for_update_on_startup = true

################################################################################
# Web Search
################################################################################

# Web search mode: disabled | cached | live. Default: "cached"
# cached serves results from a web search cache (an OpenAI-maintained index).
# cached returns pre-indexed results; live fetches the most recent data.
# If you use --yolo or another full access sandbox setting, web search defaults to live.
web_search = "cached"

################################################################################
# Profiles (named presets)
################################################################################

# Active profile name. When unset, no profile is applied.
# profile = "default"

################################################################################
# Agents (multi-agent roles and limits)
################################################################################

# [agents]
# Maximum concurrently open agent threads. Default: 6
# max_threads = 6
# Maximum nested spawn depth. Root session starts at depth 0. Default: 1
# max_depth = 1

# [agents.reviewer]
# description = "Find security, correctness, and test risks in code."
# config_file = "./agents/reviewer.toml"  # relative to the config.toml that defines it

################################################################################
# Skills (per-skill overrides)
################################################################################

# Disable or re-enable a specific skill without deleting it.
[[skills.config]]
# path = "/path/to/skill"
# enabled = false

################################################################################
# Experimental toggles (legacy; prefer [features])
################################################################################

experimental_use_unified_exec_tool = false

# Include apply_patch via freeform editing path (affects default tool set). Default: false
experimental_use_freeform_apply_patch = false

################################################################################
# Sandbox settings (tables)
################################################################################

# Extra settings used only when sandbox_mode = "workspace-write".
[sandbox_workspace_write]
# Additional writable roots beyond the workspace (cwd). Default: []
writable_roots = []
# Allow outbound network access inside the sandbox. Default: false
network_access = false
# Exclude $TMPDIR from writable roots. Default: false
exclude_tmpdir_env_var = false
# Exclude /tmp from writable roots. Default: false
exclude_slash_tmp = false

################################################################################
# Shell Environment Policy for spawned processes (table)
################################################################################

[shell_environment_policy]
# inherit: all (default) | core | none
inherit = "all"
# Skip default excludes for names containing KEY/SECRET/TOKEN (case-insensitive). Default: true
ignore_default_excludes = true
# Case-insensitive glob patterns to remove (e.g., "AWS_*", "AZURE_*"). Default: []
exclude = []
# Explicit key/value overrides (always win). Default: {}
set = {}
# Whitelist; if non-empty, keep only matching vars. Default: []
include_only = []
# Experimental: run via user shell profile. Default: false
experimental_use_profile = false

################################################################################
# History (table)
################################################################################

[history]
# save-all (default) | none
persistence = "save-all"
# Maximum bytes for history file; oldest entries are trimmed when exceeded. Example: 5242880
# max_bytes = 0

################################################################################
# UI, Notifications, and Misc (tables)
################################################################################

[tui]
# Desktop notifications from the TUI: boolean or filtered list. Default: true
# Examples: false | ["agent-turn-complete", "approval-requested"]
notifications = false

# Notification mechanism for terminal alerts: auto | osc9 | bel. Default: "auto"
# notification_method = "auto"

# Enables welcome/status/spinner animations. Default: true
animations = true

# Show onboarding tooltips in the welcome screen. Default: true
show_tooltips = true

# Control alternate screen usage (auto skips it in Zellij to preserve scrollback).
# alternate_screen = "auto"

# Ordered list of footer status-line item IDs. When unset, Codex uses:
# ["model-with-reasoning", "context-remaining", "current-dir"].
# Set to [] to hide the footer.
# status_line = ["model", "context-remaining", "git-branch"]

# Syntax-highlighting theme (kebab-case). Use /theme in the TUI to preview and save.
# You can also add custom .tmTheme files under $CODEX_HOME/themes.
# theme = "catppuccin-mocha"

# Control whether users can submit feedback from `/feedback`. Default: true
[feedback]
enabled = true

# In-product notices (mostly set automatically by Codex).
[notice]
# hide_full_access_warning = true
# hide_world_writable_warning = true
# hide_rate_limit_model_nudge = true
# hide_gpt5_1_migration_prompt = true
# "hide_gpt-5.1-codex-max_migration_prompt" = true
# model_migrations = { "gpt-4.1" = "gpt-5.1" }

# Suppress the warning shown when under-development feature flags are enabled.
# suppress_unstable_features_warning = true

################################################################################
# Centralized Feature Flags (preferred)
################################################################################

[features]
# Leave this table empty to accept defaults. Set explicit booleans to opt in/out.
# shell_tool = true
# apps = false
# apps_mcp_gateway = false
# web_search_cached = false
# web_search_request = false
# unified_exec = false
# shell_snapshot = false
# apply_patch_freeform = false
# multi_agent = false
# search_tool = false
# personality = true
# request_rule = true
# collaboration_modes = true
# use_linux_sandbox_bwrap = false
# remote_models = false
# runtime_metrics = false
# powershell_utf8 = true
# child_agents_md = false

################################################################################
# Define MCP servers under this table. Leave empty to disable.
################################################################################

[mcp_servers]

# --- Example: STDIO transport ---
# [mcp_servers.docs]
# enabled = true                       # optional; default true
# required = true                      # optional; fail startup/resume if this server cannot initialize
# command = "docs-server"                 # required
# args = ["--port", "4000"]               # optional
# env = { "API_KEY" = "value" }           # optional key/value pairs copied as-is
# env_vars = ["ANOTHER_SECRET"]            # optional: forward these from the parent env
# cwd = "/path/to/server"                 # optional working directory override
# startup_timeout_sec = 10.0               # optional; default 10.0 seconds
# # startup_timeout_ms = 10000              # optional alias for startup timeout (milliseconds)
# tool_timeout_sec = 60.0                  # optional; default 60.0 seconds
# enabled_tools = ["search", "summarize"]  # optional allow-list
# disabled_tools = ["slow-tool"]           # optional deny-list (applied after allow-list)

# --- Example: Streamable HTTP transport ---
# [mcp_servers.github]
# enabled = true                          # optional; default true
# required = true                         # optional; fail startup/resume if this server cannot initialize
# url = "https://github-mcp.example.com/mcp"  # required
# bearer_token_env_var = "GITHUB_TOKEN"        # optional; Authorization: Bearer <token>
# http_headers = { "X-Example" = "value" }    # optional static headers
# env_http_headers = { "X-Auth" = "AUTH_ENV" } # optional headers populated from env vars
# startup_timeout_sec = 10.0                   # optional
# tool_timeout_sec = 60.0                      # optional
# enabled_tools = ["list_issues"]             # optional allow-list

################################################################################
# Model Providers
################################################################################

# Built-ins include:
# - openai (Responses API; requires login or OPENAI_API_KEY via auth flow)
# - oss (Chat Completions API; defaults to http://localhost:11434/v1)

[model_providers]

# --- Example: OpenAI data residency with explicit base URL or headers ---
# [model_providers.openaidr]
# name = "OpenAI Data Residency"
# base_url = "https://us.api.openai.com/v1"        # example with 'us' domain prefix
# wire_api = "responses"                           # "responses" | "chat" (default varies)
# # requires_openai_auth = true                    # built-in OpenAI defaults to true
# # request_max_retries = 4                        # default 4; max 100
# # stream_max_retries = 5                         # default 5;  max 100
# # stream_idle_timeout_ms = 300000                # default 300_000 (5m)
# # experimental_bearer_token = "sk-example"       # optional dev-only direct bearer token
# # http_headers = { "X-Example" = "value" }
# # env_http_headers = { "OpenAI-Organization" = "OPENAI_ORGANIZATION", "OpenAI-Project" = "OPENAI_PROJECT" }

# --- Example: Azure (Chat/Responses depending on endpoint) ---
# [model_providers.azure]
# name = "Azure"
# base_url = "https://YOUR_PROJECT_NAME.openai.azure.com/openai"
# wire_api = "responses"                          # or "chat" per endpoint
# query_params = { api-version = "2025-04-01-preview" }
# env_key = "AZURE_OPENAI_API_KEY"
# # env_key_instructions = "Set AZURE_OPENAI_API_KEY in your environment"

# --- Example: Local OSS (e.g., Ollama-compatible) ---
# [model_providers.ollama]
# name = "Ollama"
# base_url = "http://localhost:11434/v1"
# wire_api = "chat"

################################################################################
# Profiles (named presets)
################################################################################

[profiles]

# [profiles.default]
# model = "gpt-5.2-codex"
# model_provider = "openai"
# approval_policy = "on-request"
# sandbox_mode = "read-only"
# oss_provider = "ollama"
# model_reasoning_effort = "medium"
# model_reasoning_summary = "auto"
# model_verbosity = "medium"
# personality = "friendly" # or "pragmatic" or "none"
# chatgpt_base_url = "https://chatgpt.com/backend-api/"
# model_catalog_json = "./models.json"
# experimental_compact_prompt_file = "./compact_prompt.txt"
# include_apply_patch_tool = false
# experimental_use_unified_exec_tool = false
# experimental_use_freeform_apply_patch = false
# tools.web_search = false             # deprecated legacy alias; prefer top-level `web_search`
# features = { unified_exec = false }

################################################################################
# Apps / Connectors
################################################################################

# Optional per-app controls.
[apps]
# [_default] applies to all apps unless overridden per app.
# [apps._default]
# enabled = true
# destructive_enabled = true
# open_world_enabled = true
#
# [apps.google_drive]
# enabled = false
# destructive_enabled = false            # block destructive-hint tools for this app
# default_tools_enabled = true
# default_tools_approval_mode = "prompt" # auto | prompt | approve
#
# [apps.google_drive.tools."files/delete"]
# enabled = false
# approval_mode = "approve"

################################################################################
# Projects (trust levels)
################################################################################

# Mark specific worktrees as trusted or untrusted.
[projects]
# [projects."/absolute/path/to/project"]
# trust_level = "trusted"  # or "untrusted"

################################################################################
# OpenTelemetry (OTEL) - disabled by default
################################################################################

[otel]
# Include user prompt text in logs. Default: false
log_user_prompt = false
# Environment label applied to telemetry. Default: "dev"
environment = "dev"
# Exporter: none (default) | otlp-http | otlp-grpc
exporter = "none"
# Trace exporter: none (default) | otlp-http | otlp-grpc
trace_exporter = "none"

# Example OTLP/HTTP exporter configuration
# [otel.exporter."otlp-http"]
# endpoint = "https://otel.example.com/v1/logs"
# protocol = "binary"                         # "binary" | "json"

# [otel.exporter."otlp-http".headers]
# "x-otlp-api-key" = "${OTLP_TOKEN}"

# Example OTLP/gRPC exporter configuration
# [otel.exporter."otlp-grpc"]
# endpoint = "https://otel.example.com:4317",
# headers = { "x-otlp-meta" = "abc123" }

# Example OTLP exporter with mutual TLS
# [otel.exporter."otlp-http"]
# endpoint = "https://otel.example.com/v1/logs"
# protocol = "binary"

# [otel.exporter."otlp-http".headers]
# "x-otlp-api-key" = "${OTLP_TOKEN}"

# [otel.exporter."otlp-http".tls]
# ca-certificate = "certs/otel-ca.pem"
# client-certificate = "/etc/codex/certs/client.pem"
# client-private-key = "/etc/codex/certs/client-key.pem"
```

################################################################################

# Windows

################################################################################

[windows]

# Native Windows sandbox mode (Windows only): unelevated | elevated

sandbox = "unelevated"
