---
date: 2025/02/21
---

<img src="https://web-ghw-demo.oss-cn-hangzhou.aliyuncs.com/%E3%80%90%E5%93%B2%E9%A3%8E%E5%A3%81%E7%BA%B8%E3%80%91%E5%B0%9A_%E5%87%BA%E6%B0%B4%E8%8A%99%E8%93%89-%E6%8F%92%E7%94%BB.png">

<small>萧瑟秋风今又是，换了人间。</small>

# 锁
说到锁，我们第一时间想到的肯定是Java中的Lock，sync关键字等，当一个线程持有锁时，其他线程无法拿到锁就会阻塞，这样就保证这些线程时串行的，保证数据的安全性。而在MySQL中，我们什么时候会用到锁呢？我们首先要思考什么时候数据会出现不安全。
上文提到的事务的隔离性就使用到了锁机制保证并发访问的时候数据不会出现问题
数据库的事务隔离等级为RR（可重复读）时，在普通的select语句使用了MVCC机制，但在select ... for update或select ... in share mode等语句时会使用锁机制
在MySQL中的所有锁锁住的都是索引，锁住的都是叶子节点
悲观锁与乐观锁
- 悲观锁：假定一定会出现数据并发访问问题，在一开始就直接加锁，保证数据安全
- 乐观锁：假定不会出现数据并发访问问题，一直到修改数据时再采取检查数据是否被修改过
```sql
-- 拿到数据a = 1
select * from table_t where id = 1
-- 修改a = 1的数据，检测是否被修改过
update table_t set a = 3 where id = 1 and a = 1
-- 直接加锁
select * from table_t where id = 1 for update
```
## 全局锁
MySQL中的全局锁就是对整个数据库实例加锁，MySQL提供了一个全局读锁的方法，命令是 Flush tables with read lock (FTWRL)。当我们对整个数据库实例加锁后，后续的更新语句，对表的字段进行修改的语句都会阻塞。
全局锁的典型使用场景就是做全局逻辑备份，但是一旦加了锁，付出的代价是很大的
- 给主库加锁，会导致后续的业务无法执行
- 给从库加锁，就无法接受主库的同步请求，就会出现主从不同步的情况
但我们也没法不加锁，那有没有办法解决呢？还是有的
官方自带的逻辑备份工具是 mysqldump。当 mysqldump 使用参数–single-transaction 的时候，导数据之前就会启动一个事务，来确保拿到一致性视图。而由于 MVCC 的支持，这个过程中数据是可以正常更新的。在数据备份过程中会一直使用这个一致性视图。
## 表锁
MySQL的表级锁有四种

1. 表锁
我们可以通过以下语句来加锁
```sql
-- 表级别的共享锁，也就是读锁；
-- 允许当前会话读取被锁定的表，但阻止其他会话对这些表进行写操作，不阻止读操作。
lock tables t_student read;
-- 表级别的独占锁，也就是写锁；
-- 允许当前会话对表进行读写操作，但阻止其他会话对这些表进行任何操作（读或写）。
lock tables t_stuent write;
```
在执行以上第一句SQL加锁后，本线程也只能读t_student表而不能执行其他操作，其他对该表的写操作会被阻塞，读操作不会阻塞。
所以这个表锁的颗粒度也较大，既会影响本线程又会影响其余线程，所以我们一般不提倡使用。InnoDB提供了颗粒度更低的锁
2. 元数据锁
元数据锁是MySQL隐式加的锁，分为MDL读锁，MDL写锁。
- 当我们对一个表进行CRUD时，会隐式加上MDL读锁
- 当我们对一个表的字段进行修改时，会隐式加上MDL写锁
既然是隐式调用，那么是什么时候释放呢?
是在事务提交后会自动释放，如果事务一直不提交，那么该事务会一直持有该锁
就是因为这个特性，所以我们在对表结构进行变更时要先查看对该表进行的操作，否则会有意想不到的事情发生：
- 现在，我们有一个长事务，事务名为事务A，一直没有提交，执行了select语句，加上了MDL读锁
- 我们再次开启一个新事务，取名为事务B，执行查询语句，由于读读不冲突，所以可以执行查询
- 现在我们在事务B中执行变更表结构的语句，这个操作会去获取MDL写锁，由于读写不共存，所以当前操作会被阻塞
- 接着，我们在事务A中再次执行查询语句，由于上一步的操作，当前查询操作也会被阻塞，后续的大量的查询语句都会被阻塞，对业务产生巨大的影响。
那么为什么会有这样情况出现呢，写锁并没有获取到啊
原因在于当我们的操作阻塞时，申请MDL锁的操作会排成一个队列，而申请写锁的优先级高于申请读锁，所以一旦申请MDL写锁阻塞，后续的CRUD操作也会跟着阻塞
因此，我们建议在执行变更表结构操作前，要先查看当前数据库中的长事务，如果要变更的表有一个长事务没有提交，可以选择延后变更操作或者kill这个长事务。
3. 意向锁
- 在对InnoDB引擎的表里的一些记录加上共享型锁时，会先对这个表加上意向共享锁
- 在对InnoDB引擎的表里的一些记录加上排他型锁时，会先对这个表加上意向排他锁
意向共享锁和意向独占锁是表级锁，不会和行级的共享锁和独占锁发生冲突，而且意向锁之间也不会发生冲突，只会和共享表锁（lock tables ... read）和独占表锁（lock tables ... write）发生冲突。
意向锁的作用在于如果我要对一张表加上独占锁，那么就要看这个表中的一些记录是否有独占锁，这样就要进行全表扫描，效率较慢。但我们有了意向锁，那么在对一张表加上独占锁前，不用去扫描看是否有一些记录加上了独占锁而只需要看这个表是否有意向独占锁即可，大大提高了效率。
所以，意向锁的目的是为了快速判断表里是否有记录被加锁。
4. AUTO_INC
在创建数据库时，我们通常会使用自增主键，使用 AUTO_INCREMENT 来修饰该自增字段
在我们插入数据时，我们可以不指定该字段的值，因为数据库会为这个字段赋一个自增的值，这就是通过 AUTO_INC实现的
AUTO_INC是一种特殊的表锁，并不是在事务提交后才释放，而是在这条语句结束后就会提交
当一个事务持有该锁没有释放时，那么其他事务的插入语句救会被阻塞，这就会影响到性能，但是保证值是递增的
为了在不影响数据的情况下提高性能，MySQL提供了一种轻量级的锁，这种锁在插入数据时，会为一个被AUTO_INCREMENT修饰的字段赋一个自增的值，然后直接释放锁，不用等到整个语句执行完成
InnoDB 存储引擎提供了个 innodb_autoinc_lock_mode 的系统变量，是用来控制选择用 AUTO-INC 锁，还是轻量级的锁。

- 当 innodb_autoinc_lock_mode = 0，就采用 AUTO-INC 锁，语句执行结束后才释放锁
- 当 innodb_autoinc_lock_mode = 2，就采用轻量级锁，申请自增主键后就释放锁，并不需要等语句执行后才释放。
- 当 innodb_autoinc_lock_mode = 1：

普通 insert 语句，自增锁在申请之后就马上释放；
类似 insert … select 这样的批量插入数据的语句，自增锁还是要等语句结束后才被释放；
当 innodb_autoinc_lock_mode = 2 是性能最高的方式，但是当搭配 binlog 的日志格式是 statement 一起使用的时候，在「主从复制的场景」中会发生数据不一致的问题。
举个例子，考虑下面场景：

session A 往表 t 中插入了 4 行数据，然后创建了一个相同结构的表 t2，然后两个 session 同时执行向表 t2 中插入数据。
如果 innodb_autoinc_lock_mode = 2，意味着「申请自增主键后就释放锁，不必等插入语句执行完」。那么就可能出现这样的情况：
- 首先，session B 先插入了两个记录，(1,1,1)、(2,2,2)；
- 然后，session A 来申请自增 id 得到 id=3，插入了（3,5,5)；
- 之后，session B 继续执行，插入两条记录 (4,3,3)、 (5,4,4)。

可以看到，session B 的 insert 语句，生成的 id 不连续。
当「主库」发生了这种情况，binlog 面对 t2 表的更新只会记录这两个 session 的 insert 语句，如果 binlog_format=statement，记录的语句就是原始语句。记录的顺序要么先记 session A 的 insert 语句，要么先记 session B 的 insert 语句。
但不论是哪一种，这个 binlog 拿去「从库」执行，这时从库是按「顺序」执行语句的，只有当执行完一条 SQL 语句后，才会执行下一条 SQL。因此，在从库上「不会」发生像主库那样两个 session 「同时」执行向表 t2 中插入数据的场景。所以，在备库上执行了 session B 的 insert 语句，生成的结果里面，id 都是连续的。这时，主从库就发生了数据不一致。
要解决这问题，binlog 日志格式要设置为 row，这样在 binlog 里面记录的是主库分配的自增值，到备库执行的时候，主库的自增值是什么，从库的自增值就是什么。
所以，当 innodb_autoinc_lock_mode = 2 时，并且 binlog_format = row，既能提升并发性，又不会出现数据一致性问题。
## 行锁
1. Record Lock（记录锁）
Record Lock称为记录锁，锁住的是一条记录。记录锁分为共享锁（S锁）和独占锁（X锁）
```sql
-- 加上独占锁
select * from table_t where id = 1 for update
-- 加上共享锁
select * from table_t where id = 1 lock in share mode
```
当加上共享锁后，其他事务人就可以加上共享锁；加上独占锁后，其他的所有操作都会被阻塞
当事务提交后所有的锁都会被释放

2. Gap Lock（间隙锁）
间隙锁，顾名思义，锁住的是一个间隙，一个区间，目的是为了解决可重复读级别下的幻读问题
当我们对一张表加上id范围为（3，5）的间隙锁后，我们就无法插入id为4的数据。
间隙锁分为X型间隙锁和S型的间隙锁，不同的事务可以同时持有X型间隙锁。因为间隙锁的目的是为了阻塞像某个范围内插入数据的操作，即使不同事务同时持有业务不会发生数据安全问题

3. Next Key Lock（临键锁）
临键锁是记录锁和间隙锁的结合体
当我们对一张表加上范围为（3，5]的临键锁时，我们不仅无法插入ID为4的数据，也无法修改ID为5的数据
因为临键锁是记录锁和间隙锁的结合，所以X型临键锁无法被多个事务共同持有

4. 插入意向锁
当我们向一张表插入数据时，要先判断该插入位置是否被设置了间隙锁，如果发现已经存在间隙锁，就会阻塞插入操作，直到间隙锁被释放。
在插入操作被阻塞后，会生成一个插入意向锁，将锁的状态设置为等待状态（在MySQL中加锁时，是先生成锁结构，然后设置锁的状态，只有当锁的状态为正常状态时才户认为事务获取到了锁）
同样的插入意向锁也是不能被不同的事务同时持有