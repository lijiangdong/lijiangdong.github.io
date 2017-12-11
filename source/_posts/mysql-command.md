---
title:  Mysql常用命令
date: 2017-12-11 21:24
category: [数据库]
tags: [mysql]
comments: true
---

1. 登录

```
mysql -u root -p
```

2. mysql导出整个数据库
```
mysqldump -u 用户名 -p 数据库名 > 导出的文件名

mysqldump -u dbuser -p dbname > dbname.sql

ljd@ljd-Ubuntu:~$ mysqldump -u root -p freewords >freewords.sql
```

3. 导入数据库

（1）首先建空数据库

```
mysql>create database abc;
```

2. 导入数据库

（1）选择数据库
```
mysql>use abc;
```

（2）设置数据库编码
```
mysql>set names utf8;
```

（3）导入数据（注意sql文件的路径）

```
mysql>source /home/abc/abc.sql;
```