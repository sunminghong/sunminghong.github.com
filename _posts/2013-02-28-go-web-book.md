---
layout: post
categories: read,golang
title: 第一本golang(go语言)开源书籍《Go Web 编程》 - 好书推荐
description: 最近在学习go语言，相关的书籍很少，中文的就更少了。而 astaxie 编写并开源了 《Go Web编程》《Build Web Application with Golang》。
keywords: golang,go语言,web
---

### 引言
经过很久的“观望”我最终决定学习[golang](http://code.google.com/p/golang/) 。于是开始找资料和书籍，无意中看到 [采访：关于Go语言和《Go Web编程》](http://www.infoq.com/cn/articles/go-web-programming-interview) 一文后在github上 fork 了该书项目并开始研读《Build Web Application with Golang》，感觉该书由浅入深，通俗易懂，于是在此推荐给朋友们。

对于作者[astaxie](https://github.com/astaxie/)及该书的联合作者们的无私奉献，我无限钦佩之至！国内的同学们都如此，那。。。

### 1. 作者在github上开源了此书籍
 [《Go Web 编程》](https://github.com/astaxie/build-web-application-with-golang) 
 [作者astaxie](https://github.com/astaxie/)（在github上的页面）
 
### 2. 作者对《Go Web编程》介绍 

> # 《Go Web 编程》
> 因为自己对Web开发比较感兴趣，所以最近抽空在写一本开源的书籍《Go Web编程》《Build Web Application with Golang》。写这本书不表示我能力很强，而是我愿意分享，和大家一起分享Go写Web应用的一些东西。
> 
> - 对于从PHP/Python/Ruby转过来的同学了解Go怎么写Web应用开发的
> 
> - 对于从C/C++转过来的同学了解Web到底是怎么运行起来的
> 
> 我一直认为知识是用来分享的，让更多的人分享自己拥有的一切知识这个才是人生最大的快乐。
> 
> 这本书目前我放在Github上，我现在基本每天晚上抽空会写一些，时间有限、能力有限，所以希望更多的朋友参与到这个开源项目中来。
> 
> 
> ## 撰写方法
> ### 文件命名
> 每个章节建立一个md文件，如第11章的第3节，则建立**11.3.md**。
> ### 代码文件
> 代码文件置于src目录之下。每小节代码按目录存放。如第11章的第3节的代码保存于**src/11.3/**目录下。在正文中按需要添加代码。
> 
> ## 格式规范
> ### 正文
> 请参看已有章节的规范，要注意的是，每个章节在底部都需要有一个links节，包含“目录”，“上一节”和“下一节”的链接。
> ### 代码
> 代码要**`go fmt`**后提交。注释文件注明其所属章节。
> 
> ## 如何编译
> `build.go`依赖markdown的一个解析包，所以第一步先
> 
> 	go get github.com/russross/blackfriday
> 
> 这样读者就可以把相应的Markdown文件编译成html文件，执行`go build build.go`，执行生成的文件，就会在底目录下生成相应的html文件
> 
> ## 交流
> 欢迎大家加入QQ群：259316004 《Go Web编程》专用交流群
> 
> 大家有问题还可以上德问上一起交流学习：http://www.dewen.org/topic/165
> 
> ## 致谢
> 首先要感谢Golang-China的QQ群102319854，里面的每一个人都很热心，同时要特别感谢几个人
> 
>  - [四月份平民](https://plus.google.com/110445767383269817959) (review代码)
>  - [Hong Ruiqi](https://github.com/hongruiqi) (review代码)
>  - [BianJiang](https://github.com/border) (编写go开发工具Vim和Emacs的设置)
>  - [Oling Cat](https://github.com/OlingCat)(review代码)
>  - [Wenlei Wu](mailto:spadesacn@gmail.com)(提供一些图片展示)
>  - [polaris](https://github.com/polaris1119)(review书)
>  - [雨痕](https://github.com/qyuhen)(review第二章)
> 
> ## 授权许可
> 除特别声明外，本书中的内容使用[CC BY-SA 3.0 License](http://creativecommons.org/licenses/by-sa/3.0/)（创作共用 署名-相同方式共享3.0许可协议）授权，代码遵循[BSD 3-Clause License](<https://github.com/astaxie/build-web-application-with-golang/blob/master/LICENSE.md>)（3项条款的BSD许可协议）。
> 
> ## 开始阅读
> [开始阅读](<https://github.com/astaxie/build-web-application-with-golang/blob/master/preface.md>)
> 
> 
> [![githalytics.com alpha](https://cruel-carlota.pagodabox.com/44c98c9d398b8319b6e87edcd3e34144 "githalytics.com")](http://githalytics.com/astaxie/build-web-application-with-golang)
> 

### 3. 国内快速在线查看地址

由于国内访问github比较慢，于是我将 .md 文档生成html后放到了我的blog和sina sae 上，方便自己看（因为没有任何多余内容，很快，因此我常在手机上看）。现在分享给大家：

 + 在sina sae 上[http://maxia.sinaapp.com/static/goweb/index.html](http://maxia.sinaapp.com/static/goweb/index.html)（推荐）
 + [http://blog.5d13.cn/resources/goweb](http://blog.5d13.cn/resources/goweb)

声明：这仅仅是为了方便呀，作者如果反感请给我email。

