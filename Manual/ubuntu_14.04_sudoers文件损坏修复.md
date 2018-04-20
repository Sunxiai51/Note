# ubuntu_14.04 /etc/sudoers文件损坏修复

>本文转载并整理自：[ubuntu14.04 /etc/sudoers文件损坏修复 - CSDN博客](http://blog.csdn.net/github_22672485/article/details/40584173)

/etc/sudoers文件损坏后，会导致无法获取root权限的死循环：`sudo`命令报错，修改该文件又需要root权限。

例如以下几种损坏情况:

1. 文件权限被篡改，sudoers文件不能有任何写权限

```shell
$ sudo echo 'veee'
sudo: /etc/sudoers is world writable  
sudo: no valid sudoers sources found, quitting
sudo: unable to initialize policy plugin
```

2. 语法错误

```shell
$ sudo echo 'veee'
>>> /etc/sudoers: syntax error near line 51 <<<
sudo: parse error in /etc/sudoers near line 51
sudo: no valid sudoers sources found, quitting
sudo: unable to initialize policy plugin
```

3. 文件丢失

```shell
$ sudo echo 'veee'
sudo: unable to stat /etc/sudoers: No such file or directory
sudo: no valid sudoers sources found, quitting
sudo: unable to initialize policy plugin
```

## 修复方案

- 如果为root用户登录，直接修改或新建正确的sudoers文件即可

- 无法使用root用户时，可在系统的**修复模式**下修改

	ubuntu_14.04进入修复模式方法：
	1. 重启/开机，长按`shift`键进入GRUB模式
	2. 选择修复模式*Recovery Mode*，进入Recovery Menu
	3. 选择*Drop to root shell prompt*
	4. 重新挂载`/`目录，获取权限
		```shell
		$ mount -o remount,rw /
		```
	5. 编辑sudoers文件至正确
	6. reboot重启系统

## 附：ubuntu_14.04默认`/etc/sudoers`文件内容

```
#
# This file MUST be edited with the 'visudo' command as root.
#
# Please consider adding local content in /etc/sudoers.d/ instead of
# directly modifying this file.
#
# See the man page for details on how to write a sudoers file.
#
Defaults	env_reset
Defaults	mail_badpass
Defaults	secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"

# Host alias specification

# User alias specification

# Cmnd alias specification

# User privilege specification
root	ALL=(ALL:ALL) ALL

# Members of the admin group may gain root privileges
%admin ALL=(ALL) ALL

# Allow members of group sudo to execute any command
%sudo	ALL=(ALL:ALL) ALL

# See sudoers(5) for more information on "#include" directives:

#includedir /etc/sudoers.d
```
