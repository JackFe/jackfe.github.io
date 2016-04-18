---
layout: post
comments: true
title: "从commons-dbcp源码学习设计思路"
description: "从commons-dbcp源码学习设计思路"
categories: ["commons-dbcp", "源码", "设计"]
---

由于整个连接池的性能是由commons-pool决定的，有空再讲解一下commons-pool的实现，特别是1.x和2.x的区别。  
此次分析的是commons-dbcp 1.x源码，对应commons-pool 1.x版本。

### commons-dbcp怎样与commons-pool集成?

{% plantuml %}
package connection-datasource {
  interface DataSource {
    +getConnection()
  }

  class BasicDataSource {
    #DataSource datasource
    #GenericObjectPool connectionPool
  }

  class PoolingDataSource {
    -ObjectPool _pool
  }

  interface ObjectPool {
    +borrowObject()
    +returnObject()
    +...()
  }

  ObjectPool <|-- GenericObjectPool
  ObjectPool <|-- AbandonedObjectPool

  DataSource <|-- BasicDataSource
  DataSource <|-- PoolingDataSource
  BasicDataSource o-- PoolingDataSource : 委托给PoolingDataSource负责
  PoolingDataSource o-- ObjectPool : 使用commons-pool进行维护

  note as T2
    用于管理连接池的状态
  end note
}

package connection-factory {
  interface PoolableObjectFactory {
    +makeObject()
    +destroyObject()
    +...()
  }

  class PoolableConnectionFactory {
    -ConnectionFactory _connFactory
    -ObjectPool _pool
  }

  interface ConnectionFactory {
    +createConnection()
  }

  ConnectionFactory <|-- DriverConnectionFactory
  PoolableObjectFactory <|-- PoolableConnectionFactory
  ObjectPool o--o PoolableConnectionFactory : 连接的实际生成由Factory负责
  DriverConnectionFactory --o PoolableConnectionFactory

  note as T1
    用于管理数据库drvier与连接的生成
  end note
}
{% endplantuml %}


如上图所示，集成commons-dbcp的时候采用BasicDataSource这个实现类，它的实际功能是交给PoolingDataSource的(内部是通过commons-pool来管理连接对象)。  
不过,我不是很理解为什么要这么设计?

### commons-dbcp的连接有什么特别?
{% plantuml %}
package connection-delegating {
  Connection <|-- DelegatingConnection
  DelegatingConnection <|-- PoolGuardConnectionWrapper
  DelegatingConnection <|-- PoolingConnection
  DelegatingConnection <|-- PoolableConnection


  interface Connection {
  }

  class DelegatingConnection {
    -boolean _closed
  }

  note as N1
    DelegatingConnection:
      _closed用于标识原始的conn是否已经关闭
    PoolingConnection:
      同样由PoolableConnectionFactory的makeObject产生,只在开启statement对象池的时候出现
    PoolableConnection:
      由PoolableConnectionFactory的makeObject产生,close方法会尝试返回池
    PoolGuardConnectionWrapper:
      由PoolingDataSource产生,能够避免已关闭的连接被误用
  end note
}
{% endplantuml %}

连接这种对象有点特殊的，所以commons-dbcp提供了一些connection方面的增强特性。例如:

* PoolGuardConnectionWrapper是最终客户端拿到的对象，能够防止多次关闭等误操作
* PoolableConnection是PoolGuardConnectionWrapper内部的对象，可以结合pool进行管理，最大的优势就是可以保留客户端代码无需任何改动。**实际上，很多自带生命周期api的对象，一旦池化之后都会考虑这么设计。**
* PoolingConnection是开启statement pool的时候PoolableConnection的内部对象，内部采用一个KeyedObjectPool进行管理(key主要是通过执行的sql语句来生成的)。不过这种对象一般不需要池化

### 如何优化Connection、Statement、ResultSet的生命周期管理?

jdbc的api有个非常烦人的地方，就是每个Connection、Statement、ResultSet对象都是需要关闭。所以写起来代码繁琐的，很多人就跳过这些健壮性代码。  
我研究了一下dbcp的实现，发现它能够发现未关闭的Statement、ResultSet对象，并在适当的时候进行关闭。

{% plantuml %}
package connection-abandoned-trace {
  AbandonedTrace <|-- DelegatingConnection
  AbandonedTrace <|-- DelegatingStatement
  AbandonedTrace <|-- DelegatingResultSet
  AbandonedTrace <|-- DelegatingDatabaseMetaData

  DelegatingConnection "1" o-- "*" DelegatingStatement
  DelegatingConnection "1" o-- "*" DelegatingResultSet
  DelegatingStatement "1" o-- "*" DelegatingResultSet

  class AbandonedTrace {
    -List traceList
  }
}
{% endplantuml %}

具体实现思路是这样的:

* 需要实现生命周期管理的对象需要继承AbandonedTrace，这包括了DelegatingStatement、DelegatingResultSet、DelegatingConnection等
* 通过DelegatingConnection生成的statement、resultset等都是带Delegating的，也就是带trace特性的。
* 对于上图，有个特别的是DelegatingConnection的trace可能包括ResultSet，这个主要由DelegatingDatabaseMetaData产生的。因为metadata的查询不需要先有statement。
* 处理流程调用connection.close(), 会返回到池中(见PoolableConnection)， 触发PoolableConnectionFactory的passivateObject(commons-pool的内置回调)，最后触发DelegatingConnection的passivate，在这里会递归检查所有的trace。
* 注意的是，DelegatingConnection的close方法除了触发trace对象的关闭，还会关闭底层的连接对象。





