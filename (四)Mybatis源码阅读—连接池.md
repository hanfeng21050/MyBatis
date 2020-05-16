# (四)Mybatis源码阅读—连接池

上一节我们说到，`StatementHandler`在会获取数据连接`Connection`，那么Mybatis是怎么获取数据库连接的呢？让给我们继续来往下阅读。

### 一、获取数据库的连接

##### 1.BaseExecutor#getConnection

```java
protected Connection getConnection(Log statementLog) throws SQLException {
    Connection connection = transaction.getConnection();
    if (statementLog.isDebugEnabled()) {
        return ConnectionLogger.newInstance(connection, statementLog, queryStack);
    } else {
        return connection;
    }
}
```

##### 2.这里mybatis提供了两种事务管理方式

-  使用JDBC的事务管理机制,就是利用java.sql.Connection对象完成对事务的提交 

  ```java
  @Override
  public Connection getConnection() throws SQLException {
      if (connection == null) {
          openConnection();
      }
      return connection;
  }
  
  protected void openConnection() throws SQLException {
      if (log.isDebugEnabled()) {
          log.debug("Opening JDBC Connection");
      }
      //通过dataSource.getConnection()获取连接
      connection = dataSource.getConnection();
      if (level != null) {
          connection.setTransactionIsolation(level.getLevel());
      }
      setDesiredAutoCommit(autoCommit);
  }
  ```

-  使用MANAGED的事务管理机制，这种机制通过容器来进行事务管理，所有它对事务提交和回滚并不会做任何操作 

  ```java
  @Override
  public Connection getConnection() throws SQLException {
      if (this.connection == null) {
          openConnection();
      }
      return this.connection;
  }
  
  protected void openConnection() throws SQLException {
      if (log.isDebugEnabled()) {
          log.debug("Opening JDBC Connection");
      }
      this.connection = this.dataSource.getConnection();
      if (this.level != null) {
          this.connection.setTransactionIsolation(this.level.getLevel());
      }
  }
  ```

这里可以很清楚的得到，mybatis是通过**dataSource.getConnection()**来获取数据库连接的

### 二、DataSourceFactory

既然mybtis是通过datasource来获取connection的，那么必然由mybatis配置文件来对dataspurce进行配置。

##### 1.配置文件

```xml
<dataSource type="POOLED">
    <property name="driver" value=""/>
    <property name="url" value=""/>
    <property name="username" value=""/>
    <property name="password" value=""/>
</dataSource>
```

更多配置详见[mybatis中文手册](http://www.dba.cn/book/mybatis/)

这里需要理解的就是连接池的概念， 在用户和数据库之间创建一个”池”，这个池中有若干个连接对象，当用户想要连接数据库，就要先从连接池中获取连接对象，然后操作数据库。一旦连接池中的连接对象被拿光了，下一个想要操作数据库的用户必须等待，等待其他用户释放连接对象，把它放回连接池中，这时候等待的用户才能获取连接对象，从而操作数据库。

在mybatis中，可以通过`<dataSource type="">`来进行配置

- UNPOOLED：mybatis会为每一个数据库的操作创建一个新的连接
- POOLED：Mybatis会创建一个数据库连接池，池中的连接会被用于数据库的操作，一旦数据库的操作完成，连接会被返回到连接池。
- JNDI：Mybatis会从在应用服务器中配置好的JNDI数据源dataSource获取数据库连接。

##### 2.解析dataSource节点

前面初始化中我们讲到，mybatis会利用`XMLConfigBuilder#parse()`来逐一解析节点内容。dataSource节点是environments节点的子节点，因此

**XMLConfigBuilder#environmentsElement**

```java
private void environmentsElement(XNode context) throws Exception {
    if (context != null) {
        if (environment == null) {
            environment = context.getStringAttribute("default");
        }
        for (XNode child : context.getChildren()) {
            String id = child.getStringAttribute("id");
            if (isSpecifiedEnvironment(id)) {
                TransactionFactory txFactory = transactionManagerElement(child.evalNode("transactionManager"));
                //这里来解析dataSource节点，返回的是一个工厂类
                DataSourceFactory dsFactory = dataSourceElement(child.evalNode("dataSource"));
                //得到dataSource
                DataSource dataSource = dsFactory.getDataSource();
                //将dataSouurce加入到Configuration中
                Environment.Builder environmentBuilder = new Environment.Builder(id)
                    .transactionFactory(txFactory)
                    .dataSource(dataSource);
                configuration.setEnvironment(environmentBuilder.build());
            }
        }
    }
}

private DataSourceFactory dataSourceElement(XNode context) throws Exception {
    if (context != null) {
        String type = context.getStringAttribute("type");
        Properties props = context.getChildrenAsProperties();
        DataSourceFactory factory = (DataSourceFactory) resolveClass(type).getDeclaredConstructor().newInstance();
        factory.setProperties(props);
        return factory;
    }
    throw new BuilderException("Environment declaration requires a DataSourceFactory.");
}
```



### 三、连接池

接下来，让我们来看mybatis默认的连接池，PooledDataSource类，需要在配置文件中设置`<dataSource type="POOLED">`

##### 1.申请连接

```java
//申请连接
private PooledConnection popConnection(String username, String password) throws SQLException {
    boolean countedWait = false;
    PooledConnection conn = null;
    long t = System.currentTimeMillis();
    int localBadConnectionCount = 0;

    while (conn == null) {
        //同步代码块，保证线程安全
        synchronized (state) {
            if (!state.idleConnections.isEmpty()) {
                // Pool has available connection
                //i空闲连接不为空，表示有连接可用，直接选取第一个元素
                conn = state.idleConnections.remove(0);
                if (log.isDebugEnabled()) {
                    log.debug("Checked out connection " + conn.getRealHashCode() + " from pool.");
                }
            } else {
                // Pool does not have available connection
                if (state.activeConnections.size() < poolMaximumActiveConnections) {
                    // Can create new connection
                    //active连接未满，可以建立新连接
                    conn = new PooledConnection(dataSource.getConnection(), this);
                    if (log.isDebugEnabled()) {
                        log.debug("Created connection " + conn.getRealHashCode() + ".");
                    }
                } else {
                    // Cannot create new connection
                    // 如果活跃的连接数已经达到允许的最大值了，则不能创建新的数据库连接
                    // 获取最先创建的那个活跃的连接
                    PooledConnection oldestActiveConnection = state.activeConnections.get(0);
                    long longestCheckoutTime = oldestActiveConnection.getCheckoutTime();
                    if (longestCheckoutTime > poolMaximumCheckoutTime) {
                        // Can claim overdue connection
                        //超时，将原连接废弃，根据条件创建新连接
                        // 连接超时的连接个数++
                        state.claimedOverdueConnectionCount++;
                        // 累计超时时间
                        state.accumulatedCheckoutTimeOfOverdueConnections += longestCheckoutTime;
                        // checkoutTime 表示从连接池中获取连接到归还连接的时间
                        // accumulatedCheckoutTime 记录了所有连接的累计 checkoutTime 时长
                        state.accumulatedCheckoutTime += longestCheckoutTime;
                        // 将超时连接移出 activeConnections 集合
                        state.activeConnections.remove(oldestActiveConnection);
                        // 如果超时未提交，则自动回滚
                        if (!oldestActiveConnection.getRealConnection().getAutoCommit()) {
                            try {
                                oldestActiveConnection.getRealConnection().rollback();
                            } catch (SQLException e) {
                                /*
                     Just log a message for debug and continue to execute the following
                     statement like nothing happened.
                     Wrap the bad connection with a new PooledConnection, this will help
                     to not interrupt current executing thread and give current thread a
                     chance to join the next competition for another valid/good database
                     connection. At the end of this loop, bad {@link @conn} will be set as null.
                   */
                                log.debug("Bad connection. Could not roll back");
                            }
                        }
                        // 创建新的 PooledConnection 对象，但是真正的数据库连接并没有创建
                        conn = new PooledConnection(oldestActiveConnection.getRealConnection(), this);
                        conn.setCreatedTimestamp(oldestActiveConnection.getCreatedTimestamp());
                        conn.setLastUsedTimestamp(oldestActiveConnection.getLastUsedTimestamp());
                        oldestActiveConnection.invalidate();
                        if (log.isDebugEnabled()) {
                            log.debug("Claimed overdue connection " + conn.getRealHashCode() + ".");
                        }
                    } else {
                        // Must wait
                        // 如果无空闲连接，无法创建新的连接且无超时连接，则只能阻塞等待
                        try {
                            if (!countedWait) {
                                // 等待次数
                                state.hadToWaitCount++;
                                countedWait = true;
                            }
                            if (log.isDebugEnabled()) {
                                log.debug("Waiting as long as " + poolTimeToWait + " milliseconds for connection.");
                            }
                            long wt = System.currentTimeMillis();
                            // 阻塞等待
                            state.wait(poolTimeToWait);
                            // 累计等待时间
                            state.accumulatedWaitTime += System.currentTimeMillis() - wt;
                        } catch (InterruptedException e) {
                            break;
                        }
                    }
                }
            }
            // 已经获取到连接
            if (conn != null) {
                // ping to server and check the connection is valid or not
                if (conn.isValid()) {
                    if (!conn.getRealConnection().getAutoCommit()) {
                        // 如果连接有效，事务未提交则回滚
                        conn.getRealConnection().rollback();
                    }
                    // 设置 PooledConnection 相关属性
                    conn.setConnectionTypeCode(assembleConnectionTypeCode(dataSource.getUrl(), username, password));
                    conn.setCheckoutTimestamp(System.currentTimeMillis());
                    conn.setLastUsedTimestamp(System.currentTimeMillis());
                    // 把连接加入到活跃集合中去
                    state.activeConnections.add(conn);
                    state.requestCount++;
                    state.accumulatedRequestTime += System.currentTimeMillis() - t;
                } else {
                    // 无效连接
                    if (log.isDebugEnabled()) {
                        log.debug("A bad connection (" + conn.getRealHashCode() + ") was returned from the pool, getting another connection.");
                    }
                    state.badConnectionCount++;
                    localBadConnectionCount++;
                    conn = null;
                    //无效线程数过多，抛异常
                    if (localBadConnectionCount > (poolMaximumIdleConnections + poolMaximumLocalBadConnectionTolerance)) {
                        if (log.isDebugEnabled()) {
                            log.debug("PooledDataSource: Could not get a good connection to the database.");
                        }
                        throw new SQLException("PooledDataSource: Could not get a good connection to the database.");
                    }
                }
            }
        }

    }

    if (conn == null) {
        if (log.isDebugEnabled()) {
            log.debug("PooledDataSource: Unknown severe error condition.  The connection pool returned a null connection.");
        }
        throw new SQLException("PooledDataSource: Unknown severe error condition.  The connection pool returned a null connection.");
    }

    return conn;
}
```

##### 2.返还连接

```java
//归还连接
protected void pushConnection(PooledConnection conn) throws SQLException {

    synchronized (state) {
        // 首先从活跃的集合中移除掉该连接
        state.activeConnections.remove(conn);
        if (conn.isValid()) {
            // 如果空闲连接数没有达到最大值，且 PooledConnection 为该连接池的连接
            if (state.idleConnections.size() < poolMaximumIdleConnections && conn.getConnectionTypeCode() == expectedConnectionTypeCode) {
                // 累计 checkout 时长
                state.accumulatedCheckoutTime += conn.getCheckoutTime();
                // 事务回滚
                if (!conn.getRealConnection().getAutoCommit()) {
                    conn.getRealConnection().rollback();
                }
                // 为返还的连接创建新的 PooledConnection 对象
                PooledConnection newConn = new PooledConnection(conn.getRealConnection(), this);
                // 把该连接添加的空闲链表中
                state.idleConnections.add(newConn);
                newConn.setCreatedTimestamp(conn.getCreatedTimestamp());
                newConn.setLastUsedTimestamp(conn.getLastUsedTimestamp());
                // 设置该连接为无效状态
                conn.invalidate();
                if (log.isDebugEnabled()) {
                    log.debug("Returned connection " + newConn.getRealHashCode() + " to pool.");
                }
                // 唤醒阻塞等待的线程
                state.notifyAll();
            } else {
                state.accumulatedCheckoutTime += conn.getCheckoutTime();
                // 如果空闲连接数已经达到最大值
                if (!conn.getRealConnection().getAutoCommit()) {
                    conn.getRealConnection().rollback();
                }
                // 则关闭真正的数据库连接
                conn.getRealConnection().close();
                if (log.isDebugEnabled()) {
                    log.debug("Closed connection " + conn.getRealHashCode() + ".");
                }
                // 设置该连接为无效状态
                conn.invalidate();
            }
        } else {
            if (log.isDebugEnabled()) {
                log.debug("A bad connection (" + conn.getRealHashCode() + ") attempted to return to the pool, discarding connection.");
            }
            // 无效连接个数加1
            state.badConnectionCount++;
        }
    }
}
```



##### 3.流程图

![从连接池获取连接](img\从连接池获取连接.png)