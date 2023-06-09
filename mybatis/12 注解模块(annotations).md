# 精尽 MyBatis 源码分析 —— 注解模块

## 1. *概述

本文，我们来分享 MyBatis 的注解模块，对应 `annotations` 包。如下图所示：[![`annotations` 包](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202201141543422.png)](http://static.iocoder.cn/images/MyBatis/2020_02_04/01.png)`annotations` 包

在 [《精尽 MyBatis 源码解析 —— 项目结构一览》](http://svip.iocoder.cn/MyBatis/intro) 中，简单介绍了这个模块如下：

> 随着 Java 注解的慢慢流行，MyBatis 提供了**注解**的方式，使得我们方便的在 Mapper 接口上编写简单的数据库 SQL 操作代码，而无需像之前一样，必须编写 SQL 在 XML 格式的 Mapper 文件中。虽然说，实际场景下，大家还是喜欢在 XML 格式的 Mapper 文件中编写响应的 SQL 操作。

注解比较多，艿艿尽量对它们的用途，进行规整。

另外，想要看 MyBatis 注解文档的胖友，可以看看 [《MyBatis 文档 —— Java API》](http://www.mybatis.org/mybatis-3/zh/java-api.html) 。

> 艿艿的补充：在写完本文后，发现田守枝对注解的整理更好，引用如下：
>
> FROM [《mybatis 注解配置详解》](http://www.tianshouzhi.com/api/tutorials/mybatis/393)
>
> - **增删改查：** @Insert、@Update、@Delete、@Select、@MapKey、@Options、@SelelctKey、@Param、@InsertProvider、@UpdateProvider、@DeleteProvider、@SelectProvider
> - **结果集映射：** @Results、@Result、@ResultMap、@ResultType、@ConstructorArgs、@Arg、@One、@Many、@TypeDiscriminator、@Case
> - **缓存：** @CacheNamespace、@Property、@CacheNamespaceRef、@Flush

## 2. CRUD 常用操作注解

示例如下：

```java
package com.whut.inter;
import java.util.List;
import org.apache.ibatis.annotations.Delete;
import org.apache.ibatis.annotations.Insert;
import org.apache.ibatis.annotations.Param;
import org.apache.ibatis.annotations.Select;
import org.apache.ibatis.annotations.Update;
import com.whut.model.User;

// 最基本的注解CRUD
public interface IUserDAO {

    @Select("select *from User")
    public List<User> retrieveAllUsers();
                                                                                                                                                                                                                                  
    //注意这里只有一个参数，则#{}中的标识符可以任意取
    @Select("select *from User where id=#{idss}")
    public User retrieveUserById(int id);
                                                                                                                                                                                                                                  
    @Select("select *from User where id=#{id} and userName like #{name}")
    public User retrieveUserByIdAndName(@Param("id")int id,@Param("name")String names);
                                                                                                                                                                                                                                  
    @Insert("INSERT INTO user(userName,userAge,userAddress) VALUES(#{userName},"
            + "#{userAge},#{userAddress})")
    public void addNewUser(User user);
                                                                                                                                                                                                                                  
    @Delete("delete from user where id=#{id}")
    public void deleteUser(int id);
                                                                                                                                                                                                                                  
    @Update("update user set userName=#{userName},userAddress=#{userAddress}"
            + " where id=#{id}")
    public void updateUser(User user);
    
}
```

### 2.1 @Select

`org.apache.ibatis.annotations.@Select` ，查询语句注解。代码如下：

```java
// Select.java

@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD) // 方法
public @interface Select {

    /**
     * @return 查询语句
     */
    String[] value();

}
```

### 2.2 @Insert

`org.apache.ibatis.annotations.@Insert` ，插入语句注解。代码如下：

```java
// Insert.java

@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD) // 方法
public @interface Insert {

    /**
     * @return 插入语句
     */
    String[] value();

}
```

### 2.3 @Update

`org.apache.ibatis.annotations.@Update` ，更新语句注解。代码如下：

```java
// Update.java

@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Update {

    /**
     * @return 更新语句
     */
    String[] value();

}
```

### 2.4 @Delete

`org.apache.ibatis.annotations.@Delete` ，删除语句注解。代码如下：

```java
// Delete.java

@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD) // 方法
public @interface Delete {

    /**
     * @return 删除语句
     */
    String[] value();

}
```

### 2.5 @Param

`org.apache.ibatis.annotations.@Param` ，方法参数名的注解。代码如下：

> 当映射器方法需多个参数，这个注解可以被应用于映射器方法参数来给每个参数一个名字。否则，多参数将会以它们的顺序位置来被命名。比如 `#{1}`，`#{2}` 等，这是默认的。
>
> 使用 `@Param("person")` ，SQL 中参数应该被命名为 `#{person}` 。

```java
// Param.java

@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.PARAMETER) // 参数
public @interface Param {

    /**
     * @return 参数名
     */
    String value();

}
```

## 3. CRUD 高级操作注解

示例如下：

- IBlogDAO 接口：

  ```java
  package com.whut.inter;
  import java.util.List;
  import org.apache.ibatis.annotations.CacheNamespace;
  import org.apache.ibatis.annotations.DeleteProvider;
  import org.apache.ibatis.annotations.InsertProvider;
  import org.apache.ibatis.annotations.Options;
  import org.apache.ibatis.annotations.Param;
  import org.apache.ibatis.annotations.Result;
  import org.apache.ibatis.annotations.ResultMap;
  import org.apache.ibatis.annotations.Results;
  import org.apache.ibatis.annotations.SelectProvider;
  import org.apache.ibatis.annotations.UpdateProvider;
  import org.apache.ibatis.type.JdbcType;
  import com.whut.model.Blog;
  import com.whut.sqlTool.BlogSqlProvider;
  
  @CacheNamespace(size=100)
  public interface IBlogDAO {
  
      @SelectProvider(type = BlogSqlProvider.class, method = "getSql") 
      @Results(value ={ 
              @Result(id=true, property="id",column="id",javaType=Integer.class,jdbcType=JdbcType.INTEGER),
              @Result(property="title",column="title",javaType=String.class,jdbcType=JdbcType.VARCHAR),
              @Result(property="date",column="date",javaType=String.class,jdbcType=JdbcType.VARCHAR),
              @Result(property="authername",column="authername",javaType=String.class,jdbcType=JdbcType.VARCHAR),
              @Result(property="content",column="content",javaType=String.class,jdbcType=JdbcType.VARCHAR),
              }) 
      public Blog getBlog(@Param("id") int id);
                                                                                                                                                                                        
      @SelectProvider(type = BlogSqlProvider.class, method = "getAllSql") 
      @Results(value ={ 
              @Result(id=true, property="id",column="id",javaType=Integer.class,jdbcType=JdbcType.INTEGER),
              @Result(property="title",column="title",javaType=String.class,jdbcType=JdbcType.VARCHAR),
              @Result(property="date",column="date",javaType=String.class,jdbcType=JdbcType.VARCHAR),
              @Result(property="authername",column="authername",javaType=String.class,jdbcType=JdbcType.VARCHAR),
              @Result(property="content",column="content",javaType=String.class,jdbcType=JdbcType.VARCHAR),
              }) 
      public List<Blog> getAllBlog();
                                                                                                                                                                                        
      @SelectProvider(type = BlogSqlProvider.class, method = "getSqlByTitle") 
      @ResultMap(value = "sqlBlogsMap") 
      // 这里调用resultMap，这个是SQL配置文件中的,必须该SQL配置文件与本接口有相同的全限定名
      // 注意文件中的namespace路径必须是使用@resultMap的类路径
      public List<Blog> getBlogByTitle(@Param("title")String title);
                                                                                                                                                                                        
      @InsertProvider(type = BlogSqlProvider.class, method = "insertSql") 
      public void insertBlog(Blog blog);
                                                                                                                                                                                        
      @UpdateProvider(type = BlogSqlProvider.class, method = "updateSql")
      public void updateBlog(Blog blog);
                                                                                                                                                                                        
      @DeleteProvider(type = BlogSqlProvider.class, method = "deleteSql")
      @Options(useCache = true, flushCache = false, timeout = 10000) 
      public void deleteBlog(int ids);
                                                                                                                                                                                        
  }
  ```

- BlogSqlProvider 类：

  ```java
  package com.whut.sqlTool;
  import java.util.Map;
  import static org.apache.ibatis.jdbc.SqlBuilder.*;
  package com.whut.sqlTool;
  import java.util.Map;
  import static org.apache.ibatis.jdbc.SqlBuilder.*;
  
  public class BlogSqlProvider {
  
      private final static String TABLE_NAME = "blog";
      
      public String getSql(Map<Integer, Object> parameter) {
          BEGIN();
          //SELECT("id,title,authername,date,content");
          SELECT("*");
          FROM(TABLE_NAME);
          //注意这里这种传递参数方式，#{}与map中的key对应，而map中的key又是注解param设置的
          WHERE("id = #{id}");
          return SQL();
      }
      
      public String getAllSql() {
          BEGIN();
          SELECT("*");
          FROM(TABLE_NAME);
          return SQL();
      }
      
      public String getSqlByTitle(Map<String, Object> parameter) {
          String title = (String) parameter.get("title");
          BEGIN();
          SELECT("*");
          FROM(TABLE_NAME);
          if (title != null)
              WHERE(" title like #{title}");
          return SQL();
      }
      
      public String insertSql() {
          BEGIN();
          INSERT_INTO(TABLE_NAME);
          VALUES("title", "#{title}");
          //  VALUES("title", "#{tt.title}");
          //这里是传递一个Blog对象的，如果是利用上面tt.方式，则必须利用Param来设置别名
          VALUES("date", "#{date}");
          VALUES("authername", "#{authername}");
          VALUES("content", "#{content}");
          return SQL();
      }
      
      public String deleteSql() {
          BEGIN();
          DELETE_FROM(TABLE_NAME);
          WHERE("id = #{id}");
          return SQL();
      }
      
      public String updateSql() {
          BEGIN();
          UPDATE(TABLE_NAME);
          SET("content = #{content}");
          WHERE("id = #{id}");
          return SQL();
      }
  }
  ```

  - 该示例使用 `org.apache.ibatis.jdbc.SqlBuilder` 来实现 SQL 的拼接与生成。实际上，目前该类已经废弃，推荐使用个的是 `org.apache.ibatis.jdbc.SQL` 类。
  - 具体的 SQL 使用示例，可参见 `org.apache.ibatis.jdbc.SQLTest` 单元测试类。

- Mapper XML 配置：

  ```xml
  <?xml version="1.0" encoding="UTF-8"?>
  <!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
  "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
  <mapper namespace="com.whut.inter.IBlogDAO">
     <resultMap type="Blog" id="sqlBlogsMap">
        <id property="id" column="id"/>
        <result property="title" column="title"/>
        <result property="authername" column="authername"/>
        <result property="date" column="date"/>
        <result property="content" column="content"/>
     </resultMap> 
  </mapper>
  ```

### 3.1 @SelectProvider

`org.apache.ibatis.annotations.@SelectProvider` ，查询语句提供器。代码如下：

```java
// SelectProvider.java

@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD) // 方法
public @interface SelectProvider {

    /**
     * @return 提供的类
     */
    Class<?> type();

    /**
     * @return 提供的方法
     */
    String method();

}
```

- 从上面的使用示例可知，XXXProvider 的用途是，指定一个类( `type` )的指定方法( `method` )，返回使用的 SQL 。并且，该方法可以使用 `Map<String,Object> params` 来作为方法参数，传递参数。

### 3.2 @InsertProvider

`org.apache.ibatis.annotations.@InsertProvider` ，插入语句提供器。代码如下：

```java
// InsertProvider.java

@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD) // 方法
public @interface InsertProvider {

    /**
     * @return 提供的类
     */
    Class<?> type();

    /**
     * @return 提供的方法
     */
    String method();

}
```

### 3.3 @UpdateProvider

`org.apache.ibatis.annotations.@UpdateProvider` ，更新语句提供器。代码如下：

```java
// UpdateProvider.java

@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD) // 方法
public @interface UpdateProvider {

    /**
     * @return 提供的类
     */
    Class<?> type();

    /**
     * @return 提供的方法
     */
    String method();

}
```

### 3.4 @DeleteProvider

`org.apache.ibatis.annotations.@DeleteProvider` ，删除语句提供器。代码如下：

```java
// DeleteProvider.java

@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD) // 方法
public @interface DeleteProvider {

    /**
     * @return 提供的类
     */
    Class<?> type();

    /**
     * @return 提供的方法
     */
    String method();

}
```

### 3.5 @Results

`org.apache.ibatis.annotations.@Results` ，结果的注解。代码如下：

> 对应 XML 标签为 `<resultMap />`

```java
// Results.java

@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Results {

    /**
     * The name of the result map.
     */
    String id() default "";

    /**
     * @return {@link Result} 数组
     */
    Result[] value() default {};

}
```

### 3.6 @Result

`org.apache.ibatis.annotations.@Results` ，结果字段的注解。代码如下：

```java
// Result.java

@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({})
public @interface Result {

    /**
     * @return 是否是 ID 字段
     */
    boolean id() default false;

    /**
     * @return Java 类中的属性
     */
    String property() default "";

    /**
     * @return 数据库的字段
     */
    String column() default "";

    /**
     * @return Java Type
     */
    Class<?> javaType() default void.class;

    /**
     * @return JDBC Type
     */
    JdbcType jdbcType() default JdbcType.UNDEFINED;

    /**
     * @return 使用的 TypeHandler 处理器
     */
    Class<? extends TypeHandler> typeHandler() default UnknownTypeHandler.class;

    /**
     * @return {@link One} 注解
     */
    One one() default @One;

    /**
     * @return {@link Many} 注解
     */
    Many many() default @Many;

}
```

#### 3.6.1 @One

`org.apache.ibatis.annotations.@One` ，复杂类型的单独属性值的注解。代码如下：

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({})
public @interface One {

    /**
     * @return 已映射语句（也就是映射器方法）的全限定名
     */
    String select() default "";

    /**
     * @return 加载类型
     */
    FetchType fetchType() default FetchType.DEFAULT;

}
```

#### 3.6.2 @Many

`org.apache.ibatis.annotations.@Many` ，复杂类型的集合属性值的注解。代码如下：

```java
// Many.java

@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({})
public @interface Many {

    /**
     * @return 已映射语句（也就是映射器方法）的全限定名
     */
    String select() default "";

    /**
     * @return 加载类型
     */
    FetchType fetchType() default FetchType.DEFAULT;

}
```

### 3.7 @ResultMap

`org.apache.ibatis.annotations.@ResultMap` ，使用的结果集的注解。代码如下：

```java
// ResultMap.java

@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD) // 方法
public @interface ResultMap {

    /**
     * @return 结果集
     */
    String[] value();

}
```

- 例如上述示例的 `#getBlogByTitle(@Param("title")String title)` 方法，使用的注解为 `@ResultMap(value = "sqlBlogsMap")`，而 `"sqlBlogsMap"` 中 Mapper XML 中有相关的定义。

### 3.8 @ResultType

`org.apache.ibatis.annotations.@ResultType` ，结果类型。代码如下：

```java
// ResultType.java

@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface ResultType {

    /**
     * @return 类型
     */
    Class<?> value();

}
```

### 3.9 @CacheNamespace

`org.apache.ibatis.annotations.@CacheNamespace` ，缓存空间配置的注解。代码如下：

> 对应 XML 标签为 `<cache />`

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE) // Mapper 类上
public @interface CacheNamespace {

    /**
     * @return 负责存储的 Cache 实现类
     */
    Class<? extends org.apache.ibatis.cache.Cache> implementation() default PerpetualCache.class;

    /**
     * @return 负责过期的 Cache 实现类
     */
    Class<? extends org.apache.ibatis.cache.Cache> eviction() default LruCache.class;

    /**
     * @return 清空缓存的频率。0 代表不清空
     */
    long flushInterval() default 0;

    /**
     * @return 缓存容器大小
     */
    int size() default 1024;

    /**
     * @return 是否序列化。{@link org.apache.ibatis.cache.decorators.SerializedCache}
     */
    boolean readWrite() default true;

    /**
     * @return 是否阻塞。{@link org.apache.ibatis.cache.decorators.BlockingCache}
     */
    boolean blocking() default false;

    /**
     * Property values for a implementation object.
     * @since 3.4.2
     *
     * {@link Property} 数组
     */
    Property[] properties() default {};

}
```

#### 3.9.1 @Property

`org.apache.ibatis.annotations.@Property` ，属性的注解。代码如下：

```java
// Property.java

@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({})
public @interface Property {

    /**
     * 属性名
     *
     * A target property name
     */
    String name();

    /**
     * 属性值
     *
     * A property value or placeholder
     */
    String value();

}
```

### 3.10 @CacheNamespaceRef

`org.apache.ibatis.annotations.@CacheNamespaceRef` ，指向指定命名空间的注解。代码如下：

> 对应 XML 标签为 `<cache-ref />`

```java
// CacheNamespaceRef.java

@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE) // 类型
public @interface CacheNamespaceRef {

    /**
     * 见 {@link MapperAnnotationBuilder#parseCacheRef()} 方法
     *
     * A namespace type to reference a cache (the namespace name become a FQCN of specified type)
     */
    Class<?> value() default void.class;

    /**
     * 指向的命名空间
     *
     * A namespace name to reference a cache
     * @since 3.4.2
     */
    String name() default "";

}
```

### 3.11 *@Options

`org.apache.ibatis.annotations.@Options` ，操作可选项。代码如下：

```java
// Options.java

@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Options {

    /**
     * The options for the {@link Options#flushCache()}.
     * The default is {@link FlushCachePolicy#DEFAULT}
     */
    enum FlushCachePolicy {
        /** <code>false</code> for select statement; <code>true</code> for insert/update/delete statement. */
        DEFAULT,
        /** Flushes cache regardless of the statement type. */
        TRUE,
        /** Does not flush cache regardless of the statement type. */
        FALSE
    }

    /**
     * @return 是否使用缓存
     */
    boolean useCache() default true;

    /**
     * @return 刷新缓存的策略
     */
    FlushCachePolicy flushCache() default FlushCachePolicy.DEFAULT;

    /**
     * @return 结果类型
     */
    ResultSetType resultSetType() default ResultSetType.DEFAULT;

    /**
     * @return 语句类型
     */
    StatementType statementType() default StatementType.PREPARED;

    /**
     * @return 加载数量
     */
    int fetchSize() default -1;

    /**
     * @return 超时时间
     */
    int timeout() default -1;

    /**
     * @return 是否生成主键
     */
    boolean useGeneratedKeys() default false;

    /**
     * @return 主键在 Java 类中的属性
     */
    String keyProperty() default "";

    /**
     * @return 主键在数据库中的字段
     */
    String keyColumn() default "";

    /**
     * @return 结果集
     */
    String resultSets() default "";

}
```

- 通过 `useGeneratedKeys` + `keyProperty` + `keyColumn` 属性，可实现返回自增 ID 。示例见 [《【MyBatis】 MyBatis修炼之八 MyBatis 注解方式的基本用法》的 「返回自增主键」 小节](https://www.jianshu.com/p/b5f823ac5355) 。

### 3.12 *@SelectKey

`org.apache.ibatis.annotations.@SelectKey` ，通过 SQL 语句获得主键的注解。代码如下：

```java
// SelectKey.java

@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface SelectKey {

    /**
     * @return 语句
     */
    String[] statement();

    /**
     * @return {@link #statement()} 的类型
     */
    StatementType statementType() default StatementType.PREPARED;

    /**
     * @return Java 对象的属性
     */
    String keyProperty();

    /**
     * @return 数据库的字段
     */
    String keyColumn() default "";

    /**
     * @return 在插入语句执行前，还是执行后
     */
    boolean before();

    /**
     * @return 返回类型
     */
    Class<?> resultType();

}
```

- 具体使用示例，可见 [《【MyBatis】 MyBatis修炼之八 MyBatis 注解方式的基本用法》的 「返回非自增主键」 小节](https://www.jianshu.com/p/b5f823ac5355) 。

### 3.13 *@MapKey

`org.apache.ibatis.annotations.@MapKey` ，Map 结果的键的注解。代码如下：

```java
// MapKey.java

@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD) // 方法上
public @interface MapKey {

    /**
     * @return 键名
     */
    String value();

}
```

- 这个注解看的艿艿一脸懵逼，具体使用示例，见 [《MyBatis使用@MapKey注解接收多个查询记录到Map中，以便方便地用get()方法获取字段的值》](https://blog.csdn.net/ClementAD/article/details/50589459) 。

### 3.14 @Flush

`org.apache.ibatis.annotations.@Flush` ，Flush 注解。代码如下：

> 如果使用了这个注解，定义在 Mapper 接口中的方法能够调用 `SqlSession#flushStatements()` 方法。（Mybatis 3.3及以上）

```java
// Flush.java

@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD) // 方法上
public @interface Flush {
}
```

## 4. 其它注解

### 4.1 *@Mapper

`org.apache.ibatis.annotations.Mapper` ，标记这是个 Mapper 的注解。代码如下：

```java
// Mapper.java

@Documented
@Inherited
@Retention(RUNTIME)
@Target({TYPE, METHOD, FIELD, PARAMETER})
public @interface Mapper {

    // Interface Mapper

}
```

- 使用示例，见 [《MyBatis中的@Mapper注解及配套注解使用详解（上）》](https://blog.csdn.net/phenomenonsTell/article/details/79033144) 。

### 4.2 *@Lang

`org.apache.ibatis.annotations.@Lang` ，语言驱动的注解。代码如下：

```java
// Lang.java

@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD) // 方法
public @interface Lang {

    /**
     * @return 驱动类
     */
    Class<? extends LanguageDriver> value();

}
```

- 具体使用示例，可见 [《增强MyBatis注解》的 「自定义Select In注解」 小节](https://www.jianshu.com/p/03642b807688) 。

### 4.3 暂时省略

如下几个注解，暂时省略，使用较少。

- `@TypeDiscriminator` + `@Case`
- `@ConstructorArgs` + `@Arg`
- `@AutomapConstructor`

## 666. *彩蛋

实际代码编写时，还是推荐使用 XML 的方式，而不是注解的方式。因为，好统一维护。

参考和推荐如下文章：

- 田守枝 [《mybayis注解配置详解》](http://www.tianshouzhi.com/api/tutorials/mybatis/393)
- 开心跳蚤 [《【MyBatis】 MyBatis修炼之八 MyBatis 注解方式的基本用法》](https://www.jianshu.com/p/b5f823ac5355)
- zhao_xiao_long [《MyBatis注解Annotation介绍及Demo》](http://blog.51cto.com/computerdragon/1399742)