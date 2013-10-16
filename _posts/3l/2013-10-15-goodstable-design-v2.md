---
layout: post
permalink: /3L/goodstable-design-v2.html
categories:
- work 
- 3l
title: 创新手机游戏《3L》开发点滴(3)——道具、物品、装备表设计 V2(最终版)
description: 在总结了以前游戏开发的经验和参考了网上的一些文章后形成了我们新手机游戏的物品数据表（或道具表、装备表）设计方案。
keywords: 手机游戏,道具,物品,装备,数据结构,table,字段,设计
---


我们正在开发一款新手游，里面有道具，之前也写了[一篇博文][link1]记录了下我们的设计思路，但是国庆到了，于是我有了时间继续纠结这个问题。

其实我主要是在到底是用mysql还是mongodb上纠结。这个复杂、痛苦、找虐的挣扎过程就不说了，最终我选择了mysql（可能我还是比较保守吧， ^-^）。

现在将我们最终方案记录下来备忘。

###一、道具buff存储、使用
1. 所有buff都以如下类似键值对的方式拼接成字符串后存于mysql数据库：

	`atk+30;def-20;hp+30`
2. 某条原始道具数据会在redis里缓存（具体见下文）
3. **属性分拆缓存**：将buff数据解析成为map后在redis里以hashmap 形式再存一份，键名为itemid-buff
4. 最终buff消费者都是从redis里读取：
	* 后台需要buff进行计算的，如战斗、装备升级等，直接读取 **属性分拆缓存**  
	* 不需要对具体属性值进行处理的，如直接发送到前端的，如存入数据库，读取原始属性（buff字符串）  
5. 在运营中可能需要对属性进行一些分析、查询，而现有的方案buff都是以字符串形式存储，用 like '%***%' 查询，从性能上显然是不允许的。于是我们想到了一个“空间换时间”的方法——对常用的buff值进行单独建表索引，如：
	
	`atk+30;def-20;hp+30`

	create table index-goods-buff-atk (good-id bigint,val int , primary key (good-id,val), index (val))

	create table index-goods-buff-def (good-id bigint,val int , primary key (good-id,val), index (val))

	create table index-goods-buff-hp (good-id bigint,val int , primary key (good-id,val), index (val))

*在增加、修改相关属性时，增加、修改属性索引表数据

###二、道具数据处理流程
**所有数据访问都封装在model层。**

1. 在程序的model层里定义公共函数GetDb(playerid,areaid)，用于物品数据及其它需要进行按用户水平分割的数据获取数据库（连接）  
2. 每一个物品的主键id 由 **[函数根据自定义规则][link2]**(这个修改了v1.0里的方式) 生成，确保表内唯一  
3. 将装备buff及某道具特有属性统一以键值对字符串形式保存在一个varchar字段里  
4. 道具表定义 *失效时间* 字段，这个字段同时表示道具有效期的三种情况：  
    * =0 ，表示是不限时的永久性道具  
    * 小于等于600000，表示有效时间段，如双倍经验卡有效时长3小时，那么着个字段值为 **180** （分钟），使用此道具时，用当前时间加上这个字段值得到失效时间戳修改此字段或做其他处理  
    * 大于600000，表示该道具的失效时间戳（unix时间戳格式）  
5. 玩家道具数据查询流程：  
    * 首先检查redis里有没有  
    * 如果有，返回  
    * 如果没有从数据库里查询  
	* 调用 **属性分拆缓存** ， 并存到redis  
	* 返回

6. 修改/或增加道具数据流程：  
    * 首先检查redis里有没有  
    * 如果没有，从数据库里查询并存到redis  
    * 如果有，继续  
    * 修改/增加redis数据，如果buff字段有修改，则调用 **属性分拆缓存** 处理
	* 同时在key为“item-update-list”的redis list 数据里记录下该道具的主键  
    * 定时根据“item-update-List”数据写回mysql，并清除“item-update-list”

7. 删除一个道具数据流程：  
    * 在redis 里key为“item-delete-list” 的redis list 数据里记录下该道具的主键  
    * 删除redis里记录  
	* 删除 **属性分拆缓存**   
    * 定时根据item-delete-list”数据删除mysql里记录，并清除“item-delete-list”  

8. 注意：  
    * 配置redis 为 **在每次更新操作后进行日志记录**  
    * 缓存定时同步mysql处理程序可以作为一个独立的进程运行
    * 这个解决方案可以针对其它用户私有数据，如技能、buff。




------
//  
//每次开发一款新的游戏都是一个新的开始、新的征程、新的挑战，我们一直在路上！  
//[创新手机游戏《3L》开发点滴][link3l]  
//  

[link3l]: http://blog.5d13.cn/3l.html"创新手游3L开发点滴"
[link1]: http://blog.5d13.cn/3l/goodstable-design.html "RPG手机游戏道具、物品、装备表设计"
[link2]: http://blog.5d13.cn/3l/table-primary-key-design.html "手机游戏数据表主键设计"

