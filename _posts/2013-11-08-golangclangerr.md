
---
layout: post
categories: 
- work
- golang
title: golang编译出现clang错误
description: 两三周前开始golang编译包含cgo的包会报clang错误，困扰我几周，今天终于解决了，特此纪念。
keywords: golang,clang
---

###错误现象
几周前，突然发现我的go 项目编译开始报一种以前从来没有出现过的错误:

    # runtime/cgo
    clang: warning: argument unused during compilation: '-pthread'
    # runtime/cgo
    clang: error: no such file or directory: 'libgcc.a'

*需要说明下：我的开发机器是mac mini,系统当时是10.8.5, 上周升级为mavericks。*


###问题进一步探索
刚出现时我有点慌，当然上了google，查出一大堆结果，答案五花八门，一一试过都不能解决，这下我慌了——毕竟go语言是一门新语言，而apple又是google的“死敌”！
我进一步发现，我用 **go build -a** 就会报这个错误，否则不报这个错误。这就有些蛋疼了：
+ 我修改了某个包
+ 在引用了它的程序里会用它原来的包（在 ** $GOPATH/pkg/darwin_amd64/ ** 下)，以前用go build -a 就可以保证是最新的（加上 **-a** 参数会对引用的包重新编译），但现在会报错，于是我只能手动删除 ** $GOPATH/pkg/darwin_amd64/ ** （后来我写了一个脚本），然后再编译。

在写了清包脚本和编译脚本后，我就将这事暂时放下了。

###问题加剧，不解决不行了
今天早上我需要编译一个linux和Windows版本，于是就到 $GOROOT/src 下执行:

    $ sudo CGO_ENABLED=0 GOOS=linux GOARCH=amd64 ./make.bash
    $ sudo CGO_ENABLED=0 GOOS=windows GOARCH=amd64 ./make.bash

这两个命令都报错了，在编译到runtime/cgo 时！问题也是clang 出错。
更令我上火的是，我发现不论在什么情况下 ** go build ** 也报文章开头的错误了，这下我只能又一次google。结果与三周前一样一样的。

###问题回溯
我在举手无措时，回想几周前出现这个问题我的电脑装了些什么东西呢？我想到那天进行了Xcode 5 的升级。一想到这，我就几乎断定就是xcode5 引起的了。因为xcode是用clang 进行编译的。于是google ：** upgrade xcode 5 golang warning: argument unused during compilation: ' "-pthread" ** ，进入到 [Mac XCode 5 build problem](http://https://groups.google.com/forum/#!topic/golang-nuts/cTQaJGtZkYQ) 看了下，里面提到重新安装go 1.2rc3 就解决了。我依葫芦画瓢，下载golang1.2rc3 ，安装，测试，终于正常了！

###总结
写这篇blog有两个目的：
1. 我相信很多golanger 都会遇到这个问题，而golang 1.2rc3 才发布没有多久，或许有些朋友还不知道，写此文已做提醒。
2. 提醒自己对待 “爱机” 像写程序一样细心。

