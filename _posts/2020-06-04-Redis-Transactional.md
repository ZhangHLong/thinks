---
layout: post
title: Redis 事务机制
categories: Redis
description: Redis 事务原理
keywords: Redis, 事务
---

# 概述
redis 事务略有别于传统数据库的事务行为（begin、commit、rollback），redis事务指令：multi（开启事务 ）、exec（执行事务）、discard（丢弃事务）
### 1、Redis 事务原理
**原理**：
所有指令在执行exec 之前都是先发送到事务到缓存队列，服务器收到exec 指令才会开始执行事务队列，执行完毕一次性返回所有结果，因为单线程的原因，所以这个操作过程是"原子行为""

 <img src="/images/posts/redis/redis-transactional.jpg" alt="redis multi" />

**事务执行过程**：
```$xslt
IT-C02ZR1VTLVDL:src zhanghuilong$ redis-cli
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379> set key1 20
QUEUED
127.0.0.1:6379> INCR key1
QUEUED
127.0.0.1:6379> EXEC
1) OK
2) (integer) 21
127.0.0.1:6379>
```

### 2、Redis 原子性
**模拟事务内部异常操作**
```$xslt
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379> set key2 value2
QUEUED
127.0.0.1:6379> INCR key2
QUEUED
127.0.0.1:6379> set key3 1
QUEUED
127.0.0.1:6379> INCR key3
QUEUED
127.0.0.1:6379> exec
1) OK
2) (error) ERR value is not an integer or out of range
3) OK
4) (integer) 2
127.0.0.1:6379>

```
"INCR key2" 字符串自增异常  (error) ERR value is not an integer or out of range
并不影响key3 的自增
所以，redis 的事务是**非原子性**的

### 3、Watch 乐观锁
Watch要用在multi 之前
**范例**：
```$Java
 public static void main(String[] args) throws InterruptedException {
        String KEY = "zhanghuilong";
        CountDownLatch countDownLatch = new CountDownLatch(1);
        Jedis jedis = new Jedis("127.0.0.1", 6379);
        jedis.set(KEY,"29");

        // 模拟在Thread2开启事务操作前，对KEY 做数据变更操作
        new Thread(()->{
            try {
                TimeUnit.SECONDS.sleep(3);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            jedis.incr(KEY);
            countDownLatch.countDown();
            System.out.println("===countDown====");
        },"Thread1").start();


        // Thread2 采用乐观锁 watch机制进行执行事务 
        new Thread(()->{
            // 乐观锁，监听key 是否有变化，如果有，事务执行失败
            while (true){
                String watch = jedis.watch(KEY);
                System.out.println("watch key:"+ watch);
                try {
                    System.out.println("===await====");
                    countDownLatch.await();

                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                // 1开启事务
                Transaction multi = jedis.multi();

                // 2放入缓存队列
                multi.decr(KEY);
                multi.decr(KEY);
                // 3真正执行，取决于watch 变量是否变化
                List<Object> exec = multi.exec();
                if (exec !=null){
                    System.out.println(" success,exec result : "+ JSON.toJSONString(exec));
                    break;
                }else {
                    System.out.println(" fail,exec result : "+ JSON.toJSONString(exec));
                }

            }

            String zhl = jedis.get(KEY);
            System.out.println(zhl);

        },"Thread2").start();
    }
```

**输出**：
```$xslt
watch key:OK
===await====
===countDown====
watch key:OK
 fail,exec result : null
watch key:OK
===await====
watch key:OK
 success,exec result : [29,28]
28
```