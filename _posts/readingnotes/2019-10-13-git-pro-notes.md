---
layout: post
title: Pro  Git 笔记
category: 读书笔记
tags: Git
---
* content
{:toc}

## 远程仓库
git remote -v  显示对应的克隆地址，如果有多个远程仓库，将会全部列出来

git remote add pb    添加一个远程仓库 ,pb 是远程仓库的地址，例如 git@github.com:hoyouly/LearnGit.git

git remote show 查看某个远程仓库的详细信息

git remote rename pb paul  修改某个仓库的简短名称，比如把pb改为paul

git remote rm paul 移除对应的远程仓库



## git diff
* git diff 比较的是工作目录中当前文件和暂存区之间的差异
* git diff --cached   查看已经暂存起来的文件和上次提交时候的差异，其实就是查看git add 后文件和已经提交上去的差异
* git diff --staged   在git 1.6.1版本以及更高版本使用，和git diff --cached 效果一样


## git commit
git commit -v 修改差异的每一行都包含到注释中来
git commit -a 自动吧所有已经跟着过的文件暂存起来一并提交，从而跳过git add 步骤

git commit --amend 重新编写提交说明，修正第一个提交的内容

## git log
参数列表
* -p   : 按照补丁格式显示每个更新之间的差异
* --stat  ：显示每次更新的文件修改统计信息
* --shortstat : 只显示 --stat 中最后的行数修改添加移除统计记录
* --name--only : 仅在提交信息后显示已修改的文件清单
* --name-status : 显示新增，修改，删除的文件清单
* --abbrev-commit : 仅显示SHA-1 的前几个字符，而非所有的40个字符
* --relative-date 使用较短的相对显示时间
* --graph 显示ASCII图形表示的分支合并历史
* pretty 使用其他格式显示历史提交信息，可用的选项包括
  * oneline     git log --pretty=oneline 将每个提交放在一行显示
  * short
  * full
  * fuller
  * format  格式化输出   
  eg: git log --pretty=format : "%h - %an ,%ar : %s"
    * %H 提交对象 commit的完整哈希子串
    * %h 提交对象的尖端哈希串
    * %T 树对象的完整哈希串
    * %t 树对象的简短哈希串
    * %P 父对象的完整哈希串
    * %p 父对象的简短哈希串
    * %an 作者的名字
    * %ad 作者修订的日期，可以用 -date=选项定制格式
    * %ar 作者修改日期，按多久以前的方式显示
    * %cn 提交者的名字
    * %ce 提交者的电子邮件
    * %cd 提交日期
    * %cr 提交日期，安多久以前的方式显示
    * %s 提交说明

* --since , --after : 仅显示指定时间之后的提交  
  eg: git log --since=2.weaks  列出最近两周内的提交
* --until  ,--before   : 仅显示指定时间之前的提交  
* --author  :  仅显示指定作者相关的提交
* --committer  :  仅显示指定提交者相关的提交


git log -g  格式化输出引用日志信息

git log master..experiement 所有可从experient分支中获得而不能从master分支获得的提交，其实就是只存在experiement中的提交

## git show
git show HEAD 查查最近一次提交
git show HEAD~ 查看最近一次之前的一次提交，
git show HEAD~2 查看最近第三次提交，

## git reset
取消暂存
git reset HEAD <file>  撤销add   <file> 文件的操作
git reset HEAD ^ 撤销所有add 的操作


git 使用HEAD 来代替不存在的（省略的）一边
## git 别名
git config --global alias.co  checkout   git co  相当于 git checkout    
git config --global alias.br  branch     git br  相当于 git branch    
git config --global alias.ci  commit     git ci  相当于 git commit     
git config --global alias.st  status     git st  相当于 git status     

<span style="border-bottom:1px solid red;">  git config --global alias.logf  'log --pretty=format:"%h - %an ,%ar : %s"' </span> , 那么输入 git logf 相当于 git log --pretty=format:"%h - %an ,%ar : %s"


## 删除 分支
git branch -d 分支名  删除本地分支，如果该分支没有merge到当前分支，会删除失败

git branch -D 分支名  强制删除本地分支


### 远程分支
像是一个书签，提醒这你上次连接远程长裤时上面各个分支的为位置

git fetch  同步，获取尚未拥有的数据，更新你本地的数据库，但是不会合并，

git push origin --delete 分支名    删除远程分支


## git rebase


git rebase --onto dev server client   检出client，找出client 分支和server分支的共同祖先之后的变化，然后把他们在dev上重演一遍

git rebase [主分支] [特性分支] 检出特性分支，然后在主分支上重演
git rebase dev server  检出server分支，然后在dev分支上重演

永远不要rebase 那些已经推送到公共仓库的更新

## git reflog
查看引用日志，可以理解为查看你最近几个月在当前项目中使用的git 命令


## git blam

git blam simplegit.rb  显示文件中对没一行进行修改的最近一次提交，

git blam -L 12,22 simplegit.rb 查看文件simplegit.rb 的每一次提交，使用-L 选项限制输出范围在第12行到22行

## .gitignore文件不起作用处理
1. 先清除 本地 git 缓存
git rm -r --cached .     

2. 重新 add 和 commit 就行了
git add .

git commit


---

搬运地址：    

Pro  Git 简体中文版
