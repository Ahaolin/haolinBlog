# 精尽 MyBatis 源码分析 —— 类型模块

## 1. 概述

本文，我们来分享 MyBatis 的类型模块，对应 `type` 包。如下图所示：[![`type` 包](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202201101626800.png)](http://static.iocoder.cn/images/MyBatis/2020_01_25/01.png)`type` 包

在 [《精尽 MyBatis 源码解析 —— 项目结构一览》](http://svip.iocoder.cn/MyBatis/intro) 中，简单介绍了这个模块如下：

> ① MyBatis 为简化配置文件提供了**别名机制**，该机制是类型转换模块的主要功能之一。
>
> ② 类型转换模块的另一个功能是**实现 JDBC 类型与 Java 类型之间**的转换，该功能在为 SQL 语句绑定实参以及映射查询结果集时都会涉及：
>
> - 在为 SQL 语句绑定实参时，会将数据由 Java 类型转换成 JDBC 类型。
> - 而在映射结果集时，会将数据由 JDBC 类型转换成 Java 类型。

本文涉及的类如下图所示：[![类图](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202201101626814.png)](http://static.iocoder.cn/images/MyBatis/2020_01_25/02.png)类图

## 2. TypeHandler [(java <> jdbc) type] 

`org.apache.ibatis.type.TypeHandler` ，类型转换处理器。代码如下：

```java
// TypeHandler.java

public interface TypeHandler<T> {

    /**
     * 设置 PreparedStatement 的指定参数
     *
     * Java Type => JDBC Type
     *
     * @param ps PreparedStatement 对象
     * @param i 参数占位符的位置
     * @param parameter 参数
     * @param jdbcType JDBC 类型
     * @throws SQLException 当发生 SQL 异常时
     */
    void setParameter(PreparedStatement ps, int i, T parameter, JdbcType jdbcType) throws SQLException;

    /**
     * 获得 ResultSet 的指定字段的值
     *
     * JDBC Type => Java Type
     *
     * @param rs ResultSet 对象
     * @param columnName 字段名
     * @return 值
     * @throws SQLException 当发生 SQL 异常时
     */
    T getResult(ResultSet rs, String columnName) throws SQLException;

    /**
     * 获得 ResultSet 的指定字段的值
     *
     * JDBC Type => Java Type
     *
     * @param rs ResultSet 对象
     * @param columnIndex 字段位置
     * @return 值
     * @throws SQLException 当发生 SQL 异常时
     */
    T getResult(ResultSet rs, int columnIndex) throws SQLException;

    /**
     * 获得 CallableStatement 的指定字段的值
     *
     * JDBC Type => Java Type
     *
     * @param cs CallableStatement 对象，支持调用存储过程
     * @param columnIndex 字段位置
     * @return 值
     * @throws SQLException
     */
    T getResult(CallableStatement cs, int columnIndex) throws SQLException;

}
```

- 一共有两类方法，分别是：

  - `#setParameter(...)` 方法，是 `Java Type => JDBC Type` 的过程。
  - `#getResult(...)` 方法，是 `JDBC Type => Java Type` 的过程。

- 流程如下图：

  ![流程](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202201101626747.png)

  流程

  - 左边是 `#setParameter(...)` 方法，是 `Java Type => JDBC Type` 的过程，从上往下看。
  - 右边是 `#getResult(...)` 方法，是 `JDBC Type => Java Type` 的过程，从下往上看。

## 2.1 BaseTypeHandler

`org.apache.ibatis.type.BaseTypeHandler` ，实现 TypeHandler 接口，继承 TypeReference 抽象类，TypeHandler 基础抽象类。

- 关于 TypeReference ，我们在 [「3. TypeHandler」](http://svip.iocoder.cn/MyBatis/type-package/#) 中，详细解析。

### 2.1.1 setParameter

`#setParameter(PreparedStatement ps, int i, T parameter, JdbcType jdbcType)` 方法，代码如下：

```java
// BaseTypeHandler.java

@Override
public void setParameter(PreparedStatement ps, int i, T parameter, JdbcType jdbcType) throws SQLException {
    // <1> 参数为空时，设置为 null 类型
    if (parameter == null) {
        if (jdbcType == null) {
            throw new TypeException("JDBC requires that the JdbcType must be specified for all nullable parameters.");
        }
        try {
            ps.setNull(i, jdbcType.TYPE_CODE);
        } catch (SQLException e) {
            throw new TypeException("Error setting null for parameter #" + i + " with JdbcType " + jdbcType + " . " +
                    "Try setting a different JdbcType for this parameter or a different jdbcTypeForNull configuration property. " +
                    "Cause: " + e, e);
        }
    // 参数非空时，设置对应的参数
    } else {
        try {
            setNonNullParameter(ps, i, parameter, jdbcType);
        } catch (Exception e) {
            throw new TypeException("Error setting non null for parameter #" + i + " with JdbcType " + jdbcType + " . " +
                    "Try setting a different JdbcType for this parameter or a different configuration property. " +
                    "Cause: " + e, e);
        }
    }
}
```

- `<1>` 处，参数为空，设置为 `null` 类型。

- `<2>` 处，参数非空，调用 `#setNonNullParameter(PreparedStatement ps, int i, T parameter, JdbcType jdbcType)` **抽象**方法，设置对应的参数。代码如下：

  ```java
  // BaseTypeHandler.java
  
  public abstract void setNonNullParameter(PreparedStatement ps, int i, T parameter, JdbcType jdbcType) throws SQLException;
  ```

  - 该方法由子类实现。

- 当发生异常时，统一抛出 TypeException 异常。

### 2.1.2 getResult

`#getResult(...)` 方法，代码如下：

```java
// BaseTypeHandler.java

@Override
public T getResult(ResultSet rs, String columnName) throws SQLException {
    try {
        return getNullableResult(rs, columnName);
    } catch (Exception e) {
        throw new ResultMapException("Error attempting to get column '" + columnName + "' from result set.  Cause: " + e, e);
    }
}

@Override
public T getResult(ResultSet rs, int columnIndex) throws SQLException {
    try {
        return getNullableResult(rs, columnIndex);
    } catch (Exception e) {
        throw new ResultMapException("Error attempting to get column #" + columnIndex + " from result set.  Cause: " + e, e);
    }
}

@Override
public T getResult(CallableStatement cs, int columnIndex) throws SQLException {
    try {
        return getNullableResult(cs, columnIndex);
    } catch (Exception e) {
        throw new ResultMapException("Error attempting to get column #" + columnIndex + " from callable statement.  Cause: " + e, e);
    }
}
```

- 调用 `#getNullableResult(...)` **抽象**方法，获得指定结果的字段值。代码如下：

  ```java
  // BaseTypeHandler.java
  
  public abstract T getNullableResult(ResultSet rs, String columnName) throws SQLException;
  
  public abstract T getNullableResult(ResultSet rs, int columnIndex) throws SQLException;
  
  public abstract T getNullableResult(CallableStatement cs, int columnIndex) throws SQLException;
  ```

  - 该方法由子类实现。

- 当发生异常时，统一抛出 ResultMapException 异常。

## 2.2 子类

TypeHandler 有非常多的子类，**当然所有子类都是继承自 BaseTypeHandler 抽象类**。考虑到篇幅，我们就挑选几个来聊聊。

### 2.2.1 IntegerTypeHandler

`org.apache.ibatis.type.IntegerTypeHandler` ，继承 BaseTypeHandler 抽象类，Integer 类型的 TypeHandler 实现类。代码如下：

```java
// IntegerTypeHandler.java

public class IntegerTypeHandler extends BaseTypeHandler<Integer> {

    @Override
    public void setNonNullParameter(PreparedStatement ps, int i, Integer parameter, JdbcType jdbcType)
            throws SQLException {
        // 直接设置参数即可
        ps.setInt(i, parameter);
    }

    @Override
    public Integer getNullableResult(ResultSet rs, String columnName)
            throws SQLException {
        // 获得字段的值
        int result = rs.getInt(columnName);
        // 先通过 rs 判断是否空，如果是空，则返回 null ，否则返回 result
        return (result == 0 && rs.wasNull()) ? null : result;
    }

    @Override
    public Integer getNullableResult(ResultSet rs, int columnIndex)
            throws SQLException {
        // 获得字段的值
        int result = rs.getInt(columnIndex);
        // 先通过 rs 判断是否空，如果是空，则返回 null ，否则返回 result
        return (result == 0 && rs.wasNull()) ? null : result;
    }

    @Override
    public Integer getNullableResult(CallableStatement cs, int columnIndex)
            throws SQLException {
        // 获得字段的值
        int result = cs.getInt(columnIndex);
        // 先通过 cs 判断是否空，如果是空，则返回 null ，否则返回 result
        return (result == 0 && cs.wasNull()) ? null : result;
    }
}
```

- 比较简单，胖友瞅瞅。比较有意思的是 `ResultSet#wasNull()` 方法，它会判断**最后读取**的字段是否为空。

### 2.2.2 DateTypeHandler

`org.apache.ibatis.type.DateTypeHandler` ，继承 BaseTypeHandler 抽象类，Date 类型的 TypeHandler 实现类。代码如下：

```java
// DateTypeHandler.java

public class DateTypeHandler extends BaseTypeHandler<Date> {

    @Override
    public void setNonNullParameter(PreparedStatement ps, int i, Date parameter, JdbcType jdbcType)
            throws SQLException {
        // 将 Date 转换成 Timestamp 类型
        // 然后设置到 ps 中
        ps.setTimestamp(i, new Timestamp(parameter.getTime()));
    }

    @Override
    public Date getNullableResult(ResultSet rs, String columnName)
            throws SQLException {
        // 获得 Timestamp 的值
        Timestamp sqlTimestamp = rs.getTimestamp(columnName);
        // 将 Timestamp 转换成 Date 类型
        if (sqlTimestamp != null) {
            return new Date(sqlTimestamp.getTime());
        }
        return null;
    }

    @Override
    public Date getNullableResult(ResultSet rs, int columnIndex)
            throws SQLException {
        // 获得 Timestamp 的值
        Timestamp sqlTimestamp = rs.getTimestamp(columnIndex);
        // 将 Timestamp 转换成 Date 类型
        if (sqlTimestamp != null) {
            return new Date(sqlTimestamp.getTime());
        }
        return null;
    }

    @Override
    public Date getNullableResult(CallableStatement cs, int columnIndex)
            throws SQLException {
        // 获得 Timestamp 的值
        Timestamp sqlTimestamp = cs.getTimestamp(columnIndex);
        // 将 Timestamp 转换成 Date 类型
        if (sqlTimestamp != null) {
            return new Date(sqlTimestamp.getTime());
        }
        return null;
    }

}
```

- `java.util.Date` 和 `java.sql.Timestamp` 的互相转换。

### 2.2.3 DateOnlyTypeHandler

`org.apache.ibatis.type.DateOnlyTypeHandler` ，继承 BaseTypeHandler 抽象类，Date 类型的 TypeHandler 实现类。代码如下：

```java
// DateOnlyTypeHandler.java

public class DateOnlyTypeHandler extends BaseTypeHandler<Date> {

    @Override
    public void setNonNullParameter(PreparedStatement ps, int i, Date parameter, JdbcType jdbcType)
            throws SQLException {
        // 将 java Date 转换成 sql Date 类型
        ps.setDate(i, new java.sql.Date(parameter.getTime()));
    }

    @Override
    public Date getNullableResult(ResultSet rs, String columnName)
            throws SQLException {
        // 获得 sql Date 的值
        java.sql.Date sqlDate = rs.getDate(columnName);
        // 将 sql Date 转换成 java Date 类型
        if (sqlDate != null) {
            return new Date(sqlDate.getTime());
        }
        return null;
    }

    @Override
    public Date getNullableResult(ResultSet rs, int columnIndex)
            throws SQLException {
        // 获得 sql Date 的值
        java.sql.Date sqlDate = rs.getDate(columnIndex);
        // 将 sql Date 转换成 java Date 类型
        if (sqlDate != null) {
            return new Date(sqlDate.getTime());
        }
        return null;
    }

    @Override
    public Date getNullableResult(CallableStatement cs, int columnIndex)
            throws SQLException {
        // 获得 sql Date 的值
        java.sql.Date sqlDate = cs.getDate(columnIndex);
        // 将 sql Date 转换成 java Date 类型
        if (sqlDate != null) {
            return new Date(sqlDate.getTime());
        }
        return null;
    }

}
```

- `java.util.Date` 和 `java.sql.Date` 的互相转换。
- 数据库里的时间有多种类型，以 MySQL 举例子，有 `date`、`timestamp`、`datetime` 三种类型。

### 2.2.4 EnumTypeHandler

`org.apache.ibatis.type.EnumTypeHandler` ，继承 BaseTypeHandler 抽象类，Enum 类型的 TypeHandler 实现类。代码如下：

```java
// EnumTypeHandler.java

public class EnumTypeHandler<E extends Enum<E>> extends BaseTypeHandler<E> {

    /**
     * 枚举类
     */
    private final Class<E> type;

    public EnumTypeHandler(Class<E> type) {
        if (type == null) {
            throw new IllegalArgumentException("Type argument cannot be null");
        }
        this.type = type;
    }

    @Override
    public void setNonNullParameter(PreparedStatement ps, int i, E parameter, JdbcType jdbcType) throws SQLException {
        // 将 Enum 转换成 String 类型
        if (jdbcType == null) {
            ps.setString(i, parameter.name());
        } else {
            ps.setObject(i, parameter.name(), jdbcType.TYPE_CODE); // see r3589
        }
    }

    @Override
    public E getNullableResult(ResultSet rs, String columnName) throws SQLException {
        // 获得 String 的值
        String s = rs.getString(columnName);
        // 将 String 转换成 Enum 类型
        return s == null ? null : Enum.valueOf(type, s);
    }

    @Override
    public E getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
        // 获得 String 的值
        String s = rs.getString(columnIndex);
        // 将 String 转换成 Enum 类型
        return s == null ? null : Enum.valueOf(type, s);
    }

    @Override
    public E getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
        // 获得 String 的值
        String s = cs.getString(columnIndex);
        // 将 String 转换成 Enum 类型
        return s == null ? null : Enum.valueOf(type, s);
    }

}
```

- `java.lang.Enum` 和 `java.util.String` 的互相转换。
- 因为数据库不存在枚举类型，所以讲枚举类型持久化到数据库有两种方式，`Enum.name <=> String` 和 `Enum.ordinal <=> int` 。我们目前看到的 EnumTypeHandler 是前者，下面我们将看到的 EnumOrdinalTypeHandler 是后者。

### 2.2.5 EnumOrdinalTypeHandler

`org.apache.ibatis.type.EnumOrdinalTypeHandler` ，继承 BaseTypeHandler 抽象类，Enum 类型的 TypeHandler 实现类。代码如下：

```java
// EnumOrdinalTypeHandler.java

public class EnumOrdinalTypeHandler<E extends Enum<E>> extends BaseTypeHandler<E> {

    /**
     * 枚举类
     */
    private final Class<E> type;
    /**
     * {@link #type} 下所有的枚举
     *
     * @see Class#getEnumConstants()
     */
    private final E[] enums;

    public EnumOrdinalTypeHandler(Class<E> type) {
        if (type == null) {
            throw new IllegalArgumentException("Type argument cannot be null");
        }
        this.type = type;
        this.enums = type.getEnumConstants();
        if (this.enums == null) {
            throw new IllegalArgumentException(type.getSimpleName() + " does not represent an enum type.");
        }
    }

    @Override
    public void setNonNullParameter(PreparedStatement ps, int i, E parameter, JdbcType jdbcType) throws SQLException {
        // 将 Enum 转换成 int 类型
        ps.setInt(i, parameter.ordinal());
    }

    @Override
    public E getNullableResult(ResultSet rs, String columnName) throws SQLException {
        // 获得 int 的值
        int i = rs.getInt(columnName);
        // 将 int 转换成 Enum 类型
        if (i == 0 && rs.wasNull()) {
            return null;
        } else {
            try {
                return enums[i];
            } catch (Exception ex) {
                throw new IllegalArgumentException("Cannot convert " + i + " to " + type.getSimpleName() + " by ordinal value.", ex);
            }
        }
    }

    @Override
    public E getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
        // 获得 int 的值
        int i = rs.getInt(columnIndex);
        // 将 int 转换成 Enum 类型
        if (i == 0 && rs.wasNull()) {
            return null;
        } else {
            try {
                return enums[i];
            } catch (Exception ex) {
                throw new IllegalArgumentException("Cannot convert " + i + " to " + type.getSimpleName() + " by ordinal value.", ex);
            }
        }
    }

    @Override
    public E getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
        // 获得 int 的值
        int i = cs.getInt(columnIndex);
        // 将 int 转换成 Enum 类型
        if (i == 0 && cs.wasNull()) {
            return null;
        } else {
            try {
                return enums[i];
            } catch (Exception ex) {
                throw new IllegalArgumentException("Cannot convert " + i + " to " + type.getSimpleName() + " by ordinal value.", ex);
            }
        }
    }

}
```

- `java.lang.Enum` 和 `int` 的互相转换。

### 2.2.6 ObjectTypeHandler

`org.apache.ibatis.type.ObjectTypeHandler` ，继承 BaseTypeHandler 抽象类，Object 类型的 TypeHandler 实现类。代码如下：

```java
// ObjectTypeHandler.java

public class ObjectTypeHandler extends BaseTypeHandler<Object> {

    @Override
    public void setNonNullParameter(PreparedStatement ps, int i, Object parameter, JdbcType jdbcType)
            throws SQLException {
        ps.setObject(i, parameter);
    }

    @Override
    public Object getNullableResult(ResultSet rs, String columnName)
            throws SQLException {
        return rs.getObject(columnName);
    }

    @Override
    public Object getNullableResult(ResultSet rs, int columnIndex)
            throws SQLException {
        return rs.getObject(columnIndex);
    }

    @Override
    public Object getNullableResult(CallableStatement cs, int columnIndex)
            throws SQLException {
        return cs.getObject(columnIndex);
    }

}
```

### 2.2.7 UnknownTypeHandler

`org.apache.ibatis.type.UnknownTypeHandler` ，继承 BaseTypeHandler 抽象类，**未知的** TypeHandler 实现类。通过获取对应的 TypeHandler ，进行处理。代码如下：

```java
// UnknownTypeHandler.java

public class UnknownTypeHandler extends BaseTypeHandler<Object> {

    /**
     * ObjectTypeHandler 单例
     */
    private static final ObjectTypeHandler OBJECT_TYPE_HANDLER = new ObjectTypeHandler();

    /**
     * TypeHandler 注册表
     */
    private TypeHandlerRegistry typeHandlerRegistry;

    public UnknownTypeHandler(TypeHandlerRegistry typeHandlerRegistry) {
        this.typeHandlerRegistry = typeHandlerRegistry;
    }

    @Override
    public void setNonNullParameter(PreparedStatement ps, int i, Object parameter, JdbcType jdbcType)
            throws SQLException {
        // 获得参数对应的处理器
        TypeHandler handler = resolveTypeHandler(parameter, jdbcType); // <1>
        // 使用 handler 设置参数
        handler.setParameter(ps, i, parameter, jdbcType);
    }

    @Override
    public Object getNullableResult(ResultSet rs, String columnName)
            throws SQLException {
        // 获得参数对应的处理器
        TypeHandler<?> handler = resolveTypeHandler(rs, columnName); // <2>
        // 使用 handler 获得值
        return handler.getResult(rs, columnName);
    }

    @Override
    public Object getNullableResult(ResultSet rs, int columnIndex)
            throws SQLException {
        // 获得参数对应的处理器
        TypeHandler<?> handler = resolveTypeHandler(rs.getMetaData(), columnIndex); // <3>
        // 如果找不到对应的处理器，使用 OBJECT_TYPE_HANDLER
        if (handler == null || handler instanceof UnknownTypeHandler) {
            handler = OBJECT_TYPE_HANDLER;
        }
        // 使用 handler 获得值
        return handler.getResult(rs, columnIndex);
    }

    @Override
    public Object getNullableResult(CallableStatement cs, int columnIndex)
            throws SQLException {
        return cs.getObject(columnIndex);
    }

    private TypeHandler<? extends Object> resolveTypeHandler(Object parameter, JdbcType jdbcType) { // <1>
        TypeHandler<? extends Object> handler;
        // 参数为空，返回 OBJECT_TYPE_HANDLER
        if (parameter == null) {
            handler = OBJECT_TYPE_HANDLER;
        // 参数非空，使用参数类型获得对应的 TypeHandler
        } else {
            handler = typeHandlerRegistry.getTypeHandler(parameter.getClass(), jdbcType);
            // check if handler is null (issue #270)
            // 获取不到，则使用 OBJECT_TYPE_HANDLER
            if (handler == null || handler instanceof UnknownTypeHandler) {
                handler = OBJECT_TYPE_HANDLER;
            }
        }
        return handler;
    }

    private TypeHandler<?> resolveTypeHandler(ResultSet rs, String column) {
        try {
            // 获得 columnIndex
            Map<String, Integer> columnIndexLookup = new HashMap<>();
            ResultSetMetaData rsmd = rs.getMetaData(); // 通过 metaData
            int count = rsmd.getColumnCount();
            for (int i = 1; i <= count; i++) {
                String name = rsmd.getColumnName(i);
                columnIndexLookup.put(name, i);
            }
            Integer columnIndex = columnIndexLookup.get(column);
            TypeHandler<?> handler = null;
            // 首先，通过 columnIndex 获得 TypeHandler
            if (columnIndex != null) {
                handler = resolveTypeHandler(rsmd, columnIndex); // <3>
            }
            // 获得不到，使用 OBJECT_TYPE_HANDLER
            if (handler == null || handler instanceof UnknownTypeHandler) {
                handler = OBJECT_TYPE_HANDLER;
            }
            return handler;
        } catch (SQLException e) {
            throw new TypeException("Error determining JDBC type for column " + column + ".  Cause: " + e, e);
        }
    }

    private TypeHandler<?> resolveTypeHandler(ResultSetMetaData rsmd, Integer columnIndex) { // <3>
        TypeHandler<?> handler = null;
        // 获得 JDBC Type 类型
        JdbcType jdbcType = safeGetJdbcTypeForColumn(rsmd, columnIndex);
        // 获得 Java Type 类型
        Class<?> javaType = safeGetClassForColumn(rsmd, columnIndex);
        //获得对应的 TypeHandler 对象
        if (javaType != null && jdbcType != null) {
            handler = typeHandlerRegistry.getTypeHandler(javaType, jdbcType);
        } else if (javaType != null) {
            handler = typeHandlerRegistry.getTypeHandler(javaType);
        } else if (jdbcType != null) {
            handler = typeHandlerRegistry.getTypeHandler(jdbcType);
        }
        return handler;
    }

    private JdbcType safeGetJdbcTypeForColumn(ResultSetMetaData rsmd, Integer columnIndex) {
        try {
            // 从 ResultSetMetaData 中，获得字段类型
            // 获得 JDBC Type
            return JdbcType.forCode(rsmd.getColumnType(columnIndex));
        } catch (Exception e) {
            return null;
        }
    }

    private Class<?> safeGetClassForColumn(ResultSetMetaData rsmd, Integer columnIndex) {
        try {
            // 从 ResultSetMetaData 中，获得字段类型
            // 获得 Java Type
            return Resources.classForName(rsmd.getColumnClassName(columnIndex));
        } catch (Exception e) {
            return null;
        }
    }
    
}
```

- 代码比较简单，胖友自己瞅瞅。

## 3. TypeReference [解析类的泛型]

`org.apache.ibatis.type.TypeReference` ，引用泛型抽象类。目的很简单，就是解析类上定义的泛型。

`2.1 BaseTypeHandler`继承了此类，该类代码如下：

```java
// TypeReference.java

public abstract class TypeReference<T> {

    /**
     * 泛型
     */
    private final Type rawType;

    protected TypeReference() {
        rawType = getSuperclassTypeParameter(getClass());
    }

    Type getSuperclassTypeParameter(Class<?> clazz) {
        // 【1】从父类中获取 <T>
        Type genericSuperclass = clazz.getGenericSuperclass();
        if (genericSuperclass instanceof Class) {
            // 能满足这个条件的，例如 GenericTypeSupportedInHierarchiesTestCase.CustomStringTypeHandler 这个类
            // try to climb up the hierarchy until meet something useful
            if (TypeReference.class != genericSuperclass) { // 排除 TypeReference 类
                return getSuperclassTypeParameter(clazz.getSuperclass());
            }

            throw new TypeException("'" + getClass() + "' extends TypeReference but misses the type parameter. "
                    + "Remove the extension or add a type parameter to it.");
        }

        // 【2】获取 <T>
        Type rawType = ((ParameterizedType) genericSuperclass).getActualTypeArguments()[0];
        // TODO remove this when Reflector is fixed to return Types
        // 必须是泛型，才获取 <T>
        if (rawType instanceof ParameterizedType) {
            rawType = ((ParameterizedType) rawType).getRawType();
        }

        return rawType;
    }

    public final Type getRawType() {
        return rawType;
    }

    @Override
    public String toString() {
        return rawType.toString();
    }

}
```

- 举个例子，[「2.2.1 IntegerTypeHandler」](http://svip.iocoder.cn/MyBatis/type-package/#) 解析后的结果 `rawType` 为 Integer 。

- `【1】` 处，从父类中获取 `<T>` 。举个例子，代码如下：

  ```java
  // GenericTypeSupportedInHierarchiesTestCase.java 的内部静态类
  
  public static final class CustomStringTypeHandler extends StringTypeHandler {
  
      /**
       * Defined as reported in #581
       */
      @Override
      public void setNonNullParameter(PreparedStatement ps, int i, String parameter, JdbcType jdbcType) throws SQLException {
          // do something
          super.setNonNullParameter(ps, i, parameter, jdbcType);
      }
  
  }
  ```

  - 因为 CustomStringTypeHandler 自身是没有泛型的，需要从父类 StringTypeHandler 中获取。并且，获取的结果会是 `rawType` 为 String 。

- `【2】` 处，从当前类获取 `<T>` 。

## 4. 注解

`type` 包中，也定义了三个注解，我们逐个来看看。

### 4.1 @MappedTypes

`org.apache.ibatis.type.@MappedTypes` ，匹配的 Java Type 类型的注解。代码如下：

```java
// MappedTypes.java

@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE) // 注册到类
public @interface MappedTypes {

    /**
     * @return 匹配的 Java Type 类型的数组
     */
    Class<?>[] value();

}
```

### 4.2 @MappedJdbcTypes

`org.apache.ibatis.type.@MappedJdbcTypes` ，匹配的 JDBC Type 类型的注解。代码如下：

```java
// MappedJdbcTypes.java

@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE) // 注册到类
public @interface MappedJdbcTypes {

    /**
     * @return 匹配的 JDBC Type 类型的注解
     */
    JdbcType[] value();

    /**
     * @return 是否包含 {@link java.sql.JDBCType#NULL}
     */
    boolean includeNullJdbcType() default false;

}
```

### 4.3 @Alias

`org.apache.ibatis.type.@Alias` ，别名的注解。代码如下：

```java
// Alias.java

@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface Alias {

    /**
     * @return 别名
     */
    String value();

}
```

## 5. JdbcType

`org.apache.ibatis.type.JdbcType` ，Jdbc Type 枚举。代码如下：

```java
// JdbcType.java

public enum JdbcType {

    /*
     * This is added to enable basic support for the
     * ARRAY data type - but a custom type handler is still required
     */
    ARRAY(Types.ARRAY),
    BIT(Types.BIT),
    TINYINT(Types.TINYINT),
    SMALLINT(Types.SMALLINT),
    INTEGER(Types.INTEGER),
    BIGINT(Types.BIGINT),
    FLOAT(Types.FLOAT),
    REAL(Types.REAL),
    DOUBLE(Types.DOUBLE),
    NUMERIC(Types.NUMERIC),
    DECIMAL(Types.DECIMAL),
    CHAR(Types.CHAR),
    VARCHAR(Types.VARCHAR),
    LONGVARCHAR(Types.LONGVARCHAR),
    DATE(Types.DATE),
    TIME(Types.TIME),
    TIMESTAMP(Types.TIMESTAMP),
    BINARY(Types.BINARY),
    VARBINARY(Types.VARBINARY),
    LONGVARBINARY(Types.LONGVARBINARY),
    NULL(Types.NULL),
    OTHER(Types.OTHER),
    BLOB(Types.BLOB),
    CLOB(Types.CLOB),
    BOOLEAN(Types.BOOLEAN),
    CURSOR(-10), // Oracle
    UNDEFINED(Integer.MIN_VALUE + 1000),
    NVARCHAR(Types.NVARCHAR), // JDK6
    NCHAR(Types.NCHAR), // JDK6
    NCLOB(Types.NCLOB), // JDK6
    STRUCT(Types.STRUCT),
    JAVA_OBJECT(Types.JAVA_OBJECT),
    DISTINCT(Types.DISTINCT),
    REF(Types.REF),
    DATALINK(Types.DATALINK),
    ROWID(Types.ROWID), // JDK6
    LONGNVARCHAR(Types.LONGNVARCHAR), // JDK6
    SQLXML(Types.SQLXML), // JDK6
    DATETIMEOFFSET(-155); // SQL Server 2008

    /**
     * 类型编号。嘿嘿，此处代码不规范
     */
    public final int TYPE_CODE;

    /**
     * 代码编号和 {@link JdbcType} 的映射
     */
    private static Map<Integer, JdbcType> codeLookup = new HashMap<>();

    static {
        // 初始化 codeLookup
        for (JdbcType type : JdbcType.values()) {
            codeLookup.put(type.TYPE_CODE, type);
        }
    }

    JdbcType(int code) {
        this.TYPE_CODE = code;
    }

    public static JdbcType forCode(int code) {
        return codeLookup.get(code);
    }

}
```

## 6. TypeHandlerRegistry【TypeHandler 注册表】

`org.apache.ibatis.type.TypeHandlerRegistry` ，TypeHandler 注册表，相当于管理 TypeHandler 的容器，从其中能获取到对应的 TypeHandler 。

### 6.1 构造方法

```java
// TypeHandlerRegistry.java

/**
 * 空 TypeHandler 集合的标识，即使 {@link #TYPE_HANDLER_MAP} 中，某个 KEY1 对应的 Map<JdbcType, TypeHandler<?>> 为空。
 *
 * @see #getJdbcHandlerMap(Type)
 */
private static final Map<JdbcType, TypeHandler<?>> NULL_TYPE_HANDLER_MAP = Collections.emptyMap();

/**
 * JDBC Type 和 {@link TypeHandler} 的映射
 *
 * {@link #register(JdbcType, TypeHandler)}
 */
private final Map<JdbcType, TypeHandler<?>> JDBC_TYPE_HANDLER_MAP = new EnumMap<>(JdbcType.class);
/**
 * {@link TypeHandler} 的映射
 *
 * KEY1：JDBC Type
 * KEY2：Java Type
 * VALUE：{@link TypeHandler} 对象
 */
private final Map<Type, Map<JdbcType, TypeHandler<?>>> TYPE_HANDLER_MAP = new ConcurrentHashMap<>();
/**
 * 所有 TypeHandler 的“集合”
 *
 * KEY：{@link TypeHandler#getClass()}
 * VALUE：{@link TypeHandler} 对象
 */
private final Map<Class<?>, TypeHandler<?>> ALL_TYPE_HANDLERS_MAP = new HashMap<>();

/**
 * {@link UnknownTypeHandler} 对象
 */
private final TypeHandler<Object> UNKNOWN_TYPE_HANDLER = new UnknownTypeHandler(this);
/**
 * 默认的枚举类型的 TypeHandler 对象
 */
private Class<? extends TypeHandler> defaultEnumTypeHandler = EnumTypeHandler.class;

public TypeHandlerRegistry() {
    // ... 省略其它类型的注册

    // <1> add  TYPE_HANDLER_MAP、ALL_TYPE_HANDLERS_MAP
    register(Date.class, new DateTypeHandler());
    register(Date.class, JdbcType.DATE, new DateOnlyTypeHandler());
    register(Date.class, JdbcType.TIME, new TimeOnlyTypeHandler());
    // <2> add JDBC_TYPE_HANDLER_MAP
    register(JdbcType.TIMESTAMP, new DateTypeHandler());
    register(JdbcType.DATE, new DateOnlyTypeHandler());
    register(JdbcType.TIME, new TimeOnlyTypeHandler());

    // ... 省略其它类型的注册
}
```

- `TYPE_HANDLER_MAP`属性，TypeHandler 的映射。
  - 一个 Java Type 可以对应多个 JDBC Type ，也就是多个 TypeHandler ，所以 Map 的第一层的值是 `Map<JdbcType, TypeHandler<?>` 。在 `<1>` 处，我们可以看到，Date 对应了多个 JDBC 的 TypeHandler 的注册。
  - 当一个 Java Type 不存在对应的 JDBC Type 时，就使用 `NULL_TYPE_HANDLER_MAP` **静态**属性，添加到 `TYPE_HANDLER_MAP` 中进行占位。

- `JDBC_TYPE_HANDLER_MAP`属性，JDBC Type 和 TypeHandler 的映射。
  - 一个 JDBC Type 只对应一个 Java Type ，也就是一个 TypeHandler ，不同于 `TYPE_HANDLER_MAP` 属性。在 `<2>` 处，我们可以看到，我们可以看到，三个时间类型的 JdbcType 注册到 `JDBC_TYPE_HANDLER_MAP` 中。
  - 那么可能会有胖友问，`JDBC_TYPE_HANDLER_MAP` 是**一一**映射，简单就可以获得 JDBC Type 对应的 TypeHandler ，而 `TYPE_HANDLER_MAP` 是**一对多**映射，一个 JavaType 怎么获取到对应的 TypeHandler 呢？继续往下看，答案在 `#getTypeHandler(Type type, JdbcType jdbcType)` 方法。

- `ALL_TYPE_HANDLERS_MAP` 属性，所有 TypeHandler 的“集合” 。

- `UNKNOWN_TYPE_HANDLER` 属性，UnknownTypeHandler 对象，用于 Object 类型的注册。

- `defaultEnumTypeHandler` 属性，默认的枚举类型的 TypeHandler 对象。

- 在构造方法中，有默认的 Java Type 和 JDBC Type 对 TypeHandler 的注册，因为有丢丢多，所以被艿艿省略了。

- 另外，这里的变量命名是不符合 Java 命名规范，不要学习。

### 6.2 getInstance

`#getInstance(Class<?> javaTypeClass, Class<?> typeHandlerClass)` 方法，创建 TypeHandler 对象。代码如下：

```java
// TypeHandlerRegistry.java

public <T> TypeHandler<T> getInstance(Class<?> javaTypeClass, Class<?> typeHandlerClass) {
    // 获得 Class 类型的构造方法
    if (javaTypeClass != null) {
        try {
            Constructor<?> c = typeHandlerClass.getConstructor(Class.class);
            return (TypeHandler<T>) c.newInstance(javaTypeClass); // 符合这个条件的，例如 EnumTypeHandler
        } catch (NoSuchMethodException ignored) {
            // ignored 忽略该异常，继续向下
        } catch (Exception e) {
            throw new TypeException("Failed invoking constructor for handler " + typeHandlerClass, e);
        }
    }
    // <2> 获得空参的构造方法
    try {
        Constructor<?> c = typeHandlerClass.getConstructor();
        return (TypeHandler<T>) c.newInstance(); // 符合这个条件的，例如 IntegerTypeHandler
    } catch (Exception e) {
        throw new TypeException("Unable to find a usable constructor for " + typeHandlerClass, e);
    }
}
```

- `<1>` 处，获得 Class 类型的构造方法，适合 [「2.2.4 EnumTypeHandler」](http://svip.iocoder.cn/MyBatis/type-package/#) 的情况。
- `<2>` 处，获得空参的构造方法，适合 [「2.2.1 IntegerTypeHandler」](http://svip.iocoder.cn/MyBatis/type-package/#) 的情况。

### 6.3 register

`#register(...)` 方法，注册 TypeHandler 。TypeHandlerRegistry 中有大量该方法的重载实现，大体整理如下：

> FROM 徐郡明 [《MyBatis 技术内幕》](https://item.jd.com/12125531.html)
>
> [<img src="http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202201101626809.png" alt="register 方法" style="zoom:150%;" />](http://static.iocoder.cn/images/MyBatis/2020_01_25/04.png)register 方法

除了 ⑤ 以外，所有方法最终都会调用 ④ ，即 `#register(Type javaType, JdbcType jdbcType, TypeHandler<?> handler)` 方法，代码如下：

```java
// TypeHandlerRegistry.java

private void register(Type javaType, JdbcType jdbcType, TypeHandler<?> handler) {
    // <1> 添加 handler 到 TYPE_HANDLER_MAP 中
    if (javaType != null) {
        // 获得 Java Type 对应的 map
        Map<JdbcType, TypeHandler<?>> map = TYPE_HANDLER_MAP.get(javaType);
        if (map == null || map == NULL_TYPE_HANDLER_MAP) { // 如果不存在，则进行创建
            map = new HashMap<>();
            TYPE_HANDLER_MAP.put(javaType, map);
        }
        // 添加到 handler 中 map 中
        map.put(jdbcType, handler);
    }
    // <2> 添加 handler 到 ALL_TYPE_HANDLERS_MAP 中
    ALL_TYPE_HANDLERS_MAP.put(handler.getClass(), handler);
}
```

- `<1>` 处，添加 `handler` 到 `TYPE_HANDLER_MAP` 中。
- `<2>` 处，添加 `handler` 到 `ALL_TYPE_HANDLERS_MAP` 中。
- 这个方法还是比较简单的，我们看看其他调用的方法。

① `#register(String packageName)` 方法，扫描指定包下的所有 TypeHandler 类，并发起注册。代码如下：

```java
// TypeHandlerRegistry.java
    
public void register(String packageName) {
    // 扫描指定包下的所有 TypeHandler 类
    ResolverUtil<Class<?>> resolverUtil = new ResolverUtil<>();
    resolverUtil.find(new ResolverUtil.IsA(TypeHandler.class), packageName);
    Set<Class<? extends Class<?>>> handlerSet = resolverUtil.getClasses();
    // 遍历 TypeHandler 数组，发起注册
    for (Class<?> type : handlerSet) {
        //Ignore inner classes and interfaces (including package-info.java) and abstract classes
        // 排除匿名类、接口、抽象类
        if (!type.isAnonymousClass() && !type.isInterface() && !Modifier.isAbstract(type.getModifiers())) {
            register(type);
        }
    }
}
```

- 在方法，会调用 ⑥ `#register(Class<?> typeHandlerClass)` 方法，注册指定 TypeHandler **类**。代码如下：

  ```java
  // TypeHandlerRegistry.java
  
  public void register(Class<?> typeHandlerClass) {
      boolean mappedTypeFound = false;
      // <3> 获得 @MappedTypes 注解
      MappedTypes mappedTypes = typeHandlerClass.getAnnotation(MappedTypes.class);
      if (mappedTypes != null) {
          // 遍历注解的 Java Type 数组，逐个进行注册
          for (Class<?> javaTypeClass : mappedTypes.value()) {
              register(javaTypeClass, typeHandlerClass);
              mappedTypeFound = true;
          }
      }
      // <4> 未使用 @MappedTypes 注解，则直接注册
      if (!mappedTypeFound) {
          register(getInstance(null, typeHandlerClass)); // 创建 TypeHandler 对象
      }
  }
  ```

  - 分成 `<3>` `<4>` 两种情况。

  - `<3>` 处，**基于** `@MappedTypes` 注解，调用 `#register(Class<?> javaTypeClass, Class<?> typeHandlerClass)` 方法，注册指定 Java Type 的指定 TypeHandler **类**。代码如下：

    ```java
    // TypeHandlerRegistry.java
    
    public void register(Class<?> javaTypeClass, Class<?> typeHandlerClass) {
        register(javaTypeClass, getInstance(javaTypeClass, typeHandlerClass)); // 创建 TypeHandler 对象
    }
    ```

    - 调用 ③ `#register(Class<T> javaType, TypeHandler<? extends T> typeHandler)` 方法，注册指定 Java Type 的指定 TypeHandler **对象**。代码如下：

      ```java
      // TypeHandlerRegistry.java
      
      private <T> void register(Type javaType, TypeHandler<? extends T> typeHandler) {
          // 获得 MappedJdbcTypes 注解
          MappedJdbcTypes mappedJdbcTypes = typeHandler.getClass().getAnnotation(MappedJdbcTypes.class);
          if (mappedJdbcTypes != null) {
              // 遍历 MappedJdbcTypes 注册的 JDBC Type 进行注册
              for (JdbcType handledJdbcType : mappedJdbcTypes.value()) {
                  register(javaType, handledJdbcType, typeHandler);
              }
              if (mappedJdbcTypes.includeNullJdbcType()) {
                  // <5>
                  register(javaType, null, typeHandler); // jdbcType = null
              }
          } else {
              // <5>
              register(javaType, null, typeHandler); // jdbcType = null
          }
      }
      ```

      - **有** `@MappedJdbcTypes` 注解的 ④ `#register(Type javaType, JdbcType jdbcType, TypeHandler<?> handler)` 方法，发起**最终**注册。
      - 对于 `<5>` 处，发起注册时，`jdbcType` 参数为 `null` ，这是为啥？😈 下文详细解析。

  - `<4>` 处，调用 ② `#register(TypeHandler<T> typeHandler)` 方法，**未使用** `@MappedTypes` 注解，调用 `#register(TypeHandler<T> typeHandler)` 方法，注册 TypeHandler 对象。代码如下：

    ```java
    // TypeHandlerRegistry.java
    
    public <T> void register(TypeHandler<T> typeHandler) {
        boolean mappedTypeFound = false;
        // <5> 获得 @MappedTypes 注解
        MappedTypes mappedTypes = typeHandler.getClass().getAnnotation(MappedTypes.class);
        // 优先，使用 @MappedTypes 注解的 Java Type 进行注册
        if (mappedTypes != null) {
            for (Class<?> handledType : mappedTypes.value()) {
                register(handledType, typeHandler);
                mappedTypeFound = true;
            }
        }
        // @since 3.1.0 - try to auto-discover the mapped type
        // <6> 其次，当 typeHandler 为 TypeReference 子类时，进行注册
        if (!mappedTypeFound && typeHandler instanceof TypeReference) {
            try {
                TypeReference<T> typeReference = (TypeReference<T>) typeHandler;
                register(typeReference.getRawType(), typeHandler); // Java Type 为 <T> 泛型
                mappedTypeFound = true;
            } catch (Throwable t) {
                // maybe users define the TypeReference with a different type and are not assignable, so just ignore it
            }
        }
        // <7> 最差，使用 Java Type 为 null 进行注册
        if (!mappedTypeFound) {
            register((Class<T>) null, typeHandler);
        }
    }
    ```

    - 分成三种情况，最终都是调用 `#register(Type javaType, TypeHandler<? extends T> typeHandler)` 方法，进行注册，也就是跳到 ③ 。
    - `<5>` 处，优先，有符合的 `@MappedTypes` 注解时，使用 `@MappedTypes` 注解的 Java Type 进行注册。
    - `<6>` 处，其次，当 `typeHandler` 为 TypeReference 子类时，使用 `<T>` 作为 Java Type 进行注册。
    - `<7>` 处，最差，使用 `null` 作为 Java Type 进行注册。但是，这种情况下，只会将 `typeHandler` 添加到 `ALL_TYPE_HANDLERS_MAP` 中。因为，实际上没有 Java Type 。

因为重载的方法有点多，理解起来可能比较绕，胖友可能会比较闷逼。哈哈哈，实际在自己写的过程中，也有点懵逼。那怎么办？大体理解就好，另外 `@MappedTypes` 和 `@MappedJdbcTypes` 这两个注解基本不会用到，所以也可以先“忽略”。

另外，`#register(...)` 方法，还有其它重载方法，胖友可以自己翻下。

------

⑤ `#register(JdbcType jdbcType, TypeHandler<?> handler)` 方法，注册 `handler` 到 `JDBC_TYPE_HANDLER_MAP` 中。代码如下：

```java
// TypeHandlerRegistry.java

public void register(JdbcType jdbcType, TypeHandler<?> handler) {
    JDBC_TYPE_HANDLER_MAP.put(jdbcType, handler);
}
```

- 和上述的 `#register(...)` 方法是不同的。

### 6.4 getTypeHandler

`#getTypeHandler(...)` 方法，获得 TypeHandler 。TypeHandlerRegistry 有大量该方法的重载实现，大体整体如下：

> FROM 徐郡明 [《MyBatis 技术内幕》](https://item.jd.com/12125531.html)
>
> [![getTypeHandler 方法](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202201101626813.png)](http://static.iocoder.cn/images/MyBatis/2020_01_25/05.png)getTypeHandler 方法

从图中，我们可以看到，最终会调用 ① 处的 `#getTypeHandler(Type type, JdbcType jdbcType)` 方法。当然，我们先来看看三种调用的情况：

- 调用情况一：`#getTypeHandler(Class<T> type)` 方法，代码如下：

  ```java
  // TypeHandlerRegistry.java
  
  public <T> TypeHandler<T> getTypeHandler(Class<T> type) {
      return getTypeHandler((Type) type, null);
  }
  ```

  - `jdbcType` 为 `null` 。

- 调用情况二：`#getTypeHandler(Class<T> type, JdbcType jdbcType)` 方法，代码如下：

  ```java
  // TypeHandlerRegistry.java
  
  public <T> TypeHandler<T> getTypeHandler(Class<T> type, JdbcType jdbcType) {
      return getTypeHandler((Type) type, jdbcType);
  }
  ```

  - 将 `type` 转换成 Type 类型。

- 调用情况三：`#getTypeHandler(TypeReference<T> javaTypeReference, ...)` 方法，代码如下：

  ```java
  // TypeHandlerRegistry.java
  
  public <T> TypeHandler<T> getTypeHandler(TypeReference<T> javaTypeReference) {
      return getTypeHandler(javaTypeReference, null);
  }
  
  public <T> TypeHandler<T> getTypeHandler(TypeReference<T> javaTypeReference, JdbcType jdbcType) {
      return getTypeHandler(javaTypeReference.getRawType(), jdbcType);
  }
  ```

  - 使用 `<T>` 泛型作为 `type` 。

下面，正式来看看 ① 处的 `#getTypeHandler(Type type, JdbcType jdbcType)` 方法。代码如下：

```java
// TypeHandlerRegistry.java

private <T> TypeHandler<T> getTypeHandler(Type type, JdbcType jdbcType) {
    // 忽略 ParamMap 的情况
    if (ParamMap.class.equals(type)) {
        return null;
    }
    // <1> 获得 Java Type 对应的 TypeHandler 集合
    Map<JdbcType, TypeHandler<?>> jdbcHandlerMap = getJdbcHandlerMap(type);
    TypeHandler<?> handler = null;
    if (jdbcHandlerMap != null) {
        // <2.1> 优先，使用 jdbcType 获取对应的 TypeHandler
        handler = jdbcHandlerMap.get(jdbcType);
        // <2.2> 其次，使用 null 获取对应的 TypeHandler ，可以认为是默认的 TypeHandler
        if (handler == null) {
            handler = jdbcHandlerMap.get(null);
        }
        // <2.3> 最差，从 TypeHandler 集合中选择一个唯一的 TypeHandler
        if (handler == null) {
            // #591
            handler = pickSoleHandler(jdbcHandlerMap);
        }
    }
    // type drives generics here
    return (TypeHandler<T>) handler;
}
```

- `<1>` 处，调用 `#getJdbcHandlerMap(Type type)` 方法，获得 Java Type 对应的 TypeHandler 集合。代码如下：

  ```java
  // TypeHandlerRegistry.java
  
  private Map<JdbcType, TypeHandler<?>> getJdbcHandlerMap(Type type) {
      // <1.1> 获得 Java Type 对应的 TypeHandler 集合
      Map<JdbcType, TypeHandler<?>> jdbcHandlerMap = TYPE_HANDLER_MAP.get(type);
      // <1.2> 如果为 NULL_TYPE_HANDLER_MAP ，意味着为空，直接返回
      if (NULL_TYPE_HANDLER_MAP.equals(jdbcHandlerMap)) {
          return null;
      }
      // <1.3> 如果找不到
      if (jdbcHandlerMap == null && type instanceof Class) {
          Class<?> clazz = (Class<?>) type;
          // 枚举类型
          if (clazz.isEnum()) {
              // 获得父类对应的 TypeHandler 集合
              jdbcHandlerMap = getJdbcHandlerMapForEnumInterfaces(clazz, clazz);
              // 如果找不到
              if (jdbcHandlerMap == null) {
                  // 注册 defaultEnumTypeHandler ，并使用它
                  register(clazz, getInstance(clazz, defaultEnumTypeHandler));
                  // 返回结果
                  return TYPE_HANDLER_MAP.get(clazz);
              }
          // 非枚举类型
          } else {
              // 获得父类对应的 TypeHandler 集合
              jdbcHandlerMap = getJdbcHandlerMapForSuperclass(clazz);
          }
      }
      // <1.4> 如果结果为空，设置为 NULL_TYPE_HANDLER_MAP ，提升查找速度，避免二次查找
      TYPE_HANDLER_MAP.put(type, jdbcHandlerMap == null ? NULL_TYPE_HANDLER_MAP : jdbcHandlerMap);
      // 返回结果
      return jdbcHandlerMap;
  }
  ```

  - `<1.1>` 处，获得 Java Type 对应的 TypeHandler 集合。

  - `<1.2>` 处，如果为 `NULL_TYPE_HANDLER_MAP` ，意味着为空，直接返回。原因可见 `<1.4>` 处。

  - `<1.3>` 处，找不到，则根据 `type` 是否为枚举类型，进行不同处理。

    - 【枚举】

    - 先调用 `#getJdbcHandlerMapForEnumInterfaces(Class<?> clazz, Class<?> enumClazz)` 方法， 获得父类对应的 TypeHandler 集合。代码如下：

      ```java
      // TypeHandlerRegistry.java
      
      private Map<JdbcType, TypeHandler<?>> getJdbcHandlerMapForEnumInterfaces(Class<?> clazz, Class<?> enumClazz) {
          // 遍历枚举类的所有接口
          for (Class<?> iface : clazz.getInterfaces()) {
              // 获得该接口对应的 jdbcHandlerMap 集合
              Map<JdbcType, TypeHandler<?>> jdbcHandlerMap = TYPE_HANDLER_MAP.get(iface);
              // 为空，递归 getJdbcHandlerMapForEnumInterfaces 方法，继续从父类对应的 TypeHandler 集合
              if (jdbcHandlerMap == null) {
                  jdbcHandlerMap = getJdbcHandlerMapForEnumInterfaces(iface, enumClazz);
              }
              // 如果找到，则从 jdbcHandlerMap 初始化中 newMap 中，并进行返回
              if (jdbcHandlerMap != null) {
                  // Found a type handler regsiterd to a super interface
                  HashMap<JdbcType, TypeHandler<?>> newMap = new HashMap<>();
                  for (Entry<JdbcType, TypeHandler<?>> entry : jdbcHandlerMap.entrySet()) {
                      // Create a type handler instance with enum type as a constructor arg
                      newMap.put(entry.getKey(), getInstance(enumClazz, entry.getValue().getClass()));
                  }
                  return newMap;
              }
          }
          // 找不到，则返回 null
          return null;
      }
      ```

      - 代码比较简单，看下代码注释。

    - 找不到，则注册 `defaultEnumTypeHandler` ，并使用它。

    - 【非枚举】

    - 调用 `#getJdbcHandlerMapForSuperclass(Class<?> clazz)` 方法，获得父类对应的 TypeHandler 集合。代码如下：

      ```java
      // TypeHandlerRegistry.java
      
      private Map<JdbcType, TypeHandler<?>> getJdbcHandlerMapForSuperclass(Class<?> clazz) {
          // 获得父类
          Class<?> superclass = clazz.getSuperclass();
          // 不存在非 Object 的父类，返回 null
          if (superclass == null || Object.class.equals(superclass)) {
              return null;
          }
          // 获得父类对应的 TypeHandler 集合
          Map<JdbcType, TypeHandler<?>> jdbcHandlerMap = TYPE_HANDLER_MAP.get(superclass);
          // 找到，则直接返回
          if (jdbcHandlerMap != null) {
              return jdbcHandlerMap;
          // 找不到，则递归 getJdbcHandlerMapForSuperclass 方法，继续获得父类对应的 TypeHandler 集合
          } else {
              return getJdbcHandlerMapForSuperclass(superclass);
          }
      }
      ```

      - 代码比较简单，看下代码注释。

  - `<1.4>` 处，如果结果为空，设置为 `NULL_TYPE_HANDLER_MAP` ，提升查找速度，避免二次查找。

- `<2.1>` 处，优先，使用 `jdbcType` 获取对应的 TypeHandler 。

- `<2.2>` 处，其次，使用 `null` 获取对应的 TypeHandler ，可以认为是**默认**的 TypeHandler 。这里是解决一个 Java Type 可能对应多个 TypeHandler 的**方式之一**。

- `<2.3>` 处，最差，调用 `#pickSoleHandler(Map<JdbcType, TypeHandler<?>> jdbcHandlerMap)` 方法，从 TypeHandler 集合中选择一个**唯一**的 TypeHandler 。代码如下：

  ```java
  // TypeHandlerRegistry.java
  
  private TypeHandler<?> pickSoleHandler(Map<JdbcType, TypeHandler<?>> jdbcHandlerMap) {
      TypeHandler<?> soleHandler = null;
      for (TypeHandler<?> handler : jdbcHandlerMap.values()) {
          // 选择一个
          if (soleHandler == null) {
              soleHandler = handler;
          // 如果还有，并且不同类，那么不好选择，所以返回 null
          } else if (!handler.getClass().equals(soleHandler.getClass())) {
              // More than one type handlers registered.
              return null;
          }
      }
      return soleHandler;
  }
  ```

  - 这段代码看起来比较绕，其实目的很清晰，就是选择**第一个**，并且不能有其它的**不同类**的处理器。
  - 这里是解决一个 Java Type 可能对应多个 TypeHandler 的**方式之一**。

- 通过 `<2.1>` + `<2.2>` + `<2.3>` 三处，解决 Java Type 对应的 TypeHandler 集合。

------

`#getTypeHandler(JdbcType jdbcType)` 方法，获得 `jdbcType` 对应的 TypeHandler 。代码如下：

```java
// TypeHandlerRegistry.java

public TypeHandler<?> getTypeHandler(JdbcType jdbcType) {
    return JDBC_TYPE_HANDLER_MAP.get(jdbcType);
}
```

## 7. TypeAliasRegistry [类名与别名映射]

`org.apache.ibatis.type.TypeAliasRegistry` ，类型与别名的注册表。通过别名，我们在 Mapper XML 中的 `resultType` 和 `parameterType` 属性，直接使用，而不用写全类名。

### 7.1 构造方法

```java
// TypeAliasRegistry.java

/**
 * 类型与别名的映射。
 */
private final Map<String, Class<?>> TYPE_ALIASES = new HashMap<>();

/**
 * 初始化默认的类型与别名
 *
 * 另外，在 {@link org.apache.ibatis.session.Configuration} 构造方法中，也有默认的注册
 */
public TypeAliasRegistry() {
    registerAlias("string", String.class);

    registerAlias("byte", Byte.class);
    registerAlias("long", Long.class);
    registerAlias("short", Short.class);
    registerAlias("int", Integer.class);
    registerAlias("integer", Integer.class);
    registerAlias("double", Double.class);
    registerAlias("float", Float.class);
    registerAlias("boolean", Boolean.class);

    // ... 省略其他注册调用
}
```

- `TYPE_ALIASES` 属性，类型与别名的映射。
- 构造方法，初始化默认的类型与别名。
- 另外，在 `org.apache.ibatis.session.Configuration` 构造方法中，也有默认的注册类型与别名。

### 7.2 registerAlias

`#registerAlias(Class<?> type)` 方法，注册指定类。代码如下：

```java
// TypeAliasRegistry.java

public void registerAlias(Class<?> type) {
    // <1> 默认为，简单类名
    String alias = type.getSimpleName();
    // <2> 如果有注解，使用注册上的名字
    Alias aliasAnnotation = type.getAnnotation(Alias.class);
    if (aliasAnnotation != null) {
        alias = aliasAnnotation.value();
    }
    // <3> 注册类型与别名的注册表
    registerAlias(alias, type);
}
```

- 别名的规则

  - `<1>` ，默认为，简单类名。
  - `<2>` ，可通过 `@Alias` 注解的别名。

- `<3>` ，调用 `#registerAlias(String alias, Class<?> value)` 方法，注册类型与别名的注册表。代码如下：

  ```java
  // TypeAliasRegistry.java
  
  public void registerAlias(String alias, Class<?> value) {
      if (alias == null) {
          throw new TypeException("The parameter alias cannot be null");
      }
      // issue #748
      // <1> 转换成小写
      String key = alias.toLowerCase(Locale.ENGLISH);
      if (TYPE_ALIASES.containsKey(key) && TYPE_ALIASES.get(key) != null && !TYPE_ALIASES.get(key).equals(value)) { // <2> 冲突，抛出 TypeException 异常
          throw new TypeException("The alias '" + alias + "' is already mapped to the value '" + TYPE_ALIASES.get(key).getName() + "'.");
      }
      // <3>
      TYPE_ALIASES.put(key, value);
  }
  ```

- `<1>` 处，将别名转换成**小写**。这样的话，无论我们在 Mapper XML 中，写 `String` 还是 `string` 甚至是 `STRING` ，都是对应的 String 类型。 

-  `<2>` 处，如果已经注册，并且类型不一致，说明有冲突，抛出 TypeException 异常。 

-  `<3>` 处，添加到 `TYPE_ALIASES` 中。 * 

- 另外，`#registerAlias(String alias, String value)` 方法，也会调用该方法。代码如下：

  ```java
  // TypeAliasRegistry.java
  
  public void registerAlias(String alias, String value) {
      try {
          registerAlias(alias,
                  Resources.classForName(value) // 通过类名的字符串，获得对应的类。
          );
      } catch (ClassNotFoundException e) {
          throw new TypeException("Error registering type alias " + alias + " for " + value + ". Cause: " + e, e);
      }
  }
  ```

  

### 7.3 registerAliases

`#registerAliases(String packageName, ...)` 方法，扫描指定包下的所有类，并进行注册。代码如下：

```java
// TypeAliasRegistry.java

/**
 * 注册指定包下的别名与类的映射
 *
 * @param packageName 指定包
 */
public void registerAliases(String packageName) {
    registerAliases(packageName, Object.class);
}

/**
 * 注册指定包下的别名与类的映射。另外，要求类必须是 {@param superType} 类型（包括子类）。
 *
 * @param packageName 指定包
 * @param superType 指定父类
 */
public void registerAliases(String packageName, Class<?> superType) {
    // 获得指定包下的类门
    ResolverUtil<Class<?>> resolverUtil = new ResolverUtil<>();
    resolverUtil.find(new ResolverUtil.IsA(superType), packageName);
    Set<Class<? extends Class<?>>> typeSet = resolverUtil.getClasses();
    // 遍历，逐个注册类型与别名的注册表
    for (Class<?> type : typeSet) {
        // Ignore inner classes and interfaces (including package-info.java)
        // Skip also inner classes. See issue #6
        if (!type.isAnonymousClass() // 排除匿名类
                && !type.isInterface()  // 排除接口
                && !type.isMemberClass()) { // 排除内部类
            registerAlias(type);
        }
    }
}
```

### 7.4 resolveAlias

`#resolveAlias(String string)` 方法，获得别名对应的类型。代码如下：

```Java
// TypeAliasRegistry.java

public <T> Class<T> resolveAlias(String string) {
    try {
        if (string == null) {
            return null;
        }
        // issue #748
        // <1> 转换成小写
        String key = string.toLowerCase(Locale.ENGLISH);
        Class<T> value;
        // <2.1> 首先，从 TYPE_ALIASES 中获取
        if (TYPE_ALIASES.containsKey(key)) {
            value = (Class<T>) TYPE_ALIASES.get(key);
        // <2.2> 其次，直接获得对应类
        } else {
            value = (Class<T>) Resources.classForName(string);
        }
        return value;
    } catch (ClassNotFoundException e) { // <2.3> 异常
        throw new TypeException("Could not resolve type alias '" + string + "'.  Cause: " + e, e);
    }
}
```

- `<1>` 处，将别名转换成**小写**。
- `<2.1>` 处，首先，从 `TYPE_ALIASES` 中获取对应的类型。
- `<2.2>` 处，其次，直接获取对应的类。所以，这个方法，同时处理了别名与全类名两种情况。
- `<2.3>` 处，最差，找不到对应的类，发生异常，抛出 TypeException 异常。

## 8. SimpleTypeRegistry

`org.apache.ibatis.type.SimpleTypeRegistry` ，简单类型注册表。代码如下：

```java
// SimpleTypeRegistry.java

public class SimpleTypeRegistry {

    /**
     * 简单类型的集合
     */
    private static final Set<Class<?>> SIMPLE_TYPE_SET = new HashSet<>();

    // 初始化常用类到 SIMPLE_TYPE_SET 中
    static {
        SIMPLE_TYPE_SET.add(String.class);
        SIMPLE_TYPE_SET.add(Byte.class);
        SIMPLE_TYPE_SET.add(Short.class);
        SIMPLE_TYPE_SET.add(Character.class);
        SIMPLE_TYPE_SET.add(Integer.class);
        SIMPLE_TYPE_SET.add(Long.class);
        SIMPLE_TYPE_SET.add(Float.class);
        SIMPLE_TYPE_SET.add(Double.class);
        SIMPLE_TYPE_SET.add(Boolean.class);
        SIMPLE_TYPE_SET.add(Date.class);
        SIMPLE_TYPE_SET.add(Class.class);
        SIMPLE_TYPE_SET.add(BigInteger.class);
        SIMPLE_TYPE_SET.add(BigDecimal.class);
    }

    private SimpleTypeRegistry() {
        // Prevent Instantiation
    }

    /*
     * Tells us if the class passed in is a known common type
     *
     * @param clazz The class to check
     * @return True if the class is known
     */
    public static boolean isSimpleType(Class<?> clazz) {
        return SIMPLE_TYPE_SET.contains(clazz);
    }

}
```

## 9. ByteArrayUtils

`org.apache.ibatis.type.ByteArrayUtils` ，Byte 数组的工具类。代码如下：

```java
// ByteArrayUtils.java

class ByteArrayUtils {

    private ByteArrayUtils() {
        // Prevent Instantiation
    }

    // Byte[] => byte[]
    static byte[] convertToPrimitiveArray(Byte[] objects) {
        final byte[] bytes = new byte[objects.length];
        for (int i = 0; i < objects.length; i++) {
            bytes[i] = objects[i];
        }
        return bytes;
    }

    // byte[] => Byte[]
    static Byte[] convertToObjectArray(byte[] bytes) {
        final Byte[] objects = new Byte[bytes.length];
        for (int i = 0; i < bytes.length; i++) {
            objects[i] = bytes[i];
        }
        return objects;
    }

}
```

## 666. 彩蛋

`type` 模块的代码，还是相对多的，不过比较简单的。

参考和推荐如下文章：

- 祖大俊 [《Mybatis3.3.x技术内幕（十二）：Mybatis之TypeHandler》](https://my.oschina.net/zudajun/blog/671075)
- 徐郡明 [《MyBatis 技术内幕》](https://item.jd.com/12125531.html) 的 [「2.3 类型转换」](http://svip.iocoder.cn/MyBatis/type-package/#) 小节