---
layout: post
permalink: /3L/table-primary-key-design.html
categories:
- work 
- 3L
title: 创新手机游戏《3L》开发点滴(2)——数据表主键设计 
description: 记录下我们新款手机游戏的数据表主键设计过程及最终方案。 
keywords: 手机游戏,数据结构,table,自增长,主键
---

###一、主键方案选择
几经考虑，其间也参考了一些博客文章，获益匪浅。同行们采用了很多种方案，各有优缺点，最终我们确定采用自定义规则来程序生成主键值——唯一理由就是灵活：
+ 可以将区服编号融入到id里，在程序编码、运营查询时就可以只需要一个id就锁定数据记录
+ 跨区玩法、合服逻辑更简单


###二、主键计算规则
	id（19位） = areaid(3位) + timestamp(8位)+ idCheckcode(1位）+ sequence(7位）
	注解：
	areaid，区服id，每个区服有一个唯一编号，999个区服应该够了吧， ^_^
	timestamp，unixt时间戳 - 2013-10-01 号的时间戳 ，共8位数
	idCheckcode，区分不同的进程，我们每一个区都会有一组服务器，每个服务器编号即serverid，用serverid 取 10 的模，确保id唯一
	sequence，在一秒内不可能需要生成9999999个数字吧？这里可以用随机数，但是为了让id更加友好，和自然增加（且插入数据库进行索引时也会快一些），我们推荐用全局顺序号实现。
	例如生成的id有：133857420000001,133882920000001

	*areaid,timestamp,idCheckcode,sequence四个因子的数据位长度是可以根据实际情况加以调整的，比如sequence可以用5位，idCheckcode用3位。

go lang 程序代码如下：
	 
{% highlight go linenos %}
package fun

import (
    "sync"
    "time"
    //"reflect"
)

var lock *sync.RWMutex = new(sync.RWMutex)
var idcounts = map[string]int {}

//2013-10-01 00:00:00
const IdBaseTimestamp int = 1380556800
var _idCheckcode int = 0

func SetServerId(sid int) {
    _idCheckcode = sid % 10
}

//return newid by rule:
//19(64) = areaid(3)+unixtimestamp - basetimestamp(8) + idCheckcode(1) + idcount(7)
func NewId(areaid int,tablename string) int {
    var num int
    lock.Lock()
    if n,ok := idcounts[tablename]; ok {
        //num = n & 0xffff
        //num = n & 0x4ffffff
        num = (n+1) & 0x7fffff //< 838 8607

        idcounts[tablename] = num
        lock.Unlock()
    } else {
        idcounts[tablename] = 2
        lock.Unlock()

        num = 1
    }

    tt := int(time.Now().Unix()) - IdBaseTimestamp
    return areaid * 10000000000000000 + tt * 100000000 + _idCheckcode * 10000000 + num
}
{% endhighlight %}


**python 代码如下：**

{% highlight python linenos %}
_idcounts = {}

#2013-10-01 00:00:00
IdBaseTimestamp = 1380556800
_idCheckcode = 0

#return newid of all tables primary key (but not playerid and log tmp table) 
#by rule:
#19(64) = areaid(3)+unixtimestamp - basetimestamp(8) + idCheckcode(1) + idcount(7)
def NewId(areaid,tablename):
    num = 0
    if tablename in _idcounts:
        n = _idcounts[tablename]
        num = (n+1) & 0x7fffff

        _idcounts[tablename] = num
    else:
        _idcounts[tablename] = 2

        num = 1

    tt = int(time.time()) - IdBaseTimestamp
    
    return areaid * 10000000000000000 + tt * 100000000 + _idCheckcode * 10000000 + num
{% endhighlight %}

###三、完善
根据以往的运营游戏的经验，玩家id是个最频繁的关键数据，按以上规则计算出的id较长、不易记忆和传播，于是我将playerid区别对待，它的规则如下：
+ 建立login表，专门用于处理登陆验证、玩家身份确认，同时它又id（增长）和areaid两个字段。
+ 玩家游戏数据都记录在player表，player表的id 这由loginid和areaid算出来，规则如下：
	playerid = areaid * 10000000 + loginid

+ 其它表全部用以上的规则计算生成
+ 以后合区时，只需要修改login表的自增长计数器为该表合区并后的最大值即可。

2. 接下来我们考虑到工作各方面的实际情况，我们对《3l》里的玩家id主键做一种特殊处理：


###四、吐槽
此文写了两遍，第一遍时不知道进行了什么错误操作我的文档内容突然成了另一篇文章的内容，恢复了半小时没有搞定，又写第二遍，悲剧呀！第二遍没有第一遍写的详细。



参考资源：
<a title="何雨泉的如何在高并发分布式系统中生成全局唯一Id" href="http://www.cnblogs.com/heyuquan/p/3261250.html" target="_blank">如何在高并发分布式系统中生成全局唯一Id</a>


------
//  
//每次开发一款新的游戏都是一个新的开始、新的征程、新的挑战，我们一直在路上！  
//[创新手机游戏《3L》开发点滴][link3l]  
//  

[link3l]: http://blog.5d13.cn/3L.html "创新手游《3L》开发点滴"

