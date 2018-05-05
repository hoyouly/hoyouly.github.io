---
layout: post
title:  我的代码块也能显示行数和高亮
category: 工具
tags: jekyll  prismjs
---
* content
{:toc}

## 代码块滑动
使用现在这个模板后，一直对一个地方不满意，那就是代码显示这一块
可以参照下面这个图片，这是我以前的代码块显示
![](http://p5sfwb51p.bkt.clouddn.com/old_cold_block.png)
主要是在红框那一段，我很不喜欢原本一行的代码给我折行的，特别喜欢那种带有滑块的效果，怎么处理呢，怎么才能不让折行。大学学的那点HTML，CSS，JS早考试完都物归原主，还给老师了，那就先看看网上有没有现成的能滑动博客吧，然后就找到了我之前用的那个模板中发现还真是这样的，
![](http://p5sfwb51p.bkt.clouddn.com/yansu_code.png)
这个代码块就能滑动，而且看上去比我的代码块好看，他是怎么做呢，然后发现他自己的博客中有关于代码块高亮的记录 [Jekyll的中的代码高亮](http://yansu.org/2013/04/22/highlight-of-jekyll.html) 可是我看完后发现还是找到怎么让代码块滑动的，至于高亮显示，我也没弄出来，因为我在他的整个项目搜prettify关键字，结果发现为null，不知道他是怎么实现的，给他留言了，希望他能看到给我解惑。既然他这个能实现滑动，那我就二分查找，在本地jekyll 服务器运行，也发现了点苗头
在main.scss中有这些设置
```CSS
$post-code-inline-color: lighten($post-text-color, 10%)
$post-code-inline-bg: darken($post-bg, 10%)
$post-code-font: 'Droid Sans Mono', monospace

$post-code-block-bg: #282a36
```
看着像是关于代码块的设置，那就看看什么地方用了这些设置吧，然后就找到了05-post.sass 文件，
```CSS
pre
  margin-top: 0
  margin-bottom: $post-paragraph-spacing
  padding: 1em
  line-height: 1.3
  background: $post-code-block-bg
  overflow: auto
  -webkit-overflow-scrolling: touch
  &::-webkit-scrollbar
    height: $code-scrollbar-height
  &::-webkit-scrollbar-thumb
    background: $code-scrollbar-color
    &:hover
      background: darken($code-scrollbar-color, 10%)
```
刚开始以为是scrollbar 之类设置的，英文也不咋地，但是scrollbar 还是认识的，不就是滚动条嘛，可是删除后还是可以，后来就查其他属性，然后就找到了 overflow ，这个才是真正设置，overflow属性的介绍可以 参考 [CSS overflow 属性](http://www.w3school.com.cn/cssref/pr_pos_overflow.asp),
那就在我的博客中找找看有没有设置 overflow: auto 东西吧，从哪里找呢，人家是main.scss ,那我也从我的main.scss 中找吧，结果发现就

```CSS
@import "reset", "header", "post", "page", "syntax-highlighting", "index", "demo", "footer", "scrollbar", "backToTop";
```
看样子是引入了几个文件，尽管英语很烂，但是却也知道syntax-highlighting 是语法高亮的意思，看着像是处理代码块的，那就进去看看吧。找到syntax-highlighting.scss文件，
发现了点苗头
```CSS
pre {
    margin: 12px 0;
     padding: 8px 12px;
    //  overflow-x: auto;
     word-wrap: break-word;      /* IE 5.5-7 */
     white-space: -moz-pre-wrap; /* Firefox 1.0-2.0 */
     white-space: pre-wrap;      /* current browsers */
     > code {
         margin: 0;
         padding: 0;
        font-size: 13px;
        color:#d1d1c9;
        border: none;
        background-color: #272822
     }
}
```
overflow-x: auto 看着和overflow: auto 有点像，而且还是注释掉的，二话不说，先打开注释再说，然后失望了，，于是我就又把 >code {}里面添加上overflow: auto，再一次失望， 难道我定位错了，我该了一下font-size: 的大小，代码字体大小确实变了啊，运行发现有效果啊，说明我定位的没错，那为啥overflow: auto没生效呢，问度娘和谷哥吧，也没得到的什么具体的答案，但是我感觉像是和某个属性冲突了，那就一个一个注释属性，反正也没几个，然后就发现，如果把 white-space 注释掉，就显示滑块了，white-space属性具体用法还是参照W3School，[CSS white-space 属性](http://www.w3school.com.cn/cssref/pr_text_white-space.asp),终于可以让我的代码快滑动了，开心一把，可是看其他人的博客，代码都显示行数，至于高亮，这个我要求并不高，但是我却很想自己的代码块也显示行数，这样逼格更高一些啊，
## 显示行数，高亮，一键复制
然后继续度娘+谷哥，找到了这样一篇文章，[使用prismjs实现Jekyll代码语法高亮并显示行号](http://wanshicheng.org/jekyll/%E4%BD%BF%E7%94%A8prismjs%E5%AE%9E%E7%8E%B0Jekyll%E4%BB%A3%E7%A0%81%E8%AF%AD%E6%B3%95%E9%AB%98%E4%BA%AE%E5%B9%B6%E6%98%BE%E7%A4%BA%E8%A1%8C%E5%8F%B7.html),这很和我的胃口哦。





---
搬运地址：   
[使用prismjs实现Jekyll代码语法高亮并显示行号](http://wanshicheng.org/jekyll/%E4%BD%BF%E7%94%A8prismjs%E5%AE%9E%E7%8E%B0Jekyll%E4%BB%A3%E7%A0%81%E8%AF%AD%E6%B3%95%E9%AB%98%E4%BA%AE%E5%B9%B6%E6%98%BE%E7%A4%BA%E8%A1%8C%E5%8F%B7.html)
