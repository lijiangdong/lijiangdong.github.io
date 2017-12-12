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

3.再通过hostname或者hostnamectl status命令查看更改是否生效

> \#hostnamectl status