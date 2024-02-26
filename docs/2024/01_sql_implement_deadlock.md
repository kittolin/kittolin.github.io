# MySQL 底层实现和死锁问题分析

MySQL 应该是我们平时开发中用的最多的中间件了。它的 SQL 功能非常强大，一条简单的 SQL 语句背后，隐含了很多复杂而巧妙的设计思想。

本文主要分享 MySQL 中两个重要且巧妙的底层实现：事务的隔离性，update 语句的执行流程。这其中涉及到了 MySQL 的三大日志：binlog、redo log 和 undo log。这些日志是 MySQL 实现高性能和一致性的保证，也是组建主从和主备架构的前提基础。

最后再分享几个死锁问题。基于此，了解下 SQL 执行背后的加锁逻辑，以及可以如何通过日志来分析定位这些问题，在以后开发中如何规避。

首先，先来看一下 MySQL 的基础架构。

## 基础架构

MySQL 整体架构如下所示，主要包含两大组件：server 层和存储引擎层。

<img src="../../images/2024/01_sql_implement_deadlock/mysql_arch.png" width="600">

server 层又包含多个组件，负责 SQL 的检查和解析，所有跨存储引擎的功能都在这一层实现。

- 连接器：管理客户端连接，权限校验等
- 分析器：词法和语法分析
- 优化器：选择索引，生成执行计划
- 执行器：调用存储引擎接口来读写数据

存储引擎层是实际的数据存取层，这一层已经没有 SQL 的概念了。引擎层是插件式的，可以接入多种不同的引擎。

总结来说就是，经过 server 层的检测和 SQL 解析后，由执行器调用具体的存储引擎接口来完成数据读写。

## 事务隔离是如何实现的

事务，是我们使用 MySQL 过程中经常听到的词了，也是 MySQL 最为亮点的设计之一了。事务有 ACID 四大特性，这里主要以 MySQL 默认隔离级别 REPEATABLE READ（可重复读）来分享下隔离性的底层实现机制。

可重复读：在同一个事务内，多次读取相同数据时，结果保持一致，期间其他事务提交的更改对当前事务不可见。

来看下一个例子。假设有一张 account 表存储用户余额，有两个字段 user_id 和 money，初始时有一条 user_id = 1 && money = 100 的记录。下图的 S1 和 S2 都表示同一条查询语句，U1 和 U2 都表示同一条更新语句，sql 如下：

```sql
select money from account where user_id = 1;

update account set money = money + 100 where user_id = 1;
```

<img src="../../images/2024/01_sql_implement_deadlock/concurrency_t_sample.png" width="400">

分析下 S1 的取值，先给出结论：S1 = 100，可以从两个角度来看：

1. S1 查询时事务 A 还未提交，S1 得到的结果应该和事务 A 刚启动时是一样的，即初值 100；
2. S1 如果能看到 U1 的变更，即读取到了事务 B 提交的数据，这种情况属于不可重复读；S1 如果能看到 U2 的变更，即读取到了事务 C 未提交的数据，这种情况属于脏读。而可重复读级别可以避免脏读和不可重复读。

S2 的分析同 S1，结果也为 100；S3 是在事务 A 提交后，能看到 U1 和 U2 的变更，结果为 300。

### MVCC 机制

事务 A 读取到的值是 100，事务 B 在 U1 变更后读取到的值为 200，事务 C 在 U2 变更后读取到的值为 300，就好像这一行数据有三个版本，三个事务能分别读取到对应的版本。

的确，在 MySQL 里面每一行记录是可以存在有多个版本的，每次变更都会重新生成一个新的版本，同时记录下此次变更所属的事务 ID，如下图所示：

<img src="../../images/2024/01_sql_implement_deadlock/mvcc_diagram.png" width="500">

在具体实现上，MySQL 是通过 undo log 机制来模拟出多个版本的，上图的箭头 u1 和 u2 就是指的 undo log。

u1 和 u2 记录的内容可以分别理解为： 将 300 改为 100，将 400 改为 300，然后当前 V3 的 400 再通过 u2 和 u1 的回溯就能得到 V1 的 100 了。

也就是说，V1 和 V2 并不是真实存在的，而是 V3 通过真实存在的 u1 和 u2 来回溯模拟出来的。

这一机制，就是 MySQL 中经典的 MVCC 机制（多版本并发控制）。

### 一致性视图

有了 undo log，每行记录就可以从当前版本回溯到之前的任意版本了，那么每个事务又是如何确定自己应该回溯到具体哪个版本呢？这里就要引入一致性视图（read-view）的概念了。

在可重复读隔离级别下，视图是在事务启动时创建的，整个事务期间都使用该视图，即视图在整个事务内都是静态的不变的。

- 每个事务对应一个数组，保存着该事务启动时正在活跃的事务 ID，即仍在运行未提交的事务
- 低水位：上述数组里面最小的事务 ID
- 高水位：当前系统创建过的事务 ID 的最大值加 1，即应该给下一个事务分配的 ID，注意不是数组里面最大的事务 ID

### 数据可见性规则

视图数组加上高低水位，就把每行记录的多个版本分成了三个区间，每个区间内的数据版本对于当前事务是否可见的规则并不相同，具体如下：

- 若数据版本的 trx_id 小于低水位，表明该版本是由当前事务启动前的其他事务生成的，并且一定提交了的，故可见（规则 1）
- 若版本的 trx_id 大于或等于高水位，表明该版本是由当前事务启动后的事务生成的，不管提交与否，一定不可见（规则 2）
- 若版本的 trx_id 位于高低水位之间，分两种情况
  - 版本的 trx_id 在数组里面，表明该版本是由当前事务启动时正活跃的事务生成的，不管提交与否，一定不可见（规则 3）
  - 版本的 trx_id 不在数组里面，表明该版本是由当前事务启动前的其他事务生成的，且一定提交了，故可见（规则 4）

从记录的当前版本出发，一直回溯到第一个可见的版本，就是当前事务应该看到的记录的版本。

我们还是用刚才演示可重复读的例子，利用可见性规则来分析下 S1、S2 和 S3 的取值。

不妨假设事务 A、B、C 的事务 ID 分别为 31、32、30，有两个 ID 分别为 10、20 的活跃事务贯穿于整个期间，user_id = 1 的记录是由先前已提交的 ID 为 5 的事务创建的，则 A、B、C 的一致性视图如下所示：

<img src="../../images/2024/01_sql_implement_deadlock/view_arr_at_start.png" width="700">

在 S1 查询前，已经有了 U1 和 U2 两次变更，则 user_id = 1 的记录当前存在有三个版本了：

<img src="../../images/2024/01_sql_implement_deadlock/row_mvcc_sample.png" width="300">

S1 的取值分析过程如下：

- 当前版本 money = 300，trx_id 位于事务 A 的高低水位间，且在 A 的数组中，符合规则 3，不可见，往前找
- money = 200 的版本，trx_id 等于高水位，符合规则 2，不可见，继续往前找
- money = 100 的版本，trx_id 小于低水位，符合规则 1，可见，故 S1 取值为 100

S2 的分析同 S1。S3 查询时，事务 A 已提交，S3 单次查询也是个新事务，假设新事务 ID 为 33，视图如下所示：

<img src="../../images/2024/01_sql_implement_deadlock/view_arr_after_commit.png" width="300">

S3 的取值分析过程如下：

- 当前版本 money = 300，trx_id 位于事务 A2 的高低水位间，且不在 A2 的数组中，符合规则 4，可见，故 S3 取值为 300

## update 语句背后都做了什么

再来看下 update 语句是如何执行的。每次 update 都要写磁盘吗？并不是的，这样性能太差了，根本支持不了高并发。要想理解 update 的执行流程，需要先了解下两个关键日志：redo log、binlog。

### redo log

redo log 是 InnoDB 引擎特有的日志，确保了数据库的持久性和一致性，主要用于 MySQL 的崩溃恢复。

redo log 是物理日志，因为其是引擎内部日志，所以是没有 sql 语句和记录之类的概念的，记录的内容大致类似于 “在某数据页的某一偏移量上做了什么修改”。

这里你可能有疑问了：数据文件保存在磁盘，日志文件也是保存在磁盘，同样是写磁盘，为什么不直接写数据文件，还要多此一举呢？

比如执行如下 update 语句：

```sql
update account set money = money + 100 where user_id = 1;
```

如果是直接写数据文件，首先要读磁盘找到 user_id = 1 的记录，更新 money 字段后，再写回磁盘，属于随机读写；
如果是写日志文件，不用关心记录的具体位置，只需在日志文件后面追加日志，属于顺序写，磁盘的顺序写性能远大于随机写。

只要写了 redo log 日志，更新就算完成了；InnoDB 会在适当的时候将日志的变更刷新到磁盘的数据文件里去。

redo log 由多个文件组成，写过程类似于循环队列的循环写，如下图所示：

- write pos 表示当前日志接下来要写的位置
- check pos 表示已刷新到磁盘的日志的位置
- 图中绿色部分表示接下来日志可以写入的，黄色部分表示已写入但还未刷新到磁盘的

<img src="../../images/2024/01_sql_implement_deadlock/redo_log_curcle_write.png" width="300">

在具体实现上，日志是先写到 buffer 的，可配置相关参数，在事务提交时将 buffer 日志刷到磁盘去；
此外，InnoDB 还有个后台线程，定时刷 buffer 到磁盘。

### binlog

binlog 是 Server 层实现的，故所有引擎都可以使用，用于记录数据库的所有增删改操作。只要日志齐全，就可以用其重放出一个完全一样的数据库。

binlog 是逻辑日志，记录的是 sql 语句的原始逻辑，类似于 “给 user_id = 1 的记录的 money 字段加 100”。

binlog 日志写磁盘的过程如下图所示：

- 为了保证事务日志的原子性，每个线程都有其单独的 cache，在事务提交时，用 write 系统调用一次性刷到 page cache 去
- 至于何时如何调用 fsync 写到磁盘，可通过 sync_binlog 参数配置

<img src="../../images/2024/01_sql_implement_deadlock/binlog_to_disk.png" width="400">

binlog 和 redo log 的对比如下：

|          | redo log | binlog             |
| -------- | -------- | ------------------ |
| 实现层   | InnoDB   | Server             |
| 日志类型 | 物理日志 | 逻辑日志           |
| 写机制   | 循环写   | 追加写             |
| 用途     | 故障恢复 | 主从复制、备份还原 |

### 两阶段提交

update 语句主要做了三件事：更新 buffer pool，写 redo log，写 binlog。

由于 redo log 和 binlog 是两个独立的逻辑，为了保证日志和数据的一致性，MySQL 采用了两阶段提交机制，具体流程如下图所示：

<img src="../../images/2024/01_sql_implement_deadlock/update_sql_exc.png" width="300">

MySQL 重启后会去扫描 redo log 文件，分三种情况进行处理：

- redo log 里面事务完整，即有 commit 标识，则直接提交（情况 1）
- redo log 里面事务不完整，只有 prepare，则去判断对应的 binlog 是否完整存在
  - 若是，则提交事务（情况 2）
  - 否则，回滚事务（情况 3）

假设 MySQL 分别在 update 执行过程中的各个中间时间点崩溃了，来分析下重启后是如何进行恢复的。

- t1 时崩溃，redo log 和 binlog 都还未写，故一致；重启后内存的新数据丢失，从磁盘载入旧数据，update 执行不生效
- t2 时崩溃，符合情况 3，回滚事务，update 执行不生效
- t3 时崩溃，符合情况 2，提交事务，update 执行生效

可见，只要 redo log 写了 prepare 标识，且 binlog 正常写入了，此时更新操作就算生效了，后续的 commit 标识更新是否完成也不影响结果的。

## 死锁问题分析

平时使用 MySQL 过程中有可能遇到过这种错误：ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction。

这是数据库多个并发事务之间发生了死锁。那么死锁是什么，如何发生的，以及如何规避呢？

### 死锁是什么

死锁是两个及以上并发操作在互相等待对方释放资源的情况下发生的一种阻塞状态。举个例子，有进程 A、B 都需要申请资源 1、2，A 已拥有 1，在申请 2；B 已拥有 2，在申请 1。A、B 互相阻塞，都无法往下执行，就发生了死锁。

死锁有以下四个必要条件，即如果发生了死锁，这四种情况一定存在：

- 互斥条件：对互斥资源的争抢
- 不剥夺条件：资源只能主动释放，不能被强行夺走
- 请求和保持条件：进程请求的资源被其他占用，又对自己占用的资源保持不放
- 循环等待条件：存在至少一种资源的循环等待链

所以只要破坏了至少一个条件，就一定不会发生死锁。应用到 MySQL 上，前两个条件无法避免，可以通过破坏后两个条件来规避死锁的发生。

在举例之前，还需要先了解下两个概念：两阶段锁和 next-key lock。

- 两阶段锁：InnoDB 事务中，行锁是在需要时才加上的，而要等到事务结束时才统一释放
- next-key lock
  - 间隙锁：当某个事务获取了间隙锁，则锁定一个具体范围，其他事务无法在该范围内插入记录；多个事务可对同一范围同时加间隙锁；间隙锁只在可重复读级别下存在
  - 间隙锁和行锁合称 next-key lock，是前开后闭区间。比如主键 id 上的 next-key lock (5, 10]，表示间隙锁 (5, 10)，以及 id=10 的行锁

下面通过三个例子，来演示下 MySQL 中死锁的发生过程，以及如何规避。演示环境：MySQL 8.0.30，可重复读隔离级别，Innodb 引擎

### 三个死锁例子

1、用户佩戴装饰
有一张用户装饰表，记录用户所拥有的装饰，以及正在佩戴的。每个用户可拥有多个装饰，但只能佩戴其中一个。当前表中用户 1 拥有装饰 1、2、3，且佩戴装饰 1。初始化数据如下：

```sql
create table `user_decoration` (
  `id` int(11) auto_increment,
  `user_id` int(11) comment '用户 id',
  `decoration_id` int(11) comment '装饰 id',
  `is_wear` tinyint(4) comment '是否佩戴',
  primary key (`id`),
  unique key `idx_user_id_decoration_id` (`user_id`, `decoration_id`)
) comment '用户装饰表';

insert into `user_decoration` (`user_id`, `decoration_id`, `is_wear`) values
(1, 1, 1),
(1, 2, 0),
(1, 3, 0);
```

假设有并发事务 A、B，分别将用户 1 当前佩戴的装饰改成 2 和 3，执行序列如下所示，发生了死锁。
<img src="../../images/2024/01_sql_implement_deadlock/deadlock_1_original.png" width="600">

<!-- original:
transaction A
update user_decoration set is_wear = 1 where user_id = 1 and decoration_id = 2;
update user_decoration set is_wear = 0 where user_id = 1 and decoration_id != 2;

transaction B
update user_decoration set is_wear = 1 where user_id = 1 and decoration_id = 3;
update user_decoration set is_wear = 0 where user_id = 1 and decoration_id != 3;


hotfix:
transaction A
update user_decoration set is_wear = 0 where user_id = 1;
update user_decoration set is_wear = 1 where user_id = 1 and decoration_id = 2;

transaction B
update user_decoration set is_wear = 0 where user_id = 1;
update user_decoration set is_wear = 1 where user_id = 1 and decoration_id = 3; -->

MySQL 中可以通过 data_locks 表来查看 sql 执行时加了哪些锁。重点看下 LOCK_MODE 字段：

- X：next-key 锁，此时 LOCK_DATA 表示区间右端点
- X, REC_NOT_GAP：行锁
- X, GAP：间隙锁，此时 LOCK_DATA 表示区间右端点

```sql
select * from performance_schema.data_locks\G;
```

来看下事务 A T2 时刻执行的 sql 加了哪些锁。可以看到，除了在索引 idx_user_id_decoration_id 上加了 1,2 的行锁，还会回表在主键索引上加 id = 2 的行锁。主键上的锁不影响下面的分析，故不再提及。
<img src="../../images/2024/01_sql_implement_deadlock/deadlock_1_A_T2_gain_lock.png" width="400">

同理，事务 B T3 时刻在索引 idx_user_id_decoration_id 上加了 1,3 的行锁。

再来看事务 A T4 时刻为什么会阻塞。可以看到，事务 A 在 waiting 事务 B 所占据的索引 idx_user_id_decoration_id 上的 1, 3 next-key 锁。
<img src="../../images/2024/01_sql_implement_deadlock/deadlock_1_A_T4_block_lock.png" width="400">

至此，发生死锁的原因可以解释了：

- A 占据了装饰 2 的锁，在等装饰 3 的锁；
- B 占据了装饰 3 的锁，在等装饰 2 的锁

那么，可以如何规避这个死锁的发生呢？

如果是在一开始表设计阶段，可以考虑将这张表拆分成两张表：用户拥有的所有装饰，用户当前正在佩戴的装饰。
这样事务 A、B 的业务逻辑就是更改第二张表的同一记录，只需争抢一个资源，破坏了请求和保持条件，自然不会发生死锁了。

如果该表已在线上运行，可以在不变更表结构的前提下，通过改 sql 来避免吗？
观察分析可以看到，A 和 B 在整个事务期间都需要同时获取用户 1 的所有装饰的锁的。
那么是否可以在事务一开始就先占据所有需要的锁，后续就不会再因为请求锁而被阻塞，也就破坏了请求和保持条件。

基于此想法，变更之后的 sql 如下，A 在 T2 时刻就同时占据了装饰 1、2、3 的锁，B 在 T3 时刻就只能阻塞，直到 A 提交事务释放锁。
<img src="../../images/2024/01_sql_implement_deadlock/deadlock_1_hotfix.png" width="600">

2、互相转账
有一张账户表，用于存储用户的余额，转账采用转出者先扣费、转入者再入账的执行逻辑，初始数据如下：

```sql
create table `account` (
  `id` int(11) comment '账户 id',
  `money` int(11) comment '余额',
  primary key (`id`)
) comment '账户表';

insert into `account` (`id`, `money`) values
(1, 1000),
(3, 3000);
```

<!-- original
transaction A
update `account` set money=money - 100 where id = 1;
update `account` set money=money + 100 where id = 3;

transaction B
update `account` set money=money - 300 where id = 3;
update `account` set money=money + 300 where id = 1;


hotfix
transaction A
update `account` set money=money - 100 where id = 1;
update `account` set money=money + 100 where id = 3;

transaction B
update `account` set money=money + 300 where id = 1;
update `account` set money=money - 300 where id = 3; -->

现在用户 1 向 3 转账 100，3 向 1 转账 300，两个事务并发执行，发生了死锁。
<img src="../../images/2024/01_sql_implement_deadlock/deadlock_2_original.png" width="600">

还是通过 data_locks 表来查看加了哪些锁。
可以看到，A 在 T2 时刻占据了主键上 id=1 的行锁，同理 B 在 T3 时刻占据了 id=3 的行锁。
<img src="../../images/2024/01_sql_implement_deadlock/deadlock_2_A_T2_gain_lock.png" width="400">

A 在 T4 时刻因为等待 id=3 的行锁而阻塞。
<img src="../../images/2024/01_sql_implement_deadlock/deadlock_2_A_T4_block_lock.png" width="400">

至此，死锁的原因就找到了：

- A 占据 id=1 的锁，等待 id=3 的锁
- B 占据 id=3 的锁，等待 id=1 的锁

可以看到，A 是按 id 从小到大的顺序来获取锁的，而 B 是按反顺序来获取锁的，自然会出现循环互锁的情况。
那么如果 A、B 都按相同顺序获取锁呢？后来事务会因所需的锁被先前事务占据而被阻塞，也就破坏了循环等待条件。

基于此，只需把 B 的两条 sql 换下顺序就可以了，转账逻辑就变成了 id 越小的 sql 就越先执行。
<img src="../../images/2024/01_sql_implement_deadlock/deadlock_2_hotfix.png" width="600">

3、账户表插入新用户
<!-- ```sql
original
select * from `account` where id = 2 for update;
insert into `account`(id, money) values (2, 2000);

hotfix
insert ignore into `account`(id, money) values (2, 2000);
``` -->

还是例子 2 中的账户表，现有并发事务 A、B，同时判断表中是否存在 id 为 2 的用户，如果不存在则写入，执行 sql 如下，发生了死锁：
（注意：for update 表示查询是当前读，会加锁；普通查询是快照读，不加锁）
<img src="../../images/2024/01_sql_implement_deadlock/deadlock_3_original_current_read.png" width="600">

查看 data_locks 表，事务 B 在 T3 时刻执行完后，事务 A 在主键 id 的 (1, 3) 上加了间隙锁，事务 B 也在 (1, 3) 上加了间隙锁，因为间隙锁之间互不冲突。
<img src="../../images/2024/01_sql_implement_deadlock/deadlock_3_original_current_B_T3_gain_lock.png" width="400">

事务 A 在 T4 时刻插入时被事务 B 的 (1, 3) 间隙锁所阻塞。
<img src="../../images/2024/01_sql_implement_deadlock/deadlock_3_original_current_A_T4_block_lock.png" width="400">

至此，死锁发生的原因找到了：
- A 插入时被 B 的间隙锁阻塞
- B 插入时被 A 的间隙锁阻塞

如果 select 语句去掉 for update，也就是普通查询，则执行逻辑会有所不同：
<img src="../../images/2024/01_sql_implement_deadlock/deadlock_3_original_view_read.png" width="600">

原因在于普通查询不会加间隙锁，事务 B T5 时刻被事务 A 在 T4 时刻占据的 id=2 的行锁所阻塞，直到事务 A 提交释放锁，事务 B 才能往下执行，并报错唯一键冲突。
<img src="../../images/2024/01_sql_implement_deadlock/deadlock_3_original_view_B_T5_block_lock.png" width="400">

上述的演示都是在可重复读级别下的，如果是读已提交级别，是不存在间隙锁的，不管是当前读还是快照读的演示，都是报唯一键冲突而不是死锁。

如果即不想死锁也不想报唯一键冲突，可以采用 insert ignore 语句，该语句在不冲突时写入、冲突时直接忽略本次更改。
<img src="../../images/2024/01_sql_implement_deadlock/deadlock_3_hotfix.png" width="600">

总结上面三个死锁例子，在规避死锁时可以采用这些思路：
- 减少事务期间依赖的锁，可降低死锁概率，如果只需申请一个锁，则破坏了请求和保持条件，从而避免了死锁
- 在事务一开始就申请所有所需的锁，破坏请求和保持条件（注意这种方式比较适合短事务，如果用于长事务，可能会让后续依赖相同的锁的短事务一直阻塞，导致饥饿超时）
- 按同样的顺序去申请锁，破坏循环等待条件

## 参考资料
- [MySQL 实战 45 讲](https://time.geekbang.org/column/intro/100020801?tab=catalog)
- [事务隔离级别是怎么实现的？](https://xiaolincoding.com/mysql/transaction/mvcc.html)
