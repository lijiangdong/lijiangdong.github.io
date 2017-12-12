---
title:  Linux常用命令
date: 2017-12-12 10:15
category: [Linux]
tags: [CentOS]
comments: true
---

# CentOS修改主机名

1.查看当前主机名

> \#hostname

2.修改命令

> \#hostnamectl set-hostname name

<!--more-->

3.再通过hostname或者hostnamectl status命令查看更改是否生效

> \#hostnamectl status

# 修改root用户密码

>[root@centos ~] \# passwd

# 创建新用户

创建一个用户名
>[root@centos ~]\# adduser lijiangdong

更改用户的密码
>[root@centos ~]\# passwd lijiangdong

# 新用户授权

新创建的用户并不能使用sudo命令，需要给他添加授权。

sudo命令的授权管理是在sudoers文件里
>[root@localhost ~]# sudoers </br>
-bash: sudoers: 未找到命令

找到sudoers命令
>[root@localhost ~]# whereis sudoers</br>
sudoers: /etc/sudoers /etc/sudoers.d /usr/libexec/sudoers.so /usr/share/man/man5/sudoers.5.gz

查看权限
>[root@localhost ~]# ls -l /etc/sudoers </br>
-r--r----- 1 root root 3907 6月  23 03:42 /etc/sudoers
 
只有r权限，添加w权限
>[root@localhost ~]#  chmod -v u+w /etc/sudoers <\br>
mode of "/etc/sudoers" changed from 0440 (r--r-----) to 0640 (rw-r-----)

追加新用户
>[root@localhost ~]# vim /etc/sudoers

添加以下内容，wq保存
```shell
## Allow root to run any commands anywhere 
root    ALL=(ALL)       ALL
lijiangdong     ALL=(ALL)       ALL
```
将w权限回收
>[root@localhost ~]# chmod -v u-w /etc/sudoers <\br>
mode of ‘/etc/sudoers’ changed from 0640 (rw-r-----) to 0440 (r--r-----)









