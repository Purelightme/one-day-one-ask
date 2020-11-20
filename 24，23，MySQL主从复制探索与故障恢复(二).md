接上篇 [MySQL主从复制探索与故障恢复(二)](http://www.baidu.com) 。

#### 从库执行写操作会发生什么？

先在 slave1 上执行：

```sql
insert into t1(name,age) values('slave1',10);
```

现在 slave1 上的 test01.t1 表数据是：

```sql
mysql> select * from t1;
+----+--------+------+
| id | name   | age  |
+----+--------+------+
|  1 | zc     |   20 |
|  2 | ls     |   17 |
|  3 | ww     |   15 |
|  4 | hl     |   31 |
|  5 | slave1 |   10 |
+----+--------+------+
5 rows in set (0.01 sec)
```

然后，切到 master 执行 insert 语句：

```sql
insert into t1(name,age) values('master',40);
```

