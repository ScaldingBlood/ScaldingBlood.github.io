---
title: nio
tags:
---

* buffer
* channel
* selector

#### buffer
capacity position limit
limit 读时为可读到的量 写时为capacity

方法：
allocate put flip get rewind(把position重置为0，可以重复读数据) 
clear(不删除值，转为写模式) compact(移动未读数据到buffer开头，转为写模式) 
mark reset(设置一个标记，可以重置position到标记)
equals(相同的类型，剩余元素个数和元素都相同) compareTo（先比值在比个数）

#### selector
必须与非阻塞channel(channel.configureBlocking(false))配合使用（FileChannel不能使用）
`SelectionKey key = channel.register(selector, SelectionKey.OP_READ)`
register第二个参数为感兴趣的操作；
OP_CONNECT OP_ACCEPT OP_READ OP_WRITE
如果不止对一个操作感兴趣使用位操作符`|`

`SelectionKey` 包含 interest集合 ready集合 Channel Selector 附加对象

```
int interestOps = key.interestOps();
int readOps = key.readyOps();
```
其中可以用isAcceptable()等方法判断ready的操作
channel() selector() attach(obj) attachment()

selector的操作
select() select(timeout) selectNow()(非阻塞) 返回的是自上次调用select后准备就绪通道的数量 wakeup()在其他线程上调用使阻塞的select立刻返回。
selectedKeys() 当select方法返回大于0的值后，调用该方法可以返回已经就绪的管道注册的键的集合
close() 使所有注册的selectionKey无效

IO/NIO
面向流/面向缓冲
阻塞/非阻塞
选择器


