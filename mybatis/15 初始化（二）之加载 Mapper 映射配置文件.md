# 精尽 MyBatis 源码分析 —— MyBatis 初始化（二）之加载 Mapper 映射配置文件

## 1. *概述

本文接 [《精尽 MyBatis 源码分析 —— MyBatis 初始化（一）之加载 mybatis-config》](http://svip.iocoder.cn/MyBatis/builder-package-1) 一文，来分享 MyBatis 初始化的第二步，**加载 Mapper 映射配置文件**。而这个步骤的入口是 XMLMapperBuilder 。下面，我们一起来看看它的代码实现。

> FROM [《Mybatis3.3.x技术内幕（八）：Mybatis初始化流程（上）》](https://my.oschina.net/zudajun/blog/668738)
>
> [![解析](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/7y6Ttk4EnVWwrGX.png)](http://static.iocoder.cn/images/MyBatis/2020_02_13/01.png)解析

- 上图，就是 Mapper 映射配置文件的解析结果。

## 2. XMLMapperBuilder

`org.apache.ibatis.builder.xml.XMLMapperBuilder` ，继承 BaseBuilder 抽象类，Mapper XML 配置构建器，主要负责解析 Mapper 映射配置文件。

### 2.1 构造方法

#### 2.1.1 测试示例

```xml
<mapper resource="org/apache/ibatis/builder/BlogMapper.xml"/>
```

```java
// 调用路径  org.apache.ibatis.builder.xml.XMLConfigBuilder#mapperElement
XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, resource, configuration.getSqlFragments());
 mapperParser.parse();
```

#### 2.1.2 方法

```java
// XMLMapperBuilder.java

/**
 * 基于 Java XPath 解析器
 */
private final XPathParser parser;
/**
 * Mapper 构造器助手
 */
private final MapperBuilderAssistant builderAssistant;
/**
 * 可被其他语句引用的可重用语句块的集合
 *
 * 例如：<sql id="userColumns"> ${alias}.id,${alias}.username,${alias}.password </sql>
 */
private final Map<String, XNode> sqlFragments;
/**
 * 资源引用的地址
 */
private final String resource;

public XMLMapperBuilder(InputStream inputStream, Configuration configuration, String resource, Map<String, XNode> sqlFragments, String namespace) {
    this(inputStream, configuration, resource, sqlFragments);
    this.builderAssistant.setCurrentNamespace(namespace);
}

public XMLMapperBuilder(InputStream inputStream, Configuration configuration, String resource, Map<String, XNode> sqlFragments) {
    this(new XPathParser(inputStream, true, configuration.getVariables(), new XMLMapperEntityResolver()),
            configuration, resource, sqlFragments);
}

// 测试用例 走的该方法
private XMLMapperBuilder(XPathParser parser, Configuration configuration, String resource, Map<String, XNode> sqlFragments) {
    super(configuration);
    // 创建 MapperBuilderAssistant 对象
    this.builderAssistant = new MapperBuilderAssistant(configuration, resource);
    this.parser = parser;
    this.sqlFragments = sqlFragments;
    this.resource = resource;
}
```

- `builderAssistant` 属性，MapperBuilderAssistant 对象，是 XMLMapperBuilder 和 MapperAnnotationBuilder 的小助手，提供了一些公用的方法，例如创建 ParameterMap、MappedStatement 对象等等。关于 MapperBuilderAssistant 类，可见 [「3. MapperBuilderAssistant」](#3. MapperBuilderAssistant) 。

### 2.2 parse

#### 2.2.1 测试用例

```xml
<mapper namespace="org.apache.ibatis.domain.blog.mappers.BlogMapper">

  <cache-ref namespace="org.apache.ibatis.submitted.xml_external_ref.MultipleCrossIncludePetMapper" />

  <!-- One hour cache -->
  <cache flushInterval="3600000"/>

  <resultMap id="blogWithPosts1" type="Blog">
    <id property="id" column="id"/>
    <result property="title" column="title"/>
    <association property="author" column="author_id"
             select="org.apache.ibatis.domain.blog.mappers.AuthorMapper.selectAuthorWithInlineParams"/>
    <collection property="posts" column="id" select="selectPostsForBlog"/>
  </resultMap>

  <select id="selectBlogWithPostsUsingSubSelect" parameterType="int" resultMap="blogWithPosts1">
    select * from Blog where id = #{id}
  </select>

  <resultMap id="blogWithPosts2" type="Blog">
    <id property="id" column="id"/>
    <result property="title" column="title"/>
    <association property="author"  column="author_id" resultMap="joinedAuthor"/>
    <collection property="posts" column="id" select="selectPostsForBlog"/>
  </resultMap>

  <resultMap id="joinedAuthor" type="org.apache.ibatis.domain.blog.Author">
    <id property="id" column="author_id"/>
    <result property="username" column="author_username"/>
    <result property="password" column="author_password"/>
    <result property="email" column="author_email"/>
    <result property="bio" column="author_bio"/>
    <result property="favouriteSection" column="author_favourite_section"/>
  </resultMap>

  <select id="selectBlogWithPostsForId" parameterType="int" resultMap="blogWithPosts2">
      select
        B.id as id,
        B.title as title,
        A.id as author_id,
        A.author_username as author_username
      from Blog B
             left outer join Author A on B.author_id = A.id
      where B.id = #{id}
  </select>

  <select id="selectAllPosts" resultType="hashmap">
    select * from post order by id
  </select>

</mapper>
```

#### 2.2.2 正文

`#parse()` 方法，解析 Mapper XML 配置文件。代码如下：

```java
// XMLMapperBuilder.java

public void parse() {
    // <1> 判断当前 Mapper 是否已经加载过
    if (!configuration.isResourceLoaded(resource)) {  // 参考示例 ：org/apache/ibatis/builder/BlogMapper.xml
        // <2> 解析 `<mapper />` 节点   见 |- 2.3 configurationElement
        configurationElement(parser.evalNode("/mapper"));
        // <3> 标记该 Mapper 已经加载过
        configuration.addLoadedResource(resource);
        // <4> 绑定 Mapper   见 |- 2.3 bindMapperForNamespace
        bindMapperForNamespace();
    }

    // 见 2.5 parsePendingxxx
    // <5> 解析待定的 <resultMap /> 节点
    parsePendingResultMaps();
    // <6> 解析待定的 <cache-ref /> 节点
    parsePendingCacheRefs();
    // <7> 解析待定的 SQL 语句的节点
    parsePendingStatements();
}
```

- `<1>` 处，调用 `Configuration#isResourceLoaded(String resource)` 方法，判断当前 Mapper 是否已经加载过。代码如下：

  ```java
  // Configuration.java
  
  /**
   * 已加载资源( Resource )集合
   */
  protected final Set<String> loadedResources = new HashSet<>();
  
  public boolean isResourceLoaded(String resource) {
      return loadedResources.contains(resource);
  }
  ```

- `<3>` 处，调用 `Configuration#addLoadedResource(String resource)` 方法，标记该 Mapper 已经加载过。代码如下：

  ```java
  // Configuration.java
  
  public void addLoadedResource(String resource) {
      loadedResources.add(resource);
  }
  ```

- <span id='go2.3_2'>`<2>` </span>处，调用 `#configurationElement(XNode context)` 方法，解析 `<mapper />` 节点。详细解析，见 [「2.3 configurationElement」](#2.3 configurationElement) 。

- `<4>` 处，调用 `#bindMapperForNamespace()` 方法，绑定 Mapper 。详细解析，

- <span id='go2.3_5'>`<5>` </span>、`<6>`、`<7>` 处，解析对应的**待定**的节点。详细解析，见 [「2.5 parsePendingXXX」](#2.5 parsePendingXXX) 。

### 2.3 configurationElement

`#configurationElement(XNode context)` 方法，解析 `<mapper />` 节点。代码如下：[<-](#go2.3_2)

```java
// XMLMapperBuilder.java

private void configurationElement(XNode context) {
    try {
        // <1> 获得 namespace 属性
        String namespace = context.getStringAttribute("namespace"); // 参考示例：org.apache.ibatis.domain.blog.mappers.BlogMapper
        if (namespace == null || namespace.equals("")) {
            throw new BuilderException("Mapper's namespace cannot be empty");
        }
        // <1> 设置 namespace 属性   		 |- 见 [3.2 setCurrentNamespace]
        builderAssistant.setCurrentNamespace(namespace); 
        // <2> 解析 <cache-ref /> 节点  	    |- 见「2.3.1 cacheElement」
        cacheRefElement(context.evalNode("cache-ref"));
        // <3> 解析 <cache /> 节点  		  |- 见 2.3.2 cacheElement」
        cacheElement(context.evalNode("cache"));
        // 已废弃！老式风格的参数映射。内联参数是首选,这个元素可能在将来被移除，这里不会记录。
        parameterMapElement(context.evalNodes("/mapper/parameterMap"));
        // <4> 解析 <resultMap /> 节点们  	 |- 见「 2.3.3 resultMapElements」
        resultMapElements(context.evalNodes("/mapper/resultMap"));
        // <5> 解析 <sql /> 节点们		      |- 见 「2.3.4 sqlElement」	
        sqlElement(context.evalNodes("/mapper/sql"));
        // <6> 解析 <select /> <insert /> <update /> <delete /> 节点们  														 |- 见 「2.3.5 buildStatementFromContext」
        buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
    } catch (Exception e) {
        throw new BuilderException("Error parsing Mapper XML. The XML location is '" + resource + "'. Cause: " + e, e);
    }
}
```

#### 2.3.1 cacheRefElement

`#cacheRefElement(XNode context)` 方法，解析 `<cache-ref />` 节点。代码如下：

```java
// XMLMapperBuilder.java

private void cacheRefElement(XNode context) {
    if (context != null) {
        // <1> 获得指向的 namespace 名字，并添加到 configuration 的 cacheRefMap 中
        configuration.addCacheRef(builderAssistant.getCurrentNamespace(), context.getStringAttribute("namespace"));
        // <2> 创建 CacheRefResolver 对象，并执行解析
        CacheRefResolver cacheRefResolver = new CacheRefResolver(builderAssistant, context.getStringAttribute("namespace"));
        try {
            cacheRefResolver.resolveCacheRef();
        } catch (IncompleteElementException e) {
            // <3> 解析失败，添加到 configuration 的 incompleteCacheRefs 中
            configuration.addIncompleteCacheRef(cacheRefResolver);
        }
    }
}
```

- 示例如下：

  ```xml
  <cache-ref namespace="com.someone.application.data.SomeMapper"/>
  ```

- `<1>` 处，获得指向的 `namespace` 名字，并调用 `Configuration#addCacheRef(String namespace, String referencedNamespace)` 方法，添加到 `configuration` 的 `cacheRefMap` 中。代码如下：

  ```java
  // Configuration.java
  
  /**
   * A map holds cache-ref relationship. The key is the namespace that
   * references a cache bound to another namespace and the value is the
   * namespace which the actual cache is bound to.
   *
   * Cache 指向的映射
   *
   * @see #addCacheRef(String, String)
   * @see org.apache.ibatis.builder.xml.XMLMapperBuilder#cacheRefElement(XNode)
   */
  protected final Map<String, String> cacheRefMap = new HashMap<>();
  
  public void addCacheRef(String namespace, String referencedNamespace) {
      cacheRefMap.put(namespace, referencedNamespace);
  }
  ```

- <span id='go2.3.1_2'>`<2>` </span>处，创建 CacheRefResolver 对象，并调用 `CacheRefResolver#resolveCacheRef()` 方法，执行解析。关于 CacheRefResolver ，在 [「2.3.1.1 CacheRefResolver」](#2.3.1.1  CacheRefResolver) 详细解析。

- `<3>` 处，解析失败，因为此处指向的 Cache 对象可能未初始化，则先调用 `Configuration#addIncompleteCacheRef(CacheRefResolver incompleteCacheRef)` 方法，添加到 `configuration` 的 `incompleteCacheRefs` 中。代码如下：

  ```java
  // Configuration.java
  
  /**
   * CacheRefResolver 集合
   */
  protected final Collection<CacheRefResolver> incompleteCacheRefs = new LinkedList<>();
  
  public void addIncompleteCacheRef(CacheRefResolver incompleteCacheRef) {
      incompleteCacheRefs.add(incompleteCacheRef);
  }
  ```

##### 2.3.1.1  CacheRefResolver

`org.apache.ibatis.builder.CacheRefResolver` ，Cache 指向解析器。代码如下：[<-](#go2.3.1_2) 

```java
// CacheRefResolver.java

public class CacheRefResolver {

    private final MapperBuilderAssistant assistant;
    /**
     * Cache 指向的命名空间
     */
    private final String cacheRefNamespace;

    public CacheRefResolver(MapperBuilderAssistant assistant, String cacheRefNamespace) {
        this.assistant = assistant;
        this.cacheRefNamespace = cacheRefNamespace;
    }

    public Cache resolveCacheRef() {
        return assistant.useCacheRef(cacheRefNamespace);
    }

}
```

- 在 `#resolveCacheRef()` 方法中，会调用 `MapperBuilderAssistant#useCacheRef(String namespace)` 方法，获得指向的 Cache 对象。<span id='go2.3.2'>详细解析</span>，见 [「3.3 useCacheRef」](#3.3 useCacheRef) 。

#### 2.3.2 cacheElement

`#cacheElement(XNode context)` 方法，解析 `cache />` 标签。代码如下：

```java
// XMLMapperBuilder.java

private void cacheElement(XNode context) throws Exception {
    if (context != null) {
        // <1> 获得负责存储的 Cache 实现类
        String type = context.getStringAttribute("type", "PERPETUAL");
        Class<? extends Cache> typeClass = typeAliasRegistry.resolveAlias(type);
        // <2> 获得负责过期的 Cache 实现类
        String eviction = context.getStringAttribute("eviction", "LRU");
        Class<? extends Cache> evictionClass = typeAliasRegistry.resolveAlias(eviction);
        // <3> 获得 flushInterval、size、readWrite、blocking 属性
        Long flushInterval = context.getLongAttribute("flushInterval");
        Integer size = context.getIntAttribute("size");
        boolean readWrite = !context.getBooleanAttribute("readOnly", false);
        boolean blocking = context.getBooleanAttribute("blocking", false);
        // <4> 获得 Properties 属性
        Properties props = context.getChildrenAsProperties();
        // <5> 创建 Cache 对象
        builderAssistant.useNewCache(typeClass, evictionClass, flushInterval, size, readWrite, blocking, props);
    }
}
```

- 示例如下：

  ```java
  // 使用默认缓存
  <cache eviction="FIFO" flushInterval="60000"  size="512" readOnly="true"/>
  
  // 使用自定义缓存
  <cache type="com.domain.something.MyCustomCache">
    <property name="cacheFile" value="/tmp/my-custom-cache.tmp"/>
  </cache>
  ```

- `<1>`、`<2>`、`<3>`、`<4>` 处，见代码注释即可。

- <span id='go2.3.3_5'>`<5>` </span>处，调用 `MapperBuilderAssistant#useNewCache(...)` 方法，创建 Cache 对象。详细解析，见 [「3.4 useNewCache」](#3.4 useNewCache) 中。

#### 2.3.3 resultMapElements

> 老艿艿：开始高能，保持耐心。

整体流程如下：

> FROM [《Mybatis3.3.x技术内幕（十）：Mybatis初始化流程（下）》](https://my.oschina.net/zudajun/blog/669868)
>
> ![](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/03.png)

`#resultMapElements(List<XNode> list)` 方法，解析 `<resultMap />` 节点们。代码如下： [->](#go2.3.3.2_2.1)

```java
// XMLMapperBuilder.java

// 解析 <resultMap /> 节点们
private void resultMapElements(List<XNode> list) throws Exception {
    // 遍历 <resultMap /> 节点们
    for (XNode resultMapNode : list) {
        try {
            // 处理单个 <resultMap /> 节点
            resultMapElement(resultMapNode);
        } catch (IncompleteElementException e) {
            // ignore, it will be retried
        }
    }
}

// 解析 <resultMap /> 节点
private ResultMap resultMapElement(XNode resultMapNode) throws Exception {
    return resultMapElement(resultMapNode, Collections.<ResultMapping>emptyList());
}

// 解析 <resultMap /> 节点
private ResultMap resultMapElement(XNode resultMapNode, List<ResultMapping> additionalResultMappings) throws Exception {
    ErrorContext.instance().activity("processing " + resultMapNode.getValueBasedIdentifier());
    // <1> 获得 id 属性
    String id = resultMapNode.getStringAttribute("id",
            resultMapNode.getValueBasedIdentifier());
    // <1> 获得 type 属性
    String type = resultMapNode.getStringAttribute("type",
            resultMapNode.getStringAttribute("ofType",
                    resultMapNode.getStringAttribute("resultType",
                            resultMapNode.getStringAttribute("javaType"))));
    // <1> 获得 extends 属性
    String extend = resultMapNode.getStringAttribute("extends");
    // <1> 获得 autoMapping 属性
    Boolean autoMapping = resultMapNode.getBooleanAttribute("autoMapping");
    // <1> 解析 type 对应的类
    Class<?> typeClass = resolveClass(type);
    Discriminator discriminator = null;
    // <2> 创建 ResultMapping 集合
    List<ResultMapping> resultMappings = new ArrayList<>();
    resultMappings.addAll(additionalResultMappings);
    // <2> 遍历 <resultMap /> 的子节点
    List<XNode> resultChildren = resultMapNode.getChildren();
    for (XNode resultChild : resultChildren) {
        // <2.1> 处理 <constructor /> 节点
        if ("constructor".equals(resultChild.getName())) {
            processConstructorElement(resultChild, typeClass, resultMappings);
        // <2.2> 处理 <discriminator /> 节点
        } else if ("discriminator".equals(resultChild.getName())) {
            discriminator = processDiscriminatorElement(resultChild, typeClass, resultMappings);
        // <2.3> 处理其它节点
        } else {
            List<ResultFlag> flags = new ArrayList<>();
            if ("id".equals(resultChild.getName())) {
                flags.add(ResultFlag.ID);
            }
            resultMappings.add(buildResultMappingFromContext(resultChild, typeClass, flags));
        }
    }
    // <3> 创建 ResultMapResolver 对象，执行解析
    ResultMapResolver resultMapResolver = new ResultMapResolver(builderAssistant, id, typeClass, extend, discriminator, resultMappings, autoMapping);
    try {
        return resultMapResolver.resolve();
    } catch (IncompleteElementException e) {
        // <4> 解析失败，添加到 configuration 中
        configuration.addIncompleteResultMap(resultMapResolver);
        throw e;
    }
}
```

- `<resultMap />` 标签的解析，是相对复杂的过程，情况比较多，所以胖友碰到不懂的，可以看看 [《MyBatis 文档 —— Mapper XML 文件》](http://www.mybatis.org/mybatis-3/zh/sqlmap-xml.html) 文档。

- `<1>` 处，获得 `id`、`type`、`extends`、`autoMapping` 属性，并解析 `type` 对应的类型。

- `<2>`处，创建 ResultMapping 集合，后遍历`<resultMap />`的子节点们，将每一个子节点解析成一个或多个 ResultMapping 对象，添加到集合中。即如下图所示：
  
  ![ResultMap 与 ResultMapping 的映射](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/roaULRXCy8vWuhQ.png)

  - <span id='go2.3.3_2.1'>`<2.1>` </span>处，调用 `#processConstructorElement(...)` 方法，处理 `<constructor />` 节点。详细解析，见 [「2.3.3.1 processConstructorElement」](#2.3.3.1 processConstructorElement) 。
  - <span id='go2.3.3_2.2'>`<2.2>` </span> 处，调用 `#processDiscriminatorElement(...)` 方法，处理 `<discriminator />` 节点。详细解析，见 [「2.3.3.2 processDiscriminatorElement」](#2.3.3.2 processDiscriminatorElement) 。
  - <span id='go2.3.3_2.3'>`<2.3>` </span> 处，调用 `#buildResultMappingFromContext(XNode context, Class<?> resultType, List<ResultFlag> flags)` 方法，将当前子节点构建成 ResultMapping 对象，并添加到 `resultMappings` 中。详细解析，见 [「2.3.3.3 buildResultMappingFromContext」](#2.3.3.3 buildResultMappingFromContext) 。🌞 这一块，和 [「2.3.3.1 processConstructorElement」](#2.3.3.1 processConstructorElement) 的 `<3>` 是一致的。

- <span id='go2.3.3_3'>`<3>` </span>处，创建 ResultMapResolver 对象，执行解析。关于 ResultMapResolver ，在 [2.3.3.4 ResultMapResolver](#2.3.3.4 ResultMapResolver) 详细解析。

- `<4>` 处，如果解析失败，说明有依赖的信息不全，所以调用 `Configuration#addIncompleteResultMap(ResultMapResolver resultMapResolver)` 方法，添加到 Configuration 的 <span id='jumpIncompleteResultMaps'>`incompleteResultMaps`</span> 中。代码如下：

  ```java
  // Configuration.java
  
  /**
   * ResultMapResolver 集合
   */
  protected final Collection<ResultMapResolver> incompleteResultMaps = new LinkedList<>();
  
  public void addIncompleteResultMap(ResultMapResolver resultMapResolver) {
      incompleteResultMaps.add(resultMapResolver);
  }
  ```

##### 2.3.3.1 processConstructorElement

`#processConstructorElement(XNode resultChild, Class<?> resultType, List<ResultMapping> resultMappings)` 方法，处理 `<constructor />` 节点。代码如下：[<-](#go2.3.3_2.1)

```java
// XMLMapperBuilder.java

private void processConstructorElement(XNode resultChild, Class<?> resultType, List<ResultMapping> resultMappings) throws Exception {
    // <1> 遍历 <constructor /> 的子节点们
    List<XNode> argChildren = resultChild.getChildren();
    for (XNode argChild : argChildren) {
        // <2> 获得 ResultFlag 集合
        List<ResultFlag> flags = new ArrayList<>();
        flags.add(ResultFlag.CONSTRUCTOR);
        if ("idArg".equals(argChild.getName())) {
            flags.add(ResultFlag.ID);
        }
        // <3> 将当前子节点构建成 ResultMapping 对象，并添加到 resultMappings 中
        resultMappings.add(buildResultMappingFromContext(argChild, resultType, flags));
    }
}
```

- `<1>` 和 `<3>` 处，遍历 `<constructor />` 的子节点们，调用 `#buildResultMappingFromContext(XNode context, Class<?> resultType, List<ResultFlag> flags)` 方法，将当前子节点构建成 ResultMapping 对象，并添加到 `resultMappings` 中。详细解析，见 [「2.3.3.3 buildResultMappingFromContext」](#2.3.3.3 buildResultMappingFromContext) 。

- `<2>` 处，我们可以看到一个 `org.apache.ibatis.mapping.ResultFlag` 枚举类，结果标识。代码如下：

  ```java
  // ResultFlag.java
  
  public enum ResultFlag {
  
      /**
       * ID
       */
      ID,
      /**
       * 构造方法
       */
      CONSTRUCTOR
  
  }
  ```

  - 具体的用途，见下文。

##### 2.3.3.2 processDiscriminatorElement

`#processDiscriminatorElement(XNode context, Class<?> resultType, List<ResultMapping> resultMappings)` 方法，处理 `<constructor />` 节点。代码如下：[<-](#go2.3.3_2.2)

```java
// XMLMapperBuilder.java

private Discriminator processDiscriminatorElement(XNode context, Class<?> resultType, List<ResultMapping> resultMappings) throws Exception {
    // <1> 解析各种属性
    String column = context.getStringAttribute("column");
    String javaType = context.getStringAttribute("javaType");
    String jdbcType = context.getStringAttribute("jdbcType");
    String typeHandler = context.getStringAttribute("typeHandler");
    // <1> 解析各种属性对应的类
    Class<?> javaTypeClass = resolveClass(javaType);
    Class<? extends TypeHandler<?>> typeHandlerClass = resolveClass(typeHandler);
    JdbcType jdbcTypeEnum = resolveJdbcType(jdbcType);
    // <2> 遍历 <discriminator /> 的子节点，解析成 discriminatorMap 集合
    Map<String, String> discriminatorMap = new HashMap<>();
    for (XNode caseChild : context.getChildren()) {
        String value = caseChild.getStringAttribute("value");
        String resultMap = caseChild.getStringAttribute("resultMap", processNestedResultMappings(caseChild, resultMappings)); // <2.1>
        discriminatorMap.put(value, resultMap);
    }
    // <3> 创建 Discriminator 对象
    return builderAssistant.buildDiscriminator(resultType, column, javaTypeClass, jdbcTypeEnum, typeHandlerClass, discriminatorMap);
}
```

- 可能大家对 `<discriminator />` 标签不是很熟悉，可以打开 [《MyBatis 文档 —— Mapper XML 文件》](http://www.mybatis.org/mybatis-3/zh/sqlmap-xml.html) 文档，然后下【鉴别器】。😈 当然，这块简单了解下就好，实际场景下，艿艿貌似都不知道它的存在，哈哈哈哈。

- `<1>` 处，解析各种属性以及属性对应的类。

- `<2>` 处，遍历 `<discriminator />` 的子节点，解析成 `discriminatorMap` 集合。

- <span id='go2.3.3.2_2.1'>`<2.1>`</span> 处，如果是内嵌的 ResultMap 的情况，则调用 <span id='go2.3.3.2_processNestedResultMappings'>`processNestedResultMappings`</span>`(XNode context, List<ResultMapping> resultMappings)` 方法，处理**内嵌**的 ResultMap 的情况。代码如下：[->](#go2.3.3.4_0)

  ```java
  // XMLMapperBuilder.java
  
  private String processNestedResultMappings(XNode context, List<ResultMapping> resultMappings) throws Exception {
      if ("association".equals(context.getName())
              || "collection".equals(context.getName())
              || "case".equals(context.getName())) {
          if (context.getStringAttribute("select") == null) {
              // 解析，并返回 ResultMap
              ResultMap resultMap = resultMapElement(context, resultMappings);
              return resultMap.getId();
          }
      }
      return null;
  }
  ```

  - 该方法，会“递归”调用 `#resultMapElement(XNode context, List<ResultMapping> resultMappings)` 方法，处理内嵌的 ResultMap 的情况。也就是返回到 [「2.3.3 resultMapElement」](#2.3.3 resultMapElements) 流程。

- <span id='go2.3.3.2_3'>`<3>`</span> 处，调用 `MapperBuilderAssistant#buildDiscriminator(...)` 方法，创建 Discriminator 对象。详细解析，见 [「3.6 buildDiscriminator」](#3.6 buildDiscriminator) 。

##### 2.3.3.3 buildResultMappingFromContext

`#buildResultMappingFromContext(XNode context, Class<?> resultType, List<ResultFlag> flags)` 方法，将当前节点构建成 ResultMapping 对象。代码如下：[<-](#go2.3.3_2.3)

```java
// XMLMapperBuilder.java

private ResultMapping buildResultMappingFromContext(XNode context, Class<?> resultType, List<ResultFlag> flags) throws Exception {
    // <1> 获得各种属性
    String property;
    if (flags.contains(ResultFlag.CONSTRUCTOR)) {
        property = context.getStringAttribute("name");
    } else {
        property = context.getStringAttribute("property");
    }
    String column = context.getStringAttribute("column");
    String javaType = context.getStringAttribute("javaType");
    String jdbcType = context.getStringAttribute("jdbcType");
    String nestedSelect = context.getStringAttribute("select");
    
    // <0>
    String nestedResultMap = context.getStringAttribute("resultMap",
            processNestedResultMappings(context, Collections.emptyList()));
    String notNullColumn = context.getStringAttribute("notNullColumn");
    String columnPrefix = context.getStringAttribute("columnPrefix");
    String typeHandler = context.getStringAttribute("typeHandler");
    String resultSet = context.getStringAttribute("resultSet");
    String foreignColumn = context.getStringAttribute("foreignColumn");
    boolean lazy = "lazy".equals(context.getStringAttribute("fetchType", configuration.isLazyLoadingEnabled() ? "lazy" : "eager"));
    // <1> 获得各种属性对应的类
    Class<?> javaTypeClass = resolveClass(javaType);
    Class<? extends TypeHandler<?>> typeHandlerClass = resolveClass(typeHandler);
    JdbcType jdbcTypeEnum = resolveJdbcType(jdbcType);
    // <2> 构建 ResultMapping 对象
    return builderAssistant.buildResultMapping(resultType, property, column, javaTypeClass, jdbcTypeEnum, nestedSelect, nestedResultMap, notNullColumn, columnPrefix, typeHandlerClass, flags, resultSet, foreignColumn, lazy);
}
```

- <span id='go2.3.3.4_0'>`<0>`</span>处，不为`resultMap`时都需要走该方法[processNestedResultMappings](#go2.3.3.2_processNestedResultMappings)
- `<1>` 处，解析各种属性以及属性对应的类。
- <span id='go2.3.3.4_2'>`<2>` </span>处，调用 `MapperBuilderAssistant#buildResultMapping(...)` 方法，构建 ResultMapping 对象。详细解析，见 [「3.5 buildResultMapping」](#3.5 buildResultMapping) 。

##### 2.3.3.4 ResultMapResolver

`org.apache.ibatis.builder.ResultMapResolver`，ResultMap 解析器。代码如下：[<-](#go2.3.3_3)

```java
// ResultMapResolver.java

public class ResultMapResolver {

    private final MapperBuilderAssistant assistant;
    /**
     * ResultMap 编号
     */
    private final String id;
    /**
     * 类型
     */
    private final Class<?> type;
    /**
     * 继承自哪个 ResultMap
     */
    private final String extend;
    /**
     * Discriminator 对象
     */
    private final Discriminator discriminator;
    /**
     * ResultMapping 集合
     */
    private final List<ResultMapping> resultMappings;
    /**
     * 是否自动匹配
     */
    private final Boolean autoMapping;

    public ResultMapResolver(MapperBuilderAssistant assistant, String id, Class<?> type, String extend, Discriminator discriminator, List<ResultMapping> resultMappings, Boolean autoMapping) {
        this.assistant = assistant;
        this.id = id;
        this.type = type;
        this.extend = extend;
        this.discriminator = discriminator;
        this.resultMappings = resultMappings;
        this.autoMapping = autoMapping;
    }

    public ResultMap resolve() {
        return assistant.addResultMap(this.id, this.type, this.extend, this.discriminator, this.resultMappings, this.autoMapping);
    }

}
```

- 在 `#resolve()` 方法中，会调用 `MapperBuilderAssistant#addResultMap(...)` 方法，创建 <span id='go2.3.4_ResultMap'>`ResultMap `</span>对象。详细解析，见 [「3.7 addResultMap」](#3.7 addResultMap) 。

> 最后：解析的`<ResultMap>`标签转化为了[ResultMapResolver](#2.3.3.4 ResultMapResolver)，并且调用了`resolve（）`方法。解析成功的全部放在了[configuration.resultMap](#3.7 addResultMap),失败的则放在[configuration.IncompleteResultMap](#jumpIncompleteResultMaps)。
>
> **一个ResultMap标签对应一个ResultMap对象。**

#### 2.3.4 sqlElement

`#sqlElement(List<XNode> list)` 方法，解析 `<sql />` 节点们。代码如下：

```java
// XMLMapperBuilder.java

private void sqlElement(List<XNode> list) throws Exception {
    if (configuration.getDatabaseId() != null) {
        sqlElement(list, configuration.getDatabaseId());
    }
    sqlElement(list, null);
    // 上面两块代码，可以简写成 sqlElement(list, configuration.getDatabaseId());
}

private void sqlElement(List<XNode> list, String requiredDatabaseId) throws Exception {
    // <1> 遍历所有 <sql /> 节点
    for (XNode context : list) {
        // <2> 获得 databaseId 属性
        String databaseId = context.getStringAttribute("databaseId");
        // <3> 获得完整的 id 属性，格式为 `${namespace}.${id}` 。
        String id = context.getStringAttribute("id");
        id = builderAssistant.applyCurrentNamespace(id, false);
        // <4> 判断 databaseId 是否匹配
        if (databaseIdMatchesCurrent(id, databaseId, requiredDatabaseId)) {
            // <5> 添加到 sqlFragments 中
            sqlFragments.put(id, context);
        }
    }
}
```

- `<1>` 处，遍历所有 `<sql />` 节点，逐个处理。

- `<2>` 处，获得 `databaseId` 属性。

- `<3>` 处，获得完整的 `id` 属性，格式为 `${namespace}.${id}` 。

- `<4>` 处，调用 `#databaseIdMatchesCurrent(String id, String databaseId, String requiredDatabaseId)` 方法，判断 `databaseId` 是否匹配。代码如下：

  ```java
  // XMLMapperBuilder.java
  
  private boolean databaseIdMatchesCurrent(String id, String databaseId, String requiredDatabaseId) {
      // 如果不匹配，则返回 false
      if (requiredDatabaseId != null) {
          return requiredDatabaseId.equals(databaseId);
      } else {
          // 如果未设置 requiredDatabaseId ，但是 databaseId 存在，说明还是不匹配，则返回 false
          // mmp ，写的好绕
          if (databaseId != null) {
              return false;
          }
          // skip this fragment if there is a previous one with a not null databaseId
          // 判断是否已经存在
          if (this.sqlFragments.containsKey(id)) {
              XNode context = this.sqlFragments.get(id);
              // 若存在，则判断原有的 sqlFragment 是否 databaseId 为空。因为，当前 databaseId 为空，这样两者才能匹配。
              return context.getStringAttribute("databaseId") == null;
          }
      }
      return true;
  }
  ```

- `<5>` 处，添加到 `sqlFragments` 中。因为 `sqlFragments` 是来自 Configuration 的 `sqlFragments` 属性，所以相当于也被添加了。代码如下：

  ```java
  // Configuration.java
  
   /**
   * 可被其他语句引用的可重用语句块的集合
   *
   * 例如：<sql id="userColumns"> ${alias}.id,${alias}.username,${alias}.password </sql>
   */
  protected final Map<String, XNode> sqlFragments = new StrictMap<>("XML fragments parsed from previous mappers");
  ```
  
  
  
  > 代码写的有点绕，但其实要表达的很简单。就行解析`<sql>`标签，判断标签上是否有`databaseId`与指定的是否一致，一致就将其添加到 `Configuration.sqlFragments`中（添加的是`XNode`对象）。

#### 2.3.5 buildStatementFromContext

`#buildStatementFromContext(List<XNode> list)` 方法，解析 `<select />`、`<insert />`、`<update />`、`<delete />` 节点们。代码如下：

```java
// XMLMapperBuilder.java

private void buildStatementFromContext(List<XNode> list) {
    if (configuration.getDatabaseId() != null) {
        buildStatementFromContext(list, configuration.getDatabaseId());
    }
    buildStatementFromContext(list, null);
    // 上面两块代码，可以简写成 buildStatementFromContext(list, configuration.getDatabaseId());
}

private void buildStatementFromContext(List<XNode> list, String requiredDatabaseId) {
    // <1> 遍历 <select /> <insert /> <update /> <delete /> 节点们
    for (XNode context : list) {
        // <1> 创建 XMLStatementBuilder 对象，执行解析
        final XMLStatementBuilder statementParser = new XMLStatementBuilder(configuration, builderAssistant, context, requiredDatabaseId);
        try {
            statementParser.parseStatementNode();
        } catch (IncompleteElementException e) {
            // <2> 解析失败，添加到 configuration 中
            configuration.addIncompleteStatement(statementParser);
        }
    }
}
```

- `<1>` 处，遍历 `<select />`、`<insert />`、`<update />`、`<delete />` 节点们，逐个创建 `XMLStatementBuilder `对象，执行解析。关于 XMLStatementBuilder 类，我们放在下篇文章，详细解析。

- `<2>` 处，解析失败，调用 `Configuration#addIncompleteStatement(XMLStatementBuilder incompleteStatement)` 方法，添加到 `configuration` 中。代码如下：

  ```java
  // Configuration.java
  
  /**
   * XMLStatementBuilder 集合
   */
  protected final Collection<XMLStatementBuilder> incompleteStatements = new LinkedList<>();
  
  public void addIncompleteStatement(XMLStatementBuilder incompleteStatement) {
      incompleteStatements.add(incompleteStatement);
  }
  ```

### 2.4 bindMapperForNamespace

`#bindMapperForNamespace()` 方法，绑定 Mapper 。代码如下：

```java
// XMLMapperBuilder.java

private void bindMapperForNamespace() {
    String namespace = builderAssistant.getCurrentNamespace();
    if (namespace != null) {
        // <1> 获得 Mapper 映射配置文件对应的 Mapper 接口，实际上类名就是 namespace 。嘿嘿，这个是常识。
        Class<?> boundType = null;
        try {
            boundType = Resources.classForName(namespace);
        } catch (ClassNotFoundException e) {
            //ignore, bound type is not required
        }
        if (boundType != null) {
            // <2> 不存在该 Mapper 接口，则进行添加
            if (!configuration.hasMapper(boundType)) {
                // Spring may not know the real resource name so we set a flag
                // to prevent loading again this resource from the mapper interface
                // look at MapperAnnotationBuilder#loadXmlResource
                // <3> 标记 namespace 已经添加，避免 MapperAnnotationBuilder#loadXmlResource(...) 重复加载
                configuration.addLoadedResource("namespace:" + namespace);
                // <4> 添加到 configuration 中
                configuration.addMapper(boundType);
            }
        }
    }
}
```

- `<1>` 处，获得 Mapper 映射配置文件对应的 Mapper 接口，实际上类名就是 `namespace` 。嘿嘿，这个是常识。

- `<2>` 处，调用 `Configuration#hasMapper(Class<?> type)` 方法，判断不存在该 Mapper 接口，则进行绑定。代码如下：

  ```java
  // Configuration.java
  
  /**
   * MapperRegistry 对象
   */
  protected final MapperRegistry mapperRegistry = new MapperRegistry(this);
  
  public boolean hasMapper(Class<?> type) {
      return mapperRegistry.hasMapper(type);
  }
  ```

- `<3>` 处，调用 `Configuration#addLoadedResource(String resource)` 方法，标记 `namespace` 已经添加，避免 `MapperAnnotationBuilder#loadXmlResource(...)` 重复加载。代码如下：

  ```java
  // MapperAnnotationBuilder.java
  
  private void loadXmlResource() {
      // Spring may not know the real resource name so we check a flag
      // to prevent loading again a resource twice
      // this flag is set at XMLMapperBuilder#bindMapperForNamespace
      if (!configuration.isResourceLoaded("namespace:" + type.getName())) {
          // ... 省略创建 XMLMapperBuilder ，进行解析的代码
      }
  }
  ```

- `<4>` 处，调用 `Configuration#addMapper(Class<T> type)` 方法，添加到 `configuration` 的 `mapperRegistry` 中。代码如下：

  ```java
  // Configuration.java
  
  public <T> void addMapper(Class<T> type) {
      mapperRegistry.addMapper(type);
  }
  ```

### 2.5 parsePendingXXX

有三个 parsePendingXXX 方法，代码如下：[<-](#go2.3_5)

```java
// XMLMapperBuilder.java

private void parsePendingResultMaps() {
    // 获得 ResultMapResolver 集合，并遍历进行处理
    Collection<ResultMapResolver> incompleteResultMaps = configuration.getIncompleteResultMaps();
    synchronized (incompleteResultMaps) {
        Iterator<ResultMapResolver> iter = incompleteResultMaps.iterator();
        while (iter.hasNext()) {
            try {
                // 执行解析
                iter.next().resolve();
                // 移除
                iter.remove();
            } catch (IncompleteElementException e) {
                // ResultMap is still missing a resource...
                // 解析失败，不抛出异常
            }
        }
    }
}

private void parsePendingCacheRefs() {
    // 获得 CacheRefResolver 集合，并遍历进行处理
    Collection<CacheRefResolver> incompleteCacheRefs = configuration.getIncompleteCacheRefs();
    synchronized (incompleteCacheRefs) {
        Iterator<CacheRefResolver> iter = incompleteCacheRefs.iterator();
        while (iter.hasNext()) {
            try {
                // 执行解析
                iter.next().resolveCacheRef();
                // 移除
                iter.remove();
            } catch (IncompleteElementException e) {
                // Cache ref is still missing a resource...
            }
        }
    }
}

private void parsePendingStatements() {
    // 获得 XMLStatementBuilder 集合，并遍历进行处理
    Collection<XMLStatementBuilder> incompleteStatements = configuration.getIncompleteStatements();
    synchronized (incompleteStatements) {
        Iterator<XMLStatementBuilder> iter = incompleteStatements.iterator();
        while (iter.hasNext()) {
            try {
                // 执行解析
                iter.next().parseStatementNode();
                // 移除
                iter.remove();
            } catch (IncompleteElementException e) {
                // Statement is still missing a resource...
            }
        }
    }
}
```

- 三个方法的逻辑思路基本一致：1）获得对应的集合；2）遍历集合，执行解析；3）执行成功，则移除出集合；4）执行失败，忽略异常。
- 当然，实际上，此处还是可能有执行解析失败的情况，但是随着每一个 Mapper 配置文件对应的 XMLMapperBuilder 执行一次这些方法，逐步逐步就会被全部解析完。😈

## 3. MapperBuilderAssistant

`org.apache.ibatis.builder.MapperBuilderAssistant` ，继承 BaseBuilder 抽象类，Mapper 构造器的小助手，提供了一些公用的方法，例如创建 ParameterMap、MappedStatement 对象等等。

### 3.1 构造方法

```java
// MapperBuilderAssistant.java

/**
 * 当前 Mapper 命名空间
 */
private String currentNamespace;
/**
 * 资源引用的地址
 */
private final String resource;
/**
 * 当前 Cache 对象
 */
private Cache currentCache;
/**
 * 是否未解析成功 Cache 引用
 */
private boolean unresolvedCacheRef; // issue #676

public MapperBuilderAssistant(Configuration configuration, String resource) {
    super(configuration);
    ErrorContext.instance().resource(resource);
    this.resource = resource;
}
```

- 实际上，😈 如果要不是为了 XMLMapperBuilder 和 MapperAnnotationBuilder 都能调用到这个公用方法，可能都不需要这个类。

### 3.2 setCurrentNamespace

`#setCurrentNamespace(String currentNamespace)` 方法，设置 `currentNamespace` 属性。代码如下：

```java
// MapperBuilderAssistant.java

public void setCurrentNamespace(String currentNamespace) {
    // 如果传入的 currentNamespace 参数为空，抛出 BuilderException 异常
    if (currentNamespace == null) {
        throw new BuilderException("The mapper element requires a namespace attribute to be specified.");
    }

    // 如果当前已经设置，并且还和传入的不相等，抛出 BuilderException 异常
    if (this.currentNamespace != null && !this.currentNamespace.equals(currentNamespace)) {
        throw new BuilderException("Wrong namespace. Expected '"
                + this.currentNamespace + "' but found '" + currentNamespace + "'.");
    }

    // 设置
    this.currentNamespace = currentNamespace;
}
```

### 3.3 useCacheRef

`#useCacheRef(String namespace)` 方法，获得指向的 Cache 对象。如果获得不到，则抛出 IncompleteElementException 异常。代码如下：[<-](#go2.3.2)

```java
// MapperBuilderAssistant.java

public Cache useCacheRef(String namespace) {
    if (namespace == null) {
        throw new BuilderException("cache-ref element requires a namespace attribute.");
    }
    try {
        unresolvedCacheRef = true; // 标记未解决
        // <1> 获得 Cache 对象
        Cache cache = configuration.getCache(namespace);
        // 获得不到，抛出 IncompleteElementException 异常
        if (cache == null) {
            throw new IncompleteElementException("No cache for namespace '" + namespace + "' could be found.");
        }
        // 记录当前 Cache 对象
        currentCache = cache;
        unresolvedCacheRef = false; // 标记已解决
        return cache;
    } catch (IllegalArgumentException e) {
        throw new IncompleteElementException("No cache for namespace '" + namespace + "' could be found.", e);
    }
}
```

- `<1>` 处，调用 `Configuration#getCache(String id)` 方法，获得 Cache 对象。代码如下：

  ```java
  // Configuration.java
  
  /**
   * Cache 对象集合
   *
   * KEY：命名空间 namespace
   */
  protected final Map<String, Cache> caches = new StrictMap<>("Caches collection");
  
  public Cache getCache(String id) {
      return caches.get(id);
  }
  ```

### 3.4 useNewCache

`#useNewCache(Class<? extends Cache> typeClass, Class<? extends Cache> evictionClass, Long flushInterval, Integer size, boolean readWrite, boolean blocking, Properties props)` 方法，创建 Cache 对象。代码如下：[<-](#go2.3.3_5)

```java
// MapperBuilderAssistant.java

public Cache useNewCache(Class<? extends Cache> typeClass,
                         Class<? extends Cache> evictionClass,
                         Long flushInterval,
                         Integer size,
                         boolean readWrite,
                         boolean blocking,
                         Properties props) {
    // <1> 创建 Cache 对象
    Cache cache = new CacheBuilder(currentNamespace)
            .implementation(valueOrDefault(typeClass, PerpetualCache.class))
            .addDecorator(valueOrDefault(evictionClass, LruCache.class))
            .clearInterval(flushInterval)
            .size(size)
            .readWrite(readWrite)
            .blocking(blocking)
            .properties(props)
            .build();
    // <2> 添加到 configuration 的 caches 中
    configuration.addCache(cache);
    // <3> 赋值给 currentCache
    currentCache = cache;
    return cache;
}
```

- `<1>` 处，创建 Cache 对象。关于 CacheBuilder 类，详细解析，见 [「3.4.1 CacheBuilder」](http://svip.iocoder.cn/MyBatis/builder-package-2/#) 。

- `<2>` 处，调用 `Configuration#addCache(Cache cache)` 方法，添加到 `configuration` 的 `caches` 中。代码如下：

  ```java
  // Configuration.java
  
  public void addCache(Cache cache) {
      caches.put(cache.getId(), cache);
  }
  ```

- `<3>` 处，赋值给 `currentCache` 。

#### 3.4.1 CacheBuilder

`org.apache.ibatis.mapping.CacheBuilder` ，Cache 构造器。基于装饰者设计模式，进行 Cache 对象的构造。代码比较简单，但是有点略长，胖友直接点击 [链接](https://github.com/YunaiV/mybatis-3/blob/master/src/main/java/org/apache/ibatis/mapping/CacheBuilder.java) 查看，已经添加了完整的注释。

### 3.5 buildResultMapping

`#buildResultMapping(Class<?> resultType, String property, String column,Class<?> javaType, JdbcType jdbcType, String nestedSelect, String nestedResultMap, String notNullColumn, String columnPrefix, Class<? extends TypeHandler<?>> typeHandler, List<ResultFlag> flags, String resultSet, String foreignColumn, boolean lazy)` 方法，构造 ResultMapping 对象。代码如下：[<-](#go2.3.3.4_2)

```java
// MapperBuilderAssistant.java

/**
  * 构造 ResultMapping 对象。<ResultMap>子节点解析构造（discriminator不解析）
  *
  * @param resultType <ResultMap>上的 类型
  * @param property  子节点的 name | property 属性
  * @param column    子节点的 column 属性
  * @param javaType  子节点的 javaType 属性
  * @param jdbcType  子节点的 jdbcType 属性
  * @param nestedSelect  子节点的 select 属性
  * @param nestedResultMap 处理过的子节点的 resultMap属性（有则用，否则走xxx）
  * @param notNullColumn 子节点的 notNullColumn 属性
  * @param columnPrefix  子节点的 columnPrefix 属性
  * @param typeHandler  子节点的 typeHandler 属性
  * @param flags    子节点是 <constructor> （2个 全部）  该标签的子节点 会再次循环调用 该方法
  *                 其他标签 （只有 ID）
  * @param resultSet   子节点的 resultSet 属性
  * @param foreignColumn  子节点的 foreignColumn 属性
  * @param lazy      子节点的 fetchType 属性(与{@link Configuration#isLazyLoadingEnabled()})有关
  */
public ResultMapping buildResultMapping(
        Class<?> resultType,
        String property,
        String column,
        Class<?> javaType,
        JdbcType jdbcType,
        String nestedSelect,
        String nestedResultMap,
        String notNullColumn,
        String columnPrefix,
        Class<? extends TypeHandler<?>> typeHandler,
        List<ResultFlag> flags,
        String resultSet,
        String foreignColumn,
        boolean lazy) {
    // <1> 解析对应的 Java Type 类和 TypeHandler 对象
    Class<?> javaTypeClass = resolveResultJavaType(resultType, property, javaType);
    TypeHandler<?> typeHandlerInstance = resolveTypeHandler(javaTypeClass, typeHandler);
    // <2> 解析组合字段名称成 ResultMapping 集合。涉及「关联的嵌套查询」
    List<ResultMapping> composites = parseCompositeColumnName(column);
    // <3> 创建 ResultMapping 对象
    return new ResultMapping.Builder(configuration, property, column, javaTypeClass)
            .jdbcType(jdbcType)
            .nestedQueryId(applyCurrentNamespace(nestedSelect, true)) // <3.1>
            .nestedResultMapId(applyCurrentNamespace(nestedResultMap, true)) // <3.1>
            .resultSet(resultSet)
            .typeHandler(typeHandlerInstance)
            .flags(flags == null ? new ArrayList<>() : flags)
            .composites(composites) // 「关联的嵌套查询」
            .notNullColumns(parseMultipleColumnNames(notNullColumn)) // <3.2>
            .columnPrefix(columnPrefix)
            .foreignColumn(foreignColumn)
            .lazy(lazy)
}
```

- `<1>` 处，解析对应的 Java Type 类和 TypeHandler 对象。

- `<2>` 处，调用 `#parseCompositeColumnName(String columnName)` 方法，解析组合字段名称成 ResultMapping 集合。详细解析，见 [「3.5.1 parseCompositeColumnName」](#3.5.1 parseCompositeColumnName) 中。

- `<3>`处，创建 ResultMapping 对象。
  - `<3.1>` 处，调用 `#applyCurrentNamespace(String base, boolean isReference)` 方法，拼接命名空间。详细解析，见 [「3.5.2 applyCurrentNamespace」](#3.5.2 applyCurrentNamespace) 。
  - `<3.2>` 处，调用 `#parseMultipleColumnNames(String notNullColumn)` 方法，将字符串解析成集合。详细解析，见 [「3.5.3 parseMultipleColumnNames」](#3.5.3 parseMultipleColumnNames) 。
  - 关于 ResultMapping 类，在 [「3.5.4 ResultMapping」](#3.5.4 ResultMapping) 中详细解析。

#### 3.5.1 parseCompositeColumnName

`#parseCompositeColumnName(String columnName)` 方法，解析组合字段名称成 ResultMapping 集合。代码如下：

```java
// MapperBuilderAssistant.java

private List<ResultMapping> parseCompositeColumnName(String columnName) {
    List<ResultMapping> composites = new ArrayList<>();
    // 分词，解析其中的 property 和 column 的组合对
    if (columnName != null && (columnName.indexOf('=') > -1 || columnName.indexOf(',') > -1)) {
        StringTokenizer parser = new StringTokenizer(columnName, "{}=, ", false);
        while (parser.hasMoreTokens()) {
            String property = parser.nextToken();
            String column = parser.nextToken();
            // 创建 ResultMapping 对象
            ResultMapping complexResultMapping = new ResultMapping.Builder(
                    configuration, property, column, configuration.getTypeHandlerRegistry().getUnknownTypeHandler()).build();
            // 添加到 composites 中
            composites.add(complexResultMapping);
        }
    }
    return composites;
}
```

- 对于这种情况，官方文档说明如下：

  > FROM [《MyBatis 文档 —— Mapper XML 文件》](http://www.mybatis.org/mybatis-3/zh/sqlmap-xml.html) 的 「关联的嵌套查询」 小节
  >
  > 来自数据库的列名,或重命名的列标签。这和通常传递给 resultSet.getString(columnName)方法的字符串是相同的。 column 注 意 : 要 处 理 复 合 主 键 , 你 可 以 指 定 多 个 列 名 通 过 column= ” {prop1=col1,prop2=col2} ” 这种语法来传递给嵌套查询语 句。这会引起 prop1 和 prop2 以参数对象形式来设置给目标嵌套查询语句。

- 😈 不用理解太细，如果胖友和我一样，基本用不到这个特性。

- ![image-20220119161615039](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202201191616381.png)

  debug对应的结果为：

  ![image-20220119161707017](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202201191617088.png)

#### 3.5.2 applyCurrentNamespace

`#applyCurrentNamespace(String base, boolean isReference)` 方法，拼接命名空间。代码如下：

```java
// MapperBuilderAssistant.java

public String applyCurrentNamespace(String base, boolean isReference) {
    if (base == null) {
        return null;
    }
    if (isReference) {
        // is it qualified with any namespace yet?
        if (base.contains(".")) {
            return base;
        }
    } else {
        // is it qualified with this namespace yet?
        if (base.startsWith(currentNamespace + ".")) {
            return base;
        }
        if (base.contains(".")) {
            throw new BuilderException("Dots are not allowed in element names, please remove it from " + base);
        }
    }
    // 拼接 currentNamespace + base
    return currentNamespace + "." + base;
}
```

- 通过这样的方式，生成**唯一**在的标识。

#### 3.5.3 parseMultipleColumnNames

`#parseMultipleColumnNames(String notNullColumn)` 方法，将字符串解析成集合。代码如下：

```java
// MapperBuilderAssistant.java

private Set<String> parseMultipleColumnNames(String columnName) {
    Set<String> columns = new HashSet<>();
    if (columnName != null) {
        // 多个字段，使用 ，分隔
        if (columnName.indexOf(',') > -1) {
            StringTokenizer parser = new StringTokenizer(columnName, "{}, ", false);
            while (parser.hasMoreTokens()) {
                String column = parser.nextToken();
                columns.add(column);
            }
        } else {
            columns.add(columnName);
        }
    }
    return columns;
}
```

#### 3.5.4 ResultMapping

`org.apache.ibatis.mapping.ResultMapping` ，ResultMap 中的每一条结果字段的映射。代码比较简单，但是有点略长，胖友直接点击 [链接](https://github.com/YunaiV/mybatis-3/blob/master/src/main/java/org/apache/ibatis/mapping/ResultMapping.java) 查看，已经添加了完整的注释。

```java
import org.apache.ibatis.session.Configuration;
import org.apache.ibatis.type.JdbcType;
import org.apache.ibatis.type.TypeHandler;
import org.apache.ibatis.type.TypeHandlerRegistry;
/**
 * {@link ResultMap} 中的每一条结果字段的映射
 */
public class ResultMapping {

    /**
     * MyBatis Configuration 对象
     */
    private Configuration configuration;
    /**
     * Java 对象的属性名
     */
    private String property;
    /**
     * 数据库的字段名
     */
    private String column;
    /**
     * Java 对象的属性的类型
     */
    private Class<?> javaType;
    /**
     * 数据库的字段的类型
     */
    private JdbcType jdbcType;
    /**
     * TypeHandler 对象
     */
    private TypeHandler<?> typeHandler;
    /**
     * 内嵌的 ResultMap 编号
     */
    private String nestedResultMapId;
    /**
     * 内嵌的查询语句编号
     */
    private String nestedQueryId;
    /**
     * 非空字段集合
     */
    private Set<String> notNullColumns;
    /**
     * 当连接多表时，你将不得不使用列别名来避免ResultSet中的重复列名。指定columnPrefix允许你映射列名到一个外部的结果集中。
     */
    private String columnPrefix;
    /**
     * ResultFlag 集合
     */
    private List<ResultFlag> flags;
    /**
     * 组合字段解析后的 ResultMapping 集合
     *
     * {@link org.apache.ibatis.builder.MapperBuilderAssistant#parseCompositeColumnName(String)}
     */
    private List<ResultMapping> composites;
    /**
     * 标识这个将会从哪里加载的复杂类型数据的结果集合的名称
     */
    private String resultSet; // 存储过程相关，忽略
    /**
     * 标识出包含 foreign keys 的列的名称。这个 foreign keys的值将会和父类型中指定的列属性的值相匹配
     */
    private String foreignColumn;
    /**
     * 是否懒加载
     */
    private boolean lazy;

    ResultMapping() {
    }
}
```

```java

    /**
     * 构造器
     */
    public static class Builder {

        private ResultMapping resultMapping = new ResultMapping();

        public Builder(Configuration configuration, String property, String column, TypeHandler<?> typeHandler) {
            this(configuration, property);
            resultMapping.column = column;
            resultMapping.typeHandler = typeHandler;
        }

        public Builder(Configuration configuration, String property, String column, Class<?> javaType) {
            this(configuration, property);
            resultMapping.column = column;
            resultMapping.javaType = javaType;
        }

        public Builder(Configuration configuration, String property) {
            resultMapping.configuration = configuration;
            resultMapping.property = property;
            resultMapping.flags = new ArrayList<>();
            resultMapping.composites = new ArrayList<>();
            resultMapping.lazy = configuration.isLazyLoadingEnabled();
        }

       // 单一字段build省略。

        public ResultMapping build() {
            // lock down collections
            resultMapping.flags = Collections.unmodifiableList(resultMapping.flags);
            resultMapping.composites = Collections.unmodifiableList(resultMapping.composites);
            // 解析 TypeHandler
            resolveTypeHandler();
            // 校验
            validate();
            return resultMapping;
        }

        /**
         * 校验
         */
        private void validate() {
            // Issue #697: cannot define both nestedQueryId and nestedResultMapId
            if (resultMapping.nestedQueryId != null && resultMapping.nestedResultMapId != null) {
                throw new IllegalStateException("Cannot define both nestedQueryId and nestedResultMapId in property " + resultMapping.property);
            }
            // Issue #5: there should be no mappings without typehandler
            if (resultMapping.nestedQueryId == null && resultMapping.nestedResultMapId == null && resultMapping.typeHandler == null) {
                throw new IllegalStateException("No typehandler found for property " + resultMapping.property);
            }
            // Issue #4 and GH #39: column is optional only in nested resultmaps but not in the rest
            if (resultMapping.nestedResultMapId == null && resultMapping.column == null && resultMapping.composites.isEmpty()) {
                throw new IllegalStateException("Mapping is missing column attribute for property " + resultMapping.property);
            }
            if (resultMapping.getResultSet() != null) {
                int numColumns = 0;
                if (resultMapping.column != null) {
                    numColumns = resultMapping.column.split(",").length;
                }
                int numForeignColumns = 0;
                if (resultMapping.foreignColumn != null) {
                    numForeignColumns = resultMapping.foreignColumn.split(",").length;
                }
                if (numColumns != numForeignColumns) {
                    throw new IllegalStateException("There should be the same number of columns and foreignColumns in property " + resultMapping.property);
                }
            }
        }

        /**
         * 解析 TypeHandler
         */
        private void resolveTypeHandler() {
            // 使用 javaType + jdbcType ，获取对应的 TypeHandler 对象
            if (resultMapping.typeHandler == null && resultMapping.javaType != null) {
                Configuration configuration = resultMapping.configuration;
                TypeHandlerRegistry typeHandlerRegistry = configuration.getTypeHandlerRegistry();
                resultMapping.typeHandler = typeHandlerRegistry.getTypeHandler(resultMapping.javaType, resultMapping.jdbcType);
            }
        }

        public Builder column(String column) {
            resultMapping.column = column;
            return this;
        }
    }
```



### 3.6 buildDiscriminator

`#buildDiscriminator(Class<?> resultType, String column, Class<?> javaType, JdbcType jdbcType, Class<? extends TypeHandler<?>> typeHandler, Map<String, String> discriminatorMap)` 方法，构建 Discriminator 对象。代码如下：[<-](#go2.3.3.2_3)

```java
// MapperBuilderAssistant.java

/**
     * 构建 Discriminator 对象。 <ResultMap>中的子节点，为<Discriminator>
     * @param resultType <ResultMap>上的 类型
     * @param column 子节点的 column
     * @param javaType 子节点的 javaType
     * @param jdbcType  子节点的 jdbcType
     * @param typeHandler  子节点的 typeHandler
     * @param discriminatorMap <Discriminator>子节点
     *                         key ->  value属性
     *                         val ->  处理过的resultMap属性（有则用，否则...）
     */
public Discriminator buildDiscriminator(
        Class<?> resultType,
        String column,
        Class<?> javaType,
        JdbcType jdbcType,
        Class<? extends TypeHandler<?>> typeHandler,
        Map<String, String> discriminatorMap) {
    // 构建 ResultMapping 对象
    ResultMapping resultMapping = buildResultMapping(
            resultType,
            null,
            column,
            javaType,
            jdbcType,
            null,
            null,
            null,
            null,
            typeHandler,
            new ArrayList<ResultFlag>(),
            null,
            null,
            false);
    // 创建 namespaceDiscriminatorMap 映射
    Map<String, String> namespaceDiscriminatorMap = new HashMap<>();
    for (Map.Entry<String, String> e : discriminatorMap.entrySet()) {
        String resultMap = e.getValue();
        resultMap = applyCurrentNamespace(resultMap, true); // 生成 resultMap 标识
        namespaceDiscriminatorMap.put(e.getKey(), resultMap);
    }
    // 构建 Discriminator 对象
    return new Discriminator.Builder(configuration, resultMapping, namespaceDiscriminatorMap).build();
}
```

- 简单看看就好，`<discriminator />` 平时用的很少。

#### 3.6.1 Discriminator

`org.apache.ibatis.mapping.Discriminator` ，鉴别器，代码比较简单，胖友直接点击 [链接](https://github.com/YunaiV/mybatis-3/blob/master/src/main/java/org/apache/ibatis/mapping/Discriminator.java) 查看，已经添加了完整的注释。

```java

/**
 * 鉴别器
 *
 *   <discriminator javaType="int" column="vehicle_type">
 *     <case value="1" resultMap="carResult"/>
 *     <case value="2" resultMap="truckResult"/>
 *     <case value="3" resultMap="vanResult"/>
 *     <case value="4" resultMap="suvResult"/>
 *   </discriminator>
 */
public class Discriminator {

    /**
     * ResultMapping 对象
     */
    private ResultMapping resultMapping;
    /**
     * 集合。即注释上的 N 条示例
     *
     * KEY ：value 属性
     * VALUE ：resultMap 属性
     */
    private Map<String, String> discriminatorMap;

    Discriminator() {
    }
}
```

```java
public static class Builder {

    private Discriminator discriminator = new Discriminator();

    public Builder(Configuration configuration, ResultMapping resultMapping, Map<String, String> discriminatorMap) {
        discriminator.resultMapping = resultMapping;
        discriminator.discriminatorMap = discriminatorMap;
    }

    public Discriminator build() {
        assert discriminator.resultMapping != null;
        assert discriminator.discriminatorMap != null;
        assert !discriminator.discriminatorMap.isEmpty();
        // lock down map 生成不可变集合，避免修改
        discriminator.discriminatorMap = Collections.unmodifiableMap(discriminator.discriminatorMap);
        return discriminator;
    }
}
```



### 3.7 addResultMap

> `#addResultMap(String id, Class<?> type, String extend, Discriminator discriminator, List<ResultMapping> resultMappings, Boolean autoMapping)` 方法，创建 ResultMap 对象，并添加到 Configuration 中。
>
> - 该方法[xml配置](#2.3.3.4 ResultMapResolver)调用。

代码如下：[<-](#go2.3.4_ResultMap)

```java
// MapperBuilderAssistant.java

public ResultMap addResultMap(
        String id,
        Class<?> type,
        String extend,
        Discriminator discriminator,
        List<ResultMapping> resultMappings,
        Boolean autoMapping) {
    // <1> 获得 ResultMap 编号，即格式为 `${namespace}.${id}` 。
    id = applyCurrentNamespace(id, false);
    // <2.1> 获取完整的 extend 属性，即格式为 `${namespace}.${extend}` 。从这里的逻辑来看，貌似只能自己 namespace 下的 ResultMap 。
    extend = applyCurrentNamespace(extend, true);

    // <2.2> 如果有父类，则将父类的 ResultMap 集合，添加到 resultMappings 中。
    if (extend != null) {
        // <2.2.1> 获得 extend 对应的 ResultMap 对象。如果不存在，则抛出 IncompleteElementException 异常
        if (!configuration.hasResultMap(extend)) {
            throw new IncompleteElementException("Could not find a parent resultmap with id '" + extend + "'");
        }
        ResultMap resultMap = configuration.getResultMap(extend);
        // 获取 extend 的 ResultMap 对象的 ResultMapping 集合，并移除 resultMappings
        List<ResultMapping> extendedResultMappings = new ArrayList<>(resultMap.getResultMappings());
        extendedResultMappings.removeAll(resultMappings);
        // Remove parent constructor if this resultMap declares a constructor.
        // 判断当前的 resultMappings 是否有构造方法，如果有，则从 extendedResultMappings 移除所有的构造类型的 ResultMapping 们
        boolean declaresConstructor = false;
        for (ResultMapping resultMapping : resultMappings) {
            if (resultMapping.getFlags().contains(ResultFlag.CONSTRUCTOR)) {
                declaresConstructor = true;
                break;
            }
        }
        if (declaresConstructor) {
            extendedResultMappings.removeIf(resultMapping -> resultMapping.getFlags().contains(ResultFlag.CONSTRUCTOR));
        }
        // 将 extendedResultMappings 添加到 resultMappings 中
        resultMappings.addAll(extendedResultMappings);
    }
    // <3> 创建 ResultMap 对象
    ResultMap resultMap = new ResultMap.Builder(configuration, id, type, resultMappings, autoMapping)
            .discriminator(discriminator)
            .build();
    // <4> 添加到 configuration 中
    configuration.addResultMap(resultMap);
    return resultMap;
}
```

- `<1>` 处，获得 ResultMap 编号，即格式为 `${namespace}.${id}` 。

- `<2.1>` 处，获取完整的 extend 属性，即格式为 `${namespace}.${extend}` 。**从这里的逻辑来看，貌似只能自己 namespace 下的 ResultMap** 。

- `<2.2>` 处，如果有父类，则将父类的 ResultMap 集合，添加到 `resultMappings` 中。逻辑有些绕，胖友耐心往下看。

  - `<2.2.1>` 处，获得 `extend` 对应的 ResultMap 对象。如果不存在，则抛出 IncompleteElementException 异常。代码如下：

    ```java
    // Configuration.java
    
    /**
     * ResultMap 的映射
     *
     * KEY：`${namespace}.${id}`
     */
    protected final Map<String, ResultMap> resultMaps = new StrictMap<>("Result Maps collection");
    
    public ResultMap getResultMap(String id) {
        return resultMaps.get(id);
    }
    
    public boolean hasResultMap(String id) {
        return resultMaps.containsKey(id);
    }
    ```

    - x

- `<3>` 处，创建 ResultMap 对象。详细解析，见 [「3.7.1 ResultMap」](http://svip.iocoder.cn/MyBatis/builder-package-2/#) 。

- `<4>` 处， 调用 `Configuration#addResultMap(ResultMap rm)` 方法，添加到 Configuration 的 `resultMaps` 中。代码如下：

  ```java
  // Configuration.java
  
  public void addResultMap(ResultMap rm) {
      // <1> 添加到 resultMaps 中
      resultMaps.put(rm.getId(), rm);
      // 遍历全局的 ResultMap 集合，若其拥有 Discriminator 对象，则判断是否强制标记为有内嵌的 ResultMap
      checkLocallyForDiscriminatedNestedResultMaps(rm);
      // 若传入的 ResultMap 不存在内嵌 ResultMap 并且有 Discriminator ，则判断是否需要强制表位有内嵌的 ResultMap
      checkGloballyForDiscriminatedNestedResultMaps(rm);
  }
  ```

  - `<1>` 处，添加到 `resultMaps` 中。

  - `<2>` 处，调用 `#checkLocallyForDiscriminatedNestedResultMaps(ResultMap rm)` 方法，遍历全局的 ResultMap 集合，若其拥有 Discriminator 对象，则判断是否强制标记为有内嵌的 ResultMap 。代码如下：

    ```java
    // Configuration.java
    
    // Slow but a one time cost. A better solution is welcome.
    protected void checkGloballyForDiscriminatedNestedResultMaps(ResultMap rm) {
        // 如果传入的 ResultMap 有内嵌的 ResultMap
        if (rm.hasNestedResultMaps()) {
            // 遍历全局的 ResultMap 集合
            for (Map.Entry<String, ResultMap> entry : resultMaps.entrySet()) {
                Object value = entry.getValue();
                if (value != null) {
                    ResultMap entryResultMap = (ResultMap) value;
                    // 判断遍历的全局的 entryResultMap 不存在内嵌 ResultMap 并且有 Discriminator
                    if (!entryResultMap.hasNestedResultMaps() && entryResultMap.getDiscriminator() != null) {
                        // 判断是否 Discriminator 的 ResultMap 集合中，使用了传入的 ResultMap 。
                        // 如果是，则标记为有内嵌的 ResultMap
                        Collection<String> discriminatedResultMapNames = entryResultMap.getDiscriminator().getDiscriminatorMap().values();
                        if (discriminatedResultMapNames.contains(rm.getId())) {
                            entryResultMap.forceNestedResultMaps();
                        }
                    }
                }
            }
        }
    }
    ```

    - 逻辑有点绕，胖友耐心看下去。。。

  - `<3>` 处，调用 `#checkLocallyForDiscriminatedNestedResultMaps(ResultMap rm)` 方法，若传入的 ResultMap 不存在内嵌 ResultMap 并且有 Discriminator ，则判断是否需要强制表位有内嵌的 ResultMap 。代码如下：

    ```java
    // Configuration.java
    
    // Slow but a one time cost. A better solution is welcome.
    protected void checkLocallyForDiscriminatedNestedResultMaps(ResultMap rm) {
        // 如果传入的 ResultMap 不存在内嵌 ResultMap 并且有 Discriminator
        if (!rm.hasNestedResultMaps() && rm.getDiscriminator() != null) {
            // 遍历传入的 ResultMap 的 Discriminator 的 ResultMap 集合
            for (Map.Entry<String, String> entry : rm.getDiscriminator().getDiscriminatorMap().entrySet()) {
                String discriminatedResultMapName = entry.getValue();
                if (hasResultMap(discriminatedResultMapName)) {
                    // 如果引用的 ResultMap 存在内嵌 ResultMap ，则标记传入的 ResultMap 存在内嵌 ResultMap
                    ResultMap discriminatedResultMap = resultMaps.get(discriminatedResultMapName);
                    if (discriminatedResultMap.hasNestedResultMaps()) {
                        rm.forceNestedResultMaps();
                        break;
                    }
                }
            }
        }
    }
    ```

    - 逻辑有点绕，胖友耐心看下去。。。整体逻辑，和 `#checkGloballyForDiscriminatedNestedResultMaps(ResultMap rm)` 方法是**类似**的，互为“倒影”。

#### 3.7.1 ResultMap

`org.apache.ibatis.mapping.ResultMap` ，结果集，例如 `<resultMap />` 解析后的对象。代码比较简单，但是有点略长，胖友直接点击 [链接](https://github.com/YunaiV/mybatis-3/blob/master/src/main/java/org/apache/ibatis/mapping/ResultMap.java) 查看，已经添加了完整的注释。

```java

import org.apache.ibatis.logging.LogFactory;
import org.apache.ibatis.reflection.ParamNameUtil;
import org.apache.ibatis.session.Configuration;
import java.lang.annotation.Annotation;
import java.lang.reflect.Constructor;
/**
 * 结果集，例如 <resultMap /> 解析后的对象
 */
public class ResultMap {

    /**
     * Configuration 对象
     */
    private Configuration configuration;

    /**
     * ResultMap 对象
     */
    private String id;
    /**
     * 类型
     */
    private Class<?> type;
    /**
     * ResultMapping 集合
     */
    private List<ResultMapping> resultMappings;
    /**
     * ID ResultMapping 集合
     *
     * 当 idResultMappings 为空时，使用 {@link #resultMappings} 赋值
     */
    private List<ResultMapping> idResultMappings;
    /**
     * 构造方法 ResultMapping 集合
     *
     * 和 {@link #propertyResultMappings} 只有一个值
     */
    private List<ResultMapping> constructorResultMappings;
    /**
     * 属性 ResultMapping 集合
     */
    private List<ResultMapping> propertyResultMappings;
    /**
     * 数据库的字段集合
     */
    private Set<String> mappedColumns;
    /**
     * Java 对象的属性集合
     */
    private Set<String> mappedProperties;
    /**
     * Discriminator 对象
     */
    private Discriminator discriminator;
    /**
     * 是否有内嵌的 ResultMap
     */
    private boolean hasNestedResultMaps;
    /**
     * 是否有内嵌的查询
     */
    private boolean hasNestedQueries;
    /**
     * 是否开启自动匹配
     *
     * 如果设置这个属性，MyBatis将会为这个ResultMap开启或者关闭自动映射。这个属性会覆盖全局的属性 autoMappingBehavior。默认值为：unset。
     */
    private Boolean autoMapping;
}

```

一般都是使用`Builder`来进行构建。

```java
public static class Builder {

        private static final Log log = LogFactory.getLog(Builder.class);

        private ResultMap resultMap = new ResultMap();

        public Builder(Configuration configuration, String id, Class<?> type, List<ResultMapping> resultMappings) {
            this(configuration, id, type, resultMappings, null);
        }

        public Builder(Configuration configuration, String id, Class<?> type, List<ResultMapping> resultMappings, Boolean autoMapping) {
            resultMap.configuration = configuration;
            resultMap.id = id;
            resultMap.type = type;
            resultMap.resultMappings = resultMappings;
            resultMap.autoMapping = autoMapping;
        }

        public Builder discriminator(Discriminator discriminator) {
            resultMap.discriminator = discriminator;
            return this;
        }

        public Class<?> type() {
            return resultMap.type;
        }

        /**
         * 构造 ResultMap 对象
         */
        public ResultMap build() {
            if (resultMap.id == null) {
                throw new IllegalArgumentException("ResultMaps must have an id");
            }
            resultMap.mappedColumns = new HashSet<>();
            resultMap.mappedProperties = new HashSet<>();
            resultMap.idResultMappings = new ArrayList<>();
            resultMap.constructorResultMappings = new ArrayList<>();
            resultMap.propertyResultMappings = new ArrayList<>();
            final List<String> constructorArgNames = new ArrayList<>();
            for (ResultMapping resultMapping : resultMap.resultMappings) {
                // 初始化 hasNestedQueries
                resultMap.hasNestedQueries = resultMap.hasNestedQueries || resultMapping.getNestedQueryId() != null;
                // 初始化 hasNestedResultMaps
                resultMap.hasNestedResultMaps = resultMap.hasNestedResultMaps || (resultMapping.getNestedResultMapId() != null && resultMapping.getResultSet() == null);
                // 添加到 mappedColumns
                final String column = resultMapping.getColumn();
                if (column != null) {
                    resultMap.mappedColumns.add(column.toUpperCase(Locale.ENGLISH));
                } else if (resultMapping.isCompositeResult()) {
                    for (ResultMapping compositeResultMapping : resultMapping.getComposites()) {
                        final String compositeColumn = compositeResultMapping.getColumn();
                        if (compositeColumn != null) {
                            resultMap.mappedColumns.add(compositeColumn.toUpperCase(Locale.ENGLISH));
                        }
                    }
                }
                // 添加到 mappedProperties
                final String property = resultMapping.getProperty();
                if (property != null) {
                    resultMap.mappedProperties.add(property);
                }
                // 初始化 constructorResultMappings
                if (resultMapping.getFlags().contains(ResultFlag.CONSTRUCTOR)) {
                    resultMap.constructorResultMappings.add(resultMapping);
                    if (resultMapping.getProperty() != null) {
                        constructorArgNames.add(resultMapping.getProperty());
                    }
                // 初始化 propertyResultMappings
                } else {
                    resultMap.propertyResultMappings.add(resultMapping);
                }
                // 初始化 idResultMappings
                if (resultMapping.getFlags().contains(ResultFlag.ID)) {
                    resultMap.idResultMappings.add(resultMapping);
                }
            }
            // 保证 idResultMappings 非空
            if (resultMap.idResultMappings.isEmpty()) {
                resultMap.idResultMappings.addAll(resultMap.resultMappings);
            }
            // 将 constructorResultMappings 排序成符合的构造方法的参数顺序
            if (!constructorArgNames.isEmpty()) {
                // 获得真正的构造方法的参数数组 actualArgNames
                final List<String> actualArgNames = argNamesOfMatchingConstructor(constructorArgNames);
                if (actualArgNames == null) {
                    throw new BuilderException("Error in result map '" + resultMap.id
                            + "'. Failed to find a constructor in '"
                            + resultMap.getType().getName() + "' by arg names " + constructorArgNames
                            + ". There might be more info in debug log.");
                }
                // 基于 actualArgNames ，将 constructorResultMappings 排序
                resultMap.constructorResultMappings.sort((o1, o2) -> {
                    int paramIdx1 = actualArgNames.indexOf(o1.getProperty());
                    int paramIdx2 = actualArgNames.indexOf(o2.getProperty());
                    return paramIdx1 - paramIdx2;
                });
            }
            // lock down collections
            resultMap.resultMappings = Collections.unmodifiableList(resultMap.resultMappings);
            resultMap.idResultMappings = Collections.unmodifiableList(resultMap.idResultMappings);
            resultMap.constructorResultMappings = Collections.unmodifiableList(resultMap.constructorResultMappings);
            resultMap.propertyResultMappings = Collections.unmodifiableList(resultMap.propertyResultMappings);
            resultMap.mappedColumns = Collections.unmodifiableSet(resultMap.mappedColumns);
            return resultMap;
        }

        private List<String> argNamesOfMatchingConstructor(List<String> constructorArgNames) {
            // 获得所有构造方法
            Constructor<?>[] constructors = resultMap.type.getDeclaredConstructors();
            // 遍历所有构造方法
            for (Constructor<?> constructor : constructors) {
                Class<?>[] paramTypes = constructor.getParameterTypes();
                // 参数数量一致
                if (constructorArgNames.size() == paramTypes.length) {
                    // 获得构造方法的参数名的数组
                    List<String> paramNames = getArgNames(constructor);
                    if (constructorArgNames.containsAll(paramNames) // 判断名字
                            && argTypesMatch(constructorArgNames, paramTypes, paramNames)) { // 判断类型
                        return paramNames;
                    }
                }
            }
            return null;
        }

        /**
         * 判断构造方法的参数类型是否符合
         *
         * @param constructorArgNames 构造方法的参数名数组
         * @param paramTypes 构造方法的参数类型数组
         * @param paramNames 声明的参数名数组
         * @return 是否符合
         */
        private boolean argTypesMatch(final List<String> constructorArgNames,
                                      Class<?>[] paramTypes, List<String> paramNames) {
            // 遍历所有参数名
            for (int i = 0; i < constructorArgNames.size(); i++) {
                // 判断类型是否匹配
                Class<?> actualType = paramTypes[paramNames.indexOf(constructorArgNames.get(i))];
                Class<?> specifiedType = resultMap.constructorResultMappings.get(i).getJavaType();
                if (!actualType.equals(specifiedType)) {
                    if (log.isDebugEnabled()) {
                        log.debug("While building result map '" + resultMap.id
                                + "', found a constructor with arg names " + constructorArgNames
                                + ", but the type of '" + constructorArgNames.get(i)
                                + "' did not match. Specified: [" + specifiedType.getName() + "] Declared: ["
                                + actualType.getName() + "]");
                    }
                    return false;
                }
            }
            return true;
        }

        /**
         * 获得构造方法的参数名的数组
         *
         * 因为参数上会有 {@link Param} 注解，所以会使用注解上设置的名字
         *
         * @param constructor 构造方法
         * @return 参数名数组
         */
        private List<String> getArgNames(Constructor<?> constructor) {
            // 结果
            List<String> paramNames = new ArrayList<>();
            List<String> actualParamNames = null;
            final Annotation[][] paramAnnotations = constructor.getParameterAnnotations();
            int paramCount = paramAnnotations.length;
            for (int paramIndex = 0; paramIndex < paramCount; paramIndex++) {
                String name = null;
                for (Annotation annotation : paramAnnotations[paramIndex]) {
                    if (annotation instanceof Param) {
                        name = ((Param) annotation).value();
                        break;
                    }
                }
                if (name == null && resultMap.configuration.isUseActualParamName()) {
                    if (actualParamNames == null) {
                        actualParamNames = ParamNameUtil.getParamNames(constructor);
                    }
                    if (actualParamNames.size() > paramIndex) {
                        name = actualParamNames.get(paramIndex);
                    }
                }
                paramNames.add(name != null ? name : "arg" + paramIndex);
            }
            return paramNames;
        }
    }
```



## 4. ???总结

> 书接上文: Mapper映射初始化是我们关注的重点，即`XMLConfigBuilder#mapperElement(root.evalNode("mappers"))`方法

![解析](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/7y6Ttk4EnVWwrGX.png)

> 这些Xml配置元素，Mybatis将它们分别封装成了`ParameterMap`、`ParameterMapping`、`ResultMap`、`ResultMapping`、`MappedStatement`、`BoundSql`等内部数据结构对象。
>
> - `ParameterMap`：`<parameterMap>`节点(即 ResultSet 的映射规则)
> - `ParameterMapping`：`<parameter>` 节点(即属性映射)
> - `ResultMap`：`<resultMap>`节点(即 ResultSet 的映射规则)
> - `ResultMapping`：`<result>` 节点(即属性映射)
> - `MappedStatement`：整个`<select>`节点
> - `BoundSql`：节点标签中的文本,可能包含有其他标签。

这些数据库结构对象，均放置于`Configuration`内部保存起来。

```java
protected final Map<String, MappedStatement> mappedStatements = new StrictMap<MappedStatement>("Mapped Statements collection");
protected final Map<String, ResultMap> resultMaps = new StrictMap<ResultMap>("Result Maps collection");
protected final Map<String, ParameterMap> parameterMaps = new StrictMap<ParameterMap>("Parameter Maps collection");
```

### 4.1 **namespace如何映射Mapper接口**

[`XMLMapperBuilder#bindMapperForNamespace`](#2.4 bindMapperForNamespace)。

直接使用Class.forName()，成功找到就注册，找不到就什么也不做。

Mybatis中的namespace有两个功能。

1. 和其名字含义一样，作为名称空间使用。`namespace + id`，就能找到对应的Sql。

2. 作为Mapper接口的全限名使用，通过namespace，就能找到对应的Mapper接口（也有称Dao接口的）。Mybatis推荐的最佳实践，但并不强制使用。

```java
// Mapper接口注册至`Configuration`的MapperRegistry mapperRegistry内。
private final Map<Class<?>, MapperProxyFactory<?>> knownMappers = new HashMap<Class<?>, MapperProxyFactory<?>>();
```

Mapper接口将通过`MapperProxyFactory`创建动态代理对象 ,参考[动态代理之投鞭断流（自动映射器Mapper的底层实现原理）](http://my.oschina.net/zudajun/blog/666223)。

### 4.2 **MappedStatement引用被缓存两次？**

> ```java
> configuration.addMappedStatement(statement);
> // 调用上面一句话，往Map里放置一个MappedStatement对象，结果Map中变成两个元素。
> 
> //com.mybatis3.mappers.StudentMapper.findAllStudents=org.apache.ibatis.mapping.MappedStatement@add0edd
> //findAllStudents=org.apache.ibatis.mapping.MappedStatement@add0edd
> ```

**为什么会变成两个元素？同一个对象，为什么要存有两个键的引用？**

其实，在Mybatis中，这些Map，都是`StrictMap`类型，Mybatis在StrictMap内做了手脚。

```java
// Configuration.StrictMap
protected static class StrictMap<V> extends HashMap<String, V> {

    public V put(String key, V value) {
      if (containsKey(key)) {
        throw new IllegalArgumentException(name + " already contains value for " + key);
      }
      if (key.contains(".")) {
        final String shortKey = getShortName(key);
        // 不存在shortKey键值，放进去
        if (super.get(shortKey) == null) {
          super.put(shortKey, value);
        } else {
        // 存在shortKey键值，填充占位对象Ambiguity
          super.put(shortKey, (V) new Ambiguity(shortKey));
        }
      }
      return super.put(key, value);
    }
}
// Mybatis重写了put方法，将id和namespace+id的键，都put了进去，指向同一个MappedStatement对象。如果shortKey键值存在，就填充为占位符对象Ambiguity，属于覆盖操作。

// 这样做的好处  方便我们编程 (Mybatis不强制我们一定要加namespace名称空间)
Student std  = sqlSession.selectOne("findStudentById", 1);
Student std  = sqlSession.selectOne("com.mybatis3.mappers.StudentMapper.findStudentById", 1);
```

> 但是**不同namespace空间下的id，必须不同**
>
> ```java
> // Configuration.StrictMap.get()
> public V get(Object key) {
>       V value = super.get(key);
>       if (value == null) {
>         throw new IllegalArgumentException(name + " does not contain value for " + key);
>       }
>       if (value instanceof Ambiguity) {
>         throw new IllegalArgumentException(((Ambiguity) value).getSubject() + " is ambiguous in " + name
>             + " (try using the full name including the namespace, or rename one of the entries)");
>       }
>       return value;
>     }
> 
> // 如果id相同 ，会执行Ambiguity异常。
> ```
>
> 

### 4.3 **初始化过程中的mapped和incomplete对象**

翻译为搞定的和还没搞定的。这恐怕是Mybatis框架中比较奇葩的设计了，给人很多迷惑，我们来看看它具体是什么意思。

```xml
<resultMap type="Student" id="StudentResult" extends="Parent">
	<id property="studId" column="stud_id" />
	<result property="name" column="name" />
	<result property="email" column="email" />
	<result property="dob" column="dob" />
</resultMap>
	
<resultMap type="Student" id="Parent">
	<result property="phone" column="phone" />
</resultMap>
```

Mapper.xml中的很多元素，是可以指定父元素的，像上面`extends="Parent"`。然而，Mybatis解析元素时，是按顺序解析的，于是先解析的`id="StudentResult"`的元素，然而该元素继承自`id="Parent"`的元素，但是，Parent被配置在下面了，还没有解析到，内存中尚不存在，怎么办呢？Mybatis就把`id="StudentResult"`的元素标记为`incomplete`的，然后继续解析后续元素。等程序把`id="Parent"`的元素也解析完后，再回过头来解析`id="StudentResult"`的元素，就可以正确继承父元素的内容。

简言之就是，你的父元素可以配置在你的后边，不限制非得配置在前面。无论你配置在哪儿，Mybatis都能“智能”的获取到，并正确继承。

这便是在`Configuration`对象内，有的叫`mapped`，有的叫`incomplete`的原因。

```java
protected final Collection<XMLStatementBuilder> incompleteStatements = new LinkedList<XMLStatementBuilder>();
protected final Collection<CacheRefResolver> incompleteCacheRefs = new LinkedList<CacheRefResolver>();
protected final Collection<ResultMapResolver> incompleteResultMaps = new LinkedList<ResultMapResolver>();
protected final Collection<MethodResolver> incompleteMethods = new LinkedList<MethodResolver>();
```

```java
// XMLMapperBuilder.parse()内，触发了incomplete的再度解析。  
public void parse() {
    if (!configuration.isResourceLoaded(resource)) {
      configurationElement(parser.evalNode("/mapper"));
      configuration.addLoadedResource(resource);
      bindMapperForNamespace();
    }
    // 执行incomplete的地方  Pending含义为待定的，悬而未决的意思。
    parsePendingResultMaps();
    parsePendingChacheRefs();
    parsePendingStatements();
  }
```

## 666. **彩蛋

😈 又写到早上 1 点 30 左右，哈哈哈哈。

涉及配置解析类的文章，往往写起来枯燥，胖友读起来也枯燥。当然，本文提供的配置示例非常少（自我吐槽），所以胖友先辛苦下，耐心搭配 [《MyBatis 文档 —— Mapper XML 文件》](http://www.mybatis.org/mybatis-3/zh/sqlmap-xml.html) 一起看看。

如果，胖友看到有疑问，一定一定一定要来星球给艿艿提供。这样，艿艿也好针对的优化这个文章。hohoho 。

参考和推荐如下文章：

- 祖大俊 [《Mybatis3.3.x技术内幕（八）：Mybatis初始化流程（上）》](https://my.oschina.net/zudajun/blog/668738)
- 祖大俊 [《Mybatis3.3.x技术内幕（九）：Mybatis初始化流程（中）》](https://my.oschina.net/zudajun/blog/668787)
- 祖大俊 [《Mybatis3.3.x技术内幕（十）：Mybatis初始化流程（下）》](https://my.oschina.net/zudajun/blog/669868)
- 田小波 [《MyBatis 源码分析 - 映射文件解析过程》](https://www.tianxiaobo.com/2018/07/30/MyBatis-源码分析-映射文件解析过程/)
- 徐郡明 [《MyBatis 技术内幕》](https://item.jd.com/12125531.html) 的 [「3.1 MyBatis 初始化」](http://svip.iocoder.cn/MyBatis/builder-package-2/#) 小节