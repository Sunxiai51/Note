# Mycat-web安装手册

本文档记录了安装Mycat-web-1.0的过程，附带了一些必要文件的下载链接。

[TOC]

> 参考文档：
>
> - [官方文档 - Lepus(天兔)数据库监控系统在线手册](http://www.lepus.cc/manual/)
> - [天兔(Lepus)数据库监控系统快速安装部署 - 51CTO](http://blog.51cto.com/suifu/1770493)
>
> 编写时间：2018-6-1 09:18:13

## 0. 安装需求

本文档使用 `CentOS-7-x86_64-Minimal-1804` 系统部署，需要jdk1.7+。

> CentOS-7下载：[CentOS-7-x86_64-Minimal-1804.iso](http://isoredirect.centos.org/centos/7/isos/x86_64/CentOS-7-x86_64-Minimal-1804.iso)

Mycat-web的运行依赖于Zookeeper，需要先安装并启动Zookpeer后再启动Mycat-web。

## 1. 安装Zookeeper

> Zookeeper下载：[zookeeper-3.4.12.tar.gz](http://mirrors.shu.edu.cn/apache/zookeeper/zookeeper-3.4.12/zookeeper-3.4.12.tar.gz)

```shell
> tar -zxvf zookeeper-3.4.12.tar.gz -C /usr/local/
> cd /usr/local/zookeeper-3.4.12/
> cd conf
> cp zoo_sample.cfg zoo.cfg

# 安装完成，启动
> cd /usr/local/zookeeper-3.4.12/bin/
> ./zkServer.sh start
```

## 2. 安装Mycat-web 

> Mycat-web-1.0下载：[Mycat-download · GitHub](https://github.com/MyCATApache/Mycat-download/blob/master/mycat-web-1.0/Mycat-web-1.0-SNAPSHOT-20160617163048-linux.tar.gz)

```shell
> tar -xvf Mycat-web-1.0-SNAPSHOT-20160617163048-linux.tar.gz -C /usr/local/
> cd /usr/local/mycat-web/mycat-web/WEB-INF/classes
> vi mycat.properties
zookeeper=127.0.0.1:2181

# 安装完成，启动
> cd /usr/local/mycat-web/
> ./start.sh &
```

启动完成后访问地址：http://{这里输入服务器IP地址}:8082/mycat/。