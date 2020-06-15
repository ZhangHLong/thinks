---
layout: post
title: Redis 过期策略&内存淘汰策略
categories: Redis 过期策略 内存淘汰策略
description: Redis 内存回收 内存淘汰策略
keywords: Redis、 过期策略、内存回收策略
---

## Redis 过期策略

采用定时删除+惰性删除方式

1. 定时删除

Redis 默认每秒进行10次过期扫描，采用贪心策略，而且增加了扫描时间范围，默认不会超过25ms。

大型Redis实例 ，假如所有KEY同一时间过期，Redis会持续扫描过期字典（循环多次），直至过期字典中过期的key变得稀疏，才会停止。
这势必会导致正常的业务读写请求出现卡顿现象。导致这种卡顿的另外一种原因是：内存管理器会频繁回收内存页，产生一定的CPU消耗。

2. 惰性删除

客户端请求访问某KEY,Redis 会对key过期时间做检测，过期即删除。

3. 主从问题

从节点不会进行定时扫描，所以从节点的处理也是被动处理。主节点key 过期时，会在aof文件追加一条 del 指令，用于同步到所有从节点
，从节点通过执行这条del命令进行删除过期key 。由于指令同步是异步而为，主节点设置的过期del 指令同步到从节点有一定的延迟，
那么从从节点获取的数据就会存在数据一致性问题。


## Redis 内存淘汰策略

当redis内存超出物理内存限制时，内存数据和磁盘产生频繁交换，这个过程会让redis 性能急剧下降。
Redis提供了配置参数<front color="red"> maxmemory </front> 来限制内存大小.
当实际内存超出maxmemory时，Redis提供了几种可选策略（maxmemory-policy）来让用户决定如何腾出空间继续提供服务。
1. <front color="red"> noeviction </front> :不会继续服务写请求（del 请求可以继续），读请求可以继续，默认淘汰策略。
2. <front color="red"> volatitle-lru </front> :尝试淘汰设置了过期时间的key，最少使用的key 优先被淘汰，持久化的数据不会删除。
3. <front color="red"> volatitle-ttl </front> :淘汰策略不是lru，而是比较key的剩余寿命ttl的值，ttl越小越优先被淘汰。
4. <front color="red">  volatitle-random  </front> :淘汰的key是过期中随机的key。
5. <front color="red">  allkeys-lru  </front> :区别于volatitle-lru ，这个策略要淘汰的key对象 是全体的key集合，而不只是过期key集合。
这就意味着没有设置过期时间的key 也会被删除
 
6. <front color="red">  allkeys-random  </front> :随机淘汰key

**volatitle-xxx**策略只会针对带有**过期时间**的key进行淘汰，**allkeys-xxx**策略会对所有的key进行淘汰。
如若redis只做缓存，可以使用**allkeys-xxx**。
如若redis做持久化使用，可以使用**volatitle-xxx**。
