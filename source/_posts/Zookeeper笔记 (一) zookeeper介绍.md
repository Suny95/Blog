---
title: Zookeeper笔记 (一) zookeeper介绍
date: 2019-04-30 11:26:12
tags:
- zookeeper
- znode
categories:
- zookeeper
cover: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1556617451718&di=cd0ef998111940345f9ba9452d96bdd3&imgtype=0&src=http%3A%2F%2Fb-ssl.duitang.com%2Fuploads%2Fitem%2F201204%2F03%2F20120403223937_dsku3.jpeg
---
本文参考地址: [【Zookeeper 学习笔记】—zookeeper基本介绍](http://cmsblogs.com/?p=4099)

- ### zookeeper是什么

zookeeper是一个高性能、开源的分布式应用协调服务，它提供了简单原始的功能，分布式应用可以基于它实现更高级的服务，比如实现同步(分布式锁)、配置管理、集群管理。它被设计为易于编程，使用文件系统目录树作为数据模型。服务端使用Java语言编写，并且提供了Java和C语言的客户端。

- ### zookeeper数据模型

![image](https://gitee.com/chenssy/blog-home/raw/master/image/series-images/zookeeper/zookeeper-1001.png)

- 树状结构中每个节点成为znode
- 每个znode都可以有数据(byte[]类型)，也可以有子节点
- znode路径使用斜线分割，zk中没有相对路径的说法，所有路径都要写为绝对路径的方式
- 当zk中节点数据发生变化时，版本号递增
- 可以对znode中的数据进行读写操作

- ### zookeeper典型的应用场景

> 数据发布/订阅

数据发布/订阅即所谓的配置中心：发布者将数据发布到zk的一个或一系列节点上，订阅者进行数据订阅，可以及时得到数据的变化通知，如下图所示：

![image](https://gitee.com/chenssy/blog-home/raw/master/image/series-images/zookeeper/zookeeper-1002.png)

应用A将数据发布到zkServer的某个znode上，应用B和C会现在zkServer上注册监听该节点的watcher(相当于Listener，基于RPC实现)，一旦该节点有数据变化，B和C的watcher就会得到通知，继而从zkServer获取最新的数据。

> 负载均衡

- zk实现负载均衡本质上市利用zk的配置管理功能，zk实现负载均衡的步骤为：
- 服务提供者把自己的域名及IP端口映射注册到zk中
- 服务消费者通过域名从zk中获取到对于的ip和端口，这里的ip和端口可能有多个，只获取其中一个
- 当服务提供者宕机时，对应的域名和ip就会减少一个映射

> 命名服务

在分布式系统中，命名服务(name service)是很重要的应用场景，通过zk也可以实现类似于J2EE中JNDI的效果。分布式环境下命名服务更多的是资源定位，并不是真正的实体资源，其本质也是zk的集中配置和管理。

> 分布式协调/通知

通过zk的watcher和通知机制实现分布式锁和分布式事务

> 集群管理

获取当前集群中机器的数量、集群中机器的运行状态、集群中节点的上下线操作、集群节点的统一配置等。

此外还可以通过zk实现集群master节点的选举、分布式锁(排它锁、共享锁)、分布式队列等。

- ### zookeeper中的一些基本概念

> 集群角色

- Leader：为客户端提供读写服务，一个zk集群同一时间只会有一个实际工作的leader，它会发起并维护各Follower及Observer间的心跳。所有写操作必须要通过leader完成再由leader将写操作广播给其他服务器
- Follower：为客户端提供读服务，客户端到Follower的写请求会转交给Leader，Follower参与Leader选举
- Observer：为客户端提供读服务，不参与Leader的选举，一般是为了增强zk集群的读请求并发能力

> 会话(Session)

- session是客户端与zk服务端之间建立的长连接
- zk在一个会话中进行心跳检测来感知客户端连接的存活
- zk客户端在一个会话中接收来自服务端的watch事件通知
- zk可以给会话设置超时时间

> Znode

- znode是zk树形结构中的数据节点，用于存储数据
- znode分为持久节点和临时节点
  - 持久节点：一旦创建，除非主动调用删除操作， 否则一直存储在zk中
  - 临时节点：与客户端会话绑定，一旦客户端时效，这个客户端创建的所以临时节点都会自动删除
- 可以为持久节点或临时节点设置Sequential属性，该属性会自动在节点名称后面追加10位的整形数字

> watcher

- watcher监听在znode节点上
- 当节点的数据更新或子节点的状态发生变化都会使客户端的watcher得到通知

> ACL(访问控制)

类似于Linux下的权限控制

- CREATE：创建子节点的权限
- READ：获取节点数据和子节点列表的权限
- WRITE：更新节点数据的权限
- DELETE：删除子节点的权限
- ADMIN：设置节点ACL的权限