# (八)Mybatis源码阅读—Mapper动态代理

经过前面几节的学习，相信你已经对Mybatis已经有了自己的理解。可能你会提出一个问题，我们平时调用Mapper接口中的方法，而不是调用`SqlSession`中的方法。一个没有实现类的接口是怎么查询数据的啊？

这就要引出这一节的知识点，Mybatis的动态代理。

同样的，我们使用一个小demo来作为入口，来研究Myabtis中是如何使用动态代理的

```java
public static void main(String[] args) throws IOException {
    String resource = "config.xml";
    InputStream inputStream = Resources.getResourceAsStream(resource);
    SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
    SqlSession sqlSession = sqlSessionFactory.openSession();

    PersonMapper mapper = sqlSession.getMapper(PersonMapper.class);
    Person person = mapper.selectOne(2);
    System.out.println(person);
    
    sqlSession.close();
}
```



### 一、绑定映射空间

我们前面在讲Mybatis初始化的时候，会绑定Mapper配置文件的映射空间，也就是Mapper接口

#### 1.XMLMapperBuilder#bindMapperForNamespace 

```java
private void bindMapperForNamespace() {
    //绑定xml对应的mapper接口类，并加入到mapperRegistry属性中。
    String namespace = builderAssistant.getCurrentNamespace();
    if (namespace != null) {
        Class<?> boundType = null;
        try {
            //拿到Mapper的Class对象
            boundType = Resources.classForName(namespace);
        } catch (ClassNotFoundException e) {
            //ignore, bound type is not required
        }
        if (boundType != null) {
            if (!configuration.hasMapper(boundType)) {
                configuration.addLoadedResource("namespace:" + namespace);
                //将接口类对象添加到到mapperRegistry
                configuration.addMapper(boundType);
            }
        }
    }
}
```

这里拿到了Mapper接口的Class对象

#### 2.MapperRegistry#addMapper

```java
public <T> void addMapper(Class<T> type) {
    //类必须是接口类
    if (type.isInterface()) {
        //判断是否已经注册过
        if (hasMapper(type)) {
            throw new BindingException("Type " + type + " is already known to the MapperRegistry.");
        }
        boolean loadCompleted = false;
        try {
            // 放入knownMappers中，并以MapperProxyFactory来进行包装
            knownMappers.put(type, new MapperProxyFactory<>(type));
            // It's important that the type is added before the parser is run
            // otherwise the binding may automatically be attempted by the
            // mapper parser. If the type is already known, it won't try.
            MapperAnnotationBuilder parser = new MapperAnnotationBuilder(config, type);
            parser.parse();
            loadCompleted = true;
        } finally {
            if (!loadCompleted) {
                knownMappers.remove(type);
            }
        }
    }
}
```

这里将mapper接口放入knownMappers中，并以MapperProxyFactory来进行包装

### 二、动态代理

##### 1.MapperRegistry#getMapper

```java
@SuppressWarnings("unchecked")
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    //获取mapper的代理工厂
    final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
    //在map中找不到则表示没有将mapper类注册进来,抛出BindingException
    if (mapperProxyFactory == null) {
        throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
    }
    try {
        //返回一个代理对象
        return mapperProxyFactory.newInstance(sqlSession);
    } catch (Exception e) {
        throw new BindingException("Error getting mapper instance. Cause: " + e, e);
    }
}

public T newInstance(SqlSession sqlSession) {
    final MapperProxy<T> mapperProxy = new MapperProxy<>(sqlSession, mapperInterface, methodCache);
    return newInstance(mapperProxy);
}

@SuppressWarnings("unchecked")
protected T newInstance(MapperProxy<T> mapperProxy) {
    return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
}
```

前面说到，在mybatis解析mapper配置的过程中，会将mapper接口放入knownMappers中，并以MapperProxyFactory来进行包装。因此我们需要从MapperProxyFactory中获取mapper接口,然后通过`newInstance`方法来生成一个代理对象。





### 待续。。。。