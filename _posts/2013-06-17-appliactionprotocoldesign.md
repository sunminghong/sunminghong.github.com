---
layout: post
categories: 
- work
- read
- golang
title: 手机网络游戏应用协议设计
description: 为了设计新游戏的网络应用协议，参考了golang 的gob，google protocol buf，以及之前项目的经验完成最终设计，本文做一备忘和与网友们交流之用。
keywords: golang,网络应用协议
---


应用协议V1.0设计方案
===========
在《卧龙转》的开发中，我设计了一种最简洁、最高效的网络协议结构，我就称它为V1.0:
> 一个数据包 = 消息尺寸(int32) + 消息类型码(short) + 消息正文
> 消息正文 = 数据项1 + 数据项2 +。。。
> 客户端用一个md文档进行约束，格式如下：
>
> 
> **1050 客户端请求同步时间**
> 
> 枚举： `GET_TIME_C`
> 数据：
>     + 发送协议时客户端的时间戳（t1)
> 
> **2050 服务端返回时间**
> 
> 枚举： `GET_TIME_S`
> 数据：
> 	+ 发送协议时客户端的时间戳（t1)
> 	+ 服务器端就收到协议时服务器端的时间戳（t2)
> 	+ 回发该协议时服务器端的时间戳（t3)
> 	+ 心跳频率（秒数）
>

在程序里这样写：
{% highlight python lineno%}
    t2 = int(time.time())
    wr=RW.RWStream(body)
    t1 = wr.readUInt()

    wr.init()
    wr.writeUInt([t1,t2])
    wr.writeUInt(timer.getNetTimestamp(int(time.time())))
    wr.writeUInt(com._cc.CLEAR_INTERVAL_TIME)

    body=wr.getBytes()
    connection.sendmsg(2010,body)
{% endhighlight %}

\*备注：
+ 其中数据项都采用了varint 编码，极大的节省的传输数据流。
+ 由于应用简单得“简陋”，也使得它的数据编码效率超高，几乎没有多余字节
+ 对较长的字符串数据，我们进行gzip压缩。
+ 以上几点是《卧龙传》在2011、2012年的国内移动网络较低带宽水平下流畅运行的重要前提。
+ 不需要依赖外部的协议书写工具（如google protocol buffer 需要依赖编码工具和数据结构定义文档），很适合小型开发团队高效设计、高效开发（只需维护一份如上的协议定义文本文档）。

###V1.0的问题
显而易见，V1.0版效率虽然高，但是很简陋，一些先天不足在前端、后端工程师编写协议解析代码时显现出来：

1. 必须严格遵循协议定义顺序读写数据项
2. 必须注意数据项的数据类型，如果将unsigned varint 与unsigned integer 32 混淆，导致数据项读写错误，且只能在运行过程中才能单步跟踪才能定位，而编码错误是不可避免
3. 几乎不能支持客户端、与服务器端协议版本不一致的情况，而实际运营过程中，常常需要允许多个版本的客户端同时存在，如微信新老版本都可以正常运行。
4. 协议文档不能多种形式查看，它是一个markdown格式纯文本文件，呵呵。

###思考
1. 数据总是按一定顺序编码的，所以在数据编解码层级是必须顺序读写数据项，但是我们可以开发辅助工具：
    + 自动读写（如golang语言的gob包）
    + 或者自动生成协议读写代码，如google protocol buffer等

2. 数据项类型读写错误问题：  
    + 给每个数据项编码规则上加上一个类型码，是数据有一定的自描述性
    + 读写函数可以判断类型是否正确，如readUint()读出的是string时就可以抛出异常进行处理，方便类型读写错误定位。
    + 通过识别、写入类型码，可以设计编写通用的读写函数（关于通用读函数可能只能在动态语言里可以实现了）。

3. 协议的多版本兼容：
    + 首先我们要记住旧的客户端并不知道新版本协议的定义
    + 我们可以在每条协议的请求和返回数据项里加上版本号，程序根据版本号分别处理，如上面的1050号协议改为：
    > 
    > **1204 联盟成员列表 V1.0**
    >   + 版本号
    > 	+ 联盟id
    > 	+ 成员类型（=0 为待审核成员；=1 为正式成员）
    > 	+ 页码(0,开始)
    > 	+ 每页多少个（第一次请求为0，后续请求以2104返回的尺寸为准）
    > 
    > **2204 联盟成员列表 V1.0**
    >   + 版本号
    > 	+ 压缩方式(=0 没有压缩；=1 gzip压缩)
    > 	+ 成员类型（=0 为待审核成员；=1 为正式成员）
    > 	+ 联盟ID
    > 	+ 成员数量(==0时后面无）
    > 	+ [(
    > 		+ uid（uint32）
    > 		+ vip等级
    > 		+ 名称(string)
    > 		+ 爵位等级
    > 		+ 武将数
    > 		+ 领地数
    > 
    > 		+ 加入时间
    > 		+ 职位(=0 为待审核，=1为一般成员，=2为副盟主，=50为盟主)
    > 
    > 		+ 玩家头像文件(string)
    > 	+ )...]
    > 	+ 每页多少个
    > 	+ 第几页

    + 上面的1204号协议的定义里有一个列表数据（就像struct list），如果列表里增加一个元素，那么读取方肯定会出错，这也是V1.0不能支持多版本的最重要的原因。
    + 我们在版本迭代的过程中，多数情况是增加新的数据项，比如在2204协议中给每个联盟成员增加数据项【玩家贡献值】，旧客户端要能兼容，
    + 我们可以在协议数据项前增加一个元数据快，定义数据项编号和类型；列表项前增加列表元数据块，这样读取程序就可以知道数据结构，为程序的“智能”处理提供编码基础。


4. 协议文档的编写格式：
    + 为了方便不同文件版本的比较（我们会用git管理设计文档），所以要求必须要用文本文件定义
    + 可以用xml定义
    + 可以自定义特定的格式来书写协议

###现有方案的分析
1. google protocol buffer（protobuf） 是现在应用的非常广泛的一种解决方案，基本上可以解决以上所有问题，但是我们开发团队成员讨论分析有如下顾虑：
    + 修改一个协议，调用不同语言版本的工具重新生成读写文件，而开发初期阶段可能更改很频繁，比较麻烦；而一般修改协议，意味着逻辑也会修改，自动生成数据读写代码并没有省却多少代码量而仅仅只是起到了规避数据读写代码编码错误，这是可以通过其他方式解决的。
    + protobuf 数据定义功能很强大，但是我们游戏数据似乎只需要两种
    + protobuf 有默认值的功能，这是个很好的设计，在某种情况下可以减少数据传输，但是我们的游戏数据这种情况很少
    + 很多协议之间只是简单传递几个数据，如果都要定义数据文件（.proto）文件再生成语言代码，似乎有点“大财小用”；而如果这时不用protobuf，又会导致方法不统一的不良后果
    + protobuf 给每个数据项前增加一个字节的元数据，而列表数据明显是重复冗余了
    + 我细看了一些语言的代码，由于要实现protobuf的强大的特性，一些语言生成的代码非常复杂，代码量巨大，效率低下

    终上所述，protobuf 用于我们的游戏数据传输，无效数据率较大，且我们团队认为用起来较繁复。

2. golang 的gob包的设计可以解决以上问题，它也是google的大牛们设计的，规避了protobuf一些问题：
    + 数据默认值（0,''）不传送，但不支持自定义默认值
    + 不考虑必选项/可选项，所有都采用属性名：属性值的方式传输，一种方法解决了这个问题
    + 不考虑多语言通用（原设计只考虑golang之间通讯），这样设计可以很轻巧。
    + 充分考虑了golang得特性（如高效的反射、struct）等，gob代码很简短
    + 自描述，且只在两端第一次传输时传送元数据（这个我还没有搞懂具体实现原理）
    + 所有整数型数据它都已varint 方式编码。

    终上所述，我们不能用它了（我们是多语言体系），但我们可以吸收它的一些优点。

###V2.0数据编码基本思路
    + 包含自描述元数据：序号、类型
    + 将整数型数据归结为两种：varint和unsigned varint。
    + 0,'' 不传输
    + 版本号成为协议必要数据项


应用协议V2.0设计方案
===========
经过以上的分析，最后形成如下新的设计方案，就称为V2.0吧，它已用于我们的新游戏中。


###数据类型定义

\* 编码采用BigEndian

* uvint		unsigned varint ,默认为此类型
* vint		varint
* string	(长度（uvint）+字符串)

* int32			(4 bytes)
* uint32		(4 bytes)
* short/int16	·		(2 bytes)
* ushort/uint16			(2 bytes)
* byte			(1 byte)


###数据包（datagram）基本结构定义

    flag1(byte) + flag2(byte) + 数据类型(datatype,byte) + 数据尺寸(dataSize,int32) + 数据(bytes)
    flag1 = 0x89,  
    flag2 = 0x1a,  
    

###客户端与服务器message传输包（message）结构定义

    数据类型 = 0 
    数据 = message = message类型码(messageCode,short) + 版本号（version,byte） + message正文(body,bytes)

    即：
    flag1(byte) + flag2(byte) + 数据类型(datatype,byte=0) + 数据尺寸(dataSize,int32) + message类型码(messageCode,short) + 版本号（version,byte） + message正文(body,bytes)
    **message正文的数据项的第一个一定是执行成功标志(成功为0，服务器端出现错误为1）**

###广播数据包(broadcast)结构定义

    数据类型 = 3
    数据 = （同 message）

    即：
    flag1(byte) + flag2(byte) + 数据类型(datatype,byte=1) + 数据尺寸(dataSize,int32) + 数据(message) + 玩家socketID(clientid,int32)

    

###数据中转传输包(delay)结构定义

    数据类型 = 1 
    数据 =原始数据(message) +  玩家socketID(clientid,int32) 

    即：
    flag1(byte) + flag2(byte) + 数据类型(datatype,byte=1) + 数据尺寸(dataSize,int32) + 原始数据(message) + 玩家socketID(clientid,int32)


###messagebody结构定义(数据项编码设计)
    message正文是程序间相互传递的数据主体，有多个数据项组成，每个数据项都有元数据描述数据，这样可以达到以下目的：
    1. 更好的解决前后端协议版本不一致时数据传输的兼容性
    2. 简化数据读取时的代码

    + **元数据** = 数据项数量（byte）+ 数据项描述1(byte) + ... + 数据项描述n(byte)  
    + messageBody = 元数据 + 数据项1 +... + 数据项n
    + 数据项= 数据项描述(byte) + 数据
    + 数据项描述 = 数据编号（7~3bit） + 数据类型(2~0bit)
    + 数据类型:
        + uvint = 0
        + string = 1
        + vint = 2
        + list = 3

    + list 数据类型用于表示数据序列（list)，如玩家列表、武将列表，数据结构定义
        + list = 数据字节长度(uint16) + 列表长度(byte) + 元数据 + 列表数据1 + ... + 列表数据2
        + 列表数据 = 数据项1 +... + 数据项n


###协议定义规范

    1. 客户端请求的message类型码以1开头（如1xxx)
    2. 服务端向客户端发起请求的message类型码以2开头（如2xxx）；
    3. 每条协议返回数据第一数据项都是【成功否】，且0表示成功，1为服务器出错，2为客户端退出
    4. 服务器端推送的错误message代码以9开头；
    5. 所有涉及到等级的地方都是0基，除非特别说明。

\* 后面的“协议”部分，只包含“messageBody数据项”的定义，其他都省略。

### 协议文档定义举例
> 
> ###**1000** 客户端请求同步时间
> ver:0
> data:
> + 0,,t1,发送协议时客户端的时间戳(t1)
> \* 这是一个备注说明
> 
> ###**2000** 客户端请求同步时间
> ver:0
> data:
> + 0,,returnFlag,成功否（=0 成功 =1 服务器端出错)
> + 1,,t1,发送协议时客户端的时间戳（t1)
> + 2,,t2,服务器端就收到协议时服务器端的时间戳（t2)
> + 3,,t3,回发该协议时服务器端的时间戳（t3)
> + 4,,seconds,心跳频率（秒数）
> + 5,list,generaList,武将列表
>     + 0,,name,武将名称
>     + 1,,id,武将ID
> + 6,,abc,市委
> + 7,,abcd,是市委第三方
> 
\*协议文档格式备注：
+ 采用了markdown语法，方便阅读（可用markdown阅读器阅读）
+ 定义了固定的格式，既方便编写，也方便处理，如我写了一个python脚本将它转成xml文档。：
    > ###**协议编号** 协议标题
    > ver:版本号
    > data:
    > + 数据项序号,数据想类型（省略则为默认的uvint类型）,变量名,数据项意义说明
    >
+ 列表数据项必须缩进(参见上面的例子),可以嵌套


数据读写
=======
我还是贴出golang 的一段测试代码，以作说明：
{% highlight go lineno%}
func LGTest_MessageWrite(t *testing.T) {
    msgw := NewMessageWriter(BigEndian)

    a1 := 989887834
    a2 := 243
    a3 := 3298374
    a4 := -432423423
    a5 := uint32(23)
    a6 := uint16(32234)
    a7 := "aasalfjnsaknhfaksdfashdr8o324rskjdfh8oq734tjkdfq9ytfhasdbhuewrq364tqfgeawgiruhsb njafeuaaa"

    b1 :=uint(32342334)
    b2 :=uint(42323499)
    b3 :="bsdbbbb"

    //编码/写数据
    b4 := NewMessageListWriter(BigEndian) 
    for i:=0;i<5;i++ {
        b4.WriteStartTag()
        
        b4.WriteUint(i,0)
        b4.WriteUint(i+1,0)
        b4.WriteUint(i+2,0)
        b4.WriteString(string(i+3),0)

        b4.WriteEndTag()
    }

    //msgw.Write(a1)
    msgw.Write(a1,a2,a3,a4,a5,a6,a7)
    msgw.WriteUint(int(b1),9)
    msgw.WriteU(b2)
    msgw.WriteU(b3)
    msgw.WriteList(b4,0)
    //fmt.Println("messageWrite",msgw.ToBytes(1,1))

    msgw.SetCode(1,1)
    data := msgw.ToBytes()

    //解码/读数据
    msg := NewMessageReader(data,BigEndian)

    v1 := msg.ReadInt() 
    if v1!= a1 {
        t.Error("item a1 ReadInt is wrong:",v1,a1)
    }

    v2 := msg.ReadInt() 
    if v2!= a2 {
        t.Error("item a2 ReadInt is wrong:",v2,a2)
    }

    v3 := msg.ReadInt() 
    if v3!= a3 {
        t.Error("item a3 ReadInt is wrong:",v3,a3)
    }

    v4 := msg.ReadInt() 
    if v4!= a4 {
        t.Error("item a4 ReadInt is wrong:",v4,a4)
    }

    v5 := msg.ReadUint32() 
    if v5!= int(a5) {
        t.Error("item a5 ReadInt is wrong:",v5,a5)
    }

    v6 := msg.ReadUint16() 
    if v6!= int(a6) {
        t.Error("item a6 ReadInt is wrong:",v6,a6)
    }

    v7 := msg.ReadString() 
    if v7!= a7 {
        t.Error("item a7 ReadInt is wrong:",v7,a7)
    }
    _ = msg.ReadUint() 
    _ = msg.ReadUint() 

    vv1 := msg.ReadUint() 
    if vv1!= int(b1) {
        t.Error("item a1 ReadInt is wrong:",vv1,b1)
    }

    vv2 := msg.ReadUint() 
    if vv2!= int(b2) {
        t.Error("item a1 ReadInt is wrong:",vv2,b2)
    }

    vv3 := msg.ReadString() 
    if vv3!= b3 {
        t.Error("item a1 ReadInt is wrong:",vv3,b3)
    }

    fmt.Println("------------------------------------------------------")
    vv4 := msg.ReadList() 
    
    if vv4.Length != 5 {
        t.Error("item list Readlist length is wrong:",vv4.Length,5)
    }
    
    for i:=0;i<5;i++ {
        vv4.ReadStartTag()
        x := vv4.ReadUint()
        if x!= i {
            t.Error("item list(",i,",1) is wrong:",x,i)
        }
        x = vv4.ReadUint()
        if x!= (i+1) {
            t.Error("item list(",i,",2) is wrong:",x,i+1)
        }
        x = vv4.ReadUint()
        if x!= (i+2) {
            t.Error("item list(",i,",3) is wrong:",x,i+2)
        }
        x1 := vv4.ReadString()
        if x1!= string(i+3) {
            t.Error("item list(",i,",4) is wrong:",x1,i+3)
        }
        vv4.ReadEndTag()
    }


}
{% endhighlight %}


这是我写的最长的一篇博客了，详细的记录我对网络应用协议编解码的学习、思考及解决方法，以备忘，更希望大大们可以指点交流。
