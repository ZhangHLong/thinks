---
layout: post
title: RPC 调用耗时分析
categories: RPC
description: 一次RPC调用耗时分析 
keywords: RPC、 TP、CUP、FULL GC
---

## 一次 RPC 调用时间都去那里了

<img src="/images/posts/rpc/rpc-time-cost.png" />

**整体看耗时点分为：5部分 **

<img src="/images/posts/rpc/rpc-sum.png"   />

1、 消费方业务执行逻辑耗时（业务逻辑+JVM GC）
2、 消费方调用服务提供方，RPC框架序列化（大对象耗时） 、发起网络请求（网络抖动）
3、 服务调用者接收请求，内部逻辑执行
4、 服务提供者发起网络请求，返回执行结果，RPC框架进行序列化结果返回（序列化耗时）
5、 消费方接受接口响应结果，RPC 框架反序列化 提供者结果（（期间可能存在 失败超时重试等耗时情况））


