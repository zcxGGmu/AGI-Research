# 多代理

> 原文链接：https://developers.openai.com/codex/multi-agent

Codex 可以通过并行生成专用代理，然后在一个响应中收集其结果来运行多代理工作流程。这对于高度并行的复杂任务特别有帮助，例如代码库探索或实施多步骤功能计划。

通过多代理工作流程，您还可以根据代理的不同模型配置和说明来定义自己的代理集。

有关多代理工作流程背后的概念和权衡（包括上下文污染/上下文腐烂和模型选择指导），请参阅[多代理概念](https://developers.openai.com/codex/concepts/multi-agents)。

## 启用多代理

多代理工作流程目前处于实验阶段，需要明确启用。

你可以在 CLI 中使用 `/experimental` 启用该功能。启用
**Multi-agents** 后，重启 Codex。

当前多代理活动主要显示在 CLI 中。
其他界面（Codex App 和 IDE 扩展）的可视化支持即将推出。

您还可以将 [`multi_agent` 功能标志](https://developers.openai.com/codex/config-basic#feature-flags) 直接添加到您的配置文件 (`~/.codex/config.toml`) 中：

```toml
[features]
multi_agent = true
```

## 典型工作流程

Codex 处理跨代理的编排，包括生成新的子代理、路由后续指令、等待结果和关闭代理线程。

当许多代理正在运行时，Codex 会等待，直到所有请求的结果都可用，然后返回合并的响应。

Codex 将自动决定何时生成新代理，或者您可以明确要求它这样做。

对于长时间运行的命令或轮询工作流程，Codex 还可以使用内置 `monitor` 角色，该角色针对等待和重复状态检查进行了调整。

要查看其实际效果，请在您的项目中尝试以下提示：

```text
I would like to review the following points on the current PR (this branch vs main). Spawn one agent per point, wait for all of them, and summarize the result for each point.
1. Security issue
2. Code quality
3. Bugs
4. Race
5. Test flakiness
6. Maintainability of the code
```

## 管理子代理

- 在 CLI 中使用 `/agent` 在活动代理线程之间切换并检查正在进行的线程。
- 直接要求 Codex 引导、停止正在运行的子代理或关闭已完成的代理线程。
- `wait` 工具支持长轮询窗口来监控工作流程（每次调用最多 1 小时）。

## 审批和沙箱控制

子代理会继承当前沙箱策略，但审批模式是非交互式的。
如果子代理尝试执行需要新审批的动作，该动作会失败，错误会回传到父工作流。

您还可以覆盖单个[代理角色](#agent-roles)的沙箱配置，例如明确标记代理以只读模式工作。

## 代理角色

您可以在 [配置](https://developers.openai.com/codex/config-basic#configuration-precedence) 的 `[agents]` 部分中配置代理角色。

代理角色可以在本地配置（通常为 `~/.codex/config.toml`）中定义，也可以在特定于项目的 `.codex/config.toml` 中共享。

每个角色都可以为 Codex 何时应使用此代理提供指导 (`description`)，并可选择加载
当 Codex 生成具有该角色的代理时，角色特定的配置文件 (`config_file`)。

Codex 附带内置角色：

- `default`：通用兜底角色。
- `worker`：以执行为中心的角色，用于实施和修复。
- `explorer`：大量读取的代码库探索角色。
- `monitor`：长时间运行的命令/任务监视角色（针对等待/轮询进行了优化）。

每个代理角色都可以覆盖您的默认配置。代理角色要覆盖的常见设置有：

- `model` 和 `model_reasoning_effort` 为您的代理角色选择特定模型
- `sandbox_mode` 将代理标记为 `read-only`
- `developer_instructions` 向代理角色提供附加指令，而不依赖父代理传递这些指令

### Schema

| 字段                       | 类型          | 必填 | 用途                                                                       |
| --------------------------- | ------------- | :------: | ----------------------------------------------------------------------------- |
| `agents.max_threads`        | `number`      | 否 | 同时打开的代理线程最大数量。                            |
| `agents.max_depth`          | `number`      | 否 | 代理线程最大嵌套深度（根会话从 0 开始）。   |
| `[agents.<name>]`           | `table`       | 否 | 声明一个角色。生成代理时，`<name>` 作为 `agent_type`。 |
| `agents.<name>.description` | `string`      | 否 | 该角色的说明，帮助 Codex 决定何时使用该角色。  |
| `agents.<name>.config_file` | `string(path)` | 否 | 应用于该角色生成代理的 TOML 配置层路径。          |

**说明：**

- `[agents.<name>]` 中的未知字段将被拒绝。
- `agents.max_depth` 默认为 `1`，它允许直接子代理生成，但会阻止更深层次的嵌套。
- 相对 `config_file` 路径是相对于定义角色的 `config.toml` 文件进行解析的。
- `agents.<name>.config_file` 会在配置加载时进行校验，且必须指向已存在的文件。
- 如果角色名称与内置角色匹配（例如 `explorer`），则您的用户定义角色优先。
- 如果 Codex 无法加载角色配置文件，则代理生成可能会失败，直到您修复该文件。
- 代理角色未设置的任何配置都将从父会话继承。

### 代理角色示例

最好的角色定义是狭隘的和固执己见的。为每个角色提供一项明确的工作、与该工作相匹配的工具界面以及防止其漂移到相邻工作的说明。

#### 示例 1：PR 评审团队

该模式将审核分为三个重点角色：

- `explorer` 映射代码库并收集证据。
- `reviewer` 寻找正确性、安全性和测试风险。
- `docs_researcher` 通过专用 MCP 服务器检查框架或 API 文档。

项目配置（`.codex/config.toml`）：

```toml
[agents]
max_threads = 6
max_depth = 1

[agents.explorer]
description = "Read-only codebase explorer for gathering evidence before changes are proposed."
config_file = "agents/explorer.toml"

[agents.reviewer]
description = "PR reviewer focused on correctness, security, and missing tests."
config_file = "agents/reviewer.toml"

[agents.docs_researcher]
description = "Documentation specialist that uses the docs MCP server to verify APIs and framework behavior."
config_file = "agents/docs-researcher.toml"
```

`agents/explorer.toml`：

```toml
model = "gpt-5.3-codex-spark"
model_reasoning_effort = "medium"
sandbox_mode = "read-only"
developer_instructions = """
Stay in exploration mode.
Trace the real execution path, cite files and symbols, and avoid proposing fixes unless the parent agent asks for them.
Prefer fast search and targeted file reads over broad scans.
"""
```

`agents/reviewer.toml`：

```toml
model = "gpt-5.3-codex"
model_reasoning_effort = "high"
sandbox_mode = "read-only"
developer_instructions = """
Review code like an owner.
Prioritize correctness, security, behavior regressions, and missing test coverage.
Lead with concrete findings, include reproduction steps when possible, and avoid style-only comments unless they hide a real bug.
"""
```

`agents/docs-researcher.toml`：

```toml
model = "gpt-5.3-codex-spark"
model_reasoning_effort = "medium"
sandbox_mode = "read-only"
developer_instructions = """
Use the docs MCP server to confirm APIs, options, and version-specific behavior.
Return concise answers with links or exact references when available.
Do not make code changes.
"""

[mcp_servers.openaiDeveloperDocs]
url = "https://developers.openai.com/mcp"
```

此设置适用于以下提示：

```text
Review this branch against main. Have explorer map the affected code paths, reviewer find real risks, and docs_researcher verify the framework APIs that the patch relies on.
```

#### 示例2：前端集成调试团队

此模式对于 UI 回归、不稳定的浏览器流程或跨应用程序代码和正在运行的产品的集成错误非常有用。

项目配置（`.codex/config.toml`）：

```toml
[agents]
max_threads = 6
max_depth = 1

[agents.explorer]
description = "Read-only codebase explorer for locating the relevant frontend and backend code paths."
config_file = "agents/explorer.toml"

[agents.browser_debugger]
description = "UI debugger that uses browser tooling to reproduce issues and capture evidence."
config_file = "agents/browser-debugger.toml"

[agents.worker]
description = "Implementation-focused agent for small, targeted fixes after the issue is understood."
config_file = "agents/worker.toml"
```

`agents/explorer.toml`：

```toml
model = "gpt-5.3-codex-spark"
model_reasoning_effort = "medium"
sandbox_mode = "read-only"
developer_instructions = """
Map the code that owns the failing UI flow.
Identify entry points, state transitions, and likely files before the worker starts editing.
"""
```

`agents/browser-debugger.toml`：

```toml
model = "gpt-5.3-codex"
model_reasoning_effort = "high"
sandbox_mode = "workspace-write"
developer_instructions = """
Reproduce the issue in the browser, capture exact steps, and report what the UI actually does.
Use browser tooling for screenshots, console output, and network evidence.
Do not edit application code.
"""

[mcp_servers.chrome_devtools]
url = "http://localhost:3000/mcp"
startup_timeout_sec = 20
```

`agents/worker.toml`：

```toml
model = "gpt-5.3-codex"
model_reasoning_effort = "medium"
developer_instructions = """
Own the fix once the issue is reproduced.
Make the smallest defensible change, keep unrelated files untouched, and validate only the behavior you changed.
"""

[[skills.config]]
path = "/Users/me/.agents/skills/docs-editor/SKILL.md"
enabled = false
```

此设置适用于以下提示：

```text
Investigate why the settings modal fails to save. Have browser_debugger reproduce it, explorer trace the responsible code path, and worker implement the smallest fix once the failure mode is clear.
```
