# MySQL

## 脏读
    一个事务读取到了其他事务没有提交的数据。
## 不可重复读
    同一个事务前后两次读取的数据内容不同。读取了其他事务提交的变更。
## 幻读
    同一个事务，使用同样的读取操作。得到的记录数不同。

    注意幻读和不可重复读的区别。 一个是数量不同，一个是数量相同但是内容不同。

## 乐观锁（数据库自己实现的）
    数据有版本号。每次更新的时候，数据库根据版本号来判断，只有版本号相同才更新。
    数据库自己实现的
## 悲观锁（程序自己实现的）
    每次请求数据的时候，需要在用户程序内加锁。
    共享锁（read lock） 排他锁() 是悲观锁的不同实现。
    要使用悲观锁，需要关闭autocommit模式。

## 按权限来划分
### 共享锁 (read lock 读锁)
    可以并发读。但是不能修改。只有数据上的 所有的 共享锁都解除了。才能写。
### 排他锁（write lock 写锁）
    锁定后，其他线程不能对记录进行，读写操作。直到当前线程释放锁。

## 按锁定数据量来划分
### 行锁   record locks
### 间隙锁  gap locks
### 表锁   table locks
### 区间锁  next-key locks
    行锁 + 间隙锁 

## mysql 隔离级别

### 读未提交  read uncommitted RN 
    不用，其他回话可以读到本回话内未提交的变动。

### 读已提交 read committed  RC  不可重复读
    大部分数据库默认级别（oracle）。
    其他事务（回话）可以读到本事务（回话）内已经提交的数据变动。 
    读取不加锁。修改时加行锁

    由于MySQL 默认级别是 可重复读 RR 更改事务隔离级别
    SET session transaction isolation level read committed;

### 可重复读 （repeatable read） RR 
    MVCC 技术实现的。innodb默认隔离级别。
    事务期间 查询条件一致的情况下。读取的数据也一致。
    锁使用情况
    如果命中索引使用。
        行锁（防止其他事务update,delete解决 可重复读
        间歇锁（防止其他事务insert解决幻读）。
    如果不命中索引那么全表锁。

### 串行化  
    最严。所有请求由并行化变为串行化。
    每次读写都要获取表级别的共享锁，写加排它锁，读写互相阻塞。

### 读
    快照读
    select xx from table where :  快照读 不加锁
    当前读
    select ... lock in share mode: 会加共享锁(Shared Locks)
    select ... for update: 会加排它锁




