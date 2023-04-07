# 精尽 MyBatis 源码分析 —— 事务模块

## 1. 概述

本文，我们来分享 MyBatis 的事务模块，对应 `transaction` 包。如下图所示：

![img](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202201090248320.png)

在 [《精尽 MyBatis 源码解析 —— 项目结构一览》](http://svip.iocoder.cn/MyBatis/intro) 中，简单介绍了这个模块如下：

> MyBatis 对数据库中的事务进行了抽象，其自身提供了**相应的事务接口和简单实现**。
>
> 在很多场景中，MyBatis 会与 Spring 框架集成，并由 **Spring 框架管理事务**。

本文涉及的类如下图所示：![img](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202201090300792.png)

> - 一说到事务，人们可能又会想起`create`、`begin`、`commit`、`rollback`、`close`、`suspend`。可实际上，只有`commit`、`rollback`是实际存在的，剩下的create、begin、close、suspend都是虚幻的，是业务层或数据库底层应用语意，而非JDBC事务的真实命令。
>   - `create`（事务创建）：不存在。
>   - `begin`（事务开始）：姑且认为存在于DB的命令行中，比如Mysql的start transaction命令，以及其他数据库中的begin transaction命令。JDBC中不存在。
>   - `close`（事务关闭）：不存在。应用程序接口中的close()方法，是为了把connection放回数据库连接池中，供下一次使用，与事务毫无关系。
>   - `suspend`（事务挂起）：不存在。Spring中事务挂起的含义是，需要新事务时，将现有的connection1保存起来（它还有尚未提交的事务），然后创建connection2，connection2提交、回滚、关闭完毕后，再把connection1取出来，完成提交、回滚、关闭等动作，保存connection1的动作称之为事务挂起。在JDBC中，是根本不存在事务挂起的说法的，也不存在这样的接口方法。

下面，我们就一起来看看具体的源码实现。

## 2. Transaction

`org.apache.ibatis.transaction.Transaction` ，事务接口。代码如下：

```java
// Transaction.java

public interface Transaction {

    /**
     * 获得连接
     * 
     * Retrieve inner database connection
     *
     * @return DataBase connection
     * @throws SQLException
     */
    Connection getConnection() throws SQLException;

    /**
     * 事务提交
     * 
     * Commit inner database connection.
     *
     * @throws SQLException
     */
    void commit() throws SQLException;

    /**
     * 事务回滚
     * 
     * Rollback inner database connection.
     *
     * @throws SQLException
     */
    void rollback() throws SQLException;

    /**
     * 关闭连接
     * 
     * Close inner database connection.
     *
     * @throws SQLException
     */
    void close() throws SQLException;

    /**
     * 获得事务超时时间
     * 
     * Get transaction timeout if set
     *
     * @throws SQLException
     */
    Integer getTimeout() throws SQLException;

}
```

![image-20220202151142357](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202202021511402.png)

> 1. `JdbcTransaction`：单**独使用Mybatis时，默认的事务管理实现类**，就和它的名字一样，它就是我们常说的JDBC事务的极简封装，和编程使用mysql-connector-java-5.1.38-bin.jar事务驱动没啥差别。其极简封装，仅是**让connection支持连接池**而已。
> 2. `ManagedTransaction`：含义为托管事务，空壳事务管理器，皮包公司。仅是提醒用户，在其它环境中应用时，把事务托管给其它框架，比如托管给Spring，让Spring去管理事务。

- 连接相关
  - `#getConnection()` 方法，获得连接。
  - `#close()` 方法，关闭连接。
- 事务相关
  - `#commit()` 方法，事务提交。
  - `#rollback()` 方法，事务回滚。
  - `#getTimeout()` 方法，事务超时时间。实际上，目前这个方法都是空实现。

### 2.1 JdbcTransaction

`org.apache.ibatis.transaction.jdbc.JdbcTransaction` ，实现 Transaction 接口，基于 JDBC 的事务实现类。代码如下：

```java
// JdbcTransaction.java

public class JdbcTransaction implements Transaction {

    private static final Log log = LogFactory.getLog(JdbcTransaction.class);

    /**
     * Connection 对象
     */
    protected Connection connection;
    /**
     * DataSource 对象
     */
    protected DataSource dataSource;
    /**
     * 事务隔离级别
     */
    protected TransactionIsolationLevel level;
    /**
     * 是否自动提交
     */
    protected boolean autoCommit;

    public JdbcTransaction(DataSource ds, TransactionIsolationLevel desiredLevel, boolean desiredAutoCommit) {
        dataSource = ds;
        level = desiredLevel;
        autoCommit = desiredAutoCommit;
    }

    public JdbcTransaction(Connection connection) {
        this.connection = connection;
    }

    @Override
    public Connection getConnection() throws SQLException {
        // 连接为空，进行创建
        if (connection == null) {
            openConnection();
        }
        return connection;
    }

    @Override
    public void commit() throws SQLException {
        // 非自动提交，则执行提交事务
        if (connection != null && !connection.getAutoCommit()) {
            if (log.isDebugEnabled()) {
                log.debug("Committing JDBC Connection [" + connection + "]");
            }
            connection.commit();
        }
    }

    @Override
    public void rollback() throws SQLException {
        // 非自动提交。则回滚事务
        if (connection != null && !connection.getAutoCommit()) {
            if (log.isDebugEnabled()) {
                log.debug("Rolling back JDBC Connection [" + connection + "]");
            }
            connection.rollback();
        }
    }

    @Override
    public void close() throws SQLException {
        if (connection != null) {
            // 重置连接为自动提交
            resetAutoCommit();
            if (log.isDebugEnabled()) {
                log.debug("Closing JDBC Connection [" + connection + "]");
            }
            // 关闭连接
            connection.close();
        }
    }

    /**
     * 设置指定的 autoCommit 属性
     *
     * @param desiredAutoCommit 指定的 autoCommit 属性
     */
    protected void setDesiredAutoCommit(boolean desiredAutoCommit) {
        try {
            if (connection.getAutoCommit() != desiredAutoCommit) {
                if (log.isDebugEnabled()) {
                    log.debug("Setting autocommit to " + desiredAutoCommit + " on JDBC Connection [" + connection + "]");
                }
                connection.setAutoCommit(desiredAutoCommit);
            }
        } catch (SQLException e) {
            // Only a very poorly implemented driver would fail here,
            // and there's not much we can do about that.
            throw new TransactionException("Error configuring AutoCommit.  "
                    + "Your driver may not support getAutoCommit() or setAutoCommit(). "
                    + "Requested setting: " + desiredAutoCommit + ".  Cause: " + e, e);
        }
    }

    /**
     * 重置 autoCommit 属性
     */
    protected void resetAutoCommit() {
        try {
            if (!connection.getAutoCommit()) {
                // MyBatis does not call commit/rollback on a connection if just selects were performed.
                // Some databases start transactions with select statements
                // and they mandate a commit/rollback before closing the connection.
                // A workaround is setting the autocommit to true before closing the connection.
                // Sybase throws an exception here.
                if (log.isDebugEnabled()) {
                    log.debug("Resetting autocommit to true on JDBC Connection [" + connection + "]");
                }
                connection.setAutoCommit(true);
            }
        } catch (SQLException e) {
            if (log.isDebugEnabled()) {
                log.debug("Error resetting autocommit to true "
                        + "before closing the connection.  Cause: " + e);
            }
        }
    }

    /**
     * 获得 Connection 对象
     *
     * @throws SQLException 获得失败
     */
    protected void openConnection() throws SQLException {
        if (log.isDebugEnabled()) {
            log.debug("Opening JDBC Connection");
        }
        // 获得连接
        connection = dataSource.getConnection();
        // 设置隔离级别
        if (level != null) {
            connection.setTransactionIsolation(level.getLevel());
        }
        // 设置 autoCommit 属性
        setDesiredAutoCommit(autoCommit);
    }

    @Override
    public Integer getTimeout() throws SQLException {
        return null;
    }

}
```

- 简单，胖友瞄一瞄。

### 2.2 ManagedTransaction

`org.apache.ibatis.transaction.managed.ManagedTransaction` ，实现 Transaction 接口，基于容器管理的事务实现类。代码如下：

```java
// ManagedTransaction.java

public class ManagedTransaction implements Transaction {

    private static final Log log = LogFactory.getLog(ManagedTransaction.class);

    /**
     * Connection 对象
     */
    private Connection connection;
    /**
     * DataSource 对象
     */
    private DataSource dataSource;
    /**
     * 事务隔离级别
     */
    private TransactionIsolationLevel level;
    /**
     * 是否关闭连接
     *
     * 这个属性是和 {@link org.apache.ibatis.transaction.jdbc.JdbcTransaction} 不同的
     */
    private final boolean closeConnection;

    public ManagedTransaction(Connection connection, boolean closeConnection) {
        this.connection = connection;
        this.closeConnection = closeConnection;
    }

    public ManagedTransaction(DataSource ds, TransactionIsolationLevel level, boolean closeConnection) {
        this.dataSource = ds;
        this.level = level;
        this.closeConnection = closeConnection;
    }

    @Override
    public Connection getConnection() throws SQLException {
        // 连接为空，进行创建
        if (this.connection == null) {
            openConnection();
        }
        return this.connection;
    }

    @Override
    public void commit() throws SQLException {
        // Does nothing
    }

    @Override
    public void rollback() throws SQLException {
        // Does nothing
    }

    @Override
    public void close() throws SQLException {
        // 如果开启关闭连接功能，则关闭连接
        if (this.closeConnection && this.connection != null) {
            if (log.isDebugEnabled()) {
                log.debug("Closing JDBC Connection [" + this.connection + "]");
            }
            this.connection.close();
        }
    }

    protected void openConnection() throws SQLException {
        if (log.isDebugEnabled()) {
            log.debug("Opening JDBC Connection");
        }
        // 获得连接
        this.connection = this.dataSource.getConnection();
        // 设置隔离级别
        if (this.level != null) {
            this.connection.setTransactionIsolation(this.level.getLevel());
        }
    }

    @Override
    public Integer getTimeout() throws SQLException {
        return null;
    }

}
```

- 和 JdbcTransaction 相比，少了 `autoCommit` 属性，空实现 `#commit()` 和 `#rollback()` 方法。因此，事务的管理，交给了容器。
- 简单了解下就成，实际没用过这个。

### 2.3 SpringManagedTransaction

`org.mybatis.spring.transaction.SpringManagedTransaction` ，实现 Transaction 接口，基于 Spring 管理的事务实现类。实际真正在使用的，本文暂时不分享，感兴趣的胖友可以自己先愁一愁 [SpringManagedTransaction](https://github.com/eddumelendez/mybatis-spring/blob/master/src/main/java/org/mybatis/spring/transaction/SpringManagedTransaction.java) 。

```java

package org.mybatis.spring.transaction;
import org.apache.ibatis.transaction.Transaction;
import org.springframework.jdbc.datasource.DataSourceUtils;

/**
 * {@code SpringManagedTransaction} handles the lifecycle of a JDBC connection.
 * It retrieves a connection from Spring's transaction manager and returns it back to it
 * when it is no longer needed.
 * <p>
 * If Spring's transaction handling is active it will no-op all commit/rollback/close calls
 * assuming that the Spring transaction manager will do the job.
 * <p>
 * If it is not it will behave like {@code JdbcTransaction}.
 *
 * @author Hunter Presnall
 * @author Eduardo Macarron
 * 
 * @version $Id$
 */
public class SpringManagedTransaction implements Transaction {

  private static final Log LOGGER = LogFactory.getLog(SpringManagedTransaction.class);

  private final DataSource dataSource;

  private Connection connection;

  private boolean isConnectionTransactional;

  private boolean autoCommit;

  public SpringManagedTransaction(DataSource dataSource) {
    notNull(dataSource, "No DataSource specified");
    this.dataSource = dataSource;
  }

  /**
   * {@inheritDoc}
   */
  @Override
  public Connection getConnection() throws SQLException {
    if (this.connection == null) {
      openConnection();
    }
    return this.connection;
  }

  /**
   * Gets a connection from Spring transaction manager and discovers if this
   * {@code Transaction} should manage connection or let it to Spring.
   * <p>
   * It also reads autocommit setting because when using Spring Transaction MyBatis
   * thinks that autocommit is always false and will always call commit/rollback
   * so we need to no-op that calls.
   */
  private void openConnection() throws SQLException {
    this.connection = DataSourceUtils.getConnection(this.dataSource);
    this.autoCommit = this.connection.getAutoCommit();
    this.isConnectionTransactional = DataSourceUtils.isConnectionTransactional(this.connection, this.dataSource);

    if (LOGGER.isDebugEnabled()) {
      LOGGER.debug(
          "JDBC Connection ["
              + this.connection
              + "] will"
              + (this.isConnectionTransactional ? " " : " not ")
              + "be managed by Spring");
    }
  }

  /**
   * {@inheritDoc}
   */
  @Override
  public void commit() throws SQLException {
    if (this.connection != null && !this.isConnectionTransactional && !this.autoCommit) {
      if (LOGGER.isDebugEnabled()) {
        LOGGER.debug("Committing JDBC Connection [" + this.connection + "]");
      }
      this.connection.commit();
    }
  }

  /**
   * {@inheritDoc}
   */
  @Override
  public void rollback() throws SQLException {
    if (this.connection != null && !this.isConnectionTransactional && !this.autoCommit) {
      if (LOGGER.isDebugEnabled()) {
        LOGGER.debug("Rolling back JDBC Connection [" + this.connection + "]");
      }
      this.connection.rollback();
    }
  }

  /**
   * {@inheritDoc}
   */
  @Override
  public void close() throws SQLException {
    DataSourceUtils.releaseConnection(this.connection, this.dataSource);
  }

}
```



## 3. TransactionFactory

`org.apache.ibatis.transaction.TransactionFactory` ，Transaction 工厂接口。代码如下：

```java
// TransactionFactory.java

public interface TransactionFactory {

    /**
     * Sets transaction factory custom properties.
     *
     * 设置工厂的属性
     *
     * @param props 属性
     */
    void setProperties(Properties props);

    /**
     * Creates a {@link Transaction} out of an existing connection.
     *
     * 创建 Transaction 事务
     *
     * @param conn Existing database connection
     * @return Transaction
     * @since 3.1.0
     */
    Transaction newTransaction(Connection conn);

    /**
     * Creates a {@link Transaction} out of a datasource.
     *
     * 创建 Transaction 事务
     *
     * @param dataSource DataSource to take the connection from
     * @param level      Desired isolation level
     * @param autoCommit Desired autocommit
     * @return Transaction
     * @since 3.1.0
     */
    Transaction newTransaction(DataSource dataSource, TransactionIsolationLevel level, boolean autoCommit);

}
```

![img](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202202021513406.png)

> 顾名思义，一个生产`JdbcTransaction`实例，一个生产`ManagedTransaction`实例。两个毫无实际意义的工厂类，除了new之外，没有其他代码。
>
> ```xml
> <transactionManager type="JDBC" />
> ```
>
> ybatis-config.xml配置文件内，可配置事务管理类型。

### 3.1 JdbcTransactionFactory

`org.apache.ibatis.transaction.jdbc.JdbcTransactionFactory` ，实现 TransactionFactory 接口，JdbcTransaction 工厂实现类。代码如下：

```java
// JdbcTransactionFactory.java

public class JdbcTransactionFactory implements TransactionFactory {

    @Override
    public void setProperties(Properties props) {
    }

    @Override
    public Transaction newTransaction(Connection conn) {
        // 创建 JdbcTransaction 对象
        return new JdbcTransaction(conn);
    }

    @Override
    public Transaction newTransaction(DataSource ds, TransactionIsolationLevel level, boolean autoCommit) {
        // 创建 JdbcTransaction 对象
        return new JdbcTransaction(ds, level, autoCommit);
    }

}
```

- 胖友随便瞅一眼。

### 3.2 ManagedTransactionFactory

`org.apache.ibatis.transaction.managed.ManagedTransactionFactory` ，实现 TransactionFactory 接口，ManagedTransaction 工厂实现类。代码如下：

```java
// ManagedTransactionFactory.java

public class ManagedTransactionFactory implements TransactionFactory {

    /**
     * 是否关闭连接
     */
    private boolean closeConnection = true;

    @Override
    public void setProperties(Properties props) {
        // 获得是否关闭连接属性
        if (props != null) {
            String closeConnectionProperty = props.getProperty("closeConnection");
            if (closeConnectionProperty != null) {
                closeConnection = Boolean.valueOf(closeConnectionProperty);
            }
        }
    }

    @Override
    public Transaction newTransaction(Connection conn) {
        // 创建 ManagedTransaction 对象
        return new ManagedTransaction(conn, closeConnection);
    }

    @Override
    public Transaction newTransaction(DataSource ds, TransactionIsolationLevel level, boolean autoCommit) {
        // Silently ignores autocommit and isolation level, as managed transactions are entirely
        // controlled by an external manager.  It's silently ignored so that
        // code remains portable between managed and unmanaged configurations.
        // 创建 ManagedTransaction 对象
        return new ManagedTransaction(ds, level, closeConnection);
    }

}
```

### 3.3 SpringManagedTransactionFactory

`org.mybatis.spring.transaction.SpringManagedTransactionFactory` ，实现 TransactionFactory 接口，SpringManagedTransaction 工厂实现类。实际真正在使用的，本文暂时不分享，感兴趣的胖友可以自己先愁一愁 [SpringManagedTransactionFactory](https://github.com/eddumelendez/mybatis-spring/blob/c5834f93bd4a5879f86854fe188957787e56ef95/src/main/java/org/mybatis/spring/transaction/SpringManagedTransactionFactory.java) 。

```java
package org.mybatis.spring.transaction;

import java.sql.Connection;
import java.util.Properties;

import javax.sql.DataSource;

import org.apache.ibatis.session.TransactionIsolationLevel;
import org.apache.ibatis.transaction.Transaction;
import org.apache.ibatis.transaction.TransactionFactory;

/**
 * Creates a {@code SpringManagedTransaction}.
 *
 * @author Hunter Presnall
 * 
 * @version $Id$
 */
public class SpringManagedTransactionFactory implements TransactionFactory {

  /**
   * {@inheritDoc}
   */
  @Override
  public Transaction newTransaction(DataSource dataSource, TransactionIsolationLevel level, boolean autoCommit) {
    return new SpringManagedTransaction(dataSource);
  }

  /**
   * {@inheritDoc}
   */
  @Override
  public Transaction newTransaction(Connection conn) {
    throw new UnsupportedOperationException("New Spring transactions require a DataSource");
  }

  /**
   * {@inheritDoc}
   */
  @Override
  public void setProperties(Properties props) {
    // not needed in this version
  }

}
```



## 4.总结

### 4.1 Transaction的用法

> 无论是SqlSession，还是Executor，它们的事务方法，最终都指向了Transaction的事务方法，即都是由Transaction来完成事务提交、回滚的。

配一个简单的时序图。

![img](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202202021556512.png)

```java
public static void main(String[] args) {
		SqlSession sqlSession = MybatisSqlSessionFactory.openSession();
		try {
			StudentMapper studentMapper = sqlSession.getMapper(StudentMapper.class);
			
			Student student = new Student();
			student.setName("yy");
			student.setEmail("email@email.com");
			student.setDob(new Date());
			student.setPhone(new PhoneNumber("123-2568-8947"));
			
			studentMapper.insertStudent(student);
			sqlSession.commit();
		} catch (Exception e) {
			sqlSession.rollback();
		} finally {
			sqlSession.close();
		}
	}
```

> 注：`Executor`在执行`insertStudent(student)`方法时，与事务的提交、回滚、关闭毫无瓜葛（方法内部不会提交、回滚事务），需要像上面的代码一样，手动显示调用commit()、rollback()、close()等方法。

### 4.2 有关事务的几种特殊场景表现（重要）

#### 4.2.1 一个conn生命周期内，可以存在无数多个事务。

```java
	   // 执行了connection.setAutoCommit(false)，并返回
            SqlSession sqlSession = MybatisSqlSessionFactory.openSession();
		try {
			StudentMapper studentMapper = sqlSession.getMapper(StudentMapper.class);
			
			Student student = new Student();
			student.setName("yy");
			student.setEmail("email@email.com");
			student.setDob(new Date());
			student.setPhone(new PhoneNumber("123-2568-8947"));
			
			studentMapper.insertStudent(student);
			// 提交
			sqlSession.commit();
			
			studentMapper.insertStudent(student);
			// 多次提交
			sqlSession.commit();
		} catch (Exception e) {
		        // 回滚，只能回滚当前未提交的事务
			sqlSession.rollback();
		} finally {
			sqlSession.close();
		}
```

对于JDBC来说，`autoCommit=false`时，是自动开启事务的，执行commit()后，该事务结束。以上代码正常情况下，开启了2个事务，向数据库插入了2条数据。JDBC中不存在Hibernate中的session的概念，在JDBC中，insert了几次，数据库就会有几条记录，切勿混淆。而rollback()，只能回滚当前未提交的事务。

#### 4.2.2 autoCommit=false，没有执行commit()，仅执行close()，会发生什么？

```java
try {
    studentMapper.insertStudent(student);
} finally {
    sqlSession.close();
}
```

`insert`后，`close`之前，如果数据库的事务隔离级别是`read uncommitted`，那么，我们可以在数据库中查询到该条记录。

接着执行`sqlSession.close()`时，经过`SqlSession`的判断，决定执行`rollback()`操作，于是，事务回滚，数据库记录消失。

```java
// DefaultSqlSession.java
@Override
  public void close() {
    try {
      executor.close(isCommitOrRollbackRequired(false));
      dirty = false;
    } finally {
      ErrorContext.instance().reset();
    }
  }

// 事务是否回滚，依靠isCommitOrRollbackRequired(false)方法来判断。
 private boolean isCommitOrRollbackRequired(boolean force) {
    return (!autoCommit && dirty) || force;  // return (true && dirty) || false
  }

```

**最终是否回滚事务，只有`dirty`参数了。**

```java
@Override
  public int insert(String statement, Object parameter) {
    return update(statement, parameter);
  }

  @Override
  public int update(String statement, Object parameter) {
    try {
      dirty = true;
      MappedStatement ms = configuration.getMappedStatement(statement);
      return executor.update(ms, wrapCollection(parameter));
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error updating database.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
// 源码很明确，只要执行update操作，就设置dirty=true。insert、delete最终也是执行update操作。只有在执行完commit()、rollback()、close()等方法后，才会再次设置dirty=false。
```

> **因此，得出结论：autoCommit=false，但是没有手动commit，在sqlSession.close()时，Mybatis会将事务进行rollback()操作，然后才执行conn.close()关闭连接，当然数据最终也就没能持久化到数据库中了。**

#### 4.2.3 autoCommit=false，没有commit，也没有close，会发生什么？

```java
studentMapper.insertStudent(student);
```

干脆，就这一句话，即不commit，也不close。

> **结论：insert后，jvm结束前，如果事务隔离级别是read uncommitted，我们可以查到该条记录。jvm结束后，事务被rollback()，记录消失。通过断点debug方式，你可以看到效果。**

​	**这说明JDBC驱动实现，已经Kao虑到这样的特例情况，底层已经有相应的处理机制了。这也超出了我们的探究范围。**

​	**注：无参的openSession()方法，会自动设置autoCommit=false。**

## *666. 彩蛋

水文一篇，开心。

参考和推荐如下文章：

- 祖大俊 [《Mybatis3.3.x技术内幕（三）：Mybatis事务管理（将颠覆你心中目前对事务的理解）》](https://my.oschina.net/zudajun/blog/666764)
- 徐郡明 [《MyBatis 技术内幕》](https://item.jd.com/12125531.html) 的 [「2.7 Transaction」](http://svip.iocoder.cn/MyBatis/transaction-package/#) 小节