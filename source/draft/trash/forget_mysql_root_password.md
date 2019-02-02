
---
title:  忘记mysql密码
date: 2017-01-20 16:50 
category: [mysql]
tags: [mysql]
comments: true
---

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