# 代理技能

> 原文链接：https://developers.openai.com/codex/skills

使用代理技能扩展 Codex 的任务能力。一个技能会打包说明、资源和可选脚本，让 Codex 能更稳定地执行工作流。你可以在团队内共享技能，也可以与社区共享。技能基于[开放 Agent Skills 标准](https://agentskills.io)。

Codex CLI、IDE 扩展和 Codex 应用程序中提供了技能。

技能使用**渐进式披露**来高效管理上下文：Codex 先读取每个技能的元数据（`name`、`description`、文件路径，以及来自 `agents/openai.yaml` 的可选元数据）。只有在决定使用该技能时，Codex 才会加载完整 `SKILL.md` 指令。

技能是一个包含 `SKILL.md` 文件以及可选脚本和引用的目录。 `SKILL.md` 文件必须包含 `name` 和 `description`。

<FileTree
  class="mt-4"
  tree={[
    {
      name: "my-skill/",
      open: true,
      children: [
        {
          name: "SKILL.md",
          comment: "Required: instructions + metadata",
        },
        {
          name: "scripts/",
          comment: "Optional: executable code",
        },
        {
          name: "references/",
          comment: "Optional: documentation",
        },
        {
          name: "assets/",
          comment: "Optional: templates, resources",
        },
        {
          name: "agents/",
          open: true,
          children: [
            {
              name: "openai.yaml",
              comment: "Optional: appearance and dependencies",
            },
          ],
        },
      ],
    },

]}
/>

## Codex 如何使用技能

Codex 可以通过两种方式激活技能：

1. **显式调用：** 直接在提示中包含技能。在 CLI/IDE 中，运行 `/skills` 或输入 `$` 来提及技能。
2. **隐式调用：** 当您的任务与技能 `description` 匹配时，Codex 可以选择一项技能。

由于隐式匹配依赖于 `description`，因此请编写具有明确范围和边界的描述。

## 创建技能

首先使用内置的创建器：

```text
$skill-creator
```

创建器会询问该技能做什么、何时触发，以及应该是纯指令型还是包含脚本。默认是纯指令型。

您还可以通过创建包含 `SKILL.md` 文件的文件夹来手动创建技能：

```md
---
name: skill-name
description: Explain exactly when this skill should and should not trigger.
---

Skill instructions for Codex to follow.
```

Codex 自动检测技能变化。如果未出现更新，请重新启动 Codex。

## 技能保存在哪里

Codex 从存储库、用户、管理员和系统位置读取技能。对于存储库，Codex 会扫描从当前工作目录到存储库根目录的每个目录中的 `.agents/skills`。如果两个技能共享相同的 `name`，Codex 不会合并它们；两者都可以出现在技能选择器中。

| 技能范围 | 位置                                                                                                  | 建议用途                                                                                                                                                                                        |
| :---------- | :-------------------------------------------------------------------------------------------------------- | :--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `REPO`      | `$CWD/.agents/skills` <br /> 当前工作目录：启动 Codex 的位置。                           | 如果您处于存储库或代码环境中，团队可以签入与工作文件夹相关的技能。例如，仅与微服务或模块相关的技能。                              |
| `REPO`      | `$CWD/../.agents/skills` <br /> 当您在 Git 存储库中启动 Codex 时，CWD 上方的文件夹。         | 如果您位于包含嵌套文件夹的存储库中，组织可以签入与父文件夹中的共享区域相关的技能。                                                                       |
| `REPO`      | `$REPO_ROOT/.agents/skills` <br /> 当您在 Git 存储库中启动 Codex 时最顶层的根文件夹。 | 如果您位于包含嵌套文件夹的存储库中，组织可以签入与使用该存储库的每个人相关的技能。这些作为根技能可用于存储库中的任何子文件夹。 |
| `USER`      | `$HOME/.agents/skills` <br /> 签入用户个人文件夹中的任何技能。                         | 用于管理与用户相关的技能，这些技能适用于用户可能使用的任何存储库。                                                                                                           |
| `ADMIN`     | `/etc/codex/skills` <br /> 在共享系统位置签入机器或容器的任何技能。 | 用于 SDK 脚本、自动化以及检查计算机上每个用户可用的默认管理技能。                                                                                     |
| `SYSTEM`    | 由 OpenAI 与 Codex 捆绑在一起。                                                                             | 与广泛受众相关的有用技能，例如技能创建者和计划技能。每个人在启动 Codex 时都可以使用。                                                                   |

Codex 支持符号链接的技能文件夹，并在扫描这些位置时遵循符号链接目标。

## 安装技能

要安装内置技能之外的技能，请使用 `$skill-installer`：

```bash
$skill-installer install the linear skill from the .experimental folder
```

您还可以提示安装程序从其他存储库下载技能。 Codex 自动检测新安装的技能；如果没有出现，请重新启动 Codex。

## 启用或禁用技能

使用 `~/.codex/config.toml` 中的 `[[skills.config]]` 条目禁用技能而不删除它：

```toml
[[skills.config]]
path = "/path/to/skill/SKILL.md"
enabled = false
```

更改 `~/.codex/config.toml` 后重新启动 Codex。

## 可选元数据

添加 `agents/openai.yaml` 以在 [Codex 应用程序](https://developers.openai.com/codex/app) 中配置 UI 元数据、设置调用策略并声明工具依赖项，以获得更无缝的技能使用体验。

```yaml
interface:
  display_name: "Optional user-facing name"
  short_description: "Optional user-facing description"
  icon_small: "./assets/small-logo.svg"
  icon_large: "./assets/large-logo.png"
  brand_color: "#3B82F6"
  default_prompt: "Optional surrounding prompt to use the skill with"

policy:
  allow_implicit_invocation: false

dependencies:
  tools:
    - type: "mcp"
      value: "openaiDeveloperDocs"
      description: "OpenAI Docs MCP server"
      transport: "streamable_http"
      url: "https://developers.openai.com/mcp"
```

`allow_implicit_invocation`（默认：`true`）：当其为 `false` 时，Codex 不会根据用户提示隐式调用该技能；显式 `$skill` 调用仍然有效。

## 最佳实践

- 让每项技能都专注于一项工作。
- 优先使用指令而不是脚本，除非您需要确定性行为或外部工具。
- 编写具有明确输入和输出的命令式步骤。
- 根据技能描述测试提示以确认正确的触发行为。

有关更多示例，请参阅 [github.com/openai/skills](https://github.com/openai/skills) 和 [Agent Skills 规范](https://agentskills.io/specification)。
