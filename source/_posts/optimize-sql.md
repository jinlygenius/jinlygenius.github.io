---
title: SQL 调优
date: 2019-04-05 11:50:02
tags:
- sql
- mysql
---


这两天公司一直在招人，CTO经常问面试者的问题：怎么sql调优？之后大家讨论，感觉sql调优的方法太多了，这里总结一下


因为我们主要用MySQL（InnoDB），先砸一下MySQL的官方文档

https://dev.mysql.com/doc/refman/8.0/en/optimization.html


#### 1. 索引相关 ####

* 索引是面试者最容易想到的答案。我们都知道，字段加了索引以后，在 where, join, order by 字段的时候都会变得非常快。

* 如果where条件后过滤的是 is null
```sql
select id from mytable where count is null
```

这个时候即使字段加了索引，搜索 null 值会让数据库引擎不使用索引。所以例子中尽量填充 count=0 ，不要留 null 值。上面例子将变成
```sql
select id from mytable where count = 0
```
* 尽量不要用 <> 或 != ，因为这些也会导致数据库引擎不使用索引，使用全表扫描
* 同样的原因，选择一个范围的时候，能用 between 尽量都用 between
* 用 exists 代替 in ，比如
```sql
select id, number from sparrow_test_0903.sparrow_orders_order
where id in
(
    select order_id from sparrow_test_0903.sparrow_orders_aftersale
)
;
```
最好改成
```sql
select o.id, o.number from sparrow_test_0903.sparrow_orders_order o
where exists
(
    select 1 from sparrow_test_0903.sparrow_orders_aftersale a
    where a.order_id=o.id
)
;
```
同理，用 not exists 代替 not in
* 关于 like，如果全模糊查找 "%test%" 肯定是要全表扫描的。如果只有右模糊查找 "test%" 是可以使用索引的。而左模糊索引就用不了了
* 其实也是同样的原理，（对MySQL InnoDB）不要
```sql
select count(*) from mytable
```
要达成同样的目的一般使用
```sql
select count(1) from mytable
```
因为一般第一列默认都是主键，都有索引。这里补充一下，如果
```sql
select count(id) from mytable
```
会过滤掉是null的值，速度会变慢很多
* 不要对过滤字段本身做计算，比如
```sql
select id from mytable where count/2=10
```
这样会导致不用索引。应该改成
```sql
select id from mytable where count=10*2
```
* 同样的道理，如果 where 筛选的字段用了函数，比如
```sql
select id from mytable where substring(name, 1, 3)='aaa'
```
索引也用不了。应该改成
```sql
select id from mytable where name like 'aaa%'
```
* 实际执行插入和删除大量数据的时候（大事务操作），尽可能拆分任务，不要一次执行全量，以防锁表时间过长
* 确认删除全表时，先 truncate，再 drop
* on, where, having 都可以过滤。on 最先执行，where其次，然后再已经计算出所有结果之后，having才会起作用。所以能用where尽量不用having。如果是单边的左或右连接能用on就不用where。




#### 2. 其他操作 ####

* 当只要m行数据时使用 LIMIT m，比如 LIMIT 1
* 创建一些中间表，把大量反复查询多表并且没有很高时效要求（或一段时间数据不发生变化）的结果存在中间表里。再基于这些中间表查询
* 存储过程


#### 4. 架构上的改进 ####
* 查看数据库最大连接数，查看分配的CPU，内存空间（比如临时表等都会占用内存）
* 增加线程缓存大小。如果希望服务器每秒接收数百个连接请求，那么应该将thread_cache_size设置的足够高，以便大多数新连接可以使用缓存线程。
* 从内存读取数据。设置innodb_buffer_pool_size。检查当前的配置是否合理
```sql
SHOW GLOBAL STATUS LIKE 'innodb_buffer_pool_pages_%';
```
然后查看结果里的 Innodb_buffer_pool_pages_free 值，如果为0说明缓存不够了
* 读写分离。大量的只读操作都在读库，没有任何锁，速度非常快
* 查看慢sql日志：/var/log/mysql/log-slow-queries.log



#### 5. Django相关 ####

我们平时大量使用 Django 的 ORM，效率高的写法很重要。附一下Django的官方文档 https://docs.djangoproject.com/en/2.2/topics/db/optimization/  官方文档写的非常全面。以下记录一些自己跳过的坑。


* 批量更新数据的时候，如果是一条一条更新，对数据库会有多次I/O，导致非常慢。尽量选择一次更新多条。有一种简单的情况是所有条目要更新的字段内容是一样的时候
```python
Entry.objects.select_related().filter(blog=b).update(headline='Everything is the same')
```
这种时候ORM里的save()方法不会被执行。如果想要执行的话需要手动调用一下
```python
for item in my_queryset:
    item.save()
```
但是大多数时候更新多条数据的每条数据值可能是不同的。如果是有规律的，比如是在原有值的变化，可以
```python
from django.db.models import F
Entry.objects.all().update(n_pingbacks=F('n_pingbacks') + 1)
```
如果没有规律，原生sql也只能执行多条的，ORM就没有办法更优化了，只能多次I/O。

* 同理，减少I/O，创建多条尽量用 bulk_create
```python
# insert multiple items in ProductPromotionMain
for product in products[start_pos:end_pos]:
    product_id = product.id
    productpromotion_data = {
        "promotionmain_id": self.promotionmain_id,
        "product_id": product.id,
        "promotion_type": promotion_type,
        "start_time": promotionmain.start_time,
        "end_time": promotionmain.end_time,
    }
    productpromotions.append(ProductPromotionMain(**productpromotion_data))

if len(productpromotions) > 0:
    ProductPromotionMain.objects.bulk_create(productpromotions)
```

* 当需要很多张表join以后的多字段值的时候，正常用Django的ORM获取，需要每张表都分别获取再组合，或者是用Model里property的方式把其他表相关字段拿过来放某个主表里，然后通过ORM获取这个主表对象，连带获取到所有相关表数据。有的时候属性会造成一些重复sql，比如
```python
class A(models.Model):
    id = models.Integerfield()
    aaa = models.Charfield()

    @property
    def bbb(self):
        return B.objects.get(a_id=self.id)

    @property
    def ccc(self):
        bbb = self.bbb
        return C.objects.get(b_id=bbb.id)

```
需要返回所有字段
```json
{
    "id": 1,
    "aaa": "my str",
    "bbb": {
        "id": 10,
        "a_id": 1
    },
    "ccc": {
        "id": 100,
        "b_id": 10
    }
}
```
这个时候，数据库会执行两次
```sql
select id, a_id from B where a_id=10
```
进而造成了重复查询。查看官方文档发现，这种情况完全可以使用 @cached_property 。
```python
from django.utils.functional import cached_property

class A(models.Model):
    id = models.Integerfield()
    aaa = models.Charfield()

    @cached_property
    def bbb(self):
        return B.objects.get(a_id=self.id)

    @property
    def ccc(self):
        bbb = self.bbb
        return C.objects.get(b_id=bbb.id)

```


* 上述场景还可以直接使用 raw sql，可以直接在sql语句里join，也会变快
```python
result = A.objects.raw('SELECT a.id, a.aaa, b.id, c.id FROM A a left join B on a.id=b.a_id left join C on b.id=c.b_id')
```
* queryset很大的时候，当需要轮询处理，应该使用iterator，否则全部加载很消耗内存
* 不要用len(queryset)，应该用 queryset.count()
* if some_queryset.exists() 比 if some_queryset 性能好很多，能用exists()就多用. 这些原理都和上述原生sql要注意的地方类似



##### 参考文献/文章 #####
https://www.cnblogs.com/yunfeifei/p/3850440.html
https://blog.csdn.net/dev_csdn/article/details/78721426
https://zhuanlan.zhihu.com/p/39038788