# Mysql

### 并发控制

##### 读写锁
Mysql通过实现一个由两种类型的锁组成的锁系统来进行并发控制，这两种类型的锁通常被称为`共享锁（shared lock）`和`排它锁（exclusive lock）`, 也叫`读锁（read lock）`和`写锁（write lock）`。

##### 锁的概念
读锁是`共享`的，或者说是`互相不阻塞`的。多个用户在同一时刻可以读取同一个资源，而互不干扰。

写锁是`排他`的，一个写锁会`阻塞`其他的写锁和读锁，这是出于安全策略的考虑，只有这样才能确保在既定的时间里，只有一个用户能执行写入，并防止其他用户读取正在写入的同一资源。

##### 锁粒度





##### 隔离级别简单演示
查看 `系统级`的隔离级别 和 `会话级` 的隔离级别
ps:&nbsp;&nbsp;&nbsp;&nbsp;mysql的默认隔离级别是`可重复读（REPEATABLE READ）`
```sh
mysql> select @@global.tx_isolation, @@tx_isolation;
+-----------------------+-----------------+
| @@global.tx_isolation | @@tx_isolation  |
+-----------------------+-----------------+
| REPEATABLE-READ       | REPEATABLE-READ |
+-----------------------+-----------------+
1 row in set, 2 warnings (0.00 sec)
```

使用两个Session，并修改两个会话的隔离级别为`未提交读`。
```
mysql> set session transaction isolation level read uncommitted;
Query OK, 0 rows affected
```
创建演示的数据库
```
mysql> CREATE DATABASE `isolation_level_demo` DEFAULT CHARACTER SET utf8mb4;
Query OK, 1 row affected (0.00 sec)
```

创建演示的用户表
```
mysql> CREATE TABLE `user` (
    -> `id`, int(11), NOT NULL, AUTO_INCREMENT,
    -> `name` varchar(50), NULL,
    -> `balance` int(11) UNSIGNED NOT NULL DEFAULT 0,
    -> PRIMARY KEY (`id`)
    -> );
```

Session A `开启事务`并对 user 表做一次查询。
```
mysql> START TRANSACTION;
Query OK, 0 rows affected
mysql> SELECT * FROM `user`;
+----+--------+---------+
| id | name   | balance |
+----+--------+---------+
|  1 | Janice |    1000 |
|  2 | Ben    |    1000 |
+----+--------+---------+
2 rows in set
```

Session B `开启事务` 并把 Janice 的余额增加 500，暂不提交。
```
mysql> START TRANSACTION;
Query OK, 0 rows affected
mysql> UPDATE `user` SET `balance` = `balance` + 500 WHERE id = 1;
Query OK, 1 row affected
Rows matched: 1  Changed: 1  Warnings: 0
```

Session A 在事务内再进行一次查询。
```
mysql> select * from user;
+----+--------+---------+
| id | name   | balance |
+----+--------+---------+
|  1 | Janice |    1500 |
|  2 | Ben    |    1000 |
+----+--------+---------+
2 rows in set
```
此时发现 Janice 的余额增加了 500。

Session B 进行事务回滚
```
mysql> rollback;
Query OK, 0 rows affected
```

Session A 再次进行查询，发现数据跟最初一致。
```
mysql> select * from user;
+----+--------+---------+
| id | name   | balance |
+----+--------+---------+
|  1 | Janice |    1000 |
|  2 | Ben    |    1000 |
+----+--------+---------+
2 rows in set
```

其中 Session A 事务中读取到的1500 就是为 脏读。

Session A 把 Ben 的余额增加 500，暂不提交。
```
mysql> UPDATE `user` SET `balance` = `balance` + 500 WHERE id = 2;
Query OK, 1 row affected
Rows matched: 1  Changed: 1  Warnings: 0
```

Session B 开启事务，并分别尝试对两条数据进行修改。
```
mysql> START TRANSACTION;
Query OK, 0 rows affected

mysql> UPDATE `user` SET `balance` = `balance` + 500 WHERE id = 2;
1205 - Lock wait timeout exceeded; try restarting transaction

mysql> UPDATE `user` SET `balance` = `balance` + 500 WHERE id = 1;
Query OK, 1 row affected
Rows matched: 1  Changed: 1  Warnings: 0
```

Session B 对 Ben 的余额修改被挂起，直至超时，对 Janice 的余额修改成功。从这里可以看出 Session A 的修改 Ben 的数据行加上了共享锁。

READ-UNCOMMITTED（未提交读）
