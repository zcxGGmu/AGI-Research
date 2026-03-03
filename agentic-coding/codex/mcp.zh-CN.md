# 模型上下文协议

> 原文链接：https://developers.openai.com/codex/mcp

模型上下文协议 (MCP) 将模型连接到工具和上下文。使用它可以让 Codex 访问第三方文档，或者让它与浏览器或 Figma 等开发人员工具进行交互。

Codex 在 CLI 和 IDE 扩展中支持 MCP 服务器。

## 支持的 MCP 功能

- **STDIO 服务器**：作为本地进程运行的服务器（由命令启动）。
  - 环境变量
- **可流式 HTTP 服务器**：通过 URL 访问的服务器。
  - Bearer Token 身份验证
  - OAuth 身份验证（对支持 OAuth 的服务器运行 `codex mcp login <server-name>`）

## 将 Codex 连接到 MCP 服务器

Codex 将 MCP 配置与其他 Codex 配置设置一起存储在 `config.toml` 中。默认情况下，这是 `~/.codex/config.toml`，但您也可以将 MCP 服务器范围限定为具有 `.codex/config.toml` 的项目（仅限受信任的项目）。

CLI 和 IDE 扩展共享此配置。配置 MCP 服务器后，您可以在两个 Codex 客户端之间切换，而无需重新设置。

要配置 MCP 服务器，请选择一个选项：

1. **使用 CLI**：运行 `codex mcp` 添加和管理服务器。
2. **编辑 `config.toml`**：直接更新 `~/.codex/config.toml` （或受信任项目中的项目范围 `.codex/config.toml`）。

### 使用 CLI 配置

#### 添加 MCP 服务器

```bash
codex mcp add <server-name> --env VAR1=VALUE1 --env VAR2=VALUE2 -- <stdio server-command>
```

例如，要添加 Context7（用于开发人员文档的免费 MCP 服务器），您可以运行以下命令：

```bash
codex mcp add context7 -- npx -y @upstash/context7-mcp
```

#### 其他 CLI 命令

要查看所有可用的 MCP 命令，您可以运行 `codex mcp --help`。

#### 终端用户界面 (TUI)

在 `codex` TUI 中，使用 `/mcp` 查看您的活动 MCP 服务器。

### 使用 config.toml 进行配置

要对 MCP 服务器选项进行更细粒度的控制，请编辑 `~/.codex/config.toml` （或项目范围的 `.codex/config.toml`）。在 IDE 扩展中，从齿轮菜单中选择 **MCP 设置** > **打开 config.toml**。

在配置文件中使用 `[mcp_servers.<server-name>]` 表配置每个 MCP 服务器。

#### STDIO 服务器

- `command`（必需）：启动服务器的命令。
- `args` （可选）：传递给服务器的参数。
- `env` （可选）：为服务器设置的环境变量。
- `env_vars` （可选）：允许和转发的环境变量。
- `cwd` （可选）：启动服务器的工作目录。

#### 可流式传输的 HTTP 服务器

- `url`（必填）：服务器地址。
- `bearer_token_env_var`（可选）：用于提供 Bearer Token 的环境变量名，值会通过 `Authorization` 头发送。
- `http_headers` （可选）：标头名称到静态值的映射。
- `env_http_headers` （可选）：标头名称到环境变量名称（从环境中提取的值）的映射。

#### 其他配置选项

- `startup_timeout_sec`（可选）：服务器启动超时（秒）。默认值：`10`。
- `tool_timeout_sec`（可选）：服务器运行工具的超时（秒）。默认值：`60`。
- `enabled`（可选）：设置 `false` 以禁用服务器而不删除它。
- `required`（可选）：设置 `true` 以使启用的服务器无法初始化时启动失败。
- `enabled_tools`（可选）：工具允许列表。
- `disabled_tools`（可选）：工具拒绝列表（在 `enabled_tools` 之后应用）。

如果您的 OAuth 提供程序需要固定回调端口，请在 `config.toml` 中设置顶级 `mcp_oauth_callback_port`。如果未设置，Codex 会绑定到临时端口。

如果您的 MCP OAuth 流必须使用特定回调 URL（例如，远程 devbox 入口 URL 或自定义回调路径），请设置 `mcp_oauth_callback_url`。 Codex 使用该值作为 OAuth `redirect_uri`，同时仍使用 `mcp_oauth_callback_port` 作为回调侦听器端口。本地回调 URL（例如 `localhost`）在环回上绑定；非本地回调 URL 绑定在 `0.0.0.0` 上，以便回调可以到达主机。

#### config.toml 示例

```toml
[mcp_servers.context7]
command = "npx"
args = ["-y", "@upstash/context7-mcp"]

[mcp_servers.context7.env]
MY_ENV_VAR = "MY_ENV_VALUE"
```

```toml
# Optional MCP OAuth callback overrides (used by `codex mcp login`)
mcp_oauth_callback_port = 5555
mcp_oauth_callback_url = "https://devbox.example.internal/callback"
```

```toml
[mcp_servers.figma]
url = "https://mcp.figma.com/mcp"
bearer_token_env_var = "FIGMA_OAUTH_TOKEN"
http_headers = { "X-Figma-Region" = "us-east-1" }
```

```toml
[mcp_servers.chrome_devtools]
url = "http://localhost:3000/mcp"
enabled_tools = ["open", "screenshot"]
disabled_tools = ["screenshot"] # applied after enabled_tools
startup_timeout_sec = 20
tool_timeout_sec = 45
enabled = true
```

## 有用的 MCP 服务器示例

MCP 服务器列表不断增长。以下是一些常见的：

- [OpenAI 文档 MCP](https://developers.openai.com/resources/docs-mcp)：搜索和阅读 OpenAI 开发人员文档。
- [Context7](https://github.com/upstash/context7)：连接到最新开发者文档。
- Figma [Local](https://developers.figma.com/docs/figma-mcp-server/local-server-installation/) 与 [Remote](https://developers.figma.com/docs/figma-mcp-server/remote-server-installation/)：访问你的 Figma 设计。
- [剧作家](https://www.npmjs.com/package/@playwright/mcp)：使用 Playwright 控制和检查浏览器。
- [Chrome 开发者工具](https://github.com/ChromeDevTools/chrome-devtools-mcp/)：控制和检查 Chrome。
- [哨兵](https://docs.sentry.io/product/sentry-mcp/#codex)：访问哨兵日志。
- [GitHub](https://github.com/github/github-mcp-server)：管理超出 `git` 支持范围的 GitHub（例如，拉取请求和问题）。
