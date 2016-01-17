---
title: MySQL简易教程
date: 2016-01-14 00:41:38
tags: MySQL
categories: MySQL
toc: true
---

许久没用mysql了，`root`密码都忘了... 

简单的教程：如何使用mysql客户端创建和使用数据库。（详细版请参考[官网文档](http://dev.mysql.com/doc/refman/5.7/en/tutorial.html))

<!-- more -->

## 连接/断开数据库服务器

``` bash
mysql -h host -u user -p
```

* `-h` 用来指定主机（如果是本地`localhost`，可以忽略）
* `-u` 指定用户名
* `-p` 表示使用密码登陆

输入密码后就可以看到mysql欢迎页面：

``` bash
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 6
Server version: 5.7.10 Homebrew

Copyright (c) 2000, 2015, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>
```

当然，你也有可能得到这样的提示：`ERROR 2002 (HY000): Can't connect to local MySQL server through socket '/tmp/mysql.sock' (2)`，那是因为你忘记启动数据库服务器了，使用以下命令启动：

``` bash
mysql.server start
```

断开连接使用`quit`或者`\q`，mac下也可以使用组合键`control + D`：

``` bash
mysql> quit
Bye
```

## 查询

我们来查询下当前mysql版本以及当前时间：

``` bash
mysql> select version(), current_date;
+-----------+--------------+
| version() | current_date |
+-----------+--------------+
| 5.7.10    | 2016-01-14   |
+-----------+--------------+
1 row in set (0.00 sec)

mysql> 
```

* 一条`query`由一个以分号`;`结尾的SQL表达式组成。（也有很多别的场景可以忽略分号，比如`quit`）
* 查询的信息以逗号`,`分割，否则会语法抛错
* 查询结果有2部分
  * 第一部分已表格形式呈现，第一行为表头，随后查询的数据
  * 第二部分为一些其他信息，如`返回多少行`，以及`本次查询耗时`
* 查询语句是大小写不敏感的

多条查询语句可以放在一行，分别已分号结尾。当你忘记输入分号或者想多行输入时，会是这个样子：

``` bash
mysql> select
    -> user()
    -> ,
    -> current_date;
+----------------+--------------+
| user()         | current_date |
+----------------+--------------+
| root@localhost | 2016-01-14   |
+----------------+--------------+
1 row in set (0.00 sec)

mysql> 
```

如果你不想执行当前的查询，可用通过`\c`来取消：

``` bash
mysql> select user()
    -> \c
mysql>
```

一些提示符及意义：

* `mysql>`：准备接受下次查询
* `->`：等待一次查询中的下一行
* `'>`：等待`'`与之匹配表示一个完整的`string`
* `">`：等待`"`与之匹配表示一个完整的`string`
* ``\>``：等待`` ` ``与之匹配表示一个完整的标识符
* `/*>`：等待`*/`与之匹配表示一个完整的注释

## 创建/使用库








