---
layout: post
comments: true
title: "log4j源码心得"
description: "learn log4j source"
categories: ["java", "log4j", "source reading"]
---

log4j是一个非常流行的日志框架，最近研究了一下log4j的源代码，这里记录一下心得体会。

### Logger、Appender、Layout之间的结构关系?

这是log4j的基本数据结构，之间的关系还是很好理解的。首先，一个Logger都要一个名字(通常是类名或包名),它是有父子结构的，相关的实现是Hierarchy。  
一个Logger会对应多个Appender，表示可以日志输出的地方。  
Appender的输出格式是通过Layout来实现的，Logger在输出的时候是包装一个LogEvent对象，有Layout来格式化成一个字符串。

简单点:
```xml
Logger 1--* Appender 1--* Layout
```

### 加载log4j的配置文件

先考虑加载的是log4j.xml, 然后才是log4j.properties。我平时主要使用log4j.properties.

### log4j果然是年代久远

以前都说log4j在jdk1.1版本就存在了，现在从源码的细节上看，的确是这样的。例如:

1. 加载配置文件的时候，使用context class loader，它不是用标准的api，而是采用反射。
2. 处理FileAppender的时候，针对父级目录不存在的时候需要创建目录，如果是我就直接使用mkdirs，它确是手工处理。
3. Logger的名字是通过CategoryKey作为key存储到HashTable的，按理说用String作为key就可以了，估计是很早之前String的hashcode是不带缓存实现的吧。
4. 

### 其他细节

#### 没有配置文件的时候，如何优雅处理?

正常的时候，会在static初始化的时候得到LogRepositorySelector，如果失败的话，这个值就是空的。  
然后就生成一个repositorySelector = new DefaultRepositorySelector(new NOPLoggerRepository());，这个处理方式就是进行空输出。

#### log4j配置中的占位符处理

log4j配置中的占位符处理是在OptionConverter的substVars方法处理。优先使用系统变量，然后才是properties配置的定义
log4j.threshold用来配置log level的阀值，就是最低级别。默认是不设置。

#### log4j在重复加载文件时的处理

log4j在重复加载的时候，有个做法可以学习一下。这个做法和commons-logging的方式是一样的。

```
  void doConfigure(java.net.URL configURL, LoggerRepository hierarchy) {
    Properties props = new Properties();
    LogLog.debug("Reading configuration from URL " + configURL);
    InputStream istream = null;
    URLConnection uConn = null;
    try {
      uConn = configURL.openConnection();
      uConn.setUseCaches(false);
      istream = uConn.getInputStream();
      props.load(istream);
    }
```

#### layout或者appender的参数设置

log4j配置中的有时候组成一个layout或者appender，都是可以设置参数的，这些是怎么对应到类中的变量的? 实现在
```
 PropertySetter.setProperties(layout, props, layoutPrefix + ".");
```
通常还需要实现OptionHandler来做一些后续的初始化动作.

#### appender filter如何排序?

log4j的appender filter是可以配置多个的，处理顺序是按照对应的key的排序来的

```
    // sort filters by IDs, insantiate filters, set filter options,
    // add filters to the appender
    Enumeration g = new SortedKeyEnumeration(filters);
```

但是它为什么不使用内置的Arrays sort或者Collections sort呢?
```
  public SortedKeyEnumeration(Hashtable ht) {
    Enumeration f = ht.keys();
    Vector keys = new Vector(ht.size());
    for (int i, last = 0; f.hasMoreElements(); ++last) {
      String key = (String) f.nextElement();
      for (i = 0; i < last; ++i) {
        String s = (String) keys.get(i);
        if (key.compareTo(s) <= 0) break;
      }
      keys.add(i, key);
    }
    e = keys.elements();
  }
```
