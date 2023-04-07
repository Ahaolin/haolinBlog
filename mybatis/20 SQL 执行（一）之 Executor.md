# 精尽 MyBatis 源码分析 —— SQL 执行（一）之 Executor

## 1. 概述

从本文开始，我们来分享 SQL **执行**的流程。在 [《精尽 MyBatis 源码分析 —— 项目结构一览》](http://svip.iocoder.cn/MyBatis/intro) 中，我们简单介绍这个流程如下：

> 对应 `executor` 和 `cursor` 模块。前者对应**执行器**，后者对应执行**结果的游标**。
>
> SQL 语句的执行涉及多个组件 ，其中比较重要的是 Executor、StatementHandler、ParameterHandler 和 ResultSetHandler 。
>
> - **Executor** 主要负责维护一级缓存和二级缓存，并提供事务管理的相关操作，它会将数据库相关操作委托给 StatementHandler完成。
> - **StatementHandler** 首先通过 **ParameterHandler** 完成 SQL 语句的实参绑定，然后通过 `java.sql.Statement` 对象执行 SQL 语句并得到结果集，最后通过 **ResultSetHandler** 完成结果集的映射，得到结果对象并返回。
>
> 整体过程如下图：
>
> [![整体过程](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202201261421050.png)](http://static.iocoder.cn/images/MyBatis/2020_01_04/05.png)整体过程

下面，我们在看看 `executor` 包下的列情况，如下图所示：[![`executor` 包](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202201261421023.png)](http://static.iocoder.cn/images/MyBatis/2020_02_28/01.png)`executor` 包

- 正如该包下的分包情况，每个包对应一个功能。
- `statement` 包，实现向数据库发起 SQL 命令。
- `parameter`包，实现设置 PreparedStatement 的占位符参数。
  - 目前只有一个 ParameterHandler 接口，在 [《精尽 MyBatis 源码分析 —— SQL 初始化（下）之 SqlSource》](http://svip.iocoder.cn/MyBatis/scripting-2) 已经详细解析。
- `keygen` 包，实现数据库主键生成( 获得 )的功能。
- `resultset` 包，实现 ResultSet 结果集的处理，将其映射成对应的结果对象。
- `result` 包，结果的处理，被 `resultset` 包所调用。可能胖友会好奇为啥会有 `resultset` 和 `result` 两个“重叠”的包。答案见 [《精尽 MyBatis 源码分析 —— SQL 执行（四）之 ResultSetHandler》](http://svip.iocoder.cn/MyBatis/executor-4) 。
- `loader` 包，实现延迟加载的功能。
- 根目录，Executor 接口及其实现类，作为 SQL 执行的核心入口。

考虑到整个 `executor` 包的代码量近 5000 行，所以我们将每一个子包，作为一篇文章，逐包解析。所以，本文我们先来分享 根目录，也就是 Executor 接口及其实现类。

## 2. Executor

`org.apache.ibatis.executor.Executor` ，执行器接口。代码如下：

```java
// Executor.java

public interface Executor {

    // 空 ResultHandler 对象的枚举
    ResultHandler NO_RESULT_HANDLER = null;

    // 更新 or 插入 or 删除，由传入的 MappedStatement 的 SQL 所决定
    int update(MappedStatement ms, Object parameter) throws SQLException;

    // 查询，带 ResultHandler + CacheKey + BoundSql
    <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey cacheKey, BoundSql boundSql) throws SQLException;
    // 查询，带 ResultHandler
    <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException;
    // 查询，返回值为 Cursor
    <E> Cursor<E> queryCursor(MappedStatement ms, Object parameter, RowBounds rowBounds) throws SQLException;

    // 刷入批处理语句
    List<BatchResult> flushStatements() throws SQLException;

    // 提交事务
    void commit(boolean required) throws SQLException;
    // 回滚事务
    void rollback(boolean required) throws SQLException;

    // 创建 CacheKey 对象
    CacheKey createCacheKey(MappedStatement ms, Object parameterObject, RowBounds rowBounds, BoundSql boundSql);
    // 判断是否缓存
    boolean isCached(MappedStatement ms, CacheKey key);
    // 清除本地缓存
    void clearLocalCache();

    // 延迟加载
    void deferLoad(MappedStatement ms, MetaObject resultObject, String property, CacheKey key, Class<?> targetType);

    // 获得事务
    Transaction getTransaction();
    // 关闭事务
    void close(boolean forceRollback);
    // 判断事务是否关闭
    boolean isClosed();

    // 设置包装的 Executor 对象
    void setExecutorWrapper(Executor executor);

}
```

- 读和写操作相关的方法
- 事务相关的方法
- 缓存相关的方法
- 设置延迟加载的方法
- 设置包装的 Executor 对象的方法

------

Executor 的实现类如下图所示：[![Executor 类图](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202201261421025.png)](http://static.iocoder.cn/images/MyBatis/2020_02_28/02.png)Executor 类图

- 我们可以看到，Executor 的直接子类有 BaseExecutor 和 CachingExecutor 两个。
- 实际上，CachingExecutor 在 BaseExecutor 的基础上，实现**二级缓存**功能。
- 在下文中，BaseExecutor 的**本地**缓存，就是**一级**缓存。
- `flushStatements()``flushStatements()`调用时机时序图
  - <img src="http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202202021851151.png" style="zoom:67%;" />


下面，我们按照先看 BaseExecutor 侧的实现类的源码解析，再看 CachingExecutor 的。

## 3. BaseExecutor

`org.apache.ibatis.executor.BaseExecutor` ，实现 Executor 接口，提供骨架方法，从而使子类只要实现指定的几个抽象方法即可。

### 3.1 构造方法

```java
// BaseExecutor.java

/**
 * 事务对象
 */
protected Transaction transaction;
/**
 * 包装的 Executor 对象
 */
protected Executor wrapper;

/**
 * DeferredLoad( 延迟加载 ) 队列
 */
protected ConcurrentLinkedQueue<DeferredLoad> deferredLoads;
/**
 * 本地缓存，即一级缓存
 */
protected PerpetualCache localCache;
/**
 * 本地输出类型的参数的缓存
 */
protected PerpetualCache localOutputParameterCache;
protected Configuration configuration;

/**
 * 记录嵌套查询的层级
 */
protected int queryStack;
/**
 * 是否关闭
 */
private boolean closed;

protected BaseExecutor(Configuration configuration, Transaction transaction) {
    this.transaction = transaction;
    this.deferredLoads = new ConcurrentLinkedQueue<>();
    this.localCache = new PerpetualCache("LocalCache");
    this.localOutputParameterCache = new PerpetualCache("LocalOutputParameterCache");
    this.closed = false;
    this.configuration = configuration;
    this.wrapper = this; // 自己
}
```

- 和延迟加载相关，后续文章，详细解析。详细解析，见 [《精尽 MyBatis 源码分析 —— SQL 执行（五）之延迟加载》](http://svip.iocoder.cn/MyBatis/executor-5) 。

  - `queryStack` 属性，记录**递归**嵌套查询的层级。
  - `deferredLoads` 属性，DeferredLoad( 延迟加载 ) 队列。

- `wrapper` 属性，在构造方法中，初始化为 `this` ，即自己。

- `localCache` 属性，本地缓存，即**一级缓存**。那什么是一级缓存呢？

  > 基于 [《MyBatis 的一级缓存实现详解及使用注意事项》](https://blog.csdn.net/luanlouis/article/details/41280959) 进行修改
  >
  > 每当我们使用 MyBatis 开启一次和数据库的会话，MyBatis 会创建出一个 SqlSession 对象表示一次数据库会话，**而每个 SqlSession 都会创建一个 Executor 对象**。
  >
  > 在对数据库的一次会话中，我们有可能会反复地执行完全相同的查询语句，如果不采取一些措施的话，每一次查询都会查询一次数据库，而我们在极短的时间内做了完全相同的查询，那么它们的结果极有可能完全相同，由于查询一次数据库的代价很大，这有可能造成很大的资源浪费。
  >
  > 为了解决这一问题，减少资源的浪费，MyBatis 会在表示会话的SqlSession 对象中建立一个简单的缓存，将每次查询到的结果结果缓存起来，当下次查询的时候，如果判断先前有个完全一样的查询，会直接从缓存中直接将结果取出，返回给用户，不需要再进行一次数据库查询了。😈 **注意，这个“简单的缓存”就是一级缓存，且默认开启，无法关闭**。
  >
  > 如下图所示，MyBatis 会在一次会话的表示 —— 一个 SqlSession 对象中创建一个本地缓存( `localCache` )，对于每一次查询，都会尝试根据查询的条件去本地缓存中查找是否在缓存中，如果在缓存中，就直接从缓存中取出，然后返回给用户；否则，从数据库读取数据，将查询结果存入缓存并返回给用户。
  >
  > [![整体过程](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202201261421016.png)](http://static.iocoder.cn/images/MyBatis/2020_02_28/03.png)整体过程
  >
  > - 关于这段话，胖友要理解 SqlSession 和 Executor 和一级缓存的关系。
  > - 😈 另外，下文，我们还会介绍**二级缓存**是什么。

- `transaction` 属性，事务对象。该属性，是通过构造方法传入。为什么呢？待我们看 `org.apache.ibatis.session.session` 包。

### 3.2 clearLocalCache

`#clearLocalCache()` 方法，清理一级（本地）缓存。代码如下：

```java
// BaseExecutor.java

@Override
public void clearLocalCache() {
    if (!closed) {
        // 清理 localCache
        localCache.clear();
        // 清理 localOutputParameterCache
        localOutputParameterCache.clear();
    }
}
```

### 3.3 createCacheKey

`#createCacheKey(MappedStatement ms, Object parameterObject, RowBounds rowBounds, BoundSql boundSql)` 方法，创建 CacheKey 对象。代码如下：[-> #query](#go3.5_2)

```java
// BaseExecutor.java

@Override
public CacheKey createCacheKey(MappedStatement ms, Object parameterObject, RowBounds rowBounds, BoundSql boundSql) {
    if (closed) {
        throw new ExecutorException("Executor was closed.");
    }
    // <1> 创建 CacheKey 对象
    CacheKey cacheKey = new CacheKey();
    // <2> 设置 id、offset、limit、sql 到 CacheKey 对象中
    cacheKey.update(ms.getId());
    cacheKey.update(rowBounds.getOffset());
    cacheKey.update(rowBounds.getLimit());
    cacheKey.update(boundSql.getSql());
    // <3> 设置 ParameterMapping 数组的元素对应的每个 value 到 CacheKey 对象中
    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    TypeHandlerRegistry typeHandlerRegistry = ms.getConfiguration().getTypeHandlerRegistry();
    // mimic DefaultParameterHandler logic 这块逻辑，和 DefaultParameterHandler 获取 value 是一致的。
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
            cacheKey.update(value);
        }
    }
    // <4> 设置 Environment.id 到 CacheKey 对象中
    if (configuration.getEnvironment() != null) {
        // issue #176
        cacheKey.update(configuration.getEnvironment().getId());
    }
    return cacheKey;
}
```

- `<1>` 处，创建 CacheKey 对象。关于 CacheKey 类，在 [《精尽 MyBatis 源码分析 —— 缓存模块》](http://svip.iocoder.cn/MyBatis/cache-package) 已经详细解析。
- `<2>` 处，设置 `id`、`offset`、`limit`、`sql` 到 CacheKey 对象中。
- `<3>` 处，设置 ParameterMapping 数组的元素对应的每个 `value` 到 CacheKey 对象中。注意，这块逻辑，和 DefaultParameterHandler 获取 `value` 是一致的。
- `<4>` 处，设置 `Environment.id` 到 CacheKey 对象中。

### 3.4 isCached

`#isCached(MappedStatement ms, CacheKey key)` 方法，判断一级缓存是否存在。代码如下：

```java
// BaseExecutor.java

@Override
public boolean isCached(MappedStatement ms, CacheKey key) {
    return localCache.getObject(key) != null;
}
```

### 3.5 query

① `#query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler)` 方法，读操作。代码如下：

```java
// BaseExecutor.java

@Override
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
    // <1> 获得 BoundSql 对象
    BoundSql boundSql = ms.getBoundSql(parameter);
    // <2> 创建 CacheKey 对象
    CacheKey key = createCacheKey(ms, parameter, rowBounds, boundSql);
    // <3> 查询
    return query(ms, parameter, rowBounds, resultHandler, key, boundSql);
}
```

- `<1>` 处，调用 `MappedStatement#getBoundSql(Object parameterObject)` 方法，获得 BoundSql 对象。
- <span id='go3.5_2'>`<2>` </span>处，调用 `#createCacheKey(MappedStatement ms, Object parameterObject, RowBounds rowBounds, BoundSql boundSql)` 方法，[创建 CacheKey 对象](#3.3 createCacheKey)。
- `<3>` 处，调用 `#query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)` 方法，读操作。通过这样的方式，两个 `#query(...)` 方法，实际是统一的。

② `#query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)` 方法，读操作。代码如下：

```java
// BaseExecutor.java

@Override
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
    // <1> 已经关闭，则抛出 ExecutorException 异常
    if (closed) {
        throw new ExecutorException("Executor was closed.");
    }
    // <2> 清空本地缓存，如果 queryStack 为零，并且要求清空本地缓存。
    if (queryStack == 0 && ms.isFlushCacheRequired()) {
        clearLocalCache();
    }
    List<E> list;
    try {
        // <3> queryStack + 1
        queryStack++;
        // <4.1> 从一级缓存中，获取查询结果
        list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
        // <4.2> 获取到，则进行处理
        if (list != null) {
            handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
        // <4.3> 获得不到，则从数据库中查询
        } else {
            list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
        }
    } finally {
        // <5> queryStack - 1
        queryStack--;
    }
    if (queryStack == 0) {
        // <6.1> 执行延迟加载
        for (DeferredLoad deferredLoad : deferredLoads) {
            deferredLoad.load();
        }
        // issue #601
        // <6.2> 清空 deferredLoads
        deferredLoads.clear();
        // <7> 如果缓存级别是 LocalCacheScope.STATEMENT ，则进行清理
        if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
            // issue #482
            clearLocalCache();
        }
    }
    return list;
}
```

- `<1>` 处，已经关闭，则抛出 ExecutorException 异常。

- `<2>` 处，调用 `#clearLocalCache()` 方法，清空本地缓存，如果 `queryStack` 为零，并且要求清空本地缓存。例如：`<select flushCache="true"> ... </a>` 。

- `<3>` 处，`queryStack` + 1 。

- `<4.1>`处，从一级缓存`localCache`中，获取查询结果。

- `<4.2>` 处，获取到，则进行处理。对于 `#handleLocallyCachedOutputParameters(MappedStatement ms, CacheKey key, Object parameter, BoundSql boundSql)` 方法，是处理存储过程的情况，所以我们就忽略。

- <span id='go_3.5_4.3'>`<4.3>`</span> 处，获得不到，则调用 `#queryFromDatabase()` 方法，从数据库中查询。详细解析，见 [「3.5.1 queryFromDatabase」](#3.5.1 queryFromDatabase) 。

- `<5>` 处，`queryStack` - 1 。

- `<6.1>`处，遍历 DeferredLoad 队列，逐个调用 `DeferredLoad#load()`方法，执行延迟加载。详细解析，见[《精尽 MyBatis 源码分析 —— SQL 执行（五）之延迟加载》](http://svip.iocoder.cn/MyBatis/executor-5) 。

- `<6.2>` 处，清空 DeferredLoad 队列。

- `<7>` 处，如果缓存级别是 `LocalCacheScope.STATEMENT` ，则调用 `#clearLocalCache()` 方法，清空本地缓存。默认情况下，缓存级别是 `LocalCacheScope.SESSION` 。代码如下：

  ```java
  // Configuration.java
  
  /**
   * {@link BaseExecutor} 本地缓存范围
   */
  protected LocalCacheScope localCacheScope = LocalCacheScope.SESSION;
  
  // LocalCacheScope.java
  
  public enum LocalCacheScope {
  
      /**
       * 会话级
       */
      SESSION,
      /**
       * SQL 语句级
       */
      STATEMENT
  
  }
  ```

#### 3.5.1 queryFromDatabase

`#queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)` 方法，从数据库中读取操作。代码如下：[<-#query](#go_3.5_4.3)

```java
// BaseExecutor.java

private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    List<E> list;
    // <1> 在缓存中，添加占位对象。此处的占位符，和延迟加载有关，可见 `DeferredLoad#canLoad()` 方法
    localCache.putObject(key, EXECUTION_PLACEHOLDER);
    try {
        // <2> 执行读操作
        list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
    } finally {
        // <3> 从缓存中，移除占位对象
        localCache.removeObject(key);
    }
    // <4> 添加到缓存中
    localCache.putObject(key, list);
    // <5> 暂时忽略，存储过程相关
    if (ms.getStatementType() == StatementType.CALLABLE) {
        localOutputParameterCache.putObject(key, parameter);
    }
    return list;
}
```

- `<1>` 处，在缓存中，添加**占位对象**。此处的占位符，和延迟加载有关，后续可见 `DeferredLoad#canLoad()` 方法。

  ```java
  // BaseExecutor.java
  
  public enum ExecutionPlaceholder {
  
      /**
       * 正在执行中的占位符
       */
      EXECUTION_PLACEHOLDER
  
  }
  ```

- <span id='go3.5.2_2'>`<2>`</span> 处，调用 `#doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql)` [执行读操作](#3.5.2 doQuery)。这是个抽象方法，由子类实现。

- `<3>` 处，从缓存中，移除占位对象。

- `<4>` 处，添加**结果**到缓存中。

- `<5>` 处，暂时忽略，存储过程相关。

#### 3.5.2 doQuery

```java
// BaseExecutor.java

protected abstract <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql)
        throws SQLException;
```

### 3.6 queryCursor

`#queryCursor(MappedStatement ms, Object parameter, RowBounds rowBounds)` 方法，执行查询，返回的结果为 Cursor 游标对象。代码如下：

```java
// BaseExecutor.java

@Override
public <E> Cursor<E> queryCursor(MappedStatement ms, Object parameter, RowBounds rowBounds) throws SQLException {
    // <1> 获得 BoundSql 对象
    BoundSql boundSql = ms.getBoundSql(parameter);
    // 执行查询
    return doQueryCursor(ms, parameter, rowBounds, boundSql);
}
```

- `<1>` 处，调用 `MappedStatement#getBoundSql(Object parameterObject)` 方法，获得 BoundSql 对象。
- `<2>` 处，调用 `#doQueryCursor(MappedStatement ms, Object parameter, RowBounds rowBounds, BoundSql boundSql)` 方法，执行读操作。这是个抽象方法，由子类实现。

#### 3.6.1 doQueryCursor

```java
// BaseExecutor.java

protected abstract <E> Cursor<E> doQueryCursor(MappedStatement ms, Object parameter, RowBounds rowBounds, BoundSql boundSql)
        throws SQLException;
```

### 3.7 update

`#update(MappedStatement ms, Object parameter)` 方法，执行写操作。代码如下：

```java
// BaseExecutor.java

@Override
public int update(MappedStatement ms, Object parameter) throws SQLException {
    ErrorContext.instance().resource(ms.getResource()).activity("executing an update").object(ms.getId());
    // <1> 已经关闭，则抛出 ExecutorException 异常
    if (closed) {
        throw new ExecutorException("Executor was closed.");
    }
    // <2> 清空本地缓存
    clearLocalCache();
    // <3> 执行写操作
    return doUpdate(ms, parameter);
}
```

- `<1>` 处，已经关闭，则抛出 ExecutorException 异常。
- `<2>` 处，调用 `#clearLocalCache()` 方法，清空本地缓存。因为，更新后，可能缓存会失效。但是，又没很好的办法，判断哪一些失效。所以，最稳妥的做法，就是全部清空。
- `<3>` 处，调用 `#doUpdate(MappedStatement ms, Object parameter)` 方法，执行写操作。这是个抽象方法，由子类实现。

#### 3.7.1 doUpdate

```java
// BaseExecutor.java

protected abstract int doUpdate(MappedStatement ms, Object parameter)
        throws SQLException;
```

### 3.8 flushStatements

`#flushStatements()` 方法，刷入批处理语句。代码如下：

```java
// BaseExecutor.java

@Override
public List<BatchResult> flushStatements() throws SQLException {
    return flushStatements(false);
}

public List<BatchResult> flushStatements(boolean isRollBack) throws SQLException {
    // <1> 已经关闭，则抛出 ExecutorException 异常
    if (closed) {
        throw new ExecutorException("Executor was closed.");
    }
    // <2> 执行刷入批处理语句
    return doFlushStatements(isRollBack);
}
```

- `isRollBack` 属性，目前看下来没什么逻辑。唯一看到在 BatchExecutor 中，如果 `isRollBack = true` ，则**不执行**刷入批处理语句。有点奇怪。
- `<1>` 处，已经关闭，则抛出 ExecutorException 异常。
- `<2>` 处，调用 `#doFlushStatements(boolean isRollback)` 方法，执行刷入批处理语句。这是个抽象方法，由子类实现。

#### 3.8.1 doFlushStatements

```java
// BaseExecutor.java

protected abstract List<BatchResult> doFlushStatements(boolean isRollback)
        throws SQLException;
```

------

至此，我们已经看到了 BaseExecutor 所定义的**四个抽象方法**：

- 「3.5.2 doQuery」
- 「3.6.1 doQueryCursor」
- 「3.7.1 doUpdate」
- 「3.8.1 doFlushStatements」

### 3.9 getTransaction

`#getTransaction()` 方法，获得事务对象。代码如下：

```java
// BaseExecutor.java

@Override
public Transaction getTransaction() {
    if (closed) {
        throw new ExecutorException("Executor was closed.");
    }
    return transaction;
}
```

#### 3.9.1 commit

```java
// BaseExecutor.java

@Override
public void commit(boolean required) throws SQLException {
    // 已经关闭，则抛出 ExecutorException 异常
    if (closed) {
        throw new ExecutorException("Cannot commit, transaction is already closed");
    }
    // 清空本地缓存
    clearLocalCache();
    // 刷入批处理语句
    flushStatements();
    // 是否要求提交事务。如果是，则提交事务。
    if (required) {
        transaction.commit();
    }
}
```

#### 3.9.2 rollback

```java
// BaseExecutor.java

public void rollback(boolean required) throws SQLException {
    if (!closed) {
        try {
            // 清空本地缓存
            clearLocalCache();
            // 刷入批处理语句
            flushStatements(true);
        } finally {
            if (required) {
                // 是否要求回滚事务。如果是，则回滚事务。
                transaction.rollback();
            }
        }
    }
}
```

### 3.10 close

`#close()` 方法，关闭执行器。代码如下：

```java
// BaseExecutor.java

@Override
public void close(boolean forceRollback) {
    try {
        // 回滚事务
        try {
            rollback(forceRollback);
        } finally {
            // 关闭事务
            if (transaction != null) {
                transaction.close();
            }
        }
    } catch (SQLException e) {
        // Ignore.  There's nothing that can be done at this point.
        log.warn("Unexpected exception on closing transaction.  Cause: " + e);
    } finally {
        // 置空变量
        transaction = null;
        deferredLoads = null;
        localCache = null;
        localOutputParameterCache = null;
        closed = true;
    }
}
```

#### 3.10.1 isClosed

```java
// BaseExecutor.java

@Override
public boolean isClosed() {
    return closed;
}
```

### 3.11 deferLoad

详细解析，见 [《精尽 MyBatis 源码分析 —— SQL 执行（五）之延迟加载》](http://svip.iocoder.cn/MyBatis/executor-5) 。

### 3.12 setExecutorWrapper

`#setExecutorWrapper(Executor wrapper)` 方法，设置包装器。代码如下：

```java
// BaseExecutor.java

@Override
public void setExecutorWrapper(Executor wrapper) {
    this.wrapper = wrapper;
}
```

### 3.13 其它方法

```java
// BaseExecutor.java

// 获得 Connection 对象
protected Connection getConnection(Log statementLog) throws SQLException {
    // 获得 Connection 对象
    Connection connection = transaction.getConnection();
    // 如果 debug 日志级别，则创建 ConnectionLogger 对象，进行动态代理
    if (statementLog.isDebugEnabled()) {
        return ConnectionLogger.newInstance(connection, statementLog, queryStack);
    } else {
        return connection;
    }
}

// 设置事务超时时间
protected void applyTransactionTimeout(Statement statement) throws SQLException {
    StatementUtil.applyTransactionTimeout(statement, statement.getQueryTimeout(), transaction.getTimeout());
}

// 关闭 Statement 对象
protected void closeStatement(Statement statement) {
    if (statement != null) {
        try {
            if (!statement.isClosed()) {
                statement.close();
            }
        } catch (SQLException e) {
            // ignore
        }
    }
}
```

- 关于 StatementUtil 类，可以点击 [`org.apache.ibatis.executor.statement.StatementUtil`](https://github.com/YunaiV/mybatis-3/blob/master/src/main/java/org/apache/ibatis/executor/statement/StatementUtil.java) 查看。

#### 3.13.1 StatementUtil

```java
/**
 * Utility for {@link java.sql.Statement}.
 *
 * {@link Statement} 工具类
 *
 * @since 3.4.0
 * @author Kazuki Shimizu
 */
public class StatementUtil {

    private StatementUtil() {
        // NOP
    }

    /**
     * 设置事务超时时间
     *
     * Apply a transaction timeout.
     * <p>
     * Update a query timeout to apply a transaction timeout.
     * </p>
     * @param statement a target statement
     * @param queryTimeout a query timeout
     * @param transactionTimeout a transaction timeout
     * @throws SQLException if a database access error occurs, this method is called on a closed <code>Statement</code>
     */
    public static void applyTransactionTimeout(Statement statement, Integer queryTimeout, Integer transactionTimeout) throws SQLException {
        if (transactionTimeout == null) {
            return;
        }
        // 获得 timeToLiveOfQuery
        Integer timeToLiveOfQuery = null;
        if (queryTimeout == null || queryTimeout == 0) { // 取 transactionTimeout
            timeToLiveOfQuery = transactionTimeout;
        } else if (transactionTimeout < queryTimeout) { // 取小的
            timeToLiveOfQuery = transactionTimeout;
        }
        // 设置超时时间
        if (timeToLiveOfQuery != null) {
            statement.setQueryTimeout(timeToLiveOfQuery);
        }
    }

}
```

## 4. SimpleExecutor

`org.apache.ibatis.executor.SimpleExecutor` ，继承 BaseExecutor 抽象类，简单的 Executor 实现类。

- 每次开始读或写操作，都创建对应的 Statement 对象。
- 执行完成后，关闭该 Statement 对象。

### 4.1 构造方法

```java
// SimpleExecutor.java

public SimpleExecutor(Configuration configuration, Transaction transaction) {
    super(configuration, transaction);
}
```

### 4.2 doQuery

> 老艿艿：从此处开始，我们会看到 StatementHandler 相关的调用，胖友可以结合 [《精尽 MyBatis 源码分析 —— SQL 执行（二）之 StatementHandler》](http://svip.iocoder.cn/MyBatis/executor-2) 一起看。

```java
// SimpleExecutor.java

@Override
public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
    Statement stmt = null;
    try {
        Configuration configuration = ms.getConfiguration();
        // <1> 创建 StatementHandler 对象
        StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
        // <2> 初始化 StatementHandler 对象
        stmt = prepareStatement(handler, ms.getStatementLog());
        // <3> 执行 StatementHandler  ，进行读操作
        return handler.query(stmt, resultHandler);
    } finally {
        // <4> 关闭 StatementHandler 对象
        closeStatement(stmt);
    }
}
```

- <span id='go4.2_1'>`<1>`</span> 处，调用 `Configuration#newStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql)` 方法，创建 StatementHandler 对象。

- <span id='go4.2_2'>`<2>`</span>  处，调用 `#prepareStatement(StatementHandler handler, Log statementLog)` 方法，初始化 StatementHandler 对象。代码如下：

  ```java
  // SimpleExecutor.java
  
  private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
      Statement stmt;
      // <2.1> 获得 Connection 对象
      Connection connection = getConnection(statementLog);
      // <2.2> 创建 Statement 或 PrepareStatement 对象
      stmt = handler.prepare(connection, transaction.getTimeout());
      // <2.3> 设置 SQL 上的参数，例如 PrepareStatement 对象上的占位符
      handler.parameterize(stmt);
      return stmt;
  }
  ```

  - `<2.1>` 处，调用 `#getConnection(Log statementLog)` 方法，获得 Connection 对象。
  - `<2.2>` 处，调用 `StatementHandler#prepare(Connection connection, Integer transactionTimeout)` 方法，创建 Statement 或 PrepareStatement 对象。
  - `<2.3>` 处，调用 `StatementHandler#prepare(Statement statement)` 方法，设置 SQL 上的参数，例如 PrepareStatement 对象上的占位符。

- <span id='go4.2_3'>`<3>`</span>  处，调用 `StatementHandler#query(Statement statement, ResultHandler resultHandler)` 方法，进行**读操作**。

- <span id='go4.2_4'>`<4>`</span>  处，调用 `#closeStatement(Statement stmt)` 方法，关闭 StatementHandler 对象。

### 4.3 doQueryCursor

```java
// SimpleExecutor.java

@Override
protected <E> Cursor<E> doQueryCursor(MappedStatement ms, Object parameter, RowBounds rowBounds, BoundSql boundSql) throws SQLException {
    Configuration configuration = ms.getConfiguration();
    // 创建 StatementHandler 对象
    StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, null, boundSql);
    // 初始化 StatementHandler 对象
    Statement stmt = prepareStatement(handler, ms.getStatementLog());
    // 设置 Statement ，如果执行完成，则进行自动关闭
    stmt.closeOnCompletion();
    // 执行 StatementHandler  ，进行读操作
    return handler.queryCursor(stmt);
}
```

- 和 `#doQuery(...)` 方法的思路是一致的，胖友自己看下。

### 4.4 doUpdate

```java
// SimpleExecutor.java

@Override
public int doUpdate(MappedStatement ms, Object parameter) throws SQLException {
    Statement stmt = null;
    try {
        Configuration configuration = ms.getConfiguration();
        // 创建 StatementHandler 对象
        StatementHandler handler = configuration.newStatementHandler(this, ms, parameter, RowBounds.DEFAULT, null, null);
        // 初始化 StatementHandler 对象
        stmt = prepareStatement(handler, ms.getStatementLog());
        // <3> 执行 StatementHandler ，进行写操作
        return handler.update(stmt);
    } finally {
        // 关闭 StatementHandler 对象
        closeStatement(stmt);
    }
}
```

- 相比 `#doQuery(...)` 方法，差异点在 <span id='go4.4_3'>`<3>`</span>处，换成了调用 `StatementHandler#update(Statement statement)` 方法，进行**写操作**。

### 4.5 doFlushStatements

```java
// SimpleExecutor.java

@Override
public List<BatchResult> doFlushStatements(boolean isRollback) throws SQLException {
    return Collections.emptyList();
}
```

- 不存在批量操作的情况，所以直接返回空数组。

## 5. ReuseExecutor

`org.apache.ibatis.executor.ReuseExecutor` ，继承 BaseExecutor 抽象类，可重用的 Executor 实现类。

- 每次开始读或写操作，优先从缓存中获取对应的 Statement 对象(**sql作为key查找Statement对象**)。如果不存在，才进行创建。
- 执行完成后，不关闭该 Statement 对象。
- 其它的，和 SimpleExecutor 是一致的。

> `ReuseExecutor`就是依赖`Map<String, Statement>`来完成对Statement的重用的（用完不关）。
>
> - 总不能一直不关吧？到底什么时候关闭这些Statement对象的？
>   - 方法`#flushStatements()`就是用来处理这些Statement对象的。
>   - **在执行`commit`、`rollback`等动作前，将会执行`#flushStatements()`方法，将Statement对象逐一关闭。读者可参看BaseExecutor源码。**

### 5.1 构造方法

```java
// ReuseExecutor.java

/**
 * Statement 的缓存
 *
 * KEY ：SQL
 */
private final Map<String, Statement> statementMap = new HashMap<>();

public ReuseExecutor(Configuration configuration, Transaction transaction) {
    super(configuration, transaction);
}
```

### 5.2 doQuery

```java
// ReuseExecutor.java

@Override
public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
    Configuration configuration = ms.getConfiguration();
    // 创建 StatementHandler 对象
    StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
    // <1> 初始化 StatementHandler 对象
    Statement stmt = prepareStatement(handler, ms.getStatementLog());
    // 执行 StatementHandler  ，进行读操作
    return handler.query(stmt, resultHandler);
}
```

- **差异一**，在于 `<1>` 处，调用 `#prepareStatement(StatementHandler handler, Log statementLog)` 方法，初始化 StatementHandler 对象。代码如下：

  ```java
  // ReuseExecutor.java
  
  private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
      Statement stmt;
      BoundSql boundSql = handler.getBoundSql();
      String sql = boundSql.getSql();
      // 存在
      if (hasStatementFor(sql)) {
          // <1.1> 从缓存中获得 Statement 或 PrepareStatement 对象
          stmt = getStatement(sql);
          // <1.2> 设置事务超时时间
          applyTransactionTimeout(stmt);
      // 不存在
      } else {
          // <2.1> 获得 Connection 对象
          Connection connection = getConnection(statementLog);
          // <2.2> 创建 Statement 或 PrepareStatement 对象
          stmt = handler.prepare(connection, transaction.getTimeout());
          // <2.3> 添加到缓存中
          putStatement(sql, stmt);
      }
      // <2> 设置 SQL 上的参数，例如 PrepareStatement 对象上的占位符
      handler.parameterize(stmt);
      return stmt;
  }
  ```

  - 调用 `#hasStatementFor(String sql)` 方法，判断是否存在对应的 Statement 对象。代码如下：

    ```java
    // ReuseExecutor.java
    
    private boolean hasStatementFor(String sql) {
        try {
            return statementMap.keySet().contains(sql) && !statementMap.get(sql).getConnection().isClosed();
        } catch (SQLException e) {
            return false;
        }
    }
    ```

    - 并且，要求连接**未关闭**。

  - 存在

    - `<1.1>` 处，调用 `#getStatement(String s)` 方法，获得 Statement 对象。代码如下：

      ```java
      // ReuseExecutor.java
      
      private Statement getStatement(String s) {
          return statementMap.get(s);
      }
      ```

      - x

    - 【**差异**】`<1.2>` 处，调用 `#applyTransactionTimeout(Statement stmt)` 方法，设置事务超时时间。

  - 不存在

    - `<2.1>` 处，获得 Connection 对象。

    - `<2.2>` 处，调用 `StatementHandler#prepare(Connection connection, Integer transactionTimeout)` 方法，创建 Statement 或 PrepareStatement 对象。

    - 【差异】`<2.3>` 处，调用 `#putStatement(String sql, Statement stmt)` 方法，添加 Statement 对象到缓存中。代码如下：

      ```java
      // ReuseExecutor.java
      
      private void putStatement(String sql, Statement stmt) {
          statementMap.put(sql, stmt);
      }
      ```

      - x

  - `<2>` 处，调用 `StatementHandler#prepare(Statement statement)` 方法，设置 SQL 上的参数，例如 PrepareStatement 对象上的占位符。

- **差异二**，在执行完数据库操作后，不会关闭 Statement 。

### 5.3 doQueryCursor

```java
// ReuseExecutor.java

@Override
protected <E> Cursor<E> doQueryCursor(MappedStatement ms, Object parameter, RowBounds rowBounds, BoundSql boundSql) throws SQLException {
    Configuration configuration = ms.getConfiguration();
    // 创建 StatementHandler 对象
    StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, null, boundSql);
    // 初始化 StatementHandler 对象
    Statement stmt = prepareStatement(handler, ms.getStatementLog());
    // 执行 StatementHandler  ，进行读操作
    return handler.queryCursor(stmt);
}
```

### 5.4 doUpdate

```java
// ReuseExecutor.java

@Override
public int doUpdate(MappedStatement ms, Object parameter) throws SQLException {
    Configuration configuration = ms.getConfiguration();
    // 创建 StatementHandler 对象
    StatementHandler handler = configuration.newStatementHandler(this, ms, parameter, RowBounds.DEFAULT, null, null);
    // 初始化 StatementHandler 对象
    Statement stmt = prepareStatement(handler, ms.getStatementLog());
    // 执行 StatementHandler  ，进行写操作
    return handler.update(stmt);
}
```

### 5.5 doFlushStatements

```java
// ReuseExecutor.java

@Override
public List<BatchResult> doFlushStatements(boolean isRollback) throws SQLException {
    // 关闭缓存的 Statement 对象们
    for (Statement stmt : statementMap.values()) {
        closeStatement(stmt);
    }
    statementMap.clear();
    // 返回空集合
    return Collections.emptyList();
}
```

- ReuseExecutor 考虑到重用性，但是 Statement 最终还是需要有地方关闭。答案就在 `#doFlushStatements(boolean isRollback)` 方法中。而 BaseExecutor 在关闭 `#close()` 方法中，最终也会调用该方法，从而完成关闭缓存的 Statement 对象们。
- 另外，BaseExecutor 在提交或者回滚事务方法中，最终也会调用该方法，也能完成关闭缓存的 Statement 对象们。

## 6. BatchExecutor

`org.apache.ibatis.executor.BatchExecutor` ，继承 BaseExecutor 抽象类，批量执行的 Executor 实现类。

> FROM 祖大俊 [《Mybatis3.3.x技术内幕（四）：五鼠闹东京之执行器Executor设计原本》](https://my.oschina.net/zudajun/blog/667214)
>
> 执行update（没有select，JDBC批处理不支持select），将所有sql都添加到批处理中（addBatch()），等待统一执行（executeBatch()），它缓存了多个Statement对象，每个Statement对象都是addBatch()完毕后，等待逐一执行executeBatch()批处理的；BatchExecutor相当于维护了多个桶，每个桶里都装了很多属于自己的SQL，就像苹果蓝里装了很多苹果，番茄蓝里装了很多番茄，最后，再统一倒进仓库。（可以是Statement或PrepareStatement对象）

### 6.1 构造方法

```java
// BatchExecutor.java

/** Statement 数组，每个Statement都是addBatch()后，等待执行 **/
private final List<Statement> statementList = new ArrayList<>();

/**
 * BatchResult 数组。对应的结果集（主要保存了update结果的count数量）
 *  
 *  每一个 BatchResult 元素，对应一个 {@link #statementList} 的 Statement 元素
 */
private final List<BatchResult> batchResultList = new ArrayList<>();

/** 当前 SQL，即上次执行的sql **/
private String currentSql;

/** 当前 MappedStatement 对象 **/
private MappedStatement currentStatement;


public BatchExecutor(Configuration configuration, Transaction transaction) {
    super(configuration, transaction);
}
```

- `currentSql` 和 `currentStatement` 属性，当前 SQL 和 MappedStatement 对象。
- `batchResultList` 和 `statementList` 属性，分别是 BatchResult 和 Statement 数组。并且，每一个 `batchResultList` 的 BatchResult 元素，对应一个 `statementList` 的 Statement 元素。
- 具体怎么应用上述属性，我们见 `#doUpdate(...)` 和 `#doFlushStatements(...)` 方法。

### 6.2 BatchResult

`org.apache.ibatis.executor.BatchResult` ，相同 SQL 聚合的结果。代码如下：

```java
// BatchResult.java

/**
 * MappedStatement 对象
 */
private final MappedStatement mappedStatement;
/**
 * SQL
 */
private final String sql;
/**
 * 参数对象集合
 *
 * 每一个元素，对应一次操作的参数
 */
private final List<Object> parameterObjects;
/**
 * 更新数量集合
 *
 * 每一个元素，对应一次操作的更新数量
 */
private int[] updateCounts;

// ... 省略 setting / getting 相关方法
```

### 6.3 doUpdate

```java
// BatchExecutor.java

@Override
public int doUpdate(MappedStatement ms, Object parameterObject) throws SQLException {
    final Configuration configuration = ms.getConfiguration();
    // <1> 创建 StatementHandler 对象
    final StatementHandler handler = configuration.newStatementHandler(this, ms, parameterObject, RowBounds.DEFAULT, null, null);
    final BoundSql boundSql = handler.getBoundSql();
    final String sql = boundSql.getSql();
    final Statement stmt;
    // <2> 如果匹配最后一次 currentSql 和 currentStatement ，则聚合到 BatchResult 中
    if (sql.equals(currentSql) && ms.equals(currentStatement)) {
        // <2.1> 获得最后一次的 Statement 对象
        int last = statementList.size() - 1;
        stmt = statementList.get(last);
        // <2.2> 设置事务超时时间
        applyTransactionTimeout(stmt);
        // <2.3> 设置 SQL 上的参数，例如 PrepareStatement 对象上的占位符
        handler.parameterize(stmt);//fix Issues 322
        // <2.4> 获得最后一次的 BatchResult 对象，并添加参数到其中
        BatchResult batchResult = batchResultList.get(last);
        batchResult.addParameterObject(parameterObject);
    // <3> 如果不匹配最后一次 currentSql 和 currentStatement ，则新建 BatchResult 对象
    } else {
        // <3.1> 获得 Connection
        Connection connection = getConnection(ms.getStatementLog());
        // <3.2> 创建 Statement 或 PrepareStatement 对象
        stmt = handler.prepare(connection, transaction.getTimeout());
        // <3.3> 设置 SQL 上的参数，例如 PrepareStatement 对象上的占位符
        handler.parameterize(stmt);    //fix Issues 322
        // <3.4> 重新设置 currentSql 和 currentStatement
        currentSql = sql;
        currentStatement = ms;
        // <3.5> 添加 Statement 到 statementList 中
        statementList.add(stmt);
        // <3.6> 创建 BatchResult 对象，并添加到 batchResultList 中
        batchResultList.add(new BatchResult(ms, sql, parameterObject));
    }
    // handler.parameterize(stmt);
    // <4> 批处理
    handler.batch(stmt);
    return BATCH_UPDATE_RETURN_VALUE;
}

// <2> 需要注意的是sql.equals(currentSql)和statementList.get(last)，充分说明了其有序逻辑：
//  AABB，将生成2个Statement对象；AABBAA，将生成3个Statement对象，而不是2个。因为，只要sql有变化，将导致生成新的Statement对象。
```

> **缓存了这么多Statement批处理对象，何时执行它们？在[`#doFlushStatements()`](#6.4 doFlushStatements)方法中完成执行stmt.executeBatch()，随即关闭这些Statement对象。**
>
> 注：对于批处理来说，JDBC只支持update操作（update、insert、delete等），不支持select查询操作。
>
> **BatchExecutor和JDBC批处理的区别?**
>
> - JDBC中Statement的批处理原理图。
>   - <img src="http://static.oschina.net/uploads/space/2016/0427/211258_IEzi_2727738.png" alt="img" style="zoom: 50%;" />
>   - 对于`Statement`来说，只要SQL不同，就会产生新编译动作，`Statement`不支持问号“?”参数占位符。
> - JDBC中PrepareStatement的批处理原理图。
>   - <img src="http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202202021638049.png" alt="img" style="zoom: 50%;" />
>   - 对于`PrepareStatement`，只要SQL相同，就只会编译一次，如果SQL不同呢？此时和Statement一样，会编译多次。`PrepareStatement`的优势在于支持问号“?”参数占位符，SQL相同，参数不同时，可以减少编译次数至一次，大大提高效率；另外可以防止SQL注入漏洞。
> - BatchExecutor的批处理原理图。
>   - <img src="http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202202021639698.png" alt="img" style="zoom:50%;" />
>   - `BatchExecutor`的批处理，和JDBC的批处理，主要区别就是BatchExecutor维护了一组Statement批处理对象，它有自动路由功能，SQL1、SQL2、SQL3代表不同的SQL。（Statement或Preparestatement）

- <span id='go6.3_1'>`<1>`</span>处，调用 `Configuration#newStatementHandler(...)` 方法，创建 StatementHandler 对象。

- `<2>` 和 `<3>` 处，就是两种不同的情况，差异点在于传入的 `sql` 和 `ms` 是否匹配当前的 `currentSql` 和 `currentStatement` 。如果是，则继续聚合到最后的 BatchResult 中，否则，创建新的 BatchResult 对象，进行“聚合”。

- <span id='go6.3_2'>`<2>`</span>块：

  - `<2.1>`、`<2.2>`、`<2.3>` 处，逻辑和 ReuseExecutor 相似，使用可重用的 Statement 对象，并进行初始化。
  - `<2.4>` 处，获得最后一次的 BatchResult 对象，并添加参数到其中。（使用同一个BatchResult对象，paramrObject添加新的参数对象，场景是:同样的sql，参数有点不一样）

- <span id='go6.3_3'>`<3>`</span>块：

  - `<3.1>`、`<3.2>`、`<3.3>` 处，逻辑和 SimpleExecutor 相似，创建新的 Statement 对象，并进行初始化。
  - `<3.4>` 处，重新设置 `currentSql` 和 `currentStatement` ，为当前传入的 `sql` 和 `ms` 。
  - `<3.5>` 处，添加 Statement 到 `statementList` 中。
  - `<3.6>` 处，创建 BatchResult 对象，并添加到 `batchResultList` 中。
  - 那么，如果下一次执行这个方法，如果传递相同的 `sql` 和 `ms` 进来，就会聚合到目前新创建的 BatchResult 对象中。

- <span id='go6.3_4'>`<4>`</span>处，调用 `StatementHandler#batch(Statement statement)` 方法，批处理。代码如下：

  ```java
  // 有多个实现类，先看两个
  
  // PreparedStatementHandler.java
  
  @Override
  public void batch(Statement statement) throws SQLException {
      PreparedStatement ps = (PreparedStatement) statement;
      ps.addBatch();
  }
  
  // SimpleStatementHandler.java
  
  @Override
  public void batch(Statement statement) throws SQLException {
      String sql = boundSql.getSql();
      statement.addBatch(sql);
  }
  ```

  - 这段代码，是不是非常熟悉。

### 6.4 doFlushStatements

```java
// BatchExecutor.java

@Override
public List<BatchResult> doFlushStatements(boolean isRollback) throws SQLException {
    try {
        // <1> 如果 isRollback 为 true ，返回空数组
        if (isRollback) {
            return Collections.emptyList();
        }
        // <2> 遍历 statementList 和 batchResultList 数组，逐个提交批处理
        List<BatchResult> results = new ArrayList<>();
        for (int i = 0, n = statementList.size(); i < n; i++) {
            // <2.1> 获得 Statement 和 BatchResult 对象
            Statement stmt = statementList.get(i);
            applyTransactionTimeout(stmt);
            BatchResult batchResult = batchResultList.get(i);
            try {
                // <2.2> 批量执行
                batchResult.setUpdateCounts(stmt.executeBatch());
                // <2.3> 处理主键生成
                MappedStatement ms = batchResult.getMappedStatement();
                List<Object> parameterObjects = batchResult.getParameterObjects();
                KeyGenerator keyGenerator = ms.getKeyGenerator();
                if (Jdbc3KeyGenerator.class.equals(keyGenerator.getClass())) {
                    Jdbc3KeyGenerator jdbc3KeyGenerator = (Jdbc3KeyGenerator) keyGenerator;
                    jdbc3KeyGenerator.processBatch(ms, stmt, parameterObjects);
                } else if (!NoKeyGenerator.class.equals(keyGenerator.getClass())) { //issue #141
                    for (Object parameter : parameterObjects) {
                        keyGenerator.processAfter(this, ms, stmt, parameter);
                    }
                }
                // Close statement to close cursor #1109
                // <2.4> 关闭 Statement 对象
                closeStatement(stmt);
            } catch (BatchUpdateException e) {
                // 如果发生异常，则抛出 BatchExecutorException 异常
                StringBuilder message = new StringBuilder();
                message.append(batchResult.getMappedStatement().getId())
                        .append(" (batch index #")
                        .append(i + 1)
                        .append(")")
                        .append(" failed.");
                if (i > 0) {
                    message.append(" ")
                            .append(i)
                            .append(" prior sub executor(s) completed successfully, but will be rolled back.");
                }
                throw new BatchExecutorException(message.toString(), e, results, batchResult);
            }
            // <2.5> 添加到结果集
            results.add(batchResult);
        }
        return results;
    } finally {
        // <3.1> 关闭 Statement 们
        for (Statement stmt : statementList) {
            closeStatement(stmt);
        }
        // <3.2> 置空 currentSql、statementList、batchResultList 属性
        currentSql = null;
        statementList.clear();
        batchResultList.clear();
    }
}
```

- `<1>` 处，如果 `isRollback` 为 `true` ，返回空数组。
- `<2>`处，遍历`statementList`和`batchResultList`数组，逐个 Statement 提交批处理。
  - `<2.1>` 处，获得 Statement 和 BatchResult 对象。
  - 【重要】`<2.2>` 处，调用 `Statement#executeBatch()` 方法，批量执行。执行完成后，将结果赋值到 `BatchResult.updateCounts` 中。
  - `<2.3>` 处，处理主键生成。详细解析，见 [《精尽 MyBatis 源码分析 —— SQL 执行（三）之 KeyGenerator》](http://svip.iocoder.cn/MyBatis/executor-3) 。
  - `<2.4>` 处，调用 `#closeStatement(stmt)` 方法，关闭 Statement 对象。
  - `<2.5>` 处，添加到结果集。
- `<3.1>` 处，关闭 `Statement` 们。
- `<3.2>` 处，置空 `currentSql`、`statementList`、`batchResultList` 属性。

### 6.5 doQuery

```java
// BatchExecutor.java

@Override
public <E> List<E> doQuery(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql)
        throws SQLException {
    Statement stmt = null;
    try {
        // <1> 刷入批处理语句
        flushStatements();
        Configuration configuration = ms.getConfiguration();
        // 创建 StatementHandler 对象
        StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameterObject, rowBounds, resultHandler, boundSql);
        // 获得 Connection 对象
        Connection connection = getConnection(ms.getStatementLog());
        // 创建 Statement 或 PrepareStatement 对象
        stmt = handler.prepare(connection, transaction.getTimeout());
        // 设置 SQL 上的参数，例如 PrepareStatement 对象上的占位符
        handler.parameterize(stmt);
        // 执行 StatementHandler  ，进行读操作
        return handler.query(stmt, resultHandler);
    } finally {
        // 关闭 StatementHandler 对象
        closeStatement(stmt);
    }
}
```

- 和 SimpleExecutor 的该方法，逻辑差不多。差别在于 `<1>` 处，发生查询之前，先调用 `#flushStatements()` 方法，刷入批处理语句。

### 6.6 doQueryCursor

```java
// BatchExecutor.java

@Override
protected <E> Cursor<E> doQueryCursor(MappedStatement ms, Object parameter, RowBounds rowBounds, BoundSql boundSql) throws SQLException {
    // <1> 刷入批处理语句
    flushStatements();
    Configuration configuration = ms.getConfiguration();
    // 创建 StatementHandler 对象
    StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, null, boundSql);
    // 获得 Connection 对象
    Connection connection = getConnection(ms.getStatementLog());
    // 创建 Statement 或 PrepareStatement 对象
    Statement stmt = handler.prepare(connection, transaction.getTimeout());
    // 设置 Statement ，如果执行完成，则进行自动关闭
    stmt.closeOnCompletion();
    // 设置 SQL 上的参数，例如 PrepareStatement 对象上的占位符
    handler.parameterize(stmt);
    // 执行 StatementHandler  ，进行读操作
    return handler.queryCursor(stmt);
}
```

- 和 SimpleExecutor 的该方法，逻辑差不多。差别在于 `<1>` 处，发生查询之前，先调用 `#flushStatements()` 方法，刷入批处理语句。

## 7. 二级缓存

在开始看具体源码之间，我们先来理解**二级缓存**的定义：

> FROM 凯伦 [《聊聊MyBatis缓存机制》](https://tech.meituan.com/mybatis_cache.html)
>
> 在上文中提到的一级缓存中，**其最大的共享范围就是一个 SqlSession 内部**，如果多个 SqlSession 之间需要共享缓存，则需要使用到**二级缓存**。开启二级缓存后，会使用 CachingExecutor 装饰 Executor ，进入一级缓存的查询流程前，先在 CachingExecutor 进行二级缓存的查询，具体的工作流程如下所示。
>
> [![大体流程](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202201261421015.png)](http://static.iocoder.cn/images/MyBatis/2020_02_28/04.png)大体流程

- 那么，二级缓存，反应到具体代码里，是长什么样的呢？我们来打开 MappedStatement 类，代码如下：

  ```java
  // MappedStatement.java
  
  /**
   * Cache 对象
   */
  private Cache cache;
  ```

  - 就是 `cache` 属性。在前面的文章中，我们已经看到，每个 XML Mapper 或 Mapper 接口的每个 SQL 操作声明，对应一个 MappedStatement 对象。通过 `@CacheNamespace` 或 `<cache />` 来声明，创建其所使用的 Cache 对象；也可以通过 `@CacheNamespaceRef` 或 `<cache-ref />` 来声明，使用指定 Namespace 的 Cache 对象。

  - 最终在 Configuration 类中的体现，代码如下：

    ```java
    // Configuration.java
    
    /**
     * Cache 对象集合
     *
     * KEY：命名空间 namespace
     */
    protected final Map<String, Cache> caches = new StrictMap<>("Caches collection");
    ```

    - 一个 KEY 为 Namespace 的 Map 对象。

  - 可能上述描述比较绕口，胖友好好理解下。

- 通过在 `mybatis-config.xml` 中，配置如下开启二级缓存功能：

  ```xml
  <setting name="cacheEnabled" value="true"/>
  ```

### 7.1 CachingExecutor

`org.apache.ibatis.executor.CachingExecutor` ，实现 Executor 接口，支持**二级缓存**的 `Executor `的实现类。装饰设计模式典范。

先从缓存中获取查询结果，存在就返回，不存在，再委托给`Executor `**#delegate**去数据库取，**delegate**可以是上面任一的`SimpleExecutor`、`ReuseExecutor`、`BatchExecutor`。

#### 7.1.1 构造方法

```java
// CachingExecutor.java

/**
 * 被委托的 Executor 对象
 */
private final Executor delegate;
/**
 * TransactionalCacheManager 对象
 */
private final TransactionalCacheManager tcm = new TransactionalCacheManager();

public CachingExecutor(Executor delegate) {
    // <1>
    this.delegate = delegate;
    // <2> 设置 delegate 被当前执行器所包装
    delegate.setExecutorWrapper(this);
}
```

- `tcm` 属性，TransactionalCacheManager 对象，支持事务的缓存管理器。因为**二级缓存**是支持跨 Session 进行共享，此处需要考虑事务，**那么，必然需要做到事务提交时，才将当前事务中查询时产生的缓存，同步到二级缓存中**。这个功能，就通过 `TransactionalCacheManager `来实现。
- `<1>` 处，设置 `delegate` 属性，为被委托的 Executor 对象。
- `<2>` 处，调用 `delegate` 属性的 `#setExecutorWrapper(Executor executor)` 方法，设置 `delegate` 被**当前执行器**所包装。

#### 7.1.2 直接调用委托方法

CachingExecutor 的如下方法，具体的实现代码，是直接调用委托执行器 `delegate` 的对应的方法。代码如下：

```java
// CachingExecutor.java

public Transaction getTransaction() { return delegate.getTransaction(); }

@Override
public boolean isClosed() { return delegate.isClosed(); }

@Override
public List<BatchResult> flushStatements() throws SQLException { return delegate.flushStatements(); }

@Override
public CacheKey createCacheKey(MappedStatement ms, Object parameterObject, RowBounds rowBounds, BoundSql boundSql) { return delegate.createCacheKey(ms, parameterObject, rowBounds, boundSql); }

@Override
public boolean isCached(MappedStatement ms, CacheKey key) { return delegate.isCached(ms, key); }

@Override
public void deferLoad(MappedStatement ms, MetaObject resultObject, String property, CacheKey key, Class<?> targetType) { delegate.deferLoad(ms, resultObject, property, key, targetType); }

@Override
public void clearLocalCache() { delegate.clearLocalCache(); }
```

#### 7.1.3 query

```java
// CachingExecutor.java

@Override
public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
    // 获得 BoundSql 对象
    BoundSql boundSql = ms.getBoundSql(parameterObject);
    // 创建 CacheKey 对象
    CacheKey key = createCacheKey(ms, parameterObject, rowBounds, boundSql);
    // 查询
    return query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
}

@Override
public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
        throws SQLException {
    // <1> 
    Cache cache = ms.getCache();
    if (cache != null) { // <2> 
        // <2.1> 如果需要清空缓存，则进行清空
        flushCacheIfRequired(ms);
        if (ms.isUseCache() && resultHandler == null) { // <2.2>
            // 暂时忽略，存储过程相关
            ensureNoOutParams(ms, boundSql);
            @SuppressWarnings("unchecked")
            // <2.3> 从二级缓存中，获取结果
            List<E> list = (List<E>) tcm.getObject(cache, key);
            if (list == null) {
                // <2.4.1> 如果不存在，则从数据库中查询
                list = delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
                // <2.4.2> 缓存结果到二级缓存中
                tcm.putObject(cache, key, list); // issue #578 and #116
            }
            // <2.5> 如果存在，则直接返回结果
            return list;
        }
    }
    // <3> 不使用缓存，则从数据库中查询
    return delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
}
```

- `<1>` 处，调用 `MappedStatement#getCache()` 方法，获得 Cache 对象，即当前 MappedStatement 对象的**二级缓存**。

- `<3>` 处，如果**没有** Cache 对象，说明该 MappedStatement 对象，未设置**二级缓存**，则调用 `delegate` 属性的 `#query(...)` 方法，直接从数据库中查询。

- `<2>` 处，如果**有** Cache 对象，说明该 MappedStatement 对象，有设置**二级缓存**：

  - `<2.1>` 处，调用 `#flushCacheIfRequired(MappedStatement ms)` 方法，如果需要清空缓存，则进行清空。代码如下：

    ```java
    // CachingExecutor.java
    
    private void flushCacheIfRequired(MappedStatement ms) {
        Cache cache = ms.getCache();
        if (cache != null && ms.isFlushCacheRequired()) { // 是否需要清空缓存
            tcm.clear(cache);
        }
    }
    ```

    - 通过 `@Options(flushCache = Options.FlushCachePolicy.TRUE)` 或 `<select flushCache="true">` 方式，开启需要清空缓存。
    - 调用 `TransactionalCache#clear()` 方法，清空缓存。**注意**，此**时**清空的仅仅，当前事务中查询数据产生的缓存。而**真正**的清空，在事务的提交时。这是为什么呢？还是因为**二级缓存**是跨 Session 共享缓存，在事务尚未结束时，不能对二级缓存做任何修改。😈 可能有点绕，胖友好好理解。

  - `<2.2>` 处，当 `MappedStatement#isUseCache()` 方法，返回 `true` 时，才使用二级缓存。默认开启。可通过 `@Options(useCache = false)` 或 `<select useCache="false">` 方法，关闭。

  - `<2.3>` 处，调用 `TransactionalCacheManager#getObject(Cache cache, CacheKey key)` 方法，从二级缓存中，获取结果。

  - 如果不存在缓存

    - `<2.4.1>` 处，调用 `delegate` 属性的 `#query(...)` 方法，再从数据库中查询。
    - <span id='go_7.1.4_2.4.2'>`<2.4.2>` </span>处，调用 `TransactionalCacheManager#put(Cache cache, CacheKey key, Object value)` 方法，缓存结果到二级缓存中。😈 当然，正如上文所言，实际上，此处结果还没添加到二级缓存中。那具体是怎么样的呢？答案见 [TransactionalCache](#7.2.2 putObject) 。

  - 如果存在缓存

    - `<2.5>` 处，如果存在，则直接返回结果。

#### 7.1.4 queryCursor

```java
// CachingExecutor.java

@Override
public <E> Cursor<E> queryCursor(MappedStatement ms, Object parameter, RowBounds rowBounds) throws SQLException {
    // 如果需要清空缓存，则进行清空
    flushCacheIfRequired(ms);
    // 执行 delegate 对应的方法
    return delegate.queryCursor(ms, parameter, rowBounds);
}
```

- 无法开启二级缓存，所以只好调用 `delegate` 对应的方法。

#### 7.1.5 update

```java
// CachingExecutor.java

@Override
public int update(MappedStatement ms, Object parameterObject) throws SQLException {
    // 如果需要清空缓存，则进行清空
    flushCacheIfRequired(ms);
    // 执行 delegate 对应的方法
    return delegate.update(ms, parameterObject);
}
```

#### 7.1.6 commit

```java
// CachingExecutor.java

@Override
public void commit(boolean required) throws SQLException {
    // 执行 delegate 对应的方法
    delegate.commit(required);
    // 提交 TransactionalCacheManager
    tcm.commit();
}
```

- `delegate` 和 `tcm` 先后提交。

#### 7.1.7 rollback

```java
// CachingExecutor.java

@Override
public void rollback(boolean required) throws SQLException {
    try {
        // 执行 delegate 对应的方法
        delegate.rollback(required);
    } finally {
        if (required) {
            // 回滚 TransactionalCacheManager
            tcm.rollback();
        }
    }
}
```

- `delegate` 和 `tcm` 先后回滚。

#### 7.1.8 close

```java
// CachingExecutor.java

@Override
public void close(boolean forceRollback) {
    try {
        //issues #499, #524 and #573
        // 如果强制回滚，则回滚 TransactionalCacheManager
        if (forceRollback) {
            tcm.rollback();
        // 如果强制提交，则提交 TransactionalCacheManager
        } else {
            tcm.commit();
        }
    } finally {
        // 执行 delegate 对应的方法
        delegate.close(forceRollback);
    }
}
```

- 根据 `forceRollback` 属性，进行 `tcm` 和 `delegate` 对应的操作。

### 7.2 TransactionalCacheManager

`org.apache.ibatis.cache.TransactionalCacheManager` ，TransactionalCache 管理器。

#### 7.2.1 构造方法

```java
// TransactionalCacheManager.java

/**
 * Cache 和 TransactionalCache 的映射
 */
private final Map<Cache, TransactionalCache> transactionalCaches = new HashMap<>();
```

- 我们可以看到，`transactionalCaches` 是一个使用 Cache 作为 KEY ，TransactionalCache 作为 VALUE 的 Map 对象。

- 为什么是一个 Map 对象呢？因为在一次的事务过程中，可能有多个不同的 MappedStatement 操作，而它们可能对应多个 Cache 对象。

- TransactionalCache 是怎么创建的呢？答案在 `#getTransactionalCache(Cache cache)` 方法，代码如下：

  ```java
  // TransactionalCacheManager.java
  
  private TransactionalCache getTransactionalCache(Cache cache) {
      return transactionalCaches.computeIfAbsent(cache, TransactionalCache::new);
  }
  ```

  - 优先，从 `transactionalCaches` 获得 Cache 对象，对应的 TransactionalCache 对象。
  - 如果不存在，则创建一个 TransactionalCache 对象，并添加到 `transactionalCaches` 中。

#### 7.2.2 putObject

`#putObject(Cache cache, CacheKey key, Object value)` 方法，添加 Cache + KV ，到缓存中。代码如下： [<-](#go_7.1.4_2.4.2)

```java
// TransactionalCacheManager.java

public void putObject(Cache cache, CacheKey key, Object value) {
    // 首先，获得 Cache 对应的 TransactionalCache 对象
    // 然后，添加 KV 到 TransactionalCache 对象中
    getTransactionalCache(cache).putObject(key, value);
}
```

#### 7.2.3 getObject

`#getObject(Cache cache, CacheKey key)` 方法，获得缓存中，指定 Cache + K 的值。代码如下：

```java
// TransactionalCacheManager.java

public Object getObject(Cache cache, CacheKey key) {
    // 首先，获得 Cache 对应的 TransactionalCache 对象
    // 然后从 TransactionalCache 对象中，获得 key 对应的值
    return getTransactionalCache(cache).getObject(key);
}
```

#### 7.2.4 clear

`#clear()` 方法，清空缓存。代码如下：

```java
// TransactionalCacheManager.java

public void clear(Cache cache) {
    getTransactionalCache(cache).clear();
}
```

#### 7.2.5 commit

`#commit()` 方法，提交所有 TransactionalCache 。代码如下：

```java
// TransactionalCacheManager.java

public void commit() {
    for (TransactionalCache txCache : transactionalCaches.values()) {
        txCache.commit();
    }
}
```

- 通过调用该方法，TransactionalCache 存储的当前事务的缓存，会同步到其对应的 Cache 对象。

#### 7.2.6 rollback

`#rollback()` 方法，回滚所有 TransactionalCache 。代码如下：

```java
// TransactionalCacheManager.java

public void rollback() {
    for (TransactionalCache txCache : transactionalCaches.values()) {
        txCache.rollback();
    }
}
```

### 7.3 TransactionalCache

`org.apache.ibatis.cache.decorators.TransactionalCache` ，实现 Cache 接口，支持事务的 Cache 实现类，主要用于二级缓存中。英语比较好的胖友，可以看看如下注释：[<-](#go_7.1.4_2.4.2)

> This class holds all cache entries that are to be added to the 2nd level cache during a Session.
> Entries are sent to the cache when commit is called or discarded if the Session is rolled back.
> Blocking cache support has been added. Therefore any get() that returns a cache miss
> will be followed by a put() so any lock associated with the key can be released.

#### 7.3.1 构造方法

```java
// TransactionalCache.java

/**
 * 委托的 Cache 对象。
 *
 * 实际上，就是二级缓存 Cache 对象。
 */
private final Cache delegate;
/**
 * 提交时，清空 {@link #delegate}
 *
 * 初始时，该值为 false
 * 清理后{@link #clear()} 时，该值为 true ，表示持续处于清空状态
 */
private boolean clearOnCommit;
/**
 * 待提交的 KV 映射
 */
private final Map<Object, Object> entriesToAddOnCommit;
/**
 * 查找不到的 KEY 集合
 */
private final Set<Object> entriesMissedInCache;

public TransactionalCache(Cache delegate) {
    this.delegate = delegate;
    this.clearOnCommit = false;
    this.entriesToAddOnCommit = new HashMap<>();
    this.entriesMissedInCache = new HashSet<>();
}
```

- 胖友认真看下每个变量上的注释。
- 在事务未提交时，`entriesToAddOnCommit` 属性，会暂存当前事务新产生的缓存 KV 对。
- 在事务提交时，`entriesToAddOnCommit` 属性，会同步到二级缓存 `delegate` 中。

#### 7.3.2 getObject

```java
// TransactionalCache.java

@Override
public Object getObject(Object key) {
    // issue #116
    // <1> 从 delegate 中获取 key 对应的 value
    Object object = delegate.getObject(key);
    // <2> 如果不存在，则添加到 entriesMissedInCache 中
    if (object == null) {
        entriesMissedInCache.add(key);
    }
    // issue #146
    // <3> 如果 clearOnCommit 为 true ，表示处于持续清空状态，则返回 null
    if (clearOnCommit) {
        return null;
    // <4> 返回 value
    } else {
        return object;
    }
}
```

- `<1>` 处，调用 `delegate` 的 `#getObject(Object key)` 方法，从 `delegate` 中获取 `key` 对应的 value 。
- `<2>` 处，如果不存在，则添加到 `entriesMissedInCache` 中。这是个神奇的逻辑？？？答案见 `commit()` 和 `#rollback()` 方法。
- `<3>` 处，如果 `clearOnCommit` 为 `true` ，表示处于持续清空状态，则返回 `null` 。因为在事务未结束前，我们执行的**清空缓存**操作不好同步到 `delegate` 中，所以只好通过 `clearOnCommit` 来标记处于清空状态。那么，如果处于该状态，自然就不能返回 `delegate` 中查找的结果。
- `<4>` 处，返回 value 。

#### 7.3.3 putObject

`#putObject(Object key, Object object)` 方法，暂存 KV 到 `entriesToAddOnCommit` 中。代码如下：

```java
// TransactionalCache.java

@Override
public void putObject(Object key, Object object) {
    // 暂存 KV 到 entriesToAddOnCommit 中
    entriesToAddOnCommit.put(key, object);
}
```

#### 7.3.4 removeObject

```java
// TransactionalCache.java

public Object removeObject(Object key) {
    return null;
}
```

- 不太明白为什么是这样的实现。不过目前也暂时不存在调用该方法的情况。暂时忽略。

#### 7.3.5 clear

`#clear()` 方法，清空缓存。代码如下：

```java
// TransactionalCache.java

@Override
public void clear() {
    // <1> 标记 clearOnCommit 为 true
    clearOnCommit = true;
    // <2> 清空 entriesToAddOnCommit
    entriesToAddOnCommit.clear();
}
```

- `<1>` 处，标记 `clearOnCommit` 为 `true` 。
- `<2>` 处，清空 `entriesToAddOnCommit` 。
- 该方法，不会清空 `delegate` 的缓存。真正的清空，在事务提交时。

#### 7.3.6 commit

`#commit()` 方法，提交事务。**重头戏**，代码如下：

```java
// TransactionalCache.java

public void commit() {
    // <1> 如果 clearOnCommit 为 true ，则清空 delegate 缓存
    if (clearOnCommit) {
        delegate.clear();
    }
    // 将 entriesToAddOnCommit、entriesMissedInCache 刷入 delegate 中
    flushPendingEntries();
    // 重置
    reset();
}
```

- `<1>` 处，如果 `clearOnCommit` 为 `true` ，则清空 `delegate` 缓存。

- `<2>` 处，调用 `#flushPendingEntries()` 方法，将 `entriesToAddOnCommit`、`entriesMissedInCache` 同步到 `delegate` 中。代码如下：

  ```java
  // TransactionalCache.java
  
  private void flushPendingEntries() {
      // 将 entriesToAddOnCommit 刷入 delegate 中
      for (Map.Entry<Object, Object> entry : entriesToAddOnCommit.entrySet()) {
          delegate.putObject(entry.getKey(), entry.getValue());
      }
      // 将 entriesMissedInCache 刷入 delegate 中
      for (Object entry : entriesMissedInCache) {
          if (!entriesToAddOnCommit.containsKey(entry)) {
              delegate.putObject(entry, null);
          }
      }
  }
  ```

  - 在看这段代码时，笔者一直疑惑 `entriesMissedInCache` 同步到 `delegate` 中，会不会存在问题。因为当前事务未查找到，不代表其他事务恰好实际能查到。这样，岂不是会将缓存错误的置空。后来一想，缓存即使真的被错误的置空，最多也就多从数据库中查询一次罢了。😈

- `<3>` 处，调用 `#reset()` 方法，重置对象。代码如下：

  ```java
  // TransactionalCache.java
  
  private void reset() {
      // 重置 clearOnCommit 为 false
      clearOnCommit = false;
      // 清空 entriesToAddOnCommit、entriesMissedInCache
      entriesToAddOnCommit.clear();
      entriesMissedInCache.clear();
  }
  ```

  - 因为，一个 Executor 可以提交多次事务，而 TransactionalCache 需要被重用，那么就需要重置回初始状态。

#### 7.3.7 rollback

`#rollback()` 方法，回滚事务。代码如下：

```java
// TransactionalCache.java

public void rollback() {
    // <1> 从 delegate 移除出 entriesMissedInCache
    unlockMissedEntries();
    // <2> 重置
    reset();
}
```

- `<1>` 处，调用 `#unlockMissedEntries()` 方法，将 `entriesMissedInCache` 同步到 `delegate` 中。代码如下：

  ```java
  // TransactionalCache.java
  
  private void unlockMissedEntries() {
      for (Object entry : entriesMissedInCache) {
          try {
              delegate.removeObject(entry);
          } catch (Exception e) {
              log.warn("Unexpected exception while notifiying a rollback to the cache adapter."
                      + "Consider upgrading your cache adapter to the latest version.  Cause: " + e);
          }
      }
  }
  ```

  - 即使事务回滚，也不妨碍在事务的执行过程中，发现 `entriesMissedInCache` 不存在对应的缓存。

- `<2>` 处，调用 `#reset()` 方法，重置对象。

## 8. 创建 Executor 对象

在上面的文章中，我们已经看了各种 Executor 的实现代码。那么，Executor 对象究竟在 MyBatis 中，是如何被创建的呢？Configuration 类中，提供 `#newExecutor(Transaction transaction, ExecutorType executorType)` 方法，代码如下：

```java
// Configuration.java

/**
 * 创建 Executor 对象
 *
 * @param transaction 事务对象
 * @param executorType 执行器类型
 * @return Executor 对象
 */
public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
    // <1> 获得执行器类型
    executorType = executorType == null ? defaultExecutorType : executorType; // 使用默认
    executorType = executorType == null ? ExecutorType.SIMPLE : executorType; // 使用 ExecutorType.SIMPLE
    // <2> 创建对应实现的 Executor 对象
    Executor executor;
    if (ExecutorType.BATCH == executorType) {
        executor = new BatchExecutor(this, transaction);
    } else if (ExecutorType.REUSE == executorType) {
        executor = new ReuseExecutor(this, transaction);
    } else {
        executor = new SimpleExecutor(this, transaction);
    }
    // <3> 如果开启缓存，创建 CachingExecutor 对象，进行包装
    if (cacheEnabled) {
        executor = new CachingExecutor(executor);
    }
    // <4> 应用插件
    executor = (Executor) interceptorChain.pluginAll(executor);
    return executor;
}
```

- `<1>` 处，获得执行器类型。可以通过在 `mybatis-config.xml` 配置文件，如下：

  ```xml
  // value 有三种类型：SIMPLE REUSE BATCH
  <setting name="defaultExecutorType" value="" />
  ```

- `org.apache.ibatis.session.ExecutorType` ，执行器类型。代码如下：

  ```java
  // ExecutorType.java
  
  public enum ExecutorType {
  
      /**
       * {@link org.apache.ibatis.executor.SimpleExecutor}
       */
      SIMPLE,
      /**
       * {@link org.apache.ibatis.executor.ReuseExecutor}
       */
      REUSE,
      /**
       * {@link org.apache.ibatis.executor.BatchExecutor}
       */
      BATCH
  
  }
  ```

- `<2>` 处，创建对应实现的 Executor 对象。默认为 SimpleExecutor 对象。

- `<3>` 处，如果开启缓存，创建 CachingExecutor 对象，进行包装。

- `<4>` 处，应用插件。关于**插件**，我们在后续的文章中，详细解析。

## 9. ClosedExecutor

> 老艿艿：写到凌晨 1 点多，以为已经写完了，结果发现….

在 ResultLoaderMap 类中，有一个 ClosedExecutor 内部静态类，继承 BaseExecutor 抽象类，已经关闭的 Executor 实现类。代码如下：

```java
// ResultLoaderMap.java

private static final class ClosedExecutor extends BaseExecutor {

    public ClosedExecutor() {
        super(null, null);
    }

    @Override
    public boolean isClosed() {
        return true;
    }

    @Override
    protected int doUpdate(MappedStatement ms, Object parameter) throws SQLException {
        throw new UnsupportedOperationException("Not supported.");
    }

    @Override
    protected List<BatchResult> doFlushStatements(boolean isRollback) throws SQLException {
        throw new UnsupportedOperationException("Not supported.");
    }

    @Override
    protected <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
        throw new UnsupportedOperationException("Not supported.");
    }

    @Override
    protected <E> Cursor<E> doQueryCursor(MappedStatement ms, Object parameter, RowBounds rowBounds, BoundSql boundSql) throws SQLException {
        throw new UnsupportedOperationException("Not supported.");
    }
}
```

- 仅仅在 ResultLoaderMap 中，作为一个“空”的 Executor 对象。没有什么特殊的意义和用途。

## 10. ErrorContext

`org.apache.ibatis.executor.ErrorContext` ，错误上下文，负责记录错误日志。代码比较简单，也蛮有意思，胖友可以自己研究研究。

另外，也可以参考下 [《Mybatis 源码中获取到的错误日志灵感》](https://www.jianshu.com/p/2af47a3e473c) 。

## 11.总结

### 11.1 Exector通用调用逻辑

> 通用的执行逻辑：
>
> ```java
>    // <1> 创建 StatementHandler 对象
>     StatementHandler handler = configuration.newStatementHandler(wrapper | this, ms, parameter, rowBounds, resultHandler, boundSql);
>      // <2> 初始化 StatementHandler 对象
>      // <2.1> 获得 Connection 对象
>  	Connection connection = getConnection(statementLog);
>  // <2.2> 创建 Statement 或 PrepareStatement 对象
>  	Statement stmt = handler.prepare(connection, transaction.getTimeout());
>  // <2.3> 设置 SQL 上的参数，例如 PrepareStatement 对象上的占位符
>  	handler.parameterize(stmt);
> 
> 
> // 基本上都是 先获取StatementHandler，然后 初始化StatementHandler 对Statement进行操作
> // 这里 Simple  就 直接 handler.prepare、handler.parameterize
> // 这里 Resurce就 分两种情况
> //		存在Statement ，applyTransactionTimeout(stmt); 直接用(设置事务超时时间)
> //            不存在Statement ，handler.prepare、handler.parameterize
> // 这里 Batch  也一样 直接 handler.prepare、handler.parameterize (更新方法略有不同 s) 
> ```
> 
>`#doQuery`、`#doQueryCursor`都是使用`BaseExecutor#wrapper`对象的。其他的方法都是this指代。

### 11.2 Exector比较

**作用范围：前文提到的所有Executor的作用范围，都严格限制在SqlSession生命周期范围内。**

|                                                              | 备注                                                         | doQuery                                                      | doUpdate                                                     | doFlushStatements                                            | doQueryCursor                             |
| :----------------------------------------------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | :---------------------------------------- |
| [**BaseExecutor**](#4. SimpleExecutor)<br/>`实现 Executor 接口，提供骨架方法` |                                                              |                                                              |                                                              | **特性1**：`#commit`、`#rollback`方法会调用该方法。参数不太一样（true、false） |                                           |
| [**SimpleExecutor**](#4. SimpleExecutor)<br>`简单的 Executor 实现` | 每次开始读或写操作，都创建对应的 Statement 对象。  <br>执行完成后，关闭该 Statement 对象。 | [`<1>`](#go4.2_1) 调用`Configuration#newStatementHandler` 方法，**创建 StatementHandler 对象**。<br>[`<2>`](#go4.2_2) 调用 `#prepareStatement` 方法，**初始化 StatementHandler 对象**。(通过`StatementHandler `创建Statement 或 PrepareStatement ，设置 SQL 上的参数，例如 PrepareStatement 对象上的占位符)<br>[`<3>`](#go4.2_3) ，调用 `StatementHandler#query` 方法，进行**读操作**。<br>[`<4>`](#go4.2_4)，调用 `#closeStatement` 方法，**关闭 Statement对象**。 | 相比 `#doQuery(...)` 方法，差异点在 [`<3>`](#go4.4_3) 处，换成了调用 `StatementHandler#update(Statement statement)` 方法，进行**写操作**。 | [不存在批量操作](#4.5 doFlushStatements)的情况，所以直接返回空数组。 | [类似`#doQuery`](#4.3 doQueryCursor)<br/> |
| [**ReuseExecutor**](#5. ReuseExecutor)<br>`可重用的 Executor 实现类` | 每次开始读或写操作，优先从缓存中获取对应的 Statement 对象。如果不存在，才进行创建。 <br/> 执行完成后，不关闭该 Statement 对象。 <br/> 缓存对象：`HashMap<String, Statement> statementMap`； | [`<2>`](#5.2 doQuery) 差异在于调用 `#prepareStatement` 方法时，优先从缓存中获取，且连接**未关闭**。<br>从缓存获取的Statement会调用`BaseExecutor#applyTransactionTimeout`设置事务的超时时间。<br>`<4>`没有关闭`Statement`的操作。 | [同上](#5.4 doUpdate)。 调用Statement优先从缓存中获取。      | [[**差异**]](#5.5 doFlushStatements)<br> `<1> `  关闭缓存的 Statement 对象。同样方法空集合。<br>`ReuseExecutor `考虑到重用性，但是 Statement 最终还是需要有地方关闭。<br>而 `BaseExecutor `在关闭 `#close()` 方法中，最终也会调用该方法，从而完成关闭缓存的 Statement 对象们<br>BaseExecutor 在提交或者回滚事务方法中，最终也会调用该方法，也能完成关闭缓存的 Statement 对象们。 | [类似`#doQuery`](#5.3 doQueryCursor)<br/> |
| [**BatchExecutor**](#6. BatchExecutor)<br/>`批量执行的 Executor 实现类` | 执行`#update`（没有select，JDBC批处理不支持select），将所有sql都添加到批处理中（`addBatch()`），等待统一执行（`executeBatch()`），它缓存了多个`Statement`对象，每个Statement对象都是addBatch()完毕后，等待逐一执行executeBatch()批处理的；<br>BatchExecutor相当于维护了多个桶，每个桶里都装了很多属于自己的SQL，就像苹果蓝里装了很多苹果，番茄蓝里装了很多番茄，最后，再统一倒进仓库。（可以是Statement或PrepareStatement对象） | 和 `SimpleExecutor` 的该方法，逻辑差不多。差别在于 [`<1>`](#6.5 doQuery) 处，发生查询之前，先调用 `#flushStatements()` 方法，刷入批处理语句。 | `<1>`的逻辑与`SimpleExecutor`相同。<br>[`<1>`](#go6.3_1) 用`Configuration#newStatementHandler` 方法，**创建 StatementHandler 对象**。<br/>[`<2>`](#go4.2_2) **初始化 StatementHandler 对象**。有两种情况。异点在于传入的 `sql` 和 `ms` 是否匹配当前的 `currentSql` 和 `currentStatement` 。<br>    [`<2.1>`](#go6.3_2)是:使用之前`BatchResult`<br>    [`<2.2>`](#go6.3_3)否:使用新的`BatchResult`<br>[`<3>`](#go6.3_4) ，调用 `StatementHandler#batch` 方法，进行**批量写操作**。<br/>`<4>`此处不关闭`Statement `，见`#doFlushStatements`。 | 【基于特性1】<br>如果`#rollback`调用，返回空。<br>如果`#commit`调用，遍历 `statementList `和 `batchResultList `数组，逐个提交批处理。并且逐一关闭`Statement `<br>无论是哪一种调用，最后都会清空并关闭`Statement`们还有其他额外字段。 | [类似`#doQuery`](#6.6 doQueryCursor)<br/> |



> 1. **CachingExecutor**：装饰设计模式典范，先从缓存中获取查询结果，存在就返回，不存在，再委托给Executor delegate去数据库取，delegate可以是上面任一的SimpleExecutor、ReuseExecutor、BatchExecutor。
> 2. **ClosedExecutor**：毫无用处，读者可自行查看其源码，仅作为一种标识，和Serializable标记接口作用相当。

### **11.3 Executor的创建时机和创建策略**

**Executor的创建时机是，创建`DefaultSqlSession`实例时，作为构造参数传递进去。**

```java
// DefaultSqlSessionFactory.java
Executor executor = configuration.newExecutor(tx, execType);
return new DefaultSqlSession(configuration, executor, autoCommit);
```

`Executor`有两种手段来指定创建Executor的三种策略:

1. configuration配置文件中，配置默认`ExecutorType`类型。（当不配置时，默认为`ExecutorType.SIMPLE`）

   1. ```xml
      <?xml version="1.0" encoding="UTF-8"?>
      <!DOCTYPE configuration PUBLIC "-//mybatis.org//DTD Config 3.0//EN" "http://mybatis.org/dtd/mybatis-3-config.dtd">
      <configuration>
          <settings>
              <setting name="defaultExecutorType" value="REUSE" />
          </settings>
      </configuration>
      ```

2. 手动给**DefaultSqlSessionFactory.java**的创建SqlSession的方法传递`ExecutorType`参数。

   1. ```java
      @Override
        public SqlSession openSession(ExecutorType execType, boolean autoCommit) {
          return openSessionFromDataSource(execType, null, autoCommit);
        }
      ```

### 11.4 一级、二级缓存原理分析

Mybatis的一级缓存，指的是`SqlSession`级别的缓存，默认开启；Mybatis的二级缓存，指的是`SqlSessionFactory`级别的缓存，需要配置。缓存是针对`<select>`来说的。

#### 11.4.1 一级缓存

> - 以下摘自**美团技术团队-凯伦** [《聊聊MyBatis缓存机制》](https://tech.meituan.com/mybatis_cache.html)
>
> 在应用运行过程中，我们有可能在一次数据库会话中，执行多次查询条件完全相同的SQL，MyBatis提供了一级缓存的方案优化这部分场景，如果是相同的SQL语句，会优先命中一级缓存，避免直接对数据库进行查询，提高性能。具体执行过程如下图所示。
>
> ![img](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202202071310271.jpeg)
>
> 每个`SqlSession`中持有了`Executor`，每个`Executor`中有一个`LocalCache`。当用户发起查询时，MyBatis根据当前执行的语句生成`MappedStatement`，在Local Cache进行查询，如果缓存命中的话，直接返回结果给用户，如果缓存没有命中的话，查询数据库，结果写入`Local Cache`，最后返回结果给用户。具体实现类的类关系图如下图所示。
>
> ![img](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202202071311669.jpeg)

##### 11.4.1.1 一级缓存配置

只需在MyBatis的配置文件中，添加如下语句，就可以使用一级缓存。共有两个选项，`SESSION`或者`STATEMENT`，默认是`SESSION`级别，即在一个MyBatis会话中执行的所有语句，都会共享这一个缓存。一种是`STATEMENT`级别，可以理解为缓存只对当前执行的这一个`Statement`有效。

```xml
<setting name="localCacheScope" value="SESSION"/>
```

##### 11.4.1.2 一级缓存实现总结

开启一级缓存，范围为会话级别。

1. 调用三次`getStudentById`

   - > 只有第一次真正查询了数据库，后续的查询使用了一级缓存。

2. 增加了对数据库的修改操作，验证在一次数据库会话中，如果对数据库发生了修改操作，一级缓存是否会失效。

   - > 在修改操作后执行的相同查询，查询了数据库，**一级缓存失效**。

3. 开启两个`SqlSession`，在`sqlSession1`中查询数据，使一级缓存生效，在`sqlSession2`中更新数据库，验证一级缓存只在数据库会话内部共享。

   - ```java
     @Test
     public void testLocalCacheScope() throws Exception {
             SqlSession sqlSession1 = factory.openSession(true); 
             SqlSession sqlSession2 = factory.openSession(true); 
     
             StudentMapper studentMapper = sqlSession1.getMapper(StudentMapper.class);
             StudentMapper studentMapper2 = sqlSession2.getMapper(StudentMapper.class);
     
             System.out.println("studentMapper读取数据: " + studentMapper.getStudentById(1));
             System.out.println("studentMapper读取数据: " + studentMapper.getStudentById(1));
             System.out.println("studentMapper2更新了" + studentMapper2.updateStudentName("小岑",1) + "个学生的数据");
             System.out.println("studentMapper读取数据: " + studentMapper.getStudentById(1));
             System.out.println("studentMapper2读取数据: " + studentMapper2.getStudentById(1));
     }
     ```

   - ![img](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202202071327851.jpeg)

   - > `sqlSession2`更新了id为1的学生的姓名，从凯伦改为了小岑，但`session1`之后的查询中，id为1的学生的名字还是凯伦，出现了脏数据，也证明了之前的设想，**一级缓存只在数据库会话内部共享**。

##### 11.4.1.3 一级缓存工作流程

一级缓存执行的时序图，如下图所示。

<img src="http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202202071330372.png" alt="img" style="zoom:200%;" />



##### 11.4.1.4 一级缓存源码分析

接下来将对MyBatis查询相关的核心类和一级缓存的源码进行走读。这对后面二级缓存也有帮助。

- **SqlSession**： 对外提供了用户和数据库之间交互需要的所有方法，隐藏了底层的细节。默认实现类是`DefaultSqlSession`。<img src="http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202202071332135.jpeg" alt="img" style="zoom: 2%;" />

- **Executor**： `SqlSession`向用户提供操作数据库的方法，但和数据库操作有关的职责都会委托给Executor。<img src="http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202202071340639.jpeg" alt="img" style="zoom:2%;" />
  - 如下图所示，`Executor`有若干个实现类，为`Executor`赋予了不同的能力，大家可以根据类名，自行学习每个类的基本作用。<img src="http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202202071341844.jpeg" alt="img" style="zoom:2%;" />

- **Cache**： MyBatis中的`Cache`接口，提供了和缓存相关的最基本的操作，如下图所示：<img src="http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202202071347389.jpeg" alt="img" style="zoom:4%;" />
  - 有若干个实现类，使用装饰器模式互相组装，提供丰富的操控缓存的能力，部分实现类如下图所示：<img src="http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202202071348923.jpeg" alt="img" style="zoom:5%;" />

[**在一级缓存的源码分析中，主要学习`BaseExecutor`的内部实现。**](#3. BaseExecutor)

在一级缓存的介绍中提到对`Local Cache`的查询和写入是在`Executor`内部完成的。在阅读`BaseExecutor`的代码后发现`Local Cache`是`BaseExecutor`内部的一个成员变量，如下代码所示。

```java
public abstract class BaseExecutor implements Executor {
protected ConcurrentLinkedQueue<DeferredLoad> deferredLoads;
protected PerpetualCache localCache;
```

`BaseExecutor`成员变量之一的`PerpetualCache`，是对Cache接口最基本的实现，其实现非常简单，内部持有HashMap，对一级缓存的操作实则是对HashMap的操作。如下代码所示：

```java
public class PerpetualCache implements Cache {
  private String id;
  private Map<Object, Object> cache = new HashMap<Object, Object>();
```

------

​	在阅读相关核心类代码后，从源代码层面对一级缓存工作中涉及到的相关代码，出于篇幅的考虑，对源码做适当删减，读者朋友可以结合本文，后续进行更详细的学习。

为执行和数据库的交互，首先需要初始化`SqlSession`，通过`DefaultSqlSessionFactory`开启`SqlSession`：为执行和数据库的交互，首先需要初始化`SqlSession`，通过`DefaultSqlSessionFactory`开启`SqlSession`：

```java
// DefaultSqlSessionFactory.java
private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
    ............
    final Executor executor = configuration.newExecutor(tx, execType);     
    return new DefaultSqlSession(configuration, executor, autoCommit);
}

```

在初始化`SqlSesion`时，会使用`Configuration`类创建一个全新的`Executor`，作为`DefaultSqlSession`构造函数的参数，创建Executor代码如下所示：

```java
// Configuration.java
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
    // 尤其可以注意这里，如果二级缓存开关开启的话，是使用CahingExecutor装饰BaseExecutor的子类
    if (cacheEnabled) {
      executor = new CachingExecutor(executor);                      
    }
    executor = (Executor) interceptorChain.pluginAll(executor);
    return executor;
}
```

`SqlSession`创建完毕后，根据Statment的不同类型，会进入`SqlSession`的不同方法中，如果是`Select`语句的话，最后会执行到`SqlSession`的`selectList`，代码如下所示：

```java
// SqlSession.java
@Override
public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
      MappedStatement ms = configuration.getMappedStatement(statement);
      return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
}
```

`SqlSession`把具体的查询职责委托给了Executor。如果只开启了一级缓存的话，首先会进入`BaseExecutor`的`query`方法。代码如下所示：

```java
// BaseExecutor.java
@Override
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
    BoundSql boundSql = ms.getBoundSql(parameter);
    CacheKey key = createCacheKey(ms, parameter, rowBounds, boundSql);
    return query(ms, parameter, rowBounds, resultHandler, key, boundSql);
}
```

在上述代码中，会先根据传入的参数生成`CacheKey`，进入该方法查看**CacheKey是如何生成**的，代码如下所示：

```java
CacheKey cacheKey = new CacheKey();
cacheKey.update(ms.getId());
cacheKey.update(rowBounds.getOffset());
cacheKey.update(rowBounds.getLimit());
cacheKey.update(boundSql.getSql());
//后面是update了sql中带的参数
cacheKey.update(value);
```

在上述的代码中，将`MappedStatement`的Id、SQL的offset、SQL的limit、SQL本身以及SQL中的参数传入了CacheKey这个类，最终构成CacheKey。以下是这个类的内部结构：

```java
private static final int DEFAULT_MULTIPLYER = 37;
private static final int DEFAULT_HASHCODE = 17;

private int multiplier;
private int hashcode;
private long checksum;
private int count;
private List<Object> updateList;

public CacheKey() {
    this.hashcode = DEFAULT_HASHCODE;
    this.multiplier = DEFAULT_MULTIPLYER;
    this.count = 0;
    this.updateList = new ArrayList<Object>();
}
```

首先是成员变量和构造函数，有一个初始的`hachcode`和乘数，同时维护了一个内部的`updatelist`。在`CacheKey`的`update`方法中，会进行一个`hashcode`和`checksum`的计算，同时把传入的参数添加进`updatelist`中。如下代码所示：

```java
public void update(Object object) {
    int baseHashCode = object == null ? 1 : ArrayUtil.hashCode(object); 
    count++;
    checksum += baseHashCode;
    baseHashCode *= count;
    hashcode = multiplier * hashcode + baseHashCode;
    
    updateList.add(object);
}
```

同时重写了`CacheKey`的`equals`方法，代码如下所示：

```java
@Override
public boolean equals(Object object) {
    .............
    for (int i = 0; i < updateList.size(); i++) {
      Object thisObject = updateList.get(i);
      Object thatObject = cacheKey.updateList.get(i);
      if (!ArrayUtil.equals(thisObject, thatObject)) {
        return false;
      }
    }
    return true;
}
```

除去hashcode、checksum和count的比较外，只要updatelist中的元素一一对应相等，那么就可以认为是CacheKey相等。只要两条SQL的下列五个值相同，即可以认为是相同的SQL。

> `Statement Id + Offset + Limmit + Sql + Params `  = CacheKey相等

`BaseExecutor`的`query`方法继续往下走，代码如下所示：

```java
list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
if (list != null) {
    // 这个主要是处理存储过程用的。
    handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
} else {
    list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
}
```

如果查不到的话，就从数据库查，在`queryFromDatabase`中，会对`localcache`进行写入。

在`query`方法执行的最后，会判断一级缓存级别是否是`STATEMENT`级别，如果是的话，就清空缓存，这也就是`STATEMENT`级别的一级缓存无法共享`localCache`的原因。代码如下所示：

```java
if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
        clearLocalCache();
}
```

在源码分析的最后，我们确认一下，如果是`insert/delete/update`方法，缓存就会刷新的原因。

`SqlSession`的`insert`方法和`delete`方法，都会统一走`update`的流程，代码如下所示：

```java
@Override
public int insert(String statement, Object parameter) {
    return update(statement, parameter);
  }
   @Override
  public int delete(String statement) {
    return update(statement, null);
}
```

`update`方法也是委托给了`Executor`执行。`BaseExecutor`的执行方法如下所示：

```java
@Override
public int update(MappedStatement ms, Object parameter) throws SQLException {
    ErrorContext.instance().resource(ms.getResource()).activity("executing an update").object(ms.getId());
    if (closed) {
      throw new ExecutorException("Executor was closed.");
    }
    clearLocalCache();
    return doUpdate(ms, parameter);
}
```

每次执行`update`前都会清空`localCache`。

至此，一级缓存的工作流程讲解以及源码分析完毕。

> ### 总结
>
> 1. MyBatis一级缓存的生命周期和`SqlSession`一致。
> 2. MyBatis一级缓存内部设计简单，只是一个没有容量限定的`HashMap`，在缓存的功能性上有所欠缺。
> 3. MyBatis的一级缓存最大范围是`SqlSession`内部，有多个`SqlSession`或者分布式的环境下，数据库写操作会引起脏数据，建议设定缓存级别为`Statement`。

#### 11.4.2 二级缓存

> 以下摘自**美团技术团队-凯伦** [《聊聊MyBatis缓存机制》](https://tech.meituan.com/mybatis_cache.html)
>
> 在上文中提到的一级缓存中，其最大的共享范围就是一个`SqlSession`内部，如果多个`SqlSession`之间需要共享缓存，则需要使用到二级缓存。开启二级缓存后，会使用`CachingExecutor`装饰`Executor`，进入一级缓存的查询流程前，先在`CachingExecutor`进行二级缓存的查询，具体的工作流程如下所示。
>
> ![img](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202202071526526.png)
>
> 二级缓存开启后，**同一个namespace下的所有操作语句，都影响着同一个Cache**，即**二级缓存被多个SqlSession共享**，是一个全局的变量。
>
> 当开启缓存后，数据的查询执行的流程就是 二级缓存 -> 一级缓存 -> 数据库。

##### 11.4.2.1 二级缓存配置

要正确的使用二级缓存，需完成如下配置的。

1. 在MyBatis的配置文件中开启二级缓存。

   - ```xml
     <setting name="cacheEnabled" value="true"/>
     ```

2. 在MyBatis的映射XML中配置`cache`或者 `cache-ref `。

cache标签用于声明这个namespace使用二级缓存，并且可以自定义配置。

```xml
<cache/>   
```

- `type`：cache使用的类型，默认是`PerpetualCache`，这在一级缓存中提到过。
- `eviction`： 定义回收的策略，常见的有FIFO，LRU。
- `flushInterval`： 配置一定时间自动刷新缓存，单位是毫秒。
- `size`： 最多缓存对象的个数。
- `readOnly`： 是否只读，若配置可读写，则需要对应的实体类能够序列化。
- `blocking`： 若缓存中找不到对应的key，是否会一直blocking，直到有对应的数据进入缓存。

`cache-ref`代表引用别的命名空间的Cache配置，两个命名空间的操作使用的是同一个Cache。

```xml
<cache-ref namespace="mapper.StudentMapper"/>
```

##### 11.4.2.2 二级缓存实验总结

1. 测试二级缓存效果，不提交事务，`sqlSession1`查询完数据后，`sqlSession2`相同的查询是否会从缓存中获取数据。

   - ```java
     @Test
     public void testCacheWithoutCommitOrClose() throws Exception {
             SqlSession sqlSession1 = factory.openSession(true); 
             SqlSession sqlSession2 = factory.openSession(true); 
             
             StudentMapper studentMapper = sqlSession1.getMapper(StudentMapper.class);
             StudentMapper studentMapper2 = sqlSession2.getMapper(StudentMapper.class);
     
             System.out.println("studentMapper读取数据: " + studentMapper.getStudentById(1));
             System.out.println("studentMapper2读取数据: " + studentMapper2.getStudentById(1));
     }
     ```

   - <img src="http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202202071545105.jpeg" alt="img" style="zoom: 5%;" />

   - > 当`sqlsession`没有调用`commit()`方法时，二级缓存并没有起到作用。

2. 测试二级缓存效果，当提交事务时，`sqlSession1`查询完数据后，`sqlSession2`相同的查询是否会从缓存中获取数据。

   - ```java
     @Test
     public void testCacheWithCommitOrClose() throws Exception {
             SqlSession sqlSession1 = factory.openSession(true); 
             SqlSession sqlSession2 = factory.openSession(true); 
             
             StudentMapper studentMapper = sqlSession1.getMapper(StudentMapper.class);
             StudentMapper studentMapper2 = sqlSession2.getMapper(StudentMapper.class);
     
             System.out.println("studentMapper读取数据: " + studentMapper.getStudentById(1));
             sqlSession1.commit();
             System.out.println("studentMapper2读取数据: " + studentMapper2.getStudentById(1));
     }
     ```

   - <img src="http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202202071548304.jpeg" alt="img" style="zoom:9%;" />

   - > `sqlsession2`的查询，使用了缓存，缓存的命中率是0.5。

3. 测试`update`操作是否会刷新该`namespace`下的二级缓存。

   - ```java
     @Test
     public void testCacheWithUpdate() throws Exception {
             SqlSession sqlSession1 = factory.openSession(true); 
             SqlSession sqlSession2 = factory.openSession(true); 
             SqlSession sqlSession3 = factory.openSession(true); 
             
             StudentMapper studentMapper = sqlSession1.getMapper(StudentMapper.class);
             StudentMapper studentMapper2 = sqlSession2.getMapper(StudentMapper.class);
             StudentMapper studentMapper3 = sqlSession3.getMapper(StudentMapper.class);
             
             System.out.println("studentMapper读取数据: " + studentMapper.getStudentById(1));
             sqlSession1.commit();
             System.out.println("studentMapper2读取数据: " + studentMapper2.getStudentById(1));
             
             studentMapper3.updateStudentName("方方",1);
             sqlSession3.commit();
             System.out.println("studentMapper2读取数据: " + studentMapper2.getStudentById(1));
     }
     ```

   - <img src="http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202202071550656.jpeg" alt="img" style="zoom:5%;" />

   - > 在`sqlSession3`更新数据库，并提交事务后，`sqlsession2`的`StudentMapper namespace`下的查询走了数据库，没有走Cache。

4. 验证MyBatis的二级缓存不适应用于映射文件中存在多表查询的情况。

   通常我们会为每个单表创建单独的映射文件，由于MyBatis的二级缓存是基于`namespace`的，多表查询语句所在的`namspace`无法感应到其他`namespace`中的语句对多表查询中涉及的表进行的修改，引发脏数据问题。

   - ```java
     @Test
     public void testCacheWithDiffererntNamespace() throws Exception {
             SqlSession sqlSession1 = factory.openSession(true); 
             SqlSession sqlSession2 = factory.openSession(true); 
             SqlSession sqlSession3 = factory.openSession(true); 
         
             StudentMapper studentMapper = sqlSession1.getMapper(StudentMapper.class);
             StudentMapper studentMapper2 = sqlSession2.getMapper(StudentMapper.class);
             ClassMapper classMapper = sqlSession3.getMapper(ClassMapper.class);
             
             System.out.println("studentMapper读取数据: " + studentMapper.getStudentByIdWithClassInfo(1));
             sqlSession1.close();
             System.out.println("studentMapper2读取数据: " + studentMapper2.getStudentByIdWithClassInfo(1));
     
             classMapper.updateClassName("特色一班",1);
             sqlSession3.commit();
             System.out.println("studentMapper2读取数据: " + studentMapper2.getStudentByIdWithClassInfo(1));
     }
     // 在这个实验中，我们引入了两张新的表，一张class，一张classroom。class中保存了班级的id和班级名，classroom中保存了班级id和学生id。我们在`StudentMapper`中增加了一个查询方法`getStudentByIdWithClassInfo`，用于查询学生所在的班级，涉及到多表查询。在`ClassMapper`中添加了`updateClassName`，根据班级id更新班级名的操作。 
     ```

   - <img src="http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202202071552153.jpeg" alt="img" style="zoom:5%;" />

   - > 当`sqlsession1`的`studentmapper`查询数据后，二级缓存生效。保存在**StudentMapper的namespace下的cache中**。当`sqlSession3`的`classMapper`的`updateClassName`方法对class表进行更新时，`updateClassName`不属于`StudentMapper`的`namespace`，所以`StudentMapper`下的cache没有感应到变化，没有刷新缓存。当`StudentMapper`中同样的查询再次发起时，从缓存中读取了**脏数据**。

5. 为了解决实验4的问题呢，可以使用Cache ref，让`ClassMapper`引用`StudenMapper`命名空间，这样两个映射文件对应的SQL操作都使用的是同一块缓存了。

   - <img src="http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202202071556679.jpeg" alt="img" style="zoom:5%;" />

   - > 不过这样做的后果是，缓存的粒度变粗了，多个`Mapper namespace`下的所有操作都会对缓存使用造成影响。

##### 11.4.2.3 二级缓存源码分析

MyBatis二级缓存的工作流程和前文提到的一级缓存类似，只是在一级缓存处理前，用`CachingExecutor`装饰了`BaseExecutor`的子类，在委托具体职责给`delegate`之前，实现了二级缓存的查询和写入功能，具体类关系图如下图所示。<img src="http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202202071600388.jpeg" alt="img" style="zoom:5%;" />

[源码分析从`CachingExecutor`的`query`方法展开](#7.1.3 query)，源代码走读过程中涉及到的知识点较多，不能一一详细讲解，读者朋友可以自行查询相关资料来学习。

`CachingExecutor`的`query`方法，首先会从`MappedStatement`中获得在配置初始化时赋予的Cache。

```java
Cache cache = ms.getCache();
```

本质上是装饰器模式的使用，具体的装饰链是：

> SynchronizedCache -> LoggingCache -> SerializedCache -> LruCache -> PerpetualCache。

![img](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202202071608914.jpeg)

以下是具体这些Cache实现类的介绍，他们的组合为Cache赋予了不同的能力。

- `SynchronizedCache`：同步Cache，实现比较简单，直接使用synchronized修饰方法。
- `LoggingCache`：日志功能，装饰类，用于记录缓存的命中率，如果开启了DEBUG模式，则会输出命中率日志。
- `SerializedCache`：序列化功能，将值序列化后存到缓存中。该功能用于缓存返回一份实例的Copy，用于保存线程安全。
- `LruCache`：采用了Lru算法的Cache实现，移除最近最少使用的Key/Value。
- `PerpetualCache`： 作为为最基础的缓存类，底层实现比较简单，直接使用了HashMap。

然后是判断是否需要刷新缓存，代码如下所示：

```java
flushCacheIfRequired(ms);
```

在默认的设置中`SELECT`语句不会刷新缓存，`insert/update/delte`会刷新缓存。进入该方法。代码如下所示：

```java
private void flushCacheIfRequired(MappedStatement ms) {
    Cache cache = ms.getCache();
    if (cache != null && ms.isFlushCacheRequired()) {      
      tcm.clear(cache);
    }
}
```

MyBatis的`CachingExecutor`持有了`TransactionalCacheManager`，即上述代码中的tcm。

`TransactionalCacheManager`中持有了一个Map，代码如下所示：

```java
private Map<Cache, TransactionalCache> transactionalCaches = new HashMap<Cache, TransactionalCache>();
```

这个Map保存了Cache和用`TransactionalCache`包装后的Cache的映射关系。

`TransactionalCache`实现了Cache接口，`CachingExecutor`会默认使用他包装初始生成的Cache，作用是如果事务提交，对缓存的操作才会生效，如果事务回滚或者不提交事务，则不对缓存产生影响。

在`TransactionalCache`的clear，有以下两句。清空了需要在提交时加入缓存的列表，同时设定提交时清空缓存，代码如下所示：

```java
@Override
public void clear() {
	clearOnCommit = true;
	entriesToAddOnCommit.clear();
}
```

`CachingExecutor`继续往下走，`ensureNoOutParams`主要是用来处理存储过程的，暂时不用考虑。

```java
if (ms.isUseCache() && resultHandler == null) {
	ensureNoOutParams(ms, parameterObject, boundSql);
```

之后会尝试从tcm中获取缓存的列表。

```java
List<E> list = (List<E>) tcm.getObject(cache, key);
```

在`getObject`方法中，会把获取值的职责一路传递，最终到`PerpetualCache`。如果没有查到，会把key加入Miss集合，这个主要是为了统计命中率。

```java
Object object = delegate.getObject(key);
if (object == null) {
	entriesMissedInCache.add(key);
}
```

`CachingExecutor`继续往下走，如果查询到数据，则调用`tcm.putObject`方法，往缓存中放入值。

```java
if (list == null) {
	list = delegate.<E> query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
	tcm.putObject(cache, key, list); // issue #578 and #116
}
```

tcm的`put`方法也不是直接操作缓存，只是在把这次的数据和key放入待提交的Map中。

```java
@Override
public void putObject(Object key, Object object) {
    entriesToAddOnCommit.put(key, object);
}
```

从以上的代码分析中，我们可以明白，如果不调用`commit`方法的话，由于`TranscationalCache`的作用，并不会对二级缓存造成直接的影响。因此我们看看`Sqlsession`的`commit`方法中做了什么。代码如下所示：

```java
@Override
public void commit(boolean force) {
    try {
      executor.commit(isCommitOrRollbackRequired(force));
```

因为我们使用了CachingExecutor，首先会进入CachingExecutor实现的commit方法。

```java
@Override
public void commit(boolean required) throws SQLException {
    delegate.commit(required);
    tcm.commit();
}
```

会把具体commit的职责委托给包装的`Executor`。主要是看下`tcm.commit()`，tcm最终又会调用到`TrancationalCache`。

```java
public void commit() {
    if (clearOnCommit) {
      delegate.clear();
    }
    flushPendingEntries();
    reset();
}
```

看到这里的`clearOnCommit`就想起刚才`TrancationalCache`的`clear`方法设置的标志位，真正的清理Cache是放到这里来进行的。具体清理的职责委托给了包装的Cache类。之后进入`flushPendingEntries`方法。代码如下所示：

```java
private void flushPendingEntries() {
    for (Map.Entry<Object, Object> entry : entriesToAddOnCommit.entrySet()) {
      delegate.putObject(entry.getKey(), entry.getValue());
    }
    ................
}
```

在`flushPending`Entries中，将待提交的Map进行循环处理，委托给包装的Cache类，进行`putObject`的操作。

后续的查询操作会重复执行这套流程。如果是`insert|update|delete`的话，会统一进入`CachingExecutor`的`update`方法，其中调用了这个函数，代码如下所示：

```java
private void flushCacheIfRequired(MappedStatement ms) 
```

在二级缓存执行流程后就会进入一级缓存的执行流程，因此不再赘述。

------



> ### 总结
>
> 1. MyBatis的二级缓存相对于一级缓存来说，实现了`SqlSession`之间缓存数据的共享，同时粒度更加的细，能够到`namespace`级别，通过Cache接口实现类不同的组合，对Cache的可控性也更强。
> 2. MyBatis在多表查询时，极大可能会出现脏数据，有设计上的缺陷，安全使用二级缓存的条件比较苛刻。
> 3. 在分布式环境下，由于默认的MyBatis Cache实现都是基于本地的，分布式环境下必然会出现读取到脏数据，需要使用集中式缓存将MyBatis的Cache接口实现，有一定的开发成本，直接使用Redis、Memcached等分布式缓存可能成本更低，安全性也更高。

## *666. 彩蛋

困的来，写到大半夜 1 点多。回家偷懒了下，果然不行~~~

参考和推荐如下文章：

- **美团技术团队-凯伦** [《聊聊MyBatis缓存机制》](https://tech.meituan.com/mybatis_cache.html)
- **祖大俊** [《Mybatis3.3.x技术内幕（四）：五鼠闹东京之执行器Executor设计原本》](https://my.oschina.net/zudajun/blog/667214)
- 祖大俊 [《Mybatis3.3.x技术内幕（五）：Executor之doFlushStatements()》](https://my.oschina.net/zudajun/blog/668323)
- ~~祖大俊~~ [~~《Mybatis3.4.x技术内幕（二十二）：Mybatis一级、二级缓存原理分析》~~](https://my.oschina.net/zudajun/blog/747499)
- 徐郡明 [《MyBatis 技术内幕》](https://item.jd.com/12125531.html) 的 [「3.6 Executor](http://svip.iocoder.cn/MyBatis/executor-1/#) 小节