---
title: Java应用中的数据库访问
date: 2016-01-24 13:56:04
tags: JDBC
categories:
toc: true
---


早起对数据库的访问，都是直接调用数据库厂商提供的专有API。[ODBC(Open Database Connectivity)](https://zh.wikipedia.org/zh-cn/ODBC)是微软开放服务结构（WOSA，Windows Open Service Architecture）中有关数据库的一部分，提供了Windows下统一的数据库访问方式。使用者只需要调用ODBC API，由ODBC驱动程序将调用请求转化为对特定数据库的调用请求。

Java语言问世后，Sun公司与1996年推出了[JDBC(Java Database Connectivity)](https://zh.wikipedia.org/zh-cn/Java%E6%95%B0%E6%8D%AE%E5%BA%93%E8%BF%9E%E6%8E%A5)，提供了对数据库访问的统一方式。JDBC是一套标准的访问关系数据库的Java类库，同时为数据库厂商提供了一个标准的API，让厂商为自己的数据库产品提供相应的JDBC驱动程序。应用程序调用JDBC API，由JDBC驱动程序（具体数据库厂商的实现层）处理与数据库的通信，从而使应用程序与具体数据库产品解耦。

<!-- more -->

## 加载注册数据库驱动

### Drive接口

`javax.sql.Driver`是所有JDBC驱动程序需要实现的接口。

其中，`connect(url, info)`方法用于建立到数据到的连接。而在实际的应用程序中，不需要直接调用此方法，而是通过JDBC驱动程序管理器`DriverManager`注册相应的驱动程序，使用驱动管理器来建立数据库连接。

### 加载注册JDBC驱动

加载JDBC驱动通过`Class.forName(String className)`在CLASSPATH中定位、加载驱动类。

注册JDBC驱动实例则是通过`Driver.registerDriver(Driver driver)`来完成。通常，不需要我们亲自去注册，因为实现了`Driver`的驱动类都包含一个静态区，调用驱动管理器的静态方法来注册自己的一个实例。

``` java
public class Driver extends NonRegisteringDriver implements java.sql.Driver {

    static {
        try {
            java.sql.DriverManager.registerDriver(new Driver());
        } catch (SQLException E) {
            throw new RuntimeException("Can't register driver!");
        }
    }

    public Driver() throws SQLException {
        // Required for Class.forName().newInstance()
    }
}
```

## 建立到数据库的连接

通过`DriverManager.getConnection(String url, String user, String password)`建立到数据库的连接（代理给相应的驱动程序）

### JDBC URL

`jdbc:subprotocol:subname`：

* 协议：jdbc，唯一允许的协议
* 子协议：标识一个数据库驱动程序，如mysql、sqlserver
* 子名称：与具体数据库驱动有关，如mysql中：jdbc:mysql://localhost:3306/database

## 数据库访问

数据库访问通过建立的连接的来访问，有3种方式`Statement`、`PreparedStatement`、`CallableStatement`。

### Statement

用于执行静态的SQL语句，通过`Connection.createStatement()`来创建。

``` java
public interface Statement {
    /** 执行查询语句 */
    ResultSet executeQuery(String sql) throws SQLException;

    /** 用于执行INSERT、UPDATE、DELETE等语句 */
    int executeUpdate(String sql) throws SQLException;

    /** 通过`addBatch()`批量添加sql命令，然后一起执行 */
    int[] executeBatch() throws SQLException;
}
```

### ResultSet

`ResultSet`以*逻辑表格*封装了数据库执行结果，由数据库厂商来实现。

``` java
public interface ResultSet {
    /** 将游标移动到下一行，如果该行有数据返回`true`，否则返回`false` */
    boolean next() throws SQLException;

    /** 通过索引（1开始）查看某列数据 **/
    String getString(int columnIndex) throws SQLException;

    /** 通过列名称查看某列数据 **/
    String getString(String columnLabel) throws SQLException;
}
```

### PreparedStatement

sql语句在执行以前需要预编译，包括语句分析、代码优化等。如果仅仅是参数不同的sql语句，可以使用`PreparedStatement`。

### ResultSetMetaData

用于描述数据库表结构的元数据， `ResultSet.getMetaData()`

## 事务处理

* 脏读（dirty read）
  一个事务对数据进行了修改，但没有提交，与此同时另外一个事务读取了被修改的数据。如若前一个事务发生回滚，那么后一个事务读取的数据也就是无效数据。
* 不可重复读（non-repeatable read）
  一个事务读取了一行数据，在事务结束以前另外一个事务对这行数据进行了修改，那么当前一个事务再次读取那部分数据时，得到了不同的数据。
* 幻读（phantom read）
  一个事务查询某条件下的数据，事务结束之前另外一个事务又插入一些满足条件的数据，那么当第一个事务再次查询时发现数据多出几行。

### `Connection`中关于事务隔离级别的常量

``` java
public interface Connection {
    /** 不支持事务 */
    int TRANSACTION_NONE             = 0;

    /** 允许脏读 */
    int TRANSACTION_READ_UNCOMMITTED = 1;

    /** 不允许脏读，但允许不可重复我和幻读 */
    int TRANSACTION_READ_COMMITTED   = 2;

    /** 不允许脏读、不可重复读，但允许幻读 */
    int TRANSACTION_REPEATABLE_READ  = 4;

    /** 脏读、不可重复读、幻读均不允许 */
    int TRANSACTION_SERIALIZABLE     = 8;
}
```

mysql默认级别为：`TRANSACTION_READ_COMMITTED`, 禁止脏读、不可重复读。

而事务默认为*自动提交*，可通过`Connection.setAutoCommit(false)`来重置，自行提交（commit）或回滚（rollback）。

## JDBC数据源和连接池

对数据库的访问除了加载、实例化驱动程序并通过驱动程序管理器获得连接外，还可通过`DataSource`来实现（由数据库厂商实现）。

### 什么是连接池

建立数据库连接的成本是很大的，并且一个数据库服务器能够同时建立的连接数是有限的。在web应用中可能同时会有成千上万个访问数据库的请求，如果为每个请求创建一个数据库连接，性能将急剧下降。为了能够*重复利用数据库连接*，提高对请求的响应时间和服务器性能，于是诞生了数据库连接池。

数据库连接池预先建立了一些数据库连接，然后保存到连接池中，当有访问数据库的请求时，从池中取出一个闲置的连接对象完成对数据的访问，请求结束后将连接对象放回池中。

调用物理连接的`close`方法将关闭连接，而连接池的连接的`close`方法为释放连接对象放回连接池中。

大部分servlet容器都支持基于JNDI的数据库连接池的配置，如[Tomcat](http://tomcat.apache.org/tomcat-9.0-doc/jndi-datasource-examples-howto.html)，[Jetty](http://www.eclipse.org/jetty/documentation/current/jndi-datasource-examples.html)。

### 连接池实现

* [Apache Commons DBCP](http://commons.apache.org/proper/commons-dbcp/)
* [c3p0](http://sourceforge.net/projects/c3p0/)
* [bonecp](https://github.com/wwadge/bonecp)


