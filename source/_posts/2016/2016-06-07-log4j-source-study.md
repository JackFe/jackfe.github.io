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

