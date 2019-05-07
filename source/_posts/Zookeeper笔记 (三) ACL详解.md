title: Zookeeper笔记 (二) ACL详解
date: 2019-05-05 15:23:37
tags:
- zookeeper
- acl
categories:
- zookeeper
---
** {{ title }}：** <Excerpt in index | 首页摘要>
ACL权限控制
<!-- more -->
<The rest of contents | 余下全文>

本文参考地址: [【Zookeeper 学习笔记】—ACL权限控制](http://cmsblogs.com/?p=4101)

ACL权限控制是使用 schema : id : permission来标识。

1. schema(权限模式)：鉴权的策略
2. id(授权对象)
3. permission(权限)

其特性如下：

1. zookeeper的权限控制是基于每个znode节点的，需要对每个节点设置权限
2. 每个znode支持设置多种权限控制方案和多个权限
3. 子节点不会继承父节点的权限，客户端无权访问某节点，但可能可以访问它的子节点

+ ### schema

zk内置了一些权限控制方案：

| 方案   | 描述                                   |
| ------ | -------------------------------------- |
| world  | 只有一个用户：anyone，代表所有人(默认) |
| ip     | 使用ip地址认证                         |
| auth   | 使用已添加认证的用户认证               |
| digest | 使用"用户名：密码"方式认证             |

+ ### id

授权对象id是指，权限赋予的用户或者一个实体，例如：ip地址或机器。授权模式schema与授权对象id的关系：

| 权限模式 | 授权对象                                                     |
| -------- | :----------------------------------------------------------- |
| IP       | 一个ip地址或ip段，例如："192.168.226.130"或"192.168.226.1/24" |
| Digest   | 自定义，格式为："username:BASE64(SHA-1(username:password))"，例如："foo:dasdKJHkdaOsadOkda8cd=" |
| World    | 只有一个ID："anyone"                                         |
| Super    | 与Digest一致                                                 |

+ ### permission

| 权限   | ACL简写 | 描述                             |
| ------ | ------- | -------------------------------- |
| CREATE | c       | 可以创建子节点                   |
| DELETE | d       | 可以删除子节点(仅下一级节点)     |
| READ   | r       | 可以读取节点数据及显示子节点列表 |
| WRITE  | w       | 可以设置节点数据                 |
| ADMIN  | a       | 可以设置节点访问控制列表权限     |

+ ### 权限相关命令

| 命令    | 描述         |
| ------- | ------------ |
| getAcl  | 读取ACL权限  |
| setAcl  | 设置ACL权限  |
| addauth | 添加认证用户 |

+ ### 实战

  #### 1. world方案

> 设置方式
>
> setAcl <path> world:anyone:<acl>

```ruby
[zk: localhost:2181(CONNECTED) 3] create /node 1
Created /node

[zk: localhost:2181(CONNECTED) 5] getAcl /node
'world,'anyone # 默认方案
: cdrwa
# 设置权限
[zk: localhost:2181(CONNECTED) 6] setAcl /node world:anyone:cdrwa
cZxid = 0x3b
ctime = Sun May 05 00:41:07 PDT 2019
mZxid = 0x3b
mtime = Sun May 05 00:41:07 PDT 2019
pZxid = 0x3b
cversion = 0
dataVersion = 0
aclVersion = 1
ephemeralOwner = 0x0
dataLength = 1
numChildren = 0
```

  #### 				2. IP方案

> 设置方式
>
> setAcl <path> ip:<ip>:<acl>
>
> 其中ip可以使具体ip也可以是ip/bit格式，即ip转换为二进制，匹配前bit位，如192.168.0.0/16匹配192.168..

```ruby
[zk: localhost:2181(CONNECTED) 12] create /node1 1
Created /node1
[zk: localhost:2181(CONNECTED) 13] setAcl /node1 ip:192.168.226.130:cdrw
cZxid = 0x40
ctime = Sun May 05 00:57:14 PDT 2019
mZxid = 0x40
mtime = Sun May 05 00:57:14 PDT 2019
pZxid = 0x40
cversion = 0
dataVersion = 0
aclVersion = 1
ephemeralOwner = 0x0
dataLength = 1
numChildren = 0

# 使用IP非192.168.226.130的机器
[zk: localhost:2181(CONNECTED) 0] get /node1
Authentication is not valid : /node1 #没有权限

[zk: localhost:2181(CONNECTED) 1] delete /node1 #删除成功（因为设置DELETE权限仅对下一级子节点有效，并不包含此节点）
```

#### 		3. Auth方案

> 设置方式
>
> addauth digest <user>:<password> #添加认证用户
>
> setAcl <path> auth:<user>:<acl>

```ruby
[zk: localhost:2181(CONNECTED) 14] create /node2 1
Created /node2
[zk: localhost:2181(CONNECTED) 15] addauth digest sunny:562541
[zk: localhost:2181(CONNECTED) 16] setAcl /node2 auth:sunny:cdrwa
cZxid = 0x42
ctime = Sun May 05 01:03:16 PDT 2019
mZxid = 0x42
mtime = Sun May 05 01:03:16 PDT 2019
pZxid = 0x42
cversion = 0
dataVersion = 0
aclVersion = 1
ephemeralOwner = 0x0
dataLength = 1
numChildren = 0
[zk: localhost:2181(CONNECTED) 17] getAcl /node2
'digest,'sunny:cRA6w3JNGvZU4c1GD+jVPnif1dM=
: cdrwa
# 断开会话认证用户就没了
[zk: localhost:2181(CONNECTED) 0] getAcl /node2
Authentication is not valid : /node2
```

#### 		4. Digest方案

> 设置方式
>
> setAcl <path> digest:<user>:<password>:<acl>
>
> 这里的密码是经过SHA1及BASE64处理过的密文，在shell中可以通过如下命令计算：
>
> echo -n <user>:<password> | openssl dgst -binary -sha1 | openssl base64

```ruby
[root@localhost ~]# echo -n sunny:562541 | openssl dgst -binary -sha1 | openssl base64
cRA6w3JNGvZU4c1GD+jVPnif1dM=
```

```ruby
[zk: localhost:2181(CONNECTED) 2] create /node3 1
Created /node3
[zk: localhost:2181(CONNECTED) 3] setAcl /node3 digest:sunny:cRA6w3JNGvZU4c1GD+jVPnif1dM=:cdrwa
cZxid = 0x46
ctime = Sun May 05 01:10:06 PDT 2019
mZxid = 0x46
mtime = Sun May 05 01:10:06 PDT 2019
pZxid = 0x46
cversion = 0
dataVersion = 0
aclVersion = 1
ephemeralOwner = 0x0
dataLength = 1
numChildren = 0
[zk: localhost:2181(CONNECTED) 4] getAcl /node3
Authentication is not valid : /node3 # 没权限
[zk: localhost:2181(CONNECTED) 5] addauth digest sunny:562541 # 添加认证用户
[zk: localhost:2181(CONNECTED) 6] getAcl /node3 # 再次查询
'digest,'sunny:cRA6w3JNGvZU4c1GD+jVPnif1dM=
: cdrwa
```

