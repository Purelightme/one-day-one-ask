接上篇 [MySQL主从复制探索与故障恢复(二)](https://www.jianshu.com/p/fb720c108ced) 。

### Master故障了怎么办

如果master故障了，通常的做法是选择一个从库，升级为master，然后将其他从库修改为新的master的从库。尝试一下：

先停掉master：```docker-compose stop master```

我们选择 slave1 作为新的主库：

```sql
mysql> stop slave;
Query OK, 0 rows affected, 1 warning (0.08 sec)

mysql> reset master;
Query OK, 0 rows affected (0.33 sec)

mysql> create user 'new-slave'@'%' identified by 'new-slave';
Query OK, 0 rows affected (0.25 sec)

mysql> grant replication slave on *.* to 'new-slave'@'%';
Query OK, 0 rows affected (0.01 sec)
```

将 slave2 切换为新的 master 的从库：

```sql
mysql> stop slave;
Query OK, 0 rows affected, 1 warning (0.05 sec)

mysql> change master to master_host='slave1',master_user='new-slave',master_password='new-slave',master_auto_position=1,GET_MASTER_PUBLIC_KEY=1;
Query OK, 0 rows affected, 1 warning (0.10 sec)

mysql> start slave;
Query OK, 0 rows affected, 1 warning (0.02 sec)

mysql> show slave status\G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: slave1
                  Master_User: new-slave
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000001
          Read_Master_Log_Pos: 703
               Relay_Log_File: 8512ca36aee5-relay-bin.000002
                Relay_Log_Pos: 918
        Relay_Master_Log_File: mysql-bin.000001
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
          Exec_Master_Log_Pos: 703
              Relay_Log_Space: 1134
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
             Master_Server_Id: 1
                  Master_UUID: 80ea294e-298b-11eb-af5c-0242ac140004
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
           Retrieved_Gtid_Set: 80ea294e-298b-11eb-af5c-0242ac140004:1-2
            Executed_Gtid_Set: 2ea01898-2987-11eb-84e5-0242ac140002:1-18,
80d690da-298b-11eb-bc29-0242ac140003:1-6,
80ea294e-298b-11eb-af5c-0242ac140004:1-2
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
```

去新主库执行写入操作，验证是否可以同步：

```sql
mysql> insert into t1(name,age) values('new-master',10);
Query OK, 1 row affected (0.03 sec)

mysql> select * from t1;
+----+------------+------+
| id | name       | age  |
+----+------------+------+
|  1 | zc         |  100 |
|  2 | ls         |   17 |
|  3 | ww         |   15 |
|  4 | hl         |   31 |
|  5 | master     |   40 |
|  6 | yy         |    6 |
|  7 | mm         |   20 |
|  8 | new-master |   10 |
+----+------------+------+
8 rows in set (0.00 sec)
```

查看 slave2 状态如何：

```sql
mysql> select * from t1;
+----+------------+------+
| id | name       | age  |
+----+------------+------+
|  1 | zc         |  100 |
|  2 | ls         |   17 |
|  3 | ww         |   15 |
|  4 | hl         |   31 |
|  5 | master     |   40 |
|  6 | xx         |   19 |
|  7 | mm         |   20 |
|  8 | new-master |   10 |
+----+------------+------+
8 rows in set (0.01 sec)

mysql> show slave status\G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: slave1
                  Master_User: new-slave
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000001
          Read_Master_Log_Pos: 1005
               Relay_Log_File: 8512ca36aee5-relay-bin.000002
                Relay_Log_Pos: 1220
        Relay_Master_Log_File: mysql-bin.000001
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
          Exec_Master_Log_Pos: 1005
              Relay_Log_Space: 1436
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
             Master_Server_Id: 1
                  Master_UUID: 80ea294e-298b-11eb-af5c-0242ac140004
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
           Retrieved_Gtid_Set: 80ea294e-298b-11eb-af5c-0242ac140004:1-3
            Executed_Gtid_Set: 2ea01898-2987-11eb-84e5-0242ac140002:1-18,
80d690da-298b-11eb-bc29-0242ac140003:1-6,
80ea294e-298b-11eb-af5c-0242ac140004:1-3
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
```

没毛病，数据成功同步过来，而且同步状态正常，切换成功。

如果此时 master 成功”复活“，我们可以按照同样的方式切换主库。

这样虽然解决了实际问题，但是主库故障，只能由人工去处理切换，起码也需要几分钟的时间，在如今互联网如此发达的世界，应用故障几分钟显然是不能接受的。我们需要更快的切换，更智能的切换，有这样的方案吗？必须有！

### 自动故障切换

