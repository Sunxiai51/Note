# MySQL_v5.7.22_主从配置

> 参考并整理自：[MySQL 5.7 Reference Manual](https://dev.mysql.com/doc/refman/5.7/en/)

## 环境

- Master
  - 168.33.131.101
  - SLES  11 (x86_64)
  - MySQL 5.7.22 (root@%: 123456)
- Slave
  - 168.33.131.102
  - SLES 11 (x86_64)
  - MySQL 5.7.22 (root@%: 123456)

## 1. 配置Master

修改Master `/etc/my.cnf` ，添加以下配置：

```shell
# 主从配置
log-bin=mysql-bin # 开启二进制日志
server-id=101 # 设置server-id，一般取IP最后一段
innodb_flush_log_at_trx_commit=1
sync_binlog=1
```

重启mysql：

```shell
master> service mysql restart
```

## 2. Master创建Replication用户

```shell
master> mysql -uroot -p123456
...
mysql> CREATE USER 'repl'@'%' IDENTIFIED BY '123456';
Query OK, 0 rows affected (0.96 sec)

mysql> GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
Query OK, 0 rows affected (0.00 sec)
```

## 3. 加锁并查看日志文件及Position

加锁：

```shell
master> mysql -uroot -p123456
...
mysql> FLUSH TABLES WITH READ LOCK;
Query OK, 0 rows affected (0.00 sec)
```

保持加锁状态，新创建一个session：

```shell
master> mysql -uroot -p123456
...
mysql> SHOW MASTER STATUS;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000001 |    34132 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
```

## 4. 导出Master数据并传输至Slave

导出Master数据至 `~` ：

```shell
master> cd ~
master> mysqldump --all-databases --master-data > dbdump.db
```

通过netcat传输数据，在Slave上开启监听：

```shell
slave> cd ~
slave> netcat -l -p 5151 > dbdump.db
```

发送数据：

```shell
master> netcat -w 1 168.33.131.102 5151 < ~/dbdump.db
```

## 5. 数据导入至Slave

```shell
slave> service mysql stop
slave> mysql -uroot -p123456 < ~/dbdump.db 
```

## 6. 配置Slave

修改Slave `/etc/my.cnf` ，添加以下配置：

```shell
# 主从配置
server-id=102 # 设置server-id，一般取IP最后一段
```

启动mysql并连接至Master：

```shell
slave> service mysql start
slave> mysql -uroot -p123456
...
mysql> CHANGE MASTER TO
    ->  MASTER_HOST='168.33.131.101',
    ->  MASTER_USER='repl',
    ->  MASTER_PASSWORD='123456',
    ->  MASTER_LOG_FILE='mysql-bin.000001',
    ->  MASTER_LOG_POS=34132;
Query OK, 0 rows affected, 2 warnings (0.01 sec)

mysql> START SLAVE;
Query OK, 0 rows affected (0.01 sec)

mysql> show slave status\G
...
```

## 7. 解锁Master

回到Master执行 `FLUSH TABLES WITH READ LOCK` 的session：

```shell
mysql> unlock tables;
Query OK, 0 rows affected (0.00 sec)
```

