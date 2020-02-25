<!-- GFM-TOC -->
* [一、索引](#一索引)
    * [B+ Tree 原理](#b-tree-原理)
    * [MySQL 索引](#mysql-索引)
    * [索引优化](#索引优化)
    * [索引的优点](#索引的优点)
    * [索引的使用条件](#索引的使用条件)
* [二、查询性能优化](#二查询性能优化)
    * [使用 Explain 进行分析](#使用-explain-进行分析)
    * [优化数据访问](#优化数据访问)
    * [重构查询方式](#重构查询方式)
* [三、存储引擎](#三存储引擎)
    * [InnoDB](#innodb)
    * [MyISAM](#myisam)
    * [比较](#比较)
* [四、数据类型](#四数据类型)
    * [整型](#整型)
    * [浮点数](#浮点数)
    * [字符串](#字符串)
    * [时间和日期](#时间和日期)
* [五、切分](#五切分)
    * [水平切分](#水平切分)
    * [垂直切分](#垂直切分)
    * [Sharding 策略](#sharding-策略)
    * [Sharding 存在的问题](#sharding-存在的问题)
* [六、复制](#六复制)
    * [主从复制](#主从复制)
    * [读写分离](#读写分离)
    * [过期读](#过期读)
* [七、常用参数](#七常用参数)
* [八、问题](#八问题)
* [参考资料](#参考资料)

<!-- GFM-TOC -->


# 一、索引

## B+ Tree 原理

### 1. 数据结构

B Tree 指的是 Balance Tree，也就是平衡树。平衡树是一颗查找树，并且所有叶子节点位于同一层。

B+ Tree 是基于 B Tree 和叶子节点顺序访问指针进行实现，它具有 B Tree 的平衡性，并且通过顺序访问指针来提高区间查询的性能。

在 B+ Tree 中，一个节点中的 key 从左到右非递减排列，如果某个指针的左右相邻 key 分别是 key<sub>i</sub> 和 key<sub>i+1</sub>，且不为 null，则该指针指向节点的所有 key 大于等于 key<sub>i</sub> 且小于等于 key<sub>i+1</sub>。

<div align="center"> <img src="https://cs-notes-1256109796.cos.ap-guangzhou.myqcloud.com/33576849-9275-47bb-ada7-8ded5f5e7c73.png" width="350px"> </div><br>

### 2. 操作

进行查找操作时，首先在根节点进行二分查找，找到一个 key 所在的指针，然后递归地在指针所指向的节点进行查找。直到查找到叶子节点，然后在叶子节点上进行二分查找，找出 key 所对应的 data。

插入删除操作会破坏平衡树的平衡性，因此在插入删除操作之后，需要对树进行一个分裂、合并、旋转等操作来维护平衡性。

### 3. 与红黑树的比较

红黑树等平衡树也可以用来实现索引，但是文件系统及数据库系统普遍采用 B+ Tree 作为索引结构，主要有以下两个原因：

（一）更少的查找次数

平衡树查找操作的时间复杂度和树高 h 相关，O(h)=O(log<sub>d</sub>N)，其中 d 为每个节点的出度。

红黑树的出度为 2，而 B+ Tree 的出度一般都非常大，所以红黑树的树高 h 很明显比 B+ Tree 大非常多，查找的次数也就更多。

（二）利用磁盘预读特性

为了减少磁盘 I/O 操作，磁盘往往不是严格按需读取，而是每次都会预读。预读过程中，磁盘进行顺序读取，顺序读取不需要进行磁盘寻道，并且只需要很短的磁盘旋转时间，速度会非常快。

操作系统一般将内存和磁盘分割成固定大小的块，每一块称为一页，内存与磁盘以页为单位交换数据。数据库系统将索引的一个节点的大小设置为页的大小，使得一次 I/O 就能完全载入一个节点。并且可以利用预读特性，相邻的节点也能够被预先载入。

## MySQL 索引

索引是在存储引擎层实现的，而不是在服务器层实现的，所以不同存储引擎具有不同的索引类型和实现。

### 1. B+Tree 索引

是大多数 MySQL 存储引擎的默认索引类型。

因为不再需要进行全表扫描，只需要对树进行搜索即可，所以查找速度快很多。

因为 B+ Tree 的有序性，所以除了用于查找，还可以用于排序和分组。

可以指定多个列作为索引列，多个索引列共同组成键。

适用于全键值、键值范围和键前缀查找，其中键前缀查找只适用于最左前缀查找。如果不是按照索引列的顺序进行查找，则无法使用索引。

InnoDB 的 B+Tree 索引分为主索引和辅助索引。主索引的叶子节点 data 域记录着完整的数据记录，这种索引方式被称为聚簇索引。因为无法把数据行存放在两个不同的地方，所以一个表只能有一个聚簇索引。

<div align="center"> <img src="https://cs-notes-1256109796.cos.ap-guangzhou.myqcloud.com/45016e98-6879-4709-8569-262b2d6d60b9.png" width="350px"> </div><br>

辅助索引的叶子节点的 data 域记录着主键的值，因此在使用辅助索引进行查找时，需要先查找到主键值，然后再到主索引中进行查找。

<div align="center"> <img src="https://cs-notes-1256109796.cos.ap-guangzhou.myqcloud.com/7c349b91-050b-4d72-a7f8-ec86320307ea.png" width="350px"> </div><br>

### 2. 哈希索引

哈希索引能以 O(1) 时间进行查找，但是失去了有序性：

- 无法用于排序与分组；
- 只支持精确查找，无法用于部分查找和范围查找。

InnoDB 存储引擎有一个特殊的功能叫“自适应哈希索引”，当某个索引值被使用的非常频繁时，会在 B+Tree 索引之上再创建一个哈希索引，这样就让 B+Tree 索引具有哈希索引的一些优点，比如快速的哈希查找。缺点是占用Innodb的内存缓存，使用了 lacth 锁保护内存中的hash结构。
```
mysql> show variables like '%ap%hash_index';
+----------------------------+-------+
| Variable_name              | Value |
+----------------------------+-------+
| innodb_adaptive_hash_index | ON    |
+----------------------------+-------+
1 row in set (0.01 sec)
```

### 3. 全文索引

MyISAM 存储引擎支持全文索引，用于查找文本中的关键词，而不是直接比较是否相等。

查找条件使用 MATCH AGAINST，而不是普通的 WHERE。

全文索引使用倒排索引实现，它记录着关键词到其所在文档的映射。

InnoDB 存储引擎在 MySQL 5.6.4 版本中也开始支持全文索引。

### 4. 空间数据索引

MyISAM 存储引擎支持空间数据索引（R-Tree），可以用于地理数据存储。空间数据索引会从所有维度来索引数据，可以有效地使用任意维度来进行组合查询。

必须使用 GIS 相关的函数来维护数据。

## 索引优化

### 1. 独立的列

在进行查询时，索引列不能是表达式的一部分，也不能是函数的参数，否则无法使用索引。

例如下面的查询不能使用 actor_id 列的索引：

```sql
SELECT actor_id FROM sakila.actor WHERE actor_id + 1 = 5;
```

### 2. 多列索引

在需要使用多个列作为条件进行查询时，使用多列索引比使用多个单列索引性能更好。例如下面的语句中，最好把 actor_id 和 film_id 设置为多列索引。

```sql
SELECT film_id, actor_ id FROM sakila.film_actor
WHERE actor_id = 1 AND film_id = 1;
```
[mysql联合索引的使用规则](https://blog.csdn.net/wdjxxl/article/details/79790421#commentBox)

### 3. 索引列的顺序

让选择性最强的索引列放在前面。

索引的选择性是指：不重复的索引值和记录总数的比值。最大值为 1，此时每个记录都有唯一的索引与其对应。选择性越高，每个记录的区分度越高，查询效率也越高。

例如下面显示的结果中 customer_id 的选择性比 staff_id 更高，因此最好把 customer_id 列放在多列索引的前面。

```sql
SELECT COUNT(DISTINCT staff_id)/COUNT(*) AS staff_id_selectivity,
COUNT(DISTINCT customer_id)/COUNT(*) AS customer_id_selectivity,
COUNT(*)
FROM payment;
```

```html
   staff_id_selectivity: 0.0001
customer_id_selectivity: 0.0373
               COUNT(*): 16049
```

### 4. 前缀索引

对于 BLOB、TEXT 和 VARCHAR 类型的列，必须使用前缀索引，只索引开始的部分字符。

前缀长度的选取需要根据索引选择性来确定。

### 5. 覆盖索引

索引包含所有需要查询的字段的值。

具有以下优点：

- 索引通常远小于数据行的大小，只读取索引能大大减少数据访问量。
- 一些存储引擎（例如 MyISAM）在内存中只缓存索引，而数据依赖于操作系统来缓存。因此，只访问索引可以不使用系统调用（通常比较费时）。
- 对于 InnoDB 引擎，若辅助索引能够覆盖查询，则无需访问主索引。

## 索引的优点

- 大大减少了服务器需要扫描的数据行数。

- 帮助服务器避免进行排序和分组，以及避免创建临时表（B+Tree 索引是有序的，可以用于 ORDER BY 和 GROUP BY 操作。临时表主要是在排序和分组过程中创建，不需要排序和分组，也就不需要创建临时表）。

- 将随机 I/O 变为顺序 I/O（B+Tree 索引是有序的，会将相邻的数据都存储在一起）。

## 索引的使用条件

- 对于非常小的表、大部分情况下简单的全表扫描比建立索引更高效；

- 对于中到大型的表，索引就非常有效；

- 但是对于特大型的表，建立和维护索引的代价将会随之增长。这种情况下，需要用到一种技术可以直接区分出需要查询的一组数据，而不是一条记录一条记录地匹配，例如可以使用分区技术。

# 二、查询性能优化

## 使用 Explain 进行分析

Explain 用来分析 SELECT 查询语句，开发人员可以通过分析 Explain 结果来优化查询语句。

比较重要的字段有：

- select_type : 查询类型，有简单查询、联合查询、子查询等
- key : 使用的索引
- key_len : 索引里使用的字节数
- rows : 估计要读取并检测的行数
- Extra : 额外信息。比如 Using where、Using temporary、Using filesort

## 优化数据访问

### 1. 减少请求的数据量

- 只返回必要的列：最好不要使用 SELECT * 语句。
- 只返回必要的行：使用 LIMIT 语句来限制返回的数据。
- 缓存重复查询的数据：使用缓存可以避免在数据库中进行查询，特别在要查询的数据经常被重复查询时，缓存带来的查询性能提升将会是非常明显的。

### 2. 减少服务器端扫描的行数

最有效的方式是使用索引来覆盖查询。

## 重构查询方式

### 1. 切分大查询

一个大查询如果一次性执行的话，可能一次锁住很多数据、占满整个事务日志、耗尽系统资源、阻塞很多小的但重要的查询。

```sql
DELETE FROM messages WHERE create < DATE_SUB(NOW(), INTERVAL 3 MONTH);
```

```sql
rows_affected = 0
do {
    rows_affected = do_query(
    "DELETE FROM messages WHERE create  < DATE_SUB(NOW(), INTERVAL 3 MONTH) LIMIT 10000")
} while rows_affected > 0
```

### 2. 分解大连接查询

将一个大连接查询分解成对每一个表进行一次单表查询，然后在应用程序中进行关联，这样做的好处有：

- 让缓存更高效。对于连接查询，如果其中一个表发生变化，那么整个查询缓存就无法使用。而分解后的多个查询，即使其中一个表发生变化，对其它表的查询缓存依然可以使用。
- 分解成多个单表查询，这些单表查询的缓存结果更可能被其它查询使用到，从而减少冗余记录的查询。
- 减少锁竞争；
- 在应用层进行连接，可以更容易对数据库进行拆分，从而更容易做到高性能和可伸缩。
- 查询本身效率也可能会有所提升。例如下面的例子中，使用 IN() 代替连接查询，可以让 MySQL 按照 ID 顺序进行查询，这可能比随机的连接要更高效。

```sql
SELECT * FROM tag
JOIN tag_post ON tag_post.tag_id=tag.id
JOIN post ON tag_post.post_id=post.id
WHERE tag.tag='mysql';
```

```sql
SELECT * FROM tag WHERE tag='mysql';
SELECT * FROM tag_post WHERE tag_id=1234;
SELECT * FROM post WHERE post.id IN (123,456,567,9098,8904);
```

# 三、存储引擎

## InnoDB

是 MySQL 默认的事务型存储引擎，只有在需要它不支持的特性时，才考虑使用其它存储引擎。

实现了四个标准的隔离级别，默认级别是可重复读（REPEATABLE READ）。在可重复读隔离级别下，通过多版本并发控制（MVCC）+ Next-Key Locking 防止幻影读。

主索引是聚簇索引，在索引中保存了数据，从而避免直接读取磁盘，因此对查询性能有很大的提升。

内部做了很多优化，包括从磁盘读取数据时采用的可预测性读、能够加快读操作并且自动创建的自适应哈希索引、能够加速插入操作的[插入缓冲](https://zhuanlan.zhihu.com/p/39812854)等。

支持真正的在线热备份。其它存储引擎不支持在线热备份，要获取一致性视图需要停止对所有表的写入，而在读写混合场景中，停止写入可能也意味着停止读取。

## MyISAM

设计简单，数据以紧密格式存储。对于只读数据，或者表比较小、可以容忍修复操作，则依然可以使用它。

提供了大量的特性，包括压缩表、空间数据索引等。

不支持事务。

不支持行级锁，只能对整张表加锁，读取时会对需要读到的所有表加共享锁，写入时则对表加排它锁。但在表有读取操作的同时，也可以往表中插入新的记录，这被称为并发插入（CONCURRENT INSERT）。

可以手工或者自动执行检查和修复操作，但是和事务恢复以及崩溃恢复不同，可能导致一些数据丢失，而且修复操作是非常慢的。

如果指定了 DELAY_KEY_WRITE 选项，在每次修改执行完成时，不会立即将修改的索引数据写入磁盘，而是会写到内存中的键缓冲区，只有在清理键缓冲区或者关闭表的时候才会将对应的索引块写入磁盘。这种方式可以极大的提升写入性能，但是在数据库或者主机崩溃时会造成索引损坏，需要执行修复操作。

## 比较

- 事务：InnoDB 是事务型的，可以使用 Commit 和 Rollback 语句。

- 并发：MyISAM 只支持表级锁，而 InnoDB 还支持行级锁。

- 外键：InnoDB 支持外键。

- 备份：InnoDB 支持在线热备份。

- 崩溃恢复：MyISAM 崩溃后发生损坏的概率比 InnoDB 高很多，而且恢复的速度也更慢。

- 其它特性：MyISAM 支持压缩表和空间数据索引。

# 四、数据类型

## 整型

TINYINT, SMALLINT, MEDIUMINT, INT, BIGINT 分别使用 8, 16, 24, 32, 64 位存储空间，一般情况下越小的列越好。

INT(11) 中的数字只是规定了交互工具显示字符的个数，对于存储和计算来说是没有意义的。

## 浮点数

FLOAT 和 DOUBLE 为浮点类型，DECIMAL 为高精度小数类型。CPU 原生支持浮点运算，但是不支持 DECIMAl 类型的计算，因此 DECIMAL 的计算比浮点类型需要更高的代价。

FLOAT、DOUBLE 和 DECIMAL 都可以指定列宽，例如 DECIMAL(18, 9) 表示总共 18 位，取 9 位存储小数部分，剩下 9 位存储整数部分。

## 字符串

主要有 CHAR 和 VARCHAR 两种类型，一种是定长的，一种是变长的。

VARCHAR 这种变长类型能够节省空间，因为只需要存储必要的内容。但是在执行 UPDATE 时可能会使行变得比原来长，当超出一个页所能容纳的大小时，就要执行额外的操作。MyISAM 会将行拆成不同的片段存储，而 InnoDB 则需要分裂页来使行放进页内。

在进行存储和检索时，会保留 VARCHAR 末尾的空格，而会删除 CHAR 末尾的空格。

## 时间和日期

MySQL 提供了两种相似的日期时间类型：DATETIME 和 TIMESTAMP。

### 1. DATETIME

能够保存从 1000 年到 9999 年的日期和时间，精度为秒，使用 8 字节的存储空间。

它与时区无关。

默认情况下，MySQL 以一种可排序的、无歧义的格式显示 DATETIME 值，例如“2008-01-16 22<span>:</span>37<span>:</span>08”，这是 ANSI 标准定义的日期和时间表示方法。

### 2. TIMESTAMP

和 UNIX 时间戳相同，保存从 1970 年 1 月 1 日午夜（格林威治时间）以来的秒数，使用 4 个字节，只能表示从 1970 年到 2038 年。

它和时区有关，也就是说一个时间戳在不同的时区所代表的具体时间是不同的。

MySQL 提供了 FROM_UNIXTIME() 函数把 UNIX 时间戳转换为日期，并提供了 UNIX_TIMESTAMP() 函数把日期转换为 UNIX 时间戳。

默认情况下，如果插入时没有指定 TIMESTAMP 列的值，会将这个值设置为当前时间。

应该尽量使用 TIMESTAMP，因为它比 DATETIME 空间效率更高。

# 五、切分

## 水平切分

水平切分又称为 Sharding，它是将同一个表中的记录拆分到多个结构相同的表中。

当一个表的数据不断增多时，Sharding 是必然的选择，它可以将数据分布到集群的不同节点上，从而缓存单个数据库的压力。

<div align="center"> <img src="https://cs-notes-1256109796.cos.ap-guangzhou.myqcloud.com/63c2909f-0c5f-496f-9fe5-ee9176b31aba.jpg" width=""> </div><br>

## 垂直切分

垂直切分是将一张表按列切分成多个表，通常是按照列的关系密集程度进行切分，也可以利用垂直切分将经常被使用的列和不经常被使用的列切分到不同的表中。

在数据库的层面使用垂直切分将按数据库中表的密集程度部署到不同的库中，例如将原来的电商数据库垂直切分成商品数据库、用户数据库等。

<div align="center"> <img src="https://cs-notes-1256109796.cos.ap-guangzhou.myqcloud.com/e130e5b8-b19a-4f1e-b860-223040525cf6.jpg" width=""> </div><br>

## Sharding 策略

- 哈希取模：hash(key) % N；
- 范围：可以是 ID 范围也可以是时间范围；
- 映射表：使用单独的一个数据库来存储映射关系。

## Sharding 存在的问题

### 1. 事务问题

使用分布式事务来解决，比如 XA 接口。

### 2. 连接

可以将原来的连接分解成多个单表查询，然后在用户程序中进行连接。

### 3. ID 唯一性

- 使用全局唯一 ID（GUID）
- 为每个分片指定一个 ID 范围
- 分布式 ID 生成器 (如 Twitter 的 Snowflake 算法)

# 六、复制

## 主从复制

主要涉及三个线程：binlog 线程、I/O 线程和 SQL 线程。

-   **log dump 线程**  ：负责将主服务器上的二进制日志（Binary log）发给从服务器。
-   **I/O 线程**  ：负责接受主服务器发过来的二进制日志，并写入从服务器的中继日志（Relay log）。
-   **SQL 线程**  ：负责读取中继日志，解析出主服务器已经执行的数据更改并在从服务器中重放（Replay）。

<div align="center"> <img src="https://cs-notes-1256109796.cos.ap-guangzhou.myqcloud.com/master-slave.png" width=""> </div><br>

## 读写分离

主服务器处理写操作以及实时性要求比较高的读操作，而从服务器处理读操作。

读写分离能提高性能的原因在于：

- 主从服务器负责各自的读和写，极大程度缓解了锁的争用；
- 从服务器可以使用 MyISAM，提升查询性能以及节约系统开销；
- 增加冗余，提高可用性。

读写分离常用代理方式来实现，代理服务器接收应用层传来的读写请求，然后决定转发到哪个服务器。

<div align="center"> <img src="https://cs-notes-1256109796.cos.ap-guangzhou.myqcloud.com/master-slave-proxy.png" width=""> </div><br>

## 过期读

主从延迟会导致在从库上会读到系统的一个过期状态。

### 1.  强制走主库方案

强制走主库方案其实就是，将查询请求做分类。
- 对于必须要拿到最新结果的请求，强制将其发到主库上。比如卖家发布商品，马上要返回主页面，看商品是否发布成功。
- 对于可以读到旧数据的请求，才将其发到从库上。比如买家来逛商铺页面，就算晚几秒看到最新发布的商品，也是可以接受的。

### 2.  sleep 方案

主库更新后，读从库之前先 sleep 一下。方案不太精确。

### 3. 判断主备无延迟方案

### 4.  配合 semi-sync 方案

### 5.  等主库位点方案

### 6.  等 GTID 方案


# 七、常用参数

```sql
# 最大连接数，默认151
max_connections  

# 连接不活动的超时时间，默认8小时
wait_timeout  

# 事务隔离级别，默认 REPEATABLE-READ
transaction_isolation

# binlog同步模式，默认为1，同步写磁盘
sync_binlog   

# binlog的格式也有三种：STATEMENT，ROW，MIXED，默认是 ROW
binlog_format  

# 每次提交事务的时候，事务的redo log要持久化到磁盘的策略，默认为1，每次都同步到磁盘
innodb_flush_logs_at_trx_commit  

# 并发线程数，默认0不受限制
innodb_thread_concurrency 

# 磁盘的iops
innodb_io_capacity  

# 默认128M,建议设置成可用物理内存的 60%~80%。
innodb_buffer_pool_size 

# 缓存池的实例数
innodb_buffer_pool_instances 

# redo log buffer大小，默认16M
innodb_log_buffer_size  

# 安全模式，开启后避免update和delete没有加上where和limit的操作
sql_safe_updates   

# 慢查询，默认OFF
slow_query_log

# 慢查询阈值，默认10s
long_query_time  

# 开启后，不经过索引的查询也都会记录到慢日志，默认关闭
log_queries_not_using_indexes

# 开启慢查询
set global slow_query_log=1;  
```
# 八、问题

### 1. 一个查询语句的执行流程是怎么样的？

一条查询语句的执行过程一般是经过:
- **连接器**: 管理连接，权限验证
- **分析器**: 词法分析，语法分析
- **优化器**:  生成执行计划，选择索引
- **执行器** 操作引擎，返回结果
- **存储引擎** 存储数据，提供读写接口

### 2. innodb和myisam的区别？

1) Innodb 支持事务， myisam 不支持

2) Innodb 支持行级锁， myisam 不支持

3) innodb 支持外键，myisam不支持

4) Innodb 是索引组织表， myisam 是堆表

5) Innodb不能通过直接拷贝表文件的方法拷贝表到另外一台机器,myisam可以

6) Innodb 通过redo log支持崩溃后的安全恢复，myisam不支持

7) Innodb因为mvcc的原因不保存表的具体行数，myisam保存不带条件的表行数

8) 对带有AUTO_INCREMENT 属性的字段，InnoDB 中必须包含只有该字段的索引，但是在MyISAM表中，可以和其他字段一起建立联合索引

9) DELETE FROM table的时候，Innodb是一行行删除，myisam是先drop表再create

### 3. WAL(Write-Ahead Logging) 的好处？

事务写日志是两阶段提交的:1.redo log prepare 2.写 binlog 3.redo log commit   
实际步骤:  
1) redo log prepare 写到文件系统的 page cache  
2) binlog 写到 page cache  
3) redo log prepare fsyc 这里把已经写完 page cache 的并发事务的 redo log 一起持久化到磁盘  
4) binlog fsync 这里把已经写完 page cache 的并发事务的 binlog 一起持久化到磁盘  
5) redo log 写到 page cache  
 
WAL 机制主要得益于两个方面：
- redo log 和 binlog 都是顺序写，磁盘的顺序写比随机写速度要快
- 组提交机制，可以大幅度降低磁盘的 IOPS 消耗

### 4. 事务是如何通过日志实现的？

事务是通过 redo log 和 binlog 的两阶段提交实现的

比如更新一条数据:
```
1. 引擎把更新操作记录到 redo log 中，此时 redo log 处于 prepare 状态  
2. 执行器生成并写入 binlog 到磁盘  
3. 执行器调用引擎的提交事务接口，引擎把刚刚写入的 redo log 改成提交（commit）状态
```

事务体现在:

- 当在1之后崩溃时，重启恢复发现 redo log 处于prepare 状态同时没有 binlog 就会回滚。备份恢复发现没有binlog，保持了一致。

- 当在2之后崩溃时，重启恢复发现 redo log 处于prepare 状态但是有 binlog 就会把 redo log 改成 commit 状态。备份恢复发现有binlog， 保持了一致。

redo log 和 binlog 都可以用于表示事务的提交状态，而两阶段提交就是让这两个状态保持逻辑上的一致。

### 5. 长事务有什么影响？

可重复读的隔离级别下，当系统里没有比这个回滚日志更早的 read-view 的时候，这个回滚日志就会被删除，否则会造成大量的事务视图和回滚日志。

举个例子：

    早上9点开启了一个事务A（read-view A也创建了），然后在事务里面查询了一个记录R1的一个字段f1的值为1，然后事务停在那里没有 commit。

    过了一会，一个事务B（随之而来的read-view B）也被开启了，它更新了R1.f1的值为2（同时也创建了一个由2到1的回滚日志），这是一个短事务，事务随后就被commit了。

    又过了一会，一个事务C（随之而来的read-view C）也被开启了，它更新了R1.f1的值为3（同时也创建了一个由3到2的回滚日志），这是一个短事务，事务随后就被commit了。

    后面有不断的事务把字段更新......

    到了下午3:00了，长事务A还没有commit，为了保证事务在执行期间看到的数据在前后必须是一致的，那些事务B、C、D等等的视图、回滚日志就必须存在了，这就占用了大量的存储空间。

所以我们应该尽量不要使用长事务。


### 6. 什么情况下设置了索引但无法使用？

1) 以“%(表示任意0个或多个字符)”开头的LIKE语句，模糊匹配

2) OR 语句前后没有同时使用索引

3) 条件字段函数操作, 比如 where id + 1 = 1000

4) 隐式类型转换, 比如 id 是 字符串类型，你 where 条件用了数字 ,字符串和数字做比较的话，是将字符串转换成数字,如果 id 是数字类型，则 where 条件用数字和字符串都可以;

5) 隐式字符编码转换, join 的时候驱动表字符集是 utf8mb4  被驱动表 的字符串是 utf8, 
  MySQL 内部的操作是，先把 utf8 字符串转成 utf8mb4 字符集到被驱动表查询的时候相当于 where CONVERT(traideid USING utf8mb4) 

6) 对于联合索引，必须满足最左前缀原则

7) 优化器估算代价而不选择索引

### 7. join操作的连接字段不上索引会怎么样？

- 可能会多次扫描被驱动表，占用磁盘IO资源
- 判断join条件需要执行M*N次对比（M、N分别是两张表的行数），如果是大表就会占用非常多的CPU资源
- 可能会导致Buffer Pool的热数据被淘汰，影响内存命中率

我们执行语句之前，需要通过理论分析和查看explain结果的方式，确认是否要使用BNL算法。如果确认优化器会使用BNL算法，就需要做优化。优化的常见做
法是，给被驱动表的join字段加上索引，把BNL算法转成BKA算法

### 8. order by怎么工作的？

每个线程有自己的sort_buffer，如果排序的数据超过了 sort_buffer_size ，会用到临时文件和外部排序
所以可以给排序字段加上索引，就不用排序，同时减少扫描次数。

当排序的单行长度超过 max_length_for_sort_data (默认1024)，会采用 rowid 排序,否则采用全字段排序：
- **rowid 排序**：把排序的列和主键 id 放到 sort_buffer 中排好序，再根据主键 id 回表拿数据
- **全字段排序**：把需要的字段都放到 sort_buffer 中，这样排序后就会直接从内存里面返回查询结果了
 
MySQL 的设计思想：如果内存够，就要多利用内存，尽量减少磁盘访问

### 9. SQL语句的优化？

1) 优化insert语句：一次插入多值

2) 应尽量避免在 where 子句中使用!=或<>操作符，否则将引擎放弃使用索引而进行全表扫描

3) 应尽量避免在 where 子句中对字段进行null值判断，否则将导致引擎放弃使用索引而进行全表扫描

4) 优化嵌套查询：子查询可以被更有效率的连接(Join)替代

5) 很多时候用 exists 代替 in 是一个好的选择

### 10. MySQL有哪些日志？

主要有5种日志文件
1. 错误日志(error log)：记录mysql服务的启停时正确和错误的信息，还记录启动、停止、运行过程中的错误信息。
2. 通用日志(general log)：记录建立的客户端连接和执行的语句。
3. 二进制日志(bin log)：记录所有更改数据的语句，可用于数据复制。
4. 慢查询日志(slow log)：记录所有执行时间超过long_query_time的所有查询或不使用索引的查询。
5. 中继日志(relay log)：主从复制时使用的日志。

除了这5种日志，在需要的时候还会创建DDL日志。

[参考](http://blog.51cto.com/gfsunny/1566683)

### 11. drop、delete与truncate的区别？

1. Delete是一行行删除数据的，会生成undo log、 写redo log, binlog，
  可以通过 binlog 恢复,需要确保 binlog_format=row 和 binlog_row_image=FULL,
  delete命令会触发这个表上所有的delete触发器；

2. Truncate删除表中的所有数据，这个操作不能回滚，也不会触发这个表上的触发器，TRUNCATE比delete更快，
  表和索引所占用的空间会恢复到初始大小,binlog 里面就只有一个 truncate/drop 语句

3. Drop命令从数据库中删除表，所有的数据行，索引和权限也会被删除，所有的DML触发器也不会被触发，这个命令也不能回滚。

因此，在不再需要一张表的时候，用drop；在想删除部分数据行时候，用delete；在保留表而删除所有数据的时候用truncate。

如果用 delete 命令删除的数据，你还可以用 Flashback 来恢复, 
如果是 drop和truncate 可以通过全量备份，加增量日志的方式恢复，除了误删除数据的日志外。


# 参考资料

- BaronScbwartz, PeterZaitsev, VadimTkacbenko, 等. 高性能 MySQL[M]. 电子工业出版社, 2013.
- 姜承尧. MySQL 技术内幕: InnoDB 存储引擎 [M]. 机械工业出版社, 2011.
- [面试总被问分库分表怎么办？你可以这样怼他](https://juejin.im/post/5e53aa67f265da573b0da9f6?utm_source=gold_browser_extension)
- [20+ 条 MySQL 性能优化的最佳经验](https://www.jfox.info/20-tiao-mysql-xing-nen-you-hua-de-zui-jia-jing-yan.html)
- [服务端指南 数据存储篇 | MySQL（09） 分库与分表带来的分布式困境与应对之策](http://blog.720ui.com/2017/mysql_core_09_multi_db_table2/ "服务端指南 数据存储篇 | MySQL（09） 分库与分表带来的分布式困境与应对之策")
- [How to create unique row ID in sharded databases?](https://stackoverflow.com/questions/788829/how-to-create-unique-row-id-in-sharded-databases)
- [SQL Azure Federation – Introduction](http://geekswithblogs.net/shaunxu/archive/2012/01/07/sql-azure-federation-ndash-introduction.aspx "Title of this entry.")
- [MySQL 索引背后的数据结构及算法原理](http://blog.codinglabs.org/articles/theory-of-mysql-index.html)
- [MySQL 性能优化神器 Explain 使用分析](https://segmentfault.com/a/1190000008131735)
- [How Sharding Works](https://medium.com/@jeeyoungk/how-sharding-works-b4dec46b3f6)
- [大众点评订单系统分库分表实践](https://tech.meituan.com/dianping_order_db_sharding.html)
- [B + 树](https://zh.wikipedia.org/wiki/B%2B%E6%A0%91)

