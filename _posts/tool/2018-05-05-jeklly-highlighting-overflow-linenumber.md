---
layout: post
title:  我的代码块也能显示行数，滑动，高亮和一键复制
category: 工具
tags: jekyll  prismjs
---
* content
{:toc}

## 代码块滑动
使用现在这个模板后，一直对一个地方不满意，那就是代码显示这一块，他不能滚动，
可以参照下面这个图片，这是我以前的代码块显示
![](https://github.com/hoyouly/BlogResource/blob/master/imges/old_cold_block.png)
主要是在红框那一段，我很不喜欢原本一行的代码给我折行的，特别喜欢那种带有滑块的效果，怎么处理呢，怎么才能不让折行。大学学的那点HTML，CSS，JS早考试完都物归原主，还给老师了，那就先看看网上有没有现成的能滑动博客吧，然后就找到了我之前用的那个模板中发现还真是这样的，
![](https://github.com/hoyouly/BlogResource/blob/master/imges/yansu_code.png)
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
刚开始以为是scrollbar 之类设置的，英文也不咋地，但是scrollbar 还是认识的，不就是滚动条嘛，可是删除后还是可以滑动，后来就查其他属性，然后就找到了 overflow ，这个才是真正设置代码块是否滑动的，overflow属性的介绍可以 参考 [CSS overflow 属性](http://www.w3school.com.cn/cssref/pr_pos_overflow.asp),既然找到了原因，那就看看我的博客中找找看有没有设置 overflow: auto 吧，从哪里找呢，人家是main.scss ,那我也从我的main.scss 中找吧，结果发现就

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
有点意思。overflow-x: auto 看着和overflow: auto 有点像，而且还是注释掉的，二话不说，先打开注释再说，然后失望了，于是我就又把 >code {}里面添加上overflow: auto，再一次失望， 难道我定位错了，为了验证我是否定位出错，我修改了一下font-size: 的大小，运行发现有效果啊，代码字体大小确实变了啊，说明我定位的没错，那为啥overflow: auto没生效呢，问度娘和谷哥吧，也没得到的什么具体的答案，但是我感觉像是和某个属性冲突了，那就一个一个注释属性，反正也没几个，然后就发现，如果把 white-space 注释掉，就显示滑块了，white-space属性具体用法还是参照W3School，[CSS white-space 属性](http://www.w3school.com.cn/cssref/pr_text_white-space.asp),终于可以让我的代码快滑动了，小激动一把。   
可是看其他人的博客，代码都显示行数，至于高亮，这个我要求并不高，但是我却很想自己的代码块也显示行数，这样逼格更高一些。
## 显示行数，高亮，一键复制
然后继续度娘+谷哥，找到了这样一篇文章，[使用prismjs实现Jekyll代码语法高亮并显示行号](http://wanshicheng.org/jekyll/%E4%BD%BF%E7%94%A8prismjs%E5%AE%9E%E7%8E%B0Jekyll%E4%BB%A3%E7%A0%81%E8%AF%AD%E6%B3%95%E9%AB%98%E4%BA%AE%E5%B9%B6%E6%98%BE%E7%A4%BA%E8%A1%8C%E5%8F%B7.html),这很和我的胃口哦。然后就按照他里面说的去做，先进到[Prism 官网下载页面](http://prismjs.com/download.html),
1. 在Core中选择主题，可以点击看看显示效果，觉得那个符合自己的审美就选那个， 我选的是 Okaidia，
2. 在 Languages 选支持的语言，因为我是Android开发，就主要选了java,JSON,kotlin，Markdown还有Python这几种，
3. 在Plugins 里面选了Line Numbers，就是要这个的，所以必须得选，然后我就无意间看到了一个Copy to Clipboard  ，这个好像是复制剪贴板啊，感觉有用，于是也选上了，
4. 点击DOWNLOAD JS和DOWNLOAD CSS，

把下载好的prism.js 放到自己对应的js文件夹中，然而prism.css 我只是把里面的内容复制到了_syntax-highlighting.scss 中
### 是否引用jQuery
因为文章中说用到的jQuery，我都不知道自己里面用没用jQuery，前端那点知识早都忘记的一干二净了，没办法，继续度娘和谷哥吧，然后发现了判断引用JQuery的方法，就是看有没有引用下面的代码。
```java
<script>!window.jQuery && document.write("<script src=\"jquery.js\">"+"</scr"+"ipt>");</script>
```
那就在我的项目中搜关键字“jquery.js”把，发现还真没有，没事，引用上就可以了，于是我就在我的default.html 上引用上了
![](https://github.com/hoyouly/BlogResource/blob/master/imges/default_html.png)
红框里面的就是我改动的部分，然后再次运行，完美    
看效果图，红框里面就是主要的修改，显示行号，可以滑动，而且当鼠标落到这个区域的时候，右上角显示copy,点击copy就可以一键复制了，简直爽歪歪啊，
![](https://github.com/hoyouly/BlogResource/blob/master/imges/new_cold_block.png)
接下来再看看之前的样式吧，没有对比就没有伤害，
![](https://github.com/hoyouly/BlogResource/blob/master/imges/old_cold_block.png)
是不是高亮我就不知道了，看着顺眼就行，主题想换可以随时换，终于达到自己想要的效果了。忙了一天终于忙点效果。

---
搬运地址：     
[使用prismjs实现Jekyll代码语法高亮并显示行号](http://wanshicheng.org/jekyll/%E4%BD%BF%E7%94%A8prismjs%E5%AE%9E%E7%8E%B0Jekyll%E4%BB%A3%E7%A0%81%E8%AF%AD%E6%B3%95%E9%AB%98%E4%BA%AE%E5%B9%B6%E6%98%BE%E7%A4%BA%E8%A1%8C%E5%8F%B7.html)   
[判断页面中是否引用jQuery](https://www.cnblogs.com/snowbaby-kang/p/4815489.html)
