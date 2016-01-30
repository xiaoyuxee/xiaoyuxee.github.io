---
title: study-spring-mve
date: 2016-01-15 01:21:09
tags:
- spring
- mvc
categories:
- spring
---

## 一个请求的生命周期

![请求的生命周期](/images/a_request's_life_in_spring_mvc.png)

`DispatcherServlet`是spring mvc中的前端控制器`front controller`。前端控制器是MVC中常用的模式，其职责是将请求委托给其他应用组件。在这里，`DispatcherServlet`的工作就是将请求转发给`controller`。一个典型的web应用会有很多`controller`，那么`DispatcherServlet`应该如何分发请求呢？通过handler mapping。

`controller`是用来处理用户请求的，但一个设计良好的`controller`应该是不自己处理或者处理得很少，而是将业务逻辑委托给其他业务逻辑服务对象。逻辑处理完后往往需要带回一些信息或数据给用户，然后展示在浏览器端，也就是`model`。当然仅仅返回原始的数据是不够的，需要将其格式化友好的呈现在用户面前，比如典型的HTML格式。所以这些信息需要包含一个`view`，典型的有jsp。注意，这里的 view 仅仅是个名称，`DispatcherServlet`通过视图解析器（view resolver）将之与具体的实现匹配起来。

## control 怎么写

## quest 如何 handle














