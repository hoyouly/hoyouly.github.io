---
layout: post
title: 免输密码登录服务器的Shell 脚本
category: 工具
tags: 免输密码 服务器 Shell脚本
---
* content
{:toc}

## 免输密码脚本
一个 免输密码登录服务器的脚本如下，记得改成用户名和密码以及服务器端口：
```shell
#!/bin/bash
src_host=172.16.20.137
src_pwd=123456
src_user=helei
expect -c "
spawn ssh ${src_user}@${src_host}
expect \"password:\"
send \"${src_pwd}\r\"
interact
"
```


##  任何路径下都能执行的基本步骤：
1. 创建一个目录，例如当前用户下面创建一个bin 目录 /Users/hoyouly/bin

2. 把该目录添加到环境变量中，mac上是在当前用户下面的 .bash_profile 文件中最后添加 `export PATH=$PATH:/Users/hoyouly/bin` ，Linux用户自行查找。

3. 把写好的shell 脚本放到该文件夹下面，可以不带后缀名的，例如命名为lg ，这个lg就是你要执行的命令 ，注意：别直接使用login这个单词，我试了两次都没成功，不知道为啥。

4.  添加可执行权限， `sudo chmod -R 777 lg`
