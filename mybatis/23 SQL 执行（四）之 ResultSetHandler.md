# 精尽 MyBatis 源码分析 —— SQL 执行（四）之 ResultSetHandler

# 1. 概述

本文，我们来分享 SQL 执行的第四部分，SQL 执行后，响应的结果集 ResultSet 的处理，涉及 `executor/resultset`、`executor/result`、`cursor` 包。整体类图如下：[![类图](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202201292241553.png)](http://static.iocoder.cn/images/MyBatis/2020_03_09/01.png)类图

- 核心类是 ResultSetHandler 接口及其实现类 DefaultResultSetHandler 。在它的代码逻辑中，会调用类图中的其它类，实现将查询结果的 ResultSet ，转换成映射的对应结果。

# 2. ResultSetWrapper

> 老艿艿：在看具体的 `DefaultResultSetHandler` 的实现代码之前，我们先看看 ResultSetWrapper 的代码。因为 DefaultResultSetHandler 对 ResultSetWrapper 的调用比较多，避免混着解析。

`org.apache.ibatis.executor.resultset.ResultSetWrapper` ，`java.sql.ResultSet` 的 包装器，可以理解成 **`ResultSet` 的工具类**，**提供给 `DefaultResultSetHandler `使用**。

## 2.1 构造方法

```java
// ResultSetWrapper.java

/**
 * ResultSet 对象
 */
private final ResultSet resultSet;
private final TypeHandlerRegistry typeHandlerRegistry;
/**
 * 字段的名字的数组
 */
private final List<String> columnNames = new ArrayList<>();
/**
 * 字段的 Java Type 的数组
 */
private final List<String> classNames = new ArrayList<>();
/**
 * 字段的 JdbcType 的数组
 */
private final List<JdbcType> jdbcTypes = new ArrayList<>();
private final Map<String, Map<Class<?>, TypeHandler<?>>> typeHandlerMap = new HashMap<>();
private final Map<String, List<String>> mappedColumnNamesMap = new HashMap<>();
private final Map<String, List<String>> unMappedColumnNamesMap = new HashMap<>();

public ResultSetWrapper(ResultSet rs, Configuration configuration) throws SQLException {
    this.typeHandlerRegistry = configuration.getTypeHandlerRegistry();
    this.resultSet = rs;
    // <1> 遍历 ResultSetMetaData 的字段们，解析出 columnNames、jdbcTypes、classNames 属性
    final ResultSetMetaData metaData = rs.getMetaData();
    final int columnCount = metaData.getColumnCount();
    for (int i = 1; i <= columnCount; i++) {
        columnNames.add(configuration.isUseColumnLabel() ? metaData.getColumnLabel(i) : metaData.getColumnName(i));
        jdbcTypes.add(JdbcType.forCode(metaData.getColumnType(i)));
        classNames.add(metaData.getColumnClassName(i));
    }
}
```

- `resultSet` 属性，被包装的 ResultSet 对象。
- `columnNames`、`classNames`、`jdbcTypes` 属性，在 `<1>` 处，通过遍历 ResultSetMetaData 的字段们，从而解析出来。

## 2.2 getTypeHandler

```java
// ResultSetWrapper.java

/**
 * TypeHandler 的映射
 *
 * KEY1：字段的名字
 * KEY2：Java 属性类型
 */
private final Map<String, Map<Class<?>, TypeHandler<?>>> typeHandlerMap = new HashMap<>();

/**
 * Gets the type handler to use when reading the result set.
 * Tries to get from the TypeHandlerRegistry by searching for the property type.
 * If not found it gets the column JDBC type and tries to get a handler for it.
 *
 * 获得指定字段名的指定 JavaType 类型的 TypeHandler 对象
 *
 * @param propertyType JavaType
 * @param columnName 执行字段
 * @return TypeHandler 对象
 */
public TypeHandler<?> getTypeHandler(Class<?> propertyType, String columnName) {
    TypeHandler<?> handler = null;
    // <1> 先从缓存的 typeHandlerMap 中，获得指定字段名的指定 JavaType 类型的 TypeHandler 对象
    Map<Class<?>, TypeHandler<?>> columnHandlers = typeHandlerMap.get(columnName);
    if (columnHandlers == null) {
        columnHandlers = new HashMap<>();
        typeHandlerMap.put(columnName, columnHandlers);
    } else {
        handler = columnHandlers.get(propertyType);
    }
    // <2> 如果获取不到，则进行查找
    if (handler == null) {
        // <2> 获得 JdbcType 类型
        JdbcType jdbcType = getJdbcType(columnName);
        // <2> 获得 TypeHandler 对象
        handler = typeHandlerRegistry.getTypeHandler(propertyType, jdbcType);
        // Replicate logic of UnknownTypeHandler#resolveTypeHandler
        // See issue #59 comment 10
        // <3> 如果获取不到，则再次进行查找
        if (handler == null || handler instanceof UnknownTypeHandler) {
            // <3> 使用 classNames 中的类型，进行继续查找 TypeHandler 对象
            final int index = columnNames.indexOf(columnName);
            final Class<?> javaType = resolveClass(classNames.get(index));
            if (javaType != null && jdbcType != null) {
                handler = typeHandlerRegistry.getTypeHandler(javaType, jdbcType);
            } else if (javaType != null) {
                handler = typeHandlerRegistry.getTypeHandler(javaType);
            } else if (jdbcType != null) {
                handler = typeHandlerRegistry.getTypeHandler(jdbcType);
            }
        }
        // <4> 如果获取不到，则使用 ObjectTypeHandler 对象
        if (handler == null || handler instanceof UnknownTypeHandler) {
            handler = new ObjectTypeHandler();
        }
        // <5> 缓存到 typeHandlerMap 中
        columnHandlers.put(propertyType, handler);
    }
    return handler;
}
```

- `<1>` 处，先从缓存的 `typeHandlerMap` 中，获得指定字段名的指定 JavaType 类型的 TypeHandler 对象。

- `<2>` 处，如果获取不到，则基于 `propertyType` + `jdbcType` 进行查找。其中，`#getJdbcType(String columnName)` 方法，获得 JdbcType 类型。代码如下：

  ```java
  // ResultSetWrapper.java
  
  public JdbcType getJdbcType(String columnName) {
      for (int i = 0; i < columnNames.size(); i++) {
          if (columnNames.get(i).equalsIgnoreCase(columnName)) {
              return jdbcTypes.get(i);
          }
      }
      return null;
  }
  ```

  - 通过 `columnNames` 索引到位置 `i` ，从而到 `jdbcTypes` 中获得 JdbcType 类型。

- `<3>` 处，如果获取不到，则基于 `javaType` + `jdbcType` 进行查找。其中，`javaType` 使用 `classNames` 中的类型。而 `#resolveClass(String className)` 方法，获得对应的类。代码如下：

  ```java
  // ResultSetWrapper.java
  
  private Class<?> resolveClass(String className) {
      try {
          // #699 className could be null
          if (className != null) {
              return Resources.classForName(className);
          }
      } catch (ClassNotFoundException e) {
          // ignore
      }
      return null;
  }
  ```

- `<4>` 处，如果获取不到，则使用 ObjectTypeHandler 对象。

- `<5>` 处，缓存 TypeHandler 对象，到 `typeHandlerMap` 中。

## 2.3 loadMappedAndUnmappedColumnNames

`#loadMappedAndUnmappedColumnNames(ResultMap resultMap, String columnPrefix)` 方法，初始化**有 mapped** 和**无 mapped**的字段的名字数组。代码如下：

```java
// ResultSetWrapper.java

/**
 * 有 mapped 的字段的名字的映射
 *
 * KEY：{@link #getMapKey(ResultMap, String)}
 * VALUE：字段的名字的数组
 */
private final Map<String, List<String>> mappedColumnNamesMap = new HashMap<>();
/**
 * 无 mapped 的字段的名字的映射
 *
 * 和 {@link #mappedColumnNamesMap} 相反
 */
private final Map<String, List<String>> unMappedColumnNamesMap = new HashMap<>();

private void loadMappedAndUnmappedColumnNames(ResultMap resultMap, String columnPrefix) throws SQLException {
    List<String> mappedColumnNames = new ArrayList<>();
    List<String> unmappedColumnNames = new ArrayList<>();
    // <1> 将 columnPrefix 转换成大写，并拼接到 resultMap.mappedColumns 属性上
    final String upperColumnPrefix = columnPrefix == null ? null : columnPrefix.toUpperCase(Locale.ENGLISH);
    final Set<String> mappedColumns = prependPrefixes(resultMap.getMappedColumns(), upperColumnPrefix);
    // <2> 遍历 columnNames 数组，根据是否在 mappedColumns 中，分别添加到 mappedColumnNames 和 unmappedColumnNames 中
    for (String columnName : columnNames) {
        final String upperColumnName = columnName.toUpperCase(Locale.ENGLISH);
        if (mappedColumns.contains(upperColumnName)) {
            mappedColumnNames.add(upperColumnName);
        } else {
            unmappedColumnNames.add(columnName);
        }
    }
    // <3> 将 mappedColumnNames 和 unmappedColumnNames 结果，添加到 mappedColumnNamesMap 和 unMappedColumnNamesMap 中
    mappedColumnNamesMap.put(getMapKey(resultMap, columnPrefix), mappedColumnNames);
    unMappedColumnNamesMap.put(getMapKey(resultMap, columnPrefix), unmappedColumnNames);
}
```

- `<1>` 处，将 `columnPrefix` 转换成大写，后调用 `#prependPrefixes(Set<String> columnNames, String prefix)` 方法，拼接到 `resultMap.mappedColumns` 属性上。代码如下：

  ```java
  // ResultSetWrapper.java
  
  private Set<String> prependPrefixes(Set<String> columnNames, String prefix) {
      // 直接返回 columnNames ，如果符合如下任一情况
      if (columnNames == null || columnNames.isEmpty() || prefix == null || prefix.length() == 0) {
          return columnNames;
      }
      // 拼接前缀 prefix ，然后返回
      final Set<String> prefixed = new HashSet<>();
      for (String columnName : columnNames) {
          prefixed.add(prefix + columnName);
      }
      return prefixed;
  }
  ```

  - 当然，可能有胖友，跟我会懵逼，可能已经忘记什么是 `resultMap.mappedColumns` 。我们来举个示例：

    ```xml
    <resultMap id="B" type="Object">
        <result property="year" column="year"/>
    </resultMap>
    
    <select id="testResultMap" parameterType="Integer" resultMap="A">
        SELECT * FROM subject
    </select>
    ```

    - 此处的 `column="year"` ，就会被添加到 `resultMap.mappedColumns` 属性上。

- `<2>` 处，遍历 `columnNames` 数组，根据是否在 `mappedColumns` 中，分别添加到 `mappedColumnNames` 和 `unmappedColumnNames` 中。

- `<3>` 处，将 `mappedColumnNames` 和 `unmappedColumnNames` 结果，添加到 `mappedColumnNamesMap` 和 `unMappedColumnNamesMap` 中。其中，`#getMapKey(ResultMap resultMap, String columnPrefix)` 方法，获得缓存的 KEY 。代码如下：

  ```java
  // ResultSetWrapper.java
  
  private String getMapKey(ResultMap resultMap, String columnPrefix) {
      return resultMap.getId() + ":" + columnPrefix;
  }
  ```

下面，我们看个类似的，会调用该方法的方法：

- `#getMappedColumnNames(ResultMap resultMap, String columnPrefix)` 方法，获得**有** mapped 的字段的名字的数组。代码如下：

  ```java
  // ResultSetWrapper.java
  
  public List<String> getMappedColumnNames(ResultMap resultMap, String columnPrefix) throws SQLException {
      // 获得对应的 mapped 数组
      List<String> mappedColumnNames = mappedColumnNamesMap.get(getMapKey(resultMap, columnPrefix));
      if (mappedColumnNames == null) {
          // 初始化
          loadMappedAndUnmappedColumnNames(resultMap, columnPrefix);
          // 重新获得对应的 mapped 数组
          mappedColumnNames = mappedColumnNamesMap.get(getMapKey(resultMap, columnPrefix));
      }
      return mappedColumnNames;
  }
  ```

- `#getUnmappedColumnNames(ResultMap resultMap, String columnPrefix)` 方法，获得**无** mapped 的字段的名字的数组。代码如下：

  ```java
  // ResultSetWrapper.java
  
  public List<String> getUnmappedColumnNames(ResultMap resultMap, String columnPrefix) throws SQLException {
      // 获得对应的 unMapped 数组
      List<String> unMappedColumnNames = unMappedColumnNamesMap.get(getMapKey(resultMap, columnPrefix));
      if (unMappedColumnNames == null) {
          // 初始化
          loadMappedAndUnmappedColumnNames(resultMap, columnPrefix);
          // 重新获得对应的 unMapped 数组
          unMappedColumnNames = unMappedColumnNamesMap.get(getMapKey(resultMap, columnPrefix));
      }
      return unMappedColumnNames;
  }
  ```

😈 具体这两个方法什么用途呢？待到我们在 `DefaultResultSetHandler `类里来看。

# 3. ResultSetHandler

`org.apache.ibatis.executor.resultset.ResultSetHandler` ，`java.sql.ResultSet` 处理器接口。代码如下：

```java
// ResultSetHandler.java

public interface ResultSetHandler {

    /**
     * 处理 {@link java.sql.ResultSet} 成映射的对应的结果
     *
     * @param stmt Statement 对象
     * @param <E> 泛型
     * @return 结果数组
     */
    <E> List<E> handleResultSets(Statement stmt) throws SQLException;

    /**
     * 处理 {@link java.sql.ResultSet} 成 Cursor 对象
     *
     * @param stmt Statement 对象
     * @param <E> 泛型
     * @return Cursor 对象
     */
    <E> Cursor<E> handleCursorResultSets(Statement stmt) throws SQLException;

    // 暂时忽略，和存储过程相关
    void handleOutputParameters(CallableStatement cs) throws SQLException;

}
```

## 3.1 DefaultResultSetHandler

> 老艿艿：保持冷静，DefaultResultSetHandler 有小 1000 行的代码。

`org.apache.ibatis.executor.resultset.DefaultResultSetHandler` ，实现 ResultSetHandler 接口，默认的 ResultSetHandler 实现类。

### 3.1.1 构造方法

```java
// DefaultResultSetHandler.java

private static final Object DEFERED = new Object();

private final Executor executor;
private final Configuration configuration;
private final MappedStatement mappedStatement;
private final RowBounds rowBounds;
private final ParameterHandler parameterHandler;
/**
 * 用户指定的用于处理结果的处理器。
 *
 * 一般情况下，不设置
 */
private final ResultHandler<?> resultHandler;
private final BoundSql boundSql;
private final TypeHandlerRegistry typeHandlerRegistry;
private final ObjectFactory objectFactory;
private final ReflectorFactory reflectorFactory;

// nested resultmaps
private final Map<CacheKey, Object> nestedResultObjects = new HashMap<>();
private final Map<String, Object> ancestorObjects = new HashMap<>();
private Object previousRowValue;

// multiple resultsets
// 存储过程相关的多 ResultSet 涉及的属性，可以暂时忽略
private final Map<String, ResultMapping> nextResultMaps = new HashMap<>();
private final Map<CacheKey, List<PendingRelation>> pendingRelations = new HashMap<>();

// Cached Automappings
/**
 * 自动映射的缓存
 *
 * KEY：{@link ResultMap#getId()} + ":" +  columnPrefix
 *
 * @see #createRowKeyForUnmappedProperties(ResultMap, ResultSetWrapper, CacheKey, String) 
 */
private final Map<String, List<UnMappedColumnAutoMapping>> autoMappingsCache = new HashMap<>();

// temporary marking flag that indicate using constructor mapping (use field to reduce memory usage)
/**
 * 是否使用构造方法创建该结果对象
 */
private boolean useConstructorMappings;

public DefaultResultSetHandler(Executor executor, MappedStatement mappedStatement, ParameterHandler parameterHandler, ResultHandler<?> resultHandler, BoundSql boundSql, RowBounds rowBounds) {
    this.executor = executor;
    this.configuration = mappedStatement.getConfiguration();
    this.mappedStatement = mappedStatement;
    this.rowBounds = rowBounds;
    this.parameterHandler = parameterHandler;
    this.boundSql = boundSql;
    this.typeHandlerRegistry = configuration.getTypeHandlerRegistry();
    this.objectFactory = configuration.getObjectFactory();
    this.reflectorFactory = configuration.getReflectorFactory();
    this.resultHandler = resultHandler;
}
```

- 属性比较多，我们看重点的几个。
- `resultHandler` 属性，ResultHandler 对象。用户指定的用于处理结果的处理器，一般情况下，不设置。详细解析，见 [「5. ResultHandler」](#5. ResultHandler) 和 [「3.1.2.3.3 storeObject」](#3.1.2.3.3 storeObject) 。
- `autoMappingsCache` 属性，自动映射的缓存。其中，KEY 为 `{@link ResultMap#getId()} + ":" + columnPrefix` 。详细解析，见 [「3.1.2.3.2.4 applyAutomaticMappings」](3.1.2.3.2.4 applyAutomaticMappings) 。

### 3.1.2 handleResultSets

`#handleResultSets(Statement stmt)` 方法，处理 `java.sql.ResultSet` 结果集，转换成映射的对应结果。代码如下：

```java
// DefaultResultSetHandler.java

@Override
public List<Object> handleResultSets(Statement stmt) throws SQLException {
    ErrorContext.instance().activity("handling results").object(mappedStatement.getId());

    // <1> 多 ResultSet 的结果集合，每个 ResultSet 对应一个 Object 对象。而实际上，每个 Object 是 List<Object> 对象。
    // 在不考虑存储过程的多 ResultSet 的情况，普通的查询，实际就一个 ResultSet ，也就是说，multipleResults 最多就一个元素。
    final List<Object> multipleResults = new ArrayList<>();

    int resultSetCount = 0;
    // <2> 获得首个 ResultSet 对象，并封装成 ResultSetWrapper 对象
    ResultSetWrapper rsw = getFirstResultSet(stmt);

    // <3> 获得 ResultMap 数组
    // 在不考虑存储过程的多 ResultSet 的情况，普通的查询，实际就一个 ResultSet ，也就是说，resultMaps 就一个元素。
    List<ResultMap> resultMaps = mappedStatement.getResultMaps();
    int resultMapCount = resultMaps.size();
    validateResultMapsCount(rsw, resultMapCount); // <3.1> 校验
    while (rsw != null && resultMapCount > resultSetCount) {
        // <4.1> 获得 ResultMap 对象
        ResultMap resultMap = resultMaps.get(resultSetCount);
        // <4.2> 处理 ResultSet ，将结果添加到 multipleResults 中
        handleResultSet(rsw, resultMap, multipleResults, null);
        // <4.3> 获得下一个 ResultSet 对象，并封装成 ResultSetWrapper 对象
        rsw = getNextResultSet(stmt);
        // <4.4> 清理
        cleanUpAfterHandlingResultSet();
        // resultSetCount ++
        resultSetCount++;
    }

    // <5> 因为 `mappedStatement.resultSets` 只在存储过程中使用，本系列暂时不考虑，忽略即可
    String[] resultSets = mappedStatement.getResultSets();
    if (resultSets != null) {
        while (rsw != null && resultSetCount < resultSets.length) {
            ResultMapping parentMapping = nextResultMaps.get(resultSets[resultSetCount]);
            if (parentMapping != null) {
                String nestedResultMapId = parentMapping.getNestedResultMapId();
                ResultMap resultMap = configuration.getResultMap(nestedResultMapId);
                handleResultSet(rsw, resultMap, null, parentMapping);
            }
            rsw = getNextResultSet(stmt);
            cleanUpAfterHandlingResultSet();
            resultSetCount++;
        }
    }

    // <6> 如果是 multipleResults 单元素，则取首元素返回
    return collapseSingleResultList(multipleResults);
}
```

- 这个方法，不仅仅支持处理 Statement 和 PreparedStatement 返回的结果集，也支持存储过程的 CallableStatement 返回的结果集。而 CallableStatement 是支持返回多结果集的，这个大家要注意。😈 当然，还是老样子，本文不分析仅涉及存储过程的相关代码。哈哈哈。
- `<1>` 处，多 ResultSet 的结果集合，每个 ResultSet 对应一个 Object 对象。而实际上，每个 Object 是 List 对象。在不考虑存储过程的多 ResultSet 的情况，普通的查询，实际就一个 ResultSet ，也就是说，`multipleResults` **最多就一个元素**。

- `<2>` 处，调用 `#getFirstResultSet(Statement stmt)` 方法，获得首个 ResultSet 对象，并封装成 ResultSetWrapper 对象。代码如下：

  ```java
  // DefaultResultSetHandler.java
  
  private ResultSetWrapper getFirstResultSet(Statement stmt) throws SQLException {
      ResultSet rs = stmt.getResultSet();
      // 可以忽略
      while (rs == null) {
          // move forward to get the first resultset in case the driver
          // doesn't return the resultset as the first result (HSQLDB 2.1)
          if (stmt.getMoreResults()) {
              rs = stmt.getResultSet();
          } else {
              if (stmt.getUpdateCount() == -1) {
                  // no more results. Must be no resultset
                  break;
              }
          }
      }
      // 将 ResultSet 对象，封装成 ResultSetWrapper 对象
      return rs != null ? new ResultSetWrapper(rs, configuration) : null;
  }
  ```

- `<3>` 处，调用 `MappedStatement#getResultMaps()` 方法，获得 ResultMap 数组。在不考虑存储过程的多 ResultSet 的情况，普通的查询，实际就一个 ResultSet ，也就是说，`resultMaps` **就一个元素**。

  - `<3.1>` 处，调用 `#validateResultMapsCount(ResultSetWrapper rsw, int resultMapCount)` 方法，校验至少有一个 ResultMap 对象。代码如下：

    ```java
    // DefaultResultSetHandler.java
    
    private void validateResultMapsCount(ResultSetWrapper rsw, int resultMapCount) {
        if (rsw != null && resultMapCount < 1) {
            throw new ExecutorException("A query was run and no Result Maps were found for the Mapped Statement '" + mappedStatement.getId()
                    + "'.  It's likely that neither a Result Type nor a Result Map was specified.");
        }
    }
    ```

    - 不符合，则抛出 ExecutorException 异常。

- `<4.1>` 处，获得 ResultMap 对象。

- <span id='go_3.1.2_4.2'>`<4.2>` </span>处，调用 `#handleResultSet(ResultSetWrapper rsw, ResultMap resultMap, List<Object> multipleResults, ResultMapping parentMapping)` 方法，处理 ResultSet ，将结果添加到 `multipleResults` 中。详细解析，见 [「3.1.2.1 handleResultSet」](#3.1.2.1 handleResultSet) 。

- `<4.3>` 处，调用 `#getNextResultSet(Statement stmt)` 方法，获得下一个 ResultSet 对象，并封装成 ResultSetWrapper 对象。😈 只有存储过程才有多 ResultSet 对象，所以可以**忽略**。也就是说，实际上，这个 `while` 循环对我们来说，就不需要啦。

- `<4.4>` 处，调用 `#cleanUpAfterHandlingResultSet()` 方法，执行清理。代码如下：

  ```java
  // DefaultResultSetHandler.java
  
  private void cleanUpAfterHandlingResultSet() {
      nestedResultObjects.clear();
  }
  ```

- `<5>` 处，因为 `mappedStatement.resultSets` 只在存储过程中使用，本系列暂时不考虑，忽略即可。

- `<6>` 处，调用 `#collapseSingleResultList(List<Object> multipleResults)` 方法，如果是 `multipleResults` 单元素，则取首元素返回。代码如下：

  ```java
  // DefaultResultSetHandler.java
  
  private List<Object> collapseSingleResultList(List<Object> multipleResults) {
      return multipleResults.size() == 1 ? (List<Object>) multipleResults.get(0) : multipleResults;
  }
  ```

  - 对于非存储过程的结果处理，都能符合 `multipleResults.size()` 。

#### 3.1.2.1 handleResultSet

`#handleResultSet(ResultSetWrapper rsw, ResultMap resultMap, List<Object> multipleResults, ResultMapping parentMapping)` 方法，处理 ResultSet ，将结果添加到 `multipleResults` 中。代码如下： [<-](#go_3.1.2_4.2)

```java
// DefaultResultSetHandler.java

private void handleResultSet(ResultSetWrapper rsw, ResultMap resultMap, List<Object> multipleResults, ResultMapping parentMapping) throws SQLException {
    try {
        // <1> 暂时忽略，因为只有存储过程的情况，调用该方法，parentMapping 为非空
        if (parentMapping != null) {
            handleRowValues(rsw, resultMap, null, RowBounds.DEFAULT, parentMapping);
        } else {
            // <2> 如果没有自定义的 resultHandler ，则创建默认的 DefaultResultHandler 对象
            if (resultHandler == null) {
                // <2> 创建 DefaultResultHandler 对象
                DefaultResultHandler defaultResultHandler = new DefaultResultHandler(objectFactory);
                // <3> 处理 ResultSet 返回的每一行 Row
                handleRowValues(rsw, resultMap, defaultResultHandler, rowBounds, null);
                // <4> 添加 defaultResultHandler 的处理的结果，到 multipleResults 中
                multipleResults.add(defaultResultHandler.getResultList());
            } else {
                // <3> 处理 ResultSet 返回的每一行 Row
                handleRowValues(rsw, resultMap, resultHandler, rowBounds, null);
            }
        }
    } finally {
        // issue #228 (close resultsets)
        // 关闭 ResultSet 对象
        closeResultSet(rsw.getResultSet());
    }
}
```

- `<1>` 处，暂时忽略，因为只有存储过程的情况，调用该方法，`parentMapping` 为非空。

- `<2>` 处，如果没有自定义的 `resultHandler` ，则创建默认的 DefaultResultHandler 对象。

- <span id='go_3.1.2.1_3'>`<3>` </span>处，调用 `#handleRowValues(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping)` 方法，处理 ResultSet 返回的每一行 Row 。详细解析，见 [「3.1.2.2 handleRowValues」](#3.1.2.2 handleRowValues) 。

- 【特殊】`<4>` 处，使用**默认**的 DefaultResultHandler 对象，最终会将 `defaultResultHandler` 的处理的结果，到 `multipleResults` 中。而使用**自定义**的 `resultHandler` ，不会添加到 `multipleResults` 中。当然，因为自定义的 `resultHandler` 对象，是作为一个对象传入，所以在其内部，还是可以存储结果的。例如：

  ```java
  @Select("select * from users")
  @ResultType(User.class)
  void getAllUsers(UserResultHandler resultHandler);
  ```

  - 感兴趣的胖友，可以看看 [《mybatis ResultHandler 示例》](http://outofmemory.cn/code-snippet/13271/mybatis-complex-bean-property-handle-with-custom-ResultHandler) 。

- `<5>` 处，调用 `#closeResultSet(ResultSet rs)` 方法关闭 ResultSet 对象。代码如下：

  ```java
  // DefaultResultSetHandler.java
  
  private void closeResultSet(ResultSet rs) {
      try {
          if (rs != null) {
              rs.close();
          }
      } catch (SQLException e) {
          // ignore
      }
  }
  ```

#### 3.1.2.2 handleRowValues

`#handleRowValues(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping)` 方法，处理 ResultSet 返回的每一行 Row 。代码如下：[<-](#go_3.1.2.1_3)

```java
// DefaultResultSetHandler.java

public void handleRowValues(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping) throws SQLException {
    // <1> 处理嵌套映射的情况
    if (resultMap.hasNestedResultMaps()) {
        // 校验不要使用 RowBounds
        ensureNoRowBounds();
        // 校验不要使用自定义的 resultHandler
        checkResultHandler();
        // 处理嵌套映射的结果
        handleRowValuesForNestedResultMap(rsw, resultMap, resultHandler, rowBounds, parentMapping);
    // <2> 处理简单映射的情况
    } else {
        // <2.1> 处理简单映射的结果
        handleRowValuesForSimpleResultMap(rsw, resultMap, resultHandler, rowBounds, parentMapping);
    }
}
```

- 分成**嵌套映射**和**简单映射**的两种情况。

- `<2>`处理嵌套映射的情况：

  - <span id='go3.1.2.2_2.1'>`<2.1>` </span>处，调用 `#handleRowValuesForSimpleResultMap(...)` 方法，处理简单映射的结果。详细解析，见 [「3.1.2.3 handleRowValuesForSimpleResultMap 简单映射」](#3.1.2.3 handleRowValuesForSimpleResultMap 简单映射) 。

- `<1>` 处理嵌套映射的情况：

  - `<1.1>` 处，调用 `#ensureNoRowBounds()` 方法，校验不要使用 RowBounds 。代码如下：

    ```java
    // DefaultResultSetHandler.java
    
    private void ensureNoRowBounds() {
        // configuration.isSafeRowBoundsEnabled() 默认为 false
        if (configuration.isSafeRowBoundsEnabled() && rowBounds != null && (rowBounds.getLimit() < RowBounds.NO_ROW_LIMIT || rowBounds.getOffset() > RowBounds.NO_ROW_OFFSET)) {
            throw new ExecutorException("Mapped Statements with nested result mappings cannot be safely constrained by RowBounds. "
                    + "Use safeRowBoundsEnabled=false setting to bypass this check.");
        }
    }
    ```

    - 简单看看即可。

  - `<1.2>` 处，调用 `#checkResultHandler()` 方法，校验不要使用自定义的 `resultHandler` 。代码如下：

    ```java
    // DefaultResultSetHandler.java
    
    protected void checkResultHandler() {
        // configuration.isSafeResultHandlerEnabled() 默认为 false
        if (resultHandler != null && configuration.isSafeResultHandlerEnabled() && !mappedStatement.isResultOrdered()) {
            throw new ExecutorException("Mapped Statements with nested result mappings cannot be safely used with a custom ResultHandler. "
                    + "Use safeResultHandlerEnabled=false setting to bypass this check "
                    + "or ensure your statement returns ordered data and set resultOrdered=true on it.");
        }
    }
    ```

    - 简单看看即可。

  - `<1.3>` 处，调用 `#handleRowValuesForSimpleResultMap(...)` 方法，处理嵌套映射的结果。详细解析，见 [「3.1.2.3 handleRowValuesForNestedResultMap 嵌套映射」](http://svip.iocoder.cn/MyBatis/executor-4/#) 。

#### 3.1.2.3 handleRowValuesForSimpleResultMap 简单映射

`#handleRowValuesForSimpleResultMap(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping)` 方法，处理简单映射的结果。代码如下：[<-](#go3.1.2.2_2.1)

```java
// DefaultResultSetHandler.java

private void handleRowValuesForSimpleResultMap(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping) throws SQLException {
    // <1> 创建 DefaultResultContext 对象
    DefaultResultContext<Object> resultContext = new DefaultResultContext<>();
    // <2> 获得 ResultSet 对象，并跳到 rowBounds 指定的开始位置
    ResultSet resultSet = rsw.getResultSet();
    skipRows(resultSet, rowBounds);
    // <3> 循环
    while (shouldProcessMoreRows(resultContext, rowBounds) // 是否继续处理 ResultSet
            && !resultSet.isClosed() // ResultSet 是否已经关闭
            && resultSet.next()) { // ResultSet 是否还有下一条
        // <4> 根据该行记录以及 ResultMap.discriminator ，决定映射使用的 ResultMap 对象
        ResultMap discriminatedResultMap = resolveDiscriminatedResultMap(resultSet, resultMap, null);
        // <5> 根据最终确定的 ResultMap 对 ResultSet 中的该行记录进行映射，得到映射后的结果对象
        Object rowValue = getRowValue(rsw, discriminatedResultMap, null);
        // <6> 将映射创建的结果对象添加到 ResultHandler.resultList 中保存
        storeObject(resultHandler, resultContext, rowValue, parentMapping, resultSet);
    }
}
```

- <span id='go3.1.2.3_1'>`<1>` </span>处，创建 DefaultResultContext 对象。详细解析，胖友**先**跳到 [「4. ResultContext」](#4. ResultContext) 中，看完就回来。

- `<2>` 处，获得 ResultSet 对象，并调用 `#skipRows(ResultSet rs, RowBounds rowBounds)` 方法，跳到 `rowBounds` 指定的开始位置。代码如下：

  ```java
  // DefaultResultSetHandler.java
  
  private void skipRows(ResultSet rs, RowBounds rowBounds) throws SQLException {
      if (rs.getType() != ResultSet.TYPE_FORWARD_ONLY) {
          // 直接跳转到指定开始的位置
          if (rowBounds.getOffset() != RowBounds.NO_ROW_OFFSET) {
              rs.absolute(rowBounds.getOffset());
          }
      } else {
          // 循环，不断跳到开始的位置
          for (int i = 0; i < rowBounds.getOffset(); i++) {
              if (!rs.next()) {
                  break;
              }
          }
      }
  }
  ```

  - 关于 `org.apache.ibatis.session.RowBounds` 类，胖友可以看看 [《Mybatis3.3.x技术内幕（十三）：Mybatis之RowBounds分页原理》](https://my.oschina.net/zudajun/blog/671446) ，解释的非常不错。

- `<3>` 处，循环，满足如下三个条件。其中 `#shouldProcessMoreRows(ResultContext<?> context, RowBounds rowBounds)` 方法，是否继续处理 ResultSet 。代码如下：

  ```java
  // DefaultResultSetHandler.java
  
  private boolean shouldProcessMoreRows(ResultContext<?> context, RowBounds rowBounds) {
      return !context.isStopped() && context.getResultCount() < rowBounds.getLimit();
  }
  ```

- <span id='go3.1.2.3.1_4'>`<4>` </span>处，调用 `#resolveDiscriminatedResultMap(...)` 方法，根据该行记录以及 `ResultMap.discriminator` ，决定映射使用的 ResultMap 对象。详细解析，等下看 [「3.1.2.3.1 resolveDiscriminatedResultMap」](#3.1.2.3.1 resolveDiscriminatedResultMap) 。

- <span id='go3.1.2.3.1_5'>`<5>` </span>处，调用 `#getRowValue(...)` 方法，根据最终确定的 ResultMap 对 ResultSet 中的该行记录进行映射，得到映射后的结果对象。详细解析，等下看 [「3.1.2.3.2 getRowValue」](#3.1.2.3.2 getRowValue) 。

- <span id='go3.1.2.3.1_6'>`<6>` </span> 处，调用 `#storeObject(...)` 方法，将映射创建的结果对象添加到 `ResultHandler.resultList` 中保存。详细解析，等下看 [「3.1.2.3.3 storeObject」](#3.1.2.3.3 storeObject) 。

##### 3.1.2.3.1 resolveDiscriminatedResultMap

`#resolveDiscriminatedResultMap(ResultSet rs, ResultMap resultMap, String columnPrefix)` 方法，根据该行记录以及 `ResultMap.discriminator` ，决定映射使用的 ResultMap 对象。代码如下：[<-](#go3.1.2.3.1_4)

```java
// DefaultResultSetHandler.java

public ResultMap resolveDiscriminatedResultMap(ResultSet rs, ResultMap resultMap, String columnPrefix) throws SQLException {
    // 记录已经处理过的 Discriminator 对应的 ResultMap 的编号
    Set<String> pastDiscriminators = new HashSet<>();
    // 如果存在 Discriminator 对象，则基于其获得 ResultMap 对象
    Discriminator discriminator = resultMap.getDiscriminator();
    while (discriminator != null) { // 因为 Discriminator 可以嵌套 Discriminator ，所以是一个递归的过程
        // 获得 Discriminator 的指定字段，在 ResultSet 中该字段的值
        final Object value = getDiscriminatorValue(rs, discriminator, columnPrefix);
        // 从 Discriminator 获取该值对应的 ResultMap 的编号
        final String discriminatedMapId = discriminator.getMapIdFor(String.valueOf(value));
        // 如果存在，则使用该 ResultMap 对象
        if (configuration.hasResultMap(discriminatedMapId)) {
            // 获得该 ResultMap 对象
            resultMap = configuration.getResultMap(discriminatedMapId);
            // 判断，如果出现“重复”的情况，结束循环
            Discriminator lastDiscriminator = discriminator;
            discriminator = resultMap.getDiscriminator();
            if (discriminator == lastDiscriminator || !pastDiscriminators.add(discriminatedMapId)) {
                break;
            }
        // 如果不存在，直接结束循环
        } else {
            break;
        }
    }
    return resultMap;
}
```

- 对于大多数情况下，大家不太会使用 Discriminator 的功能，此处就直接返回 `resultMap` ，不会执行这个很复杂的逻辑。😈 所以，如果看不太懂的胖友，可以略过这个方法，问题也不大。

- 代码比较繁杂，胖友跟着注释看看，甚至可以调试下。其中，`<1>` 处，调用 `#getDiscriminatorValue(ResultSet rs, Discriminator discriminator, String columnPrefix)` 方法，获得 Discriminator 的指定字段，在 ResultSet 中该字段的值。代码如下：

  ```java
  // DefaultResultSetHandler.java
  
  /**
   * 获得 ResultSet 的指定字段的值
   *
   * @param rs ResultSet 对象
   * @param discriminator Discriminator 对象
   * @param columnPrefix 字段名的前缀
   * @return 指定字段的值
   */
  private Object getDiscriminatorValue(ResultSet rs, Discriminator discriminator, String columnPrefix) throws SQLException {
      final ResultMapping resultMapping = discriminator.getResultMapping();
      final TypeHandler<?> typeHandler = resultMapping.getTypeHandler();
      // 获得 ResultSet 的指定字段的值
      return typeHandler.getResult(rs, prependPrefix(resultMapping.getColumn(), columnPrefix));
  }
  
  /**
   * 拼接指定字段的前缀
   *
   * @param columnName 字段的名字
   * @param prefix 前缀
   * @return prefix + columnName
   */
  private String prependPrefix(String columnName, String prefix) {
      if (columnName == null || columnName.length() == 0 || prefix == null || prefix.length() == 0) {
          return columnName;
      }
      return prefix + columnName;
  }
  ```

##### 3.1.2.3.2 getRowValue

`#getRowValue(ResultSetWrapper rsw, ResultMap resultMap, String columnPrefix)` 方法，根据最终确定的 ResultMap 对 ResultSet 中的该行记录进行映射，得到映射后的结果对象。代码如下：[<-](#go3.1.2.3.1_5)

```java
// DefaultResultSetHandler.java

private Object getRowValue(ResultSetWrapper rsw, ResultMap resultMap, String columnPrefix) throws SQLException {
    // <1> 创建 ResultLoaderMap 对象
    final ResultLoaderMap lazyLoader = new ResultLoaderMap();
    // <2> 创建映射后的结果对象
    Object rowValue = createResultObject(rsw, resultMap, lazyLoader, columnPrefix);
    // <3> 如果 hasTypeHandlerForResultObject(rsw, resultMap.getType()) 返回 true ，意味着 rowValue 是基本类型，无需执行下列逻辑。
    if (rowValue != null && !hasTypeHandlerForResultObject(rsw, resultMap.getType())) {
        // <4> 创建 MetaObject 对象，用于访问 rowValue 对象
        final MetaObject metaObject = configuration.newMetaObject(rowValue);
        // <5> foundValues 代表，是否成功映射任一属性。若成功，则为 true ，若失败，则为 false
        boolean foundValues = this.useConstructorMappings;
        /// <6.1> 判断是否开启自动映射功能
        if (shouldApplyAutomaticMappings(resultMap, false)) {
            // <6.2> 自动映射未明确的列
            foundValues = applyAutomaticMappings(rsw, resultMap, metaObject, columnPrefix) || foundValues;
        }
        // <7> 映射 ResultMap 中明确映射的列
        foundValues = applyPropertyMappings(rsw, resultMap, metaObject, lazyLoader, columnPrefix) || foundValues;
        // <8> ↑↑↑ 至此，当前 ResultSet 的该行记录的数据，已经完全映射到结果对象 rowValue 的对应属性种
        foundValues = lazyLoader.size() > 0 || foundValues;
        // <9> 如果没有成功映射任意属性，则置空 rowValue 对象。
        // 当然，如果开启 `configuration.returnInstanceForEmptyRow` 属性，则不置空。默认情况下，该值为 false
        rowValue = foundValues || configuration.isReturnInstanceForEmptyRow() ? rowValue : null;
    }
    return rowValue;
}
```

- `<1>` 处，创建 ResultLoaderMap 对象。延迟加载相关。

- <span id='go3.1.2.3.2_2'>`<2>` </span>处，调用 `#createResultObject(...)` 方法，创建映射后的结果对象。详细解析，见 [「3.1.2.3.2.1 createResultObject」](#3.1.2.3.2.1 createResultObject) 。😈 mmp ，这个逻辑的嵌套，真的太深太深了。

- `<3>` 处，调用 `#hasTypeHandlerForResultObject(rsw, resultMap.getType())` 方法，返回 `true` ，意味着 `rowValue` 是基本类型，无需执行下列逻辑。代码如下：

  ```java
  // DefaultResultSetHandler.java
  
  // 判断是否结果对象是否有 TypeHandler 对象
  private boolean hasTypeHandlerForResultObject(ResultSetWrapper rsw, Class<?> resultType) {
      // 如果返回的字段只有一个，则直接判断该字段是否有 TypeHandler 对象
      if (rsw.getColumnNames().size() == 1) {
          return typeHandlerRegistry.hasTypeHandler(resultType, rsw.getJdbcType(rsw.getColumnNames().get(0)));
      }
      // 判断 resultType 是否有对应的 TypeHandler 对象
      return typeHandlerRegistry.hasTypeHandler(resultType);
  }
  ```

  - 有点绕，胖友可以调试下 `BindingTest#shouldInsertAuthorWithSelectKeyAndDynamicParams()` 方法。
  - 再例如，`<select resultType="Integer" />` 的情况。

- `<4>` 处，创建 MetaObject 对象，用于访问 `rowValue` 对象。

- `<5>` 处，`foundValues` 代表，是否成功映射任一属性。若成功，则为 `true` ，若失败，则为 `false` 。另外，此处使用 `useConstructorMappings` 作为 `foundValues` 的初始值，原因是，使用了构造方法创建该结果对象，意味着一定找到了任一属性。

- `<6.1>` 处，调用 `#shouldApplyAutomaticMappings(ResultMap resultMap, boolean isNested)` 方法，**判断是否使用自动映射的功能**。代码如下：

  ```java
  // DefaultResultSetHandler.java
  
  private boolean shouldApplyAutomaticMappings(ResultMap resultMap, boolean isNested) {
      // 判断是否开启自动映射功能
      if (resultMap.getAutoMapping() != null) {
          return resultMap.getAutoMapping();
      } else {
          // 内嵌查询或嵌套映射时
          if (isNested) {
              return AutoMappingBehavior.FULL == configuration.getAutoMappingBehavior(); // 需要 FULL
          // 普通映射
          } else {
              return AutoMappingBehavior.NONE != configuration.getAutoMappingBehavior(); // 需要 PARTIAL 或 FULL
          }
      }
  }
  ```

  - `org.apache.ibatis.session.AutoMappingBehavior` ，自动映射行为的枚举。代码如下：

    ```java
    // AutoMappingBehavior.java
    
    /**
     * Specifies if and how MyBatis should automatically map columns to fields/properties.
     *
     * 自动映射行为的枚举
     *
     * @author Eduardo Macarron
     */
    public enum AutoMappingBehavior {
    
        /**
         * Disables auto-mapping.
         *
         * 禁用自动映射的功能
         */
        NONE,
    
        /**
         * Will only auto-map results with no nested result mappings defined inside.
         *
         * 开启部分映射的功能
         */
        PARTIAL,
    
        /**
         * Will auto-map result mappings of any complexity (containing nested or otherwise).
         *
         * 开启全部映射的功能
         */
        FULL
    
    }
    ```

    - x

  - `Configuration.autoMappingBehavior` 属性，默认为 `AutoMappingBehavior.PARTIAL` 。

- <span id='go3.1.2.3.2.1_6.2'>`<6.2>`</span> 处，调用 `#applyAutomaticMappings(...)` 方法，自动映射未明确的列。代码有点长，所以，详细解析，见 [「3.1.2.3.2.3 applyAutomaticMappings」](#3.1.2.3.2.3 applyAutomaticMappings) 。

- <span id='go3.1.2.3.2.1_7'>`<7>`</span>  处，调用 `#applyPropertyMappings(...)` 方法，映射 ResultMap 中明确映射的列。代码有点长，所以，详细解析，见 [「3.1.2.3.2.4 applyPropertyMappings」](#3.1.2.3.2.4 applyPropertyMappings) 。

- `<8>` 处，↑↑↑ 至此，当前 ResultSet 的该行记录的数据，已经完全映射到结果对象 `rowValue` 的对应属性中。😈 整个过程，非常非常非常长，胖友耐心理解和调试下。

- `<9>` 处，如果没有成功映射任意属性，则置空 rowValue 对象。当然，如果开启 `configuration.returnInstanceForEmptyRow` 属性，则不置空。默认情况下，该值为 `false` 。

###### 3.1.2.3.2.1 createResultObject

`#createResultObject(ResultSetWrapper rsw, ResultMap resultMap, ResultLoaderMap lazyLoader, String columnPrefix)` 方法，创建映射后的结果对象。代码如下：[<-](#go3.1.2.3.2_2)

```java
// DefaultResultSetHandler.java

private Object createResultObject(ResultSetWrapper rsw, ResultMap resultMap, ResultLoaderMap lazyLoader, String columnPrefix) throws SQLException {
    // <1> useConstructorMappings ，表示是否使用构造方法创建该结果对象。此处将其重置
    this.useConstructorMappings = false; // reset previous mapping result
    final List<Class<?>> constructorArgTypes = new ArrayList<>(); // 记录使用的构造方法的参数类型的数组
    final List<Object> constructorArgs = new ArrayList<>(); // 记录使用的构造方法的参数值的数组
    // <2> 创建映射后的结果对象
    Object resultObject = createResultObject(rsw, resultMap, constructorArgTypes, constructorArgs, columnPrefix);
    if (resultObject != null && !hasTypeHandlerForResultObject(rsw, resultMap.getType())) {
        // <3> 如果有内嵌的查询，并且开启延迟加载，则创建结果对象的代理对象
        final List<ResultMapping> propertyMappings = resultMap.getPropertyResultMappings();
        for (ResultMapping propertyMapping : propertyMappings) {
            // issue gcode #109 && issue #149
            if (propertyMapping.getNestedQueryId() != null && propertyMapping.isLazy()) {
                resultObject = configuration.getProxyFactory().createProxy(resultObject, lazyLoader, configuration, objectFactory, constructorArgTypes, constructorArgs);
                break;
            }
        }
    }
    // <4> 判断是否使用构造方法创建该结果对象
    this.useConstructorMappings = resultObject != null && !constructorArgTypes.isEmpty(); // set current mapping result
    return resultObject;
}
```

- `<1>` 处，`useConstructorMappings` ，表示是否使用构造方法创建该结果对象。而此处，将其重置为 `false` 。
- <span id='go_3.1.2.3.2.1_2'>`<2>` </span>处，调用 `#createResultObject(...)` 方法，创建映射后的结果对象。[详细解析，往下看](#go_3.1.2.3.2.1_createResultObject)。😈 再次 mmp ，调用链太长了。
- `<3>` 处，如果有内嵌的查询，并且开启延迟加载，则调用 `ProxyFactory#createProxy(...)` 方法，创建结果对象的代理对象。详细解析，见 [《精尽 MyBatis 源码分析 —— SQL 执行（五）之延迟加载》](http://svip.iocoder.cn/MyBatis/executor-5) 。
- `<4>` 处，判断是否使用构造方法创建该结果对象，并设置到 `useConstructorMappings` 中。

------

<span id='go_3.1.2.3.2.1_createResultObject'>`#createResultObject(ResultSetWrapper rsw, ResultMap resultMap, List<Class<?>> constructorArgTypes, List<Object> constructorArgs, String columnPrefix)` </span>span>方法，创建映射后的结果对象。代码如下：[<-](#go_3.1.2.3.2.1_2)

```java
// DefaultResultSetHandler.java

private Object createResultObject(ResultSetWrapper rsw, ResultMap resultMap, List<Class<?>> constructorArgTypes, List<Object> constructorArgs, String columnPrefix)
        throws SQLException {
    final Class<?> resultType = resultMap.getType();
    final MetaClass metaType = MetaClass.forClass(resultType, reflectorFactory);
    final List<ResultMapping> constructorMappings = resultMap.getConstructorResultMappings();
    // 下面，分成四种创建结果对象的情况
    // <1> 情况一，如果有对应的 TypeHandler 对象，则意味着是基本类型，直接创建对结果应对象
    if (hasTypeHandlerForResultObject(rsw, resultType)) {
        return createPrimitiveResultObject(rsw, resultMap, columnPrefix);
    // 情况二，如果 ResultMap 中，如果定义了 `<constructor />` 节点，则通过反射调用该构造方法，创建对应结果对象
    } else if (!constructorMappings.isEmpty()) {
        return createParameterizedResultObject(rsw, resultType, constructorMappings, constructorArgTypes, constructorArgs, columnPrefix);
    // 情况三，如果有默认的无参的构造方法，则使用该构造方法，创建对应结果对象
    } else if (resultType.isInterface() || metaType.hasDefaultConstructor()) {
        return objectFactory.create(resultType);
    // 情况四，通过自动映射的方式查找合适的构造方法，后使用该构造方法，创建对应结果对象
    } else if (shouldApplyAutomaticMappings(resultMap, false)) {
        return createByConstructorSignature(rsw, resultType, constructorArgTypes, constructorArgs, columnPrefix);
    }
    // 不支持，抛出 ExecutorException 异常
    throw new ExecutorException("Do not know how to create an instance of " + resultType);
}
```

- 分成四种创建结果对象的情况。

- `<1>` 处，情况一，如果有对应的 TypeHandler 对象，则意味着是基本类型，直接创建对结果应对象。调用 `#createPrimitiveResultObject(ResultSetWrapper rsw, ResultMap resultMap, String columnPrefix)` 方法，代码如下：

  ```java
  // DefaultResultSetHandler.java
  
  private Object createPrimitiveResultObject(ResultSetWrapper rsw, ResultMap resultMap, String columnPrefix) throws SQLException {
      final Class<?> resultType = resultMap.getType();
      // 获得字段名
      final String columnName;
      if (!resultMap.getResultMappings().isEmpty()) {
          final List<ResultMapping> resultMappingList = resultMap.getResultMappings();
          final ResultMapping mapping = resultMappingList.get(0);
          columnName = prependPrefix(mapping.getColumn(), columnPrefix);
      } else {
          columnName = rsw.getColumnNames().get(0);
      }
      // 获得 TypeHandler 对象
      final TypeHandler<?> typeHandler = rsw.getTypeHandler(resultType, columnName);
      // 获得 ResultSet 的指定字段的值
      return typeHandler.getResult(rsw.getResultSet(), columnName);
  }
  ```

- `<2>` 处，情况二，如果 ResultMap 中，如果定义了 `<constructor />` 节点，则通过反射调用该构造方法，创建对应结果对象。调用 `#createParameterizedResultObject(ResultSetWrapper rsw, Class<?> resultType, List<ResultMapping> constructorMappings, List<Class<?>> constructorArgTypes, List<Object> constructorArgs, String columnPrefix)` 方法，代码如下：

  ```java
  // DefaultResultSetHandler.java
  
  Object createParameterizedResultObject(ResultSetWrapper rsw, Class<?> resultType, List<ResultMapping> constructorMappings,
                                         List<Class<?>> constructorArgTypes, List<Object> constructorArgs, String columnPrefix) {
      // 获得到任一的属性值。即，只要一个结果对象，有一个属性非空，就会设置为 true
      boolean foundValues = false;
      for (ResultMapping constructorMapping : constructorMappings) {
          // 获得参数类型
          final Class<?> parameterType = constructorMapping.getJavaType();
          // 获得数据库的字段名
          final String column = constructorMapping.getColumn();
          // 获得属性值
          final Object value;
          try {
              // 如果是内嵌的查询，则获得内嵌的值
              if (constructorMapping.getNestedQueryId() != null) {
                  value = getNestedQueryConstructorValue(rsw.getResultSet(), constructorMapping, columnPrefix);
              // 如果是内嵌的 resultMap ，则递归 getRowValue 方法，获得对应的属性值
              } else if (constructorMapping.getNestedResultMapId() != null) {
                  final ResultMap resultMap = configuration.getResultMap(constructorMapping.getNestedResultMapId());
                  value = getRowValue(rsw, resultMap, constructorMapping.getColumnPrefix());
              // 最常用的情况，直接使用 TypeHandler 获取当前 ResultSet 的当前行的指定字段的值
              } else {
                  final TypeHandler<?> typeHandler = constructorMapping.getTypeHandler();
                  value = typeHandler.getResult(rsw.getResultSet(), prependPrefix(column, columnPrefix));
              }
          } catch (ResultMapException | SQLException e) {
              throw new ExecutorException("Could not process result for mapping: " + constructorMapping, e);
          }
          // 添加到 constructorArgTypes 和 constructorArgs 中
          constructorArgTypes.add(parameterType);
          constructorArgs.add(value);
          // 判断是否获得到属性值
          foundValues = value != null || foundValues;
      }
      // 查找 constructorArgTypes 对应的构造方法
      // 查找到后，传入 constructorArgs 作为参数，创建结果对象
      return foundValues ? objectFactory.create(resultType, constructorArgTypes, constructorArgs) : null;
  }
  ```

  - 代码比较简单，胖友看下注释即可。
  - 当然，里面的<span id='go3.1.2.3.2.1_getNestedQueryConstructorValue'> `#getNestedQueryConstructorValue(ResultSet rs, ResultMapping constructorMapping, String columnPrefix)` </span>方法的逻辑，还是略微比较复杂的。所以，我们在讲完情况三、情况四，我们再来看看它的实现。😈 写到这里，艿艿的心里无比苦闷。详细解析，见 [「3.1.2.3.2.2 getNestedQueryConstructorValue」](#3.1.2.3.2.2 getNestedQueryConstructorValue 嵌套查询) 。

- `<3>` 处，情况三，如果有默认的无参的构造方法，则使用该构造方法，创建对应结果对象。

- `<4>` 处，情况四，通过自动映射的方式查找合适的构造方法，后使用该构造方法，创建对应结果对象。调用 `#createByConstructorSignature(ResultSetWrapper rsw, Class<?> resultType, List<Class<?>> constructorArgTypes, List<Object> constructorArgs, String columnPrefix)` 方法，代码如下：

  ```java
  // DefaultResultSetHandler.java
  
  private Object createByConstructorSignature(ResultSetWrapper rsw, Class<?> resultType, List<Class<?>> constructorArgTypes, List<Object> constructorArgs,
                                              String columnPrefix) throws SQLException {
      // <1> 获得所有构造方法
      final Constructor<?>[] constructors = resultType.getDeclaredConstructors();
      // <2> 获得默认构造方法
      final Constructor<?> defaultConstructor = findDefaultConstructor(constructors);
      // <3> 如果有默认构造方法，使用该构造方法，创建结果对象
      if (defaultConstructor != null) {
          return createUsingConstructor(rsw, resultType, constructorArgTypes, constructorArgs, columnPrefix, defaultConstructor);
      } else {
          // <4> 遍历所有构造方法，查找符合的构造方法，创建结果对象
          for (Constructor<?> constructor : constructors) {
              if (allowedConstructorUsingTypeHandlers(constructor, rsw.getJdbcTypes())) {
                  return createUsingConstructor(rsw, resultType, constructorArgTypes, constructorArgs, columnPrefix, constructor);
              }
          }
      }
      throw new ExecutorException("No constructor found in " + resultType.getName() + " matching " + rsw.getClassNames());
  }
  ```

  - `<1>` 处，获得所有构造方法。

  - `<2>` 处，调用 `#findDefaultConstructor(final Constructor<?>[] constructors)` 方法，获得默认构造方法。代码如下：

    ```java
    // DefaultResultSetHandler.java
    
    private Constructor<?> findDefaultConstructor(final Constructor<?>[] constructors) {
        // 构造方法只有一个，直接返回
        if (constructors.length == 1) return constructors[0];
        // 获得使用 @AutomapConstructor 注解的构造方法
        for (final Constructor<?> constructor : constructors) {
            if (constructor.isAnnotationPresent(AutomapConstructor.class)) {
                return constructor;
            }
        }
        return null;
    }
    ```

    - 两种情况，比较简单。

  - `<3>` 处，如果有默认构造方法，调用 `#createUsingConstructor(ResultSetWrapper rsw, Class<?> resultType, List<Class<?>> constructorArgTypes, List<Object> constructorArgs, String columnPrefix, Constructor<?> constructor)` 方法，使用该构造方法，创建结果对象。代码如下：

    ```java
    // DefaultResultSetHandler.java
    
    private Object createUsingConstructor(ResultSetWrapper rsw, Class<?> resultType, List<Class<?>> constructorArgTypes, List<Object> constructorArgs, String columnPrefix, Constructor<?> constructor) throws SQLException {
        boolean foundValues = false;
        for (int i = 0; i < constructor.getParameterTypes().length; i++) {
            // 获得参数类型
            Class<?> parameterType = constructor.getParameterTypes()[i];
            // 获得数据库的字段名
            String columnName = rsw.getColumnNames().get(i);
            // 获得 TypeHandler 对象
            TypeHandler<?> typeHandler = rsw.getTypeHandler(parameterType, columnName);
            // 获取当前 ResultSet 的当前行的指定字段的值
            Object value = typeHandler.getResult(rsw.getResultSet(), prependPrefix(columnName, columnPrefix));
            // 添加到 constructorArgTypes 和 constructorArgs 中
            constructorArgTypes.add(parameterType);
            constructorArgs.add(value);
            // 判断是否获得到属性值
            foundValues = value != null || foundValues;
        }
        // 查找 constructorArgTypes 对应的构造方法
        // 查找到后，传入 constructorArgs 作为参数，创建结果对象
        return foundValues ? objectFactory.create(resultType, constructorArgTypes, constructorArgs) : null;
    }
    ```

    - 从代码实现上，和 `#createParameterizedResultObject(...)` 方法，类似。

  - `<4>` 处，遍历所有构造方法，调用 `#allowedConstructorUsingTypeHandlers(final Constructor<?> constructor, final List<JdbcType> jdbcTypes)` 方法，查找符合的构造方法，后创建结果对象。代码如下：

    ```java
    // DefaultResultSetHandler.java
    
    private boolean allowedConstructorUsingTypeHandlers(final Constructor<?> constructor, final List<JdbcType> jdbcTypes) {
        final Class<?>[] parameterTypes = constructor.getParameterTypes();
        // 结果集的返回字段的数量，要和构造方法的参数数量，一致
        if (parameterTypes.length != jdbcTypes.size()) return false;
        // 每个构造方法的参数，和对应的返回字段，都要有对应的 TypeHandler 对象
        for (int i = 0; i < parameterTypes.length; i++) {
            if (!typeHandlerRegistry.hasTypeHandler(parameterTypes[i], jdbcTypes.get(i))) {
                return false;
            }
        }
        // 返回匹配
        return true;
    }
    ```

    - 基于结果集的返回字段和构造方法的参数做比较。

###### 3.1.2.3.2.2 getNestedQueryConstructorValue 嵌套查询

> 老艿艿：冲鸭！！！太冗长了！！！各种各种各种！！！！情况！！！！！

`#getNestedQueryConstructorValue(ResultSet rs, ResultMapping constructorMapping, String columnPrefix)` 方法，获得嵌套查询的值。代码如下：[<-](#go3.1.2.3.2.1_getNestedQueryConstructorValue)

```java
// DefaultResultSetHandler.java

private Object getNestedQueryConstructorValue(ResultSet rs, ResultMapping constructorMapping, String columnPrefix) throws SQLException {
    // <1> 获得内嵌查询的编号
    final String nestedQueryId = constructorMapping.getNestedQueryId();
    // <1> 获得内嵌查询的 MappedStatement 对象
    final MappedStatement nestedQuery = configuration.getMappedStatement(nestedQueryId);
    // <1> 获得内嵌查询的参数类型
    final Class<?> nestedQueryParameterType = nestedQuery.getParameterMap().getType();
    // <2> 获得内嵌查询的参数对象
    final Object nestedQueryParameterObject = prepareParameterForNestedQuery(rs, constructorMapping, nestedQueryParameterType, columnPrefix);
    Object value = null;
    // <3> 执行查询
    if (nestedQueryParameterObject != null) {
        // <3.1> 获得 BoundSql 对象
        final BoundSql nestedBoundSql = nestedQuery.getBoundSql(nestedQueryParameterObject);
        // <3.2> 获得 CacheKey 对象
        final CacheKey key = executor.createCacheKey(nestedQuery, nestedQueryParameterObject, RowBounds.DEFAULT, nestedBoundSql);
        final Class<?> targetType = constructorMapping.getJavaType();
        // <3.3> 创建 ResultLoader 对象
        final ResultLoader resultLoader = new ResultLoader(configuration, executor, nestedQuery, nestedQueryParameterObject, targetType, key, nestedBoundSql);
        // <3.3> 加载结果
        value = resultLoader.loadResult();
    }
    return value;
}
```

- 关于这个方法，胖友可以调试 `BaseExecutorTest#shouldFetchOneOrphanedPostWithNoBlog()` 这个单元测试方法。

- `<1>` 处，获得内嵌查询的**编号**、**MappedStatement 对象**、**参数类型**。

- `<2>` 处，调用 `#prepareParameterForNestedQuery(ResultSet rs, ResultMapping resultMapping, Class<?> parameterType, String columnPrefix)` 方法，获得内嵌查询的**参数对象**。代码如下：

  ```java
  // DefaultResultSetHandler.java
  
  // 获得内嵌查询的参数类型
  private Object prepareParameterForNestedQuery(ResultSet rs, ResultMapping resultMapping, Class<?> parameterType, String columnPrefix) throws SQLException {
      if (resultMapping.isCompositeResult()) { // ② 组合
          return prepareCompositeKeyParameter(rs, resultMapping, parameterType, columnPrefix);
      } else { // ① 普通
          return prepareSimpleKeyParameter(rs, resultMapping, parameterType, columnPrefix);
      }
  }
  
  // ① 获得普通类型的内嵌查询的参数对象
  private Object prepareSimpleKeyParameter(ResultSet rs, ResultMapping resultMapping, Class<?> parameterType, String columnPrefix) throws SQLException {
      // 获得 TypeHandler 对象
      final TypeHandler<?> typeHandler;
      if (typeHandlerRegistry.hasTypeHandler(parameterType)) {
          typeHandler = typeHandlerRegistry.getTypeHandler(parameterType);
      } else {
          typeHandler = typeHandlerRegistry.getUnknownTypeHandler();
      }
      // 获得指定字段的值
      return typeHandler.getResult(rs, prependPrefix(resultMapping.getColumn(), columnPrefix));
  }
  
  // ② 获得组合类型的内嵌查询的参数对象
  private Object prepareCompositeKeyParameter(ResultSet rs, ResultMapping resultMapping, Class<?> parameterType, String columnPrefix) throws SQLException {
      // 创建参数对象
      final Object parameterObject = instantiateParameterObject(parameterType);
      // 创建参数对象的 MetaObject 对象，可对其进行访问
      final MetaObject metaObject = configuration.newMetaObject(parameterObject);
      boolean foundValues = false;
      // 遍历组合的所有字段
      for (ResultMapping innerResultMapping : resultMapping.getComposites()) {
          // 获得属性类型
          final Class<?> propType = metaObject.getSetterType(innerResultMapping.getProperty());
          // 获得对应的 TypeHandler 对象
          final TypeHandler<?> typeHandler = typeHandlerRegistry.getTypeHandler(propType);
          // 获得指定字段的值
          final Object propValue = typeHandler.getResult(rs, prependPrefix(innerResultMapping.getColumn(), columnPrefix));
          // issue #353 & #560 do not execute nested query if key is null
          // 设置到 parameterObject 中，通过 metaObject
          if (propValue != null) {
              metaObject.setValue(innerResultMapping.getProperty(), propValue);
              foundValues = true; // 标记 parameterObject 非空对象
          }
      }
      // 返回参数对象
      return foundValues ? parameterObject : null;
  }
  
  // ② 创建参数对象
  private Object instantiateParameterObject(Class<?> parameterType) {
      if (parameterType == null) {
          return new HashMap<>();
      } else if (ParamMap.class.equals(parameterType)) {
          return new HashMap<>(); // issue #649
      } else {
          return objectFactory.create(parameterType);
      }
  }
  ```

  - 虽然代码比较长，但是非常简单。注意下，艿艿添加了 `①` 和 `②` 两个序号，分别对应两种情况。

- `<3>`处，整体是，执行查询，获得值。

  - `<3.1>` 处，调用 `MappedStatement#getBoundSql(Object parameterObject)` 方法，获得 BoundSql 对象。
  - `<3.2>` 处，创建 CacheKey 对象。
  - `<3.3>` 处，创建 ResultLoader 对象，并调用 `ResultLoader#loadResult()` 方法，加载结果。详细解析，见 [《精尽 MyBatis 源码分析 —— SQL 执行（五）之延迟加载》](http://svip.iocoder.cn/MyBatis/executor-5) 。

###### 3.1.2.3.2.3 applyAutomaticMappings

`#createResultObject(ResultSetWrapper rsw, ResultMap resultMap, List<Class<?>> constructorArgTypes, List<Object> constructorArgs, String columnPrefix)` 方法，创建映射后的结果对象。代码如下：[<-](#go3.1.2.3.2.1_6.2)

```java
// DefaultResultSetHandler.java

private boolean applyAutomaticMappings(ResultSetWrapper rsw, ResultMap resultMap, MetaObject metaObject, String columnPrefix) throws SQLException {
    // <1> 获得 UnMappedColumnAutoMapping 数组
    List<UnMappedColumnAutoMapping> autoMapping = createAutomaticMappings(rsw, resultMap, metaObject, columnPrefix);
    boolean foundValues = false;
    if (!autoMapping.isEmpty()) {
        // <2> 遍历 UnMappedColumnAutoMapping 数组
        for (UnMappedColumnAutoMapping mapping : autoMapping) {
            // 获得指定字段的值
            final Object value = mapping.typeHandler.getResult(rsw.getResultSet(), mapping.column);
            // 若非空，标记 foundValues 有值
            if (value != null) {
                foundValues = true;
            }
            // 设置到 parameterObject 中，通过 metaObject
            if (value != null || (configuration.isCallSettersOnNulls() && !mapping.primitive)) {
                // gcode issue #377, call setter on nulls (value is not 'found')
                metaObject.setValue(mapping.property, value);
            }
        }
    }
    return foundValues;
}
```

- `<1>` 处，调用 `#createAutomaticMappings(ResultSetWrapper rsw, ResultMap resultMap, MetaObject metaObject, String columnPrefix)` 方法，获得 UnMappedColumnAutoMapping 数组。代码如下：

  ```java
  // DefaultResultSetHandler.java
  
  private List<UnMappedColumnAutoMapping> createAutomaticMappings(ResultSetWrapper rsw, ResultMap resultMap, MetaObject metaObject, String columnPrefix) throws SQLException {
      // 生成 autoMappingsCache 的 KEY
      final String mapKey = resultMap.getId() + ":" + columnPrefix;
      // 从缓存 autoMappingsCache 中，获得 UnMappedColumnAutoMapping 数组
      List<UnMappedColumnAutoMapping> autoMapping = autoMappingsCache.get(mapKey);
      // 如果获取不到，则进行初始化
      if (autoMapping == null) {
          autoMapping = new ArrayList<>();
          // 获得未 mapped 的字段的名字的数组
          final List<String> unmappedColumnNames = rsw.getUnmappedColumnNames(resultMap, columnPrefix);
          // 遍历 unmappedColumnNames 数组
          for (String columnName : unmappedColumnNames) {
              // 获得属性名
              String propertyName = columnName;
              if (columnPrefix != null && !columnPrefix.isEmpty()) {
                  // When columnPrefix is specified,
                  // ignore columns without the prefix.
                  if (columnName.toUpperCase(Locale.ENGLISH).startsWith(columnPrefix)) {
                      propertyName = columnName.substring(columnPrefix.length());
                  } else {
                      continue;
                  }
              }
              // 从结果对象的 metaObject 中，获得对应的属性名
              final String property = metaObject.findProperty(propertyName, configuration.isMapUnderscoreToCamelCase());
              // 获得到属性名，并且可以进行设置
              if (property != null && metaObject.hasSetter(property)) {
                  // 排除已映射的属性
                  if (resultMap.getMappedProperties().contains(property)) {
                      continue;
                  }
                  // 获得属性的类型
                  final Class<?> propertyType = metaObject.getSetterType(property);
                  // 判断是否有对应的 TypeHandler 对象。如果有，则创建 UnMappedColumnAutoMapping 对象，并添加到 autoMapping 中
                  if (typeHandlerRegistry.hasTypeHandler(propertyType, rsw.getJdbcType(columnName))) {
                      final TypeHandler<?> typeHandler = rsw.getTypeHandler(propertyType, columnName);
                      autoMapping.add(new UnMappedColumnAutoMapping(columnName, property, typeHandler, propertyType.isPrimitive()));
                  // 如果没有，则执行 AutoMappingUnknownColumnBehavior 对应的逻辑
                  } else {
                      configuration.getAutoMappingUnknownColumnBehavior()
                              .doAction(mappedStatement, columnName, property, propertyType);
                  }
              // 如果没有属性，或者无法设置，则则执行 AutoMappingUnknownColumnBehavior 对应的逻辑
              } else {
                  configuration.getAutoMappingUnknownColumnBehavior()
                          .doAction(mappedStatement, columnName, (property != null) ? property : propertyName, null);
              }
          }
          // 添加到缓存中
          autoMappingsCache.put(mapKey, autoMapping);
      }
      return autoMapping;
  }
  ```

  - 虽然代码比较长，但是逻辑很简单。遍历未 mapped 的字段的名字的数组，映射每一个字段在**结果对象**的相同名字的属性，最终生成 UnMappedColumnAutoMapping 对象。

  - UnMappedColumnAutoMapping ，是 DefaultResultSetHandler 的内部静态类，未 mapped 字段自动映射后的对象。代码如下：

    ```java
    // DefaultResultSetHandler.java
    
    private static class UnMappedColumnAutoMapping {
    
        /**
         * 字段名
         */
        private final String column;
        /**
         * 属性名
         */
        private final String property;
        /**
         * TypeHandler 处理器
         */
        private final TypeHandler<?> typeHandler;
        /**
         * 是否为基本属性
         */
        private final boolean primitive;
    
        public UnMappedColumnAutoMapping(String column, String property, TypeHandler<?> typeHandler, boolean primitive) {
            this.column = column;
            this.property = property;
            this.typeHandler = typeHandler;
            this.primitive = primitive;
        }
    
    }
    ```

    - x

  - 当找不到映射的属性时，会调用 `AutoMappingUnknownColumnBehavior#doAction(MappedStatement mappedStatement, String columnName, String propertyName, Class<?> propertyType)` 方法，执行相应的逻辑。比较简单，胖友直接看 [`org.apache.ibatis.session.AutoMappingUnknownColumnBehavior`](https://github.com/YunaiV/mybatis-3/blob/master/src/main/java/org/apache/ibatis/session/AutoMappingUnknownColumnBehavior.java) 即可。

  - `Configuration.autoMappingUnknownColumnBehavior` 为 `AutoMappingUnknownColumnBehavior.NONE` ，即**不处理**。

- `<2>` 处，遍历 UnMappedColumnAutoMapping 数组，获得指定字段的值，设置到 `parameterObject` 中，通过 `metaObject` 。

###### 3.1.2.3.2.4 applyPropertyMappings

`#applyPropertyMappings(ResultSetWrapper rsw, ResultMap resultMap, MetaObject metaObject, ResultLoaderMap lazyLoader, String columnPrefix)` 方法，映射 ResultMap 中明确映射的列。代码如下： [<-](#go3.1.2.3.2.1_7)

```java
// DefaultResultSetHandler.java

private boolean applyPropertyMappings(ResultSetWrapper rsw, ResultMap resultMap, MetaObject metaObject, ResultLoaderMap lazyLoader, String columnPrefix)
        throws SQLException {
    // 获得 mapped 的字段的名字的数组
    final List<String> mappedColumnNames = rsw.getMappedColumnNames(resultMap, columnPrefix);
    boolean foundValues = false;
    // 遍历 ResultMapping 数组
    final List<ResultMapping> propertyMappings = resultMap.getPropertyResultMappings();
    for (ResultMapping propertyMapping : propertyMappings) {
        // 获得字段名
        String column = prependPrefix(propertyMapping.getColumn(), columnPrefix);
        if (propertyMapping.getNestedResultMapId() != null) {
            // the user added a column attribute to a nested result map, ignore it
            column = null;
        }
        if (propertyMapping.isCompositeResult() // 组合
                || (column != null && mappedColumnNames.contains(column.toUpperCase(Locale.ENGLISH))) // 属于 mappedColumnNames
                || propertyMapping.getResultSet() != null) { // 存储过程
            // <1> 获得指定字段的值
            Object value = getPropertyMappingValue(rsw.getResultSet(), metaObject, propertyMapping, lazyLoader, columnPrefix);
            // issue #541 make property optional
            final String property = propertyMapping.getProperty();
            if (property == null) {
                continue;
            // 存储过程相关，忽略
            } else if (value == DEFERED) {
                foundValues = true;
                continue;
            }
            // 标记获取到任一属性
            if (value != null) {
                foundValues = true;
            }
            // 设置到 parameterObject 中，通过 metaObject
            if (value != null || (configuration.isCallSettersOnNulls() && !metaObject.getSetterType(property).isPrimitive())) {
                // gcode issue #377, call setter on nulls (value is not 'found')
                metaObject.setValue(property, value);
            }
        }
    }
    return foundValues;
}
```

- 虽然代码比较长，但是逻辑很简单。胖友自己瞅瞅。

- 在 `<1>` 处，调用 `#getPropertyMappingValue(ResultSet rs, MetaObject metaResultObject, ResultMapping propertyMapping, ResultLoaderMap lazyLoader, String columnPrefix)` 方法，获得指定字段的值。代码如下：

  ```java
  // DefaultResultSetHandler.java
  
  private Object getPropertyMappingValue(ResultSet rs, MetaObject metaResultObject, ResultMapping propertyMapping, ResultLoaderMap lazyLoader, String columnPrefix)
          throws SQLException {
      // <2> 内嵌查询，获得嵌套查询的值
      if (propertyMapping.getNestedQueryId() != null) {
          return getNestedQueryMappingValue(rs, metaResultObject, propertyMapping, lazyLoader, columnPrefix);
      // 存储过程相关，忽略
      } else if (propertyMapping.getResultSet() != null) {
          addPendingChildRelation(rs, metaResultObject, propertyMapping);   // TODO is that OK?
          return DEFERED;
      // 普通，直接获得指定字段的值
      } else {
          final TypeHandler<?> typeHandler = propertyMapping.getTypeHandler();
          final String column = prependPrefix(propertyMapping.getColumn(), columnPrefix);
          return typeHandler.getResult(rs, column);
      }
  }
  ```

  - 在 `<2>` 处，我们又碰到了一个内嵌查询，调用 `#getNestedQueryMappingValue(ResultSet rs, MetaObject metaResultObject, ResultMapping propertyMapping, ResultLoaderMap lazyLoader, String columnPrefix)` 方法，获得嵌套查询的值。详细解析，见 [「」](http://svip.iocoder.cn/MyBatis/executor-4/#) 。

###### 3.1.2.3.2.5 getNestedQueryMappingValue 嵌套查询

`#getNestedQueryMappingValue(ResultSet rs, MetaObject metaResultObject, ResultMapping propertyMapping, ResultLoaderMap lazyLoader, String columnPrefix)` 方法，获得嵌套查询的值。代码如下：

```java
// DefaultResultSetHandler.java

private Object getNestedQueryMappingValue(ResultSet rs, MetaObject metaResultObject, ResultMapping propertyMapping, ResultLoaderMap lazyLoader, String columnPrefix)
        throws SQLException {
    // 获得内嵌查询的编号
    final String nestedQueryId = propertyMapping.getNestedQueryId();
    // 获得属性名
    final String property = propertyMapping.getProperty();
    // 获得内嵌查询的 MappedStatement 对象
    final MappedStatement nestedQuery = configuration.getMappedStatement(nestedQueryId);
    // 获得内嵌查询的参数类型
    final Class<?> nestedQueryParameterType = nestedQuery.getParameterMap().getType();
    // 获得内嵌查询的参数对象
    final Object nestedQueryParameterObject = prepareParameterForNestedQuery(rs, propertyMapping, nestedQueryParameterType, columnPrefix);
    Object value = null;
    if (nestedQueryParameterObject != null) {
        // 获得 BoundSql 对象
        final BoundSql nestedBoundSql = nestedQuery.getBoundSql(nestedQueryParameterObject);
        // 获得 CacheKey 对象
        final CacheKey key = executor.createCacheKey(nestedQuery, nestedQueryParameterObject, RowBounds.DEFAULT, nestedBoundSql);
        final Class<?> targetType = propertyMapping.getJavaType();
        // <1> 检查缓存中已存在
        if (executor.isCached(nestedQuery, key)) { //  有缓存
            // <2.1> 创建 DeferredLoad 对象，并通过该 DeferredLoad 对象从缓存中加载结采对象
            executor.deferLoad(nestedQuery, metaResultObject, property, key, targetType);
            // <2.2> 返回已定义
            value = DEFERED;
        // 检查缓存中不存在
        } else { // 无缓存
            // <3.1> 创建 ResultLoader 对象
            final ResultLoader resultLoader = new ResultLoader(configuration, executor, nestedQuery, nestedQueryParameterObject, targetType, key, nestedBoundSql);
            // <3.2> 如果要求延迟加载，则延迟加载
            if (propertyMapping.isLazy()) {
                // 如果该属性配置了延迟加载，则将其添加到 `ResultLoader.loaderMap` 中，等待真正使用时再执行嵌套查询并得到结果对象。
                lazyLoader.addLoader(property, metaResultObject, resultLoader);
                // 返回已定义
                value = DEFERED;
            // <3.3> 如果不要求延迟加载，则直接执行加载对应的值
            } else {
                value = resultLoader.loadResult();
            }
        }
    }
    return value;
}
```

- 和 [「3.1.2.3.2.2 getNestedQueryConstructorValue」](http://svip.iocoder.cn/MyBatis/executor-4/#) 一样，也是**嵌套查询**。所以，从整体代码的实现上，也是非常**类似**的。差别在于：
  - `#getNestedQueryConstructorValue(...)` 方法，用于构造方法需要用到的嵌套查询的值，它是**不用**考虑延迟加载的。
  - `#getNestedQueryMappingValue(...)` 方法，用于 setting 方法需要用到的嵌套查询的值，它是**需要**考虑延迟加载的。
- `<1>` 处，调用 `Executor#isCached(MappedStatement ms, CacheKey key)` 方法，检查缓存中已存在。下面，我们分成两种情况来解析。
- ========== 有缓存 ==========
- `<2.1>` 处，调用 `Executor#deferLoad(...)` 方法，创建 DeferredLoad 对象，并通过该 DeferredLoad 对象从缓存中加载结采对象。详细解析，见 [《精尽 MyBatis 源码分析 —— SQL 执行（五）之延迟加载》](http://svip.iocoder.cn/MyBatis/executor-5) 。
- `<2.2>` 处，返回已定义 `DEFERED` 。
- ========== 有缓存 ==========
- `<3.1>` 处，创建 ResultLoader 对象。
- `<3.2>`处，如果要求延迟加载，则延迟加载。
  - `<3.2.1>` 处，调用 `ResultLoader#addLoader(...)` 方法，如果该属性配置了延迟加载，则将其添加到 `ResultLoader.loaderMap` 中，等待真正使用时再执行嵌套查询并得到结果对象。详细解析，见 [《精尽 MyBatis 源码分析 —— SQL 执行（五）之延迟加载》](http://svip.iocoder.cn/MyBatis/executor-5) 。
- `<3.3>` 处，如果不要求延迟加载，则调用 `ResultLoader#loadResult()` 方法，直接执行加载对应的值。

##### 3.1.2.3.3 storeObject

`#storeObject(ResultHandler<?> resultHandler, DefaultResultContext<Object> resultContext, Object rowValue, ResultMapping parentMapping, ResultSet rs)` 方法，将映射创建的结果对象添加到 `ResultHandler.resultList` 中保存。代码如下： [<-](#go3.1.2.3.1_6)

```java
// DefaultResultSetHandler.java

private void storeObject(ResultHandler<?> resultHandler, DefaultResultContext<Object> resultContext, Object rowValue, ResultMapping parentMapping, ResultSet rs) throws SQLException {
    // 暂时忽略，这个情况，只有存储过程会出现
    if (parentMapping != null) {
        linkToParents(rs, parentMapping, rowValue);
    } else {
        callResultHandler(resultHandler, resultContext, rowValue);
    }
}

@SuppressWarnings("unchecked" /* because ResultHandler<?> is always ResultHandler<Object>*/)
// 调用 ResultHandler ，进行结果的处理
private void callResultHandler(ResultHandler<?> resultHandler, DefaultResultContext<Object> resultContext, Object rowValue) {
    // 设置结果对象到 resultContext 中
    resultContext.nextResultObject(rowValue);
    // <x> 使用 ResultHandler 处理结果。
    // 如果使用 DefaultResultHandler 实现类的情况，会将映射创建的结果对象添加到 ResultHandler.resultList 中保存
    ((ResultHandler<Object>) resultHandler).handleResult(resultContext);
}
```

- 逻辑比较简单，认真看下注释，特别是 `<x>` 处。

#### 3.1.2.4 handleRowValuesForNestedResultMap 嵌套映射

可能胖友对**嵌套映射**的概念不是很熟悉，胖友可以调试 `AncestorRefTest#testAncestorRef()` 这个单元测试方法。

------

> 老艿艿：本小节，还是建议看 [《MyBatis 技术内幕》](https://item.jd.com/12125531.html) 的 [「3.3.4 嵌套映射」](http://svip.iocoder.cn/MyBatis/executor-4/#) 小节。因为，它提供了比较好的这块逻辑的原理讲解，并且配置了大量的图。
>
> 😈 精力有限，后续补充哈。
> 😜 实际是，因为艿艿比较少用嵌套映射，所以对这块逻辑，不是很感兴趣。

`#handleRowValuesForNestedResultMap(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping)` 方法，处理嵌套**映射**的结果。代码如下：

```
// DefaultResultSetHandler.java
```

- TODO 9999 状态不太好，有点写不太明白。不感兴趣的胖友，可以跳过。感兴趣的胖友，可以看看 [《MyBatis 技术内幕》](https://item.jd.com/12125531.html) 的 [「3.3.4 嵌套映射」](http://svip.iocoder.cn/MyBatis/executor-4/#) 小节。

### 3.1.3 *handleCursorResultSets

`#handleCursorResultSets(Statement stmt)` 方法，处理 `java.sql.ResultSet` 成 Cursor 对象。代码如下：

```java
// DefaultResultSetHandler.java

@Override
public <E> Cursor<E> handleCursorResultSets(Statement stmt) throws SQLException {
    ErrorContext.instance().activity("handling cursor results").object(mappedStatement.getId());

    // 获得首个 ResultSet 对象，并封装成 ResultSetWrapper 对象
    ResultSetWrapper rsw = getFirstResultSet(stmt);

    // 游标方式的查询，只允许一个 ResultSet 对象。因此，resultMaps 数组的数量，元素只能有一个
    List<ResultMap> resultMaps = mappedStatement.getResultMaps();
    int resultMapCount = resultMaps.size();
    validateResultMapsCount(rsw, resultMapCount);
    if (resultMapCount != 1) {
        throw new ExecutorException("Cursor results cannot be mapped to multiple resultMaps");
    }

    // 获得 ResultMap 对象，后创建 DefaultCursor 对象
    ResultMap resultMap = resultMaps.get(0);
    return new DefaultCursor<>(this, resultMap, rsw, rowBounds); 
}
```

- 最终，创建成 DefaultCursor 对象。详细解析，见 [「6. DefaultCursor」](#6. DefaultCursor) 。
- 可能很多人没用 MyBatis Cursor 功能，所以可以看看 [《Mybatis 3.4.0 Cursor的使用》](https://www.jianshu.com/p/97d96201295b) 。

# 4. ResultContext

> 老艿艿：这个类，大体看看每个方法的用途，结合上文一起理解即可。

`org.apache.ibatis.session.ResultContext` ，结果上下文接口。代码如下：[<-](#go3.1.2.3_1)

```java
// ResultContext.java

public interface ResultContext<T> {

    /**
     * @return 当前结果对象
     */
    T getResultObject();

    /**
     * @return 总的结果对象的数量
     */
    int getResultCount();

    /**
     * @return 是否暂停
     */
    boolean isStopped();

    /**
     * 暂停
     */
    void stop();

}
```

## 4.1 DefaultResultContext

`org.apache.ibatis.executor.result.DefaultResultContext` ，实现 ResultContext 接口，默认的 ResultContext 的实现类。代码如下：

```java
// DefaultResultContext.java

public class DefaultResultContext<T> implements ResultContext<T> {

    /**
     * 当前结果对象
     */
    private T resultObject;
    /**
     * 总的结果对象的数量
     */
    private int resultCount;
    /**
     * 是否暂停
     */
    private boolean stopped;

    public DefaultResultContext() {
        resultObject = null;
        resultCount = 0;
        stopped = false; // 默认非暂停
    }

    @Override
    public T getResultObject() {
        return resultObject;
    }

    @Override
    public int getResultCount() {
        return resultCount;
    }

    @Override
    public boolean isStopped() {
        return stopped;
    }

    /**
     * 当前结果对象
     *
     * @param resultObject 当前结果对象
     */
    public void nextResultObject(T resultObject) {
        // 数量 + 1
        resultCount++;
        // 当前结果对象
        this.resultObject = resultObject;
    }

    @Override
    public void stop() {
        this.stopped = true;
    }

}
```

# 5. ResultHandler

`org.apache.ibatis.session.ResultHandler` ，结果处理器接口。代码如下：

```java
// ResultHandler.java

public interface ResultHandler<T> {

    /**
     * 处理当前结果
     *
     * @param resultContext ResultContext 对象。在其中，可以获得当前结果
     */
    void handleResult(ResultContext<? extends T> resultContext);

}
```

## 5.1 DefaultResultHandler

`org.apache.ibatis.executor.result.DefaultResultHandler` ，实现 ResultHandler 接口，默认的 ResultHandler 的实现类。代码如下：

```java
// DefaultResultHandler.java

public class DefaultResultHandler implements ResultHandler<Object> {

    /**
     * 结果数组
     */
    private final List<Object> list;

    public DefaultResultHandler() {
        list = new ArrayList<>();
    }

    @SuppressWarnings("unchecked")
    public DefaultResultHandler(ObjectFactory objectFactory) {
        list = objectFactory.create(List.class);
    }

    @Override
    public void handleResult(ResultContext<? extends Object> context) {
        // <1> 将当前结果，添加到结果数组中 
        list.add(context.getResultObject());
    }

    public List<Object> getResultList() {
        return list;
    }

}
```

- 核心代码就是 `<1>` 处，将当前结果，添加到结果数组中。

## 5.2 DefaultMapResultHandler

该类在 `session` 包中实现，我们放在**会话模块**的文章中，详细解析。

# 6. Cursor

`org.apache.ibatis.cursor.Cursor` ，继承 Closeable、Iterable 接口，游标接口。代码如下：

```java
// Cursor.java

public interface Cursor<T> extends Closeable, Iterable<T> {

    /**
     * 是否处于打开状态
     *
     * @return true if the cursor has started to fetch items from database.
     */
    boolean isOpen();

    /**
     * 是否全部消费完成
     *
     * @return true if the cursor is fully consumed and has returned all elements matching the query.
     */
    boolean isConsumed();

    /**
     * 获得当前索引
     *
     * Get the current item index. The first item has the index 0.
     * @return -1 if the first cursor item has not been retrieved. The index of the current item retrieved.
     */
    int getCurrentIndex();

}
```

## 6.1 DefaultCursor

`org.apache.ibatis.cursor.defaults.DefaultCursor` ，实现 Cursor 接口，默认 Cursor 实现类。

### 6.1.1 构造方法

```java
// DefaultCursor.java

// ResultSetHandler stuff
private final DefaultResultSetHandler resultSetHandler;
private final ResultMap resultMap;
private final ResultSetWrapper rsw;
private final RowBounds rowBounds;
/**
 * ObjectWrapperResultHandler 对象
 */
private final ObjectWrapperResultHandler<T> objectWrapperResultHandler = new ObjectWrapperResultHandler<>();

/**
 * CursorIterator 对象，游标迭代器。
 */
private final CursorIterator cursorIterator = new CursorIterator();
/**
 * 是否开始迭代
 *
 * {@link #iterator()}
 */
private boolean iteratorRetrieved;
/**
 * 游标状态
 */
private CursorStatus status = CursorStatus.CREATED;
/**
 * 已完成映射的行数
 */
private int indexWithRowBound = -1;

public DefaultCursor(DefaultResultSetHandler resultSetHandler, ResultMap resultMap, ResultSetWrapper rsw, RowBounds rowBounds) {
    this.resultSetHandler = resultSetHandler;
    this.resultMap = resultMap;
    this.rsw = rsw;
    this.rowBounds = rowBounds;
}
```

- 大体瞄下每个属性的意思。下面，每个方法，胖友会更好的理解每个属性。

### 6.1.2 CursorStatus

CursorStatus ，是 DefaultCursor 的内部枚举类。代码如下：

```java
// DefaultCursor.java

private enum CursorStatus {

    /**
     * A freshly created cursor, database ResultSet consuming has not started
     */
    CREATED,
    /**
     * A cursor currently in use, database ResultSet consuming has started
     */
    OPEN,
    /**
     * A closed cursor, not fully consumed
     *
     * 已关闭，并未完全消费
     */
    CLOSED,
    /**
     * A fully consumed cursor, a consumed cursor is always closed
     *
     * 已关闭，并且完全消费
     */
    CONSUMED
}
```

### 6.1.3 isOpen

```java
// DefaultCursor.java

@Override
public boolean isOpen() {
    return status == CursorStatus.OPEN;
}
```

### 6.1.4 isConsumed

```java
// DefaultCursor.java

@Override
public boolean isConsumed() {
    return status == CursorStatus.CONSUMED;
}
```

### 6.1.5 iterator

`#iterator()` 方法，获取迭代器。代码如下：

```java
// DefaultCursor.java

@Override
public Iterator<T> iterator() {
    // 如果已经获取，则抛出 IllegalStateException 异常
    if (iteratorRetrieved) {
        throw new IllegalStateException("Cannot open more than one iterator on a Cursor");
    }
    if (isClosed()) {
        throw new IllegalStateException("A Cursor is already closed.");
    }
    // 标记已经获取
    iteratorRetrieved = true;
    return cursorIterator;
}
```

- 通过 `iteratorRetrieved` 属性，保证有且仅返回一次 `cursorIterator` 对象。

### 6.1.6 ObjectWrapperResultHandler

ObjectWrapperResultHandler ，DefaultCursor 的内部静态类，实现 ResultHandler 接口，代码如下：

```java
// DefaultCursor.java

private static class ObjectWrapperResultHandler<T> implements ResultHandler<T> {

    /**
     * 结果对象
     */
    private T result;

    @Override
    public void handleResult(ResultContext<? extends T> context) {
        // <1> 设置结果对象
        this.result = context.getResultObject();
        // <2> 暂停
        context.stop();
    }

}
```

- `<1>` 处，暂存 [「3.1 DefaultResultSetHandler」](3.1 DefaultResultSetHandler) 处理的 ResultSet 的**当前行**的结果。

- `<2>` 处，通过调用 `ResultContext#stop()` 方法，暂停 DefaultResultSetHandler 在向下遍历下一条记录，从而实现每次在调用 `CursorIterator#hasNext()` 方法，只遍历一行 ResultSet 的记录。如果胖友有点懵逼，可以在看看 `DefaultResultSetHandler#shouldProcessMoreRows(ResultContext<?> context, RowBounds rowBounds)` 方法，代码如下：

  ```java
  // DefaultCursor.java
  
  private boolean shouldProcessMoreRows(ResultContext<?> context, RowBounds rowBounds) {
      return !context.isStopped() && context.getResultCount() < rowBounds.getLimit();
  }
  ```

### 6.1.7 CursorIterator

CursorIterator ，DefaultCursor 的内部类，实现 Iterator 接口，游标的迭代器实现类。代码如下：

```java
// DefaultCursor.java

private class CursorIterator implements Iterator<T> {

    /**
     * Holder for the next object to be returned
     *
     * 结果对象，提供给 {@link #next()} 返回
     */
    T object;

    /**
     * Index of objects returned using next(), and as such, visible to users.
     * 索引位置
     */
    int iteratorIndex = -1;

    @Override
    public boolean hasNext() {
        // <1> 如果 object 为空，则遍历下一条记录
        if (object == null) {
            object = fetchNextUsingRowBound();
        }
        // <2> 判断 object 是否非空
        return object != null;
    }

    @Override
    public T next() {
        // <3> Fill next with object fetched from hasNext()
        T next = object;

        // <4> 如果 next 为空，则遍历下一条记录
        if (next == null) {
            next = fetchNextUsingRowBound();
        }

        // <5> 如果 next 非空，说明有记录，则进行返回
        if (next != null) {
            // <5.1> 置空 object 对象
            object = null;
            // <5.2> 增加 iteratorIndex
            iteratorIndex++;
            // <5.3> 返回 next
            return next;
        }

        // <6> 如果 next 为空，说明没有记录，抛出 NoSuchElementException 异常
        throw new NoSuchElementException();
    }

    @Override
    public void remove() {
        throw new UnsupportedOperationException("Cannot remove element from Cursor");
    }

}
```

- `#hasNext()`方法，判断是否有**下一个**结果对象：
  - `<1>` 处，如果 `object` 为空，则调用 `#fetchNextUsingRowBound()` 方法，遍历下一条记录。也就是说，该方法是先获取下一条记录，然后在判断是否存在下一条记录。实际上，和 `java.util.ResultSet` 的方式，是一致的。如果再调用一次该方法，则不会去遍历下一条记录。关于 `#fetchNextUsingRowBound()` 方法，详细解析，见 [「6.1.8 fetchNextUsingRowBound」](#6.1.8 fetchNextUsingRowBound) 。
  - `<2>` 处，判断 `object` 非空。

- `#next()`方法，获得下一个结果对象：
  - `<3>` 处，先记录 `object` 到 `next` 中。为什么要这么做呢？继续往下看。
  
  - `<4>` 处，如果 `next` 为空，有两种可能性：1）使用方未调用 `#hasNext()` 方法；2）调用 `#hasNext()` 方法，发现没下一条，还是调用了 `#next()` 方法。如果 `next()` 方法为空，通过“再次”调用 `#fetchNextUsingRowBound()` 方法，去遍历下一条记录。
  
  - `<5>`处，如果`next`非空，说明有记录，则进行返回。
    - `<5.1>` 处，置空 `object` 对象。
    - `<5.2>` 处，增加 `iteratorIndex` 。
    - `<5.3>` 处，返回 `next` 。如果 `<3>` 处，不进行 `next` 的赋值，如果 `<5.1>` 处的置空，此处就无法返回 `next` 了。
    
  - `<6>` 处，如果 `next` 为空，说明没有记录，抛出 NoSuchElementException 异常。

### 6.1.8 fetchNextUsingRowBound

`#fetchNextUsingRowBound()` 方法，遍历下一条记录。代码如下：

```java
// DefaultCursor.java

protected T fetchNextUsingRowBound() {
    // <1> 遍历下一条记录
    T result = fetchNextObjectFromDatabase();
    // 循环跳过 rowBounds 的索引
    while (result != null && indexWithRowBound < rowBounds.getOffset()) {
        result = fetchNextObjectFromDatabase();
    }
    // 返回记录
    return result;
}
```

- `<1>` 处，调用 `#fetchNextObjectFromDatabase()` 方法，遍历下一条记录。代码如下：

```java
// DefaultCursor.java

protected T fetchNextObjectFromDatabase() {
    // <1> 如果已经关闭，返回 null
    if (isClosed()) {
        return null;
    }

    try {
        // <2> 设置状态为 CursorStatus.OPEN
        status = CursorStatus.OPEN;
        // <3> 遍历下一条记录
        if (!rsw.getResultSet().isClosed()) {
            resultSetHandler.handleRowValues(rsw, resultMap, objectWrapperResultHandler, RowBounds.DEFAULT, null);
        }
    } catch (SQLException e) {
        throw new RuntimeException(e);
    }

    // <4> 复制给 next
    T next = objectWrapperResultHandler.result;
    // <5> 增加 indexWithRowBound
    if (next != null) {
        indexWithRowBound++;
    }
    // No more object or limit reached
    // <6> 没有更多记录，或者到达 rowBounds 的限制索引位置，则关闭游标，并设置状态为 CursorStatus.CONSUMED
    if (next == null || getReadItemsCount() == rowBounds.getOffset() + rowBounds.getLimit()) {
        close();
        status = CursorStatus.CONSUMED;
    }
    // <7> 置空 objectWrapperResultHandler.result 属性
    objectWrapperResultHandler.result = null;
    // <8> 返回下一条结果
    return next;
}
```

- `<1>` 处，调用 `#isClosed()` 方法，判断是否已经关闭。若是，则返回 `null` 。代码如下：

  ```java
  // DefaultCursor.java
  
  private boolean isClosed() {
      return status == CursorStatus.CLOSED || status == CursorStatus.CONSUMED;
  }
  ```

- `<2>` 处，设置状态为 `CursorStatus.OPEN` 。

- `<3>` 处，调用 `DefaultResultSetHandler#handleRowValues(...)` 方法，遍历下一条记录。也就是说，回到了 [[「3.1.2.2 handleRowValues」](http://svip.iocoder.cn/MyBatis/executor-4/#) 的流程。遍历的下一条件记录，会暂存到 `objectWrapperResultHandler.result` 中。

- `<4>` 处，复制给 `next` 。

- `<5>` 处，增加 `indexWithRowBound` 。

- `<6>` 处，没有更多记录，或者到达 `rowBounds` 的限制索引位置，则关闭游标，并设置状态为 `CursorStatus.CONSUMED` 。其中，涉及到的方法，代码如下：

  ```java
  // DefaultCursor.java
  
  private int getReadItemsCount() {
      return indexWithRowBound + 1;
  }
  
  @Override
  public void close() {
      if (isClosed()) {
          return;
      }
  
      // 关闭 ResultSet
      ResultSet rs = rsw.getResultSet();
      try {
          if (rs != null) {
              rs.close();
          }
      } catch (SQLException e) {
          // ignore
      } finally {
          // 设置状态为 CursorStatus.CLOSED
          status = CursorStatus.CLOSED;
      }
  }
  ```

- `<7>` 处，置空 `objectWrapperResultHandler.result` 属性。
- `<8>` 处，返回 `next` 。

# 666. 彩蛋

卧槽，太他喵的长了。写的我的 MWeb( 写作编辑器 )都一卡一卡的。同时，写的有点崩溃，可能有些细节描述的不到位，如果有错误，或者不好理解的地方，记得给我星球留言，多多交流。

参考和推荐如下文章：

- 祖大俊 [《Mybatis3.4.x技术内幕（二十一）：参数设置、结果封装、级联查询、延迟加载原理分析》](https://my.oschina.net/zudajun/blog/747283)
- 徐郡明 [《MyBatis 技术内幕》](https://item.jd.com/12125531.html) 的 [「3.3 ResultSetHandler」](http://svip.iocoder.cn/MyBatis/executor-4/#) 小节