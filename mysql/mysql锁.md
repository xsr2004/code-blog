# mysql有哪些锁？如何分类？

按照锁的粒度来分：可分为全局锁，表级锁，页级锁，行级锁

按照锁的策略来分：可分为悲观锁和乐观锁

![image-20240608141810815](D:\picGo\images\image-20240608141810815.png)

# mysql的InnoDB如何实现这些锁？

全局锁，就是Flush tables with read lock或者set global readonly=true

通常用于数据库逻辑备份（mysqldump）

这时要注意只读的危险性：

- 如果主库备份则基本业务瘫痪
- 如果从库备份，则从库不能执行主库同步过来的binlog，导致主从延迟

当然也不能不加锁，因为备份过程中表是一个一个更新的，如果不加锁可能会导致业务上的错误。



表级锁，在innoDB中的实现有用sql手动设置表锁，比如lock table read；或者是DML锁（元数据锁）

但是对于这个前者一般不用，因为innoDB支持行级锁，锁住整个表还是影响比较大。

后者会在对表数据crud时自动加一个DML读锁，对表结构操作时会加一个DML写锁。



# 为什么需要全局读锁(FTWRL)

与FTWRL对应的是set global readonly=true。对于不支持事务的引擎，你只能去用set global。

而set global有很多不方便的点：

- 比如对super没用
- 异常处理时有差异：FTWRL的异常处理是如果客户端异常了，这个全局锁就会释放；而set global的客户端如果异常，数据库会一直保持readonly

# 如何安全地给小表加字段？



# MySQL 的乐观锁和悲观锁了解吗？

乐观锁的基本原理就是认为数据不会经常变动，所以在一个事务准备将数据commit时才会对是否冲突进行检测，如果冲突了，返回错误信息，让操作者去决定如何做。

一般应用于读多写少的操作，因为如果写的操作比较多，会更容易冲突，会引起客户端多次重试

通常可以用时间戳和version版本去实现冲突的检测：

<img src="D:\picGo\images\image-20240418202726479.png" alt="image-20240418202726479" style="zoom:33%;" />



悲观锁的基本原理就是认为每次去拿数据时都很可能被别人修改，所以一旦加上锁就必须处理完成后，再释放锁。

一般应用于并发量不大，数据一致性要求比较高的情况。

mysql中的全局锁，表锁，页锁，行锁都是悲观锁

select....for update也使用悲观锁（排他），并在事务commit或者rollback释放锁。

# 共享锁和排他锁了解吗？

都是悲观锁的思想

共享锁，也叫读锁，该事务读取数据时，不允许其他事务对其修改和操作（但可以select），避免“不可重读”

共享锁可以用select....lock in share mode实现

排他锁，也叫写锁，该事务读取数据时，不允许其他事务对其crud，不允许再加锁。

排他锁就是select...for update。

MySQL InnoDB引擎默认update,delete,insert都会自动给涉及到的数据加上排他锁，



共享锁做了一点“让步”，允许其他事务读这个数据，效率会提高。

但是可能会出现在同一个表的同一行数据update时被其他事务“脏读”。



# mysql意向锁

意向锁可以分为共享和排他

它是一种表级别的锁，所谓”意向“就是事务有“意向”对表进行操作，这时需要对操作的级别进行考虑

- 如果这个操作是对表的操作（比如表加读锁和写锁），那么该事务就可以去直接检查这个表的意向锁，如果存在意向锁，那么该事务就会被阻塞
- 如果没有意向锁呢？就需要去检查每一行的行锁，检查出行锁以后还要判断其是读锁还是写锁，如果是读锁，并且加的也是读锁就可以，否则要阻塞

那么说白了其实意向锁的作用就是：当事务B想对表A进行表级别加锁时，可以直接看表A是否存在意向锁，不必去检查每个页的锁，甚至是行锁。相当于是个“锁的缓存”，解决的是效率问题。

mysql innoDB的意向锁由存储引擎维护，用户无法手动操作，但需要知道的是存储引擎有这样一个优化手段吧。

更为关键的一点：意向锁不会与行级的锁互斥。



也就是如果事务B想对表A进行行级别的加锁时，就不会care意向锁，因为这个操作不是表级的。





# mysql的间隙锁，临键锁，记录锁

三者是innoDB存储引擎为了解决幻读的情况

都是基于索引来实现

记录锁就是给单行记录加锁，间隙锁就是给一个范围（开区间）加锁，临键锁是给一个范围（左开右闭）区域加锁

next-key是核心。

在 InnoDB 默认的隔离级别 REPEATABLE-READ 下，行锁默认使用的是 Next-Key Lock。但是，如果操作的索引是唯一索引或主键，InnoDB 会对 Next-Key Lock 进行优化，将其降级为 Record Lock，



**下面是next-key加锁范围（都加的X锁）**

总结如下：

- 唯一索引（主键或者unique）
  - 等值查询
    - 数据存在，比如查id=3，退化为record lock，只锁住3
    - 数据不存在，比如查id=4，先是(3,4]，又因为没有4，所以找到下一个，为(3,6]，退化为gap lock，所以为(3,6)
  - 范围查询
    - 数据存在，比如id >=3 and id<4，id>=3是(1,3]又因为3存在所以只锁住3，等于record lock 3；id<4，遍历到6，6不符合id<4，退化gap lock(3,6)，所以最后锁住的是 [3,6]
    - 数据不存在，比如id>=4 and id<=5，找到第一个不符合条件的id，退化gap lock。6，即(3,6)
- 非唯一索引
  - 等值查询：1.当查询的记录是存在的，除了会加 Next-key Lock 外，还会额外加间隙锁（规则是向下遍历到第一个不符合条件的值才能停止），也就是会加两把锁
    2.当查询的记录是不存在的，Next-key Lock 会退化成间隙锁（这个规则和唯一索引的等值查询是一样的）
  - 范围查询：和唯一索引范围查询不同的是，非唯一索引的范围查询并不会退化成 Record Lock 或者 Gap Lock。

https://blog.csdn.net/LT11hka/article/details/131392416

这是测试数据，对唯一索引/非唯一索引 和 等值查询/范围查询 的测试数据。

![image-20240607205038371](D:\picGo\images\image-20240607205038371.png)

和b+tree索引的实现 息息相关。

```sql
-- 唯一索引 等值查询 事务1
-- 数据存在
start transaction;
select * from test where id = 3 for update;-- record 3

-- 数据不存在
start transaction;
select * from test where id = 4 for update;-- gap (3,6)

-- 唯一索引 范围查询 测试1 事务1
start transaction;
select * from test where id>=3 and id<4 for update;-- record 3 gap (3,6)
-- 唯一索引 范围查询 测试2 事务1
select * from test where id>=3 and id<6 for update;-- record 3 gap (3,6)
-- 唯一索引 范围查询 测试3 事务1
select * from test where id>=3 and id<13 for update;-- record 3 gap (3,16)


commit;
SHOW TABLE STATUS WHERE Name = 'test';
set @@autocommit = 0;

-- 非唯一索引 等值查询 事务1
-- 数据存在 加俩把锁
start transaction;
select * from test where age=12 for update;-- next-key (10,12] gap (12,15)
-- 数据不存在 退化gap
select * from test where age=13 for update;-- next-key (12,15)

-- 非唯一索引 范围查询 事务1
-- 数据存在
select * from test where age >= 12 and age < 14 for update;-- next-key (10,12] next-key (12,15]
-- 数据不存在
select * from test where age >= 13 and age < 14 for update;-- 实测 next-key (12,15]

commit;
```

```sql

-- 唯一索引 等值查询 事务2 
-- 数据存在
start transaction;
insert test values(2,100,'test');-- 没有阻塞，插入成功，只锁了id=3这一条（record lock）

-- 数据不存在
start transaction;
insert test values(5,100,'test');-- 阻塞，事务1commit后，插入成功，next-key退化为gap，(3,6)
-- 唯一索引 范围查询 测试1 事务2
start transaction;
update test set name='newtest' where id=3;-- 预计阻塞，锁住[3,6)
update test set name='newtest' where id=5;-- 预计阻塞
update test set name='newtest' where id=6;-- 不阻塞，直接更新
update test set name='newtest' where id=16;-- 不阻塞，直接更新
-- 唯一索引 范围查询 测试2 事务2
update test set name='newtest' where id=3;-- 预计阻塞，还是锁住[3,6)
-- 如上
-- 唯一索引 范围查询 测试3 事务2
update test set name='newtest' where id=12;-- 预计阻塞，锁住[3,16) -> 不能对id=3,6,12的记录更新 √
update test set name='newtest' where id=13;-- 没有这个id，update不锁 √
update test set name='newtest' where id=16;-- [3,16)，16不锁，正常更新 √
insert test values(13,100,'test');-- 预计阻塞，锁住[3,16) -> 不能对id在3-16的记录里插入记录（防止幻读）√
insert test values(16,100,'test');-- 预计阻塞，锁住[3,16) -> 不能对id在3-16的记录里插入记录（防止幻读）√
insert test values(6,100,'test');-- 预计先阻塞，但等事务1commit后，会报错，因为id=6的记录已存在 √

commit;

set @@autocommit = 0;


-- 非唯一索引 等值查询 事务1
-- 数据存在 加俩把锁
start transaction;
update test set name='newtest' where age=12;-- 预计阻塞，事务1commit后执行成功 √
insert test(age,`name`) values(15,'newtest');-- 不阻塞，但执行失败，因为索引age=15存在 √
insert test(age,`name`) values(16,'newtest');-- 不阻塞，执行成功 √
-- 数据不存在 退化gap
update test set name='newtest' where age=12;-- 不阻塞，执行成功 √
insert test(age,`name`) values(15,'newtest');-- 不阻塞，执行失败，因为索引age=15存在 √
update test set name='newtest' where age=15; -- 不阻塞，执行成功 √
insert test(age,`name`) values(14,'newtest');-- 阻塞，执行成功 √
commit;

-- 非唯一索引 范围查询 事务1
-- 数据存在
insert test(age,`name`) values(11,'newtest');-- 阻塞，执行成功 √
-- 数据不存在
insert test(age,`name`) values(10,'newtest');-- 不阻塞
insert test(age,`name`) values(11,'newtest');-- 不阻塞
insert test(age,`name`) values(12,'newtest');-- 不阻塞
insert test(age,`name`) values(13,'newtest');-- 阻塞
insert test(age,`name`) values(14,'newtest');-- 阻塞
insert test(age,`name`) values(15,'newtest');-- 阻塞
insert test(age,`name`) values(16,'newtest');-- 不阻塞
commit;






select @@transaction_isolation;
select @@global.transaction_isolation;
```



# innoDB如何实现行级锁





# mysql死锁的情况，如何解决

https://www.cnblogs.com/keme/p/11065025.html#23-%E5%A6%82%E4%BD%95%E5%AE%89%E5%85%A8%E5%9C%B0%E7%BB%99%E5%B0%8F%E8%A1%A8%E5%8A%A0%E5%AD%97%E6%AE%B5

https://juejin.cn/post/6931752749545553933#heading-38

https://juejin.cn/post/6844903666332368909#heading-3

https://juejin.cn/post/6844903666420285454#heading-4

https://jishu.proginn.com/doc/63926452907f0a7e5

