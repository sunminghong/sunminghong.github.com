---
layout: post
categories: 
- work
- python
title: 一个用python写类unix进程id分配器
description: 可用于一些需要在有限的号码区间内分配唯一ID的应用，如游戏里的房间号、在线客户ID等
keywords: pid,python,房间号
---

[firePhoenix][firePhoenix]项目里需要给连接的客户端分配一个唯一的客户ID，类似socketid、系统进程ID（pid），于是模拟linux PID的分配算法实现一个ID分配器。

{% highlight python linenos %}
class IDAllocor(object):
    '''分配一个唯一的ID，如clientID，pid'''

    #上次分配的ID
    _lastID = -1

    #map页，用于指示某个ID是否分配
    _page = ""

    #最大可分配的ID
    _maxID = 32767

    _free = 0

    #是否第一遍扫描
    _first = 1

    def __init__(self,maxid=32767,page=False):
        self._maxID = self._free = maxid
        self._lastID = -1
        if page == False:
            self._page = [0] * ((self._maxID+1) / 32)
        else:
            self._page = page
        self._first = 1

    def _setBit(self,offset,value):
        '''设置offset位值，0或1'''
        bit_off = offset & 0x1f
        int_off = offset >> 5

        if value:
            self._page[int_off] = self._page[int_off] | (1 << bit_off)
        else:
            self._page[int_off] = self._page[int_off] & (~(1 << bit_off))

    def test(self,offset):
        '''取offset的位的状态'''
        bit_off = offset & 0x1f
        int_off = offset >> 5

        return self._page[int_off] & (1 << bit_off)

    def _findFree(self,offset):
        '''扫描map，返回一个为0的位'''
        size = self._maxID
        page = self._page
        while (offset < size):
            bit_off = offset & 0x1f
            int_off = offset >> 5

            if page[int_off] & (1 << bit_off):
                offset += 1
                continue

            return offset

        return -1

    def alloc(self):
        '''分配一个ID，如果没有可分配的ID 了，就返回-1'''
        if not self._free:
            return -1

        self._lastID += 1
        self._free -= 1

        #if self._first:
        #    #如果还没有将整个区间分配过一遍，就先顺序分配
        #    if self._lastID <= self._maxID:
        #        self._setBit(self._lastID,1)
        #        return self._lastID

        #    self._first = 0
        #    self._lastID = 0
        
        offset = self._lastID & self._maxID

        offset = self._findFree(offset)
        if offset == -1:
            offset = self._findFree(0)
            if offset == -1:
                return -1
            
        self._setBit(offset,1)
        self._lastID = offset

        return offset

    def free(self,offset):
        '''回收ID'''
        self._setBit(offset,0)
        self._free += 1

if __name__ == '__main__':
    import random,codecs

    alloc = IDAllocor(32767)
    
    f = codecs.open('d:\\testallocid.csv' , "a", "utf-8")
    co = 0
    for i in xrange(0,100000):
        id0 = alloc.alloc()
        if id0 == -1:
            break

        print id0
        f.write('insert into ids (id) value (%s);' % (id0))
        f.write('\n')

        co += 1
        if random.randint(0,100) <50:
            alloc.free(id0)
            f.write('delete from ids where id=%s;' % (id0))

    f.close()
    print alloc._free
    print 'co = ',co
{% endhighlight %}


[firePhoenix]: https://github.com/sunminghong/firePhoenix

