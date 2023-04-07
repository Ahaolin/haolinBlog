# 精尽 MyBatis 源码分析 —— MyBatis 初始化（三）之加载 Statement 配置

## 1. 概述

本文接 [《精尽 MyBatis 源码分析 —— MyBatis 初始化（二）之加载 Mapper 映射配置文件》](#15 初始化（二）之加载 Mapper 映射配置文件) 一文，来分享 MyBatis 初始化的第三步，**加载 Statement 配置**。而这个步骤的入口是 `XMLStatementBuilder `。下面，我们一起来看看它的代码实现。

在 [《精尽 MyBatis 源码分析 —— MyBatis 初始化（二）之加载 Mapper 映射配置文件》](http://svip.iocoder.cn/MyBatis/builder-package-2) 的 [「2.3.5 buildStatementFromContext」](http://svip.iocoder.cn/MyBatis/builder-package-3/#) 中，我们已经看到对 XMLStatementBuilder 的调用代码。代码如下：

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
            statementParser.parseStatementNode(); // 参见 2.2
        } catch (IncompleteElementException e) {
            // <2> 解析失败，添加到 configuration 中
            configuration.addIncompleteStatement(statementParser);
        }
    }
}
```

## 2. XMLStatementBuilder

`org.apache.ibatis.builder.xml.XMLStatementBuilder` ，继承 BaseBuilder 抽象类，Statement XML 配置构建器，主要负责解析 Statement 配置，即 `<select />`、`<insert />`、`<update />`、`<delete />` 标签。

### 2.1 构造方法

```java
// XMLStatementBuilder.java

private final MapperBuilderAssistant builderAssistant;
/**
 * 当前 XML 节点，例如：<select />、<insert />、<update />、<delete /> 标签
 */
private final XNode context;
/**
 * 要求的 databaseId
 */
private final String requiredDatabaseId;

public XMLStatementBuilder(Configuration configuration, MapperBuilderAssistant builderAssistant, XNode context, String databaseId) {
    super(configuration);
    this.builderAssistant = builderAssistant;
    this.context = context;
    this.requiredDatabaseId = databaseId;
}
```

### 2.2 parseStatementNode

`#parseStatementNode()` 方法，执行 Statement 解析。代码如下：

```java
// XMLStatementBuilder.java

public void parseStatementNode() {
    // <1> 获得 id 属性，编号。
    String id = context.getStringAttribute("id");
    // <2> 获得 databaseId ， 判断 databaseId 是否匹配
    String databaseId = context.getStringAttribute("databaseId");
    if (!databaseIdMatchesCurrent(id, databaseId, this.requiredDatabaseId)) {
        return;
    }

    // <3> 获得各种属性
    Integer fetchSize = context.getIntAttribute("fetchSize");
    Integer timeout = context.getIntAttribute("timeout");
    String parameterMap = context.getStringAttribute("parameterMap");
    String parameterType = context.getStringAttribute("parameterType");
    Class<?> parameterTypeClass = resolveClass(parameterType);
    String resultMap = context.getStringAttribute("resultMap");
    String resultType = context.getStringAttribute("resultType");
    String lang = context.getStringAttribute("lang");

    // <4> 获得 lang 对应的 LanguageDriver 对象
    LanguageDriver langDriver = getLanguageDriver(lang);

    // <5> 获得 resultType 对应的类
    Class<?> resultTypeClass = resolveClass(resultType);
    // <6> 获得 resultSet 对应的枚举值
    String resultSetType = context.getStringAttribute("resultSetType");
    ResultSetType resultSetTypeEnum = resolveResultSetType(resultSetType);
    // <7> 获得 statementType 对应的枚举值
    StatementType statementType = StatementType.valueOf(context.getStringAttribute("statementType", StatementType.PREPARED.toString()));

    // <8> 获得 SQL 对应的 SqlCommandType 枚举值
    String nodeName = context.getNode().getNodeName();
    SqlCommandType sqlCommandType = SqlCommandType.valueOf(nodeName.toUpperCase(Locale.ENGLISH));
    // <9> 获得各种属性
    boolean isSelect = sqlCommandType == SqlCommandType.SELECT;
    boolean flushCache = context.getBooleanAttribute("flushCache", !isSelect);
    boolean useCache = context.getBooleanAttribute("useCache", isSelect);
    boolean resultOrdered = context.getBooleanAttribute("resultOrdered", false);

    // Include Fragments before parsing
    // <10> 创建 XMLIncludeTransformer 对象，并替换 <include /> 标签相关的内容
    XMLIncludeTransformer includeParser = new XMLIncludeTransformer(configuration, builderAssistant);
    includeParser.applyIncludes(context.getNode());

    // Parse selectKey after includes and remove them.
    // <11> 解析 <selectKey /> 标签
    processSelectKeyNodes(id, parameterTypeClass, langDriver);

    // Parse the SQL (pre: <selectKey> and <include> were parsed and removed)
    // <12> 创建 SqlSource
    SqlSource sqlSource = langDriver.createSqlSource(configuration, context, parameterTypeClass);
    // <13> 获得 KeyGenerator 对象
    String resultSets = context.getStringAttribute("resultSets");
    String keyProperty = context.getStringAttribute("keyProperty");
    String keyColumn = context.getStringAttribute("keyColumn");
    KeyGenerator keyGenerator;
    // <13.1> 优先，从 configuration 中获得 KeyGenerator 对象。如果存在，意味着是 <selectKey /> 标签配置的
    String keyStatementId = id + SelectKeyGenerator.SELECT_KEY_SUFFIX;
    keyStatementId = builderAssistant.applyCurrentNamespace(keyStatementId, true);
    if (configuration.hasKeyGenerator(keyStatementId)) {
        keyGenerator = configuration.getKeyGenerator(keyStatementId);
    // <13.2> 其次，根据标签属性的情况，判断是否使用对应的 Jdbc3KeyGenerator 或者 NoKeyGenerator 对象
    } else {
        keyGenerator = context.getBooleanAttribute("useGeneratedKeys", // 优先，基于 useGeneratedKeys 属性判断
                configuration.isUseGeneratedKeys() && SqlCommandType.INSERT.equals(sqlCommandType)) // 其次，基于全局的 useGeneratedKeys 配置 + 是否为插入语句类型
                ? Jdbc3KeyGenerator.INSTANCE : NoKeyGenerator.INSTANCE;
    }

    // 创建 MappedStatement 对象
    builderAssistant.addMappedStatement(id, sqlSource, statementType, sqlCommandType,
            fetchSize, timeout, parameterMap, parameterTypeClass, resultMap, resultTypeClass,
            resultSetTypeEnum, flushCache, useCache, resultOrdered,
            keyGenerator, keyProperty, keyColumn, databaseId, langDriver, resultSets);
}
```

- `<1>` 处，获得 `id` 属性，编号。
- <span id='jump2.3_2'>`<2>`</span> 处，获得 `databaseId` 属性，并调用 `#databaseIdMatchesCurrent(String id, String databaseId, String requiredDatabaseId)` 方法，判断 `databaseId` 是否匹配。详细解析，见 [「2.3 databaseIdMatchesCurrent」](#2.3 databaseIdMatchesCurrent) 。
- `<3>` 处，获得各种属性。
- <span id='jump2.3_4'>`<4>`</span> 处，调用 `#getLanguageDriver(String lang)` 方法，获得 `lang` 对应的 LanguageDriver 对象。详细解析，见 [「2.4 getLanguageDriver」](#2.4 getLanguageDriver) 。
- `<5>` 处，获得 `resultType` 对应的类。
- `<6>` 处，获得 `resultSet` 对应的枚举值。关于 `org.apache.ibatis.mapping.ResultSetType` 枚举类，点击[查看](https://github.com/YunaiV/mybatis-3/blob/master/src/main/java/org/apache/ibatis/mapping/ResultSetType.java)。一般情况下，不会设置该值。它是基于 `java.sql.ResultSet` 结果集的几种模式，感兴趣的话，可以看看 [《ResultSet 的 Type 属性》](http://jinguo.iteye.com/blog/365373) 。
- `<7>` 处，获得 `statementType` 对应的枚举值。关于 `org.apache.ibatis.mapping.StatementType` 枚举类，点击[查看](https://github.com/YunaiV/mybatis-3/blob/master/src/main/java/org/apache/ibatis/mapping/StatementType.java)。
- `<8>` 处，获得 SQL 对应的 SqlCommandType 枚举值。
- `<9>` 处，获得各种属性。
- <span id='jump2.3_10'>`<10>`</span> 处，创建 XMLIncludeTransformer 对象，并调用 `XMLIncludeTransformer#applyIncludes(Node source)` 方法，替换 `<include />` 标签相关的内容。详细解析，见 [「3. XMLIncludeTransformer」](#3. XMLIncludeTransformer) 。
- <span id='jump2.3_11'>`<11>`</span> 处，调用 `#processSelectKeyNodes(String id, Class<?> parameterTypeClass, LanguageDriver langDriver)` 方法，解析 `<selectKey />` 标签。详细解析，见 [「2.5 processSelectKeyNodes」](#2.5 processSelectKeyNodes) 。
- `<12>` 处，调用 `LanguageDriver#createSqlSource(Configuration configuration, XNode script, Class<?> parameterType)` 方法，创建 SqlSource 对象。详细解析，见后续文章。
- `<13>` 处，获得 KeyGenerator 对象。分成 `<13.1>` 和 `<13.2>` 两种情况。具体的，胖友耐心看下代码注释。
- <span id='jump2.3_14'>`<14>`</span>  处，调用 `MapperBuilderAssistant#addMappedStatement(...)` 方法，创建 MappedStatement 对象。详细解析，见 [「4.1 addMappedStatement」](#4.1 addMappedStatement) 中。

### 2.3 databaseIdMatchesCurrent

`#databaseIdMatchesCurrent(String id, String databaseId, String requiredDatabaseId)` 方法，判断 `databaseId` 是否匹配。[->](#jump2.3_2)代码如下：

```java
// XMLStatementBuilder.java

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
        // skip this statement if there is a previous one with a not null databaseId
        // 判断是否已经存在
        id = builderAssistant.applyCurrentNamespace(id, false);
        if (this.configuration.hasStatement(id, false)) {
            MappedStatement previous = this.configuration.getMappedStatement(id, false); // issue #2
            // 若存在，则判断原有的 sqlFragment 是否 databaseId 为空。因为，当前 databaseId 为空，这样两者才能匹配。
            return previous.getDatabaseId() == null;
        }
    }
    return true;
}
```

- 代码比较简单，胖友自己瞅瞅就得。从逻辑上，和我们在 XMLMapperBuilder 看到的同名方法 `#databaseIdMatchesCurrent(String id, String databaseId, String requiredDatabaseId)` 方法是一致的。

### 2.4 ？？？getLanguageDriver

`#getLanguageDriver(String lang)` 方法，获得 `lang` 对应的 LanguageDriver 对象。[->](#jump2.3_4)代码如下：

```java
// XMLStatementBuilder.java

private LanguageDriver getLanguageDriver(String lang) {
    // 解析 lang 对应的类
    Class<? extends LanguageDriver> langClass = null;
    if (lang != null) {
        langClass = resolveClass(lang);
    }
    // 获得 LanguageDriver 对象
    return builderAssistant.getLanguageDriver(langClass);
}
```

- 调用 `MapperBuilderAssistant#getLanguageDriver(lass<? extends LanguageDriver> langClass)` 方法，获得 LanguageDriver 对象。代码如下：

  ```java
  // MapperBuilderAssistant.java
  
  public LanguageDriver getLanguageDriver(Class<? extends LanguageDriver> langClass) {
      // 获得 langClass 类
      if (langClass != null) {
          configuration.getLanguageRegistry().register(langClass);
      } else { // 如果为空，则使用默认类
          langClass = configuration.getLanguageRegistry().getDefaultDriverClass();
      }
      // 获得 LanguageDriver 对象
      return configuration.getLanguageRegistry().getDriver(langClass);
  }
  ```

  - 关于 `org.apache.ibatis.scripting.LanguageDriverRegistry` 类，我们在后续的文章，详细解析。

### 2.5 processSelectKeyNodes

`#processSelectKeyNodes(String id, Class<?> parameterTypeClass, LanguageDriver langDriver)` 方法，解析 `<selectKey />` 标签。[->](#jump2.3_11)代码如下：

```java
// XMLStatementBuilder.java

private void processSelectKeyNodes(String id, Class<?> parameterTypeClass, LanguageDriver langDriver) {
    // <1> 获得 <selectKey /> 节点们
    List<XNode> selectKeyNodes = context.evalNodes("selectKey");
    // <2> 执行解析 <selectKey /> 节点们
    if (configuration.getDatabaseId() != null) {
        parseSelectKeyNodes(id, selectKeyNodes, parameterTypeClass, langDriver, configuration.getDatabaseId());
    }
    parseSelectKeyNodes(id, selectKeyNodes, parameterTypeClass, langDriver, null);
    // <3> 移除 <selectKey /> 节点们
    removeSelectKeyNodes(selectKeyNodes);
}
```

- `<1>` 处，获得 `<selectKey />` 节点们。

- <span id='jump2.5_2'>`<2>`</span> 处，调用 `#parseSelectKeyNodes(...)` 方法，执行解析 `<selectKey />` 节点们。详细解析，见 [「2.5.1 parseSelectKeyNodes」](#2.5.1 parseSelectKeyNodes) 。

- `<3>` 处，调用 `#removeSelectKeyNodes(List<XNode> selectKeyNodes)` 方法，移除 `<selectKey />` 节点们。代码如下：

  ```java
  // XMLStatementBuilder.java
  
  private void removeSelectKeyNodes(List<XNode> selectKeyNodes) {
      for (XNode nodeToHandle : selectKeyNodes) {
          nodeToHandle.getParent().getNode().removeChild(nodeToHandle.getNode());
      }
  }
  ```

#### 2.5.1 parseSelectKeyNodes

`#parseSelectKeyNodes(String parentId, List<XNode> list, Class<?> parameterTypeClass, LanguageDriver langDriver, String skRequiredDatabaseId)` 方法，执行解析 `<selectKey />` 子节点们。[->](#jump2.5_2)代码如下：

```java
// XMLStatementBuilder.java

private void parseSelectKeyNodes(String parentId, List<XNode> list, Class<?> parameterTypeClass, LanguageDriver langDriver, String skRequiredDatabaseId) {
    // <1> 遍历 <selectKey /> 节点们
    for (XNode nodeToHandle : list) {
        // <2> 获得完整 id ，格式为 `${id}!selectKey`
        String id = parentId + SelectKeyGenerator.SELECT_KEY_SUFFIX;
        // <3> 获得 databaseId ， 判断 databaseId 是否匹配
        String databaseId = nodeToHandle.getStringAttribute("databaseId");
        if (databaseIdMatchesCurrent(id, databaseId, skRequiredDatabaseId)) {
            // <4> 执行解析单个 <selectKey /> 节点
            parseSelectKeyNode(id, nodeToHandle, parameterTypeClass, langDriver, databaseId);
        }
    }
}
```

- `<1>` 处，遍历 `<selectKey />` 节点们，逐个处理。
- `<2>` 处，获得完整 `id` 编号，格式为 `${id}!selectKey` 。这里很重要，最终解析的 `<selectKey />` 节点，会创建成一个 MappedStatement 对象。而该对象的编号，就是 `id` 。
- `<3>` 处，获得 `databaseId` ，并调用 `#databaseIdMatchesCurrent(String id, String databaseId, String requiredDatabaseId)` 方法，判断 `databaseId` 是否匹配。😈 通过此处，我们可以看到，即使有多个 `<selectionKey />` 节点，但是最终只会有一个节点被解析，就是符合的 `databaseId` 对应的。因为不同的数据库实现不同，对于获取主键的方式也会不同。
- `<4>` 处，调用 `#parseSelectKeyNode(...)` 方法，执行解析**单个** `<selectKey />` 节点。详细解析，见 [「2.5.2 parseSelectKeyNode」](#2.5.2 parseSelectKeyNode) 。

#### 2.5.2 parseSelectKeyNode

`#parseSelectKeyNode(...)` 方法，执行解析**单个** `<selectKey />` 节点。代码如下：

```java
// XMLStatementBuilder.java

private void parseSelectKeyNode(String id, XNode nodeToHandle, Class<?> parameterTypeClass, LanguageDriver langDriver, String databaseId) {
    // <1.1> 获得各种属性和对应的类
    String resultType = nodeToHandle.getStringAttribute("resultType");
    Class<?> resultTypeClass = resolveClass(resultType);
    StatementType statementType = StatementType.valueOf(nodeToHandle.getStringAttribute("statementType", StatementType.PREPARED.toString()));
    String keyProperty = nodeToHandle.getStringAttribute("keyProperty");
    String keyColumn = nodeToHandle.getStringAttribute("keyColumn");
    boolean executeBefore = "BEFORE".equals(nodeToHandle.getStringAttribute("order", "AFTER"));

    // defaults
    // <1.2> 创建 MappedStatement 需要用到的默认值
    boolean useCache = false;
    boolean resultOrdered = false;
    KeyGenerator keyGenerator = NoKeyGenerator.INSTANCE;
    Integer fetchSize = null;
    Integer timeout = null;
    boolean flushCache = false;
    String parameterMap = null;
    String resultMap = null;
    ResultSetType resultSetTypeEnum = null;

    // <1.3> 创建 SqlSource 对象
    SqlSource sqlSource = langDriver.createSqlSource(configuration, nodeToHandle, parameterTypeClass);
    SqlCommandType sqlCommandType = SqlCommandType.SELECT;

    // <1.4> 创建 MappedStatement 对象
    builderAssistant.addMappedStatement(id, sqlSource, statementType, sqlCommandType,
            fetchSize, timeout, parameterMap, parameterTypeClass, resultMap, resultTypeClass,
            resultSetTypeEnum, flushCache, useCache, resultOrdered,
            keyGenerator, keyProperty, keyColumn, databaseId, langDriver, null);

    // <2.1> 获得 SelectKeyGenerator 的编号，格式为 `${namespace}.${id}`
    id = builderAssistant.applyCurrentNamespace(id, false);
    // <2.2> 获得 MappedStatement 对象
    MappedStatement keyStatement = configuration.getMappedStatement(id, false);
    // <2.3> 创建 SelectKeyGenerator 对象，并添加到 configuration 中
    configuration.addKeyGenerator(id, new SelectKeyGenerator(keyStatement, executeBefore));
}
```

- `<1.1>` 处理，获得各种属性和对应的类。

- `<1.2>` 处理，创建 MappedStatement 需要用到的默认值。

- `<1.3>` 处理，调用 `LanguageDriver#createSqlSource(Configuration configuration, XNode script, Class<?> parameterType)` 方法，创建 SqlSource 对象。详细解析，见后续文章。

- <span id='jump2.5.2_1.4'>`<1.4>`</span> 处理，调用 [MapperBuilderAssistant#`addMappedStatement(...)`](##4.1 addMappedStatement) 方法，创建 MappedStatement 对象。

- `<2.1>` 处理，获得 SelectKeyGenerator 的编号，格式为 `${namespace}.${id}` 。

- `<2.2>` 处理，获得 MappedStatement 对象。该对象，实际就是 `<1.4>` 处创建的 MappedStatement 对象。

- `<2.3>` 处理，调用 `Configuration#addKeyGenerator(String id, KeyGenerator keyGenerator)` 方法，创建 SelectKeyGenerator 对象，并添加到 `configuration` 中。代码如下：

  ```java
  // Configuration.java
  
  /**
   * KeyGenerator 的映射
   *
   * KEY：在 {@link #mappedStatements} 的 KEY 的基础上，跟上 {@link SelectKeyGenerator#SELECT_KEY_SUFFIX}
   */
  protected final Map<String, KeyGenerator> keyGenerators = new StrictMap<>("Key Generators collection");
  
  public void addKeyGenerator(String id, KeyGenerator keyGenerator) {
      keyGenerators.put(id, keyGenerator);
  }
  ```

## 3. XMLIncludeTransformer 

`org.apache.ibatis.builder.xml.XMLIncludeTransformer` ，XML `<include />` 标签的转换器，负责将 SQL 中的 `<include />` 标签转换成对应的 `<sql />` 的内容。

### 3.1 构造方法

```java
// XMLIncludeTransformer.java

private final Configuration configuration;
private final MapperBuilderAssistant builderAssistant;

public XMLIncludeTransformer(Configuration configuration, MapperBuilderAssistant builderAssistant) {
    this.configuration = configuration;
    this.builderAssistant = builderAssistant;
}
```

### 3.2 applyIncludes

`#applyIncludes(Node source)` 方法，将 `<include />` 标签，替换成引用的 `<sql />` 。[<-](#jump2.3_10)代码如下：

```java
// XMLIncludeTransformer.java

public void applyIncludes(Node source) {
    // <1> 创建 variablesContext ，并将 configurationVariables 添加到其中
    Properties variablesContext = new Properties();
    Properties configurationVariables = configuration.getVariables();
    if (configurationVariables != null) {
        variablesContext.putAll(configurationVariables);
    }
    // <2> 处理 <include />
    applyIncludes(source, variablesContext, false);
}
```

- `<1>` 处，创建 `variablesContext` ，并将 `configurationVariables` 添加到其中。这里的目的是，避免 `configurationVariables` 被下面使用时候，可能被修改。实际上，从下面的实现上，不存在这个情况。
- `<2>` 处，调用 `#applyIncludes(Node source, final Properties variablesContext, boolean included)` 方法，处理 `<include />` 。

------

`#applyIncludes(Node source, final Properties variablesContext, boolean included)` 方法，使用递归的方式，将 `<include />` 标签，替换成引用的 `<sql />` 。代码如下：

```java
// XMLIncludeTransformer.java

private void applyIncludes(Node source, final Properties variablesContext, boolean included) {
    // <1> 如果是 <include /> 标签
    if (source.getNodeName().equals("include")) {
        // <1.1> 获得 <sql /> 对应的节点
        Node toInclude = findSqlFragment(getStringAttribute(source, "refid"), variablesContext);
        // <1.2> 获得包含 <include /> 标签内的属性
        Properties toIncludeContext = getVariablesContext(source, variablesContext);
        // <1.3> 递归调用 #applyIncludes(...) 方法，继续替换。注意，此处是 <sql /> 对应的节点
        applyIncludes(toInclude, toIncludeContext, true);
        if (toInclude.getOwnerDocument() != source.getOwnerDocument()) { // 这个情况，艿艿暂时没调试出来
            toInclude = source.getOwnerDocument().importNode(toInclude, true);
        }
        // <1.4> 将 <include /> 节点替换成 <sql /> 节点
        source.getParentNode().replaceChild(toInclude, source); // 注意，这是一个奇葩的 API ，前者为 newNode ，后者为 oldNode
        // <1.4> 将 <sql /> 子节点添加到 <sql /> 节点前面
        while (toInclude.hasChildNodes()) {
            toInclude.getParentNode().insertBefore(toInclude.getFirstChild(), toInclude); // 这里有个点，一定要注意，卡了艿艿很久。当子节点添加到其它节点下面后，这个子节点会不见了，相当于是“移动操作”
        }
        // <1.4> 移除 <include /> 标签自身
        toInclude.getParentNode().removeChild(toInclude);
    // <2> 如果节点类型为 Node.ELEMENT_NODE
    } else if (source.getNodeType() == Node.ELEMENT_NODE) {
        // <2.1> 如果在处理 <include /> 标签中，则替换其上的属性，例如 <sql id="123" lang="${cpu}"> 的情况，lang 属性是可以被替换的
        if (included && !variablesContext.isEmpty()) {
            // replace variables in attribute values
            NamedNodeMap attributes = source.getAttributes();
            for (int i = 0; i < attributes.getLength(); i++) {
                Node attr = attributes.item(i);
                attr.setNodeValue(PropertyParser.parse(attr.getNodeValue(), variablesContext));
            }
        }
        // <2.2> 遍历子节点，递归调用 #applyIncludes(...) 方法，继续替换
        NodeList children = source.getChildNodes();
        for (int i = 0; i < children.getLength(); i++) {
            applyIncludes(children.item(i), variablesContext, included);
        }
    // <3> 如果在处理 <include /> 标签中，并且节点类型为 Node.TEXT_NODE ，并且变量非空
    // 则进行变量的替换，并修改原节点 source
    } else if (included && source.getNodeType() == Node.TEXT_NODE
            && !variablesContext.isEmpty()) {
        // replace variables in text node
        source.setNodeValue(PropertyParser.parse(source.getNodeValue(), variablesContext));
    }
}
```

- 这是个有**自递归逻辑**的方法，所以理解起来会有点绕，实际上还是蛮简单的。为了更好的解释，我们假设示例如下：

  ```java
  // mybatis-config.xml
  
  <properties>
      <property name="cpu" value="16c" />
      <property name="target_sql" value="123" />
  </properties>
  
  // Mapper.xml
  
  <sql id="123" lang="${cpu}">
      ${cpu}
      aoteman
      qqqq
  </sql>
  
  <select id="testForInclude">
      SELECT * FROM subject
      <include refid="${target_sql}" />
  </select>
  ```

- 方法参数 `included` ，是否**正在**处理 `<include />` 标签中。😈 一脸懵逼？不要方，继续往下看。

- 在上述示例的`<select />`节点进入这个方法时，会首先进入 `<2>`这块逻辑。

  - `<2.1>` 处，因为 不满足 `included` 条件，初始传入是 `false` ，所以跳过。

  - `<2.2>`处，遍历子节点，**递归调用**`#applyIncludes(...)`方法，继续替换。如图所示：

    ![子节点](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202201201042001.png)

    子节点

    - 子节点 `[0]` 和 `[2]` ，执行该方法时，不满足 `<1>`、`<2>`、`<3>` 任一一种情况，所以可以忽略。虽然说，满足 `<3>` 的节点类型为 `Node.TEXT_NODE` ，但是 `included` 此时为 `false` ，所以不满足。
    - 子节点 `[1]` ，执行该方法时，满足 `<1>` 的情况，所以走起。

- 在子节点[1]，即`<include />`节点进入`<1>`这块逻辑：

  - <span id='jump3.2_1.1'>`<1.1>`</span> 处，调用 `#findSqlFragment(String refid, Properties variables)` 方法，获得 `<sql />` 对应的节点，即上述示例看到的，`<sql id="123" lang="${cpu}"> ... </>` 。详细解析，见 [「3.3 findSqlFragment」](#3.3 findSqlFragment) 。
  - <span id='jump3.2_1.2'>`<1.2>`</span>  处，调用 `#getVariablesContext(Node node, Properties inheritedVariablesContext)` 方法，获得包含 `<include />` 标签内的属性 Properties 对象。详细解析，见 [「3.4 getVariablesContext」](#3.4 getVariablesContext) 。
  - `<1.3>` 处，**递归**调用 `#applyIncludes(...)` 方法，继续替换。注意，此处是 `<sql />` 对应的节点，并且 `included` 参数为 `true` 。详细的结果，见 😈😈😈 处。
  - `<1.4>` 处，将处理好的 `<sql />` 节点，替换掉 `<include />` 节点。逻辑有丢丢绕，胖友耐心看下注释，好好思考。

- 😈😈😈 在`<sql />`节点，会进入`<2>`这块逻辑：

  - `<2.1>` 处，因为 `included` 为 `true` ，所以能满足这块逻辑，会进行执行。如 `<sql id="123" lang="${cpu}">` 的情况，`lang` 属性是可以被替换的。

  - `<2.2>`处，遍历子节点，**递归**调用`#applyIncludes(...)`方法，继续替换。如图所示：

    ![子节点](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202201201042966.png)

    - 子节点 `[0]` ，执行该方法时，满足 `<3>` 的情况，所以可以使用变量 Properteis 对象，进行替换，并修改原节点。

其实，整理一下，逻辑也不会很绕。耐心耐心耐心。

> ```mermaid
> %% 递归  <include /> 标签，替换成引用的 <sql />
> 	%% <select id="testForInclude">
>     %%		SELECT * FROM subject
>     %% 		<include refid="${target_sql}" />
>   	%% </select>
> graph LR
> %% alt XMLIncludeTransformer#applyIncludes方法流程
>    start{开始}--> selectEmpty(< select > 元素)
>     %% XMLIncludeTransformer#applyIncludes方法流程
>     subgraph 情况表
>     	case1
>     	 %% <select>标签第一次调用方法 分成3个子节点
>        	case2==1.ELEMENT_NODE==>sql1>SELECT * FROM XXX] & sql2> < include>] & sql3> \n]
>        	case3==在include中 && TEXT_NODE==>res3{333}
>        	case4==不进行任何操作==>res4{return }
>     end
>    selectEmpty --> case2
>     sql1--2--> case4
>     sql2--2--> case1
>     sql3--2--> case4
>     
>     subgraph include标签正式变成sql过程
>    		case1==存在有include标签==>sqlEmpty{< sql>}
>     	sqlEmpty==3.重新调用==>case2
>     	case2==4.sql标签赋值==> sqlNode ==5.进行sql标签的重新排序==>finish{结束}
>     end
> ```
>
> - 这里的case1、case2、case3、case4 分别对应 `<1>`、`<2>`、`<3>`、最后不执行的语句。
> - 图中的5处对应`<1.4>`

### 3.3 findSqlFragment

`#findSqlFragment(String refid, Properties variables)` 方法，获得对应的 `<sql />` 节点。[->](#jump3.2_1.1)代码如下：

```java
// XMLIncludeTransformer.java

private Node findSqlFragment(String refid, Properties variables) {
    // 因为 refid 可能是动态变量，所以进行替换
    refid = PropertyParser.parse(refid, variables);
    // 获得完整的 refid ，格式为 "${namespace}.${refid}"
    refid = builderAssistant.applyCurrentNamespace(refid, true);
    try {
        // 获得对应的 <sql /> 节点
        XNode nodeToInclude = configuration.getSqlFragments().get(refid);
        // 获得 Node 节点，进行克隆
        return nodeToInclude.getNode().cloneNode(true);
    } catch (IllegalArgumentException e) {
        throw new IncompleteElementException("Could not find SQL statement to include with refid '" + refid + "'", e);
    }
}

private String getStringAttribute(Node node, String name) {
    return node.getAttributes().getNamedItem(name).getNodeValue();
}
```

- 比较简单，胖友瞅瞅注释。

### 3.4 getVariablesContext

`#getVariablesContext(Node node, Properties inheritedVariablesContext)` 方法，获得包含 `<include />` 标签内的属性 Properties 对象。[->](#jump3.2_1.2)代码如下：

```java
// XMLIncludeTransformer.java

private Properties getVariablesContext(Node node, Properties inheritedVariablesContext) {
    // 获得 <include /> 标签的属性集合
    Map<String, String> declaredProperties = null;
    NodeList children = node.getChildNodes();
    for (int i = 0; i < children.getLength(); i++) {
        Node n = children.item(i);
        if (n.getNodeType() == Node.ELEMENT_NODE) {
            String name = getStringAttribute(n, "name");
            // Replace variables inside
            String value = PropertyParser.parse(getStringAttribute(n, "value"), inheritedVariablesContext);
            if (declaredProperties == null) {
                declaredProperties = new HashMap<>();
            }
            if (declaredProperties.put(name, value) != null) { // 如果重复定义，抛出异常
                throw new BuilderException("Variable " + name + " defined twice in the same include definition");
            }
        }
    }
    // 如果 <include /> 标签内没有属性，直接使用 inheritedVariablesContext 即可
    if (declaredProperties == null) {
        return inheritedVariablesContext;
    // 如果 <include /> 标签内有属性，则创建新的 newProperties 集合，将 inheritedVariablesContext + declaredProperties 合并
    } else {
        Properties newProperties = new Properties();
        newProperties.putAll(inheritedVariablesContext);
        newProperties.putAll(declaredProperties);
        return newProperties;
    }
}
```

- 比较简单，胖友瞅瞅注释。

- 如下是 `<include />` 标签内有属性的示例：

  ```java
  <sql id="userColumns"> ${alias}.id,${alias}.username,${alias}.password </sql>
  
  <select id="selectUsers" resultType="map">
    select
      <include refid="userColumns"><property name="alias" value="t1"/></include>,
      <include refid="userColumns"><property name="alias" value="t2"/></include>
    from some_table t1
      cross join some_table t2
  </select>
  ```

## 4. MapperBuilderAssistant

### 4.1 addMappedStatement

构建 MappedStatement 对象, 对于xml配置文件的调用方式，调用如下：[->](#jump2.3_14) [->](#jump2.5.2_1.4)。

```java
// MapperBuilderAssistant.java

/**
     * 通过<select>等 ，构建 MappedStatement 对象
     * @param id   id属性
     * @param sqlSource SQL源对象。存放sql的内容
     * @param statementType statementType属性对应枚举
     * @param sqlCommandType 判断当前 标签 对应的枚举
     * @param fetchSize  fetchSize属性
     * @param timeout   timeout属性属性
     * @param parameterMap parameterMap属性
     * @param parameterType parameterType属性对应类
     * @param resultMap resultMap属性
     * @param resultType    resultType属性对应类
     * @param resultSetType resultSetType对应的枚举
     * @param flushCache   flushCache属性, 默认：查询为false,其他为true。
     * @param useCache  useCache属性, 默认：查询为true,其他为false。
     * @param resultOrdered 默认为false, resultOrdered属性
     * @param keyGenerator
     * @param keyProperty keyProperty属性
     * @param keyColumn keyColumn属性
     * @param databaseId databaseId属性
     * @param lang
     * @param resultSets    resultSets属性
     * @return
     */
public MappedStatement addMappedStatement(
        String id,
        SqlSource sqlSource,
        StatementType statementType,
        SqlCommandType sqlCommandType,
        Integer fetchSize,
        Integer timeout,
        String parameterMap,
        Class<?> parameterType,
        String resultMap,
        Class<?> resultType,
        ResultSetType resultSetType,
        boolean flushCache,
        boolean useCache,
        boolean resultOrdered,
        KeyGenerator keyGenerator,
        String keyProperty,
        String keyColumn,
        String databaseId,
        LanguageDriver lang,
        String resultSets) {

    // <1> 如果只想的 Cache 未解析，抛出 IncompleteElementException 异常
    if (unresolvedCacheRef) {
        throw new IncompleteElementException("Cache-ref not yet resolved");
    }

    // <2> 获得 id 编号，格式为 `${namespace}.${id}`
    id = applyCurrentNamespace(id, false);
    boolean isSelect = sqlCommandType == SqlCommandType.SELECT;

    // <3> 创建 MappedStatement.Builder 对象
    MappedStatement.Builder statementBuilder = new MappedStatement.Builder(configuration, id, sqlSource, sqlCommandType)
            .resource(resource)
            .fetchSize(fetchSize)
            .timeout(timeout)
            .statementType(statementType)
            .keyGenerator(keyGenerator)
            .keyProperty(keyProperty)
            .keyColumn(keyColumn)
            .databaseId(databaseId)
            .lang(lang)
            .resultOrdered(resultOrdered)
            .resultSets(resultSets)
            .resultMaps(getStatementResultMaps(resultMap, resultType, id)) // <3.1> 获得 ResultMap 集合
            .resultSetType(resultSetType)
            .flushCacheRequired(valueOrDefault(flushCache, !isSelect))
            .useCache(valueOrDefault(useCache, isSelect))
            .cache(currentCache);

    // <3.2> 获得 ParameterMap ，并设置到 MappedStatement.Builder 中
    ParameterMap statementParameterMap = getStatementParameterMap(parameterMap, parameterType, id);
    if (statementParameterMap != null) {
        statementBuilder.parameterMap(statementParameterMap);
    }

    // <4> 创建 MappedStatement 对象
    MappedStatement statement = statementBuilder.build();
    // <5> 添加到 configuration 中
    configuration.addMappedStatement(statement);
    return statement;
}
```

- `<1>` 处，如果只想的 Cache 未解析，抛出 IncompleteElementException 异常。

- `<2>` 处，获得 `id` 编号，格式为 `${namespace}.${id}` 。

- `<3>`处，创建 MappedStatement.Builder 对象。详细解析，见[4.1.1 MappedStatement](#4.1.1 MappedStatement)。
  
  - <span id='jump4.1_3.1'>`<3.1>`</span> 处，调用 `#getStatementResultMaps(...)` 方法，获得 ResultMap 集合。详细解析，见 [「4.1.3 getStatementResultMaps」](#4.1.3 getStatementResultMaps) 。
  - `<3.2>` 处，调用 `#getStatementParameterMap(...)` 方法，获得 ParameterMap ，并设置到 MappedStatement.Builder 中。详细解析，见 [4.1.4 getStatementResultMaps」](#4.1.4 getStatementResultMaps) 。
  
- `<4>` 处，创建 MappedStatement 对象。详细解析，见 [「4.1.1 MappedStatement」](http://svip.iocoder.cn/MyBatis/builder-package-3/#) 。

- `<5>` 处，调用 `Configuration#addMappedStatement(statement)` 方法，添加到 `configuration` 中。代码如下：

  ```java
  // Configuration.java
  
  /**
   * MappedStatement 映射
   *
   * KEY：`${namespace}.${id}`
   */
  protected final Map<String, MappedStatement> mappedStatements = new StrictMap<>("Mapped Statements collection");
  
  public void addMappedStatement(MappedStatement ms) {
      mappedStatements.put(ms.getId(), ms);
  }
  ```

#### 4.1.1 MappedStatement

`org.apache.ibatis.mapping.MappedStatement` ，映射的语句，每个 `<select />`、`<insert />`、`<update />`、`<delete />` 对应一个 MappedStatement 对象。代码比较简单，但是有点略长，胖友直接点击 [链接](https://github.com/YunaiV/mybatis-3/blob/master/src/main/java/org/apache/ibatis/mapping/MappedStatement.java) 查看，已经添加了完整的注释。

```java
/**
 * 映射的语句，每个 <select />、<insert />、<update />、<delete /> 对应一个 MappedStatement 对象
 *
 * 另外，比较特殊的是，`<selectKey />` 解析后，也会对应一个 MappedStatement 对象
 *
 * @author Clinton Begin
 */
public final class MappedStatement {

    /**
     * 资源引用的地址
     */
    private String resource;
    /**
     * Configuration 对象
     */
    private Configuration configuration;
    /**
     * 编号
     */
    private String id;
    /**
     * 这是尝试影响驱动程序每次批量返回的结果行数和这个设置值相等。默认值为 unset（依赖驱动）。
     */
    private Integer fetchSize;
    /**
     * 这个设置是在抛出异常之前，驱动程序等待数据库返回请求结果的秒数。默认值为 unset（依赖驱动）。
     */
    private Integer timeout;
    /**
     * 语句类型
     */
    private StatementType statementType;
    /**
     * 结果集类型
     */
    private ResultSetType resultSetType;
    /**
     * SqlSource 对象
     */
    private SqlSource sqlSource;
    /**
     * Cache 对象
     */
    private Cache cache;
    /**
     * ParameterMap 对象
     */
    private ParameterMap parameterMap;
    /**
     * ResultMap 集合
     */
    private List<ResultMap> resultMaps;
    /**
     * 将其设置为 true，任何时候只要语句被调用，都会导致本地缓存和二级缓存都会被清空，默认值：false。
     */
    private boolean flushCacheRequired;
    /**
     * 是否使用缓存
     */
    private boolean useCache;
    /**
     * 这个设置仅针对嵌套结果 select 语句适用：如果为 true，就是假设包含了嵌套结果集或是分组了，这样的话当返回一个主结果行的时候，就不会发生有对前面结果集的引用的情况。这就使得在获取嵌套的结果集的时候不至于导致内存不够用。默认值：false。
     */
    private boolean resultOrdered;
    /**
     * SQL 语句类型
     */
    private SqlCommandType sqlCommandType;
    /**
     * KeyGenerator 对象
     */
    private KeyGenerator keyGenerator;
    /**
     * （仅对 insert 和 update 有用）唯一标记一个属性，MyBatis 会通过 getGeneratedKeys 的返回值或者通过 insert 语句的 selectKey 子元素设置它的键值，默认：unset。如果希望得到多个生成的列，也可以是逗号分隔的属性名称列表。
     */
    private String[] keyProperties;
    /**
     * （仅对 insert 和 update 有用）通过生成的键值设置表中的列名，这个设置仅在某些数据库（像 PostgreSQL）是必须的，当主键列不是表中的第一列的时候需要设置。如果希望得到多个生成的列，也可以是逗号分隔的属性名称列表。
     */
    private String[] keyColumns;
    /**
     * 是否有内嵌的 ResultMap
     */
    private boolean hasNestedResultMaps;
    /**
     * 数据库标识
     */
    private String databaseId;
    /**
     * Log 对象
     */
    private Log statementLog;
    /**
     * LanguageDriver 对象
     */
    private LanguageDriver lang;
    /**
     * 这个设置仅对多结果集的情况适用，它将列出语句执行后返回的结果集并每个结果集给一个名称，名称是逗号分隔的。
     */
    private String[] resultSets;

    MappedStatement() {
        // constructor disabled
    }
```

```java
public static class Builder {

    private MappedStatement mappedStatement = new MappedStatement();

    public Builder(Configuration configuration, String id, SqlSource sqlSource, SqlCommandType sqlCommandType) {
        mappedStatement.configuration = configuration;
        mappedStatement.id = id;
        mappedStatement.sqlSource = sqlSource;
        mappedStatement.statementType = StatementType.PREPARED;
        mappedStatement.resultSetType = ResultSetType.DEFAULT;
        mappedStatement.parameterMap = new ParameterMap.Builder(configuration, "defaultParameterMap", null, new ArrayList<>()).build();
        mappedStatement.resultMaps = new ArrayList<>();
        mappedStatement.sqlCommandType = sqlCommandType;
        mappedStatement.keyGenerator = configuration.isUseGeneratedKeys() && SqlCommandType.INSERT.equals(sqlCommandType) ? Jdbc3KeyGenerator.INSTANCE : NoKeyGenerator.INSTANCE;
        // 获得 Log 对象
        String logId = id;
        if (configuration.getLogPrefix() != null) {
            logId = configuration.getLogPrefix() + id;
        }
        mappedStatement.statementLog = LogFactory.getLog(logId);
        mappedStatement.lang = configuration.getDefaultScriptingLanguageInstance();
    }

    public Builder resultMaps(List<ResultMap> resultMaps) {
        mappedStatement.resultMaps = resultMaps;
        for (ResultMap resultMap : resultMaps) {
            mappedStatement.hasNestedResultMaps = mappedStatement.hasNestedResultMaps || resultMap.hasNestedResultMaps();
        }
        return this;
    }

    public Builder resultSets(String resultSet) {
        mappedStatement.resultSets = delimitedStringToArray(resultSet);
        return this;
    }

    /** @deprecated Use {@link #resultSets} */
    @Deprecated
    public Builder resulSets(String resultSet) {
        mappedStatement.resultSets = delimitedStringToArray(resultSet);
        return this;
    }

    public MappedStatement build() {
        assert mappedStatement.configuration != null;
        assert mappedStatement.id != null;
        assert mappedStatement.sqlSource != null;
        assert mappedStatement.lang != null;
        mappedStatement.resultMaps = Collections.unmodifiableList(mappedStatement.resultMaps);
        return mappedStatement;
    }
}
```



另外，比较特殊的是，`<selectKey />` 解析后，也会对应一个 MappedStatement 对象。

在另外，MappedStatement 有一个非常重要的方法 `#getBoundSql(Object parameterObject)` 方法，获得 BoundSql 对象。代码如下：

```java
// MappedStatement.java

public BoundSql getBoundSql(Object parameterObject) {
    // 获得 BoundSql 对象
    BoundSql boundSql = sqlSource.getBoundSql(parameterObject);
    // 忽略，因为 <parameterMap /> 已经废弃，参见 http://www.mybatis.org/mybatis-3/zh/sqlmap-xml.html 文档
    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    if (parameterMappings == null || parameterMappings.isEmpty()) {
        boundSql = new BoundSql(configuration, boundSql.getSql(), parameterMap.getParameterMappings(), parameterObject);
    }

    // check for nested result maps in parameter mappings (issue #30)
    // 判断传入的参数中，是否有内嵌的结果 ResultMap 。如果有，则修改 hasNestedResultMaps 为 true
    // 存储过程相关，暂时无视
    for (ParameterMapping pm : boundSql.getParameterMappings()) {
        String rmId = pm.getResultMapId();
        if (rmId != null) {
            ResultMap rm = configuration.getResultMap(rmId);
            if (rm != null) {
                hasNestedResultMaps |= rm.hasNestedResultMaps();
            }
        }
    }

    return boundSql;
}
```

- 需要结合后续文章 [《精尽 MyBatis 源码分析 —— SQL 初始化（下）之 SqlSource》](http://svip.iocoder.cn/MyBatis/scripting-2) 。胖友可以看完那篇文章后，再回过头看这个方法。

#### 4.1.2 ParameterMap

`org.apache.ibatis.mapping.ParameterMap` ，参数集合，对应 `paramType=""` 或 `paramMap=""` 标签属性。代码比较简单，但是有点略长，胖友直接点击 [链接](https://github.com/YunaiV/mybatis-3/blob/master/src/main/java/org/apache/ibatis/mapping/ParameterMap.java) 查看，已经添加了完整的注释。

#### 4.1.3 getStatementResultMaps

`#getStatementResultMaps(...)` 方法，获得 ResultMap 集合。[->](#jump4.1_3.1)代码如下：

```java
// MapperBuilderAssistant.java

private List<ResultMap> getStatementResultMaps(
        String resultMap,
        Class<?> resultType,
        String statementId) {
    // 获得 resultMap 的编号
    resultMap = applyCurrentNamespace(resultMap, true);

    // 创建 ResultMap 集合
    List<ResultMap> resultMaps = new ArrayList<>();
    // 如果 resultMap 非空，则获得 resultMap 对应的 ResultMap 对象(们）
    if (resultMap != null) {
        String[] resultMapNames = resultMap.split(",");
        for (String resultMapName : resultMapNames) {
            try {
                resultMaps.add(configuration.getResultMap(resultMapName.trim())); // 从 configuration 中获得
            } catch (IllegalArgumentException e) {
                throw new IncompleteElementException("Could not find result map " + resultMapName, e);
            }
        }
    // 如果 resultType 非空，则创建 ResultMap 对象
    } else if (resultType != null) {
        ResultMap inlineResultMap = new ResultMap.Builder(
                configuration,
                statementId + "-Inline",
                resultType,
                new ArrayList<>(),
                null).build();
        resultMaps.add(inlineResultMap);
    }
    return resultMaps;
}
```

- 整体代码比较简单，胖友自己看下。
- 比较奇怪的是，方法参数 `resultMap` 存在使用逗号分隔的情况。这个出现在使用存储过程的时候，参见 [《mybatis调用存储过程返回多个结果集》](https://blog.csdn.net/sinat_25295611/article/details/75103358) 。

#### 4.1.4 getStatementResultMaps

`#getStatementParameterMap(...)` 方法，获得 ParameterMap 对象。代码如下：

```java
// MapperBuilderAssistant.java

private ParameterMap getStatementParameterMap(
        String parameterMapName,
        Class<?> parameterTypeClass,
        String statementId) {
    // 获得 ParameterMap 的编号，格式为 `${namespace}.${parameterMapName}`
    parameterMapName = applyCurrentNamespace(parameterMapName, true);
    ParameterMap parameterMap = null;
    // <2> 如果 parameterMapName 非空，则获得 parameterMapName 对应的 ParameterMap 对象
    if (parameterMapName != null) {
        try {
            parameterMap = configuration.getParameterMap(parameterMapName);
        } catch (IllegalArgumentException e) {
            throw new IncompleteElementException("Could not find parameter map " + parameterMapName, e);
        }
    // <1> 如果 parameterTypeClass 非空，则创建 ParameterMap 对象
    } else if (parameterTypeClass != null) {
        List<ParameterMapping> parameterMappings = new ArrayList<>();
        parameterMap = new ParameterMap.Builder(
                configuration,
                statementId + "-Inline",
                parameterTypeClass,
                parameterMappings).build();
    }
    return parameterMap;
}
```

- 主要看 `<1>` 处，如果 `parameterTypeClass` 非空，则创建 ParameterMap 对象。
- 关于 `<2>` 处，MyBatis 官方不建议使用 `parameterMap` 的方式。

## 5. 总结

> Mybatis初始化过程中，解析`parameterMap`、`resultMap`、`"select|insert|update|delete"`元素，无疑是重头戏。
>
> `<parameterMap>`将会解析为`ParameterMap`对象，该对象包含一个`List<ParameterMapping>`集合。
>
> `<resultMap>`将会解析为`ResultMap`对象，该对象包含一个`List<ResultMapping>`集合。
>
> `<"select|insert|update|delete">`将会被解析为`MappedStatement`对象，该对象包含了`ParameterMap`、`ResultMap`等对象。

### 5.1 **解析parameterMap元素**

![img](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202202041159263.jpeg)

Mybatis解析parameterMap元素的源码。

```java
// XMLMapperBuilder#parameterMapElement
// FROM 《MyBatis 官方文档 —— Mapper XML 文件》http://www.mybatis.org/mybatis-3/zh/sqlmap-xml.html
private void parameterMapElement(List<XNode> list) throws Exception {
    for (XNode parameterMapNode : list) {
      String id = parameterMapNode.getStringAttribute("id");
      String type = parameterMapNode.getStringAttribute("type");
      Class<?> parameterClass = resolveClass(type);
      List<XNode> parameterNodes = parameterMapNode.evalNodes("parameter");
      List<ParameterMapping> parameterMappings = new ArrayList<ParameterMapping>();
      // 循环获得所有的ParameterMapping集合
      for (XNode parameterNode : parameterNodes) {
        String property = parameterNode.getStringAttribute("property");
        String javaType = parameterNode.getStringAttribute("javaType");
        String jdbcType = parameterNode.getStringAttribute("jdbcType");
        String resultMap = parameterNode.getStringAttribute("resultMap");
        String mode = parameterNode.getStringAttribute("mode");
        String typeHandler = parameterNode.getStringAttribute("typeHandler");
        Integer numericScale = parameterNode.getIntAttribute("numericScale");
        ParameterMode modeEnum = resolveParameterMode(mode);
        Class<?> javaTypeClass = resolveClass(javaType);
        JdbcType jdbcTypeEnum = resolveJdbcType(jdbcType);
        @SuppressWarnings("unchecked")
        Class<? extends TypeHandler<?>> typeHandlerClass = (Class<? extends TypeHandler<?>>) resolveClass(typeHandler);
         // <1> 解析ParameterMapping 并添加到集合中
        ParameterMapping parameterMapping = builderAssistant.buildParameterMapping(parameterClass, property, javaTypeClass, jdbcTypeEnum, resultMap, modeEnum, typeHandlerClass, numericScale);
        parameterMappings.add(parameterMapping);
      }
      // <2> 创建ParameterMap并加入List<ParameterMapping>，同时把ParameterMap注册到Configuration内。
      builderAssistant.addParameterMap(id, parameterClass, parameterMappings);
    }
  }
```

`<1>`处方法源码，见[[5.1.1 构建ParameterMapping]](#5.1.1 buildParameterMapping)。

`<2>`处方法源码，见[[5.1.2 构建ParameterMap]](#5.1.2 addParameterMap)。

#### 5.1.1 buildParameterMapping

```java
// MapperBuilderAssistant#buildParameterMapping
public ParameterMapping buildParameterMapping(
      Class<?> parameterType,
      String property,
      Class<?> javaType,
      JdbcType jdbcType,
      String resultMap,
      ParameterMode parameterMode,
      Class<? extends TypeHandler<?>> typeHandler,
      Integer numericScale) {
    resultMap = applyCurrentNamespace(resultMap, true);

    Class<?> javaTypeClass = resolveParameterJavaType(parameterType, property, javaType, jdbcType);
    TypeHandler<?> typeHandlerInstance = resolveTypeHandler(javaTypeClass, typeHandler);
    // 下面的一系列方法链，其实都是赋值语句 
    return new ParameterMapping.Builder(configuration, property, javaTypeClass)
        .jdbcType(jdbcType)
        .resultMapId(resultMap)
        .mode(parameterMode)
        .numericScale(numericScale)
        .typeHandler(typeHandlerInstance)
        .build(); // 内部将调用resolveTypeHandler()方法  见下面↓
  }


// ParameterMapping#build
  public ParameterMapping build() {
      // 给每一个ParameterMapping绑定一个TypeHandler，且必须绑定
      resolveTypeHandler(); // 相当于图上的红框
      validate();
      return parameterMapping;
    }
```

一个`ParameterMapping`，其实就是一个参数属性的封装，从`jdbcType`到`javaType`的转换，或者从`javaType`到`jdbcType`转换，全由`TypeHandler`处理。`ParameterMapping`就解析结束了。

#### 5.1.2 addParameterMap

最后，看看`ParameterMap`是如何创建并注册的。

```java
// MapperBuilderAssistant#addParameterMap
public ParameterMap addParameterMap(String id, Class<?> parameterClass, List<ParameterMapping> parameterMappings) {
    // 处理namespace名称空间
    id = applyCurrentNamespace(id, false);
    ParameterMap parameterMap = new ParameterMap.Builder(configuration, id, parameterClass, parameterMappings).build();
    // 注册至Configuration
    configuration.addParameterMap(parameterMap);
    return parameterMap;
  }

// Configuration.java
public void addParameterMap(ParameterMap pm) {
    // 放到map中
    parameterMaps.put(pm.getId(), pm);
  }
```

### 5.2 **解析ResultMap元素**

![img](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202202041239338.jpeg)

解析`ResultMap`元素和解析`parameterMap`元素是极其相似的，有区别的地方，主要是ResultMap有继承（extends）的功能，以及ResultMap会将`List<ResultMapping>`再进行一次计算，拆分为多个`List<ResultMapping>`对象，也就是大集合，分类拆分为多个小集合。

```java
public class ResultMap {
  // ...
  private List<ResultMapping> resultMappings;
  private List<ResultMapping> idResultMappings;
  private List<ResultMapping> constructorResultMappings; 
  private List<ResultMapping> propertyResultMappings;
  // ...
}
```

```java

// MapperBuilderAssistant#addResultMap()
public ResultMap addResultMap(
      String id,
      Class<?> type,
      String extend,
      Discriminator discriminator,
      List<ResultMapping> resultMappings,
      Boolean autoMapping) {
    id = applyCurrentNamespace(id, false);
    extend = applyCurrentNamespace(extend, true);

    if (extend != null) {
      if (!configuration.hasResultMap(extend)) {
        throw new IncompleteElementException("Could not find a parent resultmap with id '" + extend + "'");
      }
      // 处理继承ResultMap属性
      ResultMap resultMap = configuration.getResultMap(extend);
      List<ResultMapping> extendedResultMappings = new ArrayList<ResultMapping>(resultMap.getResultMappings());
      // 删除重复元素
      extendedResultMappings.removeAll(resultMappings);
      // Remove parent constructor if this resultMap declares a constructor.
      boolean declaresConstructor = false;
      for (ResultMapping resultMapping : resultMappings) {
        if (resultMapping.getFlags().contains(ResultFlag.CONSTRUCTOR)) {
          declaresConstructor = true;
          break;
        }
      }
      if (declaresConstructor) {
        Iterator<ResultMapping> extendedResultMappingsIter = extendedResultMappings.iterator();
        while (extendedResultMappingsIter.hasNext()) {
          if (extendedResultMappingsIter.next().getFlags().contains(ResultFlag.CONSTRUCTOR)) {
            extendedResultMappingsIter.remove();
          }
        }
      }
      // 合并
      resultMappings.addAll(extendedResultMappings);
    }
    ResultMap resultMap = new ResultMap.Builder(configuration, id, type, resultMappings, autoMapping)
        .discriminator(discriminator)
        .build(); // build()内将大集合，分类拆分为多个小集合。
    // 注册到Configuration内
    configuration.addResultMap(resultMap);
    return resultMap;
  }
```

至此，一个`ResultMap`就解析完了。且每一个`ResultMapping`，都绑定了一个`TypeHandler`，和`ParameterMapping`一样。



### 5.3 ????**解析"select|insert|update|delete"元素**

![img](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202202041715802.jpeg)

[参考`2.2 parseStatementNode`](#2.2 parseStatementNode) 。全程平面式的解析，最后生成`MappedStatement`对象，并注册至`Configuration`内部。

解析过程中，出现的一些陌生的配置参数或类，如`KeyGenerator`、`SqlSource`、`ResultSetType`、`LanguageDriver`、`constructor`、`discriminator`等等，后续会逐一进行详细的分析。

### 5.4 可复用的sqlFragment

在工作中，往往有这样的需求，对于同一个sql条件查询，**首先需要统计记录条数**，用以计算`pageCount`，然后再对结果进行**分页查询显示**，看下面一个例子。

```xml
<sql id="studentProperties"><!--sql片段-->
		select  stud_id as studId , name, email , dob, phonefrom students
</sql>
	
<select id="countAll" resultType="int">
	select count(1) from (<include refid="studentProperties" />) tmp
</select>
	
<select id="findAll" resultType="Student" parameterType="map">
	select * from (<include refid="studentProperties"/>) tmp limit #{offset}, #{pagesize}
</select>
```

这就是`sqlFragment`，它可以为`select|insert|update|delete标签`服务，可以定义很多sqlFragment，然后使用`include`标签引入多个`sqlFragment`。在工作中，也是比较常用的一个功能，它的优点很明显，复用sql片段，它的缺点也很明显，不能完整的展现sql逻辑，如果一个标签，include了四至五个sqlFragment，其可读性就非常差了。

#### 5.4.1 sqlFragment的解析过程

`sqlFragment`存储于`Configuration`内部。

```java
protected final Map<String, XNode> sqlFragments = new StrictMap<XNode>("XML fragments parsed from previous mappers");
```

解析`sqlFragment`的过程非常简单。

```java
// XMLMapperBuilder#configurationElement
//  解析sqlFragment
sqlElement(context.evalNodes("/mapper/sql"));
// 为select|insert|update|delete提供服务  
// [该方法内部会调用  XMLStatementBuilder#parseStatementNode]
buildStatementFromContext(context.evalNodes("select|insert|update|delete"));

// sqlFragment存储于Map<String, XNode>结构当中。
```

#### 5.4.2 解析include标签的过程

[XMLStatementBuilder#parseStatementNode](#2.2 parseStatementNode)

```java
// XMLStatementBuilder#parseStatementNode
// Include Fragments before parsing
// 创建 XMLIncludeTransformer 对象，并替换 <include /> 标签相关的内容
XMLIncludeTransformer includeParser = new XMLIncludeTransformer(configuration, builderAssistant);
// 重点关注的方法 
includeParser.applyIncludes(context.getNode());

// <1>  Parse selectKey after includes and remove them.   解析 <selectKey /> 标签
processSelectKeyNodes(id, parameterTypeClass, langDriver);
    
// Parse the SQL (pre: <selectKey> and <include> were parsed and removed) 创建 SqlSource 对象
SqlSource sqlSource = langDriver.createSqlSource(configuration, context, parameterTypeClass);
```

`<1>` **解析 `<selectKey />` 标签**。注释含义为解析完，并移除。为什么要移除呢？秘密都隐藏在[`applyIncludes`()方法](#3.2 applyIncludes)内部了。

```java
/**
   * Recursively apply includes through all SQL fragments.
   * @param source Include node in DOM tree
   * @param variablesContext Current context for static variables with values
   */
  private void applyIncludes(Node source, final Properties variablesContext) {
    if (source.getNodeName().equals("include")) {
      // new full context for included SQL - contains inherited context and new variables from current include node
      Properties fullContext;

      String refid = getStringAttribute(source, "refid");
      // replace variables in include refid value
      refid = PropertyParser.parse(refid, variablesContext);
      Node toInclude = findSqlFragment(refid);
      Properties newVariablesContext = getVariablesContext(source, variablesContext);
      if (!newVariablesContext.isEmpty()) {
        // merge contexts
        fullContext = new Properties();
        fullContext.putAll(variablesContext);
        fullContext.putAll(newVariablesContext);
      } else {
        // no new context - use inherited fully
        fullContext = variablesContext;
      }
      // 递归调用
      applyIncludes(toInclude, fullContext);
      if (toInclude.getOwnerDocument() != source.getOwnerDocument()) {
        toInclude = source.getOwnerDocument().importNode(toInclude, true);
      }
      // 将include节点，替换为sqlFragment节点
      source.getParentNode().replaceChild(toInclude, source);
      while (toInclude.hasChildNodes()) {
        // 将sqlFragment的子节点（也就是文本节点），插入到sqlFragment的前面
        toInclude.getParentNode().insertBefore(toInclude.getFirstChild(), toInclude);
      }
      // 移除sqlFragment节点
      toInclude.getParentNode().removeChild(toInclude);
    } else if (source.getNodeType() == Node.ELEMENT_NODE) {
      NodeList children = source.getChildNodes();
      for (int i=0; i<children.getLength(); i++) {
        // 递归调用
        applyIncludes(children.item(i), variablesContext);
      }
    } else if (source.getNodeType() == Node.ATTRIBUTE_NODE && !variablesContext.isEmpty()) {
      // replace variables in all attribute values
      source.setNodeValue(PropertyParser.parse(source.getNodeValue(), variablesContext));
    } else if (source.getNodeType() == Node.TEXT_NODE && !variablesContext.isEmpty()) {
      // replace variables ins all text nodes
      source.setNodeValue(PropertyParser.parse(source.getNodeValue(), variablesContext));
    }
  }
```

上面是对源码的解读，为了便于理解，我们接下来采用图示的办法，演示其过程。

##### 5.4.2.1 图示过程演示

1. 解析节点

   1. <select id="countAll" resultType="int">
      	select count(1) from (
      		<include refid="studentProperties"></include>
      	) tmp
      </select>

2. `include`节点替换为`sqlFragment`节点

   1. <select id="countAll" resultType="int">
      	select count(1) from (
      		<sql id="studentProperties">
      			select stud_id as studId, name, email, dob, phonefrom students
      		</sql>
      	) tmp
      </select>

3. 将`sqlFragment`的子节点（文本节点）insert到`sqlFragment`节点的前面。注意，对于dom来说，文本也是一个节点，叫`TextNode`。

   1. <select id="countAll" resultType="int">
      	select count(1) from (
      		select stud_id as studId, name, email, dob, phonefrom students
      			<sql id="studentProperties">
      				select stud_id as studId, name, email, dob, phonefrom students
      			</sql>
      	) tmp
      </select>

4. 移除`sqlFragment`节点

   1. <select id="countAll" resultType="int">
      	select count(1) from (
      		select stud_id as studId, name, email, dob, phonefrom students
      	) tmp
      </select>

5. 最后结果如图所示

   1. <img src="http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202202050926637.jpeg" alt="img" style="zoom:50%;" />

如此一来，TextNode1 + TextNode2 + TextNode3，就组成了一个完整的sql。遍历select的三个子节点，分别取出TextNode的value，append到一起，就是最终完整的sql。

这也是为什么要移除`<selectKey> `and` <include>`节点的原因。

这就是Mybatis的sqlFragment，以上示例，均为静态sql，即`static sql`，有关动态sql，即`dynamic sql`，将在后续博文中进行仔细分析。

## 666. *彩蛋

相比 [《精尽 MyBatis 源码分析 —— MyBatis 初始化（二）之加载 Mapper 映射配置文件》](http://svip.iocoder.cn/MyBatis/builder-package-2) 来说，简单太多太多，可能就 XMLIncludeTransformer 相对绕一丢丢。总的来说，轻松蛮多。

- 祖大俊 [《Mybatis3.3.x技术内幕（十）：Mybatis初始化流程（下）》](https://my.oschina.net/zudajun/blog/669868)
- 祖大俊 [《Mybatis3.4.x技术内幕（十六）：Mybatis之sqlFragment（可复用的sql片段）》](https://my.oschina.net/zudajun/blog/687326)
- 徐郡明 [《MyBatis 技术内幕》](https://item.jd.com/12125531.html) 的 [「3.1 MyBatis 初始化」](http://svip.iocoder.cn/MyBatis/builder-package-3/#) 小节



