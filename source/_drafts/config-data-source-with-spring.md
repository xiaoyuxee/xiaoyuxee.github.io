---
title: Config Data Source With Spring
date: 2016-01-24 21:50:46
tags: JDBC
categories: 
---

## Spring中配置data source

### JNDI data source

### pooled data source

### JDBC driver-based data source

* `DriverManagerDataSource` —— 每次请求返回一个连接，不支持连接池。
* `SimpleDriverDataSource` —— 功能同`DriverManagerDataSource`，解决class加载问题
* `SingleConnectionDataSource` —— 每次返回一个相同的连接（连接数为1的连接池）

### embedded data source

## JDBC template


