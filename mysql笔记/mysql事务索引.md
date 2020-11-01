# MySQL事务与锁原理

显示mysql的事务级别

```sql
-- mysql 5.7
show gloabl variables like "tx_isolation";

-- mysql 8
show global variables like "transaction_isolation";

```

## 1.1数据库事务

### 1.1.1 什么是数据库事务？

事务是数据库管理系统(DBMS)执行过程中的一个逻辑单位，由一个有限的数据库操作序列构成



### 1.1.2 数据库事务的四大特性

1 原子性：通过undo log进行回滚的

2 持久性： crash-safe，比如宕机了， 根据redo log进行回滚

**3. 隔离性：** 

4 一致性：



### 1.1.3. 准备sql

```sql
-- 建表语句

CREATE TABLE `t1` (
  `id` int(11) DEFAULT NULL,
  `name` varchar(255) DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
INSERT INTO `t1` (`id`, `name`) VALUES (1, '1');
INSERT INTO `t1` (`id`, `name`) VALUES (2, '2');
INSERT INTO `t1` (`id`, `name`) VALUES (3, '3');
INSERT INTO `t1` (`id`, `name`) VALUES (4, '4');


DROP TABLE IF EXISTS `t2`;
CREATE TABLE `t2` (
  `id` int(11) NOT NULL,
  `name` varchar(255) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
INSERT INTO `t2` (`id`, `name`) VALUES (1, '1');
INSERT INTO `t2` (`id`, `name`) VALUES (4, '4');
INSERT INTO `t2` (`id`, `name`) VALUES (7, '7');
INSERT INTO `t2` (`id`, `name`) VALUES (10, '10');


DROP TABLE IF EXISTS `t3`;
CREATE TABLE `t3` (
  `id` int(11) ,
  `name` varchar(255) ,
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_name` (`name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
INSERT INTO `t3` (`id`, `name`) VALUES (1, '1');
INSERT INTO `t3` (`id`, `name`) VALUES (4, '4');
INSERT INTO `t3` (`id`, `name`) VALUES (7, '7');
INSERT INTO `t3` (`id`, `name`) VALUES (10, '10');


```





```sql
update student set name='remous' where id=1;

-- 查看数据库是否自动提交
show variables like 'authocommit';
```



```sql
begin;
update student set name ='Rem' where id=1;
-- 回滚事务
rollback；
-- 关闭事务
set session autocommit=off;
updete student set name where 'Remous' where id=1;
-- 没有提交的数据放在buffer里面
commit;
-- 连接断开-事务结束
```



## 1.2 事务引发的并发问题

### 1.2.1 脏读

<img src="E:\笔记\mysql笔记\脏读.png" alt="脏读" style="zoom:70%;" />





### 1.2.2 不可重复读

<img src="E:\笔记\mysql笔记\不可重复读.png" alt="不可重复读" style="zoom:67%;" />



### 1.2.3 幻读

只有新增造成的问题才叫幻读。

<img src="E:\笔记\mysql笔记\幻读.png" alt="幻读" style="zoom:67%;" />



## 1.3 事务并发问题解决方法

事务并发的三大问题提示都是数据库读一致性问题。必须由数据库提供一定的事务隔离机制。

<img src="E:\笔记\mysql笔记\事务的隔离级别.png" alt="事务的隔离级别" style="zoom:67%;" />



(Repeatable Read)在InnoDb可重复读不可能

#### 1.3.1 解决方案1

保证一个事务中前后两次读取数据结果一致，实现事务隔离，应该怎么做？

在读取数据前，对其加锁，阻止其他事务对数据进行修改(LBCC)Lock BASED Concurrency Control

#### 1.3.2 解决方案2

MVCC Multi Version Concurrency Control。

生成一个数据请求时间点的一致性数据库快照(Snapshot)，并用这个快照提供一定级别(语句级或事务级)的一致性读取

*MVCC只在RC RR中使用



# 2. MySQL锁

![mysql5.7锁](E:\笔记\mysql笔记\mysql5.7锁.png)

## 2.1 行锁和表锁的区别

A record lock is a lock on an **index record**. Record locks always lock **index records**, even if a table is defined with no indexes. For such cases, InnoDB creates a hidden clustered index and uses this index for record locking.

<img src="E:\笔记\mysql笔记\行锁与表锁.png" alt="行锁与表锁" style="zoom:67%;" />



### 2.1.1 共享锁和排他锁Shared and Exclusive Locks

使用场景 order_info ->order_detail 一对多

修改order_detail,又不想修改order_info。可以使用共享锁。



<img src="file:///C:\Users\Remous\AppData\Roaming\Tencent\Users\1605649124\TIM\WinTemp\RichOle\7YLI_0LN{0HCQRCUI[I98EX.png" alt="img" style="zoom:67%;" />



### 2.1.2 意向锁 Intention Locks





### 2.1.3 记录锁Record Locks

4.索引是什么？

​	索引是一种有序的数据结构



5.id有一个索引，name有一个索引，它们两个为什么冲突



6一种表有没有可能没有索引？

IOT 聚集索引：

1. primary
2. no null unique key
3. row id 

for update ;在索引字段上面用，如果不在索引字段上面有，会锁住表。



### 2.1.4 间隙锁Gap Lock



### 2.1.5 Next-Key Locks

**next-key-Locks=行锁+Gap Lock**



### 2.1.6 临建锁

最后的临建区间：最后一个key的下一个临建区间。InnoDb 行锁默认算法->临建锁。

![临建锁](E:\笔记\mysql笔记\临建锁.png)





InnoDb是如何通过Repeatable Read解决幻读的问题的？

是因为它有一种间隙锁的技术。Gap Lock

