# mysql事务的四大特性ACID？

https://www.cnblogs.com/vipstone/p/16422573.html

原子性：要么事务中的操作全部完成，要么全部失败，失败就回滚

隔离性：事务A不会受到事务B干扰

持久性：事务一旦提交，对于数据库的更改是永久的，如果系统奔溃也能恢复最后一次commit的status

上三者若达成，则保证了一致性

# mysql如何保证ACID？

undo log保证原子性：它记录了该事务发生之前的数据；当事务开始修改数据时，会先记录操作前数据到undo log里面，如果事务失败，则根据undo log回滚数据；如果成功，则undo log会在某个点清除

mvcc保证隔离性：**每个事务都有自己的数据版本，根据不同隔离等级，不同的事务看到不同的数据版本**

**每次更新记录时，都会生成一个新的数据版本，它有一个该版本的createTime和expireTime**

**当事务尝试读取记录时，它会看到该事务开始时有效的那个版本。**

redo log保证持久性：当事务进行写操作时，首先会写入redo log，并不会立即修改数据文件。

它是一种物理日志，记录了物理页的数据变化

系统奔溃时，由于之前的数据都在redo log中，所以可以重新加载并读取日志里的数据来恢复数据

# 脏读，不可重复读，幻读是什么？

脏读：事务A执行中，事务B对表数据进行了操作且尚未commit或rollback，此时事务A再次执行时可以看到事务B的这个操作，而如果事务B将其回滚，事务A的这次查看就是”脏读“

本质是**事务A可以看到事务B操作未完成的数据**

不可重复读：事务A读了一个数据，事务B对表数据进行了操作并已经完成，而事务A又读了一下这个数据，发现输出不一致，这就是“不可重复读”

幻读：事务A查了一个范围，事务B对这个范围进行了insert/delete并已完成，而事务A又查了一下这个范围，发现输出不一致，多了或者少了，这就是幻读

不可重复读重点在于update，俩次读到的同一行数据不一样

而幻读的重点在于insert和delete。俩次得到的数据行数不一样

https://cloud.tencent.com/developer/article/1429947



# mysql事务的隔离级别有哪些？分别解决了什么？

读未提交 -> 读已提交 -> 可重复读 -> 串行化

| 隔离级别                 | 脏读 | 不可重复读 | 幻读    |
| ------------------------ | ---- | ---------- | ------- |
| Read Uncommitted读未提交 | √    | √          | √       |
| Read Committed读已提交   | ×    | √          | √       |
| repeatable read可重复读  | ×    | ×          | √（少） |
| Serialzable 可串行化     | ×    | ×          | ×       |

读未提交：事务A可以看到事务B未commit或callback的数据

读未提交：事务A可以看到事务Bcommit或callback后的数据（sql server默认）

可重复读：保证一个事务不会出现“不可重复读”（mysql innodb引擎默认）

串行化：强制事务排序，使之不会发生冲突，效率低

# 不可重复读和幻读的区别

- 不可重复读：多次查询同一条记录，发现里面的值有修改
- 幻读：多次执行查询时，发现记录变多了

幻读是不可重复读的一种，由于实现二者用到的锁不同，才专门区分

如不可重复读主要是update和delete，用到的是记录锁（recond lock）

而幻读主要是insert，为了避免插入新记录，用到的是Next-Key Lock（Record Lock记录锁+Gap Lock间隙锁）

# mysql的事务隔离级别是如何实现的

**MySQL 的隔离级别基于锁和 MVCC 机制共同实现的。**

读未提交：事务读的时候不加锁，写的时候加读锁。

读取已提交&可重复读：readView，每个事务只能看到它能看到的版本

- READ COMMITTED：每次读取数据前都生成一个 ReadView
- REPEATABLE READ ：在第一次读取数据时，也就是事务开始时生成一个 ReadView

串行化：读的时候加读锁，写的时候加写锁，出现冲突就一个接着一个执行

# mvcc了解吗？如何实现的？

多版本并发控制，主要为了解决数据库并发访问，编程语言的事务内存

mysql中的mvcc解决了数据库并发场景的“读-写冲突”。

想实现的概念是：一个数据多个版本，要让读写没有冲突。

mysql中的mvcc的实现是通过 readview四个字段，记录的俩个隐藏列，undo log日志来实现的。

> 实现细节：https://blog.csdn.net/SnailMann/article/details/94724197

# 什么是当前读和快照读

- 当前读：访问数据的最新版本，读取时保证其他并发事务不会修改
- 快照都：不加锁的非阻塞读，注重性能，至于数据的崭新程度，不在乎

# Read View 在 MVCC 里如何工作的？



https://www.xiaolincoding.com/mysql/transaction/mvcc.html#read-view-%E5%9C%A8-mvcc-%E9%87%8C%E5%A6%82%E4%BD%95%E5%B7%A5%E4%BD%9C%E7%9A%84

> 核心是readView的4个字段：自身id，活跃事务id，活跃事务左边界id，右边界id
>
> 核心一句话是：只能读到早于readview创建的commit事务的最新数据

> 记录中俩个隐藏列：trx_id上次修改该行记录的事务，roll_pointer指向旧版本的记录

https://www.xiaolincoding.com/

https://javaguide.cn/

https://pdai.tech/

