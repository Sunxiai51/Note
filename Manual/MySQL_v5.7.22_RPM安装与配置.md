# MySQL_v5.7.22_RPM安装与配置

### 安装环境

- SUSE Linux Enterprise Server 11 (x86_64)
- gcc version 4.3.4 [gcc-4_3-branch revision 152973] (SUSE Linux) 
- RPM version 4.4.2.3

### 准备文件

```bash
> ll ~
-rw-r--r-- 1 7155 31415  26308638 Mar  5 05:24 mysql-community-client-5.7.22-1.sles11.x86_64.rpm
-rw-r--r-- 1 7155 31415    291423 Mar  5 05:24 mysql-community-common-5.7.22-1.sles11.x86_64.rpm
-rw-r--r-- 1 7155 31415   2353790 Mar  5 05:24 mysql-community-libs-5.7.22-1.sles11.x86_64.rpm
-rw-r--r-- 1 7155 31415 176496746 Mar  5 05:25 mysql-community-server-5.7.22-1.sles11.x86_64.rpm
```

## 1. 环境检查

检测系统是否已安装MySQL：

```shell
> rpm -qa | grep -i mysql
MySQL-server-5.0.22-0.i386
```

将已安装版本删除：

```shell
> rpm -ev MySQL-server-5.0.22-0.i386
```

## 2. 安装

```shell
> cd ~
> rpm -ivh mysql-community-common-5.7.22-1.sles11.x86_64.rpm
> rpm -ivh mysql-community-libs-5.7.22-1.sles11.x86_64.rpm
> rpm -ivh mysql-community-client-5.7.22-1.sles11.x86_64.rpm
> rpm -ivh mysql-community-server-5.7.22-1.sles11.x86_64.rpm
```

安装完成后，修改配置文件 `/etc/my.cnf` ：

```shell
> vim /etc/my.cnf
[mysqld]
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
# Disabling symbolic-links is recommended to prevent assorted security risks
symbolic-links=0
log-error=/var/log/mysql/mysqld.log
pid-file=/var/run/mysql/mysqld.pid
port = 3306
character_set_server=utf8
collation-server=utf8_general_ci
lower_case_table_names=1 # 0：表名区分大小写，1：表名不区分大小写
max_connections=1000 # 最大连接数，默认151，MySQL服务器允许的最大连接数16384
sql_mode=NO_ENGINE_SUBSTITUTION,STRICT_TRANS_TABLES
skip-name-resolve

[client]
password = 123456
port = 3306
default-character-set=utf8

[mysql]
default-character-set = utf8
```

启动MySQL：

```shell
> service mysql start
```

获取默认root密码：

```shell
> grep "temporary password" /var/log/mysql/mysqld.log
... [Note] A temporary password is generated for root@localhost: rR,>d1Ide+w,
```

尝试登录MySQL：

```shell
> mysql -u root -p
Enter password: rR,>d1Ide+w,
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 3
Server version: 5.7.22

Copyright (c) 2000, 2018, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>
```

登录成功，安装完成。

## 3. 配置

设置root密码：

```shell
mysql> set global validate_password_policy=0; # 设置简单的密码前需要先修改密码策略
mysql> set global validate_password_length=1;
mysql> SET PASSWORD = PASSWORD('123456');
mysql> exit
```

设置允许远程登录：

```shell
> mysql -uroot -p123456
mysql> use mysql;
mysql> update user set host='%' where user='root';
mysql> flush privileges;
mysql> exit
```

## 4. 创建用户

```shell
mysql> CREATE USER 'test'@'%' IDENTIFIED BY '123456';
mysql> GRANT all privileges ON *.* TO 'test'@'%';
mysql> flush privileges;
```

## 5. 初始化

清空MySQL所有数据，恢复其初始安装状态。

```shell
> service mysql stop  # 停止服务
Shutting down service MySQL:                                                   done
> service mysql status
Checking for service MySQL:                                                    unused
> cd /var/lib/mysql  # 进入数据目录，清空
> rm -rf ./*
> mysqld --install --user=mysql  # 执行install
> ll
total 110716
-rw-r----- 1 mysql mysql      215 Jul 13 08:53 ib_buffer_pool
-rw-r----- 1 mysql mysql 50331648 Jul 13 08:53 ib_logfile0
-rw-r----- 1 mysql mysql 50331648 Jul 13 08:53 ib_logfile1
-rw-r----- 1 mysql mysql 12582912 Jul 13 08:53 ibdata1
> service mysql start  # 启动服务报错，提示数据目录不为空
... [Warning] TIMESTAMP with implicit DEFAULT value is deprecated. Please use --explicit_defaults_for_timestamp server option (see documentation for more details).
... [Warning] 'NO_ZERO_DATE', 'NO_ZERO_IN_DATE' and 'ERROR_FOR_DIVISION_BY_ZERO' sql modes should be used with strict mode. They will be merged with strict mode in a future release.
... [Warning] 'NO_AUTO_CREATE_USER' sql mode was not set.
... [ERROR] --initialize specified but the data directory has files in it. Aborting.
... [ERROR] Aborting

MySQL Database could not initialize data directory:                            unused
> ll
total 110748
-rw------- 1 mysql mysql     1679 Jul 13 08:54 ca-key.pem
-rw-r--r-- 1 mysql mysql     1107 Jul 13 08:54 ca.pem
-rw-r--r-- 1 mysql mysql     1107 Jul 13 08:54 client-cert.pem
-rw------- 1 mysql mysql     1675 Jul 13 08:54 client-key.pem
-rw-r----- 1 mysql mysql      215 Jul 13 08:53 ib_buffer_pool
-rw-r----- 1 mysql mysql 50331648 Jul 13 08:53 ib_logfile0
-rw-r----- 1 mysql mysql 50331648 Jul 13 08:53 ib_logfile1
-rw-r----- 1 mysql mysql 12582912 Jul 13 08:53 ibdata1
-rw------- 1 mysql mysql     1675 Jul 13 08:54 private_key.pem
-rw-r--r-- 1 mysql mysql      451 Jul 13 08:54 public_key.pem
-rw-r--r-- 1 mysql mysql     1107 Jul 13 08:54 server-cert.pem
-rw------- 1 mysql mysql     1679 Jul 13 08:54 server-key.pem
> rm -rf ./*  # 再次清空数据目录
> service mysql start  # 启动服务，成功
Starting service MySQL:                                                        done

# 查询日志文件获取初始root密码，进行配置
```

