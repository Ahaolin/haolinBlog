# 精尽 MyBatis 源码解析 —— Spring 集成（四）之事务

## 1. 概述

本文我们就来看看，Spring 和 MyBatis 的**事务**是如何集成。需要胖友阅读的前置文章是：

- [《精尽 MyBatis 源码分析 —— 事务模块》](http://svip.iocoder.cn/MyBatis/transaction-package/)
- [《精尽 Spring 源码分析 —— Transaction 源码简单导读》](http://svip.iocoder.cn/Spring/transaction-simple-intro/) 需要胖友对 Spring Transaction 有源码级的了解，否则对该文章，会有点懵逼。

## 2. SpringManagedTransaction

`org.mybatis.spring.transaction.SpringManagedTransaction` ，实现 `org.apache.ibatis.transaction.Transaction` 接口，Spring 托管事务的 Transaction 实现类。

### 2.1 构造方法

```java
// SpringManagedTransaction.java

/**
 * DataSource 对象
 */
private final DataSource dataSource;
/**
 * Connection 对象
 */
private Connection connection;
/**
 * 当前连接是否处于事务中
 *
 * @see DataSourceUtils#isConnectionTransactional(Connection, DataSource)
 */
private boolean isConnectionTransactional;
/**
 * 是否自动提交
 */
private boolean autoCommit;

public SpringManagedTransaction(DataSource dataSource) {
    notNull(dataSource, "No DataSource specified");
    this.dataSource = dataSource;
}
```

### 2.2 getConnection

`#getConnection()` 方法，获得连接。代码如下：

```java
// SpringManagedTransaction.java

@Override
public Connection getConnection() throws SQLException {
    if (this.connection == null) {
        // 如果连接不存在，获得连接
        openConnection();
    }
    return this.connection;
}
```

- 如果 `connection` 为空，则调用 `#openConnection()` 方法，获得连接。代码如下：

  ```java
  // SpringManagedTransaction.java
  
  /**
   * Gets a connection from Spring transaction manager and discovers if this
   * {@code Transaction} should manage connection or let it to Spring.
   * <p>
   * It also reads autocommit setting because when using Spring Transaction MyBatis
   * thinks that autocommit is always false and will always call commit/rollback
   * so we need to no-op that calls.
   */
  private void openConnection() throws SQLException {
      // 获得连接
      this.connection = DataSourceUtils.getConnection(this.dataSource);
      this.autoCommit = this.connection.getAutoCommit();
      this.isConnectionTransactional = DataSourceUtils.isConnectionTransactional(this.connection, this.dataSource);
  
      LOGGER.debug(() ->
              "JDBC Connection ["
                      + this.connection
                      + "] will"
                      + (this.isConnectionTransactional ? " " : " not ")
                      + "be managed by Spring");
  }
  ```

  - 比较有趣的是，此处获取连接，不是通过 `DataSource#getConnection()` 方法，而是通过 `org.springframework.jdbc.datasource.DataSourceUtils#getConnection(DataSource dataSource)` 方法，获得 Connection 对象。而实际上，基于 Spring Transaction 体系，如果此处正在**事务中**时，已经有和当前线程绑定的 Connection 对象，就是存储在 ThreadLocal 中。

### 2.3 commit

`#commit()` 方法，提交事务。代码如下：

```java
// SpringManagedTransaction.java

@Override
public void commit() throws SQLException {
    if (this.connection != null && !this.isConnectionTransactional && !this.autoCommit) {
        LOGGER.debug(() -> "Committing JDBC Connection [" + this.connection + "]");
        this.connection.commit();
    }
}
```

### 2.4 rollback

`#rollback()` 方法，回滚事务。代码如下：

```java
// SpringManagedTransaction.java

@Override
public void rollback() throws SQLException {
    if (this.connection != null && !this.isConnectionTransactional && !this.autoCommit) {
        LOGGER.debug(() -> "Rolling back JDBC Connection [" + this.connection + "]");
        this.connection.rollback();
    }
}
```

### 2.5 close

`#close()` 方法，释放连接。代码如下：

```java
// SpringManagedTransaction.java

@Override
public void close() throws SQLException {
    DataSourceUtils.releaseConnection(this.connection, this.dataSource);
}
```

- 比较有趣的是，此处获取连接，不是通过 `Connection#close()` 方法，而是通过 `org.springframework.jdbc.datasource.DataSourceUtils#releaseConnection(Connection connection,DataSource dataSource)` 方法，“释放”连接。但是，具体会不会关闭连接，根据当前线程绑定的 Connection 对象，是不是传入的 `connection` 参数。

### 2.6 getTimeout

```java
// SpringManagedTransaction.java

@Override
public Integer getTimeout() throws SQLException {
    ConnectionHolder holder = (ConnectionHolder) TransactionSynchronizationManager.getResource(dataSource);
    if (holder != null && holder.hasTimeout()) {
        return holder.getTimeToLiveInSeconds();
    }
    return null;
}
```

## 3. SpringManagedTransactionFactory

`org.mybatis.spring.transaction.SpringManagedTransactionFactory` ，实现 TransactionFactory 接口，SpringManagedTransaction 的工厂实现类。代码如下：

```java
// SpringManagedTransactionFactory.java

public class SpringManagedTransactionFactory implements TransactionFactory {

    @Override
    public Transaction newTransaction(DataSource dataSource, TransactionIsolationLevel level, boolean autoCommit) {
        // 创建 SpringManagedTransaction 对象
        return new SpringManagedTransaction(dataSource);
    }

    @Override
    public Transaction newTransaction(Connection conn) {
        // 抛出异常，因为 Spring 事务，需要一个 DataSource 对象
        throw new UnsupportedOperationException("New Spring transactions require a DataSource");
    }

    @Override
    public void setProperties(Properties props) {
        // not needed in this version
    }

}
```

## 4. SqlSessionHolder

`org.mybatis.spring.SqlSessionHolder` ，继承 `org.springframework.transaction.support.ResourceHolderSupport` 抽象类，SqlSession 持有器，用于保存当前 SqlSession 对象，保存到 `org.springframework.transaction.support.TransactionSynchronizationManager` 中。代码如下：

```java
// SqlSessionHolder.java

/**
 * Used to keep current {@code SqlSession} in {@code TransactionSynchronizationManager}.
 *
 * The {@code SqlSessionFactory} that created that {@code SqlSession} is used as a key.
 * {@code ExecutorType} is also kept to be able to check if the user is trying to change it
 * during a TX (that is not allowed) and throw a Exception in that case.
 *
 * SqlSession 持有器，用于保存当前 SqlSession 对象，保存到 TransactionSynchronizationManager 中
 *
 * @author Hunter Presnall
 * @author Eduardo Macarron
 */
public final class SqlSessionHolder extends ResourceHolderSupport {

    /**
     * SqlSession 对象
     */
    private final SqlSession sqlSession;
    /**
     * 执行器类型
     */
    private final ExecutorType executorType;
    /**
     * PersistenceExceptionTranslator 对象
     */
    private final PersistenceExceptionTranslator exceptionTranslator;

    /**
     * Creates a new holder instance.
     *
     * @param sqlSession the {@code SqlSession} has to be hold.
     * @param executorType the {@code ExecutorType} has to be hold.
     * @param exceptionTranslator the {@code PersistenceExceptionTranslator} has to be hold.
     */
    public SqlSessionHolder(SqlSession sqlSession,
                            ExecutorType executorType,
                            PersistenceExceptionTranslator exceptionTranslator) {
        notNull(sqlSession, "SqlSession must not be null");
        notNull(executorType, "ExecutorType must not be null");

        this.sqlSession = sqlSession;
        this.executorType = executorType;
        this.exceptionTranslator = exceptionTranslator;
    }

    // ... 省略 getting 方法

}
```

- 当存储到 TransactionSynchronizationManager 中时，使用的 KEY 为创建该 SqlSession 对象的 SqlSessionFactory 对象。详细解析，见 `SqlSessionUtils#registerSessionHolder(...)` 方法。

## 5. SqlSessionUtils

`org.mybatis.spring.SqlSessionUtils` ，SqlSession 工具类。它负责处理 MyBatis SqlSession 的生命周期。它可以从 Spring TransactionSynchronizationManager 中，注册和获得对应的 SqlSession 对象。同时，它也支持当前不处于事务的情况下。

😈 当然，这个描述看起来，有点绕。所以，我们直接干起代码。

### 5.1 构造方法

```java
// SqlSessionUtils.java

private static final String NO_EXECUTOR_TYPE_SPECIFIED = "No ExecutorType specified";
private static final String NO_SQL_SESSION_FACTORY_SPECIFIED = "No SqlSessionFactory specified";
private static final String NO_SQL_SESSION_SPECIFIED = "No SqlSession specified";

/**
 * This class can't be instantiated, exposes static utility methods only.
 */
private SqlSessionUtils() {
    // do nothing
}
```

- 空的，直接跳过。

### 5.2 getSqlSession

`#getSqlSession(SqlSessionFactory sessionFactory, ...)` 方法，获得 SqlSession 对象。代码如下：

```java
// SqlSessionUtils.java

/**
 * Creates a new MyBatis {@code SqlSession} from the {@code SqlSessionFactory}
 * provided as a parameter and using its {@code DataSource} and {@code ExecutorType}
 *
 * @param sessionFactory a MyBatis {@code SqlSessionFactory} to create new sessions
 * @return a MyBatis {@code SqlSession}
 * @throws TransientDataAccessResourceException if a transaction is active and the
 *             {@code SqlSessionFactory} is not using a {@code SpringManagedTransactionFactory}
 */
public static SqlSession getSqlSession(SqlSessionFactory sessionFactory) {
    // 获得执行器类型
    ExecutorType executorType = sessionFactory.getConfiguration().getDefaultExecutorType();
    // 获得 SqlSession 对象
    return getSqlSession(sessionFactory, executorType, null);
}

/**
 * Gets an SqlSession from Spring Transaction Manager or creates a new one if needed.
 * Tries to get a SqlSession out of current transaction. If there is not any, it creates a new one.
 * Then, it synchronizes the SqlSession with the transaction if Spring TX is active and
 * <code>SpringManagedTransactionFactory</code> is configured as a transaction manager.
 *
 * @param sessionFactory a MyBatis {@code SqlSessionFactory} to create new sessions
 * @param executorType The executor type of the SqlSession to create
 * @param exceptionTranslator Optional. Translates SqlSession.commit() exceptions to Spring exceptions.
 * @return an SqlSession managed by Spring Transaction Manager
 * @throws TransientDataAccessResourceException if a transaction is active and the
 *             {@code SqlSessionFactory} is not using a {@code SpringManagedTransactionFactory}
 * @see SpringManagedTransactionFactory
 */
public static SqlSession getSqlSession(SqlSessionFactory sessionFactory, ExecutorType executorType, PersistenceExceptionTranslator exceptionTranslator) {
    notNull(sessionFactory, NO_SQL_SESSION_FACTORY_SPECIFIED);
    notNull(executorType, NO_EXECUTOR_TYPE_SPECIFIED);

    // <1> 获得 SqlSessionHolder 对象
    SqlSessionHolder holder = (SqlSessionHolder) TransactionSynchronizationManager.getResource(sessionFactory);

    // <2.1> 获得 SqlSession 对象
    SqlSession session = sessionHolder(executorType, holder);
    if (session != null) { // <2.2> 如果非空，直接返回
        return session;
    }

    LOGGER.debug(() -> "Creating a new SqlSession");
    // <3.1> 创建 SqlSession 对象
    session = sessionFactory.openSession(executorType);
    // <3.2> 注册到 TransactionSynchronizationManager 中
    registerSessionHolder(sessionFactory, executorType, exceptionTranslator, session);

    return session;
}
```

- 我们先看看每一步的作用，然后胖友在自己看下上面的英文注释，嘿嘿。

- `<1>` 处，调用 `TransactionSynchronizationManager#getResource(sessionFactory)` 方法，获得 SqlSessionHolder 对象。为什么可以获取到呢？答案在 `<3.2>` 中。关于 TransactionSynchronizationManager 类，如果不熟悉的胖友，真的真的真的，先去看懂 Spring Transaction 体系。

- `<2.1>` 处，调用 `#sessionHolder(ExecutorType executorType, SqlSessionHolder holder)` 方法，从 SqlSessionHolder 中，获得 SqlSession 对象。代码如下：

  ```java
  // SqlSessionUtils.java
  
  private static SqlSession sessionHolder(ExecutorType executorType, SqlSessionHolder holder) {
      SqlSession session = null;
      if (holder != null && holder.isSynchronizedWithTransaction()) {
          // 如果执行器类型发生了变更，抛出 TransientDataAccessResourceException 异常
          if (holder.getExecutorType() != executorType) {
              throw new TransientDataAccessResourceException("Cannot change the ExecutorType when there is an existing transaction");
          }
  
          // <1> 增加计数
          holder.requested();
  
          LOGGER.debug(() -> "Fetched SqlSession [" + holder.getSqlSession() + "] from current transaction");
          // <2> 获得 SqlSession 对象
          session = holder.getSqlSession();
      }
      return session;
  }
  ```

  - `<1>` 处，调用 `SqlSessionHolder#requested()` 方法，增加计数。注意，这个的计数，是用于关闭 SqlSession 时使用。详细解析，见 TODO 方法。
  - `<2>` 处，调用 `SqlSessionHolder#getSqlSession()` 方法，获得 SqlSession 对象。

- `<2.2>` 处，如果非空，直接返回。

- `<3.1>` 处，调用 `SqlSessionFactory#openSession(executorType)` 方法，创建 SqlSession 对象。其中，使用的执行器类型，由传入的 `executorType` 方法参数所决定。

- `<3.2>` 处，调用 `#registerSessionHolder(SqlSessionFactory sessionFactory, ExecutorType executorType, PersistenceExceptionTranslator exceptionTranslator, SqlSession session)` 方法，注册 SqlSession 对象，到 TransactionSynchronizationManager 中。代码如下：

  ```java
  // SqlSessionUtils.java
  
  /**
   * Register session holder if synchronization is active (i.e. a Spring TX is active).
   *
   * Note: The DataSource used by the Environment should be synchronized with the
   * transaction either through DataSourceTxMgr or another tx synchronization.
   * Further assume that if an exception is thrown, whatever started the transaction will
   * handle closing / rolling back the Connection associated with the SqlSession.
   *
   * @param sessionFactory sqlSessionFactory used for registration.
   * @param executorType executorType used for registration.
   * @param exceptionTranslator persistenceExceptionTranslator used for registration.
   * @param session sqlSession used for registration.
   */
  private static void registerSessionHolder(SqlSessionFactory sessionFactory, ExecutorType executorType,
                                            PersistenceExceptionTranslator exceptionTranslator, SqlSession session) {
      SqlSessionHolder holder;
      if (TransactionSynchronizationManager.isSynchronizationActive()) {
          Environment environment = sessionFactory.getConfiguration().getEnvironment();
  
          // <1> 如果使用 Spring 事务管理器
          if (environment.getTransactionFactory() instanceof SpringManagedTransactionFactory) {
              LOGGER.debug(() -> "Registering transaction synchronization for SqlSession [" + session + "]");
  
              // <1.1> 创建 SqlSessionHolder 对象
              holder = new SqlSessionHolder(session, executorType, exceptionTranslator);
              // <1.2> 绑定到 TransactionSynchronizationManager 中
              TransactionSynchronizationManager.bindResource(sessionFactory, holder);
              // <1.3> 创建 SqlSessionSynchronization 到 TransactionSynchronizationManager 中
              TransactionSynchronizationManager.registerSynchronization(new SqlSessionSynchronization(holder, sessionFactory));
              // <1.4> 设置同步
              holder.setSynchronizedWithTransaction(true);
              // <1.5> 增加计数
              holder.requested();
          // <2> 如果非 Spring 事务管理器，抛出 TransientDataAccessResourceException 异常
          } else {
              TransactionSynchronizationManager.getResource(environment.getDataSource());
              throw new TransientDataAccessResourceException(
                      "SqlSessionFactory must be using a SpringManagedTransactionFactory in order to use Spring transaction synchronization");
          }
      } else {
          LOGGER.debug(() -> "SqlSession [" + session + "] was not registered for synchronization because synchronization is not active");
      }
  }
  ```

  - `<2>` 处，如果非 Spring 事务管理器，抛出 TransientDataAccessResourceException 异常。
  - `<1>`处，如果使用 Spring 事务管理器(**`SpringManagedTransactionFactory`**)，则进行注册。
    - `<1.1>` 处，将 `sqlSession` 封装成 SqlSessionHolder 对象。
    - `<1.2>` 处，调用 `TransactionSynchronizationManager#bindResource(sessionFactory, holder)` 方法，绑定 `holder` 到 TransactionSynchronizationManager 中。**注意**，此时的 KEY 是 `sessionFactory` ，就创建 `sqlSession` 的 SqlSessionFactory 对象。
    - `<1.3>` 处，创建 SqlSessionSynchronization 到 TransactionSynchronizationManager 中。详细解析，见 [「6. SqlSessionSynchronization」](http://svip.iocoder.cn/MyBatis/Spring-Integration-4/#) 。
    - `<1.4>` 处，设置同步。
    - `<1.5>` 处，调用 `SqlSessionHolder#requested()` 方法，增加计数。

### 5.3 isSqlSessionTransactional

`#isSqlSessionTransactional(SqlSession session, SqlSessionFactory sessionFactory)` 方法，判断传入的 SqlSession 参数，是否在 Spring 事务中。代码如下：

```java
// SqlSessionUtils.java

/**
 * Returns if the {@code SqlSession} passed as an argument is being managed by Spring
 *
 * @param session a MyBatis SqlSession to check
 * @param sessionFactory the SqlSessionFactory which the SqlSession was built with
 * @return true if session is transactional, otherwise false
 */
public static boolean isSqlSessionTransactional(SqlSession session, SqlSessionFactory sessionFactory) {
    notNull(session, NO_SQL_SESSION_SPECIFIED);
    notNull(sessionFactory, NO_SQL_SESSION_FACTORY_SPECIFIED);

    // 从 TransactionSynchronizationManager 中，获得 SqlSessionHolder 对象
    SqlSessionHolder holder = (SqlSessionHolder) TransactionSynchronizationManager.getResource(sessionFactory);
    // 如果相等，说明在 Spring 托管的事务中
    return (holder != null) && (holder.getSqlSession() == session);
}
```

- 代码比较简单，直接看注释。

### 5.4 closeSqlSession

`#closeSqlSession(SqlSession session, SqlSessionFactory sessionFactory)` 方法，关闭 SqlSession 对象。代码如下：

```java
// SqlSessionUtils.java

/**
 * Checks if {@code SqlSession} passed as an argument is managed by Spring {@code TransactionSynchronizationManager}
 * If it is not, it closes it, otherwise it just updates the reference counter and
 * lets Spring call the close callback when the managed transaction ends
 *
 * @param session a target SqlSession
 * @param sessionFactory a factory of SqlSession
 */
public static void closeSqlSession(SqlSession session, SqlSessionFactory sessionFactory) {
    notNull(session, NO_SQL_SESSION_SPECIFIED);
    notNull(sessionFactory, NO_SQL_SESSION_FACTORY_SPECIFIED);

    // <1> 从 TransactionSynchronizationManager 中，获得 SqlSessionHolder 对象
    SqlSessionHolder holder = (SqlSessionHolder) TransactionSynchronizationManager.getResource(sessionFactory);

    // <2.1> 如果相等，说明在 Spring 托管的事务中，则释放 holder 计数
    if ((holder != null) && (holder.getSqlSession() == session)) {
        LOGGER.debug(() -> "Releasing transactional SqlSession [" + session + "]");
        holder.released();
    // <2.2> 如果不相等，说明不在 Spring 托管的事务中，直接关闭 SqlSession 对象
    } else {
        LOGGER.debug(() -> "Closing non transactional SqlSession [" + session + "]");
        session.close();
    }
}
```

- `<1>` 处，从 TransactionSynchronizationManager 中，获得 SqlSessionHolder 对象。
- `<2.1>` 处，如果相等，说明在 Spring 托管的事务中，则释放 `holder` 计数。那有什么用呢？具体原因，见 [「6. SqlSessionSynchronization」](http://svip.iocoder.cn/MyBatis/Spring-Integration-4/#) 。
- `<2.2>` 处，如果不相等，说明不在 Spring 托管的事务中，直接关闭 SqlSession 对象。

## 6. SqlSessionSynchronization

SqlSessionSynchronization ，是 SqlSessionUtils 的内部类，继承 TransactionSynchronizationAdapter 抽象类，SqlSession 的 同步器，基于 Spring Transaction 体系。

```java
/**
 * Callback for cleaning up resources. It cleans TransactionSynchronizationManager and
 * also commits and closes the {@code SqlSession}.
 * It assumes that {@code Connection} life cycle will be managed by
 * {@code DataSourceTransactionManager} or {@code JtaTransactionManager}
 */
```

### 6.1 构造方法

```java
// SqlSessionSynchronization.java

/**
 * SqlSessionHolder 对象
 */
private final SqlSessionHolder holder;
/**
 * SqlSessionFactory 对象
 */
private final SqlSessionFactory sessionFactory;
/**
 * 是否开启
 */
private boolean holderActive = true;

public SqlSessionSynchronization(SqlSessionHolder holder, SqlSessionFactory sessionFactory) {
    notNull(holder, "Parameter 'holder' must be not null");
    notNull(sessionFactory, "Parameter 'sessionFactory' must be not null");

    this.holder = holder;
    this.sessionFactory = sessionFactory;
}
```

### 6.2 getOrder

```java
// SqlSessionSynchronization.java

@Override
public int getOrder() {
    // order right before any Connection synchronization
    return DataSourceUtils.CONNECTION_SYNCHRONIZATION_ORDER - 1;
}
```

### 6.3 suspend

`#suspend()` 方法，当事务挂起时，取消当前线程的绑定的 SqlSessionHolder 对象。代码如下：

```java
// SqlSessionSynchronization.java

@Override
public void suspend() {
    if (this.holderActive) {
        LOGGER.debug(() -> "Transaction synchronization suspending SqlSession [" + this.holder.getSqlSession() + "]");
        TransactionSynchronizationManager.unbindResource(this.sessionFactory);
    }
}
```

### 6.4 resume

`#resume()` 方法，当事务恢复时，重新绑定当前线程的 SqlSessionHolder 对象。代码如下：

```java
// SqlSessionSynchronization.java

public void resume() {
    if (this.holderActive) {
        LOGGER.debug(() -> "Transaction synchronization resuming SqlSession [" + this.holder.getSqlSession() + "]");
        TransactionSynchronizationManager.bindResource(this.sessionFactory, this.holder);
    }
}
```

- 因为，当前 SqlSessionSynchronization 对象中，有 `holder` 对象，所以可以直接恢复。

### 6.5 beforeCommit

`#beforeCommit(boolean readOnly)` 方法，在事务提交之前，调用 `SqlSession#commit()` 方法，提交事务。虽然说，Spring 自身也会调用 `Connection#commit()` 方法，进行事务的提交。但是，`SqlSession#commit()` 方法中，不仅仅有事务的提交，还有提交批量操作，刷新本地缓存等等。代码如下：

```java
// SqlSessionSynchronization.java

@Override
public void beforeCommit(boolean readOnly) {
    // Connection commit or rollback will be handled by ConnectionSynchronization or
    // DataSourceTransactionManager.
    // But, do cleanup the SqlSession / Executor, including flushing BATCH statements so
    // they are actually executed.
    // SpringManagedTransaction will no-op the commit over the jdbc connection
    // TODO This updates 2nd level caches but the tx may be rolledback later on!
    if (TransactionSynchronizationManager.isActualTransactionActive()) {
        try {
            LOGGER.debug(() -> "Transaction synchronization committing SqlSession [" + this.holder.getSqlSession() + "]");
            // 提交事务
            this.holder.getSqlSession().commit();
        } catch (PersistenceException p) {
            // 如果发生异常，则进行转换，并抛出异常
            if (this.holder.getPersistenceExceptionTranslator() != null) {
                DataAccessException translated = this.holder
                        .getPersistenceExceptionTranslator()
                        .translateExceptionIfPossible(p);
                throw translated;
            }
            throw p;
        }
    }
}
```

- 耐心的看看英文注释，更有助于理解该方法。

### 6.6 beforeCompletion

> 老艿艿：TransactionSynchronization 的事务提交的执行顺序是：beforeCommit => beforeCompletion => 提交操作 => afterCompletion => afterCommit 。

`#beforeCompletion()` 方法，提交事务完成之前，关闭 SqlSession 对象。代码如下：

```java
// SqlSessionSynchronization.java

@Override
public void beforeCompletion() {
    // Issue #18 Close SqlSession and deregister it now
    // because afterCompletion may be called from a different thread
    if (!this.holder.isOpen()) {
        LOGGER.debug(() -> "Transaction synchronization deregistering SqlSession [" + this.holder.getSqlSession() + "]");
        // 取消当前线程的绑定的 SqlSessionHolder 对象
        TransactionSynchronizationManager.unbindResource(sessionFactory);
        // 标记无效
        this.holderActive = false;
        LOGGER.debug(() -> "Transaction synchronization closing SqlSession [" + this.holder.getSqlSession() + "]");
        // 关闭 SqlSession 对象
        this.holder.getSqlSession().close();
    }
}
```

- 因为，beforeCompletion 方法是在 beforeCommit 之后执行，并且在 beforeCommit 已经提交了事务，所以此处可以放心关闭 SqlSession 对象了。

- 要执行关闭操作之前，需要先调用 `SqlSessionHolder#isOpen()` 方法来判断，是否处于开启状态。代码如下：

  ```java
  // ResourceHolderSupport.java
  
  public boolean isOpen() {
  	return (this.referenceCount > 0);
  }
  ```

  - 这就是，我们前面看到的各种计数增减的作用。

### 6.7 afterCompletion

`#afterCompletion()` 方法，解决可能出现的**跨线程**的情况，简单理解下就好。代码如下：

```java
// ResourceHolderSupport.java

@Override
public void afterCompletion(int status) {
    if (this.holderActive) { // 处于有效状态
        // afterCompletion may have been called from a different thread
        // so avoid failing if there is nothing in this one
        LOGGER.debug(() -> "Transaction synchronization deregistering SqlSession [" + this.holder.getSqlSession() + "]");
        // 取消当前线程的绑定的 SqlSessionHolder 对象
        TransactionSynchronizationManager.unbindResourceIfPossible(sessionFactory);
        // 标记无效
        this.holderActive = false;
        LOGGER.debug(() -> "Transaction synchronization closing SqlSession [" + this.holder.getSqlSession() + "]");
        // 关闭 SqlSession 对象
        this.holder.getSqlSession().close();
    }
    this.holder.reset();
}
```

- 😈 貌似，官方没对这块做单元测试。

## 666. 彩蛋

越写越清晰，哈哈哈哈。

参考和推荐如下文章：

- fifadxj [《mybatis-spring事务处理机制分析》](https://my.oschina.net/fifadxj/blog/785621) 提供了序列图