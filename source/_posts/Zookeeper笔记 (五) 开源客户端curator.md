title: Zookeeper笔记 (五) 开源客户端curator
date: 2019-05-05 16:15:32
tags:
- zookeeper
- curator
categories:
- zookeeper
---
** {{ title }}：** <Excerpt in index | 首页摘要>
开源客户端curator
<!-- more -->
<The rest of contents | 余下全文>

本文参考地址：[【Zookeeper 学习笔记】—zookeeper开源客户端curator](http://cmsblogs.com/?p=4109)

+ ### zookeeper第三方开源客户端

zookeeper的第三方开源客户端主要有zkClient、curator。其中zkClient解决了session会话超时重连、Watcher反复注册等问题，提供了更加简洁的API，但zkClient社区不活跃，文档不够完善。而curator是Apache基金会的顶级项目之一，它解决了session会话超时重连、Watcher反复注册、NodeExitsException异常等问题，curator具有更加完善的文档。

+ ### curator客户端api介绍

1. curator-framework：对zookeeper底层api的一些封装
2. curato-client：提供一些客户端操作，如重试策略等
3. curator-recipes：封装了一些高级特性，如Cache事件监听、选举、分布式锁、分布式计数器、分布式Barrier等

+ #### 创建zookeeper会话 (一)

该方法会返回一个`org.apache.curator.framework.CuratorFramework`类型的对象

```java
public static CuratorFramework newClient(String connectString, RetryPolicy retryPolicy)
public static CuratorFramework newClient(String connectString, int sessionTimeoutMs, int connectionTimeoutMs, RetryPolicy retryPolicy)
```

参数说明：

> - connectString：逗号分开的ip：port对
> - sessionTimeoutMs：会话超时时间，单位为毫秒，默认60000ms，指连接建立完后多久没收到心跳检测，超过该时间即会话超时
> - connectTimeoutMs：连接创建超时时间，单位为毫秒，默认15000ms，指客户端与服务端建立连接时多长时间没连上就算超时
> - retryPolicy：重试策略

```java
/**
 * retryPolicy定义
 */
public interface RetryPolicy {
    /**
     * 当allowRetry返回true时继续重试，返回false不再重试，可以通过实现该接口来自定义重试策略
     *
     * @param retryCount 到目前为止已经重试的次数(初始为0)
     * @param elapsedTimeMs 尝试操作后经过的时间(毫秒)
     * @param sleeper 使用它来休眠，不需要使用Thread.sleep
     * @return true/false
     */
    public boolean allowRetry(int retryCount, long elapsedTimeMs, RetrySleeper sleeper);
}
```

+ ##### curator提供的重试策略

  + ExponentialBackofRetry：该重试策略随着重试次数的增加，sleep时间呈指数增长

  ```java
  public ExponentialBackoffRetry(int baseSleepTimeMs, int maxRetries)
  public ExponentialBackoffRetry(int baseSleepTimeMs, int maxRetries, int maxSleepMs)
  ```

  > 第retryCount次重试的sleep时间计算方式为：
  >
  > baseSleepTimeMs * Math.max(1, random.nextInt(1 << retryCount + 1))，如果该值大于maxSleepMs，则sleep时间为maxSleepMs，如果重试次数大于maxRetries，则不再重试

  + RetryNTimes：该重试策略重试指定次数，每次sleep固定时间

  ```java
  public RetryNTimes(int n, int sleepMsBetweenRetries)
  ```

  > n是重试次数
  >
  > sleepMsBetweenRetries是sleep时间

  + RetryOneTime：该重试策略只重试一次
  + RetryUntilElapsed：该重试策略对重试次数无限制，但对总的重试时间做限制

  ```java
  public RetryUntilElapsed(int maxElapsedTimeMs, int sleepMsBetweenRetries)
  ```

  >  maxElapsedTimeMs是最大的重试时间
  >
  > sleepMsBetweenRetries是sleep的时间间隔

+ #### 创建zookeeper会话 (二)

  + 通过Builder的方式创建会话
  
  ```java
  RetryPolicy retryPolicy = new ExponentialBackoffRetry(3000, 5);
  CuratorFramework client =  CuratorFrameworkFactory.builder()
          .connectString("192.168.0.102:2181,192.168.0.102:2182,192.168.0.102:2183")
          .sessionTimeoutMs(30000)
    			.connectionTimeoutMs(15000)
          .retryPolicy(retryPolicy)
    			//根节点
          .namespace("curatorTest")
          .build();
  ```
  
  + 开启会话
  
```java
  client.start();
```

  + 创建zookeeper节点

  curator执行各种操作都必须先获得一个构建该操作的包装类(Builder对象)，创建节点则需要获得一个`org.apache.curator.framework.api.CreateBuilder`对象，然后用这个对象来构建创建节点的操作：

  ```java
  // 递归创建(持久)父目录
  public ProtectACLCreateModeStatPathAndBytesable<String> creatingParentsIfNeeded()
  // 设置创建节点的属性
  public ACLBackgroundPathAndBytesable<String> withMode(CreateMode mode)
  // 设置节点的acl属性
  public ACLBackgroundPathAndBytesable<String> withACL(List<ACL> aclList, boolean applyToParents)
  // 指定创建节点的路径和节点上的数据
  public String forPath(final String givenPath, byte[] data) throws Exception
  ```

  示例：

  ```java
  String test1Data = client.create()
                      .creatingParentsIfNeeded()
                      .withMode(CreateMode.PERSISTENT)
                      .withACL(ZooDefs.Ids.OPEN_ACL_UNSAFE)
                      .forPath("/curatorTest/test1", "test1".getBytes());
  ```

  >  CreateMode.PERSISTENT：持久节点
  >
  > CreateMode.PERSISTENT_SEQUENTIAL：持久顺序节点
  >
  > CreateMode.EPHEMERAL：临时节点
  >
  > CreateMode.EPHEMERAL_SEQUENTIAL：临时顺序节点
  >
  >  
  >
  > ZooDefs.Ids：权限对象

  + 删除zookeeper节点

  同理要先获得一个删除操作的Builder

  ```java
  // 指定要删除数据的版本号
  public BackgroundPathable<Void> withVersion(int version)
  // 确保数据被删除，本质上就是重试，当删除失败时重新发起删除操作
  public ChildrenDeletable guaranteed()
  // 指定删除的节点
  public Void forPath(String path) throws Exception
  // 递归删除子节点
  public BackgroundVersionable deletingChildrenIfNeeded()
  ```

  示例：

  ```java
  client.delete()
        .guaranteed()
        .withVersion(-1)
        .deletingChildrenIfNeeded()
        .forPath("/curatorTest/test1");
  ```

  + 读取zookeeper节点数据

  ```java
  // 将节点状态信息保存到stat
  public WatchPathable<byte[]> storingStatIn(Stat stat)
  // 指定节点路径
  public byte[] forPath(String path) throws Exception
  ```

  示例：

  ```java
  Stat test1Stat = new Stat();
  byte[] test1DataBytes = client.getData()
    														.storingStatIn(test1Stat).forPath("/curatorTest/test1");
  System.out.println("test1 data: " + new String(test1DataBytes));
  ```

  + 更新zookeeper节点数据

  ```java
  // 指定版本号
  public BackgroundPathAndBytesable<Stat> withVersion(int version)
  // 指定节点路径和要更新的数据
  public Stat forPath(String path, byte[] data) throws Exception
  ```

  示例：

  ```java
  Stat test1Stat = client.setData()
                      .withVersion(-1)
                      .forPath("/curatorTest/test1", "test1DataV2".getBytes());
  ```

  + 读取zookeeper子节点

  ```java
  // 把服务器端获取到的状态数据存储到stat对象中
  public WatchPathable<List<String>> storingStatIn(Stat stat)
  // 指定获取子节点数据的节点路径
  public List<String> forPath(String path) throws Exception
  // 设置watcher，类似于zookeeper本身的api，也只能使用一次
  public BackgroundPathable<List<String>> usingWatcher(Watcher watcher)
  public BackgroundPathable<List<String>> usingWatcher(CuratorWatcher watcher)
  ```

  示例：

  ```java
  Stat childStat = new Stat();
  List<String> childs = client.getChildren()
    													.storingStatIn(childStat).forPath("/curatorTest");
  ```

  + curator异步操作

  curator为所有操作都提供了异步执行的版本，只需在构建操作的方式链中添加如下操作之一即可：

  ```java
  public ErrorListenerPathable<List<String>> inBackground()
  public ErrorListenerPathable<List<String>> inBackground(Object context)
  public ErrorListenerPathable<List<String>> inBackground(BackgroundCallback callback)
  public ErrorListenerPathable<List<String>> inBackground(BackgroundCallback callback, Object context)
  public ErrorListenerPathable<List<String>> inBackground(BackgroundCallback callback, Executor executor)
  public ErrorListenerPathable<List<String>> inBackground(BackgroundCallback callback, Object context, Executor executor)
  ```

  示例：

  ```java
  client.delete()
                .guaranteed()
                .withVersion(-1)
                .inBackground(((client1, event) -> {
                          System.out.println(event.getPath() + ", data=" + event.getData());
                          System.out.println("event type=" + event.getType());
                          System.out.println("event code=" + event.getResultCode());
                 }))
                 .forPath("/curatorTest/test1");
  ```

  + curator中的NodeCache

  NodeCache会将某一路径的节点(`节点本身`)在本地缓存一份，当zk中相应路径的节点发生更新、创建或删除操作时，NodeCache将会得到响应，并且会将最新的数据拉到本地缓存中，`NodeCache只会监听路径本身的变化，并不会监听子节点的变化`。NodeCache构建函数：

  ```java
  public NodeCache(CuratorFramework client, String path)
  public NodeCache(CuratorFramework client, String path, boolean dataIsCompressed)
  ```

  参数说明：

  > - client：curator客户端
  > - path：需要缓存的节点路径
  > - dataIsCompressed：是否压缩节点下的数据

  可以通过NodeCache注册一个监听器来获取发生变化的通知。只要往容器中添加监听器，当节点发生变更时，容器中的监听器就会得到通知。

  ```java
  private final ListenerContainer<NodeCacheListener> listeners = new ListenerContainer<NodeCacheListener>();
  ```

  添加监听器示例：

  ```java
  NodeCache nodeCache = new NodeCache(client, "/curatorTest/test1");
  // 是否立即拉取/curatorTest/test1节点下的数据缓存到本地
  nodeCache.start(true);
  // 添加listener
  nodeCache.getListenable().addListener(() -> {
      ChildData childData = nodeCache.getCurrentData();
      if (null != childData) {
        	System.out.println("path=" + childData.getPath() + ", data=" + childData.getData() + ";");
      }
  });
  ```

  + curator中的PathChildrenCache

  PathChildrenCache会将指定路径节点下的所有子节点缓存在本地，但不会缓存节点本身的信息。当节点有新增(CHILD_ADDED)、删除(CHILD_REMOVED)、更新(CHILD_UPDATED)操作时，PathChildrenCache中的Listener将会得到通知，PathChildrenCache构造函数：

  ```java
  public PathChildrenCache(CuratorFramework client, String path, boolean cacheData)
  public PathChildrenCache(CuratorFramework client, String path, boolean cacheData, ThreadFactory threadFactory)
  public PathChildrenCache(CuratorFramework client, String path, boolean cacheData, boolean dataIsCompressed, ThreadFactory threadFactory)
  public PathChildrenCache(CuratorFramework client, String path, boolean cacheData, boolean dataIsCompressed, final ExecutorService executorService)
  public PathChildrenCache(CuratorFramework client, String path, boolean cacheData, boolean dataIsCompressed, final CloseableExecutorService executorService)
  ```

  参数说明：

  > client：curator客户端
  >
  > path：缓存节点路径
  >
  > cacheData：除了缓存节点状态外是否缓存节点数据。若为true，客户端在接收节点列表变更的同时，也能获取到节点数据内容，若为false，则无法获取数据内容
  >
  > threadFactory：线程池工厂，当内部需要开启新的线程执行时，使用该线程池工厂来创建线程
  >
  > dataIsCompressed：是否压缩节点数据
  >
  > executorService：线程池

  PathChildrenCache通过start方法可以传入三种启动模式：

  > StartMode.NORMAL：异步初始化cache
  >
  > StartMode.BUILD_INITIAL_CACHE：同步初始化cache，以及创建cache后就从服务器拉取对应的数据
  >
  > POST_INITIALIZED_EVENT：异步初始化cache，初始化完成触发PathChildrenCacheEvent.Type#INITIALIZED事件，cache中Listener会收到该事件的通知

  示例：

  ```java
  // 缓存子节点
  PathChildrenCache pathChildrenCache = new PathChildrenCache(client, "/curatorTest", true);
  // startMode为BUILD_INITIAL_CACHE，cache是初始化完成会发送INITIALIZED事件
  pathChildrenCache.start(PathChildrenCache.StartMode.BUILD_INITIAL_CACHE);
  System.out.println(pathChildrenCache.getCurrentData().size());
  pathChildrenCache.getListenable().addListener(((client1, event) -> {
      ChildData data = event.getData();
      switch (event.getType()) {
        case INITIALIZED:
          System.out.println("子节点cache初始化完成(StartMode为POST_INITIALIZED_EVENT的情况)");
          System.out.println("INITIALIZED: " + pathChildrenCache.getCurrentData().size());
          break;
        case CHILD_ADDED:
          System.out.println("添加子节点，path=" + data.getPath() + ", data=" + new String(data.getData()));
          break;
        case CHILD_UPDATED:
          System.out.println("更新子节点，path=" + data.getPath() + ", data=" + new String(data.getData()));
          break;
        case CHILD_REMOVED:
          System.out.println("删除子节点，path=" + data.getPath());
          break;
        default:
          System.out.println(event.getType());
      }
  }));
  ```

+ ### 完整示例代码

```java
public class ZKTest {
    public static void main(String[] args) throws Exception {
        //创建连接
//        CuratorFramework client = CuratorFrameworkFactory.newClient("192.168.226.130:2181", new RetryNTimes(3, 1000));
        CuratorFramework client = CuratorFrameworkFactory.builder()
                .connectString("192.168.226.130:2181")
                .sessionTimeoutMs(30000)
                .connectionTimeoutMs(15000)
                .retryPolicy(new RetryNTimes(3, 3000))
//                .namespace("curatorTest")
                .build();
        //启动zk
        client.start();

        //判断节点是否存在, 存在则删除
        Stat test1Stat = client.checkExists().forPath("/curatorTest/test1");
        if (test1Stat != null) {
            client.delete().guaranteed().deletingChildrenIfNeeded().withVersion(-1).forPath("/curatorTest/test1");
        }

        //创建节点
        client.create()
                .creatingParentsIfNeeded()
                .withMode(CreateMode.PERSISTENT)
                .withACL(ZooDefs.Ids.OPEN_ACL_UNSAFE)
                .forPath("/curatorTest/test1", "test1".getBytes());

        // 获取节点信息
        test1Stat = new Stat();
        byte[] test1DataBytes = client.getData().storingStatIn(test1Stat).forPath("/curatorTest/test1");
        System.out.println("test1 stat: " + test1Stat);
        System.out.println("test1 data: " + new String(test1DataBytes));

        // 更新节点数据
        test1Stat = client.setData()
                .withVersion(-1)
                .forPath("/curatorTest/test1", "test1DataV2".getBytes());
        System.out.println("test1 stat: " + test1Stat);

        // 获取所有子节点
        Stat childStat = new Stat();
        List<String> childs = client.getChildren().storingStatIn(childStat).forPath("/curatorTest");
        System.out.println("curatorTest childs: " + childs);

        //        client.delete()
        //                .guaranteed()
        //                .withVersion(-1)
        //                .inBackground(((client1, event) -> {
        //                    System.out.println(event.getPath() + ", data=" + event.getData());
        //                    System.out.println("event type=" + event.getType());
        //                    System.out.println("event code=" + event.getResultCode());
        //                }))
        //                .forPath("/curatorTest/test1");

        // 缓存节点
        NodeCache nodeCache = new NodeCache(client, "/curatorTest/test1");
        nodeCache.start(true);
        nodeCache.getListenable().addListener(() -> {
            System.out.println("NodeCache:");
            ChildData childData = nodeCache.getCurrentData();
            if (null != childData) {
                System.out.println("path=" + childData.getPath() + ", data=" + new String(childData.getData()) + ";");
            }
        });


        // 缓存子节点
        PathChildrenCache pathChildrenCache = new PathChildrenCache(client, "/curatorTest", true);
        // startMode为BUILD_INITIAL_CACHE，cache是初始化完成会发送INITIALIZED事件
        pathChildrenCache.start(PathChildrenCache.StartMode.BUILD_INITIAL_CACHE);
        System.out.println(pathChildrenCache.getCurrentData().size());
        pathChildrenCache.getListenable().addListener(((client1, event) -> {
            ChildData data = event.getData();
            switch (event.getType()) {
                case INITIALIZED:
                    System.out.println("子节点cache初始化完成(StartMode为POST_INITIALIZED_EVENT的情况)");
                    System.out.println("INITIALIZED: " + pathChildrenCache.getCurrentData().size());
                    break;
                case CHILD_ADDED:
                    System.out.println("添加子节点，path=" + data.getPath() + ", data=" + new String(data.getData()));
                    break;
                case CHILD_UPDATED:
                    System.out.println("更新子节点，path=" + data.getPath() + ", data=" + new String(data.getData()));
                    break;
                case CHILD_REMOVED:
                    System.out.println("删除子节点，path=" + data.getPath());
                    break;
                default:
                    System.out.println(event.getType());
            }
        }));

        Thread.sleep(20000000);
        client.close();
    }
}
```

