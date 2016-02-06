---
title: Wiring Beans In Spring
date: 2016-01-30 00:43:26
tags: Wiring-Beans
categories: Spring
toc: true
---

[控制反转和依赖注入](http://martinfowler.com/articles/injection.html)是Spring的核心功能，也是其他模块的基石。Spring容器控制着bean的生命周期，并维护其之间的依赖关系。如何在容器中装配（配置）这些bean是学习Spring的基础。

本文将介绍如何在Spring4.0中进行bean的装配，及每种方式的特点。

<!-- more -->

## 自动装配（Automatically）

`@ComponentScan`：用于指定扫描路径，将所有带有注解`@Component`的类注册到ioc容器中进行管理。也可以通过XML类配置`<context:component-scan>`。

* `basePackages`：当没有指定扫描的包名时，默认扫描当前配置所在的包
* `basePackageClasses`：使用`basePackages`可能会遇到一个问题：重构包名时，扫描将会失效。那么我们可以指定扫描类所在的包。有个技巧为：*在预想扫描的包里创建一个标记类，用于标记扫描的包*。

`@Component`：用于说明该类将会注册到ioc容器中。

* `bean id`：当没有指定id时，id为首字母小写的类名，如：beanFactory，当类为BeanFactory时。
* `@Named`：与`@Component`有同样的功效，它是[Dependency Injection for Java](https://jcp.org/en/jsr/detail?id=330)中的规范。*不建议使用*，因为其语义很差劲。

`@AutoWired`：告诉ioc容器，帮其自动解决（某方法）的依赖关系。

* `使用范围`：适用于构造函数、以及其他任何函数。
* `Inject`：同`@AutoWired`。

## Java代码装配（Java-based）

虽然自动装配非常的诱人，简介而高效，但有些场景却无法适用。比如，当依赖一些第三方的jar包时，我们没有源码，没有办法为他们添加类似于`@Component`的注解，那我们必须在配置中明确指定依赖关系。

`Configuration`：表示此类为配置文件，其中包含将要注册到容器中的Bean。

`Bean`：表示此方法将会返回一个对象，并需要将其注册到容器中进行管理。
* `bean id`：默认id为*方法名*，当然也可以显式的指定：`name="foo"`。
* 依赖关系：可以通过调用通一个配置里的方法，或者构造函数解决（不在同一个配置项中时）。

## XML文件装配（XML）

XML文件配置Spring历史悠久，Spring刚面世时就是基于XML来配置bean，甚至在大多数人眼中有这样的观念：spring就是XML配置。

## Java-based with xml

使用java-based配置时，可以引入别的java-based配置以及xml配置：

* `@Import`：引入其他java-based配置
* `@ImportResource`：引入xml配置

`注意`：这里的xml必须要携带完整的路径（包括包路径），
如：`@ImportResource("classpath:/com/osceola/soundsystem/cd-config.xml")`

## Xml with java-based

在使用xml配置时，同样可以引入别的xml配置和java-based配置：

* `<import resource="config.xml" />`
* `<bean class="Config" />`：想配置bean一样将java-based类配置进来

## 推荐

`Automatically` > `Java-based` > `XML`
