---
title: Zookeeper笔记 (四) Watcher机制
date: 2019-05-05 15:46:20
tags:
- zookeeper
- watcher
categories:
- zookeeper
cover: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1557064021693&di=92260587dbb9fc5c49b1a022240cc96a&imgtype=0&src=http%3A%2F%2Fwww.33lc.com%2Farticle%2FUploadPic%2F2012-8%2F20128161646424579.jpg
---

本文参考地址：[【Zookeeper 学习笔记】—Watcher机制](http://cmsblogs.com/?p=4105)

+ ### 前言

watcher机制是zookeeper最重要的三大特性`数据节点znode、watcher机制、ACL权限控制`中的其中一个，它是zk很多应用场景的一个前提，比如集群管理、集群配置、发布/订阅。

watcher机制涉及到客户端与服务器(不止一个机器，一般是集群)的两者数据通信与消息通信，除此之外还涉及到客户端的watchManager。

+ ### watcher原理框架

![image](https://gitee.com/chenssy/blog-home/raw/master/image/series-images/zookeeper/zookeeper-4001.png)

由图看出，zk的watcher由客户端、客户端watcherManager和zk服务器组成。整个过程涉及了消息通信及数据存储。

> - zk客户端向服务器注册watcher的同时，还会降watcher对象存储在客户端的watchManager。
> - zk服务器触发watcher事件后，会向客户端发送通知，客户端线程从watchManager中回调watcher执行相应的功能。

服务端一般由多台共同对外提供服务，里面会设计到zk专有的ZAB协议(分布式原子广播协议)。因为ZAB协议是zookeeper的实现精髓，有了ZAB协议才能使zk真正落地，真正的高可靠、数据同步。

![image](https://gitee.com/chenssy/blog-home/raw/master/image/series-images/zookeeper/zookeeper-4002.png)

假设上图中的小红旗是一个watcher，当小红旗被创建并注册到node1节点后，就会监听node1+node_a+node_b或node_a+node_b。这里会有两种情况是因为在创建watcher注册时会有多种选择，并且watcher不能监听到孙节点。注意：watcher设置后，一旦触发一次就会失效。若要一直监听，可以在process回调函数中重新注册相同的watcher。

+ ### watcher注册过程

  + 客户端注册

  ![image](https://gitee.com/chenssy/blog-home/raw/master/image/series-images/zookeeper/zookeeper-4009.png)

  zk客户端在注册时会先向zk服务器请求注册，服务器会返回请求响应，如果响应成功则zk服务端把watcher对象放到客户端的watchManager管理并返回响应给客户端。

  + 服务端注册

  ![image](https://gitee.com/chenssy/blog-home/raw/master/image/series-images/zookeeper/zookeeper-40010.png)

  - watcherManager

  > - zk服务端watcher的管理者
  > - 从两个维度维护watcher
  > - watchTable从数据节点的粒度来维护
  > - watcher2Paths从watcher的粒度来维护
  > - 负责watcher事件的触发

  + 客户端回调watcher

  > - 反序列化，将字节流转换成WatcherEvent对象
  > - 处理chrootPath。获取节点的根节点路径，然后在搜索树
  > - 还原watchedEvent：把watcherEvent对象转换成WatchedEvent。主要把zk服务器的watchedEvent事件变为watcherEvent，标为已watch触发
  > - 回调watcher：把watchedEvent对象交给eventThread线程。eventThread线程主要是负责从客户端的zkWatchManager中取出watcher，并放入waitingEvents队列中，然后供客户端获取。

+ ### 特点

> - 主动推送：watch被触发时，由服务端主动将更新推送给客户端，不需要客户端轮询。
> - 一次性：数据变化时，watch只会被触发一次。若客户端想得到后续更新的通知，必须在watch触发后重新注册一个watch。
> - 可见性：如果一个客户端在读请求中附带watch，watch被触发的同时再次读取数据，客户端在得到watch消息之前无法看到更新后的数据。更新通知先于更新结果。
> - 顺序性：如果多个更新触发了多个watch，那么watch被触发的顺序与更新顺序一直。