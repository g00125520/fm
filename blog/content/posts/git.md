---
title: "Git"
date: 2018-04-23T19:33:37+08:00
draft: true
---
创建新的本地分枝：git branch dev-gyz；将本地分枝推送到对应的远端分枝，并设置upstream：git push -u origin local:remote；删除远程分枝：git branch -d <branch_name>,git push <remote_name> --delete <branch_name>,git push <remote_name> :<branch_name>

版本回退：git reset HEAD^，一个^表示一个版本，可以多个，也可以HEAD～n;软回退：git reset --soft HEAD^，将本地版本库的头指针指向指定版本，切将此次提交之后的修改移动到暂存区；也就是本地的commit的记录；默认为mixed：git reset HEAD~1，将头指针指向指定的版本，并重置暂存区，将该版本之后的提交工作区；硬退回：git reset --hard HEAD~1，回退一个版本，不仅重置头指针，也会重置暂存区和工作区；reset之后，推送到服务器：git push -f origin dev-gyz

查看提交历史：git log -3，回退可以直接指定提交id，如:git reset commit-id


