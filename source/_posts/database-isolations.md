---
title: MySQL数据库隔离级别
date: 2019-04-08 15:57:59
tags:
- mysql
- acid
---

ACID作为数据库（事务）特性大家熟知。今天想认真总结一下 I - isolation 隔离性。MySQL一共有以下四种隔离级别

#### Read Uncommitted ####

这是最弱的隔离级别——可以理解为没有任何隔离。情况发生在：
* 事务A在更新数据，比如 update count from 10 to 20
* 事务B在A刚刚更新的时候读入了数据，读入 count = 20
* 事务A在提交事务过程中出错，数据回滚，count is back to 10

这样就造成了脏读 dirty read


#### Read Committed ####

为了避免上述情况，可以让B只能读入已经commit过的数据。这样如果事务A还没有commit，它涉及到的数据变化B是看不到的。这种隔离方式有效避免了脏读。

然后又有情况发生了：
* 事务A需要多次读入同一数据。第一次读count=10
* 事务B更新 count from 10 to 20
* 事务A继续读，发现count变成20了。数据不能一致。

这就造成了不可重复读 non-repeatable read


#### Repeatable Read ####

为了避免上述情况，让A在commit之前，读入的数据（涉及到的行row）都是相同的数据快照(snapshot)（第一次select之后，创建快照，之后select都来自这个快照），这样就隔离了其他事务对涉及数据的任何操作。这个隔离级别已经相对较高了，也是MySQL默认使用的隔离级别。相对来说开销也会比较大，性能会降低。

然而还是有情况发生：
* 事务A在commit前读入的行数据本身再次read是一致的
* 事务B insert 了一条数据 count = 10
* 事务A批量更新了所选中的行数据（第一次读入的数据快照），set all counts to 20，commit后事务结束
* 事务A结束后，发现count不仅多了一行，还出现了10这个已经改掉的值

这就造成了幻读 phantom read


#### Serializable ####

终极办法，完全隔离各个事务之间的影响。基于 Repeatable Read 的基础之上，凡是涉及到的数据，在本事务commit之前，其他事务都不能对相关数据做操作，知道本事务结束。

这是最高隔离级别，开销也最大。


#### 参考文献 ####
https://mydbops.wordpress.com/2018/06/22/back-to-basics-isolation-levels-in-mysql/
