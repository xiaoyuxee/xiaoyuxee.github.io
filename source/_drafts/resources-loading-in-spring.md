---
title: Spring
date: 2016-02-03 15:55:03
tags:
categories:
toc: true
---

Spring中需要加载各种资源，包括XML中bean的配置等，而Java自带的`java.net.URL`并不能满足Application中的资源定位。Spring中将资源统称为`Resource`。

本文就Spring中的资源及资源加载展开探索。

<!-- more -->

ApplicationContext与BeanFactory的区别之一就是前者具有资源加载的功能。`ResourcePatternResolver`扩展于`ResourceLoader`，是ApplicationContext继承的接口，用于定位、解析资源。具体实现为`PathMatchingResourcePatternResolver`。

## `ResourceLoader`

### 接口说明

``` java
public interface ResourceLoader {

    // 查询指定路径的资源.
     
    // Must support fully qualified URLs, e.g. "file:C:/test.dat".
    // Must support classpath pseudo-URLs, e.g. "classpath:test.dat".
    // Should support relative file paths, e.g. "WEB-INF/test.dat".
    // (This will be implementation-specific, typically provided by an
    // ApplicationContext implementation.)
    
    Resource getResource(String location);

    ClassLoader getClassLoader();
}
```

### 默认实现

`ResourceLoader`的默认实现为`DefaultResourceLoader`，`AbstractApplicationContext`继承于`DefaultResourceLoader`实现指定路径的资源加载。

#### 默认的`ClassLoader`

资源加载必然离不开`ClassLoader`，`DefaultResourceLoader`的默认class loader是通过`ClassUtils`来初始化的。

``` java
public DefaultResourceLoader() {
    // 默认classLoader
    this.classLoader = ClassUtils.getDefaultClassLoader();
}

public ClassLoader getClassLoader() {
    return (this.classLoader != null ? this.classLoader : ClassUtils.getDefaultClassLoader());
}
```

``` java
public static ClassLoader getDefaultClassLoader() {
    ClassLoader cl = null;
    try {
    	// 当前线程的context ClassLoader
    	cl = Thread.currentThread().getContextClassLoader();
    }
    catch (Throwable ex) {
    	// Cannot access thread context ClassLoader - falling back...
    }
    if (cl == null) {
    	// No thread context class loader -> use class loader of this class.
    	cl = ClassUtils.class.getClassLoader();
    	if (cl == null) {
    		// getClassLoader() returning null indicates the bootstrap ClassLoader
    		try {
    			cl = ClassLoader.getSystemClassLoader();
    		}
    		catch (Throwable ex) {
    			// Cannot access system ClassLoader - oh well, maybe the caller can live with null...
    		}
    	}
    }
    return cl;
}
```
可以看出会依次获取当前线程的context classLoader、`ClassUtils`的classLoader、jvm系统ClassLoader，直到发现可用的为止。

#### 如何加载指定资源

``` java
public Resource getResource(String location) {
    Assert.notNull(location, "Location must not be null");
    if (location.startsWith("/")) {
        return getResourceByPath(location);
    }
    else if (location.startsWith(CLASSPATH_URL_PREFIX)) {
        return new ClassPathResource(location.substring(CLASSPATH_URL_PREFIX.length()), getClassLoader());
    }
    else {
    	try {
            // Try to parse the location as a URL...
            URL url = new URL(location);
            return new UrlResource(url);
    	}
    	catch (MalformedURLException ex) {
            // No URL -> resolve as resource path.
            return getResourceByPath(location);
    	}
    }
}

protected Resource getResourceByPath(String path) {
    return new ClassPathContextResource(path, getClassLoader());
}
```

`DefaultResourceLoader`加载资源是通过根据不同路径返回不同`Resource`来实现。具体资源内容可通过相应的`Resource`提供的api进行获取。

## `ResourcePatternResolver`

### 接口说明

``` java
public interface ResourcePatternResolver extends ResourceLoader {
    String CLASSPATH_ALL_URL_PREFIX = "classpath*:";

    Resource[] getResources(String locationPattern) throws IOException;
}
```

### 默认实现
`PathMatchingResourcePatternResolver`


  


