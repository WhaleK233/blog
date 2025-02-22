---
title: Redis相关问题及解决方案
date: 2025-02-18 21:39:59
tags: 
---

# Redis相关问题及解决方案

## 缓存穿透
**问题描述**

当出现大量对某个不存在的key的访问时，由于存储层查不到数据而不会写入缓存，会一直在数据库中进行查询，导致数据库承受很大的查询压力。可能是遭到了攻击。

**解决方案**

可以采用布隆过滤器解决
布隆过滤器使用bitmap存储数据是否在数据库中存在
先初始化一个大数组，里面存放0和1，一开始都是0，当查询到某个key的时候，对其经过3次哈希计算，将模数组长度的下标的数组的值由0改为1。这样3个位置可以标明一个key的存在。但是有误判的概率，不过可以接受。

## 缓存击穿
**问题描述**

对于某个设置了过期时间的key，在其过期的时间点恰好有对这个key有大量的并发请求。由于缓存过期，会全部转发到数据库，导致数据库承受巨大压力。

**解决方案**

方案一：采用互斥锁
当缓存失效时，先使用Redis的setnx设置互斥锁，再查询数据库并写入缓存，拿不到锁就重复尝试读取缓存。

方案二：逻辑过期
在存入缓存时，对key添加一个过期时间戳，不设置过期时间。当查询到key的时候，检测其是否过期。如果过期，先放回过期数据。并单开一个新线程进行读取数据库写入缓存操作。不过数据不能保持强一致，但确保了高可用性。

## 缓存雪崩
**问题描述**

大量同时过期的key，在过期的时间点出现大量并发请求。请求全部转发到数据库，数据库承受巨大压力而雪崩。

**解决方案**

可以在设置key的过期时间时，添加随机的偏移量，使他们不会同时过期。

---
*以上三种问题均可以通过使用Nginx或者Gateway限流的方式解决，可以作为最后的保底解决方案*

## 双写一致性
**问题描述**

需要让数据库和Redis缓存的数据保持一致

**解决方案**

强一致：
使用Redisson的读写锁，读锁采用共享锁
更新数据时，采用setnx的排他锁，使其他操作均与其互斥，保持数据一致性。

非强一致：延迟双删
更新数据时，先删除缓存，更新数据库，再延时删除缓存。延时不好确定，且可能导致短时间的数据不一致。

也可以更新数据时将消息发送给MQ，再监听MQ的消息更新缓存。
或者使用Canal，Canal会伪装成数据库的一个从节点，数据更新时，Canal读取binlog数据，再更新缓存。

## 持久化
使用RDB和AOF实现数据持久化

RDB是一个快照文件，将Redis由内存存储的数据写入到磁盘。当Redis恢复数据时，可以从RDB的快照文件中恢复。

AOF是追加文件。当Redis执行写命令时会存储日志到这个文件中。需要恢复数据时，会从这个文件中再执行一遍所有命令。AOF文件中可以设置刷盘策略，可以设置在什么情况下写入一次命令。

RDB体积小，恢复快，但有可能会丢失数据。AOF数据恢复慢，但丢数据风险小。

## 数据过期策略
- 惰性删除：设置过期事件后，仅在访问到该key时判断是否过期，如果过期就执行删除操作。
- 定期删除：每隔一段时间检查是否有过期的key并进行删除。定期删除有两种模式：SLOW模式下是定时任务可以修改执行频率和操作时间；FAST模式是高频执行。

一般采用两种方式配合使用

## 数据淘汰策略
Redis有很多种数据淘汰策略：
- 不删除数据
- 随机删除数据
- 删除过期时间最小的数据
- 随机删除有过期时间的数据
- 基于LRU或LFU算法的随机删除数据
- 基于LRU或LFU算法的随机删除有过期时间的数据

具体根据业务需求进行选择，一般使用基于LRU算法的随机删除数据allkeys-lru，也可以保证redis中保留的都是热点数据。

## 分布式锁
Redis的一个框架Redission很好的实现了分布式锁。

Redisson的锁可以控制失效时间和等待时间，并且有看门狗机制来延长执行时间较长的锁的持续时间。在高并发场景下也有很高的性能。

Redisson的分布式锁是可以重入的，锁内会记录当前线程的id和重入次数，避免死锁出现。

但是Redisson的分布式锁无法解决Redis的主从强一致性的问题。不过也可以使用红锁来解决，但性能很差，运维成本高。

如果需要保证主从强一致性，可以使用ZooKeeper的分布式锁。

## 集群方案
- 主从复制
- 哨兵模式
- 分片集群

**主从同步**

单节点Redis的并发能力有限，搭建主从集群可提高Redis的并发能力。

一般是一主多从，主节点负责写数据，从节点负责读数据，主节点负责把新数据同步到从节点。

1. 从节点请求主节点同步数据，携带自己的replication id和offset
2. 主节点根据replication id是否不同判断是否第一次请求。如果是，主节点将replication和offset发送给从节点，保持信息一致
3. 主节点执行bgsave，生成RDB文件发送给从节点载入RDB文件
4. 在此期间新增的信息会保存到某个日志文件中，发送给从节点进行增量同步

**哨兵模式**

搭建主从集群并使用哨兵模式可以保证Redis的高并发高可用。

哨兵模式可以实现主从集群的自动故障恢复，哨兵负责监控节点状态，恢复故障和通知。

如果主节点故障，哨兵会offset值最大的从节点升级为主节点，同时在故障转移时也会将最新信息推送给Redis的客户端。

**集群脑裂**

有时候由于网络原因会出现脑裂的情况。指主节点和从节点处于不同的网络分区时，哨兵感受不到主节点，会将一个从节点升级为主节点，这样会同时存在两个可用的主节点，无法同步旧主节点的数据。

可以通过设置最少从节点数量和主从数据同步的延迟时间来解决，避免大量数据丢失.

**分片集群**

当需要在Redis中缓存大量数据时，采用分片集群，集群中会存在多个主节点，每个主节点保存不同数据，并且可以给每个主节点设置多个从节点。

每个主节点之间也会通过ping检测彼此的状态。当客户端请求访问集群任意节点时，最终都会被路由转发到正确的节点。

Redis集群引入了哈希槽的概念，一共有16384个哈希槽，分配给每个集群。key通过CRC16校验后对16384取模来得到自己的位置进行存储。

## 单线程，但速度快
**原因**

1. 存储在内存中，由C语言编写
2. 单线程避免不必要的上下文切换和竞争条件
3. 使用多路I/O复用模型，非阻塞IO

**多路I/O复用模型**

利用单线程来同时监听多个Socket，在某个Socket可读或可写时获得通知，避免无效等待时间，充分利用CPU资源。

I/O多路复用采用epoll模式实现，在通知用户Socket准备就绪时将就绪的Socket写入用户空间，不需要挨个遍历，提高性能。

Redis的网络模型采用I/O多路复用模型结合事件的处理器来应对多个Socket请求.

例如：
- 连接应答处理器
- 命令回复处理器
- 命令请求处理器

Redis 6.0之后，使用多线程来处理回复事件，在命令转换中使用了多线程，进一步提高性能。