# (六)Mybatis源码阅读—缓存

前面我们在将mybatis执行的时候，讲过缓存问题。在查询的时候，会先从缓存查询数据，查询失败再从数据库查询，然后将查询结果返回并添加到缓存中。那么问题由出现了，Mybatis是怎么设计缓存机制的？接下来让我们就这个问题展开。

---

mybatis的缓存系统实现使用了**模板方法**模式和**装饰器模式**

下面是和mybatis缓存相关的类

![Cache](img/Cache.png)

### 一、Cache

Mybatis提供了一个Cache基本接口，来实现缓存相关的功能，对于每一个`namespace`都会创建一个缓存，然后将`namespace`本身作为id传入。

```java
public interface Cache {

    //获取缓存的id,即namespace
    String getId();

    //添加缓存
    void putObject(Object key, Object value);

    //根据key获取缓存
    Object getObject(Object key);

    //根据key移除缓存
    Object removeObject(Object key);

    //清空缓存
    void clear();

	//获取缓存中数据大小
    int getSize();

    //取得读写锁，从3.2.6开始没用了
    default ReadWriteLock getReadWriteLock() {
        return null;
    }

}
```

### 二、PerpetualCache

虽然`Cache`接口类由很多的实现类，但实际上，**只有`PerpetualCache`这一个类是真正的实现类**，其他的实现类都只是作为一个装饰者的角色，对`Cache`进行包装。

```java
public class PerpetualCache implements Cache {

    private final String id;

    private Map<Object, Object> cache = new HashMap<>();

    public PerpetualCache(String id) {
        this.id = id;
    }

    @Override
    public String getId() {
        return id;
    }

    @Override
    public int getSize() {
        return cache.size();
    }

    @Override
    public void putObject(Object key, Object value) {
        cache.put(key, value);
    }

    @Override
    public Object getObject(Object key) {
        return cache.get(key);
    }

    @Override
    public Object removeObject(Object key) {
        return cache.remove(key);
    }

    @Override
    public void clear() {
        cache.clear();
    }

    @Override
    public boolean equals(Object o) {
        if (getId() == null) {
            throw new CacheException("Cache instances require an ID.");
        }
        if (this == o) {
            return true;
        }
        if (!(o instanceof Cache)) {
            return false;
        }

        Cache otherCache = (Cache) o;
        return getId().equals(otherCache.getId());
    }

    @Override
    public int hashCode() {
        if (getId() == null) {
            throw new CacheException("Cache instances require an ID.");
        }
        return getId().hashCode();
    }

}
```

从代码中可以看出，`PerpetualCache`的实现并不复杂，仅仅是把数据存入一个Map中而已。也就是说，Mybatis的缓存是由一个HashMap来实现的，但是我们都知道，HashMap是一个线程不安全的类，那么Mybatis是如何保证线程安装的呢？前面我们说到，`Cache`除`PerpetualCache`这一个真正实现逻辑的实现类，其他的实现类仅作为装饰器类来包装`PerpetualCache`的，所以，`Cache`有一个实现类`SynchronizedCache`,这个类的所有方法都加上了` synchronized  `关键字，以此来保证线程安全。

### 三、CacheKey

前面我们说，Mybatis的缓存是以`HashMap`来实现的，也就是说是以Key-Value的形式保存。那么这里的key是什么呢？不卖管子，Mybatis使用`CacheKey`来表示存入`HashMap`的key值。下面我们来说明一下`CacheKey`是如何产生的。

**CachingExecutor#query**

```java
@Override
public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
    BoundSql boundSql = ms.getBoundSql(parameterObject);
    //生成CacheKey
    CacheKey key = createCacheKey(ms, parameterObject, rowBounds, boundSql);
    return query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
}
```

**BaseExecutor#createCacheKey**

```java
@Override
public CacheKey createCacheKey(MappedStatement ms, Object parameterObject, RowBounds rowBounds, BoundSql boundSql) {
    if (closed) {
        throw new ExecutorException("Executor was closed.");
    }
    CacheKey cacheKey = new CacheKey();
	//statementId
    cacheKey.update(ms.getId());
    //要求的查询结果集的范围
    cacheKey.update(rowBounds.getOffset());
    cacheKey.update(rowBounds.getLimit());
    //传给statement的sql语句
    cacheKey.update(boundSql.getSql());
    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    TypeHandlerRegistry typeHandlerRegistry = ms.getConfiguration().getTypeHandlerRegistry();
    // mimic DefaultParameterHandler logic
    for (ParameterMapping parameterMapping : parameterMappings) {
        if (parameterMapping.getMode() != ParameterMode.OUT) {
            Object value;
            String propertyName = parameterMapping.getProperty();
            if (boundSql.hasAdditionalParameter(propertyName)) {
                value = boundSql.getAdditionalParameter(propertyName);
            } else if (parameterObject == null) {
                value = null;
            } else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
                value = parameterObject;
            } else {
                MetaObject metaObject = configuration.newMetaObject(parameterObject);
                value = metaObject.getValue(propertyName);
            }
            //传给statement的参数集
            cacheKey.update(value);
        }
    }
    if (configuration.getEnvironment() != null) {
        //environment
        cacheKey.update(configuration.getEnvironment().getId());
    }
    return cacheKey;
}
```

这里可以看到，调用了CacheKey#update方法，将statementId、要求的查询结果集的范围(offset,limit)

、sql语句、statement的参数集、environment作为参数传入。这里我们来详细讲解一下CacheKey这个类，你就知道了传入这几个参数的意义了。

**CacheKey#Property**

```java
// 参与计算hashcode，默认值为37
private final int multiplier;
// CacheKey 对象的 hashcode ，默认值 17
private int hashcode;
// 检验和
private long checksum;
// updateList 集合的个数
private int count;
// 由该集合中的所有对象来共同决定两个 CacheKey 是否相等
private List<Object> updateList;
```

**CacheKey#update**

```java
// 调用该方法，向 updateList 集合添加对应的对象
public void update(Object object) {
    int baseHashCode = object == null ? 1 : ArrayUtil.hashCode(object);

    count++;
    checksum += baseHashCode;
    baseHashCode *= count;
	//hashcode = 原来的hashcode * 扩展因子（37） + 新特征值的hashcode
    hashcode = multiplier * hashcode + baseHashCode;

    updateList.add(object);
}
```

update方法就是将依据传入参数来重新计算hashcode值

**CacheKey#equals**

```java
// 判断两个 CacheKey 是否相等
@Override
public boolean equals(Object object) {
    if (this == object) {
        return true;
    }
    if (!(object instanceof CacheKey)) {
        return false;
    }

    final CacheKey cacheKey = (CacheKey) object;

    if (hashcode != cacheKey.hashcode) {
        return false;
    }
    if (checksum != cacheKey.checksum) {
        return false;
    }
    if (count != cacheKey.count) {
        return false;
    }
    // 如果前几项都不满足，则循环遍历 updateList 集合，判断每一项是否相等，如果有一项不相等则这两个CacheKey不相等
    for (int i = 0; i < updateList.size(); i++) {
        Object thisObject = updateList.get(i);
        Object thatObject = cacheKey.updateList.get(i);
        if (thisObject == null) {
            if (thatObject != null) {
                return false;
            }
        } else {
            if (!thisObject.equals(thatObject)) {
                return false;
            }
        }
    }
    return true;
}
```

这个函数重写了equils方法，来比较两个CacheKey的是否相等，若相等，我们就认为缓存中存在一个相同的查询，直接从缓存中得到结果，不用再去数据库查询。

总结一下， Mybatis 的缓存使用了 `mappedStementId + offset + limit + SQL + queryParams + environment` 生成的hashcode作为 key。 

### 四、Executor

前面我们对Executor已经有过分析，他就是一个执行器，负责处理与jdbc的交互，定义了操作数据库的基本方法，SqlSession的相关方法就是在Executor的基础上实现的。同时他也实现了事务的两个方法。

```java
public interface Executor {

    ResultHandler NO_RESULT_HANDLER = null;
	//delete|insert|update
    int update(MappedStatement ms, Object parameter) throws SQLException;
	//查询，带分页，带缓存
    <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey cacheKey, BoundSql boundSql) throws SQLException;
	//查询，带分页，
    <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException;
	//查询存储过程
    <E> Cursor<E> queryCursor(MappedStatement ms, Object parameter, RowBounds rowBounds) throws SQLException;
	//刷新statement
    List<BatchResult> flushStatements() throws SQLException;
    //事务提交
    void commit(boolean required) throws SQLException;
    //事务回滚
    void rollback(boolean required) throws SQLException;
	//创建CacheKey
    CacheKey createCacheKey(MappedStatement ms, Object parameterObject, RowBounds rowBounds, BoundSql boundSql);
	//是否缓存
    boolean isCached(MappedStatement ms, CacheKey key);
	//清空缓存
    void clearLocalCache();
	//延迟加载
    void deferLoad(MappedStatement ms, MetaObject resultObject, String property, CacheKey key, Class<?> targetType);
	//获取transaction
    Transaction getTransaction();
    void close(boolean forceRollback);
    boolean isClosed();
    void setExecutorWrapper(Executor executor);

}
```



