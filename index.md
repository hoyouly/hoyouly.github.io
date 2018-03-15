---
　　layout: default
　　title: 我的Blog
---
<!-- 它的Yaml文件头表示，
首页使用default模板，标题为"我的Blog"。
然后，首页使用了{% for post in site.posts%}，表示对所有帖子进行一个遍历。
这里要注意的是，Liquid模板语言规定，输出内容使用两层大括号，单纯的命令使用一层大括号。至于{{site.baseurl}}就是_config.yml中设置的baseurl变量。　 -->


　　<h2>{{ page.title }}</h2>

　　<p>最新文章</p>

　　<ul>

　　　　{% for post in site.posts %}

　　　　　　<li>{{ post.date | date_to_string }} <a href="{{ site.baseurl }}{{ post.url }}">{{ post.title }}</a></li>

　　　　{% endfor %}

　　</ul>

[Android 性能优化](https://hoyouly.github.io/Android性能优化.html)

[Activity的生命周期和启动模式](https://hoyouly.github.io/Activity的生命周期和启动模式.html)

[Android的线程和线程池](https://hoyouly.github.io/Android的线程和线程池.html)

[Bitmap的加载和Cache处理](https://hoyouly.github.io/Bitmap的加载和Cache处理.html)

[WiFi列表加载流程图](https://hoyouly.github.io/WiFi列表加载流程图.mdj)
