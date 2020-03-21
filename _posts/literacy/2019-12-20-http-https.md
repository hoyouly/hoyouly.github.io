---
layout: post
title: 扫盲系列之---Socket,HTTP,HTTPS
category: 扫盲系列
tags: Socket HTTP HTTPS
---
* content
{:toc}

在上一篇文章中[ 扫盲系列之---TCP/IP ](http://hoyouly.fun/2018/04/11/tcp-ip/)，我们知道以下几点
# HTTP
超文本传输协议(Hypertext Transfer Protocol)  简称HTTP，这里面已经带有协议二字了，如果再说HTTP协议，那么就是超文本传输协议协议。重复了，
是一个在计算机世界里面 专门在两点之间传输文字，图片，音频，视频等超文本数据的约定和规范。

* HTTP 位于应用层
* HTTP 是TCP/IP 参考模型中应用层的一种实现。
* HTTP 网络层是基于IP协议，传输层是基于TCP协议
* HTTP的头部加上HTTP数据共同组成了TCP的数据部分
那么HTTP协议到底是啥呢，


## HTTP的报文
HTTP的报文有两种
* 请求报文---从客户端向服务器发送的报文
* 响应报文---从服务器到客户端的回答  

![](../../../../images/http_message_formate.png)

从网络中抓取的实际例子如下

![](../../../../images/http_data_eg.png)  


请求报文和响应报文都是由三部分组成
* 开始行  用于区分请求报文还是响应报文，请求报文的开始行叫请求行（request-line）,响应报文的开始行叫状态行（status-line）,在开始行的三个字段都是以空格隔开，最后以CR和LF分别表示回车和换行    ，
* 首部行  用来说明浏览器，服务器或报文的一些信息，首部可以有好几行，也可以没有，在每一个首部行中都有首部字段名和值，每一行在结束的位置都要有回车和换行，整个首部行结束后，还有一个空行用来区分首部行和实体主体
* 实体主体  在请求报文中一般都不用这个字段，而在响应报文中也可能没有这个字段。

## HTTP 传输数据的方式。

我们知道。 请求头和响应头一般是对请求和响应的整体描述，不做任何与数据相关的行为。状态消息用来描述服务端本次相应的状态，是HTTP层面的逻辑状态，因此请求头，响应头，状态消息都不适合用来传递数据

所以，可以把 参数可以写在URL和body上

### 参数写在 URL上
下面是一个URL的结构图

![](../../../../images/url_info.png)  

* http://www.example.com:8080 是用来描述 TCP/IP 需要的信息，
  * schme表示是否使用SSL建立连接，
  * host表示服务端域名或者 IP，
  * port表示连接的服务端端口。
* /aa/bb/cc?name=kalle&age=18#usage用来描述资源，
  * path表示本次请求的资源位置，
  * query表示这个资源的特征，
  * fragment是锚点片段，只用来指导浏览器动作，在 HTTP 请求中无效

### 参数写在Body 中
body 内容是字节流，因此可以写任何内容

根据HTTP协议，更加规范了body内容，大体分为三类，URL参数，任意流（可以是JSON，字符串，文件等）和表单

<style>
table th:first-of-type {
	width: 120px;
}
table th:nth-of-type(3) {
  	width: 180px;
}
</style>
|类型|body 拼接方式(举例) |请求头Content-Type|
|:----|:------|:------|
|发送 URL 参数|name=harry&gender=man|Content-Type: application/x-www-form-urlencoded|
|JSON格式|{ "name": "harry", "gender": "man" }|Content-Type: application/json|
|任意字符串数据|你怎么这么好看？|Content-Type: text/plain|
|发送已知类型的文件|~~~~~~~~~~~~~~~~~~~~~~~~~|Content-Type: image/jpeg|
|未知类型的数据或者文件|~~~~~~~~~~~~~~~~~~~~~~~~~|Content-Type: application/octet-stream|
|表单数据| 按照表单格式上传就可以传递复杂的数据，比如可以在表单中全部传递字符串，也可以全部是文件，也可以是文件和字符串的混合，表单实际上可以理解为一个对象键值对|Content-Type: multipart/form-data; boundary={boundary}|


## HTTP 响应码
常见的响应码有以下几种
### 100 段
* 100 (继续) 请求者应当继续提出请求。 服务器返回此代码表示已收到请求的第一部分，正在等待其余部分。
* 101 (切换协议) 请求者已要求服务器切换协议，服务器已确认并准备切换。
### 200 段
* 200 请求成功，服务器已成功处理了请求。 通常，这表示服务器提供了请求的网页。
* 201 (已创建) 请求成功并且服务器创建了新的资源。
* 202 (已接受) 服务器已接受请求，但尚未处理。
* 203 (非授权信息) 服务器已成功处理了请求，但返回的信息可能来自另一来源。
* 204 (无内容) 服务器成功处理了请求，但没有返回任何内容。
* 205 (重置内容) 服务器成功处理了请求，但没有返回任何内容。
* 206 用于下载文件时的断点续传。假设服务端的一个文件有200 byte，客户端下载到100 byte时中断，下次想从100byte处继续下载，在发起请求时添加请求头Range: bytes=100-，如果服务端支持这个操作，那么返回响应码 206，并且在响应头中添加Content-Range: 100-199/200，此时客户端读到的数据就是第 101 个字节到第 200 个字节。
### 300 段
* 302 重定向到其他URL
* 304 读取缓存。当某个 URL 的请求返回Cache-Control: public, max-age={int}，Last-Modified: Wed, 21 Oct 2020 07:28:00 GMT时，客户端会缓存本次请求到的数据，在2020年10月21日7点28分前对该 URL 的请求都不会连接到服务端，而是直接读取缓存数据。在该时间点之后会请求服务端，但是加上请求头If-Modified-Since: Wed, 21 Oct 2020 07:28:00 GMT，服务端会拿该请求头的时间和该 URL 对应的数据的修改时间做对比，如果数据发生了变化了则返回200和新数据，如果发现数据没有变化，则仅仅返回304，客户端此时还应该读取缓存。
300 (多种选择) 针对请求，服务器可执行多种操作。 服务器可根据请求者 (user agent) 选择一项操作，或提供操作列表供请求者选择。
* 301 (永久移动) 请求的网页已永久移动到新位置。 服务器返回此响应(对 GET 或 HEAD 请求的响应)时，会自动将请求者转到新位置。
* 303 (查看其他位置) 请求者应当对不同的位置使用单独的 GET 请求来检索响应时，服务器返回此代码。
* 305 (使用代理) 请求者只能使用代理访问请求的网页。 如果服务器返回此响应，还表示请求者应使用代理。

### 400 段
* 401 表示需要账号密码，或者账号密码错误
* 403 权限不足
* 404 找不到网页
* 405 客户端指定的方法不允许
* 406 服务端的内容是客户端不能接收的，一般是指服务端指定的Content-type与客户端指定的Accept不能匹配
* 408 (请求超时) 服务器等候请求时发生超时。
* 414 (请求的 URI 过长) 请求的 URI(通常为网址)过长，服务器无法处理。
* 415 客户端的内容是服务端不能接受的，一般是指客户端指定的Content-type 与服务端的API不一直
* 416 服务端不支持客户端指定的Range请求头，常用语断点续传。

### 500 段
* 500 服务端发生了未知异常
* 502 (错误网关) 服务器作为网关或代理，从上游服务器收到无效响应。
* 503 (服务不可用) 服务器目前无法使用(由于超载或停机维护)。 通常，这只是暂时状态。
* 504 (网关超时) 服务器作为网关或代理，但是没有及时从上游服务器收到请求

## HTTP请求和响应头
常见的有一下几种

* Content-Range  响应头  前面的俩数字表示开始字节和结束字节的posiiton，后面的数字表示文件的总大小。例如 Content-Range: 100-199/200，此时客户端读到的数据就是第 101 个字节到第 200 个字节，一共200 字节
* Range 请求头，指定要下载的一段数据。例如 Range: bytes=100-149 ，客户端指定下载第 101 个字节到第 150 个字节
* Accept 请求头，表示客户端对服务端内容的期望类型，比如application/json，当服务端生成的内容不是 JSON 时，那么可能收到 406 响应码。
* Content-Type 在请求头和响应头中都比较常用，一般表示本次body 的内容类型或者格式，服务端或者客户端应该按照该格式来解析数据。
* Connection 表示该请求处理完后是否要关闭连接，在 HTTP1.0 中默认是close，表示要关闭连接，在 HTTP1.1 中默认是keep-alive，表示保持连接
* Content-Length  表示内容的长度，这个值对读取内容非常有用，客户端或者服务端可以根据以最优的方法读取流，可以极大的提高读取 IO 的效率。
* Referer 当前页面的来源地址，例如从https://www.google.com页面访问了https://github.com，那么 Referer 的值则是https://www.googlt.com。
* Host 每一个请求中都必须带上该头，表示客户端要请求的服务端的域名和端口，一般 Host 中没有指定端口时，HTTP 应用程序默认使用 80 端口连接服务端。
* Cookie 服务端用来标记客户端，比如用户访问过某页面，那么给他一个标记Set-Cookie: news=true; expires=...; path=...; domain=...，当用户下次请求该页面时，请求头会带上Cookie: news=true，服务端会做一些逻辑处理。

## HTTP 通信的过程

就是出栈和入栈的过程

![](../../../../images/http_commite.jpeg)

报文从 应用层到传输层，然后到网络层，链路层，传输层通过TCP三次握手建立连接，四次挥手释放连接。

说了这么多，还是有一个事情没搞明白，那就是Socket是啥玩意，和HTTP啥关系

# Socket
* TCP/IP 协议需要向程序员提供可编程的 API，该 API 就是 Socket
* 它是对 TCP/IP 协议的一个重要的实现，几乎所有的计算机系统都提供了对 TCP/IP 协议族的 Socket 实现。
* 如果TCP/IP 协议类似公司的企业文化，那么Socket 就是劳动合同。
* 是对TCP/IP的一种封装，当然Socket还可以指定其他协议，例如 UDP。
* 我们可以直接new 一个Socket对象，但是却不能new 一个TCP/IP
* Socket 本身不算是协议，只是提供了真的TCP或UDP编程的接口
* HTTP是轿车，提供了封装或显示的具体形式。Socket是发动机，提供了网络通信能力


## Socket 存在的意义

一般情况下，HTTP的客户端或者服务端都是基于Socket来实现的。
而像 Ngnix ,Apache,Chrome和IE等软件，都是基于HTTP服务端或者客户端开发的。
Ngnix 是基于Socket实现的HTTP 服务端，OkHttp,URLConnection等都是基于Socket实现的HTTP 客户端，而浏览器是这些HTTP客户端的具象

## HTTP 存在的意义

虽然 HTTP协议是基于Socket实现的，那为啥不直接使用Socket呢，而是要用HTTP协议呢。
因为HTTP 协议 方便便捷，包括数据格式，数据分割，数据传输和缓存策略等，如果要为每一个应用单独做一套这样的逻辑，还不如设计一套规范

## 建立Socket链接
建立Socket链接至少需要一对套接字，其中一个运行到客户端，叫做ClientSocket，另一个运行在服务器端，称为ServerSocket，
套接字之间的链接过程分为三个步骤:
* 服务器监听 ： 服务端套接字并不定位具体的客户端套接字，而是处于等待链接状态，实时监听网络状态，等待客户端套接字的请求
* 客户端请求  ： 客户端套接字提出链接请求，要链接的目标是服务端套接字，为此客户端套接字必须要描述它链接的服务端套接字的，支出服务端套接字的端口和IP，然后向服务端套接字提出请求。
* 链接确认  ：  当服务端套接字监听到或者接收到客户端套接字的请求时，就相应客户端套接字的请求，建立一个新的链接，把服务端的套接字的描述发送给客户端套，一旦客户端确认了此描述，双方就正式建立链接，而服务端套接字依旧处于监听状态，继续接受其他客户端套接字的链接请求。










# HTTPS
由SSL和HTTP协议构建的可加密传输，身份认证的网络协议，以安全为目的的HTTP通道，基础是SSL，需要到CA申请认证书，一般免费比较少，因而需要一定的费用。
完全不同的连接方式，端口也不一样，HTTP 端口是80，HTTPS的端口是443

这套证书就是 一对公钥和私钥，
HTTP 协议传输数据是通过明文显示的
HTTPS 是通过SSL或者TLS加密处理数据，更安全

网上看了一些资料，例如信鸽解释HTTPS，还有其他的一些，最终自己画了一个类似的流程图，有请大名鼎鼎的小明同学登场。

![](../../../../images/https_flow.png)

小明同学 要和小红同学通信，
1. 小明同学给小红同学寄一个信鸽，但是不带任何信息，
2. 小红同学收到小明同学的信鸽，然后找一个老师买了一个带锁的盒子, 这个一般都是提前买好的。
3. 小红同学就用信鸽把这个盒子捎过去了，这个时候盒子是没锁的，开着的
4. 小明同学收到这个盒子，会找老师确认，这个是不是小红买的盒子，
5. 老师通过各种方式确认是小红的盒子，没有经过任何改变。
6. 小明同学就会把加密方式放到这个盒子中。然后锁上盒子让信鸽带给小红
7. 小红同学收到上锁的盒子，拿着唯一的钥匙（私钥），打开这个盒子，得到加密方式，
8. 这样小红和小明同学都有了加密方式，就可以通过对称加密，先加密，然后发送数据，对方再解密，得到数据，加密，发送加密数据，解密，这样就可以愉快的聊天了，中间就算数据被别人拿到了，可是只要加密方式足够复杂，就能保证数据的安全。

这里面的小明同学就是客户端，小红同学就是服务端，老师就是CA，
服务器会提前向CA买证书，证书就是一对秘钥，公钥（带锁的盒子）和私钥（钥匙），
每个证书是唯一的，都是带有过期日期和序列号，申请者的姓名等，然后把这些进行数字签名，修改任何一个地方就会导致数字签名不一致的。
把一段文字和私钥进行数字签名，然后把消息和数字签名发给客户端

# HTTP 与HTTPS

* HTTPS  端口号 443
* HTTP 端口号 80
* HTTP+SSL/TSL =HTTPS
* HTTPS的内核是HTTP，只是HTTP通信接口部分由TSL/SSL 代替，
* 通常情况下，HTTP会直接和TCP通信，在使用了TSL的HTTPS中，HTTP会先和TSL通信，再由 SSL和TCP通信

## SSL/TLS

SSL是一个独立的协议，不只有HTTPS协议中使用，其他应用层协议也可以使用，比如SMTP（电子邮件协议，），Telnet(远程登录协议)等

TSL 是SSL的后续版本。在互联网两台机器之间用于身份验证和加密的一种协议

位于OSI模型的第五层。

TSL常用的对称加密算法 AES-128, AES-192、AES-256 和 ChaCha20

AES 的全称是Advanced Encryption Standard(高级加密标准)，它是 DES Data Encryption Standard(数据加密标准) 算法的替代者，安全强度很高，性能也很好，是应用最广泛的对称加密算法。

非对称加密

公钥进行加密，私钥进行解密。公开密钥可供任何人使用，私钥只有你自己能够知道。

常见的算法 DH、DSA、RSA、ECC

 TSL 使用非对称加密解决秘钥交换的问题，然后用随机数产生对称算法使用的回话秘钥，再用公钥加密，对方拿到密文后使用私钥解密，取出回话秘钥，这双方就实现了对称秘钥的安全交换。

摘要算法  一种特殊的压缩算法，他能把任意长度的数据压缩成一个固定长度的字符串，

SSL/TLS 通过将称为 X.509证书的数字文档将网站和公司的实体信息绑定到加密密钥来进行工作。每一个密钥对(key pairs) 都有一个 私有密钥(private key) 和 公有密钥(public key)，私有密钥是独有的，一般位于服务器上，用于解密由公共密钥加密过的信息；公有密钥是公有的，与服务器进行交互的每个人都可以持有公有密钥，用公钥加密的信息只能由私有密钥来解密。

X.509   公开秘钥证书的标准格式。这个文档将加密秘钥与个人或组织进行安全关联


---
搬运地址：  
[HTTP和HTTPS协议，看一篇就够了](https://blog.csdn.net/xiaoming100001/article/details/81109617)   
[Android中的TCP/IP协议，Socket，Http协议间的关系](https://blog.csdn.net/u010618194/article/details/62439168)   
[Android客户端面试基础(四)-TCP/IP](https://blog.csdn.net/johnWcheung/article/details/72835044)     
[用信鸽来解释 HTTPS](https://www.oschina.net/translate/https-explained-with-carrier-pigeons)   
[深入浅出HTTPS基本原理](https://blog.csdn.net/kobejayandy/article/details/52433660)   
[(建议收藏)TCP协议灵魂之问，巩固你的网路底层基础](https://juejin.im/post/5e527c58e51d4526c654bf41)     
[HTTP 协议理解及服务端与客户端的设计实现](https://yanzhenjie.blog.csdn.net/article/details/93098495)    
[看完这篇 HTTPS，和面试官扯皮就没问题了](https://mp.weixin.qq.com/s/PW4X6WL0t6YJiuXC-dSdKg)    
[HTTP常见状态码 200 301 302 404 500](https://www.cnblogs.com/starof/p/5035119.html)