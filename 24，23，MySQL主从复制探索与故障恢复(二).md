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

现在 master 上的 test01.t1 表数据是：

```sql
mysql> select * from t1;
+----+--------+------+
| id | name   | age  |
+----+--------+------+
|  1 | zc     |   20 |
|  2 | ls     |   17 |
|  3 | ww     |   15 |
|  4 | hl     |   31 |
|  5 | master |   40 |
+----+--------+------+
5 rows in set (0.00 sec)
```

首先看看 slave2 上的数据和同步状态：

```
数据和状态都正常，符合预期
Slave_IO_Running: Yes
Slave_SQL_Running: Yes
```

再来看看有执行”写入“操作的slave1：

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
5 rows in set (0.00 sec)

mysql> show slave status\G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: master
                  Master_User: slave1
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000004
          Read_Master_Log_Pos: 1128
               Relay_Log_File: ae6b5ad86c4d-relay-bin.000013
                Relay_Log_Pos: 458
        Relay_Master_Log_File: mysql-bin.000004
             Slave_IO_Running: Yes
            Slave_SQL_Running: No
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 1062
                   Last_Error: Could not execute Write_rows event on table test01.t1; Duplicate entry '5' for key 't1.PRIMARY', Error_code: 1062; handler error HA_ERR_FOUND_DUPP_KEY; the event's master log mysql-bin.000004, end_log_pos 1097
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 830
              Relay_Log_Space: 1274
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: NULL
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 1062
               Last_SQL_Error: Could not execute Write_rows event on table test01.t1; Duplicate entry '5' for key 't1.PRIMARY', Error_code: 1062; handler error HA_ERR_FOUND_DUPP_KEY; the event's master log mysql-bin.000004, end_log_pos 1097
  Replicate_Ignore_Server_Ids:
             Master_Server_Id: 100
                  Master_UUID: 2ea01898-2987-11eb-84e5-0242ac140002
             Master_Info_File: mysql.slave_master_info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State:
           Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp: 201119 20:22:49
               Master_SSL_Crl:
           Master_SSL_Crlpath:
           Retrieved_Gtid_Set: 2ea01898-2987-11eb-84e5-0242ac140002:1-15
            Executed_Gtid_Set: 2ea01898-2987-11eb-84e5-0242ac140002:1-14,
80ea294e-298b-11eb-af5c-0242ac140004:1-6
                Auto_Position: 1
         Replicate_Rewrite_DB:
                 Channel_Name:
           Master_TLS_Version:
       Master_public_key_path:
        Get_master_public_key: 1
            Network_Namespace:
1 row in set, 1 warning (0.01 sec)

ERROR:
No query specified

mysql>
```

果然是出现了问题，id为5的主键已经存在，所以Slave_SQL_Running状态为No。

##### 方案1

这时我们尝试手动删除刚刚插入的id为5的那条记录，然后重启复制：

```
mysql> delete from t1 where id = 5;
Query OK, 1 row affected (0.03 sec)

mysql> alter table t1 auto_increment = 5;
Query OK, 0 rows affected (0.16 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> stop slave;
Query OK, 0 rows affected, 1 warning (0.03 sec)

mysql> start slave;
Query OK, 0 rows affected, 1 warning (0.03 sec)

mysql> show slave status\G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: master
                  Master_User: slave1
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000004
          Read_Master_Log_Pos: 1128
               Relay_Log_File: ae6b5ad86c4d-relay-bin.000014
                Relay_Log_Pos: 458
        Relay_Master_Log_File: mysql-bin.000004
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 1128
              Relay_Log_Space: 1274
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 0
               Last_SQL_Error:
  Replicate_Ignore_Server_Ids:
             Master_Server_Id: 100
                  Master_UUID: 2ea01898-2987-11eb-84e5-0242ac140002
             Master_Info_File: mysql.slave_master_info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp:
               Master_SSL_Crl:
           Master_SSL_Crlpath:
           Retrieved_Gtid_Set: 2ea01898-2987-11eb-84e5-0242ac140002:1-15
            Executed_Gtid_Set: 2ea01898-2987-11eb-84e5-0242ac140002:1-15,
80ea294e-298b-11eb-af5c-0242ac140004:1-8
                Auto_Position: 1
         Replicate_Rewrite_DB:
                 Channel_Name:
           Master_TLS_Version:
       Master_public_key_path:
        Get_master_public_key: 1
            Network_Namespace:
1 row in set, 1 warning (0.01 sec)

ERROR:
No query specified

mysql> select * from t1;
+----+--------+------+
| id | name   | age  |
+----+--------+------+
|  1 | zc     |   20 |
|  2 | ls     |   17 |
|  3 | ww     |   15 |
|  4 | hl     |   31 |
|  5 | master |   40 |
+----+--------+------+
5 rows in set (0.00 sec)
```

修复成功！

##### 方案二

先在 slave1 上执行2条写入语句：

```sql
mysql> select * from t1;
+----+--------+------+
| id | name   | age  |
+----+--------+------+
|  1 | zc     |   20 |
|  2 | ls     |   17 |
|  3 | ww     |   15 |
|  4 | hl     |   31 |
|  5 | master |   40 |
+----+--------+------+
5 rows in set (0.01 sec)

mysql> update t1 set age = age + 1 where id = 1;
Query OK, 1 row affected (0.01 sec)
Rows matched: 1  Changed: 1  Warnings: 0

mysql> insert into t1(name,age) values('yy',6);
Query OK, 1 row affected (0.01 sec)

mysql> select * from t1;
+----+--------+------+
| id | name   | age  |
+----+--------+------+
|  1 | zc     |   21 |
|  2 | ls     |   17 |
|  3 | ww     |   15 |
|  4 | hl     |   31 |
|  5 | master |   40 |
|  6 | yy     |    6 |
+----+--------+------+
6 rows in set (0.01 sec)

mysql>
```

再去 master 执行：

```sql
mysql> select * from t1;
+----+--------+------+
| id | name   | age  |
+----+--------+------+
|  1 | zc     |   20 |
|  2 | ls     |   17 |
|  3 | ww     |   15 |
|  4 | hl     |   31 |
|  5 | master |   40 |
+----+--------+------+
5 rows in set (0.00 sec)

mysql> update t1 set age = 100 where id = 1;
Query OK, 1 row affected (0.04 sec)
Rows matched: 1  Changed: 1  Warnings: 0

mysql> insert into t1(name,age) values('xx',19);
Query OK, 1 row affected (0.02 sec)

mysql> select * from t1;
+----+--------+------+
| id | name   | age  |
+----+--------+------+
|  1 | zc     |  100 |
|  2 | ls     |   17 |
|  3 | ww     |   15 |
|  4 | hl     |   31 |
|  5 | master |   40 |
|  6 | xx     |   19 |
+----+--------+------+
6 rows in set (0.00 sec)

mysql>
```

slave2 此时数据自然是：

```sql
mysql> select * from t1;
+----+--------+------+
| id | name   | age  |
+----+--------+------+
|  1 | zc     |  100 |
|  2 | ls     |   17 |
|  3 | ww     |   15 |
|  4 | hl     |   31 |
|  5 | master |   40 |
|  6 | xx     |   19 |
+----+--------+------+
6 rows in set (0.00 sec)
```

查看 slave1 状态：

```sql
mysql> select * from t1;
+----+--------+------+
| id | name   | age  |
+----+--------+------+
|  1 | zc     |  100 |
|  2 | ls     |   17 |
|  3 | ww     |   15 |
|  4 | hl     |   31 |
|  5 | master |   40 |
|  6 | yy     |    6 |
+----+--------+------+
6 rows in set (0.00 sec)

mysql> show slave status\G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: master
                  Master_User: slave1
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000004
          Read_Master_Log_Pos: 1739
               Relay_Log_File: ae6b5ad86c4d-relay-bin.000014
                Relay_Log_Pos: 775
        Relay_Master_Log_File: mysql-bin.000004
             Slave_IO_Running: Yes
            Slave_SQL_Running: No
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 1062
                   Last_Error: Could not execute Write_rows event on table test01.t1; Duplicate entry '6' for key 't1.PRIMARY', Error_code: 1062; handler error HA_ERR_FOUND_DUPP_KEY; the event's master log mysql-bin.000004, end_log_pos 1708
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 1445
              Relay_Log_Space: 1885
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: NULL
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 1062
               Last_SQL_Error: Could not execute Write_rows event on table test01.t1; Duplicate entry '6' for key 't1.PRIMARY', Error_code: 1062; handler error HA_ERR_FOUND_DUPP_KEY; the event's master log mysql-bin.000004, end_log_pos 1708
  Replicate_Ignore_Server_Ids:
             Master_Server_Id: 100
                  Master_UUID: 2ea01898-2987-11eb-84e5-0242ac140002
             Master_Info_File: mysql.slave_master_info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State:
           Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp: 201122 22:05:36
               Master_SSL_Crl:
           Master_SSL_Crlpath:
           Retrieved_Gtid_Set: 2ea01898-2987-11eb-84e5-0242ac140002:1-17
            Executed_Gtid_Set: 2ea01898-2987-11eb-84e5-0242ac140002:1-16,
80ea294e-298b-11eb-af5c-0242ac140004:1-10
                Auto_Position: 1
         Replicate_Rewrite_DB:
                 Channel_Name:
           Master_TLS_Version:
       Master_public_key_path:
        Get_master_public_key: 1
            Network_Namespace:
1 row in set, 1 warning (0.01 sec)

ERROR:
No query specified

mysql>
```

虽然 id 为6的数据插入不进去，但是 id 为1的 update 可以正常执行成功，这时候可以用 sql_slave_skip_counter 跳过事务，试试：

```sql
mysql> stop slave;
Query OK, 0 rows affected, 1 warning (0.01 sec)

mysql> set global sql_slave_skip_counter = 1;
ERROR 1858 (HY000): sql_slave_skip_counter can not be set when the server is running with @@GLOBAL.GTID_MODE = ON. Instead, for each transaction that you want to skip, generate an empty transaction with the same GTID as the transaction
```

看来这个方法在 GTID 模式下不能用，好在提示了我们该怎么做，生成一个与要跳过的事务的GTID相同的空事务。怎么确定发生故障的那个事务的 GTID 是多少？

```sql
Last_Error: Could not execute Write_rows event on table test01.t1; Duplicate entry '6' for key 't1.PRIMARY', Error_code: 1062; handler error HA_ERR_FOUND_DUPP_KEY; the event's master log mysql-bin.000004, end_log_pos 1708
```

这里可以找到 master 的 binlog 相关信息，可以直接去 master 上面去找。

找到对应 GTID 为：2ea01898-2987-11eb-84e5-0242ac140002:17，开始跳过修复：

```sql
mysql> stop slave;
Query OK, 0 rows affected, 2 warnings (0.01 sec)

mysql> set session gtid_next = '2ea01898-2987-11eb-84e5-0242ac140002:17';
Query OK, 0 rows affected (0.00 sec)

mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> commit;
Query OK, 0 rows affected (0.00 sec)

mysql> set session gtid_next = AUTOMATIC;
Query OK, 0 rows affected (0.00 sec)

mysql> start slave;
Query OK, 0 rows affected, 1 warning (0.02 sec)

mysql> show slave status\G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: master
                  Master_User: slave1
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000004
          Read_Master_Log_Pos: 1739
               Relay_Log_File: ae6b5ad86c4d-relay-bin.000015
                Relay_Log_Pos: 458
        Relay_Master_Log_File: mysql-bin.000004
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 1739
              Relay_Log_Space: 1587
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 0
               Last_SQL_Error:
  Replicate_Ignore_Server_Ids:
             Master_Server_Id: 100
                  Master_UUID: 2ea01898-2987-11eb-84e5-0242ac140002
             Master_Info_File: mysql.slave_master_info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp:
               Master_SSL_Crl:
           Master_SSL_Crlpath:
           Retrieved_Gtid_Set: 2ea01898-2987-11eb-84e5-0242ac140002:1-17
            Executed_Gtid_Set: 2ea01898-2987-11eb-84e5-0242ac140002:1-17,
80ea294e-298b-11eb-af5c-0242ac140004:1-10
                Auto_Position: 1
         Replicate_Rewrite_DB:
                 Channel_Name:
           Master_TLS_Version:
       Master_public_key_path:
        Get_master_public_key: 1
            Network_Namespace:
1 row in set, 1 warning (0.01 sec)

ERROR:
No query specified

mysql>
```

然后再去 master 执行写入，看看 slave1 能否成功同步过来：

```
mysql> select * from t1;
+----+--------+------+
| id | name   | age  |
+----+--------+------+
|  1 | zc     |  100 |
|  2 | ls     |   17 |
|  3 | ww     |   15 |
|  4 | hl     |   31 |
|  5 | master |   40 |
|  6 | xx     |   19 |
+----+--------+------+
6 rows in set (0.00 sec)

mysql> insert into t1(name,age) values('mm',20);
Query OK, 1 row affected (0.03 sec)
```

slave1：

```sql
mysql> select * from t1;
+----+--------+------+
| id | name   | age  |
+----+--------+------+
|  1 | zc     |  100 |
|  2 | ls     |   17 |
|  3 | ww     |   15 |
|  4 | hl     |   31 |
|  5 | master |   40 |
|  6 | yy     |    6 |
|  7 | mm     |   20 |
+----+--------+------+
7 rows in set (0.00 sec)
```

成功同步过来了！但是此时 slave1 里面的数据已经跟 master 不一致了，这个在实际操作时候还是要考虑具体情况，能否接受主从数据不一致，以采取对应解决办法。



今天先到这里，改天继续学习~

```2020-11-22```

