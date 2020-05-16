# （一)MyBatis源码阅读—总体分析

在开始阅读源码之前，首先我们得了解一下Mybatis的整体设计，这将有助于我们在之后的学习中更好的理解



### 一、Mybatis框架设计

![img](https://img-blog.csdn.net/20141028232313593?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbHVhbmxvdWlz/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

​																																											[图片来源](https://blog.csdn.net/luanlouis/article/details/40422941)

### 二、核心类分析

#### SqlSession类

Mybatis工作最重要的java接口，通过这个接口来执行命令，获取mapper以及管理事务 。SqlSession有两个子类，DefaultSqlSession和SqlSessionManager。这里主要讨论DefaultSqlSession。

![SqlSession](https://github.com/hanfeng21050/MyBatis/blob/master/img/SqlSession.png)

#### Executor类

Executor是mybatis的核心执行类，mybatis的大部分操作处理都是在executor上完成的

![Executor](https://github.com/hanfeng21050/MyBatis/blob/master/img/Executor.png)

 **BaseExecutor 主要是使用了模板设计模式（template）**, 共性被封装在 BaseExecutor 中 , 容易变化的内容被分离到了子类中 。 

- SimpleExecutor： MyBatis 的官方文档中对 SimpleExecutor 的说明是 "普通的执行器" , 普通就在于每一次执行都会创建一个新的 Statement 对象 
- ReuseExecutor：官方文档中的解释是“执行器会重用预处理语句（prepared statements）” ， 这次倒是解释的很详细。也就是说不会每一次调用都去创建一个 Statement 对象 ， 而是会重复利用以前创建好的（如果SQL相同的话）
- BatchExecutor： 批处理处理方案所对应的的类,用于将多个sql交给一个statement处理,防止数据库压力大时崩溃 

CachingExecutor 从名字我们可以猜测得到，这个执行器主要是专门处理缓存的（二级缓存的，之后会细讲）。

```java
protected boolean cacheEnabled = true;

public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
    executorType = executorType == null ? defaultExecutorType : executorType;
    executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
    Executor executor;
    if (ExecutorType.BATCH == executorType) {
      executor = new BatchExecutor(this, transaction);
    } else if (ExecutorType.REUSE == executorType) {
      executor = new ReuseExecutor(this, transaction);
    } else {
      executor = new SimpleExecutor(this, transaction);
    }
    //在这
    if (cacheEnabled) {
      executor = new CachingExecutor(executor);
    }
    executor = (Executor) interceptorChain.pluginAll(executor);
    return executor;
  }
```

我们知道， MyBatis在开启一个数据库会话时，会 创建一个新的SqlSession对象，SqlSession对象中会有一个新的Executor对象。从上面的代码，可以很清晰的看到， CachingExecutor其实是baseExecutor的一个包装，进一步说，也就是对SimpleExecutor、ReuseExecutor、BatchExecutor进行包装 。

```java
private final Executor delegate;
private final TransactionalCacheManager tcm = new TransactionalCacheManager();

public CachingExecutor(Executor delegate) {
    this.delegate = delegate;
    delegate.setExecutorWrapper(this);
}
```

这里 mybatis 这里对 Executor 的设计使用了 **Decorator （装饰者） 设计模式**。 

#### StatementHandler

 StatementHandler 负责操作 Statement 对象与数据库进行交流，

![StatementHandler](img\StatementHandler.png)

- RoutingStatementHandler，这是一个封装类，它不提供具体的实现，只是根据Executor的类型，创建不同的类型StatementHandler。

- SimpleStatementHandler，这个类对应于JDBC的Statement对象，用于没有预编译参数的SQL的运行。

- PreparedStatementHandler 这个用于预编译参数SQL的运行。

- CallableStatementHandler 它将实存储过程的调度。

#### ParameterHandler

 参数处理与映射，给Statement对象设置参数 

####  **resultHandler** 

 结果集处理与映射，处理Statement执行后产生的结果集ResultSet，生成结果列表List。也用作储存过程执行后的输出。 

### 三、执行流程

1. 加载配置文件并初始化(SqlSession)

   配置文件来源于两个地方，一个是配置文件(主配置文件conf.xml,mapper文件*.xml)，一个是java代码中的注解，将sql的配置信息加载成为一个mappedstatement对象，存储在内存之中（包括传入参数的映射配置，结果映射配置，执行的sql语句）。

2.  接收调用请求 

   调用mybatis提供的api，传入的参数为 statementId（ mapper接口的方法名）和sql语句的参数对象，mybatis将调用请求交给请求处理层。 

3.  处理请求 

   根据sql的id找到对应的mappedstatament对象。

   根据传入参数解析mappedstatement对象，得到最终要执行的sql。

   获取数据库连接，执行sql，得到执行结果

   释放连接资源 

4.   返回处理结果 
