---
title: Git Cheat Sheet
katex: false
mathjax: false
mermaid: false
categories:
  - 教程
tags:
  - Git
excerpt: 記錄自己一些常用的 Git 指令
date: 2025-09-02 14:39:34
updated: 2025-09-02 14:39:34
index_img:
banner_img:
---


{% note primary %}

本篇不包含關於 Git 的概念教學

{% endnote %}

# 一些基礎設定

## 列出設定

### 列出目前的設定

```shell
$ git config list
```

### 列出全域設定

```shell
$ git config --global --list
```

### 列出目前 Git Repo 的設定

```shell
$ git config --local --list
```

## 修改設定

全域設定和個別 Git Repo 的設定分別加`--global`或`--local`來設定，以下只舉全域為例

### 個人資料

```shell
$ git config --global user.name "Your Name"
$ git config --global user.email "Your Email"
```

### GPG 簽章

```shell
$ git config --global commit.gpgsign "true"
$ git config --global user.signingkey "Your GPG Key"
```

### 預設分支名稱

```shell
$ git config --global init.defaultbranch "main"
```

# Repo 操作

## 初始

### 建立 Repo

```shell
$ git init
```

## 複製 Repo

```shell
# URL可以是ssh或是http(s)的，但建議用ssh，比較方便
$ git clone <URL>
```

## 修改與提交

### 查看目前的修改狀態

```shell
$ git status
```

### 將檔案移至 stage

```shell
$ git add <file>
```

### 將檔案移出 stage

```shell
$ git reset <file>
```

### Commit

```shell
$ git commit -m "commit message"
```

### 推送

```shell
$ git push
```

## 差異

### 查看目前工作目錄與 HEAD 的差異

```shell
$ git diff
```

### 查看 Stage 與 HEAD 的差異

```shell
$ git diff --staged
```

### 查看某個檔案與 HEAD 的差異

```shell
$ git diff <file>
```

### 查看兩個 Commit 的差異

```shell
$ git diff <commit1 hash> <commit2 hash>
```

### 查看某個檔案在兩個 Commit 的差異

```shell
$ git diff <commit1 hash> <commit2 hash> <file>
```

## 記錄

```shell
$ git log
$ git log --graph --oneline
$ git log --graph --oneline --all
```

## 分支與撤銷

### 建立分支

有很多種方式建立

```shell
# 建立但不切換
$ git branch <new branch name>
# 建立並切換
$ git checkout -b <new branch name>
# 建立並切換
$ git switch -c <new branch name>
```

### 切換分支

`git checkout`也可以切換，但`checkout`包含的功能很多，`checkout`被切分成多個指令，如`switch`、`restore`

```shell
$ git switch <branch name>
```

### 切換 Tag

```shell
$ git switch --detach <tag name>
```

### 將 B 的 Commit merge 進 A

```shell
$ git switch A
$ git merge B
```

### Rebase

```shell
# 將目前的修改重新接到B branch新更新的位置
$ git rebase B
# 拉取時重新rebase
$ git pull --rebase
```

### 刪除分支快取

```shell
$ git fetch --all --prune
```

### Reset

```shell
# 預設為--mixed，stage變成拆掉的commit內容，不修改工作區
$ git reset <commit>
# 只拆掉commit，不修改stage和工作區
$ git reset --soft <commit>
# 將stage清掉，工作區會到commit時的狀態
$ git reset --hard <commit>
```

### Restore

從`checkout`拆出來的指令，包含部分`reset`的功能，但不會修改`HEAD`，只動檔案

```shell
# 從stage還原到工作區
$ git restore <file>
# 將HEAD還原到stage，不動工作區
$ git restore --staged <file>
# 同時從HEAD還原到stage和工作區
$ git restore --staged --worktree <file>
```

## Stash

### 列出暫存

```shell
$ git stash list
```

### 將工作區和 Stage 都暫時儲存

```shell
$ git stash
```

### 只暫存 stage

```shell
$ git stash push --staged
```

### 恢復暫存

```shell
$ git stash pop
```

## Remote

### 列出所有 Remote

```shell
git remote show
```

### 增加 Remote

```shell
$ git remote add <remote name> <remote URL>
```

