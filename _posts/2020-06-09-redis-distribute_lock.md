---
layout: post
title: Redis 分布式锁
categories: Redis
description: Redis 分布式锁
keywords: Redis、 分布式锁、解决方案
---

# Redis 分布式锁

### 1.一般使用原理
可以理解为采用redis 的setnx方式进行加锁， 5个参数
jedis.set(lockKey, requestId, SET_IF_NOT_EXIST, SET_WITH_EXPIRE_TIME, expireTime);
```$xslt

    /**
     * 加锁
     * @param jedis 连接
     * @param key 
     * @param bizId 业务标示（解铃还须系铃人）
     * @param expireTime
     * @return
     */
   public static boolean getLock(Jedis jedis, String  key ,String bizId, Integer expireTime){
        SetParams setParams = new SetParams();
        // 过期时间
        setParams.ex(expireTime);
        //Only set the key if it does not already exist.
        setParams.nx();

        String result = jedis.set(key, bizId, setParams);

        if ("OK".equals(result)){
            return true;
        }
        return false;
    }


    /**
     * 锁释放
     * @param jedis
     * @param key
     * @param bizId 业务标示（解铃还须系铃人）
     * @return
     */
    public static boolean  unLock(Jedis jedis, String key, String bizId){
        String script = "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end";
        Object result = jedis.eval(script, Collections.singletonList(key), Collections.singletonList(bizId));

        if ("1".equals(result.toString())) {
            return true;
        }
        return false;
    }

```

### 2.集群环境下的锁问题及解决方案
1. 超时问题（锁过期） 
   A线程持有锁以后，业务逻辑花费的时间超出 锁过期时间，在业务尚未完成，B线程开始请求获取锁，由于锁已经过期，B自然获得锁，且B 执行也有一定的延迟，
    导致A 线程执行了释放锁，导致B 线程被解锁，依次循环C 会获取属于B 的锁。。。。D 获取C的锁。。。。导致整个系统或者业务 不具有安全稳定性。
  
   1. 所以一般redis 锁不会或者不建议使用在耗时比较长的业务中，除非你的业务允许少量的数据不一致问题，如果出现数据不一致问题，需要人工对账修复。
   2. 定时检查当前锁是否还在业务使用中，如果濒临过期，可以进行自动续命，增加过期时间来解决。  
   


2. 主从同步问题
  比如在Sentinel 集群环境中，主节点宕机异常，会自动选择从节点成为新的master。在此过程中，客户端请求在 Master-A上申请了锁key-lockA，锁还没有来及同步
  到Slave-B，Master-A宕机，然后Slave-B 选主变成了Master-B，新主节点没有key-lockA的数据，当另外一个客户端请求申请key 上锁时立刻就成功了，key-lockB
  所以会有两个客户端同时持有一个key 的两把锁。。。业务不可靠了。。。
  
  