---
title: Java中的NIO
date: 2016-03-11 16:36:02
tags: NIO
categories: Java-Core
toc: true
---
java.io是面向序列化得字节流，阻塞模式工作，在java.nio中引入很多新的类，它们是面向缓存、非阻塞模式工作，并且类似线程池引入了选择器来管理管道,极大的提升的I/O性能。nio是new i/o，对传统i/o的扩展，但不是替代。

<!-- more -->

## 路径（Path）
[Path](https://docs.oracle.com/javase/8/docs/api/java/nio/file/Path.html)是`java.nio.file`中一个重要的概念。

### 什么是路径
路径在文件系统中用于定位文件。比如：在Mac OS中`/Users/Xee/logs/sys.log`就表示定位sys.log的路径。在文件操作系统中，文件都是以树或者继承的形态分布。不同的是，在Unix系统下只有一个根root目录，而在Windows下可能会存在多个卷volume。路径的组成包含根目录、根目录到文件路径上的所有中间目录和文件名本身，它们用文件系统中的分隔符连接delimiter。不同的操作系统中分隔符也不一样，Unix中为slash`/`，如`/Users/Xee/logs/sys.log`，而在Windows中为black slash`\`，如`C:\home\sally\statusReport`。

### 路径类型
路径分为绝对路径和相对路径。绝对路径包含了根目录以及定位到具体文件所需的所有文件目录。相对目录是相对于某一个文件的路径，需要结合相对的文件才能定位到想定位的问题。

### `Path`的主要方法
Path包含很多获取有关路径信息的很多方法，包括访问路径中的元素、转化路径格式、匹配路径等。这些方法大部分为静态（static）方法，因为这些操作只是针对路径本身而不需要访问文件系统。

#### 创建路径
``` java
    Path path1 = Paths.get("/tmp/foo"); // /tmp/foo
    Path path2 = Paths.get(System.getProperty("user.home"), "bar"); // /Users/Xee/bar
    Path path3 = Paths.get(URI.create("file:///Users/Xee/logs/"));  // /Users/Xee/logs/
```
创建路径可以通过[Paths](https://docs.oracle.com/javase/8/docs/api/java/nio/file/Paths.html)的静态方法，支持通过`String`和`URI`来创建。其实它的实现是调用`FileSystems`：
``` java
public static Path get(String first, String... more) {
    return FileSystems.getDefault().getPath(first, more);
}
```

#### 获取路径信息
你可以将Path的存储结构想象成一个存储着路径中所有元素的列表：索引0存储的是根元素，索引n-1存储的是需要定位的文件名，n是路径中所有元素的个数。
``` java
Path path = Paths.get(URI.create("file:///Users/Xee/logs/"));

System.out.format("toString: %s%n", path.toString());
System.out.format("getFileName: %s%n", path.getFileName());
System.out.format("getName(0): %s%n", path.getName(0));
System.out.format("getNameCount: %d%n", path.getNameCount());
System.out.format("subpath(0,2): %s%n", path.subpath(0,2));
System.out.format("getParent: %s%n", path.getParent());
System.out.format("getRoot: %s%n", path.getRoot());

/* Output：
toString: /Users/Xee/logs
getFileName: logs
getName(0): Users
getNameCount: 3
subpath(0,2): Users/Xee
getParent: /Users/Xee
getRoot: /
*/
```

#### 规格化路径
`normalize()`：很多文件系统中用`.`表示当前文件夹，`..`表示父级文件夹。

#### 路径转换
* toUri：转化可以在浏览器中访问的路径
* toRealPath：
** 如果是相对路径，返回绝对路径
** 如果包含`.`、`..`等路径，返回规格化的路径
* toAbsolutePath：返回绝度路径
* normalize：去除被简化的路径信息，返回规格化的路径（*但不改变路径类型，原先是相对路径，则还是相对路径*）

``` java
// java文件所在路径：/Users/Xee/osceola/
// 创建相对路径
Path path = Paths.get("..");

System.out.format("toString: %s%n", path.toString());

System.out.format("toUri: %s%n", path.toUri());

// 返回真实路径，去除被简化的信息“..”
System.out.format("toRealPath: %s%n", path.toRealPath(LinkOption.NOFOLLOW_LINKS).toString());

// 返回绝对路径，但包含简化的“..”
System.out.format("toAbsolutePath: %s%n", path.toAbsolutePath().toString());

/* Output:
toString: ..
toUri: file:///Users/Xee/osceola/../
toRealPath: /Users/Xee/
toAbsolutePath: /Users/Xee/osceola/..
*/

```

#### 解析相对路径
``` java
Path path = Paths.get(URI.create("file:///Users/Xee/logs/"));
System.out.format("toString: %s%n", path.resolve("..").toRealPath(NOFOLLOW_LINKS));

/* Output:
toString: /Users/Xee
*/
```

#### 创建相对路径
``` java
Path home = Paths.get("/Users/Xee");
Path sysLog = Paths.get("/Users/Xee/log/sys.log");

System.out.format("home_to_sysLog: %s%n", home.relativize(sysLog));
System.out.format("sysLog_to_home: %s%n", sysLog.relativize(home));

/* Output:
home_to_sysLog: log/sys.log
sysLog_to_home: ../..
*/
```

#### 路径比较
`equals`，`startsWith`，`endsWith`：值得注意的是，这里比较的是*存储列表中的元素*，而不是真实路径*realPath*。

#### 遍历路径中的元素
因为Path实现了`Iterable`接口，所以可以通过迭代器遍历存储结构中的元素。

## Files基本操作
[Files](https://docs.oracle.com/javase/8/docs/api/java/nio/file/Files.html)包含了大量的静态方法，这些方法依赖Path实例。

而实现一般是通过`path.getFileSystem().provider()`获取`FileSystemProvider`来执行具体操作。

### 检查操作

#### 文件存在性判断
*注意：* 
* 文件存在与否可能有三种结果：存在，不存在 或者 未知（没有权限）。
* 查询结果具有时效性，返回结果表示存在，但当你访问是可能不存在（已被删除）

#### 文件访问权限判断
可读性、可写性和可执行性。

#### 路径定位是否同一文件
特别是用于判断软连接定位的文件
``` java
if (Files.isSameFile(p1, p2)) {
    
}
```
### 删除、移动、复制
#### 删除
可以删除文件、目录和链接。

* 如果删除的是链接，链接指向的真正文件是不会被删除的。
* 如果删除的是目录不为空，则删除失败。

#### 复制
复制文件时值得注意的是[Path copy(Path source, Path target, CopyOption... options)](https://docs.oracle.com/javase/8/docs/api/java/nio/file/Files.html#copy-java.nio.file.Path-java.nio.file.Path-java.nio.file.CopyOption...-)中*StandardCopyOption*选项：
如果目标文件存在，那么复制操作失败，除非指定`REPLACE_EXISTING`选项进行覆盖。

同时还支持字节流与Path之间的复制。

*注意*：复制目录时，只会复制目录本身而不会递归复制。StandardCopyOption中也没有这个枚举类型。

#### 移动/重命名
*注意*：
* 如果是在同个目录下移动，等价于*重命名*
* 移动目录时，目录里的子文件、子目录也会随着移动（跟复制有差别，复制时不会递归复制）。当然前提是源目录和目标目录在同一个[FileStore](https://docs.oracle.com/javase/8/docs/api/java/nio/file/FileStore.html)中：同一个卷、移动设备等。

## 文件元数据（文件属性）
文件的元数据包含在文件或者目录中，如是不是一个常规文件，还是一个目录或者一个链接。它的大小是多少，创建时间，最后修改时间，文件所属者，所属用户组和访问权限。

一个文件系统的元数据通常指文件的属性。`Files`中包含了获取以及设置这些属性的方法：
``` xml
+----------------------------------------------+--------------------------------------------------+
| Method                                       | Comment                                          |
+----------------------------------------------+--------------------------------------------------+
| size(Path)                                   | Returns the size of the specified file in bytes. |
+----------------------------------------------+--------------------------------------------------+
| isDirectory(Path, LinkOption)                | Returns true if the file is a directory.         |
+----------------------------------------------+--------------------------------------------------+
| isRegularFile(Path, LinkOption...)           | Returns true if the file is a regular file.      |
+----------------------------------------------+--------------------------------------------------+
| isSymbolicLink(Path)                         | Returns true if the file is a symbolic link.     |
+----------------------------------------------+--------------------------------------------------+
| isHidden(Path)                               | Returns true if the file is considered hidden.   |
+----------------------------------------------+--------------------------------------------------+
| getLastModifiedTime(Path, LinkOption...)     | Returns the specified file's last modified time. |
+----------------------------------------------+--------------------------------------------------+
| getOwner(Path, LinkOption...)                | Returns the owner of the file.                   |
+----------------------------------------------+--------------------------------------------------+
| getPosixFilePermissions(Path, LinkOption...) | Returns a file's POSIX file permissions.         |
+----------------------------------------------+--------------------------------------------------+
| getAttribute(Path, String, LinkOption...)    | Returns the value of a file attribute.           |
+----------------------------------------------+--------------------------------------------------+
```

如果想要一次获取多个属性，Files也是支持的，并且效率大大优于分别取获取单个属性：
``` xml
+----------------------------------------------+------------------------------------------------------------+
| Method                                       | Comment                                                    |
+----------------------------------------------+------------------------------------------------------------+
| readAttributes(Path, String, LinkOption...)  | The String parameter identifies the attributes to be read. |
+----------------------------------------------+------------------------------------------------------------+
|readAttributes(Path, Class<A>, LinkOption...) | The Class<A> parameter is the type of attributes requested.|
+----------------------------------------------+------------------------------------------------------------+

```

### 基本文件属性
``` java
Path file = Paths.get(System.getProperty("user.home"));
        
Map<String, Object> attributesMap = Files.readAttributes(file, "*");

for (Entry<String, Object> entry : attributesMap.entrySet()) {
    System.out.println(entry.getKey() + " = " + entry.getValue());
}

/** Output:
lastAccessTime = 2016-03-12T09:48:48Z
lastModifiedTime = 2016-03-12T09:48:49Z
size = 2686
creationTime = 2015-11-27T02:23:57Z
isSymbolicLink = false
isRegularFile = false
fileKey = (dev=1000004,ino=611925)
isOther = false
isDirectory = true
*/
```

### 设置文件时间戳
``` java
Path file = Paths.get(System.getProperty("user.home"));

FileTime currentTime = FileTime.fromMillis(System.currentTimeMillis());

// 修改“最后修改时间”
Files.setLastModifiedTime(file, currentTime);

FileTime lastModifiedTime = (FileTime) Files.getAttribute(file, "lastModifiedTime");

// 最后修改时间只能精确到秒
assertNotEquals(currentTime.toMillis(), lastModifiedTime.toMillis());
assertEquals(currentTime.to(TimeUnit.SECONDS), lastModifiedTime.to(TimeUnit.SECONDS));
```

### 其他属性获取
* [BasicFileAttributes](https://docs.oracle.com/javase/8/docs/api/java/nio/file/attribute/BasicFileAttributes.html)
* [DosFileAttributes](https://docs.oracle.com/javase/8/docs/api/java/nio/file/attribute/DosFileAttributes.html)
* [PosixFileAttributes](https://docs.oracle.com/javase/8/docs/api/java/nio/file/attribute/PosixFileAttributes.html)

### 其他属性设置
* [BasicFileAttributeView](https://docs.oracle.com/javase/8/docs/api/java/nio/file/attribute/BasicFileAttributeView.html)
* [DosFileAttributeView](https://docs.oracle.com/javase/8/docs/api/java/nio/file/attribute/DosFileAttributeView.html)
* [PosixFileAttributeView](https://docs.oracle.com/javase/8/docs/api/java/nio/file/attribute/PosixFileAttributeView.html)
* [AclFileAttributeView](https://docs.oracle.com/javase/8/docs/api/java/nio/file/attribute/AclFileAttributeView.html)
* [UserDefinedFileAttributeView](https://docs.oracle.com/javase/8/docs/api/java/nio/file/attribute/UserDefinedFileAttributeView.html)

## 文件的读取、修改与创建
### OpenOption参数
OpenOption参数是进行文件内容操作的可选参数，其中`StandardOpenOption`支持：

* READ
* WRITE
* APPEND
* TRUNCATE_EXISTING：只有在支持WRITE时生效
* CREATE
* CREATE_NEW：只有文件不存在时才创建
* DELETE_ON_CLOSE：
* SPARSE
* SYNC：内容及元数据同步
* DSYNC：仅仅内容同步

### 常用操作
主要有`readAllBytes`、`readAllLines`、`write(Path, byte[])`、`write(Path, Iterable<? extends CharSequence>`，详见下面的demo：

``` java
Path path = Paths.get(System.getProperty("user.home"), "text");

// 读取所有字节数组
byte[] bytes = Files.readAllBytes(path);
ByteArrayInputStream inputStream = new ByteArrayInputStream(bytes);

Scanner scanner = new Scanner(new BufferedReader(new InputStreamReader(inputStream)));
while (scanner.hasNextLine()) {
    System.out.println(scanner.nextLine());
}

// 读取所有行
List<String> lines = Files.readAllLines(path);
lines.forEach(System.out::println);
scanner.close();
        
// 默认的OpenOption：CREATE，TRUNCATE_EXISTING，WRITE
Files.write(path, "Hello World".getBytes());

Files.write(path, Arrays.asList(new String[]{"Hello World", "Hello NIO"}));
```

### 面向具有缓存功能io的操作
``` java
Path path = Paths.get(System.getProperty("user.home"), "text");

// 创建BufferedReader
BufferedReader reader = Files.newBufferedReader(path);
String line = null;
while ((line = reader.readLine()) != null) {
    System.out.println(line);
}
reader.close();

BufferedWriter write = Files.newBufferedWriter(path);
write.write("Hello NIO");
write.close();
```

### 面向不具有缓存功能io的操作
``` java
Path path = Paths.get(System.getProperty("user.home"), "text");

InputStream inputStream = Files.newInputStream(path);
Scanner scanner = new Scanner(new BufferedReader(new InputStreamReader(inputStream)));
while (scanner.hasNextLine()) {
    System.out.println(scanner.nextLine());
}
scanner.close();

OutputStream outputStream = Files.newOutputStream(path);
PrintWriter writer = new PrintWriter(new BufferedWriter(new OutputStreamWriter(outputStream)));
writer.write("Hello World");
writer.close();
```

### 面向管道的操作
普通流每次读取一个字符，而管道每次读取一块缓存，这也是NIO与传统IO一个重要的区别。

#### 读取缓存字节
``` java
Path path = Paths.get(System.getProperty("user.home"), "text");

SeekableByteChannel channel = Files.newByteChannel(path);

ByteBuffer buffer = ByteBuffer.allocate(3);
while (channel.read(buffer) > 0) {
    buffer.flip();
    System.out.println(Charset.forName(System.getProperty("file.encoding")).decode(buffer));
    buf.compact();
}

channel.close();
```

#### 写入缓存字节
管道可以读，也可以写。通过Files创建管道时默认是READ，当需要写操作是需要传入WRITE操作
``` java
Path path = Paths.get(System.getProperty("user.home"), "text");

// 设置可写操作
SeekableByteChannel channel = Files.newByteChannel(path, EnumSet.of(CREATE, WRITE));

ByteBuffer buffer = ByteBuffer.wrap("Hello World".getBytes());
channel.write(buffer);
channel.close();
```

### 创建文件
``` java
Path path = Paths.get(System.getProperty("user.home"), "text");
        
Files.createFile(path, asFileAttribute(fromString("rw-r-----")));
```

#### 创建临时文件
``` java
Path file = createTempFile(null, ".xxy");
System.out.println(file);

PosixFileAttributes attrbutes = readAttributes(file, PosixFileAttributes.class);
// 默认权限为 "rx-------"
System.out.println(attrbutes.permissions());
```

## 文件目录
``` java
// 系统根目录
FileSystems.getDefault().getRootDirectories().forEach(System.out::println);

// 创建目录或递归创建目录
Files.createDirectory(Paths.get(System.getProperty("user.home"), "tmp"));
Files.createDirectories(Paths.get(System.getProperty("user.home"), "tmp/a/b/c"));

// 遍历目录内容
Files.newDirectoryStream(Paths.get(System.getProperty("user.home"))).forEach(System.out::println);

// 目录内容过滤，当然你也可以创建自己的过滤器
Files.newDirectoryStream(Paths.get(System.getProperty("user.home")), ".py").forEach(System.out::println);
```

## 递归遍历文件夹
在日常应用中经常需要递归遍历某个文件夹寻找某个文件、或进行访问操作，这时可以使用Files提供的`walkFileTree`方法，不过你需要实现`FileVisitor`接口，自定义遍历时的逻辑。

### FileVisitor
FileVisitor接口定义了遍历文件树时的具体行为：访问一个文件时，访问一个目录前，访问一个目录后，以及访问出错时。

访问文件树时可支持自定义深度：`walkFileTree(Path, Set<FileVisitOption>, int, FileVisitor)`

### `FileVisitor`的实现
实现具体的遍历行为时，可以通过继承`SimpleFileVisitor`，但需要注意：

1. 递归删除文件时，那么就需要*在删除了目录里内容后，再删除目录本身*。
2. 递归创建文件时，顺序则相反
3. 递归修改文件权限时同样需要考虑，因为修改权限后，可能会导致你无法访问。

### 便利流程的控制
[FileVisitResult](https://docs.oracle.com/javase/8/docs/api/java/nio/file/FileVisitResult.html)是FileVisitor中每个方法的返回值，包括：继续（CONTINUE），终止（TERMINATE），跳过此目录（SKIP_SUBTREE），跳过同级文件或目录（SKIP_SIBLINGS）。

## 文件搜索
搜索文件可以理解为：递归遍历文件夹，然后一一进行对文件名称匹配。由于在NIO中的核心模型为Path（传统io为File），所以NIO中同样提供了用于路径匹配的`PathMatcher`：
``` java
public interface PathMatcher {
    
    boolean matches(Path path);
}
```

### 路径匹配
PathMatcher可以用过`FileSystem`的[getPathMatcher(String)](https://docs.oracle.com/javase/8/docs/api/java/nio/file/FileSystem.html#getPathMatcher-java.lang.String-)方法实例化。

创建PathMatcher时的语法为：`syntax:pattern`。FileSystem支持[glob](http://docs.oracle.com/javase/tutorial/essential/io/fileOps.html#glob)、[regex](http://docs.oracle.com/javase/tutorial/essential/regex/index.html)两种语法。

#### glob
* `*`表示匹配0个或多个字符（但没有跨越目录）
* `**`表示匹配0个或多个字符（跨目录）
* `?`表示匹配0个或1个字符
* `\`为转义字符，用来匹配`*`、`?`等
* `[ ]`表示匹配一组字符中的1个字符，比如“[abc]”可以匹配“a”、“b”或者“c”。
  常常结合`-`一起使用，表示范围。如“[a-z]”可以匹配从“a”到“z”的所有单个字符。
  有时也会搭配`!`来使用表示取反，如[!a-c]表示可以匹配任何字符但除了“a”、“b”、“c”。

  *注意：* 方括号中的`*`，`?`，`\`不在具有上述统配意义，将匹配它们本身表示的字符。而`-`符号为第一字符，也就是不再表示范围时，也表示它本身代表的字符。
* `{ }`表示匹配一组字符串中的任意一个，但不能嵌套使用。`,`为分隔符。
* `.`开头的字符串视为常规的字符，不要理解为正则中的任意字符。

比如我们要匹配所有java文件，可以这么来创建：
``` java
PathMatcher matcher = FileSystems.getDefault().getPathMatcher("glob:a.java");

// 一定要使用 getFileName
if (matcher.matches(file.getFileName())) {
    // TODO something
}
```
需要注意的是，一定要使用`getFileName`，因为file一般表示从root到文件中的所有元素。

#### regex
参见[oracle上的教程](http://docs.oracle.com/javase/tutorial/essential/regex/index.html)。

## 与`java.io.File`的比较
JDK7发布前File是主要的I/O类，但它具有很多缺点：

* 很多方法失败时不会抛出明确异常。比如文件删除失败时，你无法得知是文件不存在还是没有权限。
* 不能友好的支持文件连接。
* 不能全面的支持文件元数据，如Ownner，permission等。
* 不能友好的支持递归遍历目录。

### Path与File的转化
``` java
Path input = file.toPath();
Files.delete(fp);
```

其他场景的转化参见：[官方文档](http://docs.oracle.com/javase/tutorial/essential/io/legacy.html#mapping)























## 参考资料
* [http://tutorials.jenkov.com/java-nio/index.html](http://tutorials.jenkov.com/java-nio/index.html)




