---
layout: post
categories: 
- work
- golang
- letsgo
title:golang标准包gob和json性能比较
description: 在游戏服务器的开发过程中肯定需要用到数据序列化，本文对golang标准库的gob和json两种编码库进行了性能测试出现意料之外的事情：json的性能和字节效率都比gob高，why？
keywords: gob,json,serialize,序列化,编码,解码,golang
---

###前言
我不是一个有足够时间进行单纯学习与研究的人，但是我们的项目需要用到的技术就不能不认真了，呵呵。

在游戏服务器的开发过程中肯定需要用到数据序列化，于是我对golang标准库的gob和json两种编码方式进行了性能测试。结果却跌破我的眼睛：json的性能和字节效率都比gob高，这否定了我之前的“常识”——几乎所有介绍gob的文章都说gob的编码效率比json高。于是我将测试代码及环境记录下来，希望有兴趣的朋友共同探讨。

###测试机配置
我的测试机（也是我的开发机器）是mac mini，具体配置如下：
> 2.3GHz Mac mini
> 2.3GHz 四核 Intel Core i7 处理器
> 4GB 内存
> 1TB 硬盘1
> Intel HD Graphics 4000 图形处理器
> OS X Mountain Lion


###测试代码

**serialize.go**
{% highlight go lineno%}
package helper


import (
    "bytes"
    "encoding/gob"
	"encoding/json"
)

type LGISerialize interface {
    Serialize(src interface{}) (dst []byte, err error)
    Deserialize(src []byte, dst interface{}) (err error)
}

type LGGobSerialize struct {
}

// serialize encodes a value using gob.
func (self LGGobSerialize) Serialize(src interface{}) (v []byte, err error) {
    buf := new(bytes.Buffer)
    enc := gob.NewEncoder(buf)
    err = enc.Encode(src)
    if err != nil {
        return
    }
    v = buf.Bytes()
    return
}

// deserialize decodes a value using gob.
func (self LGGobSerialize) Deserialize(src []byte, dst interface{}) (err error) {
    dec := gob.NewDecoder(bytes.NewBuffer(src))
    err = dec.Decode(dst)
    return
}

type LGJsonSerialize struct {
}

// serialize encodes a value using gob.
func (self LGJsonSerialize) Serialize(src interface{}) (v []byte, err error) {
    v,err = json.Marshal(src)
    if err != nil {
        return
    }
    return
}

// deserialize decodes a value using gob.
func (self LGJsonSerialize) Deserialize(src []byte, dst interface{}) (err error) {
    err = json.Unmarshal(src,&dst)
    return
}

{% endhighlight %}

**serialize_b_test.go**
{% highlight go lineno%}
package helper

import (
    "testing"
    "fmt"
)

func Benchmark_SerializeGob(t *testing.B) {
    //fmt.Println("////////////////////////test serialize b //////////////////////////////")
    for i := 0; i < t.N; i++ {
        test()
    }
    bytelength := test()
    fmt.Println("bytelengthGob=",bytelength)
}

func test() int {
    bl := 0
    gs := &LGGobSerialize{}

    v1 := 23422
    v2 := -1

    v,_:= gs.Serialize(v1)
    //bl += len(v)
    _ = gs.Deserialize(v,&v2)

    v1 = 0
    v2 = -2
    v,_ = gs.Serialize(v1)
    //bl += len(v)
    _ = gs.Deserialize(v,&v2)

    a1,_ := NewA1A2()

    a2 := &A{}
    v,_ = gs.Serialize(a1)
    bl += len(v)
    _ = gs.Deserialize(v,&a2)

    return bl
}

func Benchmark_Serialize_Json(t *testing.B) {
    //fmt.Println("////////////////////////test serialize b //////////////////////////////")
    for i := 0; i < t.N; i++ {
        test2()
    }

    bytelength := test2()
    fmt.Println("bytelengthJson=",bytelength)
}

func test2() int {
    bl := 0
    gs := &LGJsonSerialize{}

    v1 := 23422
    v2 := -1

    v,_:= gs.Serialize(v1)
    //bl += len(v)
    _ = gs.Deserialize(v,&v2)

    v1 = 0
    v2 = -2
    v,_ = gs.Serialize(v1)
    //bl += len(v)
    _ = gs.Deserialize(v,&v2)

    a1,_ := NewA1A2()

    a2 := &A{}
    v,_ = gs.Serialize(a1)
    bl += len(v)
    _ = gs.Deserialize(v,&a2)

    return bl
}

{% endhighlight %}

###测试结果
>$ go test -bench=".*"
>Benchmark_SerializeGob	bytelengthGob= 507
>10000	    169793 ns/op
>Benchmark_Serialize_Json	bytelengthJson= 470
>20000	     89994 ns/op
>ok  	github.com/sunminghong/letsgo/helper	4.446s

###结论&问题
1. 显而易见，json要快一倍，且编码字节数略省（8%左右）
2. 经反复测试，gob方式可对任何数据类型编码、解码；json 对map类型编码，**key值必须是字符串类型**

数据说明一切，但是我还是希望有兴趣的朋友一起研究下。


