---
layout: post
title: Linux 卸载和安装Atom
category: 工具
tags: Linux ，卸载Atom
---
之前在Linux上一直使用的是Sublime，可是不知道为啥，最近一直提示我要升级，而且最可恼的是，使用Sublime 打开txt文件，背景竟然是白色的，网上查了半天，也没找到解决方案，解决不了，哥不用了呗，Atom之前都听说过，而且也装的有，只是觉得他打开太慢了，所以就一直没用，现在重新玩Atom吧，可是我安装的版本比较老，是1.13版本，可是最新的版本却是1.25.0 了，那就想办法更新吧，竟然没找到更新按钮，直接卸载重装吧。

## 卸载Atom
之前使用下面三个命令行卸载，但是感觉好像没卸载干净
```
sudo apt-get remove atom
sudo add-apt-repository --remove ppa:webupd8team/atom
sudo apt-get autoremove
```

然后又在网上找到了下面几个命令，这次彻底卸载干净了。
```
sudo rm /usr/local/bin/atom
sudo rm /usr/local/bin/apm
rm -rf ~/atom
rm -rf ~/.atom
rm -rf ~/.config/Atom-Shell
sudo rm -rf /usr/local/share/atom/

```
## 安装Atom

使用命令行安装的时候，不知道为啥，我安装的版本不是最新的，目前atom官网上最新的Linux版本是1.25.0版本，可是使用下面命令行安装的时候，版本号却是 1.24.6 好像是这个，具体忘记了，肯定不是最新的。

```
sudo add-apt-repository ppa:webupd8team/atom  
sudo apt-get update  
sudo apt-get install atom  
```
强迫症的我非要安装一个最新的Atom版本，咋办呢，简单，直接在Atom网站下载一个最新的安装包。例如“atom-amd64.deb”，然后直接点击安装就可以了。

摘抄：

[Ubuntu16.04安装/卸载Atom](http://blog.csdn.net/bleachswh/article/details/51684821)

[在Linux上，如何卸载 Atom 文本编辑器？](https://ask.helplib.com/atom-editor/post_1052428)
