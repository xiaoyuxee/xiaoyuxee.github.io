---
title: logback-tutorial
tags: logback
date: 2016-01-18 22:21:36
categories:
toc: true
---

一个系统，小到web框架，大到操作系统，都离不开日志。因为开发者、使用者都需要在一些重要时刻记录一些重要事件或者里程碑的发生。如[spring-framework](http://docs.spring.io/spring/docs/current/spring-framework-reference/htmlsingle/#overview-logging)在文档中讲到，它的外部依赖甚少，只有日志框架[commons-logging](https://commons.apache.org/proper/commons-logging/)。那么对日志框架的了解，对web开发、框架开发将起到非常重要的作用。

这边文章主要介绍日志框架中的`Logback`，以及现在流行的其他框架。

<!-- more -->

## SLF4J与JCL

{% blockquote "-- SLF4J user manual, " http://www.slf4j.org/manual.html %}
The Simple Logging Facade for Java (SLF4J) serves as a simple facade or abstraction for various logging frameworks (e.g. java.util.logging, logback, log4j) allowing the end user to plug in the desired logging framework at deployment time.
{% endblockquote %}

[SLF4J](http://www.slf4j.org/) 是一个面向Java的简单日志门面，是其他日志框架的抽象（logback、log4j等)，允许使用者在编译部署时插入具体的日志框架。

*JCL*（Jakarta Commons Logging），也叫[Apache Commons Logging](http://commons.apache.org/)，是Apache早起开发的一套日志框架。

## logback是什么

`Logback`是一个日志框架，用来终结现在广为流行的`Log4j`日志框架。它的设计者正是 *log4j* 的作者，*Ceki Gülcü*。

## dependency

``` xml maven
<dependency>
    <groupId>ch.qos.logback</groupId>
    <artifact>logback-classic</artifact>
    <version>1.1.3</versrion>
</dependency>
```

or

``` gradle gradle
dependencies {
    compile("ch.qos.logback:logback-classic:1.+")
}
```

Logback想要正常使用，classpath中必须可以找到`slf4j-api.jar`、`logback-core.jar`、`logback-classic.jar`3个jar包。




