---
title:  Mysql常用命令
date: 2017-12-11 21:24
category: [数据库]
tags: [mysql]
comments: true
---

# 登录

> mysql -u root -p

# mysql导出整个数据库

>mysqldump -u 用户名 -p 数据库名 > 导出的文件名 
mysqldump -u dbuser -p dbname > dbname.sql

<!--more-->

# 导入数据库(Linux)

（1）首先建空数据库

>mysql>create database abc;

（2）选择数据库

>mysql>use abc;

（3）设置数据库编码


>mysql>set names utf8;


（4）导入数据（注意sql文件的路径）

>mysql>source /home/abc/abc.sql;

# mysql忘记密码

修改/etc/my.conf: 
> \# vi /etc/my.cnf

在\[mysqld]的段中加上一句：skip-grant-tables 保存并且退出vi。

重新启动mysqld 
service mysqld restart

```
mysql> USE mysql;</br> 
mysql> update mysql.user set authentication_string=password('root') where user='root';
mysql> flush privileges ;
mysql> quit
```

将MySQL的登录设置修改回来 
>\# vi /etc/my.cnf 

将刚才在[mysqld]的段中加上的skip-grant-tables删除,保存并且退出vi。

重新启动mysqld 
service mysqld restart

