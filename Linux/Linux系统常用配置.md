# Linux系统常用配置

记录一些使用过的命令与配置，方便查询。

# 1. 查看系统版本

```shell
> lsb_release -a
```

# 2. 安装JDK

这里以安装路径为 `/usr/local/jdk1.8.0_161` 举例。

```shell
> vi /etc/profile  # 编辑文件，末尾追加
export JAVA_HOME=/usr/local/jdk1.8.0_161
export PATH=$JAVA_HOME/bin:$PATH
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
> source /etc/profile  # 立即生效
```

# 3. 设置网络

## 3.1 设置IP

即时生效：

```shell
> ifconfig eth0 192.168.51.51 netmask 255.255.255.0
```

重启生效：

```shell
> vi /etc/sysconfig/network/ifcfg-eth0  # 编辑文件（文件名对应到网卡）
BOOTPROTO='static'  #静态IP
BROADCAST=''  #广播地址
ETHTOOL_OPTIONS=''
IPADDR='192.168.51.51'  #IP地址
NETMASK='255.255.255.0'  #子网掩码
NETWORK=''  #网络地址
REMOTE_IPADDR=''
STARTMODE='auto'  #开机启动网络
USERCONTROL='no'
```
## 3.2 设置网关

即时生效：

```shell
> route add default gw 192.168.51.254
```

重启生效：

```shell
> vi /etc/sysconfig/network/routes  # 编辑文件
default 192.168.51.254 - - 
```
## 3.3 设置DNS

重启生效：

```shell
> vi /etc/resolv.conf  # 编辑文件，末尾追加
nameserver 8.8.8.8
nameserver 8.8.4.4
```
## 3.4 重启网络

```shell
> rcnetwork restart
> service network restart
> /etc/init.d/network restart
```
# 4. 关闭防火墙

```shell
> rcSuSEfirewall2 stop  # SUSE
```
