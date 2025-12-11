---
title: GitFlow
draft: "false"
tags:
  - git
created: 2025-12-11 22:01
modified: 2025-12-11 22:08
---
## 分支命名规范

首先，我们约定一套清晰的分支命名规范，这有助于团队成员快速理解分支的用途。

| 分支类型      | 前缀          | 命名格式              | 示例                                | 描述                                |
| :-------- | :---------- | :---------------- | :-------------------------------- | :-------------------------------- |
| **功能分支**  | `feature/`  | `feature/` + 功能名  | `feature/user-login`              | 开发一个新功能。最常见、最长命的分支类型。             |
| **修复分支**  | `fix/`      | `fix/` + 修复内容     | `fix/login-button-bug`            | 修复线上（或测试环境）的紧急 Bug。               |
| **热修复分支** | `hotfix/`   | `hotfix/` + 修复内容  | `hotfix/critical-security-patch`  | 紧急修复生产环境的严重问题，通常在 `master` 上直接分叉。 |
| **任务分支**  | `task/`     | `task/` + 任务描述    | `task/update-dependencies`        | 用于完成一个较小的、非功能的任务，如升级库、重构代码等。      |
| **发布分支**  | `release/`  | `release/` + 版本号  | `release/v1.2.0`                  | 用于准备新版本的发布。在此分支上最后进行测试和修复。        |
| **技术重构**  | `refactor/` | `refactor/` + 模块名 | `refactor/user-profile-component` | 对现有代码进行重构，不改变外部行为。                |
| **文档**    | `docs/`     | `docs/` + 文档内容    | `docs/api-documentation`          | 仅仅修改文档。                           |
## 详细工作流步骤

1.  **创建新功能分支**
从最新的 `master` 分支(github 通常用main)检出一个新的功能分支进行开发。
```bash
# Ensure your local master branch is up-to-date with the remote.
git checkout master
git pull origin master
# Create and switch to a new feature branch.
# Replace 'xxx' with your specific feature name, e.g., 'user-authentication'.
git checkout -b feature/xxx
```

2. **日常开发**:
在功能分支上进行编码、提交等操作。
```bash
# ... coding ...
# Stage your changes for commit.
# e.g., git add src/components/UserForm.js
git add /xxx
# Commit your changes with a clear and concise message.
# Follow a convention like: type(scope): description
# e.g., feat(auth): add login form component
git commit -m "feat: add xxx"
git add /xxx
git commit -m "feat: implement xxx"
```

3. **准备合并**:

在提交 Merge Request (MR) 之前，将你的功能分支基于最新的 master 进行变基。这可以线性地整合代码，避免不必要的合并提交。
```bash
# Rebase your branch on top of the latest remote master.
# This will fetch changes and try to apply your commits on top.
git pull --rebase origin master
# If the rebase results in conflicts:
# 1. Manually resolve the conflict files in your editor.
# 2. Stage the resolved files.
#    e.g., git src/components/conflict-file.js
# 3. Continue the rebase process.
#    e.g., git rebase --continue
# Repeat steps 1-3 until all conflicts are resolved and rebase is complete.
```

4. **推送分支并创建 MR**
将你的本地分支推送到远程仓库，
然后在 GitLab/GitHub 上创建一个 Merge Request。
```bash
git push -u origin feature/xxx
```

**在 GitLab/GitHub 上创建 MR**：
Source Branch: feature/xx
Target Branch: master
Title: feat: 实现 xxx
Description: 该功能包含了xx的数据表、后台管理服务和前端展示页面。

5. 代码审查并合并：

团队成员（或你自己）在 MR 页面审查代码。审查通过后，点击 **"Rebase and Merge"** 按钮。这个操作会先将你的分支 `rebase` 到目标分支（通常是 `master`）的最新状态，然后再执行快进合并，从而保持提交历史线性和整洁。

6. **清理残留分支**

对于远程分支, 一般可以在merge的时候, 选择`merge后自动清理`
对于本地分支, 可以

```bash
# Switch to the master branch.
git checkout master

# pull if it has been merged.
git pull origin master

# Safely delete the local branch if it has been merged.
# The '-d' flag will prevent deletion if the branch is not fully merged.
git branch -d feature/xxx
# Forcibly delete a branch, even if it's not merged.
# Use with caution. The '-D' flag is an alias for '--delete --force'.
# git branch -D feature/xxx
```

7. **处理 Bug(Hotfix)**：
假设功能合并后，发现在生产环境有一个显示错误。你可以为这个紧急问题创建一个修复分支：

```bash
# Branch off the latest master.
git checkout master
git pull origin master
# Create a new branch for the fix.
# Replace 'xxx' with the specific issue, e.g., 'points-display-error'.
git checkout -b fix/xxx
# Fix the bug...
git add /xxx
git commit -m "fix: fix display bug on page"
# Push the fix branch to the remote.
git push -u origin fix/xxx
```

然后，创建一个从 `fix/xxx` 到 `master` 的 MR，并尽快进行合并。对于紧急的线上问题，可能需要直接从 `master` 创建 `hotfix/` 分支，合并后还需要立即打一个 Tag 并部署到生产环境。

## 补充说明
通常, 我们认为, 即使是个人开发, 也应该提交MR, 然后Review, 而不是直接 push 到主分支. 
MR 保证了以下几点:
1. 强制进行代码审查
2. 清晰、可读、有意义的 Git 历史记录
3. 思考的暂停点, 可以切换去更紧急的任务
4. 主分支保持干净, 风险可控