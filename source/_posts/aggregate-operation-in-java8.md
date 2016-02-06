---
title: Java8中的Stream
date: 2016-02-06 14:41:16
tags:
- Aggregate-Operation
- Stream
- Java8
categories: Java-Core
---

Java8中为集合类`Collections`引入了新的特性：流`Stream`，使得基于集合的操作更加简洁、直观。为了更好的理解`Stream`，需要对Lambda表达式和方法引用有一定的认知，参见前一篇Note：[Java8中的Lambda表达式](http://www.xiaoyuxee.com/2016/02/05/lambda-in-java8/)。

<!-- more -->

## 引子

当我们遍历一个集合并进行打印时：

``` java
for (Person p : persons) {
    System.out.println(p.getName());
}
```

使用`Stream`、Lambda表达式后：

``` java
persons
    .stream()
    .forEach(p -> System.out.println(p.getName()));
```

## 流`Stream`

流`Stream`指的是一系列元素，但不像集合`Collection`，它不是用来存储元素的数据结构，而是通过管道（pipeline）而携带元素。通过集合中的`java.util.Collection.stream()`方法可以获得。

## 管道pipeline

管道（pipeline）指的是一系列的集成操作。如：

``` java
persons
    .stream()
    .filter(p -> p.getAge() >= 18)
    .forEach(p -> System.out.println(p.getName()));
```

管道一般由以下几部分组成：

* 来源：可能是集合、数组、生成函数或者I/O管道。
* 中间操作：比如过滤器`filter`，产生一个新的管道（pipeline）
* 终止操作：比如`forEach`。

比如，统计*年龄≥18岁的person的平均年龄*：

``` java
persons
    .stream()
    .filter(p -> p.getAge() >= 18)
    .mapToInt(Person::getAge)
    .average()
    .getAsDouble();
```

上例中，`mapToInt`操作产生了一个类型为`IntStream`的新流，包换所有*年龄≥18岁的person的年龄流*。

`average`操作将计算`IntStream`中所有元素的平均值。JDK中提供了很多类似`average`终止操作，组合流中内容并返回一个值。这类操作叫做[reduction](http://docs.oracle.com/javase/tutorial/collections/streams/reduction.html)。

## `Reduction`

类似于`average`，统计流Stream中内容而返回一个值，还有`sum`、`min`、`max`、`count`。此外，JDK还提供返回集合的终止操作。

当然，JDK还提供了更加通用的`reduce`和`collect`方法。

### `reduce`方法

例如，我们要统计*年龄≥18岁的person的年龄之和*：

``` java
persons
    .stream()
    .filter(p -> p.getAge() >= 18)
    .mapToInt(Person::getAge)
    .sum();
```

这里用到了`sum`终结操作，计算所有`IntStream`中内容之和。如果使用`reduce`则可以这么写：

``` java
persons
    .stream()
    .filter(p -> p.getAge() >= 18)
    .mapToInt(Person::getAge)
    .reduce(0, Math::addExact);
```

查询`sum`源码可以发现，其实也是如此：

``` java
public final int sum() {
    return reduce(0, Integer::sum);
}
```

关于`reduce`有三个方法：

#### `Optional<T> reduce(BinaryOperator<T> accumulator)` 用于寻找最大值、最小值等

#### `T reduce(T identity, BinaryOperator<T> accumulator)` 适合于有累加行为

`reduce`操作包含两个参数：

* 标识：reduce操作的初始化值以及默认值（如果流中没有元素）
* 累加器：累加器包含两个参数：requce操作的*部分结果和下一个流中内容的值*，然后返回一个新的局部结果。

`reduce`操作时，累加器每次都返回一个新的值。假如你的操作是返回一个更加复杂的对象，比如集合，那么就需要为你的程序担忧了。因为其效率是非常低的，正确的做法是*更新已经存在的集合*。这就是`collect`方法。

### `collect`方法

假如你要收集所有*年龄≥18岁的person的人名*：

``` java
persons
    .stream()
    .filter(p -> p.getAge() >= 18)
    .map(Person::getName)
    .collect(ArrayList::new, ArrayList::add, ArrayList::addAll);
```

`collect`有两个方法：

####  `R collect (Supplier<R>, BiConsumer<R, ? super T>, BiConsumer<R, R>)`

* 第一个参数Supplier：用于初始化返回结果
* 第二个参数BiConsumer： 用于操作*部分结果与下一个流内容*
* 第二个参数BiConsumer： 用于*合并操作*

#### `<R, A> R collect(Collector<? super T, A, R> collector)`

`Collector`将上述*初始化*、*部分接口与下一个流内容操作*、*合并*封装了起来，并且在`java.util.stream.Collectors`中提供了一些常用的collector，如`toList`、`toSet`等。

上述收集所有*年龄≥18岁的person的人名*的例子可以改写为：


``` java
persons
    .stream()
    .filter(p -> p.getAge() >= 18)
    .map(Person::getName)
    .collect(Collectors.toList()));
```

`Collectors.groupingBy(classifier)`用于分组，返回一个key为`classifier`分类的标准，value为ArrayList的Map。

``` java
persons
    .stream()
    .collect(Collectors.groupingBy(Person::getGender)));
```

如果要将persons按性别分组，返回其name：

``` java
persons
    .stream()
    .collect(Collectors.groupingBy(
            Person::getGender,
            Collectors.mapping(Person::getName, Collectors.toList()))));
```

如果要将persons按性别分组，返回每组的年龄总数/年龄最大值/平均年龄：

``` java
// 年龄和
persons
    .stream()
    .collect(Collectors.groupingBy(
            Person::getGender,
            Collectors.mapping(
                    Person::getAge, 
                    Collectors.reducing(0, Math::addExact)))));
// 年龄和
persons
    .stream()
    .collect(Collectors.groupingBy(
            Person::getGender,
            Collectors.reducing(0, Person::getAge, Math::addExact))));

// 年龄最大值
persons
    .stream()
    .collect(Collectors.groupingBy(
            Person::getGender,
            Collectors.mapping(
                    Person::getAge, 
                    Collectors.reducing(Math::max)))));

// 年龄平均值
persons
    .stream()
    .collect(Collectors.groupingBy(
            Person::getGender,
            Collectors.averagingInt(Person::getAge))));


```

## 参考
[http://docs.oracle.com/javase/8/docs/api/java/util/stream/package-summary.html](http://docs.oracle.com/javase/8/docs/api/java/util/stream/package-summary.html)

