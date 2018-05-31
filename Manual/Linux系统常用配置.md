# Linux系统常用配置

记录一些使用过的命令与配置，方便查询。

# 1. 查看系统版本

```shell
> lsb_release -a
```

# 2. 安装JDK、maven

```shell
> vi /etc/profile  # 编辑文件，末尾追加

export JAVA_HOME=/usr/local/jdk1.8.0_161
export PATH=$JAVA_HOME/bin:$PATH
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar

export MAVEN_HOME=/usr/local/apache-maven-3.5.0
export PATH=${MAVEN_HOME}/bin:${PATH}

> source /etc/profile  # 使配置立即生效
```

# 3. 设置网络

## 3.1 SUSE设置网络

### 3.1.1 设置IP

```shell
> ifconfig eth0 192.168.51.51 netmask 255.255.255.0  # 即时生效

> vi /etc/sysconfig/network/ifcfg-eth0  # 编辑文件（文件名对应到网卡），重启生效
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
### 3.1.2 设置网关

```shell
> route add default gw 192.168.51.254  # 即时生效

> vi /etc/sysconfig/network/routes  # 重启生效
default 192.168.51.254 - - 
```

### 3.1.3 设置DNS

```shell
> vi /etc/resolv.conf  # 重启生效，编辑文件，末尾追加
nameserver 8.8.8.8
nameserver 8.8.4.4
```
### 3.1.4 重启网络

```shell
> rcnetwork restart
> service network restart
> /etc/init.d/network restart
```
## 3.2 CentOS7设置网络 

> 参考并整理自：[Centos7网络配置 - CSDN博客](https://blog.csdn.net/gebitan505/article/details/54584213)

### 3.2.1 设置网络（不通过网络管理器） 

```shell
> ip addr # 查看IP地址，查看网卡名称，一般为ens开头
> cd /etc/sysconfig/network-scripts
> vi ifcfg-ens160 # 修改对应网卡的配置文件
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO="static" # 静态
IPADDR=168.33.51.219 # IP
NETMASK=255.255.0.0 # 掩码
GATEWAY=168.33.50.254 # 网关
NM_CONTROLLED=no # 表示该接口将通过该配置文件进行设置，而不是通过网络管理器进行管理
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=ens33
UUID=1a11becc-4f01-411b-b5d7-a8ada907bd63
DEVICE=ens33
ONBOOT=yes # 开机启动
```

### 3.2.2 设置DNS

```shell
> vi /etc/NetworkManager/NetworkManager.conf
[main]
plugins=ifcfg-rh
dns=none
> vi /etc/resolv.conf # 设置DNS
search localdomain
nameserver 202.96.134.133
nameserver 8.8.8.8
```

### 3.2.3 重启网络服务

```shell
> systemctl restart network.service # 重启网络服务
```

# 4. 关闭防火墙

```shell
> rcSuSEfirewall2 stop  # SUSE 方法一
---
> service SuSEfirewall2_setup stop  # SUSE 方法二
> service SuSEfirewall2_init  stop  # SUSE 方法二
---
> service iptables stop  # centOS6 临时关闭
> chkconfig iptables off  # centOS6 禁止开机启动
---
> systemctl stop firewalld  # centOS7 临时关闭
> systemctl disable firewalld  # centOS7 禁止开机启动
```

# 5. 常用命令

```shell
> sed -i 's/\r//g' path/to/file  # 删除掉文件中所有的'\r'，等效于vim中set ff=unix
```



