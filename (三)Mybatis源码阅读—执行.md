# (三)Mybatis源码阅读—执行

### 一、SqlSession

前面我们已经讲过，mybatis使用SqlSessionFactory的openSession方法创建了一个SqlSession对象，类型是DefaultSqlSession

```java
private final Configuration configuration;
private final Executor executor;
private final boolean autoCommit;
private boolean dirty;
private List<Cursor<?>> cursorList;

public DefaultSqlSession(Configuration configuration, Executor executor, boolean autoCommit) {
    this.configuration = configuration;
    this.executor = executor;
    this.dirty = false;
    this.autoCommit = autoCommit;
}
public DefaultSqlSession(Configuration configuration, Executor executor) {
    this(configuration, executor, false);
}
//以下省略
```

从源码中我们可以看到，DefaultSqlSession主要包含以下两个信息

- configuration：配置文件信息
- executor：执行器，默认是SimpleExecutor

### 二、执行过程

下面我们以从数据库查询一条数据为例

##### 1.DefaultSqlSession#selectOne

```java
@Override
public <T> T selectOne(String statement, Object parameter) {
    // Popular vote was to return null on 0 results and throw exception on too many.
    List<T> list = this.selectList(statement, parameter);
    if (list.size() == 1) {
    return list.get(0);
    } else if (list.size() > 1) {
    throw new TooManyResultsException("Expected one result (or null) to be returned by selectOne(), but found: " + list.size());
    } else {
    return null;
    }
}
```

这里第一个参数是statementId或者mapper接口的方法名，第二个参数是sql语句的参数。这里它调用了本身的selectList方法

##### 2.DefaultSqlSession#selectList

```java
@Override
public <E> List<E> selectList(String statement, Object parameter) {
    return this.selectList(statement, parameter, RowBounds.DEFAULT);
}

@Override
public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
    try {
        //拿到configuration中的MappedStatement
        MappedStatement ms = configuration.getMappedStatement(statement);
        return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
    } catch (Exception e) {
        throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
    } finally {
        ErrorContext.instance().reset();
    }
}
```

`selectList(String statement, Object parameter)`调用了本身的重载方法`selectList(String statement, Object parameter, RowBounds rowBounds)`

- rowBounds 查询的范围，默认0-2147483647

##### 3.调用Executor的query方法

```java
@Override
public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
    try {
        //拿到configuration中的MappedStatement
        //MappedStatement就是对mapper配置文件一条执行语句的封装
        MappedStatement ms = configuration.getMappedStatement(statement);
        return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
    } catch (Exception e) {
        throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
    } finally {
        ErrorContext.instance().reset();
    }
}
```

首先要注意的是，这里的`executor`是`CachingExecutor`类型的。

当开一个会话时，一个`SqlSession`对象会使用一个`Executor`对象来完成会话操作，如果用户配置了`cacheEnabled=true`，那么`MyBatis`在为`SqlSession`对象创建`Executor`对象时，会对`Executor`对象加上一个装饰者：`CachingExecutor`,且`cacheEnabled`是默认为true的。以下是节选`Confiiguration`的源码

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
    //如果cacheEnabled==true,那么返回的则是装饰类CachingExecutor
    if (cacheEnabled) {
        executor = new CachingExecutor(executor);
    }
    executor = (Executor) interceptorChain.pluginAll(executor);
    return executor;
}
```



这时SqlSession使用CachingExecutor对象来完成操作请求。CachingExecutor对于查询请求，会先判断该查询请求在Application级别的二级缓存中是否有缓存结果，如果有查询结果，则直接返回缓存结果；如果缓存中没有，再交给真正的Executor对象来完成查询操作，之后CachingExecutor会将真正Executor返回的查询结果放置到缓存中，然后在返回给用户。 

这里的`cache`是需要在mapper配置文件中配置`<cache type=""></cache>`,以开启二级缓存，type不写就默认使用mybatis本身提供的缓存功能，当然也可以使用第三方缓存

```java
@Override
public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
    BoundSql boundSql = ms.getBoundSql(parameterObject);
    CacheKey key = createCacheKey(ms, parameterObject, rowBounds, boundSql);
    return query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
}

@Override
public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
    throws SQLException {
    Cache cache = ms.getCache();
    if (cache != null) {
        flushCacheIfRequired(ms);
        if (ms.isUseCache() && resultHandler == null) {
            ensureNoOutParams(ms, boundSql);
            @SuppressWarnings("unchecked")
            //在缓存中查询
            List<E> list = (List<E>) tcm.getObject(cache, key);
            if (list == null) {
                //如果缓存未命中，使用真正的执行器处理请求
                list = delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
                //将查询结果放入缓存
                tcm.putObject(cache, key, list); // issue #578 and #116
            }
            return list;
        }
    }
     //如果缓存为空，使用真正的执行器处理请求
    return delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
}
```

##### 4.BaseExecutor#query

```java
@Override
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
    if (closed) {
        throw new ExecutorException("Executor was closed.");
    }
    if (queryStack == 0 && ms.isFlushCacheRequired()) {
        clearLocalCache();
    }
    List<E> list;
    try {
        queryStack++;
        //先去缓存中查，缓存命中失败再去数据库查询
        list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
        if (list != null) {
            handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
        } else {
            list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
        }
    } finally {
        queryStack--;
    }
    if (queryStack == 0) {
        for (DeferredLoad deferredLoad : deferredLoads) {
            deferredLoad.load();
        }
        // issue #601
        deferredLoads.clear();
        if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
            // issue #482
            clearLocalCache();
        }
    }
    return list;
}
```

这里的逻辑也比较简单，先去查询缓存，这里指的是一级缓存，查询成功则直接返回；若缓存命中失败，则从数据库中查询，再将查询结果加入到缓存当中。

##### 5.BaseExecutor#queryFromDatabase

```java
private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    List<E> list;
    localCache.putObject(key, EXECUTION_PLACEHOLDER);
    try {
        list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
    } finally {
        localCache.removeObject(key);
    }
    //将查询结果加入缓存
    localCache.putObject(key, list);
    if (ms.getStatementType() == StatementType.CALLABLE) {
        localOutputParameterCache.putObject(key, parameter);
    }
    return list;
}
```

##### 6.SimpleExecutor#doQuery

```java
@Override
public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
    Statement stmt = null;
    try {
        //获取配置文件信息
        Configuration configuration = ms.getConfiguration();
        StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
        stmt = prepareStatement(handler, ms.getStatementLog());
        return handler.query(stmt, resultHandler);
    } finally {
        closeStatement(stmt);
    }
}

private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
    Statement stmt;
    //获取数据库连接
    Connection connection = getConnection(statementLog);
    // 创建 Statement
    stmt = handler.prepare(connection, transaction.getTimeout());
    //为Statement设置参数
    handler.parameterize(stmt);
    return stmt;
}

@Override
public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
    PreparedStatement ps = (PreparedStatement) statement;
    ps.execute();
    //resultSetHandler结果集处理
    return resultSetHandler.handleResultSets(ps);
}
```

我们前面说过，`StatementHandler`是一个很重要的类，负责处理与数据库的交互的信息。由`Configuration`来实现。

- `Statement`对象我们可以理解为对Sql语句的封装
- `Connection`对象是与数据库连接的必要条件，mybatis底层是如何获取连接我们之后再详细讨论，这里只需要了解过程即可
- resultSetHandler对象是对查询结果的处理。

最后，需要注意的是，在方法结束的时候会调用` closeStatement(stmt);`，这说明SimpleExecutor对象在每处理一条语句就会创建一个Statement对象，用完之后关闭。