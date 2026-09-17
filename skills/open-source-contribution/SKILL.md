---
name: open-source-contribution
description: 使用 Git 和 GitHub CLI（`gh`）准备并提交 GitHub 开源项目贡献，包括仓库规范检查、Fork 与分支管理、本地验证、用户审核、Draft PR 以及维护者反馈处理。适用于向上游项目贡献修复、功能、文档或其他变更；不适用于用户自有仓库中的普通开发，除非用户明确要求采用上游贡献流程。
---

# 开源项目贡献

用户要求“帮我提交”时，应实际检查仓库、完成修改和验证，并推进到用户审核节点；不要只返回一份操作教程。只有用户明确询问“怎么做”时，才以讲解和命令示例为主。

默认推进到以下状态：

1. 已确认上游仓库规则、默认分支以及是否需要先开 Issue。
2. 已排查重复 Issue/PR，在独立分支完成最小范围修改。
3. 已运行相关检查，并明确记录未运行项目及原因。
4. 已准备真实 diff、Commit 列表和完整 PR 文案，等待用户审核。
5. 用户批准公开写入后，使用 `gh` 推送并创建 Draft PR；用户再次确认后才标记为 Ready for review。

如果用户一开始已经明确授权 Fork、Push 和创建 PR，仍需在发布前展示审核材料，除非用户同时明确要求跳过审核。

## 遵守项目规范和用户意图

- 将目标仓库的 `AGENTS.md`、`CONTRIBUTING.md`、`README`、PR 模板、Issue 模板、行为准则和必需检查视为最高优先级的项目规则。
- 贡献范围只覆盖用户要求解决的问题，不夹带无关重构、依赖升级或格式化。
- 保留现有工作区改动和远端配置，不覆盖、丢弃或悄悄纳入归属不明的改动。
- 不发布密钥、隐私数据、内部地址、生成的凭据或用户专属配置。
- 如果变更涉及安全漏洞，停止公开 Issue 和 PR 流程，改为遵循仓库的 `SECURITY.md` 或私密披露渠道。

## 修改前检查

先确认工具、身份、目标仓库、远端和工作区状态：

```bash
gh --version
gh auth status
git status --short --branch
git remote -v
gh repo view OWNER/REPO --json nameWithOwner,url,defaultBranchRef,viewerPermission,isFork,parent
```

不在目标仓库中时，使用 `gh repo clone OWNER/REPO` 克隆；已有本地仓库时优先复用，不重复克隆。通过下面的命令读取默认分支，不硬编码 `main`：

```bash
gh repo view OWNER/REPO --json defaultBranchRef --jq '.defaultBranchRef.name'
```

然后：

1. 阅读适用于待修改文件的仓库说明。
2. 确认默认目标分支、必需测试、提交信息规范、DCO 签署要求和 PR 模板。
3. 搜索是否已有重复或相关的 Issue 和 PR：

   ```bash
   gh issue list --repo OWNER/REPO --search "SEARCH TERMS"
   gh pr list --repo OWNER/REPO --search "SEARCH TERMS"
   ```

4. 判断维护者是否要求先创建 Issue 或进行设计讨论。任何公开消息都应先起草，并在发布前交给用户审核。

不要假设默认分支一定是 `main`，也不要假设远端名称、Fork 所有者或直接推送权限；必须从仓库实际状态中确认。

## 选择贡献方式

- 普通外部贡献或没有直接推送权限时使用 Fork。
- 只有用户具备相应权限且仓库规则允许时，才在上游仓库中直接创建分支。
- 如果已经位于本地克隆中，保留现有远端，仅用含义明确的名称补充缺失的上游或 Fork 远端。
- 创建 Fork、推送分支、创建 Issue 或 PR、提交 Review 和评论都属于外部写入。只有用户请求已授权相应操作，并通过适用的审核节点后才能执行。

## 发现或创建 Fork

先取得当前 GitHub 用户和目标仓库的规范名称：

```bash
gh api user --jq '.login'
gh repo view OWNER/REPO --json nameWithOwner,name,defaultBranchRef
```

列出当前用户已有的 Fork，并按 `parent.nameWithOwner` 查找目标上游。不要只按仓库名判断，因为 Fork 可能被重命名：

```bash
gh repo list FORK_OWNER \
  --fork \
  --limit 1000 \
  --json nameWithOwner,parent,sshUrl,url \
  --jq '.[] | select((.parent.owner.login + "/" + .parent.name) == "OWNER/REPO") | {nameWithOwner,sshUrl,url}'
```

根据查询结果处理：

- 找到一个匹配项：复用它，不再创建 Fork。使用 `gh repo view FORK_OWNER/FORK_REPO --json nameWithOwner,isFork,parent,sshUrl` 再确认 `isFork` 为 `true`，且 `parent.owner.login + "/" + parent.name` 组成的仓库名正是目标上游。
- 没有匹配项：先检查 `FORK_OWNER/REPO_NAME` 是否已被同名非目标仓库占用。若有冲突，停止并让用户选择其他名称或组织；不得覆盖、删除或改造该仓库。
- 确认不存在可复用 Fork 后，只有在用户授权创建 Fork 时才执行：

  ```bash
  gh repo fork OWNER/REPO --clone=false --remote --remote-name fork
  ```

已有 Fork 时，先查看 `git remote -v`：

- 某个现有远端已经指向该 Fork：直接复用该远端，不重复添加。
- `fork` 名称尚未使用：执行 `git remote add fork git@github.com:FORK_OWNER/FORK_REPO.git`。
- `fork` 已指向其他仓库：不要静默修改 URL；使用明确的新名称，或让用户决定是否调整现有配置。
- 确保另一个远端指向目标上游，通常命名为 `upstream`。已有正确上游远端时直接复用。

比较远端时按 GitHub 仓库身份判断，不要因为一个使用 SSH、另一个使用 HTTPS 就误判为不同仓库。配置后再次运行 `git remote -v` 并报告实际使用的上游和 Fork 远端。

## 从最新上游创建分支

不要求先同步 Fork 的默认分支。优先获取上游最新目标分支，并直接从它创建主题分支：

```bash
git fetch upstream BASE_BRANCH
git switch -c TOPIC_BRANCH upstream/BASE_BRANCH
```

如果实际的上游远端不是 `upstream`，替换为已确认的远端名称。不要从过期的 Fork 默认分支创建贡献分支。

只有用户要求同步 Fork，或仓库工作流确实依赖 Fork 默认分支时，才执行远端同步：

```bash
gh repo sync FORK_OWNER/FORK_REPO --source OWNER/REPO --branch BASE_BRANCH
```

默认只允许 Fast-forward 同步。同步发生冲突时停止并报告，不自动添加 `--force`，因为它会重置 Fork 上的目标分支。

## 在本地准备变更

1. 从上游当前目标分支开始，创建名称明确且范围单一的主题分支。
2. 只实现预期变更，并按仓库要求补充测试和文档。
3. 运行与修改范围相关的格式化、Lint、测试和构建检查。
4. 对照目标分支审核完整 diff，包括所有新增文件。
5. 按项目的提交规范创建聚焦的 Commit；只有项目要求 DCO 时才添加 `--signoff`。

没有实际运行的检查不得声称通过。无法运行的命令必须记录原因。

## 用户审核节点

默认在第一次公开写入前进行本地审核。在当前任务中向用户提供一份简洁的审核材料，包括：

- 目标仓库和目标分支；
- 相关 Issue 或讨论链接；
- 主题分支以及计划推送到的 Fork 或远端；
- 文件变更与 diff 摘要；
- 测试命令和结果；
- Commit 列表；
- 拟定的 PR 标题和正文；
- 已知风险、缺失项或仍需维护者决定的问题。

当前界面支持 Review 视图或文件链接时，应让用户能直接查看真实 diff，而不是只提供终端摘要。除非用户已经明确批准了准确的分支、内容和 PR 提交，否则发布前应等待用户确认。

如果用户希望在 GitHub 上审核并授权发布，创建 **Draft PR** 并提供链接。Draft PR 在 GitHub 上仍然可见，因此不能将创建 Draft PR 当成私密或只读操作。用户批准前保持 Draft 状态。

批准后如果范围或行为发生实质变化，必须带着新 diff 和更新后的 PR 描述重新进入审核节点。

向用户提交审核时使用下面的结构，删除不适用项，不留占位符：

```markdown
目标：OWNER/REPO ← FORK_OWNER:TOPIC_BRANCH
Base：BASE_BRANCH
关联：ISSUE_OR_DISCUSSION_URL

变更：
- FILE：做了什么，以及为什么

验证：
- `COMMAND`：通过/失败
- 未运行：COMMAND（原因）

Commits：
- SHA SUBJECT

PR：
- 标题：TITLE
- 正文：完整正文

风险或待确认：
- ITEM
```

## 使用 GitHub CLI 发布

将主题分支推送到已确认的 Fork 或可写远端：

```bash
git push -u fork HEAD
```

向上游仓库创建 Draft PR：

```bash
gh pr create \
  --repo OWNER/REPO \
  --base BASE_BRANCH \
  --head FORK_OWNER:TOPIC_BRANCH \
  --draft \
  --title "PR TITLE" \
  --body-file /path/to/pr-body.md
```

遵循仓库的 PR 模板。正文应说明问题和解决方案，按项目惯例关联 Issue，并只列出实际运行过的测试。只有 Commit 历史能准确生成标题和正文时才使用 `--fill`。

仓库没有 PR 模板时，使用下面的最小正文，不虚构 Issue、测试或兼容性声明：

```markdown
## 变更说明

- 说明改了什么。
- 说明为什么需要这样改。

## 验证

- `实际运行的命令`

## 关联事项

Closes #ISSUE_NUMBER
```

没有关联 Issue 时删除“关联事项”章节；存在用户可见行为变化时，补充变更前后的表现和必要的截图或输出。

验证发布结果：

```bash
gh pr view --repo OWNER/REPO --json number,url,state,isDraft,headRefName,baseRefName
gh pr checks PR_NUMBER --repo OWNER/REPO
```

用户批准后，将 Draft PR 标记为可供维护者正式审核：

```bash
gh pr ready PR_NUMBER --repo OWNER/REPO
```

除非用户明确要求且仓库规则允许，否则不要合并 PR。

## 处理维护者反馈

修改代码前先检查 PR、diff、评论和 CI：

```bash
gh pr view PR_NUMBER --repo OWNER/REPO --comments
gh pr diff PR_NUMBER --repo OWNER/REPO
gh pr checks PR_NUMBER --repo OWNER/REPO --watch
```

- 区分必须修改的要求、问题和可选建议。
- 在本地完成已确认的修改，重新运行相关检查，说明本次变化，然后推送同一个主题分支以更新 PR。
- 回复中涉及项目决策、分歧、承诺或信息披露时，先起草内容交给用户审核。
- 正常情况下使用普通 Push。只有确实需要改写历史且得到明确批准时才使用 `--force-with-lease`，禁止无保护的强制推送。

## 交付说明

报告 PR 链接及状态、目标和来源分支、最终 Commit、已运行的检查、尚未解决的反馈，以及下一步需要用户或维护者完成的事项。
