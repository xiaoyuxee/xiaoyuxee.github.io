---
title: 纯Java-Code配置DispatcherServlet
date: 2016-01-16 00:08:30
tags: spring-mvc
categories: spring
toc: true
---

大家都知道`DispatcherServelt`是 *spring-mvc* 中的前端控制器。按照传统的方式，可以在`web.xml`中配置这个 *servlet* 。

而在 *Servlet 3.0* 以后，支持通过注解配置 *servlet*、*listener*、*filter*等，同时也支持以注解方式配置*servlet container*，达到与配置`web.xml`一样的效果。

这篇文章主要介绍下*如何在Servlet 3.0下以 Java-Code 配置Servlet Container以及Dispatcher Servlet*。

<!-- more -->

## 传统web.xml中定义Servlet

通过`web.xml`来配置一个web应用时，一般是这样子的：

``` xml code.1-web.xml
<web-app xmlns="http://java.sun.com/xml/ns/javaee" version="2.5">
    <servlet>
        <servlet-name>dispatcher</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>/WEB-INF/spring/dispatcher-config.xml</param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>

    <servlet-mapping>
        <servlet-name>dispatcher</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping>
</web-app>
```

## ServletContainerInitializer

Servlet 3.0中，定义了一个接口：`javax.servlet.ServletContainerInitializer`，在Servlet 3.0下，将会在 *classpath* 下寻找该接口的实现，然后以此来配置Servlet Container。

> Implementations of this interface may be annotated with `HandlesTypes`, in order to receive (at their `onStartup` method) the Set of application classes that implement, extend, or have been annotated with the class types specified by the annotation. 

> Implementations of this interface must be declared by a JAR file resource located inside the `META-INF/services` directory and named for the fully qualified class name of this interface, and will be discovered using the runtime's service provider lookup mechanism or a container specific mechanism that is semantically equivalent to it.

通过其注释不难得知：

1. 被`@HandlesTypes`注解的接口的 *实现* 将被以参数的方式传给其方法 `onStartup`
2. 具体实现必须在JAR包的Spring中的 *META-INF/services* 下声明其实现。它的运行时发现机制其实是通过`ServiceLoader`实现的，具体参考官方文档[Service Provider](http://docs.oracle.com/javase/8/docs/technotes/guides/jar/jar.html#Service_Provider)

## SpringServletContainerInitializer

### provider声明

在 *spring-web* 包下的 *META-INF/services* 中定义有：

``` properties code.2-javax.servlet.ServletContainerInitializer
org.springframework.web.SpringServletContainerInitializer
```

### 接口定义

``` java SpringServletContainerInitializer.java
@HandlesTypes(WebApplicationInitializer.class)
public class SpringServletContainerInitializer implements ServletContainerInitializer {
    
    @Override
    public void onStartup(Set<Class<?>> webAppInitializerClasses, ServletContext servletContext) throws ServletException {

    }

}
```

### onStartup 具体实现

``` java code.3-SpringServletContainerInitializer.onStartup
List<WebApplicationInitializer> initializers = new LinkedList<WebApplicationInitializer>();

if (webAppInitializerClasses != null) {
    for (Class<\?> waiClass : webAppInitializerClasses) {

        // servlet container传入的参数均为具体实现，而非接口或抽象类
        if (!waiClass.isInterface() && !Modifier.isAbstract(waiClass.getModifiers())
            && WebApplicationInitializer.class.isAssignableFrom(waiClass)) {

            try {
                initializers.add((WebApplicationInitializer) waiClass.newInstance());
            }
            catch (Throwable ex) {
                throw new ServletException("Failed to instantiate WebApplicationInitializer class", ex);
            }
        }
    }
}

...

// 初始化 Servlet Context
for (WebApplicationInitializer initializer : initializers) {
    initializer.onStartup(servletContext);
}
```

## 通过java code初始化Servlet Context

初始化 *Servlet Context* 的工作其实是委托给了`WebApplicationInitializer`的实现类，那么我们就可以自定义其实现过程，如：

``` java code.4-MyWebAppInitializer.java
public class MyWebAppInitializer implements WebApplicationInitializer {
    
    @Override
    public void onStartup(ServletContext container) {
        XmlWebApplicationContext appContext = new XmlWebApplicationContext();
        appContext.setConfigLocation("/WEB-INF/spring/dispatcher-config.xml");

        ServletRegistration.Dynamic dispatcher = 
        container.addServlet("dispatcher", new DispatcherServlet(appContext));

        dispatcher.setLoadOnStartup(1);
        dispatcher.addMapping("/");
    }

}
```

这样，我们就完全取代了`web.xml`。但是，还是`/WEB-INF/spring/dispatcher-config.xml`还是XML配置，能否也用 *Java-Code* 替代呢？

答案是肯定的，因为Spring 3.0就支持通过注解`@Configuration`来实现以往通过XML形式配置的工作了：

``` java code.5-MyWebAppInitializer.java
public class MyWebAppInitializer implements WebApplicationInitializer {

    @Override
    public void onStartup(ServletContext container) {
        // Create the 'root' Spring application context
        AnnotationConfigWebApplicationContext rootContext =
        new AnnotationConfigWebApplicationContext();
        rootContext.register(AppConfig.class);

        // Manage the lifecycle of the root application context
        container.addListener(new ContextLoaderListener(rootContext));

        // Create the dispatcher servlet's Spring application context
        AnnotationConfigWebApplicationContext dispatcherContext =
        new AnnotationConfigWebApplicationContext();
        dispatcherContext.register(DispatcherConfig.class);

        // Register and map the dispatcher servlet
        ServletRegistration.Dynamic dispatcher =
        container.addServlet("dispatcher", new DispatcherServlet(dispatcherContext));
        dispatcher.setLoadOnStartup(1);
        dispatcher.addMapping("/");
    }

 }
```

以上的demo仅为简单实现，那么Spring中又是如何设计的呢？

## Spring中是如何做的

分析`code.5`，其实变化的、需要用户具体指定的内容有3块：

1. rootConfig：根容器配置
2. dispatcherConfig：dispatcher配置
3. mapping：dispatcherServlet匹配规则

* AbstractContextLoaderInitializer

``` java code.6-AbstractContextLoaderInitializer.java
public abstract class AbstractContextLoaderInitializer implements WebApplicationInitializer {
    
    @Override
    public void onStartup(ServletContext servletContext) throws ServletException {
        registerContextLoaderListener(servletContext);
    }

    protected void registerContextLoaderListener(ServletContext servletContext) {
        WebApplicationContext rootAppContext = createRootApplicationContext();
        
        // 注册root ApplicationContext
    }

    protected abstract WebApplicationContext createRootApplicationContext();

    protected ApplicationContextInitializer<?>[] getRootApplicationContextInitializers() {
        return null;
    }

}
```

* AbstractDispatcherServletInitializer

``` java code.7-AbstractDispatcherServletInitializer.java
public abstract class AbstractDispatcherServletInitializer extends AbstractContextLoaderInitializer {
   
    @Override
    public void onStartup(ServletContext servletContext) throws ServletException {
        super.onStartup(servletContext);

        registerDispatcherServlet(servletContext);
    }

    protected void registerDispatcherServlet(ServletContext servletContext) {
        WebApplicationContext servletAppContext = createServletApplicationContext();

        ...
    } 

    protected abstract WebApplicationContext createServletApplicationContext();

    protected ApplicationContextInitializer<?>[] getServletApplicationContextInitializers() {
        return null;
    }

    protected void customizeRegistration(ServletRegistration.Dynamic registration) {
    }

    // 用户自定义Url匹配规则
    protected abstract String[] getServletMappings();

}
```

* AbstractAnnotationConfigDispatcherServletInitializer

``` java code.8-AbstractAnnotationConfigDispatcherServletInitializer.java
public abstract class AbstractAnnotationConfigDispatcherServletInitializer
        extends AbstractDispatcherServletInitializer {

    @Override
    protected WebApplicationContext createRootApplicationContext() {
        // create root ApplicationContext
    }

    @Override
    protected WebApplicationContext createServletApplicationContext() {
        // create servlet ApplicationContext
    }

    // 指定root ApplicationContext配置文件
    protected abstract Class<?>[] getRootConfigClasses();

    // 指定servlet ApplicationContext配置文件
    protected abstract Class<?>[] getServletConfigClasses();

}
```

## 最终配置

``` java code.9-WebAppInitializer.java
public class WebAppInitializer 
    extends AbstractAnnotationConfigDispatcherServletInitializer {

    @Override
    protected Class<?>[] getRootConfigClasses() {
        return new Class<?>[] { RootConfig.class };
    }

    @Override
    protected Class<?>[] getServletConfigClasses() {
        return new Class<?>[] { WebConfig.class };
    }

    @Override
    protected String[] getServletMappings() {
        return new String[] { "/" };
    }

}
```







