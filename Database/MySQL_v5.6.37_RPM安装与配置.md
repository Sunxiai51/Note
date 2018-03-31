# MySQL_v5.6.37_RPM安装与配置

### 安装环境

- SUSE Linux Enterprise Server 11 (x86_64)
- gcc version 4.3.4 [gcc-4_3-branch revision 152973] (SUSE Linux) 
- RPM version 4.4.2.3

### 准备文件

```bash
> ll ~
-rw-r--r-- 1 root root  18894156 Mar 31 21:08 MySQL-client-5.6.37-1.el6.x86_64.rpm
-rw-r--r-- 1 root root   3391760 Mar 31 21:08 MySQL-devel-5.6.37-1.el6.x86_64.rpm
-rw-r--r-- 1 root root  57482996 Mar 31 21:08 MySQL-server-5.6.37-1.el6.x86_64.rpm
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

以RPM方式安装MySQL，这里为避免安装过程中报依赖错误，加上 `--force --nodeps` 选项：

```shell
> cd ~
> rpm -ivh MySQL-server-5.6.37-1.el6.x86_64.rpm --force --nodeps
> rpm -ivh MySQL-devel-5.6.37-1.el6.x86_64.rpm --force --nodeps
> rpm -ivh MySQL-client-5.6.37-1.el6.x86_64.rpm --force --nodeps
```

安装完成后，尝试执行命令：

```shell
> /usr/bin/mysql_install_db
> service mysql start
> cat /root/.mysql_secret
# The random password set for the root user at Sat Mar 31 21:12:16 2018 (local time): VNPO4UDEAH2pLUGy
```

尝试登录MySQL：

```shell
> mysql -u root -p
Enter password: VNPO4UDEAH2pLUGy
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 299
Server version: 5.6.37 MySQL Community Server (GPL)

Copyright (c) 2000, 2017, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> 
```

登录成功，安装完成。

笔者这一步没有成功登录，提示"error while loading shared libraries: libtinfo.so.5: cannot open shared object file: No such file or directory"，此时首先查看自己机器上是否存在 `libtinfo.so.5` 文件，通常在 `/usr/lib64` 或 `/usr/lib/` 下。如果本机上没有，可以在其它机器上找到后复制过来， 然后再上述两个位置建立其软链接即可。

## 3. 配置

设置root密码：

```shell
mysql> SET PASSWORD = PASSWORD('123456');
mysql> exit
```

编辑配置文件，设置字符编码，并重启MySQL：

```shell
> cp /usr/share/mysql/my-default.cnf /etc/my.cnf
> vim /etc/my.cnf
[client]
password        = 123456
port            = 3306
default-character-set=utf8

[mysqld]
port            = 3306
character_set_server=utf8
character_set_client=utf8
collation-server=utf8_general_ci
lower_case_table_names=1 # 0：表名区分大小写，1：表名不区分大小写
max_connections=1000 # 最大连接数，默认151，MySQL服务器允许的最大连接数16384
sql_mode=NO_ENGINE_SUBSTITUTION,STRICT_TRANS_TABLES

[mysql]
default-character-set = utf8

> service mysql restart
```

重新登录，设置允许远程登录：

```shell
> mysql -uroot -p123456
mysql> use mysql;
mysql> select host,user,password from user;
+-----------------------+------+-------------------------------------------+
| host                  | user | password                                  |
+-----------------------+------+-------------------------------------------+
| localhost             | root | *6BB4837EB74329105EE4568DDA7DC67ED2CA2AD9 |
| localhost.localdomain | root | *1237E2CE819C427B0D8174456DD83C47480D37E8 |
| 127.0.0.1             | root | *1237E2CE819C427B0D8174456DD83C47480D37E8 |
| ::1                   | root | *1237E2CE819C427B0D8174456DD83C47480D37E8 |
+-----------------------+------+-------------------------------------------+

mysql> update user set password=password('123456') where user='root';
mysql> update user set host='%' where user='root' and host='localhost';
mysql> flush privileges;
mysql> exit
```

设置开机自启动：

```shell
> chkconfig mysql on
> chkconfig --list | grep mysql
mysql                     0:off  1:off  2:on   3:on   4:on   5:on   6:off
```

