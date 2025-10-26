---
title: Git分支管理与多人协作
date: 2016-03-06 20:47:46
tags: Git
---
“有了远程仓库，妈妈再也不用担心我的硬盘了。”——Git点读机

## 概述

工作区有一个隐藏目录.git，这个不算工作区，而是Git的暂存区和版本库。

前面讲了我们把文件往Git版本库里添加的时候，是分两步执行的：
第一步是用git add把文件添加进去，实际上就是把文件修改添加到暂存区；
第二步是用git commit提交更改，实际上就是把暂存区的所有内容提交到当前分支。

git commit只负责把暂存区的修改提交.因为我们创建Git版本库时，Git自动为我们创建了唯一一个master分支，所以，现在，git commit就是往master分支上提交更改。
<!--more-->
## 配置用户信息

```sh
# --global参数是全局参数，也就是这些命令在这台电脑的所有Git仓库下都有用。
# 配置用户名，邮箱，彩色UI
git config --global user.name "Your Name"
git config --global user.email "email@example.com"
git config --global color.ui true
# 还有要给远程仓库添加本地的SSH-public-key

# 配置命令别名，简化操作
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.ci commit
git config --global alias.br branch
git config --global alias.unstage 'reset HEAD'
git config --global alias.last 'log -1'
git config --global alias.lg "log --color --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit"

# 本仓库配置文件，不加--global
# 每个仓库的Git配置文件都放在.git/config文件中：
$ cat .git/config 
[core]
    repositoryformatversion = 0
    filemode = true
    bare = false
    logallrefupdates = true
    ignorecase = true
    precomposeunicode = true
[remote "origin"]
    url = git@github.com:michaelliao/learngit.git
    fetch = +refs/heads/*:refs/remotes/origin/*
[branch "master"]
    remote = origin
    merge = refs/heads/master
[alias]
    last = log -1

# 全局配置
# 用户主目录下的一个隐藏文件.gitconfig中

$ cat .gitconfig
[alias]
    co = checkout
    ci = commit
    br = branch
    st = status
[user]
    name = Your Name
    email = your@email.com
```

## 常用操作

```sh
#查看状态
git status

#对比文件更新处
git diff readme.txt

# 查看历史记录
git log --pretty=oneline

# 回退到Head指针的前一个版本
git reset --hard HEAD^

# 回退到之前的某个版本
git reset --hard (commit-hash-id)

# 如果忘记了commit-hash-id怎么办,获取历史hash值
git reflog

# 将某个修改过的文件还原成上次git add时的状态
# (丢弃工作区内容为暂存区,若无，则为版本库内容)
git checkout -- fileName
# 切换分支
git checkout 
# 丢弃暂存区此文件内容
git reset HEAD fileName
```

## 分支管理

```sh
# Git鼓励你使用分支完成某个任务，合并后再删掉分支，这和直接在master分支上工作效果是一样的，但过程更安全。

# 创建并切换分支
git checkout -b dev
# 删除分支
git branch -d dev


# 合并指定分支到当前分支
# 如果指定分支是在当前分支最新版本上衍生出来的，则可以快速合并，否则可能需要解决冲突。
git merge dev

# 冲突处理

# Git用<<<<<<<，=======，>>>>>>>标记出不同分支的内容。
# 直接打开文件将当前分支冲突的部分修改成指定分支的内容或删除，再提交一个新的版本，再合并。

# 禁用快速合并,因为在这种模式下，删除分支后，会丢掉分支信息。
git merge --no-ff -m "merge with no-ff" dev

# 图形化分支历史
git log --graph --pretty=oneline --abbrev-commit

# 保存当前分支现场,包括工作区和暂存区(如果本分支还有没有做完的工作，但是需要切换分支)
git stash

# 恢复工作现场
git stash list

# 恢复不删除
git stash apply [stash@{0}]
# 删除stash
git stash drop [stash@{0}]

#  恢复的同时删除
git stash pop

# 查看远程分支
git remote -v

# 创建本地与对应的远程分支
git checkout -b dev origin/dev

# 将本地当前分支推送到远程库(不要直接用git push，这样会推送所有分支)
git push origin/dev

# 指定本地dev分支与远程origin/dev分支的链接
git branch --set-upstream dev origin/dev

# 推送到远程分支失败(解决方案)
## 拉取远程分支到本地,使用前需要将当前分支与远程分支建立连接，不然不知道pull到哪里
git pull 
## 如果有冲突，要解决冲突后commit -m " conflict & fix"，再push。

# 添加版本别名
git tag v1.0  [-m "version 0.1 released" 3628164]

# 查看版本信息
git show v0.9

# 删除本地版本号
git tag -d v0.1

# 推送版本到远程库
git push origin v1.0

# 删除远程标签，先要删本地
git push origin :refs/tags/v0.9

``` 
## 管理策略

master分支应该是非常稳定的，也就是仅用来发布新版本，平时不能在上面干活；
干活都在dev分支上，也就是说，dev分支是不稳定的，到某个时候，比如1.0版本发布时，再把dev分支合并到master上;

如果是多人协同开发，则每个人开发时都需要创建自己的分支，开发者完成一个任务时，只合并到dev分支上，--no-ff参数可以看出合并历史；

一些命名规范：
```sh

# Bug分支
git checkout -b issue-101

# Feature分支
git checkout -b feature-vulcan

# 强制删除某个未合并的分支，慎用
git branch -D feature-vulcan

```
一些远程同步的规范：
* master分支是主分支，因此要时刻与远程同步；
* dev分支是开发分支，团队所有成员都需要在上面工作，所以也需要与远程同步；
* bug分支只用于在本地修复bug，就没必要推到远程了，除非老板要看看你每周到底修复了几个bug；
* feature分支是否推到远程，取决于你是否和你的小伙伴合作在上面开发。

多人协作的工作模式：
1. 首先，可以试图用git push origin branch-name推送自己的修改；
2. 如果推送失败，则因为远程分支比你的本地更新，需要先用git pull试图合并；
3. 如果合并有冲突，则解决冲突，并在本地提交；
4. 没有冲突或者解决掉冲突后，再用git push origin branch-name推送就能成功！

复制当前Git仓库创建一个完全独立的Git仓库，完成了之后，可以发起pull request来为官方Git库贡献代码

## 添加忽略
根目录下添加```.gitignore```文件，可以参考或历史项目的此文件，在项目开始时就要添加：
* https://github.com/github/gitignore
* https://www.gitignore.io/api/jetbrains
忽略文件的原则是：
1. 忽略操作系统自动生成的文件，比如缩略图等；
2. 忽略编译生成的中间文件、可执行文件等，也就是如果一个文件是通过另一个文件自动生成的，那自动生成的文件就没必要进版本库，比如Java编译产生的.class文件；
3. 忽略你自己的带有敏感信息的配置文件，比如存放口令的配置文件。

相关命令：
```sh
# 与之对应的忽略文件行
git check-ignore -v App.class
# 强制添加
 git add -f App.class
```
## 参考资料

廖雪峰的Git教程

https://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000

