---
title: Zookeeper笔记 (六) 祥解ZAB协议
date: 2019-05-06 15:13:13
tags:
- zookeeper
categories:
- zookeeper
cover: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1557738630&di=238c89426d16639b9f9b21f58ae2a021&imgtype=jpg&er=1&src=http%3A%2F%2Fb-ssl.duitang.com%2Fuploads%2Fblog%2F201511%2F02%2F20151102140204_WUSwE.jpeg
---

本文参考地址：[【Zookeeper 学习笔记】—实例详解ZooKeeper ZAB协议、分布式锁与领导选举](http://cmsblogs.com/?p=4113)

+ ### 原子广播（ZAB）

zk设计了一种名为原子广播(ZAB)的支持崩溃恢复的一致性协议。基于该协议，zk实现了一种主从模式的系统架构来保持集群中各个副本之间的数据一致性，确保了写操作的一致性和可用性。

根据ZAB协议，所有写操作都必须通过leader完成，leader写入本地日志后再复制到所有的follower节点。

一旦leader节点无法工作，ZAB协议能够自动从follower节点中重新选出一个合适的替代者，即新的leader，该过程称为领导选举。该领导选举过程，是ZAB协议中最为重要和复杂的过程。

- #### 写leader

通过leader进行写操作的流程如下图所示：

![image](https://gitee.com/chenssy/blog-home/raw/master/image/series-images/zookeeper/7db011b1e4744c2e92eadbc427d28726.jpeg)

> 1. 客户端向leader发起写请求
> 2. leader将写请求以proposal的形式发给所有follower并等待`ACK(follower回复消息)`
> 3. follower收到leader的proposal后返回ACK
> 4. leader得到过半数的ACK(leader对自己默认有一个ACK)后向所有的follower和Observer发送Commit
> 5. leader将处理结果返回给客户端
> 6. leader不需要得到Observer的ACK，即Observer无投票权
> 7. Observer虽然无投票权，但仍需同步leader的数据从而在处理读请求时可以返现尽可能新的数据

+ #### 写follower/Observer

通过follower/Observer进行写操作的流程如下图所示：

![image](https://gitee.com/chenssy/blog-home/raw/master/image/series-images/zookeeper/87b9d739061c452d8559dcb0ad9833c6.jpeg)

> 1. follower/Observer均可接受写请求，但不能直接处理，而是将写请求转发给leader处理
> 2. 除了多了一步转发，其他流程与直接写leader一致

+ #### 读操作

leader/Observer/follower都可直接处理读请求：

![image](https://gitee.com/chenssy/blog-home/raw/master/image/series-images/zookeeper/fd0b7a4ca80949f1823e2d4e28610ea1.jpeg)

> 1. 由于处理读请求不需要服务器之间的交互，follower/Observer越多，整体可处理的读请求量越大。即读性能越好

+ #### 支持的领导选举算法

通过electionAlg配置项设置zk用于领导选举的算法。

到3.4.10版本为止，可选项有：

> - 0 基于UDP的LeaderElection
> - 1 基于UDP的FastLeaderElection
> - 2 基于UDP和认证的FastLeaderElection
> - 3 基于TCP的FastLeaderElection

在3.4.10版本中，默认值为3。另外三种算法已被弃用，并且计划在之后的版本中彻底删除而且不再支持。

### FastLeaderElection原理

+ **1、myid**

每个zk服务器都需要在数据文件夹（data）下创建一个名为myid的文件，该文件包含整个zk集群中唯一的id(整数)。例如：某zk集群包含三台服务器，hostname分别为zoo1、zoo2和zoo3，其中myid分别为1、2和3，则在配置文件中其id与hostname必须一一对应。如下所示，在该配置项里的server.后面的数据即为myid

```ruby
server.1=zoo1:2888:3888
server.2=zoo2:2888:3888
server.3=zoo3:2888:3888
```

+ **2、zxid**

类似于RDBMS中的事务id，用于标识一次更新操作的proposal id。为了保证顺序性，该zxid必须单独递增。因此zk使用一个64位的数来表示，高32位是leader的epoch（周期），从1开始，每次选出新的leader，epoch加一。低32位为该epoch内的序号，每次epoch变化，都将低32位的序号重置。这样就保证了zxid的全局递增性。

+ **3、服务器状态**

> - LOOKING：不确定leader状态。表示集群中没有leader，会发起leader选举。
> - FOLLOWING：跟随者状态。表明当前服务器角色是Follower，并且它知道Leader是谁。
> - LEADING：领导者状态。表明当前服务器角色是Leader，它会维护与FOllower间的心跳。
> - OBSERVING：观察者状态。表明当前服务器角色是Observer，不参与选举，不参与投票。

+ **4、选票数据结构**

每个服务器在进行领导选举时，会发送如下关键信息：

> - logicClock：每个服务器会维护一个自增整数，表明该服务器发起的是第多少轮的投票
> - state：当前服务器的状态
> - self_id：当前服务器的myid
> - self_zxid：当前服务器所保存的数据的最大zxid
> - vote_id：被推举的服务器的myid
> - vote_zxid：被推举的服务器所保存的数据的最大zxid

+ **5、投票流程**



**自增选举轮次**

所有有效的投票必须在同一轮次。每个服务器在开始投票时，会先对自己的logicClock进行自增操作。



**初始化选票**

每个服务器在广播自己选票前，会先将自己的投票箱清空。该投票箱记录了所收到的选票。例：服务器2投票给服务器3，服务器3投票给服务器1，则服务器1的投票箱为(2，3)，(3，1)，(1，1)。票箱中只会记录每一投票者的最后一票，如投票者更新自己的选票，则其他服务器收到该新选票后会在自己的票箱中更新该服务器的选票。



**发送初始化选票**

每个服务器最开始都是通过广播把票投给自己



**接收外部选票**

服务器会尝试从其他服务器获取投票，并记入自己的投票箱。若无法获取任何外部投票，则会确认自己是否与集群中其他服务器保持着有效连接。如果是，则再次发送自己的投票，反之则马上与之建立连接。



**判断选举轮次**

收到外部投票后，首先根据投票信息中包含的logicClock来进行不同处理：

1. 外部投票的logicClock大于自己的logicClock：说明该服务器的选举轮次落后其他服务器的选举轮次，立即清空自己的投票箱并将自己的logicClock更新为收到的logicClock，然后再对比自己之前的投票与收到的投票以确定是否需要变更自己的投票，最终再次将自己的投票广播出去。

2. 外部投票的logicClock小于自己的logicClock：当前服务器直接忽略该投票，继续处理下一个处理。

3. 外部投票的logicClock与自己相等：进行选票PK

**选票PK**

选票PK是基于(self_id，self_zxid)与(vote_id，vote_zxid)对比

1. 外部投票的logicClock大于自己的logicClock：将自己的logicClock及自己选票的logicClock变更为收到的logicClock。

2. 外部投票的logicClock与自己的logicClock一致：对比二者的vote_zxid，若外部投票的vote_zxid较大，将将自己的vote_zxid与vote_myid更新为收到的vote_zxid和vote_myid并广播出去，另外将收到的票与自己更新后的票放入自己的票箱。若票箱内已存在(self_myid，self_zxid)相同的选票，则直接覆盖。

3. 若二者vote_zxid一致：比较二者的vote_，若外部的vote_myid较大，则将自己的vote_myid更新为收到的vote_myid并广播出去，另外将收到的票与自己更新后的票放入自己的票箱。

   

**统计选票**

如果已经确定有过半服务器认可自己的投票，则终止投票。否则继续接收其他服务器的投票。



**更新服务器状态**

投票终止后，服务器更新自身状态。若过半的票投给了自己，则自身状态更新为LEADING，否则更新为FOLLOWING。



**集群领导选举**

**1、初始投票给自己**

集群刚启动时，所有服务器的logicClock为1，zxid为0。各服务器初始化后，都投票给自己，并将自己的票存入自己的票箱，如下图所示：

![image](https://gitee.com/chenssy/blog-home/raw/master/image/series-images/zookeeper/e79e5f68160c4443ae9ec2ea6cd9f417.jpeg)



> (1，1，0)：
>
> 第一位代表投出该票的服务器的logicClock
>
> 第二位表示服务器的myid
>
> 第三位表示服务器最大的zxid

**2、更新选票**

服务器收到外部投票后，进行选票PK，相应更新自己的选票并广播出去，并将合适的选票存入自己的票箱：

![image](https://gitee.com/chenssy/blog-home/raw/master/image/series-images/zookeeper/21602b7ff75e44a7b509bd5fb2b0785c.jpeg)

服务器1收到服务器2的选票1, 2, 0）和服务器3的选票（1, 3, 0）后，由于所有的logicClock都相等，所有的zxid都相等，因此根据myid判断应该将自己的选票按照服务器3的选票更新为（1, 3, 0），并将自己的票箱全部清空，再将服务器3的选票与自己的选票存入自己的票箱，接着将自己更新后的选票广播出去。此时服务器1票箱内的选票为(1, 3)，(3, 3)。同理，服务器2收到服务器3的选票后也将自己的选票更新为（1, 3, 0）并存入票箱然后广播。此时服务器2票箱内的选票为(2, 3)，(3, ,3)。服务器3根据上述规则，无须更新选票，自身的票箱内选票仍为（3, 3）。服务器1与服务器2更新后的选票广播出去后，由于三个服务器最新选票都相同，最后三者的票箱内都包含三张投给服务器3的选票。

**3、根据选票确定角色**

根据上述选票，三个服务器一致认为此时服务器3应该是Leader。因此服务器1和2都进入FOLLOWING状态，而服务器3进入LEADING状态。之后Leader发起并维护与Follower间的心跳。

![image](https://gitee.com/chenssy/blog-home/raw/master/image/series-images/zookeeper/6f07c1ece9144bcd957ca2b2c3917ddf.jpeg)

**Follower重启选举**

**1、Follower重启投票给自己**

Follower重启，或者发生网络分区后找不到Leader，会进入LOOKING状态并发起新的一轮投票。

[![img](https://gitee.com/chenssy/blog-home/raw/master/image/series-images/zookeeper/5d682ff632324eb182544ab0691908b7.jpeg)](https://gitee.com/chenssy/blog-home/raw/master/image/series-images/zookeeper/5d682ff632324eb182544ab0691908b7.jpeg)

**2、发现已有Leader后成为Follower**

服务器3收到服务器1的投票后，将自己的状态LEADING以及选票返回给服务器1。服务器2收到服务器1的投票后，将自己的状态FOLLOWING及选票返回给服务器1。此时服务器1知道服务器3是Leader，并且通过服务器2与服务器3的选票可以确定服务器3确实得到了超过半数的选票。因此服务器1进入FOLLOWING状态。

[![img](https://gitee.com/chenssy/blog-home/raw/master/image/series-images/zookeeper/ca8d8cfabe6d4c2086e7bc5b40979ce4.jpeg)](https://gitee.com/chenssy/blog-home/raw/master/image/series-images/zookeeper/ca8d8cfabe6d4c2086e7bc5b40979ce4.jpeg)

**Leader重启选举**

**1、Follower发起新投票**

Leader（服务器3）宕机后，Follower（服务器1和2）发现Leader不工作了，因此进入LOOKING状态并发起新的一轮投票，并且都将票投给自己。

[![img](https://gitee.com/chenssy/blog-home/raw/master/image/series-images/zookeeper/85cea8558ca84c9fb3cd1feae4fe611f.jpeg)](https://gitee.com/chenssy/blog-home/raw/master/image/series-images/zookeeper/85cea8558ca84c9fb3cd1feae4fe611f.jpeg)

**2、广播更新选票**

服务器1和2根据外部投票确定是否要更新自身的选票。这里有两种情况：

服务器1和2的zxid相同。例如在服务器3宕机前服务器1与2完全与之同步。此时选票的更新主要取决于myid的大小

服务器1和2的zxid不同。在旧Leader宕机之前，其所主导的写操作，只需过半服务器确认即可，而不需所有服务器确认。换句话说，服务器1和2可能一个与旧Leader同步（即zxid与之相同）另一个不同步（即zxid比之小）。此时选票的更新主要取决于谁的zxid较大

在上图中，服务器1的zxid为11，而服务器2的zxid为10，因此服务器2将自身选票更新为（3, 1, 11），如下图所示。

[![img](https://gitee.com/chenssy/blog-home/raw/master/image/series-images/zookeeper/2cec4cfe93214094bf3651c89b915c26.jpeg)](https://gitee.com/chenssy/blog-home/raw/master/image/series-images/zookeeper/2cec4cfe93214094bf3651c89b915c26.jpeg)

**3、选出新Leader**

经过上一步选票更新后，服务器1与服务器2均将选票投给服务器1，因此服务器2成为Follower，而服务器1成为新的Leader并维护与服务器2的心跳。

[![img](https://gitee.com/chenssy/blog-home/raw/master/image/series-images/zookeeper/e91d1bd8a3af48bfb51287143ad87641.jpeg)

**4、旧Leader恢复后发起选举**

旧的Leader恢复后，进入LOOKING状态并发起新一轮领导选举，并将选票投给自己。此时服务器1会将自己的LEADING状态及选票（3, 1, 11）返回给服务器3，而服务器2将自己的FOLLOWING状态及选票（3, 1, 11）返回给服务器3。如下图所示。

[![img](https://gitee.com/chenssy/blog-home/raw/master/image/series-images/zookeeper/7ad67a3837184c06963471b0fd1155dd.jpeg)](https://gitee.com/chenssy/blog-home/raw/master/image/series-images/zookeeper/7ad67a3837184c06963471b0fd1155dd.jpeg)](https://gitee.com/chenssy/blog-home/raw/master/image/series-images/zookeeper/e91d1bd8a3af48bfb51287143ad87641.jpeg)

**5、旧Leader成为Follower**

服务器3了解到Leader为服务器1，且根据选票了解到服务器1确实得到过半服务器的选票，因此自己进入FOLLOWING状态。

[![img](https://gitee.com/chenssy/blog-home/raw/master/image/series-images/zookeeper/70e35737fb934074840a6d1d759db320.jpeg)](https://gitee.com/chenssy/blog-home/raw/master/image/series-images/zookeeper/70e35737fb934074840a6d1d759db320.jpeg)

**Commit过的数据不丢失**

**1、Failover前状态**

为更好演示Leader Failover过程，本例中共使用5个ZooKeeper服务器。A作为Leader，共收到P1、P2、P3三条消息，并且Commit了1和2，且总体顺序为P1、P2、C1、P3、C2。根据顺序性原则，其它Follower收到的消息的顺序肯定与之相同。其中B与A完全同步，C收到P1、P2、C1，D收到P1、P2，E收到P1，如下图所示。

[![img](https://gitee.com/chenssy/blog-home/raw/master/image/series-images/zookeeper/2e7f5a41cdb44b6f96cf5d103b3c99e1.jpeg)](https://gitee.com/chenssy/blog-home/raw/master/image/series-images/zookeeper/2e7f5a41cdb44b6f96cf5d103b3c99e1.jpeg)

这里要注意：

由于A没有C3，意味着收到P3的服务器的总个数不会超过一半，也即包含A在内最多只有两台服务器收到P3。在这里A和B收到P3，其它服务器均未收到P3

由于A已写入C1、C2，说明它已经Commit了P1、P2，因此整个集群有超过一半的服务器，即最少三个服务器收到P1、P2。在这里所有服务器都收到了P1，除E外其它服务器也都收到了P2

**2、选出新Leader**

旧Leader也即A宕机后，其它服务器根据上述FastLeaderElection算法选出B作为新的Leader。C、D和E成为Follower且以B为Leader后，会主动将自己最大的zxid发送给B，B会将Follower的zxid与自身zxid间的所有被Commit过的消息同步给Follower，如下图所示。

[![img](https://gitee.com/chenssy/blog-home/raw/master/image/series-images/zookeeper/7e449b6c177b459a9365d7552b753f81.jpeg)](https://gitee.com/chenssy/blog-home/raw/master/image/series-images/zookeeper/7e449b6c177b459a9365d7552b753f81.jpeg)

在上图中：

P1和P2都被A Commit，因此B会通过同步保证P1、P2、C1与C2都存在于C、D和E中

P3由于未被A Commit，同时幸存的所有服务器中P3未存在于大多数据服务器中，因此它不会被同步到其它Follower

**3、通知Follower可对外服务**

同步完数据后，B会向D、C和E发送NEWLEADER命令并等待大多数服务器的ACK（下图中D和E已返回ACK，加上B自身，已经占集群的大多数），然后向所有服务器广播UPTODATE命令。收到该命令后的服务器即可对外提供服务。

[![img](https://gitee.com/chenssy/blog-home/raw/master/image/series-images/zookeeper/04dd1963c4874e37b60fe2e1d0ba9d7c.jpeg)](https://gitee.com/chenssy/blog-home/raw/master/image/series-images/zookeeper/04dd1963c4874e37b60fe2e1d0ba9d7c.jpeg)

**未Commit过的消息对客户端不可见**

在上例中，P3未被A Commit过，同时因为没有过半的服务器收到P3，因此B也未Commit P3（如果有过半服务器收到P3，即使A未Commit P3，B会主动Commit P3，即C3），所以它不会将P3广播出去。

具体做法是，B在成为Leader后，先判断自身未Commit的消息（本例中即P3）是否存在于大多数服务器中从而决定是否要将其Commit。然后B可得出自身所包含的被Commit过的消息中的最小zxid（记为min_zxid）与最大zxid（记为max_zxid）。C、D和E向B发送自身Commit过的最大消息zxid（记为max_zxid）以及未被Commit过的所有消息（记为zxid_set）。B根据这些信息作出如下操作：

如果Follower的max_zxid与Leader的max_zxid相等，说明该Follower与Leader完全同步，无须同步任何数据

如果Follower的max_zxid在Leader的(min_zxid，max_zxid)范围内，Leader会通过TRUNC命令通知Follower将其zxid_set中大于Follower的max_zxid（如果有）的所有消息全部删除

上述操作保证了未被Commit过的消息不会被Commit从而对外不可见。

上述例子中Follower上并不存在未被Commit的消息。但可考虑这种情况，如果将上述例子中的服务器数量从五增加到七，服务器F包含P1、P2、C1、P3，服务器G包含P1、P2。此时服务器F、A和B都包含P3，但是因为票数未过半，因此B作为Leader不会Commit P3，而会通过TRUNC命令通知F删除P3。如下图所示。

[![img](https://gitee.com/chenssy/blog-home/raw/master/image/series-images/zookeeper/23f89673d41a4e2fbdd4fe301cfa321f.jpeg)](https://gitee.com/chenssy/blog-home/raw/master/image/series-images/zookeeper/23f89673d41a4e2fbdd4fe301cfa321f.jpeg)

+ 小结

由于使用主从复制模式，所有的写操作都要由Leader主导完成，而读操作可通过任意节点完成，因此ZooKeeper读性能远好于写性能，更适合读多写少的场景。

虽然使用主从复制模式，同一时间只有一个Leader，但是Failover机制保证了集群不存在单点失败（SPOF）的问题。

ZAB协议保证了Failover过程中的数据一致性。

服务器收到数据后先写本地文件再进行处理，保证了数据的持久性。

+ 