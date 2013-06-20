---
layout: post
categories: 
- work
- read
- golang
title: 手机网络游戏应用协议设计（二）
description: 为了设计新游戏的网络应用协议，参考了golang 的gob，google protocol buf，以及之前项目的经验完成最终设计，本文做一备忘和与网友们交流之用。
keywords: golang,网络应用协议
---

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


这两篇博客详细的记录我对网络应用协议编解码的学习、思考及解决过程以备忘，更希望大大们可以指点交流。

