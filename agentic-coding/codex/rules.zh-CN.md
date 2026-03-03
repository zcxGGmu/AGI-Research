# 规则

> 原文链接：https://developers.openai.com/codex/rules

使用规则来控制 Codex 可以在沙箱外运行哪些命令。

规则是实验性的，可能会发生变化。

## 创建规则文件

1. 在 `./codex/rules/` 下创建 `.rules` 文件（例如 `~/.codex/rules/default.rules`）。
2. 添加规则。此示例在允许 `gh pr view` 在沙箱外运行之前进行提示。

   ```python
   # Prompt before running commands with the prefix `gh pr view` outside the sandbox.
   prefix_rule(
       # The prefix to match.
       pattern = ["gh", "pr", "view"],

       # The action to take when Codex requests to run a matching command.
       decision = "prompt",

       # Optional rationale for why this rule exists.
       justification = "Viewing PRs is allowed with approval",

       # `match` and `not_match` are optional "inline unit tests" where you can
       # provide examples of commands that should (or should not) match this rule.
       match = [
           "gh pr view 7888",
           "gh pr view --repo openai/codex",
           "gh pr view 7888 --json title,body,comments",
       ],
       not_match = [
           # Does not match because the `pattern` must be an exact prefix.
           "gh pr --repo openai/codex view 7888",
       ],
   )
   ```

3. 重新启动 Codex。

Codex 在启动时扫描每个 [团队配置](https://developers.openai.com/codex/enterprise/admin-setup#team-config) 位置下的 `rules/`。当您将命令添加到 TUI 中的允许列表时，Codex 会写入 `~/.codex/rules/default.rules` 处的用户层，以便将来的运行可以跳过提示。

当启用智能审批（默认）时，Codex 可能会在升级请求期间为你建议 `prefix_rule`。
在接受之前，请仔细检查建议的前缀。

管理员还可以强制执行限制性 `prefix_rule` 条目
[`requirements.toml`](https://developers.openai.com/codex/enterprise/managed-configuration#admin-enforced-requirements-requirementstoml)。

## 了解规则字段

`prefix_rule()` 支持这些字段：

- `pattern` **（必需）**：定义要匹配的命令前缀的非空列表。每个元素是：
  - 文字字符串（例如 `"pr"`）。
  - 文字的并集（例如 `["view", "list"]`），用于匹配该参数位置的替代项。
- `decision` **（默认为 `"allow"`）**：规则匹配时采取的操作。当多个规则匹配时（`forbidden` > `prompt` > `allow`），Codex 会采用最严格的决策。
  - `allow`：在沙箱外运行命令而不提示。
  - `prompt`：在每次匹配调用之前提示。
  - `forbidden`：阻止请求而不提示。
- `justification` **（可选）**：非空、人类可读的规则原因。 Codex 可能会在审批提示或拒绝消息中显示它。当您使用 `forbidden` 时，请在适当的情况下在理由中包含推荐的替代方案（例如 `"Use \`rg\` instead of \`grep\`."`）。
- `match` 和 `not_match` **（默认为 `[]`）**：Codex 在加载规则时会校验这些示例。使用它们可以在规则生效前发现错误。

当 Codex 考虑运行某个命令时，它会将命令的参数列表与 `pattern` 进行比较。在内部，Codex 将命令视为参数列表（就像 `execvp(3)` 接收的那样）。

## Shell 包装器和复合命令

某些工具将多个 shell 命令包装到单个调用中，例如：

```text
["bash", "-lc", "git add . && rm -rf /"]
```

由于这种命令可以在一个字符串中隐藏多个操作，因此 Codex 特别对待 `bash -lc`、`bash -c` 及其 `zsh` / `sh` 等效项。

### 当 Codex 可以安全地分割脚本时

如果 shell 脚本是仅由以下部分组成的线性命令链：

- 普通单词（无变量扩展，无 `VAR=...`、`$FOO`、`*` 等）
- 由安全运算符连接（`&&`、`||`、`;` 或 `|`）

然后 Codex 对其进行解析（使用 tree-sitter）并将其拆分为单独的命令，然后再应用您的规则。

上面的脚本被视为两个单独的命令：

- `["git", "add", "."]`
- `["rm", "-rf", "/"]`

然后，Codex 根据您的规则评估每个命令，最严格的结果获胜。

即使您允许 `pattern=["git", "add"]`，Codex 也不会自动允许 `git add . && rm -rf /`，因为 `rm -rf /` 部分是单独评估的，并且会阻止整个调用被自动允许。

这可以防止危险命令与安全命令一起偷运进来。

### 当 Codex 不拆分脚本时

如果脚本使用更高级的 shell 功能，例如：

- 重定向（`>`、`>>`、`<`）
- 替换（`$(...)`、`...`）
- 环境变量 (`FOO=bar`)
- 通配符模式（`*`、`?`）
- 控制流（`if`、`for`、`&&` 以及赋值等）

那么 Codex 不会尝试解释或拆分它。

在这些情况下，整个调用被视为：

```text
["bash", "-lc", "<full script>"]
```

并且您的规则将应用于该**单个**调用。

通过这种处理，您可以在安全时获得每个命令评估的安全性，而在不安全时获得保守的行为。

## 测试规则文件

使用 `codex execpolicy check` 测试您的规则如何应用于命令：

```shell
codex execpolicy check --pretty \
  --rules ~/.codex/rules/default.rules \
  -- gh pr view 7888 --json title,body,comments
```

该命令发出 JSON，显示最严格的决策和任何匹配规则，包括匹配规则中的任何 `justification` 值。使用多个 `--rules` 标志来组合文件，并添加 `--pretty` 来格式化输出。

## 理解规则语言

`.rules` 文件格式使用 `Starlark`（请参阅 [语言规范](https://github.com/bazelbuild/starlark/blob/master/spec.md)）。它的语法类似于 Python，但被设计为可安全运行：规则引擎执行它时不会产生副作用（例如访问文件系统）。
