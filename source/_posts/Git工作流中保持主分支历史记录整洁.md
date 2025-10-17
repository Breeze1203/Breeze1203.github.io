---
layout: git
title: Git工作流中保持主分支历史记录整洁
date: 2025-10-17 20:25:38
categories: Essay
---

### 方法一：使用 git merge --squash
步骤：
假设你的主分支是 main，你的功能分支是 feature/add-login。
确保 main 分支是最新
```Bash
git checkout main
git pull origin main
```
执行 squash merge
这会抓取 feature/add-login 上的所有更改但不会自动提交，而是将它们放入暂存区
```Bash
git merge --squash feature/add-login
```

```Bash
# 检查状态并提交
git status
``` 
你会看到所有来自 feature/add-login 的文件更改都已“暂存”(Changes to be committed)

创建一个新的、干净的提交 ,用一个有意义的信息来概括所有工作
```Bash
git commit -m "feat: 完成登录功能"
```
推送主分支

```Bash
git push origin main
```
结果： main 分支只增加了一个名为 "feat: 完成登录功能" 的提交。

### 方法二：使用 git rebase -i (交互式变基)
这是最强大但也最复杂的方法。它允许你重写你的功能分支上的提交历史。你不仅可以压缩，还可以编辑、删除、重排它们。

!! 警告： 此方法会重写提交历史。如果你的功能分支已经推送（push）到远程，并且有其他人也基于这个分支在开发，请不要使用此方法，否则会给他们带来巨大的麻烦。如果只有你一个人用这个分支，那么是安全的。

步骤：
切换到你的功能分支

```Bash
git checkout feature/add-login
```
```
git rebase main
```
如果有冲突，先解决冲突
启动交互式变基 你需要告诉 rebase 你想修改“从哪里开始”的提交。通常，我们选择从 main 分支分叉出去的地方开始。

```Bash
git rebase -i main
```
（如果你不确定，也可以用 git rebase -i HEAD~5 来修改最近的5个提交）
编辑变基脚本 执行上述命令后，Git 会打开一个文本编辑器（如 Vim），显示一个提交列表，看起来像这样：
```
Plaintext
pick 1a2b3c feat: add login button</br>
pick 4d5e6f fix: typo in button</br>
pick 7g8h9i refactor: login logic</br>
pick 1j2k3l style: adjust padding</br>
pick 4m5n6p fix: fix login bug</br>
```
修改脚本以压缩提交 要将所有提交压缩成一个,
保留第一个为 pick（保留），将其余的改为 s (squash) 或 f (fixup)。
s (squash): 会合并提交，并让你在稍后编辑合并后的提交信息。
f (fixup): 会合并提交，但会直接丢弃这个提交的信息（适用于 "fix typo" 这种无意义信息）。

修改后的文件：
```
Plaintext

pick 1a2b3c feat: add login button
f 4d5e6f fix: typo in button
s 7g8h9i refactor: login logic
f 1j2k3l style: adjust padding
s 4m5n6p fix: fix login bug
```
保存并退出编辑器 (在 Vim 中是 :wq)
编辑新的提交信息 因为你使用了 s (squash)，Git 会再次打开一个编辑器，让你为这个被压缩后的新提交编写提交信息。把所有信息整合成一个有意义的描述，例如 "feat: 完成登录功能"。
（如果需要）强制推送 因为你重写了历史，你本地的 feature/add-login 分支已经和远程（origin）上的不同了。你需要“强制推送”来更新它。

```Bash
再次警告：确保只有你一个人在这个分支上工作！
git push origin feature/add-login --force
```
正常合并 现在你的功能分支在（远程和本地）都只有一个干净的提交了。你可以去 main 分支进行一次普通的合并。
```Bash
git checkout main
git merge feature/add-login
```
