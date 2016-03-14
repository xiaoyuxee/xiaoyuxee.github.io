---
title: nginx之http rewrite
date: 2016-02-18 11:21:05
tags: http_rewrite
categories: nginx
---

nginx中经常会用到rewrite模块进行资源重定向，涉及URI正则匹配、临时重定向、永久重定向等。

<!-- more -->

## 指令执行顺序

* 顺序执行`server`中指令
* 重复执行
  * 根据请求的URI寻找相应的`location`
  * 顺序执行`location`中指令
  * 如果URI被rewrite，则重新执行，但此时不超过*10*次

##

### `if`

```
if ($slow) {
    limit_rate 10k;
}

if ($variable = "test") {
    set $variable "test";
}

if ($http_cookie ~* "id=([^;]+)(?:;|$)") {
    set $id $1;
}
```

*`if`中的配置文件继承于{}外面的配置。*

#### if中条件约定

* 变量：如果变量为`empty`或者`0`，返回`false`
* 变量比较：使用`=`或者`!=`
* 变量匹配：使用`~`（大小写敏感）或者`~*`（大小写不敏感）、`!~`、`!~*`。如果正则表达式中包含`}`、`;`字符，则表达式需要使用单引号或者双引号。
* 检查文件是否存在：`-f`、`!-f`
* 检查文件夹是否存在：`-d`、`!-d`
* 检查文件、文件夹、链接是否存在：`-e`、`!-e`
* 检查可执行文件是否存在：`-x`、`!-x`

## `break`






