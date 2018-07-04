---
layout: post
title: Atom 使用技巧集合
category: 工具
tags: Atom  快捷键
---

* content
{:toc}

之前使用的轻量级编辑器一直是Sublime，但是因为种种原因吧，决定放弃Sublime，转而使用Atom上，所以就单独新建了一个博客，用来记录使用Atom上的技巧和快捷键之类的。


## 快捷键

|平台|Mac |Win/Linux|
|:----|:------|:------|
|复制当前行到下一行|cmd-shift-D|Ctrl + shift + D|
|删除当前行|ctrl-shift-K|Ctrl + shift + K|
|增加新光标|cmd-click|Ctrl-click |
|相应括号之间，html tag之间等跳转|ctrl-m|Ctrl-m |
|Markdown预览|ctrl-shift-M|Ctrl-shift-m |
|	查找文件|cmd-t 或 cmd-p|Ctrl-t 或 Ctrl-p |
|	显示或隐藏目录树|cmd-\|Ctrl-\ |
|	打开目录|cmd-shift-o|Ctrl-shift-o |

## Atom在mac上崩溃
产生原因：在mac上使用shadowsocks的PAC模式就会使atom闪退，全局模式就不会闪退
解决方案：
1. 关闭代理
2. 打开设置 -> Core -> 往下拉， 取消Use Proxy Settings When Calling APM的勾选就可以了

搬运地址：

[Atom快捷键整理](https://www.jianshu.com/p/e33f864981bb)  
[atom在mac上闪退](https://atom-china.org/t/topic/5735)
