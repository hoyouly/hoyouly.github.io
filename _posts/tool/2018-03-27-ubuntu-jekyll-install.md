---
layout: post
title: Ubuntu 14.04 安装 jekyll
category: 工具
tags: Ubuntu jekyll install
---

* content
{:toc}

先执行下面命令，
```shell
sudo apt-get update
sudo apt-get upgrade
//这个命令可能需要，
sudo apt-get install ruby
```

然后执行
```shell
sudo apt-get install rubygems
```
结果竟然报错了
```
Reading package lists... Done
Building dependency tree       
Reading state information... Done
Package rubygems is not available, but is referred to by another package.
This may mean that the package is missing, has been obsoleted, or
is only available from another source
However the following packages replace it:
  ruby

E: Package 'rubygems' has no installation candidate
```
然后在网上查找，需要使用 下面的命令，

```shell
sudo apt-get install rubygems-integration

 ```
然后就是开始安装 jekyll 吧
```shell
sudo gem install jekyll
```
然后又报错了，错误信息如下。真是一步一个坑啊。
```shell
ERROR:  Error installing jekyll:
	public_suffix requires Ruby version >= 2.1.
```
于是我查看了一下自己的Ruby 版本，ruby -v
结果发现还真是低啊，才1.9.3 那就想办法升级Ruby 的版本吧，然后就找到了下面的命令

```shell
dpkg --get-selections | grep -i ruby
```
当提示下面的信息的时候，说明最新版本的Ruby安装成功了
```shell
Install of ruby-2.4.1 - #complete
Ruby was built without documentation, to build it run: rvm docs generate-ri
Creating alias default for ruby-2.4.1...

  * To start using RVM you need to run `source /home/hoyouly/.rvm/scripts/rvm`
    in all your open shell windows, in rare cases you need to reopen all shell windows.
```
虽然英语不咋地，但是发现
1.  Install of ruby-2.4.1 - #complete  说明Ruby 2.4.1 版本安装成功
2. To start using RVM you need to run `source /home/hoyouly/.rvm/scripts/rvm ，需要 运行 下面命令，那就运行吧
```shell
source /home/hoyouly/.rvm/scripts/rvm
```
为了保险期间，还是查看Ruby 版本

```shell
ruby -v
//结果如下
ruby 2.4.1p111 (2017-03-22 revision 58053) [x86_64-linux]

```
版本号变了，2.4.1 ，版本好变了就行，那就继续安装jekyll吧，

```shell
sudo gem install  jekyll
```
TNN 的又报错了，还是和之前的错误一样，WTF，不是版本号已经变了，怎么还提示`public_suffix requires Ruby version >= 2.1`这个错误呢，
终于在github上找到原因了，

原来尽管你 `ruby -v` 得到的版本号是 2.4.1 ，可是当你  `sudo ruby -v`   结果还是1.9.3 而你安装jekyll的时候，使用了sudo 命令，会自动查找 sudo ruby -v 对应的版本，所以安装jekyll的时候，不使用sudo 就可以了，  
具体原因参考  [glynhudson 的回答](https://github.com/jekyll/jekyll/issues/4724)
那就执行 真正的安装jekyll的命令吧,`注意注意，千万不能加 sudo`

```shell
gem install  jekyll
```
终于没报错了，提示下面下面信息

```shell
Installing ri documentation for jekyll-3.7.3
Done installing documentation for public_suffix, addressable, colorator, http_parser.rb, eventmachine, em-websocket, concurrent-ruby, i18n, rb-fsevent, ffi, rb-inotify, sass-listen, sass, jekyll-sass-converter, ruby_dep, listen, jekyll-watch, kramdown, liquid, mercenary, forwardable-extended, pathutil, rouge, safe_yaml, jekyll after 32 seconds
25 gems installed

```
同样，为了保险期间，还是查看一个jekyll 的版本号

```shell
jekyll -v

//结果如下
jekyll 3.7.3
```
这次就说明真的在Ubuntu上安装成功jekyll了，那就试一把把
进入到我的bolg文件目录下面，执行

```shell
jekyll build
```
原本以为水到渠成的事情，结果有出错了，

```shell
Deprecation: The 'gems' configuration option has been renamed to 'plugins'. Please update your config file accordingly.
Dependency Error: Yikes! It looks like you don't have jekyll-sitemap or one of its dependencies installed. In order to use Jekyll as currently configured, you'll need to install this gem. The full error message from Ruby is: 'cannot load such file - jekyll-sitemap' If you run into trouble, you can find helpful resources at https://jekyllrb.com/help/!

```
Deprecation 这个先不管它，可是下面却有一个 error message ，这是为啥呢，继续百度，Google咯，既然`cannot load such file - jekyll-sitemap` ，那我就install一个这样的文件不就得了，
```shell
gem install  jekyll-sitemap
```
然后继续 jekyll build 的时候，还遇到了相同的错误。只不过缺少的文件名不同，那就照猫画虎咯，继续安装缺少的文件
```shell
Dependency Error: Yikes! It looks like you don't have jekyll-feed or one of its dependencies installed. In order to use Jekyll as currently configured, you'll need to install this gem. The full error message from Ruby is: 'cannot load such file - jekyll-jekyll-seo-tag' If you run into trouble, you can find helpful resources at https://jekyllrb.com/help/!


Dependency Error: Yikes! It looks like you don't have jekyll-feed or one of its dependencies installed. In order to use Jekyll as currently configured, you'll need to install this gem. The full error message from Ruby is: 'cannot load such file - jekyll-feed' If you run into trouble, you can find helpful resources at https://jekyllrb.com/help/!

```

```shell
gem install  jekyll-seo-tag

gem install  jekyll-feed
```
接下来继续执行

```shell
jeklly build
```
没有报错了，然后启动服务

```shell
jekyll server
```
在浏览器中 输入： http://localhost:4000/ 完美。


说明：
配置中出了各种问题，也查了一些资料，可能不完全，也忘记了都出自那个博客上的了，就不写搬运地址了，有对不住的地方，见谅。
