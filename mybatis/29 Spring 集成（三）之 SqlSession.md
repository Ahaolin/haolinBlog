# Spring 集成（三）之 SqlSession

## 1. 概述

本文我们就来看看，Spring 和 MyBatis 的 SqlSession 是如何集成。

## 2. SqlSessionTemplate

`org.mybatis.spring.SqlSessionTemplate` ，实现 SqlSession、DisposableBean 接口，SqlSession 操作模板实现类。实际上，代码实现和 `org.apache.ibatis.session.SqlSessionManager` 超级相似。或者说，这是 `mybatis-spring` 项目实现的 “SqlSessionManager” 类。

### 2.1 构造方法

```java
// SqlSessionTemplate.java

private final SqlSessionFactory sqlSessionFactory;

/**
 * 执行器类型
 */
private final ExecutorType executorType;
/**
 * SqlSession 代理对象
 */
private final SqlSession sqlSessionProxy;
/**
 * 异常转换器
 */
private final PersistenceExceptionTranslator exceptionTranslator;

public SqlSessionTemplate(SqlSessionFactory sqlSessionFactory) {
    this(sqlSessionFactory, sqlSessionFactory.getConfiguration().getDefaultExecutorType());
}

public SqlSessionTemplate(SqlSessionFactory sqlSessionFactory, ExecutorType executorType) {
    this(sqlSessionFactory, executorType,
            new MyBatisExceptionTranslator(
                    sqlSessionFactory.getConfiguration().getEnvironment().getDataSource(), true));
}

public SqlSessionTemplate(SqlSessionFactory sqlSessionFactory, ExecutorType executorType,
                          PersistenceExceptionTranslator exceptionTranslator) {

    notNull(sqlSessionFactory, "Property 'sqlSessionFactory' is required");
    notNull(executorType, "Property 'executorType' is required");

    this.sqlSessionFactory = sqlSessionFactory;
    this.executorType = executorType;
    this.exceptionTranslator = exceptionTranslator;
    // <1> 创建 sqlSessionProxy 对象
    this.sqlSessionProxy = (SqlSession) newProxyInstance(
            SqlSessionFactory.class.getClassLoader(),
            new Class[]{SqlSession.class},
            new SqlSessionInterceptor());
}

// ... 省略 setting 方法
```

- `executorType` 属性，执行器类型。后续，根据它来创建对应类型的执行器。
- `sqlSessionProxy` 属性，SqlSession 代理对象。在 `<1>` 处，进行创建 `sqlSessionProxy` 对象，使用的方法拦截器是 `SqlSessionInterceptor` 类。😈 是不是发现和 `SqlSessionManager `灰常像。
- `exceptionTranslator` 属性，异常转换器。

### 2.2 对 SqlSession 的实现方法

1. 如下是直接调用 `sqlSessionProxy` 对应的方法。代码如下：

   ```java
   // SqlSessionTemplate.java
   
   @Override
   public <T> T selectOne(String statement) {
       return this.sqlSessionProxy.selectOne(statement);
   }
   
   @Override
   public <T> T selectOne(String statement, Object parameter) {
       return this.sqlSessionProxy.selectOne(statement, parameter);
   }
   
   @Override
   public <K, V> Map<K, V> selectMap(String statement, String mapKey) {
       return this.sqlSessionProxy.selectMap(statement, mapKey);
   }
   
   @Override
   public <K, V> Map<K, V> selectMap(String statement, Object parameter, String mapKey) {
       return this.sqlSessionProxy.selectMap(statement, parameter, mapKey);
   }
   
   @Override
   public <K, V> Map<K, V> selectMap(String statement, Object parameter, String mapKey, RowBounds rowBounds) {
       return this.sqlSessionProxy.selectMap(statement, parameter, mapKey, rowBounds);
   }
   
   @Override
   public <T> Cursor<T> selectCursor(String statement) {
       return this.sqlSessionProxy.selectCursor(statement);
   }
   
   @Override
   public <T> Cursor<T> selectCursor(String statement, Object parameter) {
       return this.sqlSessionProxy.selectCursor(statement, parameter);
   }
   
   @Override
   public <T> Cursor<T> selectCursor(String statement, Object parameter, RowBounds rowBounds) {
       return this.sqlSessionProxy.selectCursor(statement, parameter, rowBounds);
   }
   
   @Override
   public <E> List<E> selectList(String statement) {
       return this.sqlSessionProxy.selectList(statement);
   }
   
   @Override
   public <E> List<E> selectList(String statement, Object parameter) {
       return this.sqlSessionProxy.selectList(statement, parameter);
   }
   
   @Override
   public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
       return this.sqlSessionProxy.selectList(statement, parameter, rowBounds);
   }
   
   @Override
   public void select(String statement, ResultHandler handler) {
       this.sqlSessionProxy.select(statement, handler);
   }
   
   @Override
   public void select(String statement, Object parameter, ResultHandler handler) {
       this.sqlSessionProxy.select(statement, parameter, handler);
   }
   
   @Override
   public void select(String statement, Object parameter, RowBounds rowBounds, ResultHandler handler) {
       this.sqlSessionProxy.select(statement, parameter, rowBounds, handler);
   }
   
   @Override
   public int insert(String statement) {
       return this.sqlSessionProxy.insert(statement);
   }
   
   @Override
   public int insert(String statement, Object parameter) {
       return this.sqlSessionProxy.insert(statement, parameter);
   }
   
   @Override
   public int update(String statement) {
       return this.sqlSessionProxy.update(statement);
   }
   
   @Override
   public int update(String statement, Object parameter) {
       return this.sqlSessionProxy.update(statement, parameter);
   }
   
   @Override
   public int delete(String statement) {
       return this.sqlSessionProxy.delete(statement);
   }
   
   @Override
   public int delete(String statement, Object parameter) {
       return this.sqlSessionProxy.delete(statement, parameter);
   }
   
   @Override
   public <T> T getMapper(Class<T> type) {
       return getConfiguration().getMapper(type, this);
   }
   
   @Override
   public void clearCache() {
       this.sqlSessionProxy.clearCache();
   }
   
   @Override
   public Configuration getConfiguration() {
       return this.sqlSessionFactory.getConfiguration();
   }
   
   @Override
   public Connection getConnection() {
       return this.sqlSessionProxy.getConnection();
   }
   
   @Override
   public List<BatchResult> flushStatements() {
       return this.sqlSessionProxy.flushStatements();
   }
   ```

2.  如下是不支持的方法，直接抛出 UnsupportedOperationException 异常。代码如下：

   ```java
   // SqlSessionTemplate.java
   
   //   ① throw new UnsupportedOperationException("Manual commit is not allowed over a Spring managed SqlSession");  
   @Override
   public void commit() { // 略 同① }
   
   @Override
   public void commit(boolean force)  { // 略 同① }
   
   @Override
   public void rollback()  { // 略 同① }
   
   @Override
   public void rollback(boolean force) { // 略 同① }
   
   @Override
   public void close()  { // 略 同① }
   ```

   - 和事务相关的方法，不允许**手动**调用。

### 2.3 destroy

```java
// SqlSessionTemplate.java

@Override
public void destroy() throws Exception {
    //This method forces spring disposer to avoid call of SqlSessionTemplate.close() which gives UnsupportedOperationException
}
```

### 2.4 SqlSessionInterceptor

`SqlSessionInterceptor `，是 `SqlSessionTemplate `的内部类，实现 InvocationHandler 接口，将 SqlSession 的操作，路由到 Spring 托管的事务管理器中。代码如下：

```java
// SqlSessionTemplate.java

/**
 * Proxy needed to route MyBatis method calls to the proper SqlSession got
 * from Spring's Transaction Manager
 * It also unwraps exceptions thrown by {@code Method#invoke(Object, Object...)} to
 * pass a {@code PersistenceException} to the {@code PersistenceExceptionTranslator}.
 */
private class SqlSessionInterceptor implements InvocationHandler {

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // <1> 获得 SqlSession 对象
        SqlSession sqlSession = getSqlSession(
                SqlSessionTemplate.this.sqlSessionFactory,
                SqlSessionTemplate.this.executorType,
                SqlSessionTemplate.this.exceptionTranslator);
        try {
            // 执行 SQL 操作
            Object result = method.invoke(sqlSession, args);
            // 如果非 Spring 托管的 SqlSession 对象，则提交事务
            if (!isSqlSessionTransactional(sqlSession, SqlSessionTemplate.this.sqlSessionFactory)) {
                // force commit even on non-dirty sessions because some databases require
                // a commit/rollback before calling close()
                sqlSession.commit(true);
            }
            // 返回结果
            return result;
        } catch (Throwable t) {
            // <4.1> 如果是 PersistenceException 异常，则进行转换
            Throwable unwrapped = unwrapThrowable(t);
            if (SqlSessionTemplate.this.exceptionTranslator != null && unwrapped instanceof PersistenceException) {
                // release the connection to avoid a deadlock if the translator is no loaded. See issue #22
                // <4.2> 根据情况，关闭 SqlSession 对象
                // 如果非 Spring 托管的 SqlSession 对象，则关闭 SqlSession 对象
                // 如果是 Spring 托管的 SqlSession 对象，则减少其 SqlSessionHolder 的计数
                closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
                // <4.3> 置空，避免下面 final 又做处理
                sqlSession = null;
                // <4.4> 进行转换
                Throwable translated = SqlSessionTemplate.this.exceptionTranslator.translateExceptionIfPossible((PersistenceException) unwrapped);
                if (translated != null) {
                    unwrapped = translated;
                }
            }
            // <4.5> 抛出异常
            throw unwrapped;
        } finally {
            // <5> 根据情况，关闭 SqlSession 对象
            // 如果非 Spring 托管的 SqlSession 对象，则关闭 SqlSession 对象
            // 如果是 Spring 托管的 SqlSession 对象，则减少其 SqlSessionHolder 的计数
            if (sqlSession != null) {
                closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
            }
        }
    }
}
```

- 类上的英文注释，胖友可以耐心看下。
- `<1>` 处，调用 `SqlSessionUtils#getSqlSession(SqlSessionFactory sessionFactory, ExecutorType executorType, PersistenceExceptionTranslator exceptionTranslator)` 方法，获得 SqlSession 对象。此处，和 Spring 事务托管的事务已经相关。详细的解析，见 [《精尽 MyBatis 源码解析 —— Spring 集成（四）之事务》](http://svip.iocoder.cn/MyBatis/Spring-Integration-4) 中。下面，和事务相关的，我们也统一放在该文中。
- `<2>` 处，调用 `Method#invoke(sqlSession, args)` 方法，**反射执行 SQL 操作**。
- `<3>` 处，调用 `SqlSessionUtils#isSqlSessionTransactional(SqlSession session, SqlSessionFactory sessionFactory)` 方法，判断是否为**非** Spring 托管的 SqlSession 对象，则调用 `SqlSession#commit(true)` 方法，提交事务。
- `<4.1>`处，如果是 PersistenceException 异常，则：
  - `<4.2>` 处，调用 `SqlSessionUtils#closeSqlSession(SqlSession session, SqlSessionFactory sessionFactory)` 方法，根据情况，关闭 SqlSession 对象。1）如果非 Spring 托管的 SqlSession 对象，则关闭 SqlSession 对象。2）如果是 Spring 托管的 SqlSession 对象，则减少其 SqlSessionHolder 的计数。😈 也就是说，Spring 托管事务的情况下，最终是在“外部”执行最终的事务处理。
  - `<4.3>` 处，置空 `sqlSession` 属性，避免下面 `final` 又做关闭处理。
  - `<4.4>` 处，调用 `MyBatisExceptionTranslator#translateExceptionIfPossible(RuntimeException e)` 方法，转换异常。
  - `<4.5>` 处，抛出异常。
- `<5>` 处，如果 `sqlSession` 非空，则调用 `SqlSessionUtils#closeSqlSession(SqlSession session, SqlSessionFactory sessionFactory)` 方法，根据情况，关闭 SqlSession 对象。

## 3. SqlSessionDaoSupport

`org.mybatis.spring.support.SqlSessionDaoSupport` ，继承 DaoSupport 抽象类，SqlSession 的 DaoSupport 抽象类。代码如下：

```java
// SqlSessionDaoSupport.java

public abstract class SqlSessionDaoSupport extends DaoSupport {

    /**
     * SqlSessionTemplate 对象
     */
    private SqlSessionTemplate sqlSessionTemplate;

    /**
     * Set MyBatis SqlSessionFactory to be used by this DAO.
     * Will automatically create SqlSessionTemplate for the given SqlSessionFactory.
     *
     * @param sqlSessionFactory a factory of SqlSession
     */
    public void setSqlSessionFactory(SqlSessionFactory sqlSessionFactory) {
        if (this.sqlSessionTemplate == null || sqlSessionFactory != this.sqlSessionTemplate.getSqlSessionFactory()) {
            this.sqlSessionTemplate = createSqlSessionTemplate(sqlSessionFactory); // 使用 sqlSessionFactory 属性，创建 sqlSessionTemplate 对象
        }
    }

    /**
     * Create a SqlSessionTemplate for the given SqlSessionFactory.
     * Only invoked if populating the DAO with a SqlSessionFactory reference!
     * <p>Can be overridden in subclasses to provide a SqlSessionTemplate instance
     * with different configuration, or a custom SqlSessionTemplate subclass.
     * @param sqlSessionFactory the MyBatis SqlSessionFactory to create a SqlSessionTemplate for
     * @return the new SqlSessionTemplate instance
     * @see #setSqlSessionFactory
     */
    @SuppressWarnings("WeakerAccess")
    protected SqlSessionTemplate createSqlSessionTemplate(SqlSessionFactory sqlSessionFactory) {
        return new SqlSessionTemplate(sqlSessionFactory);
    }

    /**
     * Return the MyBatis SqlSessionFactory used by this DAO.
     *
     * @return a factory of SqlSession
     */
    public final SqlSessionFactory getSqlSessionFactory() {
        return (this.sqlSessionTemplate != null ? this.sqlSessionTemplate.getSqlSessionFactory() : null);
    }


    /**
     * Set the SqlSessionTemplate for this DAO explicitly,
     * as an alternative to specifying a SqlSessionFactory.
     *
     * @param sqlSessionTemplate a template of SqlSession
     * @see #setSqlSessionFactory
     */
    public void setSqlSessionTemplate(SqlSessionTemplate sqlSessionTemplate) {
        this.sqlSessionTemplate = sqlSessionTemplate;
    }

    /**
     * Users should use this method to get a SqlSession to call its statement methods
     * This is SqlSession is managed by spring. Users should not commit/rollback/close it
     * because it will be automatically done.
     *
     * @return Spring managed thread safe SqlSession
     */
    public SqlSession getSqlSession() {
        return this.sqlSessionTemplate;
    }

    /**
     * Return the SqlSessionTemplate for this DAO,
     * pre-initialized with the SessionFactory or set explicitly.
     * <p><b>Note: The returned SqlSessionTemplate is a shared instance.</b>
     * You may introspect its configuration, but not modify the configuration
     * (other than from within an {@link #initDao} implementation).
     * Consider creating a custom SqlSessionTemplate instance via
     * {@code new SqlSessionTemplate(getSqlSessionFactory())}, in which case
     * you're allowed to customize the settings on the resulting instance.
     *
     * @return a template of SqlSession
     */
    public SqlSessionTemplate getSqlSessionTemplate() {
        return this.sqlSessionTemplate;
    }

    /**
     * {@inheritDoc}
     */
    @Override
    protected void checkDaoConfig() {
        notNull(this.sqlSessionTemplate, "Property 'sqlSessionFactory' or 'sqlSessionTemplate' are required");
    }

}
```

- 主要用途是，`sqlSessionTemplate` 属性的初始化，或者注入。
- <span id='go3'>具体示例</span>，可以看看 `org.mybatis.spring.sample.dao.UserDaoImpl` 类。

## 4. 异常相关

`mybatis-spring` 项目中，有两个异常相关的类：

- `org.mybatis.spring.MyBatisExceptionTranslator` ，MyBatis 自定义的异常转换器。
- `org.mybatis.spring.MyBatisSystemException` ，MyBatis 自定义的异常类。

代码比较简单，胖友自己去看。

## 666. 彩蛋

简单水更，哈哈哈哈。

> **示例：**
>
> - [SqlSessionDaoSupport ->`org.mybatis.spring.sample.dao.UserDaoImpl`](#go3) 

参考和推荐如下文章：

- 大新博客 [《Mybatis SqlSessionTemplate 源码解析》](https://www.cnblogs.com/daxin/p/3544188.html)