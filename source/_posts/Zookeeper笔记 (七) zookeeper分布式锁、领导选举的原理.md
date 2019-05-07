---
title: Zookeeper笔记 (七) zookeeper分布式锁、领导选举的原理
date: 2019-05-06 17:10:49
tags:
- zookeeper
- ZAB
categories:
- zookeeper
- 分布式锁
cover:
---

本文参考地址：[【Zookeeper 学习笔记】—实例详解ZooKeeper ZAB协议、分布式锁与领导选举](http://cmsblogs.com/?p=4113)

+ ### 分布式锁与领导选举关键点

> + ##### 最多一个获取锁 / 成为Leader

对于分布式锁（这里特指排它锁）而言，任意时刻，最多只有一个进程（对于单进程内的锁而言是单线程）可以获得锁。

对于领导选举而言，任意时间，最多只有一个成功当选为Leader。否则即出现脑裂（Split brain）

> + ##### 锁重入 / 确认自己是Leader

对于分布式锁，需要保证获得锁的进程在释放锁之前可再次获得锁，即锁的可重入性。

对于领导选举，Leader需要能够确认自己已经获得领导权，即确认自己是Leader。

> + ##### 释放锁 / 放弃领导权

锁的获得者应该能够正确释放已经获得的锁，并且当获得锁的进程宕机时，锁应该自动释放，从而使得其他竞争方可以获得该锁，从而避免出现死锁的状态。

领导应该可以主动放弃领导权，并且当领导所在进程宕机时，领导权应自动释放，从而使得其他竞争方可以重新竞争领导而避免进入无主状态

> + ##### 感知锁释放 / 领导权的放弃

当获得锁的一方释放锁时，其它对于锁的竞争方需要能够感知到锁的释放，并再次尝试获取锁。

原来的Leader放弃领导权时，其它参与方应该能够感知该事件，并重新发起选举流程。

+ ### 非公平领导选举

分布式锁与领导选举的技术要点非常相似，这里以领导选举为例来说明二者的实现原理

> + ##### 选主过程

假设有三个zk客户端同时竞争leader，这三个客户端同时向zk集群注册临时节点（Ephemeral）且无序类型（Non-sequence）的节点。路径都为 /zkroot/leader (路径名自定义)

![image](https://gitee.com/chenssy/blog-home/raw/master/image/series-images/zookeeper/2ae468645d9a457b8f6a0bc6701ae9e9.jpeg)

由于是无序节点，这三个客户端只会有一个创建成功，其他节点均创建失败。此时，创建成功的客户端1即成功竞选为Leader。其他客户端此时均为Follower。

> + ##### 放弃领导权

如果Leader打算放弃领导权，直接删除/zkroot/leader节点即可。

如果Leader进程意外宕机，其与zk间的session也就结束，该临时节点自动删除。

此时 /zkroot/leader节点不存在了，对于其他竞选的客户端而言，之前的Leader放弃了领导权。

> + ##### 感知领导权的放弃

创建节点失败的节点，除了成为Follower之外，还会向 /zkroot/leader注册一个Watcher，一旦Leader放弃领导权，节点被删除，所有的FOllower都会收到通知。

> + ##### 重新选举

感知到旧Leader放弃领导权后，所有的FOllower可以再次发起一轮领导选举：

![image](https://gitee.com/chenssy/blog-home/raw/master/image/series-images/zookeeper/67966be66521475e93bed856ac5584df.jpeg)

新一轮的领导选举方法与最初的领导选举方法完全一样。由于选举结果无法预测，与他们在第一轮选举的顺序无关，所以这种方案才被称为非公平锁

> + ##### 小结

非公平锁实现简单，每一轮选举方法都一样。竞争方不多的情况下，效率高。若竞争方有上万的话，同时参与竞选就意味着同时会有上万个请求发给zk。由于zk存在单点写的问题，写性能不高。同时Leader放弃领导，zk还需要同时通知上万个FOllower，负载较大。

+ ### 公平领导选举

> + ##### 选主过程

公平领导选举中，各客户端创建 /zkroot/leader节点，且类型为临时顺序节点。

![image](https://gitee.com/chenssy/blog-home/raw/master/image/series-images/zookeeper/6789a44c37454f5e9712d0e6ac34bf6e.jpeg)

由于是顺序类型节点，所以各客户端均创建成功。每个客户端会判断自己创建的节点序号是否是当前最小的。如果是，则该客户端成为leader，否则为FOllower。这里客户端1成为leader。

> + ##### 放弃领导权

leader如果主动放弃领导权，直接删除其创建的节点即可。

如果leader所在进程意外宕机，session会话结束，其创建的临时节点会自动删除。

> + ##### 感知领导权的放弃

与非公平模式不同，每个FOllower并非都watch由leader创建出来的节点，而是watch刚好比自己序号小的节点。

一旦leader宕机，/zkroot/leader1删除，客户端2可得到通知。此时客户端3由于watch的是 /zkroot/leader2，所以不会得到通知。

> + ##### 重新选举

客户端2得到 /zkroot/leader1被删除的通知后，不会立即成为新的leader。而是先判断自己的序号2是不是当前最小的。如果是最小的，客户端2就会成为新的leader。

![image](https://gitee.com/chenssy/blog-home/raw/master/image/series-images/zookeeper/065670a367bd4a548cb98fe7d990cf57.jpeg)

`注意：`如果客户端1放弃领导权之前，客户端2宕机，客户端3会收到通知。此时客户端3不会立即成为新的leader，而是判断自己的序号是否是最小的。由于此时 /zkroot/leader1还在，因此客户端3不会成为新的leader，并向 /zkroot/leader1节点创建watch。

![image](https://gitee.com/chenssy/blog-home/raw/master/image/series-images/zookeeper/361658e79aa8421db21fcbe4811f4d35.jpeg)

> + ##### 小结

实现相对复杂。

扩展性好，每个客户端都只watch一个节点且每次节点被删除只会通知一个客户端。

旧leader放弃领导权时，其他客户端按节点序号的先后顺序成为新leader，这也是公平模式的由来。

延迟相对非公平模式要高，因为它必须等待特定节点得到通知才能选出新的leader

+ ### 总结

zk的领导选举或分布式锁的实现均基于zk节点的特性和通知机制。

