# 使用 AGENTS.md 的自定义指令

> 原文链接：https://developers.openai.com/codex/guides/agents-md

Codex 在执行任何工作之前会读取 `AGENTS.md` 文件。通过将全局指导与特定于项目的覆盖进行分层，无论您打开哪个存储库，您都可以以一致的期望开始每项任务。

## Codex 如何发现指导

Codex 在启动时构建指令链（每次运行一次；在 TUI 中通常是每次启动会话时一次）。发现流程遵循以下优先顺序：

1. **全局范围：** 在你的 Codex 主目录中（默认为 `~/.codex`，除非你设置 `CODEX_HOME`），Codex 会读取 `AGENTS.override.md`（如果存在）。否则，Codex 会读取 `AGENTS.md`。Codex 仅使用该层级中的第一个非空文件。
2. **项目范围：** 从项目根目录（通常是 Git 根目录）开始，Codex 会向下移动到您当前的工作目录。如果 Codex 找不到项目根目录，它只会检查当前目录。在路径上的每个目录中，它会检查 `AGENTS.override.md`，然后检查 `AGENTS.md`，然后检查 `project_doc_fallback_filenames` 中的任何后备名称。 Codex 每个目录最多包含一个文件。
3. **合并顺序：** Codex 从根向下连接文件，用空行将它们连接起来。更接近当前目录的文件会覆盖之前的指导，因为它们稍后出现在组合提示中。

一旦组合大小达到 `project_doc_max_bytes` 定义的限制（默认为 32 KiB），Codex 就会跳过空文件并停止继续添加。有关这些配置项的详细信息，请参阅[项目指令发现](https://developers.openai.com/codex/config-advanced#project-instructions-discovery)。达到上限时，可以提高限制，或将指令拆分到多级嵌套目录中。

## 创建全局指导

在您的 Codex 主目录中创建持久默认值，以便每个存储库都继承您的工作协议。

1. 确保目录存在：

   ```bash
   mkdir -p ~/.codex
   ```

2. 创建具有可重用首选项的 `~/.codex/AGENTS.md`：

   ```md
   # ~/.codex/AGENTS.md

   ## Working agreements

   - Always run `npm test` after modifying JavaScript files.
   - Prefer `pnpm` when installing dependencies.
   - Ask for confirmation before adding new production dependencies.
   ```

3. 在任何地方运行 Codex 以确认它加载文件：

   ```bash
   codex --ask-for-approval never "Summarize the current instructions."
   ```

预期：Codex 在开始提出工作方案前，会引用 `~/.codex/AGENTS.md` 中的条目。

当您需要临时全局覆盖而不删除基本文件时，请使用 `~/.codex/AGENTS.override.md` 。删除覆盖以恢复共享指导。

## 分层项目说明

仓库级文件可以让 Codex 了解项目约定，同时继承你的全局默认值。

1. 在您的存储库根目录中，添加涵盖基本设置的 `AGENTS.md` ：

   ```md
   # AGENTS.md

   ## Repository expectations

   - Run `npm run lint` before opening a pull request.
   - Document public utilities in `docs/` when you change behavior.
   ```

2. 当特定团队需要不同的规则时，在嵌套目录中添加覆盖。例如，在 `services/payments/` 内部创建 `AGENTS.override.md`：

   ```md
   # services/payments/AGENTS.override.md

   ## Payments service rules

   - Use `make test-payments` instead of `npm test`.
   - Never rotate API keys without notifying the security channel.
   ```

3. 从付款目录启动 Codex：

   ```bash
   codex --cd services/payments --ask-for-approval never "List the instruction sources you loaded."
   ```

预期：Codex 会先报告全局文件，再报告仓库根目录的 `AGENTS.md`，最后报告 payments 的覆盖文件。

Codex 一旦到达当前目录就会停止搜索，因此将覆盖尽可能靠近专门的工作。

以下是添加全局文件和特定于付款的覆盖后的示例存储库：

<FileTree
  class="mt-4"
  tree={[
    {
      name: "AGENTS.md",
      comment: "Repository expectations",
      highlight: true,
    },
    {
      name: "services/",
      open: true,
      children: [
        {
          name: "payments/",
          open: true,
          children: [
            {
              name: "AGENTS.md",
              comment: "Ignored because an override exists",
            },
            {
              name: "AGENTS.override.md",
              comment: "Payments service rules",
              highlight: true,
            },
            { name: "README.md" },
          ],
        },
        {
          name: "search/",
          children: [{ name: "AGENTS.md" }, { name: "…", placeholder: true }],
        },
      ],
    },
  ]}
/>

## 自定义后备文件名

如果您的存储库已使用不同的文件名（例如 `TEAM_GUIDE.md`），请将其添加到后备列表中，以便 Codex 将其视为指令文件。

1. 编辑您的 Codex 配置：

   ```toml
   # ~/.codex/config.toml
   project_doc_fallback_filenames = ["TEAM_GUIDE.md", ".agents.md"]
   project_doc_max_bytes = 65536
   ```

2. 重新启动 Codex 或运行新命令以便加载更新的配置。

现在 Codex 按以下顺序检查每个目录：`AGENTS.override.md`、`AGENTS.md`、`TEAM_GUIDE.md`、`.agents.md`。指令发现时将忽略不在此列表中的文件名。较大的字节限制允许在截断之前进行更多组合指导。

后备列表到位后，Codex 将备用文件视为指令：

<FileTree
  class="mt-4"
  tree={[
    {
      name: "TEAM_GUIDE.md",
      comment: "Detected via fallback list",
      highlight: true,
    },
    {
      name: ".agents.md",
      comment: "Fallback file in root",
    },
    {
      name: "support/",
      open: true,
      children: [
        {
          name: "AGENTS.override.md",
          comment: "Overrides fallback guidance",
          highlight: true,
        },
        {
          name: "playbooks/",
          children: [{ name: "…", placeholder: true }],
        },
      ],
    },
  ]}
/>

当您需要不同的配置文件（例如特定于项目的自动化用户）时，设置 `CODEX_HOME` 环境变量：

```bash
CODEX_HOME=$(pwd)/.codex codex exec "List active instruction sources"
```

预期：输出列出与自定义 `.codex` 目录相关的文件。

## 验证你的设置

- 在仓库根目录运行 `codex --ask-for-approval never "Summarize the current instructions."`。Codex 应按优先顺序复述全局文件和项目文件中的指导内容。
- 使用 `codex --cd subdir --ask-for-approval never "Show which instruction files are active."` 确认嵌套覆盖替换更广泛的规则。
- 如果您需要审核 Codex 加载的指令文件，请在会话后检查 `~/.codex/log/codex-tui.log`（如果启用了会话日志记录，则检查最新的 `session-*.jsonl` 文件）。
- 如果指令看起来过时，请在目标目录中重新启动 Codex。 Codex 在每次运行时（以及在每个 TUI 会话开始时）都会重建指令链，因此无需手动清除缓存。

## 解决发现问题

- **未加载任何内容：** 先确认你位于目标仓库中，且 `codex status` 报告的是你期望的工作区根目录。还要确保说明文件非空；Codex 会忽略空文件。
- **出现错误的指导：** 在目录树或 Codex 主目录下查找更高的 `AGENTS.override.md`。重命名或删除覆盖以回退到常规文件。
- **Codex 忽略后备名称：** 确认您在 `project_doc_fallback_filenames` 中列出的名称没有拼写错误，然后重新启动 Codex 以使更新的配置生效。
- **指令被截断：** 提高 `project_doc_max_bytes` 或跨嵌套目录分割大文件，以保持关键指导的完整性。
- **配置文件混乱：** 在启动 Codex 之前运行 `echo $CODEX_HOME`。非默认值将 Codex 指向与您编辑的主目录不同的主目录。

## 后续步骤

- 访问官方 [代理.md](https://agents.md) 网站了解更多信息。
- 查看 [提示 Codex](https://developers.openai.com/codex/prompting)，了解与持续指导搭配使用的对话模式。
