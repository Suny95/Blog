---
title: Zookeeper笔记 (二) Znode详解
date: 2019-04-30 14:33:37
tags:
- zookeeper
- znode
categories:
- zookeeper
cover:
---
本文参考地址: [【Zookeeper 学习笔记】—Znode剖析](http://cmsblogs.com/?p=4103)

- ### 简介

zookeeper中，节点也叫znode。我们对zk的操作主要也是对znode的操作。

zk才用了类似文件系统的数据模型，其节点构成了一个具有层级关系的树状结构。

![image](https://gitee.com/chenssy/blog-home/raw/master/image/series-images/zookeeper/zookeeper-3001.png)

图中，根节点 / 包含了两个子节点 /module1和 /module2，而节点 /module又包含了三个子节点 /module1/app1, /module1/app2, /module1/app3。zk中节点以绝对路径表示，不存在相对路径，且路径最好不能以 / 结尾(根节点除外)。

- ### 类型

根据节点存活时间，节点可分为持久节点和临时节点。节点类型在创建时就确定下来，无法改变。

持久节点的存活时间不依赖客户端会话，只有客户端显示执行删除操作时，节点才消失。

临时节点的存活时间依赖客户端会话，当会话结束，临时节点自动删除(也可以手动删除)。

利用临时节点的这种特性，可以使用临时节点进行集群管理，包括发现服务的上下线等。

zookeeper规定，临时节点不能拥有子节点。

> 持久节点

```java
//创建持久节点/module1, 其数据为"module1"
create /module1 module1
```

> 临时节点

```java
//创建临时节点/module2, 其数据为"module2"
create -e /module2 module2
```

> 顺序节点

每次创建顺序节点时，zk都会在路径后面自动添加上10位数字(计数器)。

例如：<path>0000000001,<path>0000000002，….这个计数器可以保证在同一个父节点下是唯一的。在zk内部使用了4个字节的有符号整形来表示这个计数器，也就是说当计数器的大小超过2147483647时，将会发生溢出。

顺序节点是节点的一种特性，持久节点和临时节点都可以设置为顺序节点。

这样一来，znode就一共有4种类型：持久节点、持久顺序节点、临时节点、临时顺序节点。

```java
//创建持久顺序节点 /module3/app0000000001
create -s /module3/app app
//若在执行此命令,则会生产节点 /module3/app0000000002
//若在create -s后再添加 -e 参数，则创建临时顺序节点
```

- ### 节点的数据

在创建节点时，可以指定节点中存储的数据（byte[]类型）。zk保证读和写都是原子操作，且每次读写操作都是对数据的完整读取和写入，并不提供对数据进行部分读取和写入操作。

zk规定节点的数据大小不能超过1M，因为数据过大会导致zk的性能明显下降。

```java
//创建持久节点 /module4 其存储的数据为"module4"
create /module4 module4
```

- ### 节点的属性

每个znode都包含一系列的属性，通过get命令，可以得到节点的属性。

```java
get /module5
module5 //数据
cZxid = 0x1c
ctime = Mon Apr 29 22:50:22 PDT 2019
mZxid = 0x1c
mtime = Mon Apr 29 22:50:22 PDT 2019
pZxid = 0x1c
cversion = 0 
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 7
numChildren = 0
```

> 版本号

> - dataVersion：数据版本号。每次对节点进行set操作，dataVersion就会自增1
>
> - cversion：子节点版本号。当znode的子节点有变化时，cversion就会自增1。
>
> - aclVersion：ACL版本号。每次对节点进行setAcl操作，aclVersion就会自增1。

每个znode的数据版本号会随着数据变化而自增。zk提供的一些API例如setData和delete就是根据版本号有条件的进行。多个客户端对同一个znode操作时，版本号的作用就会体现出来。

例如：客户端C1对znode/config写入一些配置信息。如果另一个客户端C2同事更新了这个znode，此时C1的版本号已经过期，C1调用setData一定不会成功。这正是版本机制有效避免了数据更新时出现的先后顺序问题。

![image](https://gitee.com/chenssy/blog-home/raw/master/image/series-images/zookeeper/zookeeper-3002.png)

> 事务ID

zk每次变化都会产生一个唯一的事务id，zxid(zookeeper Transaction Id)。通过zxid，可以确定更新操作的先后顺序。例如：如果zxid1小于zxid2，说明zxid1操作先于zxid2发生。

zxid对于整个zk都是唯一的。即使操作的是不同的znode。

> - cZxid：znode创建的事务id
> - mZxid：znode被修改的事务id，每次对znode的修改都会更新mZxid

![image](https://gitee.com/chenssy/blog-home/raw/master/image/series-images/zookeeper/zookeeper-3003.png)

在集群模式下，客户端有多个服务器可以连接，当尝试连接到一个不同的服务器时，这个服务器的状态要与最后连接的服务器状态要保持一致。zk正是使用zxid来标识这个状态，上图描述了客户端在重连情况下zxid的作用。

当客户端因超时与S1断开连接后，客户端开始尝试连接S2，但S2延迟于客户端所识别的状态。而S3的状态与客户端识别的状态一致。所以客户端可以安全的连接S3。

> 时间戳

创建时间是znode创建时的时间，创建后就不会改变；修改时间在每次更新znode时都会发生变化。

> - ctime：节点创建时间
> - mtime：节点修改时间,每次对znode的修改都会更新mtime

