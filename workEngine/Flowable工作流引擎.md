# **Flowable工作流引擎**

> 所谓工作流引擎是指workflow作为应用系统的一部分，并为之提供对各应用系统有决定作用的根据角色、分工和条件的不同决定信息传递路由、内容等级等核心解决方案。工作流引擎包括流程的节点管理、流向管理、流程样例管理等重要功能。

## 一、简介

### 1.1、Flowable的特点

- 是一个紧凑高效的管理业务审批工作流
- 核心使用的是BPMN技术
- 可方便的嵌入在Spring体系中

### 1.2、什么是PBMN

> 标准的业务流程模型和符号 (BPMN) 将为企业提供以图形符号理解其内部业务程序的能力，并使组织能够以标准方式交流这些程序。此外，图形符号将有助于理解组织之间的绩效协作和业务交易。这将确保企业了解自身和业务参与者，并使组织能够快速适应新的内部和 B2B 业务环境。

- BPMN 开发了一套标准的业务流程建模符号。如下图就是建模的符号。

![](http://lyx-study-notes.oss-cn-hangzhou.aliyuncs.com/img/BPMN建模符号.png)

- BPMN 定义了一个流程图，该流程图使用上述符号编写。如下图就是通过 BPMN 规则绘画的图。

![](http://lyx-study-notes.oss-cn-hangzhou.aliyuncs.com/img/BPMN流程图.png)

### 1.3、BPMN常用符号

#### 1.3.1、开始节点

表明从此处开始流程，对应上图最左侧单线圆

#### 1.3.2、任务节点

其中包含了很多种任务。其中最常用的就是用户任务。指定审批人都需要此选项。

<img src="http://lyx-study-notes.oss-cn-hangzhou.aliyuncs.com/img/任务节点.png" style="zoom:80%;" />

#### 1.3.3、结束节点

表明从此处结束流程，对应上图最右侧加粗圆

#### 1.3.4、网关组件

网关相当于判断（与，或，非），最常用的三种网关分别是互斥/排他网关，并行网关，相容网关。

- 互斥网关：相当于判断，举例说明，如果输入值大于 20 走 A 节点，小于 20 走 B 节点。

<img src="http://lyx-study-notes.oss-cn-hangzhou.aliyuncs.com/img/排他网关.png" style="zoom:80%;" />

- 并行网关：相容网关成对出现，表示网关中的人全部同意才能够进入下一节点。

<img src="http://lyx-study-notes.oss-cn-hangzhou.aliyuncs.com/img/并行网关.png" style="zoom:80%;" />

注：对应并行网关的流入和流出节点数量可以不是相等的，如一个冰箱网关流入节点有三个，流出节点有2个。

- 相容网关：互斥网关与并行网关的结合体，如果满足 A,B 都互斥条件，则都需要流转，如果只有一个满足，那么只流转满足条件的。

<img src="http://lyx-study-notes.oss-cn-hangzhou.aliyuncs.com/img/相容网关.png" style="zoom:80%;" />

### 1.4、条件表达式

将表达式用${}进行包裹，如上述相容网关中的大于50的流入线为例，再此线中的条件中写入${info>50}；执行流程时，传入变量名与表达式中一致，即info。

其他参考资料：

- https://zhuanlan.zhihu.com/p/417014073
- https://tkjohn.github.io/flowable-userguide/#_introduction

## 二、使用方式

### 2.1、整体流程

<img src="http://lyx-study-notes.oss-cn-hangzhou.aliyuncs.com/img/整体流程图.png" style="zoom: 67%;" />



### 2.2、流程引擎API与服务

引擎API是与Flowable交互的最常用手段。总入口点是`ProcessEngine`。`ProcessEngine`可以使用多种方式创建。使用`ProcessEngine`，可以获得各种提供工作流/BPM方法的服务。`ProcessEngine`与服务对象都是线程安全的，因此可以在服务器中保存并共用同一个引用。

<img src="http://lyx-study-notes.oss-cn-hangzhou.aliyuncs.com/img/Flowable中的API接口.png" style="zoom:80%;" />

具体实现类中包含图中的各个API类以及其他一些类

<img src="http://lyx-study-notes.oss-cn-hangzhou.aliyuncs.com/img/ProcessEngine实现类中的各API引擎对象.png" style="zoom:67%;" />

### 2.3、常用API的作用与用法

flowable提供的所有API中有7个是较为常用的，这7个 API 对应着上文的所有逻辑。

#### 2.3.1、FormService

主要用于表单数据管理的API。

```java
formService.getStartFormKey(); // 获取表单key
formService.getRenderedStartForm(); // 查询表单json（无数据）
```

#### 2.3.2、RepositoryService

提供了在编辑和发布审批流程的 API。主要是模型管理和流程定义的业务 API。

```java
// 1.提供了带条件的查询模型流程定义的api
repositoryService.createXXXQuery();
repositoryService.createModelQuery().list(); // 模型查询 
repositoryService.createProcessDefinitionQuery().list(); // 流程定义查询
repositoryService.createXXXXQuery().XXXKey(XXX); // 查询该key是否存在

// 2.提供一些模型与流程定义的通用方法

repositoryService.getModel(); // 获取模型
repositoryService.saveModel();  // 保存模型
repositoryService.deleteModel(); // 删除模型
repositoryService.createDeployment().deploy(); // 部署模型
repositoryService.getModelEditorSource();  // 获得模型JSON数据的UTF8字符串
repositoryService.getModelEditorSourceExtra();  // 获取PNG格式图像

// 3.流程定义相关
repositoryService.getProcessDefinition(ProcessDefinitionId);  // 获取流程定义具体信息
repositoryService.activateProcessDefinitionById(); // 激活流程定义
repositoryService.suspendProcessDefinitionById();  // 挂起流程定义
repositoryService.deleteDeployment();  // 删除流程定义
repositoryService.getProcessDiagram(); // 获取流程定义图片流
repositoryService.getResourceAsStream(); // 获取流程定义xml流
repositoryService.getBpmnModel(pde.getId()); // 获取bpmn对象（当前进行到的那个节点的流程图使用）

// 4.流程定义授权相关
repositoryService.getIdentityLinksForProcessDefinition() // 流程定义授权列表
repositoryService.addCandidateStarterGroup(); // 新增组流程授权
repositoryService.addCandidateStarterUser(); // 新增用户流程授权
repositoryService.deleteCandidateStarterGroup(); // 删除组流程授权
repositoryService.deleteCandidateStarterUser();  // 删除用户流程授权
```

#### 2.3.3、RuntimeService

处理正在运行的流程的API。

```java
// 发起流程
runtimeService.createProcessInstanceBuilder().start(); 

// 删除正在运行的流程
runtimeService.deleteProcessInstance(); 

// 挂起流程定义
runtimeService.suspendProcessInstanceById(); 

// 激活流程实例
runtimeService.activateProcessInstanceById(); 

// 获取表单中填写的值
runtimeService.getVariables(processInstanceId); 

// 获取以进行的流程图节点 （当前进行到的那个节点的流程图使用）
runtimeService.getActiveActivityIds(processInstanceId); 

// 终止流程
runtimeService.createChangeActivityStateBuilder().moveExecutionsToSingleActivityId(executionIds, endId).changeState(); 
```

#### 2.3.4、HistoryService

在用户发起审批后，会生成流程实例。historyService 为处理流程实例的 API，但是其中包括了已经完成的和未完成的流程实例。

```java
// 查询流程实例列表（历史流程,包括未完成的）
historyService.createHistoricProcessInstanceQuery().list();

// 可以获取历史中表单的信息
historyService.createHistoricProcessInstanceQuery().list().foreach().getValue();

// 根据id查询流程实例
historyService.createHistoricProcessInstanceQuery().processInstanceId(processInstanceId).singleResult(); 

// 删除历史流程
historyService.deleteHistoricProcessInstance(); 

// 删除任务实例
historyService.deleteHistoricTaskInstance(taskid); 

// 流程实例节点列表 （当前进行到的那个节点的流程图使用）
historyService.createHistoricActivityInstanceQuery().processInstanceId(processInstanceId).list(); 


```

#### 2.3.5、TaskService

对流程实例的各个节点的审批处理的API。

```java
// 流转的节点审批
taskService.createTaskQuery().list(); // 待办任务列表
taskService.createTaskQuery().taskId(taskId).singleResult();  // 待办任务详情
taskService.saveTask(task); // 修改任务
taskService.setAssignee(); // 设置审批人
taskService.addComment(); // 设置审批备注
taskService.complete(); // 完成当前审批
taskService.getProcessInstanceComments(processInstanceId); // 查看任务详情（也就是都经过哪些人的审批，意见是什么）
taskService.delegateTask(taskId, delegater); // 委派任务
taskService.claim(taskId, userId); // 认领任务
taskService.unclaim(taskId); // 取消认领
taskService.complete(taskId, completeVariables); // 完成任务

// 任务授权
taskService.addGroupIdentityLink(); // 新增组任务授权
taskService.addUserIdentityLink(); // 新增人员任务授权
taskService.deleteGroupIdentityLink(); // 删除组任务授权
taskService.deleteUserIdentityLink(); // 删除人员任务授权
```

#### 2.3.6、ManagementService

主要用于执行自定义命令的API。

```java
// 执行classA的内部方法 
managementService.executeCommand(new classA());  

// 在自定义的方法中可以使用以下方法获取 repositoryService
ProcessEngineConfiguration processEngineConfiguration =
            CommandContextUtil.getProcessEngineConfiguration(commandContext);
RepositoryService repositoryService = processEngineConfiguration.getRepositoryService();

/*
 * 获取流程定义方法集合,如findById、findLatestProcessDefinitionByKey、
 * findLatestProcessDefinitionByKeyAndTenantId等
 */
ProcessEngineConfigurationImpl processEngineConfiguration =
            CommandContextUtil.getProcessEngineConfiguration(commandContext);
ProcessDefinitionEntityManager processDefinitionEntityManager =
            processEngineConfiguration.getProcessDefinitionEntityManager();

```

2.3.7、IdentityService

用于身份信息获取和保存操作的API。

```java
// 获取审批用户的具体信息
identityService.createUserQuery().userId(userId).singleResult();  
// 获取审批组的具体信息
identityService.createGroupQuery().groupId(groupId).singleResult(); 
```

### 2.4、Flowable表的作用

Flowable初始化时会自动生成34张表，且所有数据库表都以 ACT_开头。第二部分是说明表用途的两字符标示符。服务 API 的命名也大略符合这个规则。

1. ACT_RE_*：'RE’代表 repository。带有这个前缀的表包含“静态”信息，例如流程定义与流程资源（图片、规则等）。
2. ACT_RU_*：'RU’代表 runtime。这些表存储运行时信息，例如流程实例（process instance）、用户任务（user task）、变量（variable）、作业（job）等。Flowable 只在流程实例运行中保存运行时数据，并在流程实例结束时删除记录。这样保证运行时表小和快。
3. ACT_HI_*：'HI’代表 history。这些表存储历史数据，例如已完成的流程实例、变量、任务等。
4. ACT_GE_*：通用数据。在多处使用。
5. ACT_ID_*：表示组织信息，如用户，用户组，等等。（很少使用）

#### 2.4.1、ACT_GE_*

- ACT_GE_BYTEARRAY：保存流程的 bpmn 的 xml 以及流程的 Image 缩略图等信息。
- ACT_GE_PROPERTY：Flowable 相关的基本信息。比如各个 module 使用的版本信息。

#### 2.4.2、ACT_RE_*

- ACT_RE_DEPLOYMENT：部署对象，存储流程名称 ACT_RE_MODEL：基于流程的模型信息 ACT_RE_PROCDEF：流程定义表。
- ACT_RE_PROCDEF：流程定义信息表
- ACT_RE_MODEL：模型信息表(用于Web设计器)
- ACT_PROCDEF_INFO：流程定义动态改变信息表

#### 2.4.3、ACT_RU_*

- ACT_RU_EXECUTION：流程实例与分支执行表。
- ACT_RU_TASK：用户任务表。
- ACT_RU_VARIABLE：变量信息。
- ACT_RU_IDENTITYLINK：参与者相关信息表。 
- ACT_RU_EVENT_SUBSCR：事件订阅表。
- ACT_RU_JOB； 作业表。
- ACT_RU_TIMER_JOB： 定时器表。
- ACT_RU_SUSPENDED_JOB：暂停作业表。
- ACT_RU_DEADLETTER_JOB：死信表。
- ACT_RU_HISTORY_JOB：历史作业表。

#### 2.4.4、ACT_ID_*

- ACT_ID_BYTEARRAY：暂时未知。
- ACT_ID_GROUP：用户组信息。
- ACT_ID_INFO：用户详情。
- ACT_ID_MEMBERSHIP：用户组和用户的关系。
- ACT_ID_PRIV：权限。
- ACT_ID_PRIV_MAPPING：用户组和权限之间的关系。
- ACT_ID_PROPERTY：用户或者用户组属性拓展表。
- ACT_ID_TOKEN：登录相关日志。
- ACT_ID_USER：用户。

#### 2.4.5、ACT_HI_*

- ACT_HI_ACTINST: 流程实例历史。
- ACT_HI_ATTACHMENT：实例的历史附件，几乎不会使用，会加大数据库很大的一个 loadingACT_HI_COMMENT：实例的历史备注。
- ACT_HI_DETAIL：实例流程详细信息。
- ACT_HI_IDENTITYLINK: 实例节点中，如果指定了目标人，产生的历史。
- ACT_HI_PROCINST：流程实例历史。
- ACT_HI_TASKINST：流程实例的任务历史。
- ACT_HI_VARINST：流程实例的变量历史。
- ACT_HI_COMMENT：评论表。
- ACT_EVT_LOG：事件日志表。

## 三、工作流引擎生成表的底层逻辑源码分析

### 3.1、初始化表的测试用例

```java
public void initFlowableTab(){

        ProcessEngineConfiguration cfg = ProcessEngineConfiguration.createStandaloneProcessEngineConfiguration();;
        //设置数据库连接
        cfg.setJdbcDriver("com.mysql.cj.jdbc.Driver");
        cfg.setJdbcUrl("jdbc:mysql://localhost:3306/flowable?useUnicode=true&characterEncoding=utf8" +
                "&zeroDateTimeBehavior=convertToNull&useSSL=true&serverTimezone=GMT%2B8&nullCatalogMeansCurrent=true");
        cfg.setJdbcUsername("root");
        cfg.setJdbcPassword("123456");
        
        // 设置数据库更新策略
        cfg.setDatabaseSchemaUpdate(AbstractEngineConfiguration.DB_SCHEMA_UPDATE_FALSE);
        
        // 初始化创建一次，后续使用时直接获取即可
        ProcessEngine processEngine = cfg.buildProcessEngine();
    }
```

### 3.2、Flowable的数据库更新策略：

数据库更新策略，其取值有以下五个：

```
package org.flowable.common.engine.impl;

public abstract class AbstractEngineConfiguration {
	......
	
	// 默认值。在Flowable启动时，会对比数据库表中保存的版本，如果没有表或者版本不匹配，将抛出异常。（生产环境常用）
    public static final String DB_SCHEMA_UPDATE_FALSE = "false";

    // Flowable会对数据库中所有表进行创建操作。
    public static final String DB_SCHEMA_UPDATE_CREATE = "create";

    // 在Flowable启动时创建表，在关闭时删除表（必须手动关闭引擎，才能删除表）。（单元测试常用）
    public static final String DB_SCHEMA_UPDATE_CREATE_DROP = "create-drop";

    // 在Flowable启动时删除原来的旧表，然后在创建新表（不需要手动关闭引擎）。
    public static final String DB_SCHEMA_UPDATE_DROP_CREATE = "drop-create";

    // Flowable会对数据库中所有表进行更新操作。如果表不存在，则自动创建。（开发时常用）
    public static final String DB_SCHEMA_UPDATE_TRUE = "true";

	......
}
```

### 3.3、初始化工作流的 buildProcessEngine()方法

这个方法是`org.flowable.engine.ProcessEngineConfiguration`中的一个抽象方法，主要功能是初始化引擎，获取到工作流的核心 API 接口：`ProcessEngine`。通过该 API，可获取到引擎所有的 service 服务。

#### 3.3.1、buildProcessEngine()方法

buildProcessEngine()有两个子类方法的重写，默认是用 ProcessEngineConfigurationImpl 类继承重写 buildProcessEngine 初始化方法，如下图所示：

<img src="http://lyx-study-notes.oss-cn-hangzhou.aliyuncs.com/img/buildProcessEngine子类重写图.png" style="zoom:80%;" />

该 `buildProcessEngine` 重写方法如下：

```java
@Override
public ProcessEngine buildProcessEngine() {
    // init()方法里面包含各类需要初始化的方法
	init();
	ProcessEngineImpl processEngine = new ProcessEngineImpl(this);

	// trigger build of Flowable 5 Engine
	if (flowable5CompatibilityEnabled && flowable5CompatibilityHandler != null) {
        commandExecutor.execute(new Command<Void>() {
            @Override
            public Void execute(CommandContext commandContext) {
                flowable5CompatibilityHandler.getRawProcessEngine();
                return null;
            }
        });
	}

	postProcessEngineInitialisation();

	return processEngine;
}
```

#### 3.3.2、init()中与数据库连接初始化相关的逻辑

`此标题下的小标签ctrl+鼠标左键可快速会到此处`

```java
/*
 * org.flowable.common.engine.impl.AbstractEngineConfiguration中定义了默认值是否配置关系数据库的标志;
 * 	true，则 AbstractEngineConfiguration#getDatabaseSchemaUpdate() 值将用于确定数据库模式需要发生的事情。
 * 	false，则不会进行验证或架构创建。这意味着之前必须“手动”创建数据库架构，但引擎不会验证架构是否正确,且不会使用 
 * 		AbstractEngineConfiguration#getDatabaseSchemaUpdate()值。
 */
protected boolean usingRelationalDatabase = true;

public void init() {
        ......

        if (usingRelationalDatabase) {
            initDataSource();
        } else {
            initNonRelationalDataSource();
        }
        
        if (usingRelationalDatabase || usingSchemaMgmt) {
            initSchemaManager();
            initSchemaManagementCommand();
        }

        ......
		
        if (usingRelationalDatabase) {
            //初始化sqlSessionFactory方法
            initSqlSessionFactory();
        }

        ......
    }
```

##### 3.3.2.1、查看initDataSource()方法

```java
protected void initDataSource() {
        // 判断数据源dataSource是否存在
        if (dataSource == null) {
            // 判断是否使用JNDI方式连接数据源
            if (dataSourceJndiName != null) {
                try {
                    dataSource = (DataSource) new InitialContext().lookup(dataSourceJndiName);
                } catch (Exception e) {
                    throw new FlowableException("couldn't lookup datasource from " + dataSourceJndiName + ": " + e.getMessage(), e);
                }
            // 使用非JNDI方式且数据库地址不为空，走下面的设置
            } else if (jdbcUrl != null) {
                // jdbc驱动为空或者jdbc连接账户为空
                if ((jdbcDriver == null) || (jdbcUsername == null)) {
                    throw new FlowableException("DataSource or JDBC properties have to be specified in a process engine configuration");
                }

                logger.debug("initializing datasource to db: {}", jdbcUrl);

                if (logger.isInfoEnabled()) {
                    logger.info("Configuring Datasource with following properties (omitted password for security)");
                    logger.info("datasource driver : {}", jdbcDriver);
                    logger.info("datasource url : {}", jdbcUrl);
                    logger.info("datasource user name : {}", jdbcUsername);
                }

                //创建mybatis默认连接池PooledDataSource对象，这里传进来的，就是上面pro.setJdbcDriver("com.mysql.jdbc.Driver")配置的参数，
                //debug到这里，就可以清晰明白，配置类里设置的JdbcDriver、JdbcUrl、JdbcUsername、JdbcPassword等，就是为了用来创建连接池需要用到的；
                PooledDataSource pooledDataSource = new PooledDataSource(this.getClass().getClassLoader(), jdbcDriver, jdbcUrl, jdbcUsername, jdbcPassword);

                if (jdbcMaxActiveConnections > 0) {
                    // 设置最大活跃连接数
                    pooledDataSource.setPoolMaximumActiveConnections(jdbcMaxActiveConnections);
                }
                if (jdbcMaxIdleConnections > 0) {
                    // 设置最大空闲连接数
                    pooledDataSource.setPoolMaximumIdleConnections(jdbcMaxIdleConnections);
                }
                if (jdbcMaxCheckoutTime > 0) {
                    // 最大checkout 时长
                    pooledDataSource.setPoolMaximumCheckoutTime(jdbcMaxCheckoutTime);
                }
                if (jdbcMaxWaitTime > 0) {
                    // 在无法获取连接时，等待的时间
                    pooledDataSource.setPoolTimeToWait(jdbcMaxWaitTime);
                }
                if (jdbcPingEnabled) {
                    //是否允许发送测试SQL语句
                    pooledDataSource.setPoolPingEnabled(true);
                    if (jdbcPingQuery != null) {
                        pooledDataSource.setPoolPingQuery(jdbcPingQuery);
                    }
                    pooledDataSource.setPoolPingConnectionsNotUsedFor(jdbcPingConnectionNotUsedFor);
                }
                if (jdbcDefaultTransactionIsolationLevel > 0) {
                    // 设置事务隔离级别
                    pooledDataSource.setDefaultTransactionIsolationLevel(jdbcDefaultTransactionIsolationLevel);
                }
                dataSource = pooledDataSource;
            }

            if (dataSource instanceof PooledDataSource) {
                // ACT-233: connection pool of Ibatis is not properly
                // initialized if this is not called!
                ((PooledDataSource) dataSource).forceCloseAll();
            }
        }

        // 设置数据库类型
        if (databaseType == null) {
            initDatabaseType();
        }
    }
	
	/* 设置数据库类型的方法 */
    public void initDatabaseType() {
        Connection connection = null;
        try {
            connection = dataSource.getConnection();
            DatabaseMetaData databaseMetaData = connection.getMetaData();
            /*
             * 获取数据库产品名称
             * databaseMetaData.getDatabaseProductName()这个方法在 java.sql 包中的 DatabaseMetData 接口里被定义，
             * 其作用是搜索并获取数据库的名称。
             * 会根据数据库驱动类型进入对应的包下，此处使用的mysql，所以会被 mysql 驱动中的 jdbc 中的 DatabaseMetaData 
             * 实现。
             * 具体实现如下：
             * package com.mysql.jdbc;
             * public class DatabaseMetaData implements java.sql.DatabaseMetaData 
             * 
             * public String getDatabaseProductName() throws SQLException {
			 * 		return "MySQL";
			 * }
             */
            String databaseProductName = databaseMetaData.getDatabaseProductName();
            
            logger.debug("database product name: '{}'", databaseProductName);
            databaseType = databaseTypeMappings.getProperty(databaseProductName);
            if (databaseType == null) {
                throw new FlowableException("couldn't deduct database type from database product name '" + databaseProductName + "'");
            }
            logger.debug("using database type: {}", databaseType);

        } catch (SQLException e) {
            logger.error("Exception while initializing Database connection", e);
        } finally {
            try {
                if (connection != null) {
                    connection.close();
                }
            } catch (SQLException e) {
                logger.error("Exception while closing the Database connection", e);
            }
        }

        // Special care for MSSQL, as it has a hard limit of 2000 params per statement (incl bulk statement).
        // Especially with executions, with 100 as default, this limit is passed.
        if (DATABASE_TYPE_MSSQL.equals(databaseType)) {
            maxNrOfStatementsInBulkInsert = DEFAULT_MAX_NR_OF_STATEMENTS_BULK_INSERT_SQL_SERVER;
        }
    }
```

##### 3.3.2.2、查看initSchemaManager()方法

```java
// 此方法会初始化多个管理器，下列一其中一个举例说明

@Override
public void initSchemaManager() {
    // 初始化公共Db架构管理器
    super.initSchemaManager();

    initProcessSchemaManager(); // 初始化进程架构管理器
    initIdentityLinkSchemaManager(); // 初始化标识链接架构管理器
    initEntityLinkSchemaManager(); // 初始化实体链接架构管理器
    initVariableSchemaManager(); // 初始化变量架构管理器
    initTaskSchemaManager(); // 初始化任务架构管理器
    initJobSchemaManager(); // 初始化作业架构管理器
}

/* ================================ 分割线 ================================ */
//查看super.initSchemaManager()，以下是AbstractEngineConfiguration类中部分代码

package org.flowable.common.engine.impl;
public abstract class AbstractEngineConfiguration {
    ......
    public void initSchemaManager() {
        if (this.commonSchemaManager == null) {
            this.commonSchemaManager = new CommonDbSchemaManager();
        }
    }
    ......
}

/* ================================ 分割线 ================================ */

//继续查看CommonDbSchemaManager()类，以下是类中全部代码
package org.flowable.common.engine.impl.db;

public class CommonDbSchemaManager extends ServiceSqlScriptBasedDbSchemaManager {
    
    private static final String COMMON_VERSION_PROPERTY = "common.schema.version";
    
    private static final String SCHEMA_COMPONENT = "common";
    
    public CommonDbSchemaManager() {
        super(PROPERTY_TABLE, SCHEMA_COMPONENT, null, COMMON_VERSION_PROPERTY);
    }
    
    @Override
    protected String getResourcesRootDirectory() {
        // 此处记录的路径为resources文件的根路径，在下面3.3.2.3中的this.getResourceForDbOperation()方法中会详细讲述
        return "org/flowable/common/db/";
    }
    
}
```

##### 3.3.2.3、查看initSchemaManagementCommand()方法

```java
package org.flowable.engine.impl.cfg;

public abstract class ProcessEngineConfigurationImpl extends ProcessEngineConfiguration implements
        ScriptingEngineAwareEngineConfiguration, HasExpressionManagerEngineConfiguration {
    
	public void initSchemaManagementCommand() {
        if (schemaManagementCmd == null) {
            if (usingRelationalDatabase && databaseSchemaUpdate != null) {
                this.schemaManagementCmd = new SchemaOperationsProcessEngineBuild();
            }
        }
    }
}

/* ================================ 分割线 ================================ */

// 继续进入SchemaOperationsProcessEngineBuild类中
/*
 * 在这个方法当中，将根据不同的 if 判断，执行不同的方法——而这里的不同判断，正是基于五种数据库更新策略来展开的，换句话说，这个方
 * 法，才是真正去判断不同策略，从而根据不同策略来对数据库进行对应操作：
 */
package org.flowable.engine.impl;

/**
 * @author Tom Baeyens
 * @author Joram Barrez
 */
public class SchemaOperationsProcessEngineBuild implements Command<Void> {
    
    private static final Logger LOGGER = LoggerFactory.getLogger(SchemaOperationsProcessEngineBuild.class);

    @Override
    public Void execute(CommandContext commandContext) {
        
        SchemaManager schemaManager = CommandContextUtil.getProcessEngineConfiguration(commandContext).getSchemaManager();
        String databaseSchemaUpdate = CommandContextUtil.getProcessEngineConfiguration().getDatabaseSchemaUpdate();
        
        LOGGER.debug("Executing schema management with setting {}", databaseSchemaUpdate);
        if (ProcessEngineConfiguration.DB_SCHEMA_UPDATE_DROP_CREATE.equals(databaseSchemaUpdate)) {
            try {
                schemaManager.schemaDrop();
            } catch (RuntimeException e) {
                // ignore
            }
        }
        
        if (ProcessEngineConfiguration.DB_SCHEMA_UPDATE_CREATE_DROP.equals(databaseSchemaUpdate)
                || ProcessEngineConfigurationImpl.DB_SCHEMA_UPDATE_DROP_CREATE.equals(databaseSchemaUpdate) 				|| ProcessEngineConfigurationImpl.DB_SCHEMA_UPDATE_CREATE.equals(databaseSchemaUpdate)) {
            // 下面以此方法进行讲解
            schemaManager.schemaCreate();

        } else if (ProcessEngineConfiguration.DB_SCHEMA_UPDATE_FALSE.equals(databaseSchemaUpdate)) {
            schemaManager.schemaCheckVersion();

        } else if (ProcessEngineConfiguration.DB_SCHEMA_UPDATE_TRUE.equals(databaseSchemaUpdate)) {
            schemaManager.schemaUpdate();
        }
        
        return null;
    }
}

/* ================================ 分割线 ================================ */

// 查看schemaManager.schemaCreate()方法
package org.flowable.common.engine.impl.db;

public abstract class ServiceSqlScriptBasedDbSchemaManager extends AbstractSqlScriptBasedDbSchemaManager {
    @Override
    public void schemaCreate() {
        // 判断表是否存在
        if (isUpdateNeeded()) {
            // 获取版本，进行比对
            String dbVersion = getSchemaVersion();
            if (!FlowableVersions.CURRENT_VERSION.equals(dbVersion)) {
                throw new FlowableWrongDbException(FlowableVersions.CURRENT_VERSION, dbVersion);
            }
        } else {
            //创建表的方法，继续进入此方法查看
            internalDbSchemaCreate();
        }
    }
}

/* ================================ 分割线 ================================ */
// 进入internalDbSchemaCreate()方法
protected void internalDbSchemaCreate() {
    //此处会指定操作类型，进入此方法
    executeMandatorySchemaResource("create", schemaComponent);
    if (isHistoryUsed()) {
        executeMandatorySchemaResource("create", schemaComponentHistory);
    }
}

/* ================================ 分割线 ================================ */
// 继续进入executeMandatorySchemaResource()方法
public void executeMandatorySchemaResource(String operation, String component) {
        String databaseType = getDbSqlSession().getDbSqlSessionFactory().getDatabaseType();
    	
   // 这里的 this.getResourceForDbOperation()方法是为了获取 sql 文件所存放的相对路径，并且通过这些sql文件创建这47张表
        executeSchemaResource(operation, component, getResourceForDbOperation(operation, operation, 						component, databaseType), false);
}

/* ================================ 分割线 ================================ */

// 这里我们可以看到获取sql文件路径的方法，通过 CommonDbSchemaManager类中重写的getResourcesRootDirectory()方法获取resources所在本目录，directory是前面传递的create，加上数据库类型等形成的路径,路径下的内容分别是对应各个数据库形成的sql文件，如下图
public String getResourceForDbOperation(String directory, String operation, String component, String 
                                        databaseType) {
        return getResourcesRootDirectory() + directory + "/flowable." + databaseType + "." + operation + "." + component + ".sql";
    }
```

<img src="http://lyx-study-notes.oss-cn-hangzhou.aliyuncs.com/img/各数据库类型的sql语句文件.png" style="zoom: 67%;" />

<img src="http://lyx-study-notes.oss-cn-hangzhou.aliyuncs.com/img/flowable的mysql数据库对应的建表sql文件.png" style="zoom:67%;" />

##### 3.3.2.4、查看执行sql文件代码

```java
public void executeSchemaResource(String operation, String component, String resourceName, boolean 
                                  isOptional) {
    InputStream inputStream = null;
    try {
        // 根据resourceName路径字符串，获取到对应sql文件的输入流inputStream，即读取sql文件
    	inputStream = ReflectUtil.getResourceAsStream(resourceName);
        if (inputStream == null) {
            if (!isOptional) {
            	throw new FlowableException("resource '" + resourceName + "' is not available");
            }
        } else {
            //将得到的输入流inputStream传入该方法
        	executeSchemaResource(operation, component, resourceName, inputStream);
        }
    } finally {
    	IoUtil.closeSilently(inputStream);
    }
}
```

执行sql文件的核心方法如下：

```java
protected void executeSchemaResource(String operation, String component, String resourceName, InputStream inputStream) {
        LOGGER.info("performing {} on {} with resource {}", operation, component, resourceName);
        // sql语句字符串
    	String sqlStatement = null;
        String exceptionSqlStatement = null;
        DbSqlSession dbSqlSession = getDbSqlSession();
        try {
            // 1.获取数据库连接
            Connection connection = dbSqlSession.getSqlSession().getConnection();
            Exception exception = null;
            // 2.分行读取“org/flowable/common/db/create/flowable.mysql.create.common.sql”目录下的sql文件
            byte[] bytes = IoUtil.readInputStream(inputStream, resourceName);
            // 3.将sql文件里的数据分行转换成字符串，换行的地方，可以看到字符串用转义符“\n”来代替
            String ddlStatements = new String(bytes);

            // Special DDL handling for certain databases
            try {
                if (dbSqlSession.getDbSqlSessionFactory().isMysql()) {
                    DatabaseMetaData databaseMetaData = connection.getMetaData();
                    int majorVersion = databaseMetaData.getDatabaseMajorVersion();
                    int minorVersion = databaseMetaData.getDatabaseMinorVersion();
                    LOGGER.info("Found MySQL: majorVersion={} minorVersion={}", majorVersion, minorVersion);

                    // Special care for MySQL < 5.6
                    /* 若数据库类型是在mysql 5.6版本以下，需要做一些替换，因为低于5.6版本的MySQL是不支持变体时间戳或毫秒级
                     * 的日期，故而需要在这里对sql语句的字符串做替换。（注意，这里的majorVersion代表主版本，
                     * minorVersion代表主版本下的小版本）
                     */
                    if (majorVersion <= 5 && minorVersion < 6) {
                        ddlStatements = updateDdlForMySqlVersionLowerThan56(ddlStatements);
                    }
                }
            } catch (Exception e) {
                LOGGER.info("Could not get database metadata", e);
            }
			
            // 4.以字符流形式读取字符串数据
            BufferedReader reader = new BufferedReader(new StringReader(ddlStatements));
            // 5.根据字符串中的转义符“\n”分行读取
            String line = readNextTrimmedLine(reader);
            boolean inOraclePlsqlBlock = false;
            // 6.循环每一行
            while (line != null) {
                if (line.startsWith("# ")) {
                    ......
				// 7.若下一行line还有数据，证明还没有全部读取，仍可执行读取
                } else if (line.length() > 0) {

                    if (dbSqlSession.getDbSqlSessionFactory().isOracle() && line.startsWith("begin")) {
                        inOraclePlsqlBlock = true;
                        sqlStatement = addSqlStatementPiece(sqlStatement, line);
                        
		// 8.在没有拼接够一个完整建表语句时，！line.endsWith(";")会为true，即一直循环进行拼接，当遇到";"就跳出该if语句。
                    } else if ((line.endsWith(";") && !inOraclePlsqlBlock) || (line.startsWith("/") && inOraclePlsqlBlock)) {
		/* 
		 * 9.循环拼接中若遇到符号";"，就意味着，已经拼接形成一个完整的sql建表语句，例如
		 * create table ACT_GE_PROPERTY ( 
		 * 		NAME_ varchar(64), 
		 * 		VALUE_ varchar(300),
		 * 		REV_ integer, 
		 * 		primary key (NAME_) 
		 * ) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE utf8_bin
		 * 这样，就可以先通过代码来将该建表语句执行到数据库中，实现如下：
		 */
                        if (inOraclePlsqlBlock) {
                            inOraclePlsqlBlock = false;
                        } else {
                            sqlStatement = addSqlStatementPiece(sqlStatement, line.substring(0, line.length() - 1));
                        }
						// 10.将建表语句字符串包装成Statement对象
                        Statement jdbcStatement = connection.createStatement();
                        try {
                            // no logging needed as the connection will log it
                            LOGGER.debug("SQL: {}", sqlStatement);
                            // 11.最后，执行建表语句到数据库中
                            jdbcStatement.execute(sqlStatement);
                            jdbcStatement.close();
                            
                        } catch (Exception e) {
                            if (exception == null) {
                                exception = e;
                                exceptionSqlStatement = sqlStatement;
                            }
                            LOGGER.error("problem during schema {}, statement {}", operation, sqlStatement, e);
                            
                        } finally {
                		 	/*
                		 	 * 12.到这一步，意味着上一条sql建表语句已经执行结束，若没有出现错误话，这时已经证明第一个数据库
                		 	 * 表结构已经创建完成，可以开始拼接下一条建表语句;
                		 	 */
                            sqlStatement = null;
                        }
                        
                    } else {
                        sqlStatement = addSqlStatementPiece(sqlStatement, line);
                    }
                }

                line = readNextTrimmedLine(reader);
            }

            if (exception != null) {
                throw exception;
            }

            LOGGER.debug("flowable db schema {} for component {} successful", operation, component);

        } catch (Exception e) {
            throw new FlowableException("couldn't " + operation + " db schema: " + exceptionSqlStatement, e);
        }
    }
```

以上步骤可以归纳下：

1. **jdbc 连接 mysql 数据库；**
2. **分行读取 resourceName="org/flowable/common/db/create/flowable.mysql.create.common.sql"目录底下的 sql 文件数据；**
3. **将整个 sql 文件数据分行转换成字符串 ddlStatements，有换行的地方，用转义符“\n”来代替；**
4. **以 BufferedReader 字符流形式读取字符串 ddlStatements 数据；**
5. **循环字符流里的每一行，拼接成 sqlStatement 字符串，若读取到该行结尾有“；”符号，意味着已经拼接成一个完整的 create 建表语句，这时，跳出该次拼接，直接包装成成 Statement 对象；值得注意一点是，Statement 是 Java 执行数据库操作的一个重要接口，用于在已经建立数据库连接的基础上，向数据库发送要执行的 SQL 语句。Statement 对象是用于执行不带参数的简单 SQL 语句，例如本次的 create 建表语句。**
6. **最后，执行 jdbcStatement.execute(sqlStatement)，将 create 建表语句执行进数据库中；**
7. **生成对应的数据库表；**

根据 debug 过程截图，可以更为直观地看到，这里获取到的 ddlStatements 字符串，涵盖了 sql 文件里的所有 sql 语句，同时，每一个完整的 creat 建表语句，都是以";"结尾的：

<img src="http://lyx-study-notes.oss-cn-hangzhou.aliyuncs.com/img/ddlStatements读到的所有的sql语句.png" style="zoom: 50%;" />

每次执行到"；"时，都会得到一个完整的 create 建表语句：

<img src="http://lyx-study-notes.oss-cn-hangzhou.aliyuncs.com/img/读到分号后获取到的完整的一条sql语句.png" style="zoom: 67%;" />

执行完一个建表语句，就会在数据库里同步生成一张数据库表，如上图执行的是 ACT_GE_PROPERTY 表，数据库里便生成了这张表：

<img src="http://lyx-study-notes.oss-cn-hangzhou.aliyuncs.com/img/对应数据库中生成的表.png" style="zoom: 67%;" />

控制台可以看到详细的执行了哪些sql文件

<img src="http://lyx-study-notes.oss-cn-hangzhou.aliyuncs.com/img/Flowable初始化结果.png" style="zoom:80%;" />

## 四、与activiti的区别

1. flowable已经支持所有的历史数据使用mongdb存储，activiti没有。
2. flowable支持事务子流程，activiti没有。
3. flowable支持多实例加签、减签，activiti没有。
4. flowable支持httpTask等新的类型节点，activiti没有。
5. flowable支持在流程中动态添加任务节点，activiti没有。
6. flowable支持历史任务数据通过消息中间件发送，activiti没有。
7. flowable支持java11,activiti没有。
8. flowable支持动态脚本，,activiti没有。
9. flowable支持条件表达式中自定义juel函数，activiti没有。
10. flowable支持cmmn规范，activiti没有。
11. flowable修复了dmn规范设计器，activit用的dmn设计器还是旧的框架，bug太多。
12. flowable屏蔽了pvm，activiti6也屏蔽了pvm（因为6版本官方提供了加签功能，发现pvm设计的过于臃肿，索性直接移除，这样加签实现起来更简洁、事实确实如此，如果需要获取节点、连线等信息可以使用bpmnmodel替代）。
13. flowable与activiti提供了新的事务监听器。activiti5版本只有事件监听器、任务监听器、执行监听器。
14. flowable对activiti的代码大量的进行了重构。
15. activiti以及flowable支持的数据库有h2、hsql、mysql、oracle、postgres、mssql、db2。其他数据库不支持的。
16. flowable支持jms、rabbitmq、mongodb方式处理历史数据，activiti没有。
