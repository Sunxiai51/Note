# SSH无密码登录节点配置

> 本文转载并整理自： [Hadoop安装教程_单机/伪分布式配置_Hadoop2.6.0/Ubuntu14.04_给力星](http://www.powerxing.com/install-hadoop/)

环境：ubuntu 14.04

## 一、更新apt
```shell
$ sudo apt-get update
```

## 二、安装SSH

Ubuntu默认安装了SSH client，还需要安装SSH server:

```shell
$ sudo apt-get install openssh-server
```

安装后，可以使用如下命令登陆本机：

```shell
$ ssh localhost
```

此时会有SSH首次登陆提示，输入 yes, 然后按提示输入密码，这样就登陆到本机了。

然后退出ssh:
```shell
$ exit
```

## 三、生成密钥并授权

首先生成公钥，在终端中执行：

```shell
$ cd ~/.ssh               # 如果没有该目录，先执行一次$ ssh localhost
$ rm ./id_rsa*            # 删除之前生成的公匙（如果有）
$ ssh-keygen -t rsa       # 会有提示，一直按回车
$ cat ./id_rsa.pub >> ./authorized_keys		# 加入授权
```

完成后可执行`$ ssh localhost`验证。
