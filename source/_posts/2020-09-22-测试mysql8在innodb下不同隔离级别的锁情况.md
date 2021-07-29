---
title: 测试mysql8在innodb下不同隔离级别的锁情况
date: 2020-09-22 10:26:55
categories: [mysql,lock]
tags: [mysql,lock]
---
# 目标
理解mysql在不同隔离级别下的锁情况，用到mysql8.0是因为8.0可以直接看锁的情况，8.0之前不可以。

# 准备测试数据

desc看表情况：

```
mysql> desc student;
+-------+-------------+------+-----+---------+-------+
| Field | Type        | Null | Key | Default | Extra |
+-------+-------------+------+-----+---------+-------+
| sno   | int(32)     | NO   | PRI | NULL    |       |
| name  | varchar(32) | NO   |     | NULL    |       |
+-------+-------------+------+-----+---------+-------+
2 rows in set (0.00 sec)
```

看表中的值：

```
mysql> select * from student;
+-----+------+
| sno | name |
+-----+------+
|   1 | 1    |
|   2 | 2    |
|   3 | 3    |
+-----+------+
3 rows in set (0.00 sec)
```

# RR隔离级别
在RR隔离级别下，据说mysql解决了幻读：

mysql客户端1

```
mysql> insert into student values(4,'4');
Query OK, 1 row affected (0.00 sec)

mysql> select * from student;
+-----+------+
| sno | name |
+-----+------+
|   1 | 1    |
|   2 | 2    |
|   3 | 3    |
|   4 | 4    |
+-----+------+
4 rows in set (0.00 sec)
```

mysql客户端2

```
mysql> select * from student;
+-----+------+
| sno | name |
+-----+------+
|   1 | 1    |
|   2 | 2    |
|   3 | 3    |
+-----+------+
3 rows in set (0.00 sec)
```

mysql官方8.0的文档有说明这个问题：https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html

* REPEATABLE READ

This is the default isolation level for InnoDB. Consistent reads within the same transaction read the snapshot established by the first read. This means that if you issue several plain (nonlocking) SELECT statements within the same transaction, these SELECT statements are consistent also with respect to each other. See Section 15.7.2.3, “Consistent Nonlocking Reads”.

For locking reads (SELECT with FOR UPDATE or FOR SHARE), UPDATE, and DELETE statements, locking depends on whether the statement uses a unique index with a unique search condition, or a range-type search condition.

* For a unique index with a unique search condition, InnoDB locks only the index record found, not the gap before it.

* For other search conditions, InnoDB locks the index range scanned, using gap locks or next-key locks to block insertions by other sessions into the gaps covered by the range. For information about gap locks and next-key locks, see Section 15.7.1, “InnoDB Locking”.

渣翻译：

这是InnoDB默认的隔离级别。<font color='red'>同一事务中的一致性阅读将读取第一次建立的快照</font>，这意味着如果同时请求多个普通（非锁定）语句，则这些select语句彼此之间是保持一致的。（两个客户端开启了事务，如何事务1先读取了一次建立快照，然后客户端2修改了某些行并提交释放了事务，客户端1在事务中重新再读取将还是得到一开始的快照数据，RC级别则不是这样，后面会提到）

对于锁定读取（select中带有for update 或者 for share），update，delete语句，加锁取决于使用的是带有唯一搜索条件且是唯一索引还是范围类型的搜索条件。

提出问题：如果是带唯一条件的，可是没有索引，或者非唯一索引的话会怎样？测试一下。

### 无索引情况下：

##### mysql客户端1

```
mysql> select * from student where name=2 lock in share mode;
+-----+------+
| sno | name |
+-----+------+
|   2 | 2    |
+-----+------+
1 row in set (0.00 sec)

mysql> SELECT TRX_ID FROM INFORMATION_SCHEMA.INNODB_TRX  WHERE TRX_MYSQL_THREAD_ID = CONNECTION_ID();
+-----------------+
| TRX_ID          |
+-----------------+
| 281479840930344 |
+-----------------+
1 row in set (0.00 sec)
```

##### mysql客户端2
```
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> update student set name='1' where sno=1;
^C^C -- query aborted
ERROR 1317 (70100): Query execution was interrupted
mysql> update student set name='1' where sno=2;
^C^C -- query aborted
ERROR 1317 (70100): Query execution was interrupted
mysql> update student set name='1' where sno=3;
^C^C -- query aborted
ERROR 1317 (70100): Query execution was interrupted
mysql> insert into student values(0,'0');
^C^C -- query aborted
ERROR 1317 (70100): Query execution was interrupted
mysql> insert into student values(4,'4');
^C^C -- query aborted
ERROR 1317 (70100): Query execution was interrupted
mysql> insert into student values(11,'11');
^C^C -- query aborted
ERROR 1317 (70100): Query execution was interrupted
```

##### mysql客户端3
```
mysql> SELECT * FROM performance_schema.data_locks\G;
*************************** 1. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 4864219688:1060:140371482961224
ENGINE_TRANSACTION_ID: 281479840930344
            THREAD_ID: 48
             EVENT_ID: 103
        OBJECT_SCHEMA: test
          OBJECT_NAME: student
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: NULL
OBJECT_INSTANCE_BEGIN: 140371482961224
            LOCK_TYPE: TABLE
            LOCK_MODE: IS
          LOCK_STATUS: GRANTED
            LOCK_DATA: NULL
*************************** 2. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 4864219688:3:4:1:140371501798424
ENGINE_TRANSACTION_ID: 281479840930344
            THREAD_ID: 48
             EVENT_ID: 103
        OBJECT_SCHEMA: test
          OBJECT_NAME: student
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: PRIMARY
OBJECT_INSTANCE_BEGIN: 140371501798424
            LOCK_TYPE: RECORD
            LOCK_MODE: S
          LOCK_STATUS: GRANTED
            LOCK_DATA: supremum pseudo-record
*************************** 3. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 4864219688:3:4:2:140371501798424
ENGINE_TRANSACTION_ID: 281479840930344
            THREAD_ID: 48
             EVENT_ID: 103
        OBJECT_SCHEMA: test
          OBJECT_NAME: student
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: PRIMARY
OBJECT_INSTANCE_BEGIN: 140371501798424
            LOCK_TYPE: RECORD
            LOCK_MODE: S
          LOCK_STATUS: GRANTED
            LOCK_DATA: 1
*************************** 4. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 4864219688:3:4:3:140371501798424
ENGINE_TRANSACTION_ID: 281479840930344
            THREAD_ID: 48
             EVENT_ID: 103
        OBJECT_SCHEMA: test
          OBJECT_NAME: student
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: PRIMARY
OBJECT_INSTANCE_BEGIN: 140371501798424
            LOCK_TYPE: RECORD
            LOCK_MODE: S
          LOCK_STATUS: GRANTED
            LOCK_DATA: 2
*************************** 5. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 4864219688:3:4:4:140371501798424
ENGINE_TRANSACTION_ID: 281479840930344
            THREAD_ID: 48
             EVENT_ID: 103
        OBJECT_SCHEMA: test
          OBJECT_NAME: student
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: PRIMARY
OBJECT_INSTANCE_BEGIN: 140371501798424
            LOCK_TYPE: RECORD
            LOCK_MODE: S
          LOCK_STATUS: GRANTED
            LOCK_DATA: 3
5 rows in set (0.00 sec)
```

从8.0起支持看锁的具体信息，8.0以前不支持，甚至都不存在data_locks这个表。

分析一下，第一个锁的是IS锁，表级锁定，LOCK_DATA是指锁定行的主键值，如果是表锁则为null。后面锁的分别是supremum pseudo-record，1，2，3。关于supremum pseudo-record，参考 https://dev.mysql.com/doc/refman/5.7/en/innodb-locking.html#innodb-next-key-locks。 意思大概是间隙锁定义supremum pseudo-record为最大的上界。这样就大概清楚了原因，使用gap lock和net key lock将所有的行都锁住了，看起来好像是锁表，但是跟锁表不是一回事（真正的锁表是lock tables table_name read或者lock tables table_name write，table_name可以有多个，同时锁定多个表)。这里show的数据是mysql客户端1加锁后就查的，如果在mysql客户端2进行更新操作和插入操作后查会有很多的记录，在commit或者rollback后（事务操作结束）这些多出来的记录会消失。

### 加索引（非唯一索引）

##### mysql客户端1

```
mysql> create index name on student(name);
Query OK, 0 rows affected (0.04 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> desc student;
+-------+-------------+------+-----+---------+-------+
| Field | Type        | Null | Key | Default | Extra |
+-------+-------------+------+-----+---------+-------+
| sno   | int(32)     | NO   | PRI | NULL    |       |
| name  | varchar(32) | NO   | MUL | NULL    |       |
+-------+-------------+------+-----+---------+-------+
2 rows in set (0.00 sec)

mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from student where name=2 lock in share mode;
+-----+------+
| sno | name |
+-----+------+
|   2 | 2    |
+-----+------+
1 row in set (0.00 sec)
```

##### mysql客户端3

```
mysql> SELECT * FROM performance_schema.data_locks\G;
*************************** 1. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 4864219688:1060:140371482961224
ENGINE_TRANSACTION_ID: 281479840930344
            THREAD_ID: 48
             EVENT_ID: 139
        OBJECT_SCHEMA: test
          OBJECT_NAME: student
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: NULL
OBJECT_INSTANCE_BEGIN: 140371482961224
            LOCK_TYPE: TABLE
            LOCK_MODE: IS
          LOCK_STATUS: GRANTED
            LOCK_DATA: NULL
*************************** 2. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 4864219688:3:5:1:140371501798424
ENGINE_TRANSACTION_ID: 281479840930344
            THREAD_ID: 48
             EVENT_ID: 139
        OBJECT_SCHEMA: test
          OBJECT_NAME: student
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: name
OBJECT_INSTANCE_BEGIN: 140371501798424
            LOCK_TYPE: RECORD
            LOCK_MODE: S
          LOCK_STATUS: GRANTED
            LOCK_DATA: supremum pseudo-record
*************************** 3. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 4864219688:3:5:2:140371501798424
ENGINE_TRANSACTION_ID: 281479840930344
            THREAD_ID: 48
             EVENT_ID: 139
        OBJECT_SCHEMA: test
          OBJECT_NAME: student
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: name
OBJECT_INSTANCE_BEGIN: 140371501798424
            LOCK_TYPE: RECORD
            LOCK_MODE: S
          LOCK_STATUS: GRANTED
            LOCK_DATA: '1', 1
*************************** 4. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 4864219688:3:5:3:140371501798424
ENGINE_TRANSACTION_ID: 281479840930344
            THREAD_ID: 48
             EVENT_ID: 139
        OBJECT_SCHEMA: test
          OBJECT_NAME: student
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: name
OBJECT_INSTANCE_BEGIN: 140371501798424
            LOCK_TYPE: RECORD
            LOCK_MODE: S
          LOCK_STATUS: GRANTED
            LOCK_DATA: '2', 2
*************************** 5. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 4864219688:3:5:4:140371501798424
ENGINE_TRANSACTION_ID: 281479840930344
            THREAD_ID: 48
             EVENT_ID: 139
        OBJECT_SCHEMA: test
          OBJECT_NAME: student
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: name
OBJECT_INSTANCE_BEGIN: 140371501798424
            LOCK_TYPE: RECORD
            LOCK_MODE: S
          LOCK_STATUS: GRANTED
            LOCK_DATA: '3', 3
5 rows in set (0.00 sec)
```
发现和无索引的时候差不多。

##### mysql客户端2

```
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> update student set name='4' where sno=1;
^C^C -- query aborted
ERROR 1317 (70100): Query execution was interrupted
mysql> update student set name='4' where sno=2;
^C^C -- query aborted
ERROR 1317 (70100): Query execution was interrupted
mysql> update student set name='4' where sno=3;
^C^C -- query aborted
ERROR 1317 (70100): Query execution was interrupted
mysql> insert into student value(0,'0');
^C^C -- query aborted
ERROR 1317 (70100): Query execution was interrupted
mysql> insert into student value(5,'5');
^C^C -- query aborted
ERROR 1317 (70100): Query execution was interrupted
mysql> insert into student value(11,'11');
^C^C -- query aborted
ERROR 1317 (70100): Query execution was interrupted
```
确实修改和插入失败，类似无索引一样。

到此，我们需要关心的是锁的策略，本次需要关注**record lock**，**gap lock**和**next key lock**。

record lock：记录锁，锁住索引记录本身，如果没有索引就锁聚集索引，如果没有聚集索引，mysql会自动隐示的创建聚集索引。

gap lock：间隙锁，锁住记录之间或者第一个记录之前或者最后一个记录之后，间隙可能跨越单个索引值，多个索引值，甚至空（null）。如果是使用唯一索引来锁定唯一行来锁定的语句，不需要间隙锁定（这不包括搜索条件仅包含多列唯一索引的某些列的情况；在这种情况下，会发生间隙锁定。因为不能确定是唯一的，已测试，确实如此）。个人理解是区间的左开右开。

next key lock：是record lock和gap lock的组合。即左开右闭，除了最后的supremum pseudo-record是左开右开。

# RC隔离级别

RC隔离级别禁用了间隙锁，可能会产生幻读的问题。

##### mysql客户端1

无索引，加共享锁。

```
mysql> desc student;
+-------+-------------+------+-----+---------+-------+
| Field | Type        | Null | Key | Default | Extra |
+-------+-------------+------+-----+---------+-------+
| sno   | int(32)     | NO   | PRI | NULL    |       |
| name  | varchar(32) | NO   |     | NULL    |       |
| t     | varchar(32) | NO   |     | NULL    |       |
+-------+-------------+------+-----+---------+-------+
3 rows in set (0.00 sec)

mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from student where name='2' for share;
+-----+------+---+
| sno | name | t |
+-----+------+---+
|   2 | 2    |   |
+-----+------+---+
1 row in set (0.01 sec)
```

###### mysql客户端3

没有对间隙进行加锁。

```
mysql> SELECT * FROM performance_schema.data_locks\G;
*************************** 1. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 4864220544:1060:140371482963192
ENGINE_TRANSACTION_ID: 281479840931200
            THREAD_ID: 116
             EVENT_ID: 38
        OBJECT_SCHEMA: test
          OBJECT_NAME: student
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: NULL
OBJECT_INSTANCE_BEGIN: 140371482963192
            LOCK_TYPE: TABLE
            LOCK_MODE: IS
          LOCK_STATUS: GRANTED
            LOCK_DATA: NULL
*************************** 2. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 4864220544:3:4:19:140371501803032
ENGINE_TRANSACTION_ID: 281479840931200
            THREAD_ID: 116
             EVENT_ID: 38
        OBJECT_SCHEMA: test
          OBJECT_NAME: student
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: PRIMARY
OBJECT_INSTANCE_BEGIN: 140371501803032
            LOCK_TYPE: RECORD
            LOCK_MODE: S,REC_NOT_GAP
          LOCK_STATUS: GRANTED
            LOCK_DATA: 2
2 rows in set (0.00 sec)
```

从锁的状态看是只锁了一行的记录，尝试修改和插入操作验证一下

##### mysql客户端2

```
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> update student set name='10' where sno=2;
^C^C -- query aborted
ERROR 1317 (70100): Query execution was interrupted
mysql> update student set name='10' where sno=1;
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0

mysql> update student set name='10' where sno=3;
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0

mysql> insert into student values(4,'4','');
Query OK, 1 row affected (0.00 sec)
```

摘自官方文档

* READ COMMITTED

Each consistent read, even within the same transaction, sets and reads its own fresh snapshot. For information about consistent reads, see Section 15.7.2.3, “Consistent Nonlocking Reads”.

For locking reads (SELECT with FOR UPDATE or FOR SHARE), UPDATE statements, and DELETE statements, InnoDB locks only index records, not the gaps before them, and thus permits the free insertion of new records next to locked records. Gap locking is only used for foreign-key constraint checking and duplicate-key checking.

Because gap locking is disabled, phantom problems may occur, as other sessions can insert new rows into the gaps. For information about phantoms, see Section 15.7.4, “Phantom Rows”.

Only row-based binary logging is supported with the READ COMMITTED isolation level. If you use READ COMMITTED with binlog_format=MIXED, the server automatically uses row-based logging.

Using READ COMMITTED has additional effects:

* For UPDATE or DELETE statements, InnoDB holds locks only for rows that it updates or deletes. Record locks for nonmatching rows are released after MySQL has evaluated the WHERE condition. This greatly reduces the probability of deadlocks, but they can still happen.

* For UPDATE statements, if a row is already locked, InnoDB performs a “semi-consistent” read, returning the latest committed version to MySQL so that MySQL can determine whether the row matches the WHERE condition of the UPDATE. If the row matches (must be updated), MySQL reads the row again and this time InnoDB either locks it or waits for a lock on it.

渣翻译：

<font color='red'>即使在同一个事务中，每个一致性读取都将读取最新的快照。</font>（区别于RR级别下的读取，两个客户端进行事务操作，客户端1读取了快照，客户端2修改某些行并提交释放了锁，客户端重新读取快照会获得最新的快照数据，这里的区别似乎也正是幻读上的问题）

对于加锁的读（select带上for update或者for share），update和delete语句，InnoDB只会锁住索引记录，并不会在前面加上间隙锁，因此允许在锁定的记录周围插入记录。间隙锁定只用在外键约束检查和重复键检查。

由于禁用了间隙锁定，可能幻读的问题，因为其他会话可以插入新的记录到间隙中。

RC级别仅支持基于行的二进制日志。如果你使用RC级别并设置binlog_format=MIXED，服务器将会自动使用基于行的日志记录。

使用RC将会带来额外的影响：
* 对于update或者delete语句，InnoDB只会锁住更新或者删除的行。在MySQL评估where条件后，会释放不匹配的行的record lock（这也是上面结果的原因，另外实际测试中似乎不是先锁，然后根据where条件释放不匹配的行，而是根据where条件锁住匹配的行，也可能是遍历每一条的行记录都是先加锁然后根据where条件选择是否释放锁。个人比较倾向后面的结果，在无索引的情况下，使用mysql客户端1对一行记录不通过索引列查找加X锁，mysql客户端2不通过索引进行另一行的查找加锁（X or S都行），发现客户端2在等待客户端1开启的事务的锁。）。这大大降低了死锁的可能性，但依然可以发生。

* 对于update语句，如果一行早已被锁住，InnoDB会执行“半一致”读取，将最新的提交版本返回给MySQL，以便MySQL能确认是否行符合where条件的update语句。如果行匹配（必须更新），MySQL会再次读取该行同时InnoDB对行进行锁定或者等待对行的锁定（应该是等待对行的锁定释放后就上锁啦）。


对第一个结论，个人持不同的观点，或者说这样的写法容易让人误解，会以为是把所有记录都锁住了，然后根据where条件进行释放锁操作。个人猜测可能是遍历每一条的行记录，然后先加锁，再根据where条件选择是否释放锁（怀疑是where条件后才加锁，根本就没有释放锁的操作，test2验证）。以下测试都是在百万级的数据进行。

test1：在无索引的情况下，session1对一行记录不通过索引列查找加X锁，session2不通过索引进行另一行的查找加锁（X or S都行），发现session2在等待session1开启的事务的锁。而且过程中出现了一个神奇的情况，因为是根据无索引的列进行查找，则会通过cluster index（如果有主键，则cluster index是主键索引）的顺序进行查找，那么如果session1锁的行记录是比较小的记录，session2锁住的行是比较大的记录，那么session2在遍历的过程中就在等待session1锁中的记录，如果在这个过程中如果有session3通过cluster index索引到小于等于session2要锁的行的记录并加锁，这个操作会成功（在session2之后执行），session1释放锁后，session2会等待session3的锁。

test2：在无索引的情况下，对一条行记录进行非索引记录加锁查找，由于是百万级的数据，无索引查找非常的慢，过程中进行打印锁的信息，发现确实是进行了加锁操作，验证了是加锁然后释放锁。
