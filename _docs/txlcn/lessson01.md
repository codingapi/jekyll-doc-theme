---
title:  分布式事务从0到1-认识分布式事务
permalink: /docs/txlcn-lessson01/
---

![](/img/WX20191220-102719.png)

本节课讲解的主要内容是先介绍与分布式事务相关的一些理论   
ACID 隔离级别 spring事务传播行为 乐观锁悲观锁 BASE理论 ACP理论 拜占庭将军问题 共识算法 

### 前言
随着分布式技术的发展，分布式事务问题越来越显著。如何解决分布式事务问题变得非常紧迫。当目前业界较为成熟的分布式事务框架尚不存在，且又因为分布式事务相关的实现原理介绍不够完善，就让我们从今天开始结合原理与实践一起动手去解决分布式事务问题吧。

#### ACID理论
在解决分布式事务这个问题之前，我们先来回顾一下什么事务，先理解事务的本质,先来看看本地事务的ACID理论。


* 原子性(Atomicity)
原子性是指事务是一个不可分割的工作单位，事务中的操作要么都发生，要么都不发生。

* 一致性(Consistency)
一致性是指在事务开始之前和事务结束以后，数据库的完整性约束没有被破坏。这是说数据库事务不能破坏关系数据的完整性以及业务逻辑上的一致性。

* 隔离性(Isolation)
事务的隔离性是多个用户并发访问数据库时，数据库为每一个用户开启的事务，不能被其他事务的操作数据所干扰，多个并发事务之间要相互隔离

* 持久性Durability）
这是最好理解的一个特性：持久性，意味着在事务完成以后，该事务所对数据库所作的更改便持久的保存在数据库之中，并不会被回滚。（完成的事务是系统永久的部分，对系统的影响是永久性的，该修改即使出现致命的系统故障也将一直保持）

#### 数据库的隔离级别

- √为会发生，×为不会发生：  

|  隔离级别 | 脏读 |  不可重复读 |  幻读  |
|  :----  | :----  | :----  | :----  |
| read uncommitted（未提交读）  |  √ | √ | √ |
| read committed（提交读）  | x |√ | √ |
| repeatable read（可重复读  | x | x | √ |
| serialization（可串行化）  | x | x| x |

再总结mysql 的常用命令（下面会用到）：   
查看MySQL隔离级别: `SELECT @@tx_isolation`   
会话层面设置隔离级别: `set session transaction isolation level 隔离级别`  
开启事务: `start transaction`  
提交事务:`commit`  
回滚事务:`rollback`  


##### 脏读演示
表中的数据如下，设置隔离级别为未提交读  

|  id  | name  | balacne  | 
|  ----  | ----  | ----  |
| 1  |  张三 | 1000 |
| 2  |  李四 | 0 |

执行流程

<table>
<colgroup>
<col width="10%" />
<col width="45%" />
<col width="45%" />
</colgroup>
<thead>
<tr class="header">
<th>时间</th>
<th>客户端A</th>
<th>客户端B</th>
</tr>
</thead>
<tbody>
<tr>
<td markdown="span">T1</td>
<td markdown="span"> 
set session transaction isolation level read uncommitted;start transaction(开启事务);
update account set balance = balance+1000 where id = 1;
select * from account where id = 1;
设置为未提交读，给张三账号+1000，输出为2000 
</td>
<td markdown="span">

</td>
</tr>
<tr>
<td markdown="span">T2</td>
<td markdown="span"> 

</td>
<td markdown="span">
set session transaction isolation level read uncommitted;
start transaction;
select * from account where id = 1;
查询余额输出为2000
</td>
</tr>

<tr>
<td markdown="span">T3</td>
<td markdown="span"> 
rollback
</td>
<td markdown="span">

</td>
</tr>

<tr>
<td markdown="span">T4</td>
<td markdown="span"> 

</td>
<td markdown="span">
commit
</td>
</tr>

<tr>
<td markdown="span">T5</td>
<td markdown="span"> 

</td>
<td markdown="span">
select * from account where id = 1;
查询余额输出为1000
</td>
</tr>

</tbody>
</table>


再举一个例子

<table>
<colgroup>
<col width="10%" />
<col width="45%" />
<col width="45%" />
</colgroup>
<thead>
<tr class="header">
<th>时间</th>
<th>客户端A</th>
<th>客户端B</th>
</tr>
</thead>
<tbody>
<tr>
<td markdown="span">T1</td>
<td markdown="span"> 
set session transaction isolation level read uncommitted;
start transaction;
update account set balance = balance-1000 where id = 1;
update account set balance = balance+1000 where id = 2;
</td>
<td markdown="span">

</td>
</tr>
<tr>
<td markdown="span">T2</td>
<td markdown="span"> 

</td>
<td markdown="span">
set session transaction isolation level read uncommitted;
start transaction;
select balance from account where id = 2;
update account set balance = balance -1000 where id = 2;
更新语句被阻塞
</td>
</tr>

<tr>
<td markdown="span">T3</td>
<td markdown="span"> 
rollback
</td>
<td markdown="span">

</td>
</tr>

<tr>
<td markdown="span">T4</td>
<td markdown="span"> 

</td>
<td markdown="span">
commit
</td>
</tr>

</tbody>
</table>
执行完成，数据库中的数据如下 


|  id  | name  | balacne  | 
|  ----  | ----  | ----  |
| 1  |  张三 | 1000 |
| 2  |  李四 | -1000 |

解释如下:   
T1:	1给2转账1000   
T2:	2的余额够1000，购买1000元商品，更新语句被阻塞  
T3:	1回滚，1的余额变成1000,2的余额变成0  
T4:	2成功扣款，余额为0-1000=-1000  


##### 不可重复读演示
表中的数据如下，设置隔离级别为提交读 

数据库中的数据如下 

|  id  | name  | balacne  | 
|  ----  | ----  | ----  |
| 1  |  张三 | 1000 |
| 2  |  李四 | 0 |



执行流程

<table>
<colgroup>
<col width="10%" />
<col width="45%" />
<col width="45%" />
</colgroup>
<thead>
<tr class="header">
<th>时间</th>
<th>客户端A</th>
<th>客户端B</th>
</tr>
</thead>
<tbody>
<tr>
<td markdown="span">T1</td>
<td markdown="span"> 
set session transaction isolation level read committed;
start transaction;
select * from account where id = 2;
查询出余额输出为0;
</td>
<td markdown="span">

</td>
</tr>
<tr>
<td markdown="span">T2</td>
<td markdown="span"> 

</td>
<td markdown="span">
set session transaction isolation level read committed;
start transaction;
update account set balance = balance + 1000 where id = 2;
select * from account where id = 2;
commit；
查询余额输出1000
</td>
</tr>

<tr>
<td markdown="span">T3</td>
<td markdown="span"> 
select * from account where id = 2;
commit；
查询余额输出1000
</td>
<td markdown="span">

</td>
</tr>
</tbody>
</table>


现在用上面的例子看一下可重复读是个什么过程？


<table>
<colgroup>
<col width="10%" />
<col width="45%" />
<col width="45%" />
</colgroup>
<thead>
<tr class="header">
<th>时间</th>
<th>客户端A</th>
<th>客户端B</th>
</tr>
</thead>
<tbody>
<tr>
<td markdown="span">T1</td>
<td markdown="span"> 
set session transaction isolation level repeatable read;
start transaction;
select * from account where id = 2;
查询出余额输出为0;
</td>
<td markdown="span">

</td>
</tr>
<tr>
<td markdown="span">T2</td>
<td markdown="span"> 

</td>
<td markdown="span">
set session transaction isolation level repeatable read;
start transaction;
update account set balance = balance + 1000 where id = 2;
select * from account where id = 2;
commit；
查询余额输出1000
</td>
</tr>

<tr>
<td markdown="span">T3</td>
<td markdown="span"> 
select * from account where id = 2;
commit；
查询余额输出0
</td>
<td markdown="span">

</td>
</tr>
</tbody>
</table>

仔细看这个例子和上面的例子在T3时间段的输出，理解了什么叫可重复读了吧？当我们将当前会话的隔离级别设置为可重复读的时候，当前会话可以重复读，就是每次读取的结果集都相同，而不管其他事务有没有提交。
----------但是在可重复读的隔离级别上，会产生幻读的问题。

##### 幻读演示

数据库中的数据如下 

|  id  | name  | balacne  | 
|  ----  | ----  | ----  |
| 1  |  张三 | 1000 |
| 2  |  李四 | 0 |

先上一段《高性能MySQL》对于幻读的解释   
- 所谓幻读，指的是当某个事务在读取某个范围内的记录时，另外一个事务又在该范围内插入了新的记录，当之前的事务再次读取该范围的记录时，会产生幻行。InnoDB存储引擎通过多版本并发控制（MVCC）解决了幻读的问题。   

用大白话解释一下，就是事务1查询id<10的记录时，返回了2条记录，接着事务2插入了一条id为3的记录，并提交。接着事务1查询id<10的记录时，返回了3条记录，说好的可重复读呢？结果却多了一条数据。

MySQL通过MVCC解决了这种情况下的幻读，我们可以验证一下


<table>
<colgroup>
<col width="10%" />
<col width="45%" />
<col width="45%" />
</colgroup>
<thead>
<tr class="header">
<th>时间</th>
<th>客户端A</th>
<th>客户端B</th>
</tr>
</thead>
<tbody>
<tr>
<td markdown="span">T1</td>
<td markdown="span"> 
set session transaction isolation level repeatable read;
start transaction;
select count(*) from account where id <=10;
输出2;
</td>
<td markdown="span">

</td>
</tr>
<tr>
<td markdown="span">T2</td>
<td markdown="span"> 

</td>
<td markdown="span">
set session transaction isolation level repeatable read;
start transaction;
insert into account(id,name,balance) values(“3”,“王五”,“0”) ;
select count(*) from account where id <=10;
commit；
输出3
</td>
</tr>

<tr>
<td markdown="span">T3</td>
<td markdown="span"> 
select count(*) from account where id <=10;
commit；
输出2
</td>
<td markdown="span">

</td>
</tr>
</tbody>
</table>

这种情况下的幻读被解决了，再举一例，表中的数据如下


|  id  | name  | balacne  | 
|  ----  | ----  | ----  |
| 1  |  张三 | 1000 |
| 2  |  李四 | 0 |

<table>
<colgroup>
<col width="10%" />
<col width="45%" />
<col width="45%" />
</colgroup>
<thead>
<tr class="header">
<th>时间</th>
<th>客户端A</th>
<th>客户端B</th>
</tr>
</thead>
<tbody>
<tr>
<td markdown="span">T1</td>
<td markdown="span"> 
set session transaction isolation level repeatable read;
start transaction;
select count(*) from account where id =3;
输出0;
</td>
<td markdown="span">

</td>
</tr>
<tr>
<td markdown="span">T2</td>
<td markdown="span"> 

</td>
<td markdown="span">
set session transaction isolation level repeatable read;
start transaction;
insert into account(id,name,balance) values(“3”,“王五”,“0”) ;
commit;
</td>
</tr>

<tr>
<td markdown="span">T3</td>
<td markdown="span"> 
insert into account(id,name,balance) values(“3”,“王五”,“0”);
主键重复，插入失败
</td>
<td markdown="span">

</td>
</tr>


<tr>
<td markdown="span">T4</td>
<td markdown="span"> 
select count(*) from account where id =3;
输出0;
</td>
<td markdown="span">

</td>
</tr>

<tr>
<td markdown="span">T5</td>
<td markdown="span"> 
rollback;
</td>
<td markdown="span">

</td>
</tr>


</tbody>
</table>

select 某记录是否存在，不存在，准备插入此记录，但执行 insert 时发现此记录已存在，无法插入，这个就有问题了。
很多人容易搞混不可重复读和幻读，确实这两者有些相似。但不可重复读重点在于update和delete，而幻读的重点在于insert。

注意：不可重复读和幻读的区别是：前者是指读到了已经提交的事务的更改数据（修改或删除），后者是指读到了其他已经提交事务的新增数据。
对于这两种问题解决采用不同的办法，防止读到更改数据，只需对操作的数据添加行级锁，防止操作中的数据发生变化；而防止读到新增数据，往往需要添加表级锁，将整张表锁定，防止新增数据（oracle采用多版本数据的方式实现）。

当隔离级别设置为可串行化，强制事务串行执行，避免了前面说的幻读的问题。





##### Spring事务的七种传播行为

|  事务传播行为类型  | 说明  | 
|  ----  | ----  |
| PROPAGATION_REQUIRED  |  如果当前没有事务，就新建一个事务，如果已经存在一个事务中，加入到这个事务中。这是最常见的选择。 | 
| PROPAGATION_SUPPORTS  |  支持当前事务，如果当前没有事务，就以非事务方式执行。 | 
| PROPAGATION_MANDATORY  |  使用当前的事务，如果当前没有事务，就抛出异常。 | 
| PROPAGATION_REQUIRES_NEW  |  新建事务，如果当前存在事务，把当前事务挂起。 | 
| PROPAGATION_NOT_SUPPORTED  |  以非事务方式执行操作，如果当前存在事务，就把当前事务挂起。 | 
| PROPAGATION_NEVER  |  以非事务方式执行，如果当前存在事务，则抛出异常。 | 
| PROPAGATION_NESTED  |  如果当前存在事务，则在嵌套事务内执行。如果当前没有事务，则执行与PROPAGATION_REQUIRED类似的操作。 | 


spring隔离级别与事务传播行为控制    
`@Transactional(propagation = Propagation.NESTED,isolation = Isolation.READ_UNCOMMITTED)`


##### 乐观锁与悲观锁

悲观锁(Pessimistic Lock)

顾名思义，就是很悲观，每次去拿数据的时候都认为别人会修改，所以每次在拿数据的时候都会上锁，这样别人想拿这个数据就会block直到它拿到锁。传统的关系型数据库里边就用到了很多这种锁机制，比如行锁，表锁等，读锁，写锁等，都是在做操作之前先上锁。它指的是对数据被外界（包括本系统当前的其他事务，以及来自外部系统的事务处理）修改持保守态度，因此，在整个数据处理过程中，将数据处于锁定状态。悲观锁的实现，往往依靠数据库提供的锁机制（也只有数据库层提供的锁机制才能真正保证数据访问的排他性，否则，即使在本系统中实现了加锁机制，也无法保证外部系统不会修改数据）。

乐观锁(Optimistic Lock)

顾名思义，就是很乐观，每次去拿数据的时候都认为别人不会修改，所以不会上锁，但是在更新的时候会判断一下在此期间别人有没有去更新这个数据，可以使用版本号等机制。乐观锁适用于多读的应用类型，这样可以提高吞吐量，像数据库如果提供类似于write_condition机制的其实都是提供的乐观锁。


两种锁各有优缺点，不可认为一种好于另一种，像乐观锁适用于写比较少的情况下，即冲突真的很少发生的时候，这样可以省去了锁的开销，加大了系统的整个吞吐量。但如果经常产生冲突，上层应用会不断的进行retry，这样反倒是降低了性能，所以这种情况下用悲观锁就比较合适。


MySQL select…for update的Row Lock与Table Lock
使用select…for update会把数据给锁住，不过我们需要注意一些锁的级别，MySQL InnoDB默认Row-Level Lock，所以只有「明确」地指定主键，MySQL 才会执行Row lock (只锁住被选取的数据) ，否则MySQL 将会执行Table Lock (将整个数据表单给锁住)。
```
-- (若主键1中存在数据，那么触发的就是RowLock) 
select * from user where id = 1 for update 
-- (触发的就是TableLock)   
select * from user where name like = '%小明%' for update 
``` 

##### BASE理论
BASE理论
    BASE是Basically Available（基本可用）、Soft state（软状态）和 Eventually consistent（最终一致性）三个短语的缩写。BASE理论是对CAP中一致性和可用性权衡的结果，其来源于对大规模互联网系统分布式实践的总结，是基于CAP定理逐步演化而来的。BASE理论的核心思想是：即使无法做到强一致性，但每个应用都可以根据自身业务特点，采用适当的方式来使系统达到最终一致性。

基本可用
    基本可用是指分布式系统在出现不可预知故障的时候，允许损失部分可用性----注意，这绝不等价于系统不可用。比如：

（1）响应时间上的损失。正常情况下，一个在线搜索引擎需要在0.5秒之内返回给用户相应的查询结果，但由于出现故障，查询结果的响应时间增加了1~2秒

（2）系统功能上的损失：正常情况下，在一个电子商务网站上进行购物的时候，消费者几乎能够顺利完成每一笔订单，但是在一些节日大促购物高峰的时候，由于消费者的购物行为激增，为了保护购物系统的稳定性，部分消费者可能会被引导到一个降级页面

软状态
    软状态指允许系统中的数据存在中间状态，并认为该中间状态的存在不会影响系统的整体可用性，即允许系统在不同节点的数据副本之间进行数据同步的过程存在延时

最终一致性
    最终一致性强调的是所有的数据副本，在经过一段时间的同步之后，最终都能够达到一个一致的状态。因此，最终一致性的本质是需要系统保证最终数据能够达到一致，而不需要实时保证系统数据的强一致性。


##### CAP定律

这个定理的内容是指的是在一个分布式系统中、Consistency（一致性）、 Availability（可用性）、Partition tolerance（分区容错性），三者不可得兼。

一致性（C）
    在分布式系统中的所有数据备份，在同一时刻是否同样的值。（等同于所有节点访问同一份最新的数据副本）

可用性（A）
    在集群中一部分节点故障后，集群整体是否还能响应客户端的读写请求。（对数据更新具备高可用性）

分区容错性（P）
    以实际效果而言，分区相当于对通信的时限要求。系统如果不能在时限内达成数据一致性，就意味着发生了分区的情况，必须就当前操作在C和A之间做出选择。

##### 拜占庭将军问题

![](/img/txlcn/950B4878-0B07-4233-A5AA-F6E28A036A6E.png)

##### 共识机制算法介绍 
https://www.cnblogs.com/davidwang456/articles/9001331.html


<div align="center"><img src="/img/qrcode330.jpg" style="width:300px;" /></div>

本文参考   
快速理解脏读，不可重复读，幻读 : https://blog.csdn.net/Vincent2014Linux/article/details/89669762   
Spring事务传播行为：https://blog.csdn.net/xzz1173724284/article/details/88954730   
乐观锁和悲观锁的区别: https://blog.csdn.net/coderDogg/article/details/85093741    
mysql(for update)悲观锁总结与实践: https://blog.csdn.net/zmx729618/article/details/52701972/