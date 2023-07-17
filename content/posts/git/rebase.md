---
title: "git rebase 命令"
date: 2023-05-12T16:01:07+08:00
description: ""
draft: true
tags: [git]
categories: [git]
---



1. 选择修改的commit


git rebase -i HEAD~2  表示修改前两个

会弹出编辑界面，在commit前面修改`pick`，一般是改成edit或e

保存退出

2. 进行修改和提交

# do something and add your change
# modify file
git add ..
git commit --amend


3. 修改下一个
git rebase --continue