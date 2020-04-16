---
layout: post
title: 扫盲系列 - buildscript 和 allprojects 的区别
category: 扫盲系列
tags:  gradle
---
* content
{:toc}

通过AndroidStudio 创建一个项目，就会生成两个gradle文件，一个位于项目根目录下面，一个位于app下面，
app 下面的gradle文件我们还经常修改，添加依赖库什么的，可是根目录下面的就不怎么修改了，
这里面有两个感觉很像，buildscript 和 allprojects 什么意思啊，中间还都有 repositories 这个。

```java
buildscript {
    //声明gardle脚本自身所需要使用的资源，包括依赖项、maven仓库地址、第三方插件
    repositories {
        google()
        jcenter()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:3.1.1'
    }
}

allprojects {
    repositories {
        google()
        jcenter()
        maven {url 'https://raw.github.com/hoyouly/maven-repository/master'}
    }
}

task clean(type: Delete) {
    delete rootProject.buildDir
}
```
# buildscript
构建用的。声明gardle脚本自身所需要使用的资源，包括依赖项、maven仓库地址、第三方插件等。你可以在里面手动添加一些三方插件、库的引用，这样你就可以在脚本中使用它们了。因为是引用，所以gradle在执行脚本时，会优先执行buildscript代码块中的内容。
* repositories 表示只有编译工具才会用这个仓库
* dependencies classpath 'com.android.tools.build:gradle:3.1.1' 表示使用的gradle 的版本是3.1.1

#  allprojects
* repositories用于多项目构建，为所有项目提供共同所需依赖包。而子项目可以配置自己的repositories以获取自己独需的依赖包。
* 比如子项目里面的 support 依赖库，就会去 allprojects 的repositories 去找。
* maven {url 'https://raw.github.com/hoyouly/maven-repository/master'} 就表示项目中用到了我自己的maven，


---
搬运地址：

[buildscript和allprojects的作用和区别是什么？](https://www.jianshu.com/p/ee57e4de78a3)

[project 目录build.gradle -> buildscript和 allprojects](https://www.jianshu.com/p/f6adc305ff90)
