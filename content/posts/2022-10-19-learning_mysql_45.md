---
categories:
- 学习笔记
date: '2022-10-19T04:28:00.000Z'
showToc: true
tags:
- MySQL
- 极客时间
title: 专栏学习-MySQL实战45讲

---



> 本文是个人学习极客时间专栏《MySQL实战45讲》过程中所记录的一些笔记，内容来源于专栏

## MySQL基础架构

![MySQL%E5%9F%BA%E7%A1%80%E6%9E%B6%E6%9E%84.png](https://raw.githubusercontent.com/linyanbin666/pic/master/notionimg/19/c2/19c2f44463879bd63e84181f16ad9b45.png)

### Server层

涵盖MySQL大多数功能，所有跨存储引擎的功能都在这一层实现，比如存储引擎、触发器、视图等

- 连接器：管理连接，权限校验

- 查询缓存：命中则直接返回结果（MySQL 8.0已删除这一块）

- 分析器：词法分析，语法分析

- 优化器：执行计划生成，索引选择

- 执行器：操作引擎，返回结果

- ...

### 存储引擎层

负责数据的存储和提取，提供读写接口，插件式架构

## MySQL日志

### redo log

- InnoDB引擎特有的日志；记录的是“在某个数据页上做了什么修改”；空间固定且是循环写；用于保证即使数据库发生异常重启，之前提交的记录都不会丢失（**crash-safe**）。

- 当有一条记录需要更新时，InnoDB会把记录写到redo log（先写到**log buffer，**即写入到log buffer划分的众多的**redo log block**，解决直接写磁盘带来的性能损耗**）**里面，并更新内存，这个时候更新就算完成了。同时，InnoDB 引擎会在适当的时候，将这个操作记录更新到磁盘里面。

- redo log 的写入机制

	- 事务在执行过程中，生成的 redo log 是要先写到 redo log buffer （线程共享）的，之后再经过 write 操作写入到文件系统的 page cache，fsync 操作持久化到磁盘，write 和 fsync 的时机，可由参数 **innodb_flush_log_at_trx_commit** 控制

	- 让一个没有提交的事务的 redo log 写入到磁盘中的场景

		- InnoDB 有一个后台线程，每隔 1 秒，就会把 redo log buffer 中的日志，调用 write 写到文件系统的 page cache，然后调用 fsync 持久化到磁盘

		- redo log buffer 占用的空间即将达到 **innodb_log_buffer_size** 一半的时候，后台线程会主动写盘，因事务还未提交，这个写盘动作只是 write

		- 并行的事务提交的时候，顺带将这个事务的 redo log buffer 持久化到磁盘

	- 组提交机制

### binlog

- MySQL Server层的实现，所有引擎都可以使用；记录的是语句的原始逻辑，比如“给 ID=x 这一行的 c 字段加 1 ”；追加写不会覆盖以前的日志；用于归档，主从数据同步。

- binlog 的写入机制

	- 事务执行过程中，先把日志写到 **binlog cache**（线程独占），事务提交的时候，再把 binlog cache 写到 binlog 文件中，写到 binlog 文件分 write 和 fsync，其中 **write 是写到文件系统的 page cache，fsync 才是把数据持久化到磁盘**，write 和 fsync 的时机，是由参数 **sync_binlog **控制

	- 组提交机制，可调整 binlog_group_commit_sync_delay 和 binlog_group_commit_sync_no_delay_count 参数，减少 binlog 的写盘次数

- binlog 的存储格式

	- **statement**：binlog 里面记录的就是 SQL 语句的原文，语句在不同实例执行的结果可能会不一致，从而可能导致数据不一致

	- **row**：binlog 里面记录了真实行的信息，可避免数据不一致，也可用来恢复数据，但比较占空间和IO资源（MySQL 5.7 开始默认使用的格式）

	- **mixed**：MySQL 自己会判断这条 SQL 语句是否可能引起主备不一致，如果有可能，就用 row 格式，否则就用 statement 格式

### redo log和binlog的写入顺序 - redo log两阶段提交

- Innodb引擎生成redo log，将其置为**prepare状态**

- 执行器生成binlog并写入磁盘

- 执行器调用引擎的提交事务接口，引擎把刚写入的redo log改为**commit状态**

### 常见配置参数

- **innodb_flush_log_at_trx_commit**：设置事务提交时redo log磁盘写入策略，默认值为`1`

	- **0**：提交事务时不立即把redo log buffer里的数据写入日志文件，而是依靠主线程每1s执行一次写入并刷新到磁盘，MySQL宕机时会丢失1s数据

	- **1**：提交事务时会把redo log buffer里的数据写入日志文件，并且会执行fsync（阻塞操作）强制将os buffer刷新到磁盘

	- **2**：提交事务时会把redo log buffer里的数据写入日志文件，但不会执行fsync强制将os buffer刷新到磁盘，而是依靠主线程每1s执行一次刷新到磁盘，MySQL宕机时可能会丢失1s数据

- **sync_binlog**：设置事务提交时binlog磁盘写入策略，默认值为`0`

	- **0**：当事务提交之后，MySQL不做fsync之类的磁盘同步指令刷新binlog_cache中的信息到磁盘，而让Filesystem自行决定什么时候来做同步，或者cache满了之后才同步到磁盘

	- **1**：当每进行1次事务提交之后，MySQL将进行一次fsync之类的磁盘同步指令来将binlog_cache中的数据强制写入磁盘

	- **n**：当每进行n次事务提交之后，MySQL将进行一次fsync之类的磁盘同步指令来将binlog_cache中的数据强制写入磁盘

- **binlog_cache_size**：控制单个线程内 binlog cache 所占内存的大小。如果超过了这个参数规定的大小，就要暂存到磁盘。默认值为`32KB`

- **innodb_log_buffer_size**：控制 InnoDB 写入磁盘上日志文件的缓冲区大小。默认值为`16M`

- **binlog_group_commit_sync_delay**：控制 binlog 提交在调用 fsync 之前等待的微秒数。默认值为`0`

- **binlog_group_commit_sync_no_delay_count**：控制累积多少次事务以后才调用 fsync ，如果binlog_group_commit_sync_delay 设置为 0 ，则该参数设置无效。默认值为`0`

## MySQL事务

### 隔离性与隔离级别

多个事务同时执行时，可能出现**脏读**、**不可重复读**、**幻读**的问题，为了解决这些问题，就有了“隔离级别”，SQL标准的事务隔离级别包含：

- **读未提交（Read Uncommitted）**：一个事务还未提交时，它做的变更可以被其他事务看到

- **读已提交（Read Committed）**：一个事务提交之后，它做的变更才能被其他事务看到

- **可重复读（Repeatable Read）**：一个事务在执行过程中看到的数据，总是跟这个事务在启动时看到的数据是一致的（MySQL默认隔离级别）

- **串行化（Serializable）**：对于同一记录，写会加“写锁”，读会加“读锁”，当出现锁冲突时，后访问的事务必须等前一个事务执行完成才能继续进行

### 默认隔离级别

- Oracle默认隔离级别为Read Committed，Oracle支持Read Committed、Serializable和Read-Only，Serializable和Read-Only显然都是不适合作为默认隔离级别的，那么就只剩Read Committed这个唯一的选择）

- MySQL默认隔离级别为Repeatable Read，MySQL早期只有statement这种binlog格式，这时候如果使用读提交(Read Committed)、读未提交(Read Uncommitted)这两种隔离级别可能会出现主从数据不一致问题

### 事务的启动方式

- 显示启动事务语句，begin或start transaction

- set autocommit = 0，这个命令会将这个线程的自动提交关掉，意味着如果你只执行一个select语句，这个事务就启动了，并且不会自动提交，事务将持续到你主动执行 commit 或 rollback 语句，或者断开连接

### 幻读

- 欢读的定义：指的是一个事务在前后两次查询同一个范围的时候，后一次查询看到了前一次查询没有看到的行

	- **在可重复读隔离级别下**，普通的查询是快照读，是不会看到别的事务插入的数据的。因此，**幻读在“当前读”下才会出现**

	- 幻读仅专指“新插入的行”

- 幻读的问题

	- 破坏语义上行的加锁声明

	- 破坏数据的一致性

- 幻读的解决

	- InnoDB 引入了新的锁，也就是间隙锁 (Gap Lock)。跟间隙锁存在冲突关系的，是“往这个间隙中插入一个记录”这个操作，间隙锁之间都不存在冲突关系

	- 间隙锁和行锁合称 next-key lock，每个 next-key lock 是前开后闭区间

	- **间隙锁是在可重复读隔离级别下才会生效的**。如果把隔离级别设置为读提交的话，就没有间隙锁了。但同时，要解决可能出现的数据和日志不一致问题，需要把 binlog 格式设置为 row

### 常见配置参数

- **transaction-isolation**：设置事务隔离级别，默认值为repeatable-read

## MySQL索引

### 索引的常见模型

- **哈希表**：以键值存储数据的结构，做区间查询的速度很慢（因链表无序），适用于只有等值查询的场景

- **有序数组**：在等值查询和范围查询场景中的性能都很好，但更新数据时成本较高（需要挪动记录），只适用于静态存储引擎（即数据不会修改的场景）

- **搜索树**：二叉搜索树的父节点左子树所有结点的值小于父节点的值，右子树所有结点的值大于父节点的值，大多数的数据库存储并不使用二叉树，而是使用“N叉树”（**通过改变 key 值或改变页大小可以调整N值**）。其原因是，索引不止存在内存中，还要写到磁盘上，使用”N叉树”可以减少树高，从而减少磁盘读写次数

### Innodb的索引模型

- InnoDB 使用了 B+ 树索引模型，每一个索引对应一棵 B+ 树

- 根据叶子节点的内容，索引类型分为主键索引（聚簇索引）和非主键索引（二级索引）

- 主键索引的叶子节点存的是整行数据，非主键索引的叶子节点内容是主键的值

### 索引维护

- 页分裂：在**无序插入**新数据时，如果索引数据页已经满了，根据 B+ 树的算法，这时候需要申请一个新的数据页，然后挪动部分数据过去，这个过程称为页分裂

- 页合并：当相邻两个页由于删除了数据，利用率很低之后，会将数据页做合并。合并的过程，可以认为是分裂过程的逆过程

### 为什么建议建表语句里要有自增主键

- 从性能角度看，自增主键的插入数据模式为递增插入的场景，每次插入一条新纪录，都是追加操作，都不涉及到挪动其他记录，也不会触发叶子节点的分裂（写数据成本低/索引维护成本低）

- 从存储角度看，普通索引的叶子节点需要存储主键，主键长度越小，普通索引占用的空间也就越小

### 性能优化

- **覆盖索引**：当查询字段在索引上可以直接获取结果，不需要回表操作时（即索引“覆盖”了我们的查询需求），我们称该索引为覆盖索引，覆盖索引可以减少树的搜索次数，显著提升查询性能

- **最左前缀原则**：只要满足索引的最左前缀，就可以利用索引来加速检索。这个最左前缀可以是联合索引的最左 N 个字段，也可以是字符串索引的最左 M 个字符

- **索引下推**：在 MySQL 5.6 引入的索引下推优化，可以在索引遍历过程中，对索引中包含的字段先做判断，直接过滤掉不满足条件的记录，减少回表次数

### 普通索引和唯一索引怎么选

- 从业务角度考虑，如果字段需要数据库来保证唯一性，则只能选择唯一索引

- 如果字段由业务代码保证唯一性，则可从两种索引的查询和更新性能角度考虑

	- 对于查询过程，唯一索引找到第一个满足条件的纪录后，就会停止检索；普通索引找到第一个满足条件的纪录后，需要继续往后找，直到碰到不满足条件的纪录。虽然普通索引检索次数相比唯一索引多，但因Innodb引擎是按页读写数据的，所以性能上的差异不大

	- 对于更新过程，当纪录要更新的目标页不在内存时，唯一索引需要将数据页读入内存，判断到没有冲突，再更新这个值；普通索引可以利用** change buffer **机制，将更新记录在 change buffer 就可返回。利用 change buffer 机制，普通索引更新性能相比唯一索引会高点

		- change buffer的使用场景：对于写多读少的业务来说，页面在写完以后马上被访问到的概率比较小，此时 change buffer 的使用效果最好，常见的就是账单类、日志类的系统

		- **redo log 主要节省的是随机写磁盘的 IO 消耗（转成顺序写），而 change buffer 主要节省的则是随机读磁盘的 IO 消耗**

### 索引选择异常和处理

大多数时候优化器都能找到正确的索引，但偶尔会碰到索引选择错误的问题，可尝试通过以下几种方法处理

- 由于索引统计信息不准确导致的问题，可以用 analyze table 来解决

- 对于其他优化器误判的情况，可以采用以下方法处理

	- 采用 force index 强行选择一个索引

	- 考虑修改语句，引导 MySQL 使用我们期望的索引

	- 在有些场景下，可以新建一个更合适的索引，来提供给优化器做选择，或删掉误用的索引

### 字符串创建索引

- 直接创建完整索引，这样可能比较占用空间

- 创建前缀索引，节省空间，但会增加查询扫描次数，并且不能使用覆盖索引

- 倒序存储，再创建前缀索引，用于绕过字符串本身前缀的区分度不够的问题

- 创建 hash 字段索引，查询性能稳定，有额外的存储和计算消耗，跟第三种方式一样，都不支持范围扫描

### 索引不生效的场景

- **条件字段函数操作**：**对索引字段做函数操作，可能会破坏索引值的有序性，因此优化器就决定放弃走树搜索功能**，即使是对于不改变有序性的函数，也不会考虑使用索引

- **隐式类型转换**：where条件左侧因隐式类型转换操作使用了函数操作，同第一条规则

	- 在 MySQL 中，字符串和数字做比较的话，是将字符串转换成数字

- **隐式字符编码转换**：where条件左侧因隐式字符编码转换操作使用了函数操作，同第一条规则

### 常见配置参数

- **innodb_change_buffer_max_size**：设置 Change Buffer能占用 Buffer Pool 的最大比例，默认值为25

## MySQL锁

### 全局锁

- 全局锁就是对整个数据库实例加锁。MySQL提供了一个加全局读锁的方法，命令是 **Flush tables with read lock** (FTWRL)。该命令的典型使用场景是做**全库逻辑备份**。也就是把整库每个表都 select 出来存成文本。

- 官方自带的逻辑备份工具是 **mysqldump**。当 mysqldump 使用参数 —**single-transaction **的时候，导数据之前就会启动一个事务，来确保拿到一致性视图（可重复读隔离级别下）。而由于 MVCC 的支持，这个过程中数据是可以正常更新的。但该方法只适用于所有的表使用事务引擎的库

### 表级锁

- MySQL表级锁有两种，一种是**表锁**，一种是**元数据锁**（Meta data lock，MDL）

- 表锁的语法是 lock tables … read/write，可以用 unlock tables 主动释放锁或等待客户端断开时自动释放

- 元数据锁（MDL）在 MySQL 5.5 版本引入，不需要显示使用，在访问一个表时会被自动加上。当对一个表做增删改查操作（DML）的时候，加 MDL 读锁；当要对表做结构变更操作（DDL）的时候，加 MDL 写锁。事务中的 MDL 锁，在语句执行开始时申请，但是语句结束后并不会马上释放，而会等到整个事务提交后再释放

### 行锁

- 在 InnoDB 事务中，行锁是在需要的时候才加上的，但并不是不需要了就立刻释放，而是要等到事务结束时才释放。这个就是两阶段锁协议。如果事务中需要锁多个行，要把最可能造成锁冲突、最可能影响并发度的锁尽量往后放

- 死锁应对策略

	- 直接进入等待，直到超时。这个超时时间可以通过参数 innodb_lock_wait_timeout 来设置

	- 发起死锁检测，发现死锁后，主动回滚死锁链条中的某一个事务，让其他事务得以继续执行。将参数 innodb_deadlock_detect 设置为 on，表示开启这个逻辑。
