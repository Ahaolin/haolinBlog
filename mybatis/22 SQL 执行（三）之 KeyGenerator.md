# 精尽 MyBatis 源码分析 —— SQL 执行（三）之 KeyGenerator

## 1. 概述

本文，我们来分享 SQL 执行的第三部分，`keygen` 包。整体类图如下：[![类图](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202201290619174.png)](http://static.iocoder.cn/images/MyBatis/2020_03_06/01.png)类图

- 我们可以看到，整体是以 KeyGenerator 为核心。所以，本文主要会看到的就是 KeyGenerator 对**自增主键**的获取。

## 2. KeyGenerator

`org.apache.ibatis.executor.keygen.KeyGenerator` ，主键生成器接口。代码如下：

```java
// KeyGenerator.java

public interface KeyGenerator {

    // SQL 执行前
    void processBefore(Executor executor, MappedStatement ms, Statement stmt, Object parameter);

    // SQL 执行后
    void processAfter(Executor executor, MappedStatement ms, Statement stmt, Object parameter);

}
```

- 可在 SQL 执行**之前**或**之后**，进行处理主键的生成。

- 实际上，KeyGenerator 类的命名虽然包含 Generator ，但是目前 MyBatis 默认的 KeyGenerator 实现类，都是基于数据库来实现**主键自增**的功能。

- `parameter` 参数，指的是什么呢？以下面的方法为示例：

  ```java
  @Options(useGeneratedKeys = true, keyProperty = "id")
  @Insert({"insert into country (countryname,countrycode) values (#{countryname},#{countrycode})"})
  int insertBean(Country country);
  ```

  - 上面的，`country` 方法参数，就是一个 `parameter` 参数。
  - **KeyGenerator 在获取到主键后，会设置回 `parameter` 参数的对应属性**。

KeyGenerator 有三个子类，如下图所示：[![类图](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202201290619174.png)](http://static.iocoder.cn/images/MyBatis/2020_03_06/01.png)类图

- 具体的，我们下面逐小节来分享。

## 3. Jdbc3KeyGenerator

`org.apache.ibatis.executor.keygen.Jdbc3KeyGenerator` ，实现 KeyGenerator 接口，基于 `Statement#getGeneratedKeys()` 方法的 KeyGenerator 实现类，适用于 MySQL、H2 主键生成。

### 3.1 构造方法

```java
// Jdbc3KeyGenerator.java

/**
 * A shared instance.
 *
 * 共享的单例
 *
 * @since 3.4.3
 */
public static final Jdbc3KeyGenerator INSTANCE = new Jdbc3KeyGenerator();
```

- 单例。

### 3.2 processBefore

```java
@Override
public void processBefore(Executor executor, MappedStatement ms, Statement stmt, Object parameter) {
    // do nothing
}
```

- 空实现。因为对于 Jdbc3KeyGenerator 类的主键，是在 SQL 执行后，才生成。

### 3.3 processAfter

```java
// Jdbc3KeyGenerator.java

@Override
public void processAfter(Executor executor, MappedStatement ms, Statement stmt, Object parameter) {
    processBatch(ms, stmt, parameter);
}
```

- 调用 `#processBatch(Executor executor, MappedStatement ms, Statement stmt, Object parameter)` 方法，处理返回的自增主键。单个 `parameter` 参数，可以认为是批量的一个**特例**。

### 3.4 processBatch

```java
// Jdbc3KeyGenerator.java

public void processBatch(MappedStatement ms, Statement stmt, Object parameter) {
    // <1> 获得主键属性的配置。如果为空，则直接返回，说明不需要主键
    final String[] keyProperties = ms.getKeyProperties();
    if (keyProperties == null || keyProperties.length == 0) {
        return;
    }
    ResultSet rs = null;
    try {
        // <2> 获得返回的自增主键
        rs = stmt.getGeneratedKeys();
        final Configuration configuration = ms.getConfiguration();
        if (rs.getMetaData().getColumnCount() >= keyProperties.length) {
            // <3> 获得唯一的参数对象
            Object soleParam = getSoleParameter(parameter);
            if (soleParam != null) {
                // <3.1> 设置主键们，到参数 soleParam 中
                assignKeysToParam(configuration, rs, keyProperties, soleParam);
            } else {
                // <3.2> 设置主键们，到参数 parameter 中
                assignKeysToOneOfParams(configuration, rs, keyProperties, (Map<?, ?>) parameter);
            }
        }
    } catch (Exception e) {
        throw new ExecutorException("Error getting generated key or setting result to parameter object. Cause: " + e, e);
    } finally {
        // <4> 关闭 ResultSet 对象
        if (rs != null) {
            try {
                rs.close();
            } catch (Exception e) {
                // ignore
            }
        }
    }
}
```

- `<1>` 处，获得主键属性的配置。如果为空，则直接返回，说明不需要主键。
- 【重要】`<2>` 处，调用 `Statement#getGeneratedKeys()` 方法，获得返回的自增主键。
- `<3>`处，调用`#getSoleParameter(Object parameter)`方法，获得唯一的参数对象。详细解析，先跳到[「3.4.1 getSoleParameter」](#3.4.1 getSoleParameter) 。
  - `<3.1>` 处，调用 `#assignKeysToParam(...)` 方法，设置主键们，到参数 `soleParam` 中。详细解析，见 [「3.4.2 assignKeysToParam」](#3.4.2 assignKeysToParam) 。
  - `<3.2>` 处，调用 `#assignKeysToOneOfParams(...)` 方法，设置主键们，到参数 `parameter` 中。详细解析，见 [「3.4.3 assignKeysToOneOfParams」](#3.4.3 assignKeysToOneOfParams) 。
- `<4>` 处，关闭 ResultSet 对象。

#### 3.4.1 getSoleParameter

```java
// Jdbc3KeyGenerator.java

/**
 * 获得唯一的参数对象
 *
 * 如果获得不到唯一的参数对象，则返回 null
 *
 * @param parameter 参数对象
 * @return 唯一的参数对象
 */
private Object getSoleParameter(Object parameter) {
    // <1> 如果非 Map 对象，则直接返回 parameter
    if (!(parameter instanceof ParamMap || parameter instanceof StrictMap)) {
        return parameter;
    }
    // <3> 如果是 Map 对象，则获取第一个元素的值
    // <2> 如果有多个元素，则说明获取不到唯一的参数对象，则返回 null
    Object soleParam = null;
    for (Object paramValue : ((Map<?, ?>) parameter).values()) {
        if (soleParam == null) {
            soleParam = paramValue;
        } else if (soleParam != paramValue) {
            soleParam = null;
            break;
        }
    }
    return soleParam;
}
```

- `<1>` 处，如下可以符合这个条件。代码如下：

  ```java
  @Options(useGeneratedKeys = true, keyProperty = "id")
  @Insert({"insert into country (countryname,countrycode) values (#{country.countryname},#{country.countrycode})"})
  int insertNamedBean(@Param("country") Country country);
  ```

- `<2>` 处，如下可以符合这个条件。代码如下：

  ```java
  @Options(useGeneratedKeys = true, keyProperty = "country.id")
  @Insert({"insert into country (countryname, countrycode) values (#{country.countryname}, #{country.countrycode})"})
  int insertMultiParams_keyPropertyWithWrongParamName2(@Param("country") Country country,
                                                       @Param("someId") Integer someId);
  ```

  - 虽然有 `country` 和 `someId` 参数，但是最终会被封装成一个 `parameter` 参数，类型为 ParamMap 类型。为什么呢？答案在 `ParamNameResolver#getNamedParams(Object[] args)` 方法中。
  - 如果是这个情况，获得的主键，会设置回 `country` 的 `id` 属性，因为注解上的 `keyProperty = "country.id"` 配置。
  - 😈 此处比较绕，也相对用的少。

- `<3>` 处，如下可以符合这个条件。代码如下：

  ```java
  @Options(useGeneratedKeys = true, keyProperty = "id")
  @Insert({"insert into country (countryname, countrycode) values (#{country.countryname}, #{country.countrycode})"})
  int insertMultiParams_keyPropertyWithWrongParamName3(@Param("country") Country country);
  ```

  - 相比 `<2>` 的示例，主要是 `keyProperty = "id"` 的修改，和去掉了 `@Param("someId") Integer someId` 参数。
  - 实际上，这种情况，和 `<1>` 是类似的。

- 三种情况，`<2>` 和 `<3>` 有点复杂，胖友实际上，理解 `<1>` 即可。

#### 3.4.2 assignKeysToParam

```java
// Jdbc3KeyGenerator.java

private void assignKeysToParam(final Configuration configuration, ResultSet rs, final String[] keyProperties, Object param)
        throws SQLException {
    final TypeHandlerRegistry typeHandlerRegistry = configuration.getTypeHandlerRegistry();
    final ResultSetMetaData rsmd = rs.getMetaData();
    // Wrap the parameter in Collection to normalize the logic.
    // <1> 包装成 Collection 对象
    Collection<?> paramAsCollection;
    if (param instanceof Object[]) {
        paramAsCollection = Arrays.asList((Object[]) param);
    } else if (!(param instanceof Collection)) {
        paramAsCollection = Collections.singletonList(param);
    } else {
        paramAsCollection = (Collection<?>) param;
    }
    TypeHandler<?>[] typeHandlers = null;
    // <2> 遍历 paramAsCollection 数组
    for (Object obj : paramAsCollection) {
        // <2.1> 顺序遍历 rs
        if (!rs.next()) {
            break;
        }
        // <2.2> 创建 MetaObject 对象
        MetaObject metaParam = configuration.newMetaObject(obj);
        // <2.3> 获得 TypeHandler 数组
        if (typeHandlers == null) {
            typeHandlers = getTypeHandlers(typeHandlerRegistry, metaParam, keyProperties, rsmd);
        }
        // <2.4> 填充主键们
        populateKeys(rs, metaParam, keyProperties, typeHandlers);
    }
}
```

- `<1>` 处，包装成 Collection 对象。通过这样的方式，使单个 `param` 参数的情况下，可以统一。

- `<2>` 处，遍历 `paramAsCollection` 数组：

  - `<2.1>` 处， 顺序遍历 `rs` ，相当于把当前的 ResultSet 对象的主键们，赋值给 `obj` 对象的对应属性。

  - `<2.2>` 处，创建 MetaObject 对象，实现对 `obj` 对象的属性访问。

  - `<2.3>` 处，调用 `#getTypeHandlers(...)` 方法，获得 TypeHandler 数组。代码如下：

    ```java
    // Jdbc3KeyGenerator.java
    
    private TypeHandler<?>[] getTypeHandlers(TypeHandlerRegistry typeHandlerRegistry, MetaObject metaParam, String[] keyProperties, ResultSetMetaData rsmd) throws SQLException {
        // 获得主键们，对应的每个属性的，对应的 TypeHandler 对象
        TypeHandler<?>[] typeHandlers = new TypeHandler<?>[keyProperties.length];
        for (int i = 0; i < keyProperties.length; i++) {
            if (metaParam.hasSetter(keyProperties[i])) {
                Class<?> keyPropertyType = metaParam.getSetterType(keyProperties[i]);
                typeHandlers[i] = typeHandlerRegistry.getTypeHandler(keyPropertyType, JdbcType.forCode(rsmd.getColumnType(i + 1)));
            } else {
                throw new ExecutorException("No setter found for the keyProperty '" + keyProperties[i] + "' in '"
                        + metaParam.getOriginalObject().getClass().getName() + "'.");
            }
        }
        return typeHandlers;
    }
    ```

    - x

  - `<2.4>` 处，调用 `#populateKeys(...)` 方法，填充主键们。详细解析，见 [「3.5 populateKeys」](http://svip.iocoder.cn/MyBatis/executor-3/#) 。

#### 3.4.3 assignKeysToOneOfParams

```java
// Jdbc3KeyGenerator.java

protected void assignKeysToOneOfParams(final Configuration configuration, ResultSet rs, final 	     String[] keyProperties,Map<?, ?> paramMap) throws SQLException {
    // Assuming 'keyProperty' includes the parameter name. e.g. 'param.id'.
    // <1> 需要有 `.` 。
    int firstDot = keyProperties[0].indexOf('.');
    if (firstDot == -1) {
        throw new ExecutorException(
                "Could not determine which parameter to assign generated keys to. "
                        + "Note that when there are multiple parameters, 'keyProperty' must include the parameter name (e.g. 'param.id'). "
                        + "Specified key properties are " + ArrayUtil.toString(keyProperties) + " and available parameters are "
                        + paramMap.keySet());
    }
    // 获得真正的参数值
    String paramName = keyProperties[0].substring(0, firstDot);
    Object param;
    if (paramMap.containsKey(paramName)) {
        param = paramMap.get(paramName);
    } else {
        throw new ExecutorException("Could not find parameter '" + paramName + "'. "
                + "Note that when there are multiple parameters, 'keyProperty' must include the parameter name (e.g. 'param.id'). "
                + "Specified key properties are " + ArrayUtil.toString(keyProperties) + " and available parameters are "
                + paramMap.keySet());
    }
    // Remove param name from 'keyProperty' string. e.g. 'param.id' -> 'id'
    // 获得主键的属性的配置
    String[] modifiedKeyProperties = new String[keyProperties.length];
    for (int i = 0; i < keyProperties.length; i++) {
        if (keyProperties[i].charAt(firstDot) == '.' && keyProperties[i].startsWith(paramName)) {
            modifiedKeyProperties[i] = keyProperties[i].substring(firstDot + 1);
        } else {
            throw new ExecutorException("Assigning generated keys to multiple parameters is not supported. "
                    + "Note that when there are multiple parameters, 'keyProperty' must include the parameter name (e.g. 'param.id'). "
                    + "Specified key properties are " + ArrayUtil.toString(keyProperties) + " and available parameters are "
                    + paramMap.keySet());
        }
    }
    // 设置主键们，到参数 param 中
    assignKeysToParam(configuration, rs, modifiedKeyProperties, param);
}
```

- `<1>` 处，需要有 `.` 。例如：`@Options(useGeneratedKeys = true, keyProperty = "country.id")` 。
- `<2>` 处，获得真正的参数值。
- `<3>` 处，获得主键的属性的配置。
- `<4>` 处，调用 `#assignKeysToParam(...)` 方法，设置主键们，到参数 `param` 中。所以，后续流程，又回到了 [「3.4.2」](http://svip.iocoder.cn/MyBatis/executor-3/#) 咧。
- 关于这个方法，胖友自己模拟下这个情况，调试下会比较好理解。😈 当然，也可以不理解，嘿嘿。

### 3.5 populateKeys

```java
// Jdbc3KeyGenerator.java

private void populateKeys(ResultSet rs, MetaObject metaParam, String[] keyProperties, TypeHandler<?>[] typeHandlers) throws SQLException {
    // 遍历 keyProperties
    for (int i = 0; i < keyProperties.length; i++) {
        // 获得属性名
        String property = keyProperties[i];
        // 获得 TypeHandler 对象
        TypeHandler<?> th = typeHandlers[i];
        if (th != null) {
            // 从 rs 中，获得对应的 值
            Object value = th.getResult(rs, i + 1);
            // 设置到 metaParam 的对应 property 属性种
            metaParam.setValue(property, value);
        }
    }
}
```

- 代码比较简单，胖友看下注释即可。

## 4. SelectKeyGenerator

`org.apache.ibatis.executor.keygen.SelectKeyGenerator` ，实现 KeyGenerator 接口，基于从数据库查询主键的 KeyGenerator 实现类，适用于 Oracle、PostgreSQL 。

### 4.1 构造方法

```java
// SelectKeyGenerator.java

    public static final String SELECT_KEY_SUFFIX = "!selectKey";

/**
 * 是否在 before 阶段执行
 *
 * true ：before
 * after ：after
 */
private final boolean executeBefore;
/**
 * MappedStatement 对象
 */
private final MappedStatement keyStatement;

public SelectKeyGenerator(MappedStatement keyStatement, boolean executeBefore) {
    this.executeBefore = executeBefore;
    this.keyStatement = keyStatement;
}
```

### 4.2 processBefore

```java
// SelectKeyGenerator.java

@Override
public void processBefore(Executor executor, MappedStatement ms, Statement stmt, Object parameter) {
    if (executeBefore) {
        processGeneratedKeys(executor, ms, parameter);
    }
}
```

- 调用 `#processGeneratedKeys(...)` 方法。

### 4.3 processAfter

```java
// SelectKeyGenerator.java

@Override
public void processAfter(Executor executor, MappedStatement ms, Statement stmt, Object parameter) {
    if (!executeBefore) {
        processGeneratedKeys(executor, ms, parameter);
    }
}
```

- 也是调用 `#processGeneratedKeys(...)` 方法。

### 4.4 processGeneratedKeys

```java
// SelectKeyGenerator.java

private void processGeneratedKeys(Executor executor, MappedStatement ms, Object parameter) {
    try {
        // <1> 有查询主键的 SQL 语句，即 keyStatement 对象非空
        if (parameter != null && keyStatement != null && keyStatement.getKeyProperties() != null) {
            String[] keyProperties = keyStatement.getKeyProperties();
            final Configuration configuration = ms.getConfiguration();
            final MetaObject metaParam = configuration.newMetaObject(parameter);
            // Do not close keyExecutor.
            // The transaction will be closed by parent executor.
            // <2> 创建执行器，类型为 SimpleExecutor
            Executor keyExecutor = configuration.newExecutor(executor.getTransaction(), ExecutorType.SIMPLE);
            // <3> 执行查询主键的操作
            List<Object> values = keyExecutor.query(keyStatement, parameter, RowBounds.DEFAULT, Executor.NO_RESULT_HANDLER);
            // <4.1> 查不到结果，抛出 ExecutorException 异常
            if (values.size() == 0) {
                throw new ExecutorException("SelectKey returned no data.");
            // <4.2> 查询的结果过多，抛出 ExecutorException 异常
            } else if (values.size() > 1) {
                throw new ExecutorException("SelectKey returned more than one value.");
            } else {
                // <4.3> 创建 MetaObject 对象，访问查询主键的结果
                MetaObject metaResult = configuration.newMetaObject(values.get(0));
                // <4.3.1> 单个主键
                if (keyProperties.length == 1) {
                    // 设置属性到 metaParam 中，相当于设置到 parameter 中
                    if (metaResult.hasGetter(keyProperties[0])) {
                        setValue(metaParam, keyProperties[0], metaResult.getValue(keyProperties[0]));
                    } else {
                        // no getter for the property - maybe just a single value object
                        // so try that
                        setValue(metaParam, keyProperties[0], values.get(0));
                    }
                // <4.3.2> 多个主键
                } else {
                    // 遍历，进行赋值
                    handleMultipleProperties(keyProperties, metaParam, metaResult);
                }
            }
        }
    } catch (ExecutorException e) {
        throw e;
    } catch (Exception e) {
        throw new ExecutorException("Error selecting key or setting result to parameter object. Cause: " + e, e);
    }
}
```

- `<1>` 处，有查询主键的 SQL 语句，即 `keyStatement` 对象非空。

- `<2>` 处，创建执行器，类型为 SimpleExecutor 。

- 【重要】 `<3>` 处，调用 `Executor#query(...)` 方法，执行查询主键的操作。😈 简单脑暴下，按照 SelectKeyGenerator 的思路，岂不是可以可以接入 SnowFlake 算法，从而实现分布式主键。

- `<4.1>` 处，查不到结果，抛出 ExecutorException 异常。

- `<4.2>` 处，查询的结果过多，抛出 ExecutorException 异常。

- `<4.3>` 处，创建 MetaObject 对象，访问查询主键的结果。

  - `<4.3.1>` 处，**单个主键**，调用 `#setValue(MetaObject metaParam, String property, Object value)` 方法，设置属性到 `metaParam` 中，相当于设置到 `parameter` 中。代码如下：

    ```java
    // SelectKeyGenerator.java
    
    private void setValue(MetaObject metaParam, String property, Object value) {
        if (metaParam.hasSetter(property)) {
            metaParam.setValue(property, value);
        } else {
            throw new ExecutorException("No setter found for the keyProperty '" + property + "' in " + metaParam.getOriginalObject().getClass().getName() + ".");
        }
    }
    ```

    - 简单，胖友自己瞅瞅。

  - `<4.3.2>` 处，**多个主键**，调用 `#handleMultipleProperties(String[] keyProperties, MetaObject metaParam, MetaObject metaResult)` 方法，遍历，进行赋值。代码如下：

    ```java
    // SelectKeyGenerator.java
    
    private void handleMultipleProperties(String[] keyProperties,
                                          MetaObject metaParam, MetaObject metaResult) {
        String[] keyColumns = keyStatement.getKeyColumns();
        // 遍历，进行赋值
        if (keyColumns == null || keyColumns.length == 0) {
            // no key columns specified, just use the property names
            for (String keyProperty : keyProperties) {
                setValue(metaParam, keyProperty, metaResult.getValue(keyProperty));
            }
        } else {
            if (keyColumns.length != keyProperties.length) {
                throw new ExecutorException("If SelectKey has key columns, the number must match the number of key properties.");
            }
            for (int i = 0; i < keyProperties.length; i++) {
                setValue(metaParam, keyProperties[i], metaResult.getValue(keyColumns[i]));
            }
        }
    }
    ```

    - 最终，还是会调用 `#setValue(...)` 方法，进行赋值。

### **4.5 示例

- [《MyBatis + Oracle 实现主键自增长的几种常用方式》](https://blog.csdn.net/wal1314520/article/details/77132305)
- [《mybatis + postgresql 返回递增主键的正确姿势及勘误》](https://blog.csdn.net/cdnight/article/details/72735108)

## 5. NoKeyGenerator

`org.apache.ibatis.executor.keygen.NoKeyGenerator` ，实现 KeyGenerator 接口，空的 KeyGenerator 实现类，即无需主键生成。代码如下：

```java
// NoKeyGenerator.java

public class NoKeyGenerator implements KeyGenerator {

    /**
     * A shared instance.
     * @since 3.4.3
     */
    public static final NoKeyGenerator INSTANCE = new NoKeyGenerator();

    @Override
    public void processBefore(Executor executor, MappedStatement ms, Statement stmt, Object parameter) {
        // Do Nothing
    }

    @Override
    public void processAfter(Executor executor, MappedStatement ms, Statement stmt, Object parameter) {
        // Do Nothing
    }

}
```

## 6.总结

### 6.1 **KeyGenerator的创建过程**

> 参考 [MyBatis 初始化（三）之加载 Statement 配置](#http://svip.iocoder.cn/MyBatis/builder-package-3/)的[[2.2 parseStatementNode]](#2.2 parseStatementNode)的`<11>`、`<13>`段
>
> - 先解析` <selectKey />` 标签  ,`#processSelectKeyNodes`,
>
>   - 当你配置了时:
>
>     - ```java
>       // add 自定义的主键生成器，添加到配置中
>       configuration.addKeyGenerator(id, new SelectKeyGenerator(keyStatement, executeBefore));
>       ```
>
> - 获取Key
>
>   - ```java
>     KeyGenerator keyGenerator;
>             // 优先，从 configuration 中获得 KeyGenerator 对象。如果存在，意味着是 <selectKey /> 标签配置的
>             String keyStatementId = id + SelectKeyGenerator.SELECT_KEY_SUFFIX;
>             keyStatementId = builderAssistant.applyCurrentNamespace(keyStatementId, true);
>             if (configuration.hasKeyGenerator(keyStatementId)) {
>                 keyGenerator = configuration.getKeyGenerator(keyStatementId);
>             // 其次，根据标签属性的情况，判断是否使用对应的 Jdbc3KeyGenerator 或者 NoKeyGenerator 对象
>             } else {
>                 keyGenerator = context.getBooleanAttribute("useGeneratedKeys", // 优先，基于 useGeneratedKeys 属性判断
>                         configuration.isUseGeneratedKeys() && SqlCommandType.INSERT.equals(sqlCommandType)) // 其次，基于全局的 useGeneratedKeys 配置 + 是否为插入语句类型
>                         ? Jdbc3KeyGenerator.INSTANCE : NoKeyGenerator.INSTANCE;
>             }
>     // 最后 将keyGenerator保存至MappedStatement对象中
>     ```
>
>   可见：**优先使用 `<selectKey/>`对应的自定义`SelectKeyGenerator`，否则根据类型判断是 `Jdbc3KeyGenerator` 还是 `NoKeyGenerator`**。

### 6.2KeyGenerator的使用过程

> 参考：[SQL 执行（二）之 StatementHandler](#http://svip.iocoder.cn/MyBatis/executor-2/)
>
> - `processBefore`: [[4.1 BaseStatementHandler构造方法]]()
>
>   - ```java
>       // 如果 boundSql 非空，一般是写类操作，例如：insert、update、delete ，则先获得自增主键，然后再创建 BoundSql 对象
>             if (boundSql == null) { // issue #435, get the key before calculating the statement
>                 // 获得自增主键
>                 generateKeys(parameterObject); //  调用了keyGenerator.processBefore
>                 // 创建 BoundSql 对象
>                 boundSql = mappedStatement.getBoundSql(parameterObject);
>             }
>     ```
>
> - `processAfter`: `BaseStatementHandler`的子类 ，以`SimpleStatementHandler`为例：
>
>   - ![image-20220129205325608](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202201292053697.png)

## 666. *彩蛋

- 简单小文，嘿嘿。

  参考和推荐如下文章：

  - 祖大俊 [《Mybatis3.3.x技术内幕（十四）：Mybatis之KeyGenerator》](https://my.oschina.net/zudajun/blog/673612)
  - 祖大俊 [《Mybatis3.3.x技术内幕（十五）：Mybatis之foreach批量insert，返回主键id列表（修复Mybatis返回null的bug）》](https://my.oschina.net/zudajun/blog/674946) **强烈推荐**
    - 目前已经修复，参见 https://github.com/mybatis/mybatis-3/pull/324 。
  - 徐郡明 [《MyBatis 技术内幕》](https://item.jd.com/12125531.html) 的 [「3.4 KeyGenerator」](http://svip.iocoder.cn/MyBatis/executor-3/#) 小节