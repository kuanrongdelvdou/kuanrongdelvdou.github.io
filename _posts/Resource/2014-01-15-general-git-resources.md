---
layout: post
title: Git 常用资源
category: resource
tags: Git
keywords: Git
description: 
---


## Git常用操作

### 从现有仓库克隆
	git clone <URL> [name]    克隆项目[name 指定项目目录名称]

### 检查当前文件状态
    git status

### 查看文件更新
    git diff                      查看未暂存的文件更新(比较的是工作目录中当前文件和暂存区域快照之前的差异)
    git diff --staged(cached)     查看已暂存文件的更新(比较已经暂存的文件和上次提交快照之间的差异)

### 添加文件进行跟踪
    git add <filename>      添加单个文件
    git add --all           添加全部文件

### 提交
    git commit [-a] -m "the descriptions of this commit"     提交[-a 跳过使用暂存区域]
    git commit -m "the descriptions of this commit"

### 取消某个文件的修改
    git checkout -- <filename>

### 回滚操作
    git reset --hard v0.1
    git reflog
    git reset --hard v0.2

### 删除文件
    git rm [-f] <filename>   直接删除文件[-f 强制删除]
    git rm --cached <filename>    删除文件暂存状态

### 移动文件
    git mv <sourcefile> <destfile>

### 修改最后一次提交
    git commit -amend

### 查看提交历史
    git log [-p] [-number] 查看历史[-p 展开每次提交内容差异][-number 仅显示最近number次更新,例如-2]

### 取消暂存的文件
    git reset HEAD <file> 

### 克隆远程分支
    git branch -r
    git checkout origin/android
    
### 打标签(tag)
    git tag -a [version] -m "the descriptions of this commit"
    例如： git tag -a v1.4 -m 'version 1.4'

### 查看标签列表(tag)
    git tag
    git tag -l 'v1.0.*'    特定的搜索模式

### 查看相应标签的版本信息
    git show [version]

### 推送标签
    默认情况下，git push 并不会把标签传送到远端服务器上，只有通过显式命令才能分享标签到远端仓库。
    git push origin [tagname]
    git push origin --tags         推送所有本地新增的标签

## Git远程操作

### 查看当前远程库
    git remote [-v]     查看当前远程仓库[-v 显示对应克隆地址]

### 添加远程仓库
    git remote add [shortname] [url] 

### 从远程仓库抓取数据
    git fetch [remote-name]

### 可视化冲突编辑器
    git mergetool

### 推送数据到远程仓库
    git push [remote-name] [branch-name]
    例如，如果要把本地的 master 分支推送到 origin 服务器上：git push origin master
    （克隆操作会自动使用默认的 master 和 origin 名字）

### 删除远程仓库
    git remote rm [remote-name]

## Ask & Answer
### git rm --cache 和 git reset HEAD 有什么区别？
当HEAD里没有<somefile\>时，用“git rm --cache”来unstage    
当HEAD里有<somefile\>时，用“git reset HEAD”

### git fetch 和git pull有什么区别？
git fetch：相当于是从远程获取最新版本到本地，不会自动merge
git pull：相当于是从远程获取最新版本并merge到本地，其实相当于git fetch + git merge
在实际使用中，git fetch更安全一些
因为在merge前，我们可以查看更新情况，然后再决定是否合并