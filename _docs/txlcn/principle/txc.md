---
title: 原理介绍|TXC事务模式
permalink: /docs/txlcn-principle-txc/
---

## 一、原理介绍：    
&nbsp;&nbsp;&nbsp;&nbsp;TXC模式命名来源于淘宝，实现原理是在执行SQL之前，先查询SQL的影响数据，然后保存执行的SQL快走信息和创建锁。当需要回滚的时候就采用这些记录数据回滚数据库，目前锁实现依赖redis分布式锁控制。

## 二、模式特点:
* 该模式同样对代码的嵌入性低。
* 该模式仅限于对支持SQL方式的模块支持。
* 该模式由于每次执行SQL之前需要先查询影响数据，因此相比LCN模式消耗资源与时间要多。
* 该模式不会占用数据库的连接资源。
 