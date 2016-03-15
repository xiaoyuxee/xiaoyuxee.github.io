---
title: Java中的输入/输出流
date: 2016-02-15 10:46:17
tags: IO
categories: Java-Core
toc: true
---
流（Stream）是java中输入和输出经常涉及也是最简单的概念。Java中存在各种各样的流，字符流、字节流、缓存流、基本数据流、对象流等。Java持久化中谈及的序列化与反序列化同样离不开流（将整个对象写入流中，并可以从流中获取整个对象）。

<!-- more -->

## 字节流（InputSteam）
常见的有`FileInputSteam`，主要用于读取原生的字节流，如图片。如果想要读取关于字符的字节流可以使用`FileReader`。

## 字符流（Reader）

### `Reader`
所有的字符流都继承于Reader，它包含一个同步锁`lock`，默认是将自己作为一锁。
``` java
public abstract class Reader implements Readable, Closeable {
    
    // 用于同步流
    protected Object lock;

    protected Reader() {
        this.lock = this;
    }

    protected Reader(Object lock) {
        if (lock == null) {
            throw new NullPointerException();
        }
        this.lock = lock;
    }
}
```

### `InputStreamReader`
InputStreamReader是从字节流到字符流的桥连接。它读取字节并将其编码成相应[字符集](https://docs.oracle.com/javase/8/docs/api/java/nio/charset/Charset.html)的字符。字符集可以用指定的名称、或具体的字符集对象或者系统默认的字符集。

其实现是通过`StreamDecoder`的forInputStreamReader方法进行编码。`read()`方法同样委托给`StreamDecoder`。

### `FileReader`
继承于In继承putStreamReader，其实现是通过`FileInputSteam`创建字节流然后通过桥连接构造`StreamDecoder`，其读方法均由`InputStreamReader`实现（委托给`StreamDecoder`）。

``` java
public class InputStreamReader extends Reader {

    public InputStreamReader(InputStream in) {
        super(in);
        try {
            sd = StreamDecoder.forInputStreamReader(in, this, (String)null); // check lock object
        } catch (UnsupportedEncodingException e) {
            // The default encoding should always be available
            throw new Error(e);
        }
    }

    public int read() throws IOException {
        return sd.read();
    }
}

public class FileReader extends InputStreamReader {

    public FileReader(String fileName) throws FileNotFoundException {
        super(new FileInputStream(fileName));
    }
}
```

## 缓冲流（Buffered Stream）
对于无缓冲的I/O操作，每次写或者读请求都会直接作用于操作系统，如硬盘访问、网络访问等代价非常昂贵的操作。 缓冲流的引入就是为了提升常规I/O操作的效率。

缓冲输入流是从一块缓冲的内容中读取数据，只有当缓冲区没有数据时才会调用操作系统的API获取一整块的数据，然后放入缓冲区以供程序读取。同样，缓冲输出流每次向缓冲区写入数据，只有当缓冲区被写满后才会调用操作系统的API将数据真正写入目标位置。

* [BufferedInputStream](https://docs.oracle.com/javase/8/docs/api/java/io/BufferedInputStream.html)、[BufferedOutputStream](https://docs.oracle.com/javase/8/docs/api/java/io/BufferedOutputStream.html)：用于创建缓冲的字节流
** 分别继承于`FilterInputStream`和`FilterOutputStream`，本身字节流成员。
** 必须使用字节流构造，并可以指定缓存区大小。

* [BufferedReader](https://docs.oracle.com/javase/8/docs/api/java/io/BufferedReader.html)、[BufferedWriter](https://docs.oracle.com/javase/8/docs/api/java/io/BufferedWriter.html)：用于创建缓冲的字符流


## Scanning
[Scanner](https://docs.oracle.com/javase/8/docs/api/java/util/Scanner.html)，一个文本扫描工具，使用正则表达式解析基本数据类型和`String`类型。`Scanner`可以使用分隔符将输入数据分割成诸多片段，默认的分隔符为*whitespace*（参见[Character.isWhitespace](https://docs.oracle.com/javase/8/docs/api/java/lang/Character.html#isWhitespace-char-))。可以使用不同`next`方法将这些片段转化成不同的数据类型。

### 构成方法：
* 字符流：`Readable`接口。比如继承于`Reader`的所有类，`FileReader`等。
* 字节流：`InputStream`，支持自定义charset。其实现是将*字节流转化为字符流*，使用字符流构造。
    ``` java
    public Scanner(InputStream source) {
        this(new InputStreamReader(source), WHITESPACE_PATTERN);
    }
    ```
* 字节可读管道：`ReadableByteChannel`，比如对应文件的管道`FileChannel`。
* 文件：`File`，支持自定义charset。其实现是构造`FileInputStream`，获取文件管道`FileChannel`，然后使用字节可读管道构造。
* 文件路径：`Path`，支持自定义charset。其实现是获取文件的字节流，然后构造字符流。
* 字符串：`String`。其实现是构造`StringReader`，然后构造字符流。

### 主要方法
* `hasNext、next`：不仅支持基础类型，还支持`BigDecimal`、`BigInteger`以及`nextLine`。这两个方法都会阻塞（block）等待输入。
* `useDelimiter`：自定义分隔符。
* `reset`：重置分隔符为*whitespace*。

## [PrintStream](https://docs.oracle.com/javase/8/docs/api/java/io/PrintStream.html)
* 异常：不会抛`IOException`，但是内部会有相应的标记，可以通过`checkError`来检查。
* 自动flush：当写入一个字节数组、或者调用`println`方法、或者写入换行符`\n`时会自动flush。

*注意*： 通过*PrintStream*打印的字符均会根据平台默认的字符编码被转成字节。

### 构造方法：
支持字节流、文件、文件名进行构造。使用字节流构造时，可自定义是否自动flush，默认为`false`。使用文件名或文件时，不可自定义，默认也为`false`。

相应的，用于处理字符的类为[PrintWriter](https://docs.oracle.com/javase/8/docs/api/java/io/PrintWriter.html)。一般情况下直接使用`PrintWriter`。

### 主要方法
* `format(String format, Object... args)`
  类似C语言，格式化输入源数据。具体语法参考[Formatter](https://docs.oracle.com/javase/8/docs/api/java/util/Formatter.html#syntax)。

## [PrintWriter](https://docs.oracle.com/javase/8/docs/api/java/io/PrintWriter.html)
跟`PrintStream`类似，也是用于格式化输出。并且实现了所有`PrintStream`的方法。但自动flush时有点不同：不再直接检查被写入的是否为换行符`\n`，而是检查是否为当前平台的换行符。

支持字节流（OutStream）、字符流（Writer）、文件名、文件（File）进行构造。特别的，除了字节流构造意外，其他实现均为会转化成缓存字符流（`BufferedWriter`）。

## 标准流
标准输入流：`System.in`。
标准输出流：`System.out`，`System.error`。

因为历史原因，这些标准流均为*字节流*，而非字符流。需要转化时可以通过`InputStreamReader`进行转化，如：

``` java
InputStreamReader cin = new InputStreamReader(System.in) 
```

## Console
`Console`是更强大的标准流，同时提供了输入流和输出流，并且是字符流。还有一个安全特性是针对密码输入时设计。

* 无public构造函数：使用时，通过System获取，`System.console()`。
* readPassword：返回一个字符（char）数组，不再使用时可以将其重写置空，从内存中删除。

## 数据流（DataStream）
对输入/输出字节流的包装，方便程序中输入/输出基本类型以及String。

因为DataStream操作的是字节流，所以在使用时可以使用缓存流`BufferedInputStream`和`BufferedOutputStream`。


与其他流不同的地方是检测输入结束的条件，它是通过catch`EOFException`，而不是检测返回值，参见一下具体实现：

``` java
public final byte readByte() throws IOException {
    int ch = in.read();
    if (ch < 0)
        throw new EOFException();
    return (byte)(ch);
}
```

## 对象流（Object Stream）
对象流包括`ObjectInputSteam`和`ObjectOutputStream`，实现的接口分别为`ObjectInput`和`ObjectOutput`（继承于数据流接口`DataInput`）。这也意味着在数据流中所有基本数据类型相关的方法，在对象流中都有相应的实现。

`ObjectInputSteam`用来反序列化那些用`ObjectOutputStream`序列化的基本数据类型和对象。而`ObjectOutputStream`则是一种将对象持久化的方法。

### 复杂对象的输入流与输出流
* 当一个对象引用了其他对象时，其所有引用都会被写入流中。
* 当多个对象引用了同一个对象时，被引用的对象在流中只会存在一份。




