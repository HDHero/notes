提问

### 1.1. 问题1

​	描述一条select语句的执行流程。模块的作用



### 1.2. 问题2

​	记录redo log 为什么要用两阶段提交？如果redo log写成功了，binLog写入失败会有什么影响？

### 1.3. 问题3

binLog的作用





## 2.MYSQL连接

### 2.1 第一步：建立连接。

mysql可以是长连接和短连接，TCP/UDP.

如何查看服务端有多少连接？

```sql
show global status like 'Thread&';
```

每产生一个连接会到服务端创建一个线程。mysql会把长时间不活动的线程干掉。

```sql
-- 非交互式超时时间，如JDBC查询
show global variables like 'wait_timeout';
-- 交互式超时时间，如数据库工具，Navicat
show global variables like 'interactive_timeout';
```

### 2.2 mysql默认的最大的连接数？

```sql
-- mysql5.7支持客户端最大的是100,000，默认是151，最小是1 
show variables like 'max_connections';
```

#### 2.2.1 显示mysql的事务隔离级别

```mysql
show global variables like '%isolation%';
-- 动态修改
set global transaction_isolation=
-- 永久修改my.conf

```

#### 2.2.2 频繁缓存数据，热点数据

使用redis

### 3 mysql执行语句

解析树

![mysql解析树](E:\笔记\mysql笔记\mysql解析树.png)

```sql
-- 执行之前报错，语义分析，表是否存在
select * from notExistTalbe
```

一条SQL

优化SQL

生成执行路径

<img src="E:\笔记\mysql笔记\mysql执行.png" alt="mysql执行" style="zoom:75%;" />

选择一个它认为最优的执行路径，MySQL的优化器是基于cost成本的优化器

<img src="E:\笔记\mysql笔记\mysql架构分层.png" alt="mysql架构分层" style="zoom:75%;" />

查看mysql的执行计划

```mysql
explain select * from user_innodb;
explain FORMAT=JSON select * from user_innodb;
-- 查看表存放的位置
show variables like 'datadir';
```

table1 快 不需要持久化 内存 ；Memory引擎

table2 历史数据存档 不用修改 不需要索引 压缩

talbe3 读写并发，读写不干扰，数据一致性。



3.2.1

一条update的语句的执行原理

user表 将name=Remous 修改为Emlia

开启事务transaction

(1)  disk->buffer pool ->Server层

(2) 修改page Remous ->Emlia

(3) 记录事务日志： 回滚日志undo log和崩溃恢复日志redo log

(4) 调用存储引擎API -->写入buffer pool

(5) 事务提交

刷脏 

1. 二进制日志 binlog(逻辑日志)，任何存储引擎都有。记录DDL，DML语句到binlog。
2. 默认是关闭的，消耗性能。
3. 主从复制
4. 数据恢复()
5. <img src="E:\笔记\mysql笔记\binlog.png" alt="binlog" style="zoom:75%;" />