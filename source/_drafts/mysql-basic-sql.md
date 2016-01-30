---
title: mysql-basic-sql
tags:
categories:
---

## 启动/终止mysql
`mysql.server start`
`mysql.server status`
`mysql.server stop`
`mysql.server restart`
`mysql.server reload`

## 登录mysql
`mysql -u root -p -h localhost`

* -u : username
* -p : password
* -h : host
* -P : port

## 退出mysql
`exit`
`quit`
`\q`

## 修改mysql提示符
`prompt \h`

* \h : host
* \u : username
* \d : database
* \D : date

## 常用命令
`SELECT VERSION(), NOW(), USER();`

## 数据库基本操作
`CREATE {DATABASE | SCHEMA} [IF NOT EXISTS] db_name [DEFAULT] CHARACTER SET charset_name`

`SHOW CREATE db_name;`

`ALTER {DATABASE | SCHEMA} [db_name] [DEFAULT] CHARACTER SET [=] charset_name`

`DROP {DATABASE | SCHEMA} [IF EXISTS] db_name;`

## 数据类型

* 整型
	* TINYINT 1字节
	* SMALLINT 2字节
	* MUDIUMINT 3字节
	* INT 4字节
	* BIGINT 8字节
* 字符串
* 时间

## 数据表

`CREATE TABLE table_name (column_name, data_type;`

`