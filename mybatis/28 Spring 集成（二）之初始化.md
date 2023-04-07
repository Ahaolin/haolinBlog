# 精尽 MyBatis 源码解析 —— Spring 集成（二）之初始化

## 1. 概述

在前面的，我们已经看了四篇 MyBatis 初始化相关的文章：

- [《MyBatis 初始化（一）之加载 mybatis-config》](http://svip.iocoder.cn/MyBatis/builder-package-1)
- [《MyBatis 初始化（二）之加载 Mapper 映射配置文件》](http://svip.iocoder.cn/MyBatis/builder-package-2)
- [《MyBatis 初始化（三）之加载 Statement 配置》](http://svip.iocoder.cn/MyBatis/builder-package-3)
- [《MyBatis 初始化（四）之加载注解配置》](http://svip.iocoder.cn/MyBatis/builder-package-4)

那么，本文我们就来看看，Spring 和 MyBatis 如何集成。主要涉及如下三个包：

- `annotation`
- `config`
- `mapper`

## 2. SqlSessionFactoryBean

`org.mybatis.spring.SqlSessionFactoryBean` ，实现 FactoryBean、InitializingBean、ApplicationListener 接口，负责创建 SqlSessionFactory 对象。

使用示例如下：

```xml
<!-- simplest possible SqlSessionFactory configuration -->
<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
    <property name="dataSource" ref="dataSource" />
    <!-- Directly specify the location of the MyBatis mapper xml file. This
         is NOT required when using MapperScannerConfigurer or
         MapperFactoryBean; they will load the xml automatically if it is
         in the same classpath location as the DAO interface. Rather than
         directly referencing the xml files, the 'configLocation' property
         could also be used to specify the location of a MyBatis config
         file. This config file could, in turn, contain &ltmapper&gt
         elements that point to the correct mapper xml files.
     -->
    <property name="mapperLocations" value="classpath:org/mybatis/spring/sample/mapper/*.xml" />
</bean>
```

另外，如果胖友不熟悉 Spring FactoryBean 的机制。可以看看 [《Spring bean 之 FactoryBean》](https://www.jianshu.com/p/d6c42d723464) 文章。

### 2.1 构造方法

```java
// SqlSessionFactoryBean.java

private static final Logger LOGGER = LoggerFactory.getLogger(SqlSessionFactoryBean.class);

/**
 * 指定 mybatis-config.xml 路径的 Resource 对象
 */
private Resource configLocation;

private Configuration configuration;

/**
 * 指定 Mapper 路径的 Resource 数组
 */
private Resource[] mapperLocations;

private DataSource dataSource;

private TransactionFactory transactionFactory;

private Properties configurationProperties;

private SqlSessionFactoryBuilder sqlSessionFactoryBuilder = new SqlSessionFactoryBuilder();

private SqlSessionFactory sqlSessionFactory;

//EnvironmentAware requires spring 3.1
private String environment = SqlSessionFactoryBean.class.getSimpleName();

private boolean failFast;

private Interceptor[] plugins;

private TypeHandler<?>[] typeHandlers;

private String typeHandlersPackage;

private Class<?>[] typeAliases;

private String typeAliasesPackage;

private Class<?> typeAliasesSuperType;

//issue #19. No default provider.
private DatabaseIdProvider databaseIdProvider;

private Class<? extends VFS> vfs;

private Cache cache;

private ObjectFactory objectFactory;

private ObjectWrapperFactory objectWrapperFactory;

// 省略 setting 方法
```

- 是不是各种熟悉的属性，这里就不多解释每个对象了。你比我懂，嘿嘿。

### 2.2 afterPropertiesSet

`#afterPropertiesSet()` 方法，构建 SqlSessionFactory 对象。代码如下：

```java
// SqlSessionFactoryBean.java

@Override
public void afterPropertiesSet() throws Exception {
    notNull(dataSource, "Property 'dataSource' is required");
    notNull(sqlSessionFactoryBuilder, "Property 'sqlSessionFactoryBuilder' is required");
    state((configuration == null && configLocation == null) || !(configuration != null && configLocation != null),
            "Property 'configuration' and 'configLocation' can not specified with together");

    // 创建 SqlSessionFactory 对象
    this.sqlSessionFactory = buildSqlSessionFactory();
}

/**
 * Build a {@code SqlSessionFactory} instance.
 *
 * The default implementation uses the standard MyBatis {@code XMLConfigBuilder} API to build a
 * {@code SqlSessionFactory} instance based on an Reader.
 * Since 1.3.0, it can be specified a {@link Configuration} instance directly(without config file).
 *
 * @return SqlSessionFactory
 * @throws IOException if loading the config file failed
 */
protected SqlSessionFactory buildSqlSessionFactory() throws IOException {
    Configuration configuration;

    // 初始化 configuration 对象，和设置其 `configuration.variables` 属性
    XMLConfigBuilder xmlConfigBuilder = null;
    if (this.configuration != null) {
        configuration = this.configuration;
        if (configuration.getVariables() == null) {
            configuration.setVariables(this.configurationProperties);
        } else if (this.configurationProperties != null) {
            configuration.getVariables().putAll(this.configurationProperties);
        }
    } else if (this.configLocation != null) {
        xmlConfigBuilder = new XMLConfigBuilder(this.configLocation.getInputStream(), null, this.configurationProperties);
        configuration = xmlConfigBuilder.getConfiguration();
    } else {
        LOGGER.debug(() -> "Property 'configuration' or 'configLocation' not specified, using default MyBatis Configuration");
        configuration = new Configuration();
        if (this.configurationProperties != null) {
            configuration.setVariables(this.configurationProperties);
        }
    }

    // 设置 `configuration.objectFactory` 属性
    if (this.objectFactory != null) {
        configuration.setObjectFactory(this.objectFactory);
    }

    // 设置 `configuration.objectWrapperFactory` 属性
    if (this.objectWrapperFactory != null) {
        configuration.setObjectWrapperFactory(this.objectWrapperFactory);
    }

    // 设置 `configuration.vfs` 属性
    if (this.vfs != null) {
        configuration.setVfsImpl(this.vfs);
    }

    // 设置 `configuration.typeAliasesPackage` 属性
    if (hasLength(this.typeAliasesPackage)) {
        String[] typeAliasPackageArray = tokenizeToStringArray(this.typeAliasesPackage,
                ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS);
        for (String packageToScan : typeAliasPackageArray) {
            configuration.getTypeAliasRegistry().registerAliases(packageToScan,
                    typeAliasesSuperType == null ? Object.class : typeAliasesSuperType);
            LOGGER.debug(() -> "Scanned package: '" + packageToScan + "' for aliases");
        }
    }

    // 设置 `configuration.typeAliases` 属性
    if (!isEmpty(this.typeAliases)) {
        for (Class<?> typeAlias : this.typeAliases) {
            configuration.getTypeAliasRegistry().registerAlias(typeAlias);
            LOGGER.debug(() -> "Registered type alias: '" + typeAlias + "'");
        }
    }

    // 初始化 `configuration.interceptorChain` 属性，即拦截器
    if (!isEmpty(this.plugins)) {
        for (Interceptor plugin : this.plugins) {
            configuration.addInterceptor(plugin);
            LOGGER.debug(() -> "Registered plugin: '" + plugin + "'");
        }
    }

    // 扫描 typeHandlersPackage 包，注册 TypeHandler
    if (hasLength(this.typeHandlersPackage)) {
        String[] typeHandlersPackageArray = tokenizeToStringArray(this.typeHandlersPackage,
                ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS);
        for (String packageToScan : typeHandlersPackageArray) {
            configuration.getTypeHandlerRegistry().register(packageToScan);
            LOGGER.debug(() -> "Scanned package: '" + packageToScan + "' for type handlers");
        }
    }
    // 如果 typeHandlers 非空，注册对应的 TypeHandler
    if (!isEmpty(this.typeHandlers)) {
        for (TypeHandler<?> typeHandler : this.typeHandlers) {
            configuration.getTypeHandlerRegistry().register(typeHandler);
            LOGGER.debug(() -> "Registered type handler: '" + typeHandler + "'");
        }
    }

    // 设置 `configuration.databaseId` 属性
    if (this.databaseIdProvider != null) {//fix #64 set databaseId before parse mapper xmls
        try {
            configuration.setDatabaseId(this.databaseIdProvider.getDatabaseId(this.dataSource));
        } catch (SQLException e) {
            throw new NestedIOException("Failed getting a databaseId", e);
        }
    }

    // 设置 `configuration.cache` 属性
    if (this.cache != null) {
        configuration.addCache(this.cache);
    }

    // <1> 解析 mybatis-config.xml 配置
    if (xmlConfigBuilder != null) {
        try {
            xmlConfigBuilder.parse();
            LOGGER.debug(() -> "Parsed configuration file: '" + this.configLocation + "'");
        } catch (Exception ex) {
            throw new NestedIOException("Failed to parse config resource: " + this.configLocation, ex);
        } finally {
            ErrorContext.instance().reset();
        }
    }

    // 初始化 TransactionFactory 对象
    if (this.transactionFactory == null) {
        this.transactionFactory = new SpringManagedTransactionFactory();
    }
    // 设置 `configuration.environment` 属性
    configuration.setEnvironment(new Environment(this.environment, this.transactionFactory, this.dataSource));

    // 扫描 Mapper XML 文件们，并进行解析
    if (!isEmpty(this.mapperLocations)) {
        for (Resource mapperLocation : this.mapperLocations) {
            if (mapperLocation == null) {
                continue;
            }

            try {
                XMLMapperBuilder xmlMapperBuilder = new XMLMapperBuilder(mapperLocation.getInputStream(),
                        configuration, mapperLocation.toString(), configuration.getSqlFragments());
                xmlMapperBuilder.parse(); // <2>
            } catch (Exception e) {
                throw new NestedIOException("Failed to parse mapping resource: '" + mapperLocation + "'", e);
            } finally {
                ErrorContext.instance().reset();
            }
            LOGGER.debug(() -> "Parsed mapper file: '" + mapperLocation + "'");
        }
    } else {
        LOGGER.debug(() -> "Property 'mapperLocations' was not specified or no matching resources found");
    }

    // 创建 SqlSessionFactory 对象
    return this.sqlSessionFactoryBuilder.build(configuration);
}
```

- 代码比较长，胖友自己耐心看下注释，我们只挑重点的瞅瞅。
- `<1>` 处，调用 `XMLConfigBuilder#parse()` 方法，解析 `mybatis-config.xml` 配置。即对应 [《MyBatis 初始化（一）之加载 mybatis-config》](http://svip.iocoder.cn/MyBatis/builder-package-1) 文章。
- `<2>` 处，调用 `XMLMapperBuilder#parse()` 方法，解析 Mapper XML 配置。即对应 [《MyBatis 初始化（二）之加载 Mapper 映射配置文件》](http://svip.iocoder.cn/MyBatis/builder-package-2) 。
- `<3>` 处，调用 `SqlSessionFactoryBuilder#build(Configuration config)` 方法，创建 SqlSessionFactory 对象。
- 另外，我们看到 `LOGGER` 的使用。那么，可能胖友会跟艿艿有一样的疑问，`org.mybatis.logging` 包下，又定义了 Logger 呢？因为想使用方法参数为 `Supplier<String>` 的方法，即使用起来更加方便。这也是 `mybatis-spring` 项目的 `logging` 包的用途。感兴趣的胖友，可以自己去看看。

### 2.3 FactoryBean子接口

#### 2.3.1 getObject

`#getObject()` 方法，获得 SqlSessionFactory 对象。代码如下：

```java
// SqlSessionFactoryBean.java

@Override
public SqlSessionFactory getObject() throws Exception {
    // 保证 SqlSessionFactory 对象的初始化
    if (this.sqlSessionFactory == null) {
        afterPropertiesSet();
    }

    return this.sqlSessionFactory;
}
```

#### 2.3.2 getObjectType

```java
// SqlSessionFactoryBean.java

@Override
public Class<? extends SqlSessionFactory> getObjectType() {
    return this.sqlSessionFactory == null ? SqlSessionFactory.class : this.sqlSessionFactory.getClass();
}
```

#### 2.3.3 isSingleton

```java
// SqlSessionFactoryBean.java

@Override
public boolean isSingleton() {
    return true;
}
```

### 2.4 onApplicationEvent

`#onApplicationEvent()` 方法，监听 ContextRefreshedEvent 事件，如果 MapperStatement 们，没有都初始化**都**完成，会抛出 IncompleteElementException 异常。代码如下：

```java
// SqlSessionFactoryBean.java

@Override
public void onApplicationEvent(ApplicationEvent event) {
    if (failFast && event instanceof ContextRefreshedEvent) {
        // fail-fast -> check all statements are completed
        // 如果 MapperStatement 们，没有都初始化完成，会抛出 IncompleteElementException 异常
        this.sqlSessionFactory.getConfiguration().getMappedStatementNames();
    }
}
```

- 使用该功能时，需要设置 `fastFast` 属性，为 `true` 。

------

在 `mybatis-spring` 项目中，提供了多种配置 Mapper 的方式。下面，我们一种一种来看。😈 实际上，配置 `SqlSessionFactoryBean.mapperLocations` 属性，也是方式之一。嘿嘿嘿。

## 3. MapperFactoryBean

`org.mybatis.spring.mapper.MapperFactoryBean` ，实现 `FactoryBean `接口，继承 `SqlSessionDaoSupport `抽象类，创建 Mapper 对象。

使用示例如下：

```xml
<!-- Directly injecting mappers. The required SqlSessionFactory will be autowired. -->
<bean id="userMapper" class="org.mybatis.spring.mapper.MapperFactoryBean" autowire="byType">
	<property name="mapperInterface" value="org.mybatis.spring.sample.mapper.UserMapper" />
</bean>
```

- 该示例来自 `org.mybatis.spring.sample.SampleMapperTest` 单元测试。胖友可以基于它调试。

### 3.1 MapperFactoryBean

```java
    // MapperFactoryBean.java

/**
 * Mapper 接口
 */
private Class<T> mapperInterface;
/**
 * 是否添加到 {@link Configuration} 中
 */
private boolean addToConfig = true;

public MapperFactoryBean() {
    //intentionally empty
}

public MapperFactoryBean(Class<T> mapperInterface) {
    this.mapperInterface = mapperInterface;
}

// 省略 setting 方法
```

### 3.2 checkDaoConfig

```java
// MapperFactoryBean.java

@Override
protected void checkDaoConfig() {
    // <1> 校验 sqlSessionTemplate 非空
    super.checkDaoConfig();
    // <2> 校验 mapperInterface 非空
    notNull(this.mapperInterface, "Property 'mapperInterface' is required");

    // <3> 添加 Mapper 接口到 configuration 中
    Configuration configuration = getSqlSession().getConfiguration();
    if (this.addToConfig && !configuration.hasMapper(this.mapperInterface)) {
        try {
            configuration.addMapper(this.mapperInterface);
        } catch (Exception e) {
            logger.error("Error while adding the mapper '" + this.mapperInterface + "' to configuration.", e);
            throw new IllegalArgumentException(e);
        } finally {
            ErrorContext.instance().reset();
        }
    }
}
```

- 该方法，是在 `org.springframework.dao.support.DaoSupport` 定义，被 `#afterPropertiesSet()` 方法所调用，代码如下：

  ```java
  // DaoSupport.java
  
  public final void afterPropertiesSet() throws IllegalArgumentException, BeanInitializationException {
      this.checkDaoConfig();
  
      try {
          this.initDao();
      } catch (Exception var2) {
          throw new BeanInitializationException("Initialization of DAO failed", var2);
      }
  }
  ```

  - 所以此处，就和 [「2.2 afterPropertiesSet」](#2.2 afterPropertiesSet) 的等价。

- `<1>` 处，调用父 `SqlSessionDaoSupport#checkDaoConfig()` 方法，校验 `sqlSessionTemplate` 非空。😈 关于 SqlSessionDaoSupport 抽象类，后续我们详细解析。

- `<2>` 处，校验 `mapperInterface` 非空。

- `<3>` 处，调用 `Configuration#addMapper(mapperInterface)` 方法，添加 Mapper 接口到 `configuration` 中。

### 3.3 getObject

`#getObject()` 方法，获得 Mapper 对象。**注意**，返回的是基于 Mapper 接口自动生成的代理对象。代码如下：

```java
// MapperFactoryBean.java

@Override
public T getObject() throws Exception {
    return getSqlSession().getMapper(this.mapperInterface);
}
```

### 3.4 getObjectType

```java
// MapperFactoryBean.java

@Override
public Class<T> getObjectType() {
    return this.mapperInterface;
}
```

### 3.5 isSingleton

```java
// MapperFactoryBean.java

@Override
public boolean isSingleton() {
    return true;
}
```

## 4. @MapperScan

> 艿艿：本小节，需要胖友对 Spring IOC 有一定的了解。如果不熟悉的胖友，建议不需要特别深入的理解。或者说，先去看 [《精尽 Spring 源码解析》](http://svip.iocoder.cn/categories/Spring/) 。

`org.mybatis.spring.annotation.@MapperScan` 注解，指定需要扫描的包，将包中符合的 Mapper 接口，注册成 `beanClass` 为 MapperFactoryBean 的 BeanDefinition 对象，从而实现创建 Mapper 对象。代码如下：

```java
// MapperScan.java

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(MapperScannerRegistrar.class)
@Repeatable(MapperScans.class)
public @interface MapperScan {

    /**
     * Alias for the {@link #basePackages()} attribute. Allows for more concise
     * annotation declarations e.g.:
     * {@code @MapperScan("org.my.pkg")} instead of {@code @MapperScan(basePackages = "org.my.pkg"})}.
     *
     * 和 {@link #basePackages()} 相同意思
     *
     * @return base package names
     */
    String[] value() default {};

    /**
     * Base packages to scan for MyBatis interfaces. Note that only interfaces
     * with at least one method will be registered; concrete classes will be
     * ignored.
     *
     * 扫描的包地址
     *
     * @return base package names for scanning mapper interface
     */
    String[] basePackages() default {};

    /**
     * Type-safe alternative to {@link #basePackages()} for specifying the packages
     * to scan for annotated components. The package of each class specified will be scanned.
     * <p>Consider creating a special no-op marker class or interface in each package
     * that serves no purpose other than being referenced by this attribute.
     *
     * @return classes that indicate base package for scanning mapper interface
     */
    Class<?>[] basePackageClasses() default {};

    /**
     * The {@link BeanNameGenerator} class to be used for naming detected components
     * within the Spring container.
     *
     * @return the class of {@link BeanNameGenerator}
     */
    Class<? extends BeanNameGenerator> nameGenerator() default BeanNameGenerator.class;

    /**
     * This property specifies the annotation that the scanner will search for.
     * <p>
     * The scanner will register all interfaces in the base package that also have
     * the specified annotation.
     * <p>
     * Note this can be combined with markerInterface.
     *
     * 指定注解
     *
     * @return the annotation that the scanner will search for
     */
    Class<? extends Annotation> annotationClass() default Annotation.class;

    /**
     * This property specifies the parent that the scanner will search for.
     * <p>
     * The scanner will register all interfaces in the base package that also have
     * the specified interface class as a parent.
     * <p>
     * Note this can be combined with annotationClass.
     *
     * 指定接口
     *
     * @return the parent that the scanner will search for
     */
    Class<?> markerInterface() default Class.class;

    /**
     * Specifies which {@code SqlSessionTemplate} to use in the case that there is
     * more than one in the spring context. Usually this is only needed when you
     * have more than one datasource.
     *
     * 指向的 SqlSessionTemplate 的名字
     *
     * @return the bean name of {@code SqlSessionTemplate}
     */
    String sqlSessionTemplateRef() default "";

    /**
     * Specifies which {@code SqlSessionFactory} to use in the case that there is
     * more than one in the spring context. Usually this is only needed when you
     * have more than one datasource.
     *
     * 指向的 SqlSessionFactory 的名字
     *
     * @return the bean name of {@code SqlSessionFactory}
     */
    String sqlSessionFactoryRef() default "";

    /**
     * Specifies a custom MapperFactoryBean to return a mybatis proxy as spring bean.
     *
     * 可自定义 MapperFactoryBean 的实现类
     *
     * @return the class of {@code MapperFactoryBean}
     */
    Class<? extends MapperFactoryBean> factoryBean() default MapperFactoryBean.class;

}
```

- 属性比较多，实际上，胖友常用的就是 `value()` 属性。
- 重点是 `@Import(MapperScannerRegistrar.class)`。为什么呢？`@Import` 注解，负责资源的导入。如果导入的是一个 Java 类，例如此处为 `MapperScannerRegistrar `类，Spring 会将其注册成一个 Bean 对象。而 MapperScannerRegistrar 类呢？详细解析，见 [「4.1 MapperScannerRegistrar」](#4.1 MapperScannerRegistrar) 。

<span id='go4_example'>使用示例如下：</span>

```java
@Configuration
@ImportResource("classpath:org/mybatis/spring/sample/config/applicationContext-infrastructure.xml")
@MapperScan("org.mybatis.spring.sample.mapper") // here
static class AppConfig {
}
```

- 该示例来自 `org.mybatis.spring.sample.SampleEnableTest` 单元测试。胖友可以基于它调试。

### 4.1 MapperScannerRegistrar

`org.mybatis.spring.annotation.MapperScannerRegistrar` ，实现 `ImportBeanDefinitionRegistrar`、`ResourceLoaderAware `接口，`@MapperScann` 的注册器，负责将扫描到的 Mapper 接口，注册成 beanClass 为 MapperFactoryBean 的 BeanDefinition 对象，从而实现创建 Mapper 对象。

#### 4.1.1 构造方法

```java
// MapperScannerRegistrar.java

/**
 * ResourceLoader 对象
 */
private ResourceLoader resourceLoader;

@Override
public void setResourceLoader(ResourceLoader resourceLoader) {
    this.resourceLoader = resourceLoader;
}
```

- 因为实现了 `ResourceLoaderAware `接口，所以 `resourceLoader` 属性，能够被注入。

#### 4.1.2 registerBeanDefinitions

```java
// MapperScannerRegistrar.java -> ImportBeanDefinitionRegistrar

@Override
public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
    // <1> 获得 @MapperScan 注解信息
    AnnotationAttributes mapperScanAttrs = AnnotationAttributes
            .fromMap(importingClassMetadata.getAnnotationAttributes(MapperScan.class.getName()));
    // <2> 扫描包，将扫描到的 Mapper 接口，注册成 beanClass 为 MapperFactoryBean 的 BeanDefinition 对象
    registerBeanDefinitions(mapperScanAttrs, registry);
}

void registerBeanDefinitions(AnnotationAttributes annoAttrs, BeanDefinitionRegistry registry) {
    // <3.1> 创建 ClassPathMapperScanner 对象
    ClassPathMapperScanner scanner = new ClassPathMapperScanner(registry);

    // <3.2> 设置 resourceLoader 属性到 scanner 中
    // this check is needed in Spring 3.1
    if (resourceLoader != null) {
        scanner.setResourceLoader(resourceLoader);
    }

    // <3.3> 获得 @(MapperScan 注解上的属性，设置到 scanner 中
    Class<? extends Annotation> annotationClass = annoAttrs.getClass("annotationClass");
    if (!Annotation.class.equals(annotationClass)) {
        scanner.setAnnotationClass(annotationClass);
    }
    Class<?> markerInterface = annoAttrs.getClass("markerInterface");
    if (!Class.class.equals(markerInterface)) {
        scanner.setMarkerInterface(markerInterface);
    }
    Class<? extends BeanNameGenerator> generatorClass = annoAttrs.getClass("nameGenerator");
    if (!BeanNameGenerator.class.equals(generatorClass)) {
        scanner.setBeanNameGenerator(BeanUtils.instantiateClass(generatorClass));
    }
    Class<? extends MapperFactoryBean> mapperFactoryBeanClass = annoAttrs.getClass("factoryBean");
    if (!MapperFactoryBean.class.equals(mapperFactoryBeanClass)) {
        scanner.setMapperFactoryBean(BeanUtils.instantiateClass(mapperFactoryBeanClass));
    }
    scanner.setSqlSessionTemplateBeanName(annoAttrs.getString("sqlSessionTemplateRef"));
    scanner.setSqlSessionFactoryBeanName(annoAttrs.getString("sqlSessionFactoryRef"));

    // <4> 获得要扫描的包
    List<String> basePackages = new ArrayList<>();
    basePackages.addAll(
            Arrays.stream(annoAttrs.getStringArray("value")) // 包
                    .filter(StringUtils::hasText)
                    .collect(Collectors.toList()));
    basePackages.addAll(
            Arrays.stream(annoAttrs.getStringArray("basePackages")) // 包
                    .filter(StringUtils::hasText)
                    .collect(Collectors.toList()));
    basePackages.addAll(
            Arrays.stream(annoAttrs.getClassArray("basePackageClasses")) // 类
                    .map(ClassUtils::getPackageName)
                    .collect(Collectors.toList()));

    // <5> 注册 scanner 的过滤器
    scanner.registerFilters();
    // <6> 执行扫描
    scanner.doScan(StringUtils.toStringArray(basePackages));
}
```

- `<1>` 处，获得 `@MapperScan` 注解信息。
- `<2>` 处，调用 `#registerBeanDefinitions(AnnotationAttributes annoAttrs, BeanDefinitionRegistry registry)` 方法，扫描包，将扫描到的 Mapper 接口，注册成 `beanClass` 为 MapperFactoryBean 的 BeanDefinition 对象。
- `<3.1>`处，创建 `ClassPathMapperScanner `对象。下面的代码，都是和 `ClassPathMapperScanner `相关。所以，详细的解析，[「4.2 ClassPathMapperScanner」](#4.2 ClassPathMapperScanner) 。
  - `<3.2>` 处， 设置 `resourceLoader` 属性到 `scanner` 中。
  - `<3.3>` 处，获得 `@MapperScan` 注解上的属性，设置到 `scanner` 中。
- `<4>` 处，获得要扫描的包。
- `<5>` 处，调用 `ClassPathMapperScanner#registerFilters()` 方法，注册 scanner 的过滤器。
- `<6>` 处，调用 `ClassPathMapperScanner#doScan(String... basePackages)` 方法，执行扫描，将扫描到的 Mapper 接口，注册成 beanClass 为 MapperFactoryBean 的 `BeanDefinition `对象，从而实现创建 Mapper 对象。
- 😈 上面看了一堆的 ClassPathMapperScanner 类的调用，下面开始我们的旅程。

### 4.2 ***ClassPathMapperScanner

`org.mybatis.spring.mapper.ClassPathMapperScanner` ，继承 `org.springframework.context.annotation.ClassPathMapperScanner` 类，负责执行扫描，将扫描到的 Mapper 接口，注册成 beanClass 为 MapperFactoryBean 的 BeanDefinition 对象，从而实现创建 Mapper 对象。

可能很多胖友不熟悉 ClassPathMapperScanner 类，可以看看 [《Spring自定义类扫描器》](https://fangjian0423.github.io/2017/06/11/spring-custom-component-provider/) 文章。哈哈哈，艿艿也是现学的。

#### 4.2.1 构造方法

```java
// ClassPathMapperScanner.java

/**
 * 是否添加到 {@link org.apache.ibatis.session.Configuration} 中
 */
private boolean addToConfig = true;

private SqlSessionFactory sqlSessionFactory;

private SqlSessionTemplate sqlSessionTemplate;

/**
 * {@link #sqlSessionTemplate} 的 bean 名字
 */
private String sqlSessionTemplateBeanName;

/**
 * {@link #sqlSessionFactory} 的 bean 名字
 */
private String sqlSessionFactoryBeanName;

/**
 * 指定注解
 */
private Class<? extends Annotation> annotationClass;

/**
 * 指定接口
 */
private Class<?> markerInterface;

/**
 * MapperFactoryBean 对象
 */
private MapperFactoryBean<?> mapperFactoryBean = new MapperFactoryBean<>();

// ... 省略 setting 方法
```

#### 4.2.2 registerFilters

`#registerFilters()` 方法，注册过滤器。代码如下：

```java
// ClassPathMapperScanner.java

/**
 * Configures parent scanner to search for the right interfaces. It can search
 * for all interfaces or just for those that extends a markerInterface or/and
 * those annotated with the annotationClass
 *
 * 注册过滤器
 */
public void registerFilters() {
    boolean acceptAllInterfaces = true; // 是否接受所有接口

    // if specified, use the given annotation and / or marker interface
    // 如果指定了注解，则添加 INCLUDE 过滤器 AnnotationTypeFilter 对象
    if (this.annotationClass != null) {
        addIncludeFilter(new AnnotationTypeFilter(this.annotationClass));
        acceptAllInterfaces = false; // 标记不是接受所有接口
    }

    // override AssignableTypeFilter to ignore matches on the actual marker interface
    // 如果指定了接口，则添加 INCLUDE 过滤器 AssignableTypeFilter 对象
    if (this.markerInterface != null) {
        addIncludeFilter(new AssignableTypeFilter(this.markerInterface) {
            @Override
            protected boolean matchClassName(String className) {
                return false;
            }
        });
        acceptAllInterfaces = false; // 标记不是接受所有接口
    }

    // 如果接受所有接口，则添加自定义 INCLUDE 过滤器 TypeFilter ，全部返回 true
    if (acceptAllInterfaces) {
        // default include filter that accepts all classes
        addIncludeFilter((metadataReader, metadataReaderFactory) -> true);
    }

    // exclude package-info.java
    // 添加 INCLUDE 过滤器，排除 package-info.java
    addExcludeFilter((metadataReader, metadataReaderFactory) -> {
        String className = metadataReader.getClassMetadata().getClassName();
        return className.endsWith("package-info");
    });
}
```

- 根据配置，添加 `INCLUDE `和 `EXCLUDE `过滤器。

#### 4.2.3 doScan

`#doScan(String... basePackages)` 方法，执行扫描，将扫描到的 Mapper 接口，注册成 beanClass 为 MapperFactoryBean 的 BeanDefinition 对象。代码如下：

```java
// ClassPathMapperScanner.java

/**
 * Calls the parent search that will search and register all the candidates.
 * Then the registered objects are post processed to set them as
 * MapperFactoryBeans
 */
@Override
public Set<BeanDefinitionHolder> doScan(String... basePackages) {
    // <1> 执行扫描，获得包下符合的类们，并分装成 BeanDefinitionHolder 对象的集合
    Set<BeanDefinitionHolder> beanDefinitions = super.doScan(basePackages);

    if (beanDefinitions.isEmpty()) {
        LOGGER.warn(() -> "No MyBatis mapper was found in '" + Arrays.toString(basePackages) + "' package. Please check your configuration.");
    } else {
        // 处理 BeanDefinitionHolder 对象的集合
        processBeanDefinitions(beanDefinitions);
    }

    return beanDefinitions;
}
```

- `<1>` 处，调用父 `ClassPathBeanDefinitionScanner#doScan(basePackages)` 方法，执行扫描，获得包下符合的类们，并分装成 BeanDefinitionHolder 对象的集合。

- `<2>` 处，调用 `#processBeanDefinitions((Set<BeanDefinitionHolder> beanDefinitions)` 方法，处理 BeanDefinitionHolder 对象的集合。代码如下：

  ```java
  // ClassPathMapperScanner.java
  
  private void processBeanDefinitions(Set<BeanDefinitionHolder> beanDefinitions) {
      GenericBeanDefinition definition;
      // <1> 遍历 BeanDefinitionHolder 数组，逐一设置属性
      for (BeanDefinitionHolder holder : beanDefinitions) {
          definition = (GenericBeanDefinition) holder.getBeanDefinition();
          String beanClassName = definition.getBeanClassName();
          LOGGER.debug(() -> "Creating MapperFactoryBean with name '" + holder.getBeanName()
                  + "' and '" + beanClassName + "' mapperInterface");
  
          // the mapper interface is the original class of the bean
          // but, the actual class of the bean is MapperFactoryBean
          // <2> 此处 definition 的 beanClass 为 Mapper 接口，需要修改成 MapperFactoryBean 类，从而创建 Mapper 代理对象
          definition.getConstructorArgumentValues().addGenericArgumentValue(beanClassName); // issue #59
          definition.setBeanClass(this.mapperFactoryBean.getClass());
  
          // <3> 设置 `MapperFactoryBean.addToConfig` 属性
          definition.getPropertyValues().add("addToConfig", this.addToConfig);
  
          boolean explicitFactoryUsed = false; // <4.1>是否已经显式设置了 sqlSessionFactory 或 sqlSessionFactory 属性
          // <4.2> 如果 sqlSessionFactoryBeanName 或 sqlSessionFactory 非空，设置到 `MapperFactoryBean.sqlSessionFactory` 属性
          if (StringUtils.hasText(this.sqlSessionFactoryBeanName)) {
              definition.getPropertyValues().add("sqlSessionFactory", new RuntimeBeanReference(this.sqlSessionFactoryBeanName));
              explicitFactoryUsed = true;
          } else if (this.sqlSessionFactory != null) {
              definition.getPropertyValues().add("sqlSessionFactory", this.sqlSessionFactory);
              explicitFactoryUsed = true;
          }
          // <4.3> 如果 sqlSessionTemplateBeanName 或 sqlSessionTemplate 非空，设置到 `MapperFactoryBean.sqlSessionTemplate` 属性
          if (StringUtils.hasText(this.sqlSessionTemplateBeanName)) {
              if (explicitFactoryUsed) {
                  LOGGER.warn(() -> "Cannot use both: sqlSessionTemplate and sqlSessionFactory together. sqlSessionFactory is ignored.");
              }
              definition.getPropertyValues().add("sqlSessionTemplate", new RuntimeBeanReference(this.sqlSessionTemplateBeanName));
              explicitFactoryUsed = true;
          } else if (this.sqlSessionTemplate != null) {
              if (explicitFactoryUsed) {
                  LOGGER.warn(() -> "Cannot use both: sqlSessionTemplate and sqlSessionFactory together. sqlSessionFactory is ignored.");
              }
              definition.getPropertyValues().add("sqlSessionTemplate", this.sqlSessionTemplate);
              explicitFactoryUsed = true;
          }
          // <4.4> 如果未显式设置，则设置根据类型自动注入
          if (!explicitFactoryUsed) {
              LOGGER.debug(() -> "Enabling autowire by type for MapperFactoryBean with name '" + holder.getBeanName() + "'.");
              definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);
          }
      }
  }
  ```

  - 虽然方法很长，重点就是修改 BeanDefinitionHolder 的相关属性。
  - `<1>` 处，遍历 BeanDefinitionHolder 数组，逐一设置属性。
  - `<2>` 处，此处 `definition` 的 `beanClass` 为 Mapper 接口，需要修改成 MapperFactoryBean 类，从而创建 Mapper 代理对象。
  - `<3>` 处，设置 `MapperFactoryBean.addToConfig` 属性。
  - `<4.1>`处，是否已经显式设置了`sqlSessionFactory 或 sqlSessionFactory`属性。
    - `<4.2>` 处，如果 `sqlSessionFactoryBeanName` 或 `sqlSessionFactory` 非空，设置到 `MapperFactoryBean.sqlSessionFactory` 属性。
    - `<4.3>` 处，如果 `sqlSessionTemplateBeanName` 或 `sqlSessionTemplate` 非空，设置到 `MapperFactoryBean.sqlSessionTemplate` 属性。
    - `<4.4>` 处，如果未显式设置，**则设置根据类型自动注入**。

### 4.3 @MapperScans

`org.mybatis.spring.annotation.@MapperScans` ，多 `@MapperScan` 的注解，功能是相同的。代码如下：

```java
// MapperScans.java

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(MapperScannerRegistrar.RepeatingRegistrar.class)
public @interface MapperScans {

    /**
     * @return @MapperScan 数组
     */
    MapperScan[] value();

}
```

- 此处，`@Import(MapperScannerRegistrar.RepeatingRegistrar.class)` 是 RepeatingRegistrar 类。

### 4.4 RepeatingRegistrar

RepeatingRegistrar ，是 MapperScannerRegistrar 的内部静态类，继承 MapperScannerRegistrar 类，`@MapperScans` 的注册器。代码如下：

```java
// MapperScannerRegistrar.java

static class RepeatingRegistrar extends MapperScannerRegistrar {
    
    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        // 获得 @MapperScans 注解信息
        AnnotationAttributes mapperScansAttrs = AnnotationAttributes
                .fromMap(importingClassMetadata.getAnnotationAttributes(MapperScans.class.getName()));
        // 遍历 @MapperScans 的值，调用 `#registerBeanDefinitions(mapperScanAttrs, registry)` 方法，循环扫描处理
        Arrays.stream(mapperScansAttrs.getAnnotationArray("value"))
                .forEach(mapperScanAttrs -> registerBeanDefinitions(mapperScanAttrs, registry));
    }

}
```

- [「4.1.2 registerBeanDefinitions」](http://svip.iocoder.cn/MyBatis/Spring-Integration-2/#) 的**循环**版。

## 5. 自定义 `<mybatis:scan />` 标签

使用示例如下：

```xml
 <!-- Scan for mappers and let them be autowired; notice there is no
     UserDaoImplementation needed. The required SqlSessionFactory will be
     autowired. -->
<mybatis:scan base-package="org.mybatis.spring.sample.mapper" />
```

- 该示例来自 `org.mybatis.spring.sample.SampleNamespaceTest` 单元测试。胖友可以基于它调试。

简单来理解，`<mybatis:scan />` 标签，和 `@MapperScan` 注解，用途是等价的。

### 5.1 spring.schemas

MyBatis 在 `META-INF/spring.schemas` 定义如下：

```xml-dtd
http\://mybatis.org/schema/mybatis-spring-1.2.xsd=org/mybatis/spring/config/mybatis-spring.xsd
http\://mybatis.org/schema/mybatis-spring.xsd=org/mybatis/spring/config/mybatis-spring.xsd
```

- xmlns 为 `http://mybatis.org/schema/mybatis-spring-1.2.xsd` 或 `http://mybatis.org/schema/mybatis-spring.xsd` 。
- xsd 为 `org/mybatis/spring/config/mybatis-spring.xsd` 。

### 5.2 mybatis-spring.xsd

[链接如下](https://github.com/YunaiV/mybatis-spring/blob/master/src/main/java/org/mybatis/spring/config/mybatis-spring.xsd)

![定义如下：](http://static.iocoder.cn/images/MyBatis/2020_06_04/01.png)

### 5.3 spring.handler

`spring.handlers` 定义如下：

```dtd
http\://mybatis.org/schema/mybatis-spring=org.mybatis.spring.config.NamespaceHandler
```

定义了 MyBatis 的 XML Namespace 的处理器 NamespaceHandler 。

### 5.4 NamespaceHandler

`org.mybatis.spring.config.NamespaceHandler` ，继承 NamespaceHandlerSupport 抽象类，MyBatis 的 XML Namespace 的处理器。代码如下：

```java
// NamespaceHandler.java

public class NamespaceHandler extends NamespaceHandlerSupport {

    @Override
    public void init() {
        registerBeanDefinitionParser("scan", new MapperScannerBeanDefinitionParser());
    }

}
```

- `<mybatis:scan />` 标签，使用 MapperScannerBeanDefinitionParser 解析。

### 5.5 MapperScannerBeanDefinitionParser

`org.mybatis.spring.config.MapperScannerBeanDefinitionParser` ，实现 BeanDefinitionParser 接口，`<mybatis:scan />` 的解析器。代码如下：

```java
// BeanDefinitionParser.java

public class MapperScannerBeanDefinitionParser implements BeanDefinitionParser {

    private static final String ATTRIBUTE_BASE_PACKAGE = "base-package";
    private static final String ATTRIBUTE_ANNOTATION = "annotation";
    private static final String ATTRIBUTE_MARKER_INTERFACE = "marker-interface";
    private static final String ATTRIBUTE_NAME_GENERATOR = "name-generator";
    private static final String ATTRIBUTE_TEMPLATE_REF = "template-ref";
    private static final String ATTRIBUTE_FACTORY_REF = "factory-ref";

    /**
     * {@inheritDoc}
     */
    @Override
    public synchronized BeanDefinition parse(Element element, ParserContext parserContext) {
        // 创建 ClassPathMapperScanner 对象
        ClassPathMapperScanner scanner = new ClassPathMapperScanner(parserContext.getRegistry());
        ClassLoader classLoader = scanner.getResourceLoader().getClassLoader();
        XmlReaderContext readerContext = parserContext.getReaderContext();
        scanner.setResourceLoader(readerContext.getResourceLoader()); // 设置 resourceLoader 属性
        try {
            // 解析 annotation 属性
            String annotationClassName = element.getAttribute(ATTRIBUTE_ANNOTATION);
            if (StringUtils.hasText(annotationClassName)) {
                @SuppressWarnings("unchecked")
                Class<? extends Annotation> markerInterface = (Class<? extends Annotation>) classLoader.loadClass(annotationClassName);
                scanner.setAnnotationClass(markerInterface);
            }
            // 解析 marker-interface 属性
            String markerInterfaceClassName = element.getAttribute(ATTRIBUTE_MARKER_INTERFACE);
            if (StringUtils.hasText(markerInterfaceClassName)) {
                Class<?> markerInterface = classLoader.loadClass(markerInterfaceClassName);
                scanner.setMarkerInterface(markerInterface);
            }
            // 解析 name-generator 属性
            String nameGeneratorClassName = element.getAttribute(ATTRIBUTE_NAME_GENERATOR);
            if (StringUtils.hasText(nameGeneratorClassName)) {
                Class<?> nameGeneratorClass = classLoader.loadClass(nameGeneratorClassName);
                BeanNameGenerator nameGenerator = BeanUtils.instantiateClass(nameGeneratorClass, BeanNameGenerator.class);
                scanner.setBeanNameGenerator(nameGenerator);
            }
        } catch (Exception ex) {
            readerContext.error(ex.getMessage(), readerContext.extractSource(element), ex.getCause());
        }
        // 解析 template-ref 属性
        String sqlSessionTemplateBeanName = element.getAttribute(ATTRIBUTE_TEMPLATE_REF);
        scanner.setSqlSessionTemplateBeanName(sqlSessionTemplateBeanName);
        // 解析 factory-ref 属性
        String sqlSessionFactoryBeanName = element.getAttribute(ATTRIBUTE_FACTORY_REF);
        scanner.setSqlSessionFactoryBeanName(sqlSessionFactoryBeanName);

        // 注册 scanner 的过滤器
        scanner.registerFilters();

        // 获得要扫描的包
        String basePackage = element.getAttribute(ATTRIBUTE_BASE_PACKAGE);
        // 执行扫描
        scanner.scan(StringUtils.tokenizeToStringArray(basePackage, ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS));
        return null;
    }

}
```

- 代码实现上，和 [「4.2 ClassPathMapperScanner」](#4.2 ClassPathMapperScanner) 是基本一致的。所以就不详细解析啦。

## 6. MapperScannerConfigurer

`org.mybatis.spring.mapper.MapperScannerConfigurer` ，实现BeanDefinitionRegistryPostProcessor、InitializingBean、ApplicationContextAware、BeanNameAware 接口，定义需要扫描的包，将包中符合的 Mapper 接口，注册成 beanClass 为 MapperFactoryBean 的 BeanDefinition 对象，从而实现创建 Mapper 对象。

使用示例如下：

```xml
// XML

<!-- Scan for mappers and let them be autowired; notice there is no
     UserDaoImplementation needed. The required SqlSessionFactory will be
     autowired. -->
<bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
    <property name="basePackage" value="org.mybatis.spring.sample.mapper" />
</bean>
```

- 该示例来自 `org.mybatis.spring.sample.MapperScannerConfigurer` 单元测试。胖友可以基于它调试。

### 6.1 构造方法

```java
// MapperScannerConfigurer.java

private String basePackage;

private boolean addToConfig = true;

private SqlSessionFactory sqlSessionFactory;

private SqlSessionTemplate sqlSessionTemplate;

private String sqlSessionFactoryBeanName;

private String sqlSessionTemplateBeanName;

private Class<? extends Annotation> annotationClass;

private Class<?> markerInterface;

private ApplicationContext applicationContext;

private String beanName;

private boolean processPropertyPlaceHolders;

private BeanNameGenerator nameGenerator;

// 省略 setting 方法
```

### 6.2 afterPropertiesSet

```java
// MapperScannerConfigurer.java

@Override
public void afterPropertiesSet() throws Exception {
    notNull(this.basePackage, "Property 'basePackage' is required");
}
```

- 啥都不做。

### 6.3 postProcessBeanFactory

```java
// MapperScannerConfigurer.java

@Override
public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    // left intentionally blank
}
```

### 6.4 postProcessBeanDefinitionRegistry

```java
// MapperScannerConfigurer.java

@Override
public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
    // <1> 如果有属性占位符，则进行获得，例如 ${basePackage} 等等
    if (this.processPropertyPlaceHolders) {
        processPropertyPlaceHolders();
    }

    // <2> 创建 ClassPathMapperScanner 对象，并设置其相关属性
    ClassPathMapperScanner scanner = new ClassPathMapperScanner(registry);
    scanner.setAddToConfig(this.addToConfig);
    scanner.setAnnotationClass(this.annotationClass);
    scanner.setMarkerInterface(this.markerInterface);
    scanner.setSqlSessionFactory(this.sqlSessionFactory);
    scanner.setSqlSessionTemplate(this.sqlSessionTemplate);
    scanner.setSqlSessionFactoryBeanName(this.sqlSessionFactoryBeanName);
    scanner.setSqlSessionTemplateBeanName(this.sqlSessionTemplateBeanName);
    scanner.setResourceLoader(this.applicationContext);
    scanner.setBeanNameGenerator(this.nameGenerator);
    // 注册 scanner 过滤器
    scanner.registerFilters();
    // 执行扫描
    scanner.scan(StringUtils.tokenizeToStringArray(this.basePackage, ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS));
}
```

- `<1>` 处，调用 `#processPropertyPlaceHolders()` 方法，如果有属性占位符，则进行获得，例如 `${basePackage}` 等等。代码如下：

  ```java
  // MapperScannerConfigurer.java
  
  private void processPropertyPlaceHolders() {
      Map<String, PropertyResourceConfigurer> prcs = applicationContext.getBeansOfType(PropertyResourceConfigurer.class);
  
      if (!prcs.isEmpty() && applicationContext instanceof ConfigurableApplicationContext) {
          BeanDefinition mapperScannerBean = ((ConfigurableApplicationContext) applicationContext)
                  .getBeanFactory().getBeanDefinition(beanName);
  
          // PropertyResourceConfigurer does not expose any methods to explicitly perform
          // property placeholder substitution. Instead, create a BeanFactory that just
          // contains this mapper scanner and post process the factory.
          DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
          factory.registerBeanDefinition(beanName, mapperScannerBean);
  
          for (PropertyResourceConfigurer prc : prcs.values()) {
              prc.postProcessBeanFactory(factory);
          }
  
          PropertyValues values = mapperScannerBean.getPropertyValues();
  
          this.basePackage = updatePropertyValue("basePackage", values);
          this.sqlSessionFactoryBeanName = updatePropertyValue("sqlSessionFactoryBeanName", values);
          this.sqlSessionTemplateBeanName = updatePropertyValue("sqlSessionTemplateBeanName", values);
      }
  }
  
  // 获得属性值，并转换成 String 类型
  private String updatePropertyValue(String propertyName, PropertyValues values) {
      PropertyValue property = values.getPropertyValue(propertyName);
      Object value = property.getValue();
      if (value instanceof String) {
          return value.toString();
      } else if (value instanceof TypedStringValue) {
          return ((TypedStringValue) value).getValue();
      } else {
          return null;
      }
  }
  ```

- `<2>` 处，代码实现上，和 [「4.2 ClassPathMapperScanner」](http://svip.iocoder.cn/MyBatis/Spring-Integration-2/#) 是基本一致的。所以就不详细解析啦。

## 666. 彩蛋

> **单元测试：**
>
> - [MapperFactoryBean -> SampleMapperTest](#3. MapperFactoryBean)
>
> - [@MapperScan -> `org.mybatis.spring.sample.SampleEnableTest` ](#go4_example)

略微冗长，但是易于理解的一篇文章。

我们简单把 3 ~ 6 小节做个小结的话：

- 「3」`MapperFactoryBean `类，是最**基础**的、**单个**的负责创建 Mapper 代理对象的类。
- 「4」「5」「6」，都是基于 `MapperFactoryBean `之上，使用 `ClassPathMapperScanner `扫描指定包，创建成 `MapperFactoryBean `对象，从而创建 Mapper 代理对象。