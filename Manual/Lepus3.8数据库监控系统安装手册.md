# Lepus3.8数据库监控系统安装手册

> 本文档参考并整理自：
>
> - Lepus官方文档  http://www.lepus.cc/manual/
> - [天兔(Lepus)数据库监控系统快速安装部署 - 51CTO](http://blog.51cto.com/suifu/1770493)
>
> 编写时间：2018-5-31 15:07:17

## 0. 安装需求

目前天兔系统只测试完善了Centos/RedHat系统的支持。本文档使用 `CentOS-7-x86_64-Minimal-1804` 系统部署

> CentOS-7下载：http://isoredirect.centos.org/centos/7/isos/x86_64/CentOS-7-x86_64-Minimal-1804.iso

需要的核心包如下：

以下软件包只需要部署在监控机即可。被监控机无需部署。

1. MySQL 5.0及以上（必须,用来存储监控系统采集的数据）
2. Apache 2.2及以上 （必须,WEB服务器运行服务器）
3. PHP 5.3以上 （必须,提供WEB界面支持）
4. Python2 （必须,推荐2.6及以上版本,执行数据采集和报警任务,不支持Python3）
5. Python连接和监控数据库的相关驱动模块包：
   - MySQLdb for python （Python连接MySQL的接口，用于监控MySQL，此模块必须安装）
   - cx_oracle for python  （Python连接Oracle的接口，非必须，如果需要监控oracle此模块必须安装）
   - Pymongo for python （Python连接MongoDB的接口，非必须，如果需要监控MongoDB此模块必须安装）
   - redis-py for python （Python连接Redis的接口，非必须，如果需要监控Redis此模块必须安装）

## 1. 安装LAMP基础环境

本章节通过Xampp集成环境包安装配置LAMP(Linux+Apache+MySQL +PHP)基础环境。

> 下载地址：[xampp-linux-x64-1.8.2-5-installer.run](https://jaist.dl.sourceforge.net/project/xampp/XAMPP%20Linux/1.8.2/xampp-linux-x64-1.8.2-5-installer.run)

下载后直接运行即可：

```shell
> chmod +x xampp-linux-x64-1.8.2-5-installer.run
> ./xampp-linux-x64-1.8.2-5-installer.run

# 安装完成后启动
> /opt/lampp/lampp start

# 追加环境变量
> vi /etc/profile
export PATH=$PATH:/opt/lampp/bin/
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/opt/lampp/lib
> source /etc/profile
```

Xampp启动成功后，可以在浏览器输入IP访问页面，可能出现以下禁止远程计算机访问的提示：

> ### Access forbidden!
>
> New XAMPP security concept:
>
> ​	Access to the requested object is only available from the local network.
>
> ​	This setting can be configured in the file "httpd-xampp.conf".

```shell
> vi /opt/lampp/etc/extra/httpd-xampp.conf
# 注释掉：Require local

> vi /opt/lampp/etc/extra/httpd-vhosts.conf
# 删除原有的，改为以下内容：
<VirtualHost*:80>
    AddDefaultCharset UTF-8
    DocumentRoot "/opt/lampp/htdocs"
    ServerName 168.33.51.219
    <Directory"/opt/lampp/htdocs">
        Options FollowSymLinks
        AllowOverride All
        Order allow,deny
        Allow from All
    </Directory>
    ErrorLog"|/usr/local/apache/bin/rotatelogs /home/logs/apache/php_%Y%m%d_error.log86400 480"
    CustomLog"|/usr/local/apache/bin/rotatelogs /home/logs/apache/php_%Y%m%d_access.log86400 480" common
</VirtualHost>
```

附：xampp命令

```shell
> /opt/lampp/xampp --help
Usage: xampp <action>

	start         Start XAMPP (Apache, MySQL and eventually others)
	startapache   Start only Apache
	startmysql    Start only MySQL
	startftp      Start only ProFTPD

	stop          Stop XAMPP (Apache, MySQL and eventually others)
	stopapache    Stop only Apache
	stopmysql     Stop only MySQL
	stopftp       Stop only ProFTPD

	reload        Reload XAMPP (Apache, MySQL and eventually others)
	reloadapache  Reload only Apache
	reloadmysql   Reload only MySQL
	reloadftp     Reload only ProFTPD

	restart       Stop and start XAMPP
	security      Check XAMPP's security

	enablessl     Enable SSL support for Apache
	disablessl    Disable SSL support for Apache

	backup        Make backup file of your XAMPP config, log and data files

	oci8          Enable the oci8 extenssion

	panel         Starts graphical XAMPP control panel
```

## 2. 安装MySQLdb for python 

> 下载地址：https://github.com/farcepest/MySQLdb1

```shell
> unzip MySQLdb1-master.zip
> cd MySQLdb1-master/

> which mysql_config # 查看mysql_config位置
/opt/lampp/bin/mysql_config

> vi site.cfg
mysql_config= /opt/lampp/bin/mysql_config
# 安装相关依赖
> yum install gcc libffi-devel python-devel openssl-devel
> yum install urpmi xterm

> python setup.py build
> python setup.py install
```

## 3. 安装Lepus3.8

### 3.1 下载并导入sql

> 下载地址：http://www.lepus.cc/soft/download/18

```shell
> unzip Lepus.zip # 解压下载文件，这里目录为/home/veee
> cd /home/veee/Lepus_v3.8_beta/

# 创建Lepus需要的库与用户
> mysql
mysql> create database lepus default character set utf8;
mysql> grant select,insert,update,delete,create on lepus.* to 'lepus_user'@'%' identified by '123456';
mysql> flush privileges;
mysql> exit

# 导入sql文件夹里的SQL文件(表结构和数据文件)
> mysql -uroot –p  lepus < sql/lepus_table.sql
> mysql -uroot –p  lepus < sql/lepus_data.sql
```

### 3.2 安装

```shell
> cd /home/veee/Lepus_v3.8_beta/python/
> chmod +x install.sh
> ./install.sh # 默认安装到了/usr/local/lepus
[note] lepus will be install on basedir: /usr/local/lepus
[note] /usr/local/lepus directory does not exist,will be created.
[note] /usr/local/lepus directory created success.
[note] wait copy files.......
[note] change script permission.
[note] create links.
[note] install complete.

# 配置
> cd /usr/local/lepus/
> vi etc/config.ini
###监控机MySQL数据库连接地址###
[monitor_server]
host="127.0.0.1"
port=3306
user="lepus_user"
passwd="123456"
dbname="lepus"
```

### 3.3 启动与停止

```shell
> cd /usr/local/lepus/
> ./lepus --help
lepus help:
support-site:  www.lepus.cc
====================================================================
start        Start lepus monitor server; Command: #lepus start
stop         Stop lepus monitor server; Command: #lepus stop
status       Check lepus monitor run status; Command: #lepus status
```

## 4. 安装WEB管理台

```shell
# 复制PHP文件夹里的文件到Apache对应的网站虚拟目录
> cd /home/veee/Lepus_v3.8_beta/
> cp -rf php/* /opt/lampp/htdocs/

# 修改PHP连接监控服务器的数据库信息
> cd /opt/lampp/htdocs/application/config
> vi database.php # 设置以下值
$db['default']['hostname'] = 'localhost';
$db['default']['port']     = '3306';
$db['default']['username'] = 'lepus_user';
$db['default']['password'] = '123456';
$db['default']['database'] = 'lepus';
$db['default']['dbdriver'] = 'mysql';

# 作上述改动后，可能需要重启apache与Lepus
> /opt/lampp/xampp reloadapache
> /usr/local/lepus/lepus stop
> /usr/local/lepus/lepus start
```

设置好后，可以通过浏览器输入IP地址打开监控界面，登录系统。默认管理员账号密码admin/Lepusadmin。