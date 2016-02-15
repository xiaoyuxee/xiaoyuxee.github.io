---
title: Java8中的Lambda表达式
date: 2016-02-05 00:43:08
tags: 
- Lambda
- Java8
categories: Java-Core
toc: true
---

Lambda表达式是Java8引进的新特性，用一句话概括为：*更紧凑的表达仅有一个方法的接口实例*。

<!-- more -->

## Lambda表达式如何而来

我们在日常编程中经常会遇到只有一个方法的接口，如`Runnable`，只有一个`void run()`方法。在Java8之前是不支持接口中的方法带有默认实现的，所以在Java8中，Lambda表达式更准确得讲为：*更紧凑的表达仅有一个抽象方法的接口实例*，因为Jaba8中可以有[默认实现的方法](http://docs.oracle.com/javase/tutorial/java/IandI/defaultmethods.html)和[静态方法](http://docs.oracle.com/javase/tutorial/java/IandI/defaultmethods.html#static)。

特别地，在使用策略模式时，经常会使用匿名类来实现某个接口：

``` java
public interface Operation {
    int calculate(int a, int b);
}

public static int calculate(int a, int b, Operation operator) {
    return operator.calculate(a, b);
}

public static void main(String[] args) {
    System.out.println("1 + 2 = " + calculate(1, 2, new Operation(){

        @Override
        public int calculate(int a, int b) {
            return a + b;
        }
    }));
}
```

而在引入Lambda表达式之后，上面的例子就可改写为：

``` java
public static void main(String[] args) {
    System.out.println("1 + 2 = " + calculate(1, 2, (a, b) -> a + b));
}
```

较之前的匿名类而言，更加简洁、直观、紧凑。

## Lambda表达式的语法

一个Lambda表达式由以下几部分组成：

* 一个在闭合圆括号中用逗号分隔的参数列表。
  如：`calculate(1, 3, (int a, int b) -> a + b)`。
  *Note*：在Lambda表达式中，*可以省略数据类型*。此外，*如果仅有一个参数时，也可以将圆括号省略*：`a -> a.foo()`。

* `->`

* 表达体：一个单独的表达式 或者 区块。
  如：`a > 0 && b >0` 或者 `{ return a > 0 && b > 0; }`。当然，如果返回值为`void`时，完全没必要写成后者形式。

## JDK对Lambda表达式的支持

Java8中引入注释[FunctionalInterface](http://docs.oracle.com/javase/8/docs/api/java/lang/FunctionalInterface.html)，用于标识功能性接口[](http://docs.oracle.com/javase/8/docs/api/java/util/function/package-summary.html#package.description)。

在`java.util.function`包中提供了大量的通用功能性接口：`Function`、`Consumer`、`Predicate`：

* `Function`：`R apply(T t)`
* `Consumer`：`void accept(T t)`
* `Predicate`：`boolean test(T t)`
* `Supplier`：`T get()`

还包括一些指定数据类型的`ToIntFunction`、`LongConsumer`等。

使用JDK中提供的标准接口可以将上例改写为：

``` java
public static int calculate(int a, int b, ToIntBiFunction<Integer, Integer> operator) {
    return operator.applyAsInt(a, b);
}

public static void main(String[] args) {
    System.out.println("1 + 2 = " + calculate(1, 2, (a, b) -> a + b));
}
```

## Lambda表达式的目标类型

Java编译器会通过调用时的上下文或者Lambda表达式的位置来决定其表达式的类型。

例如，当你自己定义一个相同功能的功能性接口进行使用时：

``` java
public static int calculate(int a, int b, Operation operator) {
    return operator.calculate(a, b);
}
```

`(a, b) -> a + b)`的类型为`Operation`，当你使用JDK提供的标准接口时类型为：`ToIntBiFunction`。

### 目标类型和方法参数

比如下面两个可调用的方法：

``` java
void invoke(Runnable r) {
    r.run();
}

<T> T invoke(Callable<T> c) {
    return c.call();
}
```

当执行 `String s = invoke(() -> "done");` 时，调用的方法为 `invoke(Callable<T> c)`，因为这个方法返回了值。所有Lambda表达式的类型为`Callable<T>`。

## 本地变量的访问

Lambda表达式中并不会产生一个新的局部变量，所以你不能再表达式的参数列表中包含上一次的局部变量，但是你可以在表达体中直接使用这些的变量。

当然，同内部类、匿名类一样，这些被Lambda表达式使用的变量必须是`final`类型，否则编译器会提示你：

> local variables referenced from a lambda expression must be final or effectively final。

``` java
public static void main(String[] args) {
    // Lambda expression's parameter a cannot redeclare another local variable  
    // defined in an enclosing scope.
    // int a = 0;
    
    final int c = 3;
    System.out.println("1 + 2 = " + calculate(1, 2, (a, b) -> a + b + c));
}
```

## 方法引用

还是上面的例子：计算两个数字之和。`java.lang.Math`中提供了相关api：

```
public final class Math {

    public static int addExact(int x, int y) {
        int r = x + y;

        if (((x ^ r) & (y ^ r)) < 0) {
            throw new ArithmeticException("integer overflow");
        }
        return r;
    }
}
```

所以上述例子可以改写为：

``` java
System.out.println("1 + 2 = " + calculate(1, 3, 
        new ToIntBiFunction<Integer, Integer>() {
    
            @Override
            public int applyAsInt(Integer t, Integer u) {
                return Math.addExact(t, u);
            }
        }));
```

此处的功能接口中实际上调用的是`addExact`方法，Java8中支持方法的引用，可以写得更简洁 ：

``` java
System.out.println("1 + 2 = " + calculate(1, 3, Math::addExact));
```








