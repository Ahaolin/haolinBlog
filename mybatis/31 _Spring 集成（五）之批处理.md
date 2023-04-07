# 精尽 MyBatis 源码解析 —— Spring 集成（五）之批处理

## 1. 概述

本文我们就来看看，Spring 和 MyBatis 的**批处理**是如何集成。它在 `batch` 包下实现，基于 [Spring Batch](https://spring.io/projects/spring-batch) 框架实现，整体类图如下：[![类图](http://static.iocoder.cn/images/MyBatis/2020_06_13/01.png)](http://static.iocoder.cn/images/MyBatis/2020_06_13/01.png)

- 读操作：MyBatisCursorItemReader、MyBatisPagingItemReader 。
- 写操作：MyBatisBatchItemWriter 。

## 2. MyBatisPagingItemReader

`org.mybatis.spring.batch.MyBatisPagingItemReader` ，继承 `org.springframework.batch.item.database.AbstractPagingItemReader` 抽象类，基于分页的 MyBatis 的读取器。

### 2.1 示例

```java
// SpringBatchTest.java

@Test
@Transactional
void shouldDuplicateSalaryOfAllEmployees() throws Exception {
    // <x> 批量读取 Employee 数组
    List<Employee> employees = new ArrayList<>();
    Employee employee = pagingNoNestedItemReader.read();
    while (employee != null) {
        employee.setSalary(employee.getSalary() * 2);
        employees.add(employee);
        employee = pagingNoNestedItemReader.read();
    }

    // 批量写入
    //noinspection Duplicates
    writer.write(employees);

    assertThat((Integer) session.selectOne("checkSalarySum")).isEqualTo(20000);
    assertThat((Integer) session.selectOne("checkEmployeeCount")).isEqualTo(employees.size());
}
```

- `<x>` 处，不断读取，直到为空。

### 2.2 构造方法

```java
// MyBatisPagingItemReader.java

/**
 * SqlSessionFactory 对象
 */
private SqlSessionFactory sqlSessionFactory;
/**
 * SqlSessionTemplate 对象
 */
private SqlSessionTemplate sqlSessionTemplate;
/**
 * 查询编号
 */
private String queryId;
/**
 * 参数值的映射
 */
private Map<String, Object> parameterValues;

public MyBatisPagingItemReader() {
    setName(getShortName(MyBatisPagingItemReader.class));
}

// ... 省略 setting 方法
```

### 2.3 afterPropertiesSet

```java
// MyBatisPagingItemReader.java

@Override
public void afterPropertiesSet() throws Exception {
    // 父类的处理
    super.afterPropertiesSet();
    notNull(sqlSessionFactory, "A SqlSessionFactory is required.");
    // 创建 SqlSessionTemplate 对象
    sqlSessionTemplate = new SqlSessionTemplate(sqlSessionFactory, ExecutorType.BATCH);
    notNull(queryId, "A queryId is required.");
}
```

- 主要目的，是创建 SqlSessionTemplate 对象。

### 2.4 doReadPage

`#doReadPage()` 方法，执行每一次分页的读取。代码如下：

```java
// MyBatisPagingItemReader.java

@Override
protected void doReadPage() {
    // <1> 创建 parameters 参数
    Map<String, Object> parameters = new HashMap<>();
    // <1.1> 设置原有参数
    if (parameterValues != null) {
        parameters.putAll(parameterValues);
    }
    // <1.2> 设置分页参数
    parameters.put("_page", getPage());
    parameters.put("_pagesize", getPageSize());
    parameters.put("_skiprows", getPage() * getPageSize());
    // <2> 清空目前的 results 结果
    if (results == null) {
        results = new CopyOnWriteArrayList<>();
    } else {
        results.clear();
    }
    // <3> 查询结果
    results.addAll(sqlSessionTemplate.selectList(queryId, parameters));
}
```

- `<1>`处，创建`parameters`参数。
  - `<1.1>` 处，设置原有参数。
  - `<1.2>` 处，设置分页参数。
- `<2>` 处，创建或清空当前的 `results` ，保证是空的数组。使用 CopyOnWriteArrayList 的原因是，可能存在并发读取的问题。
- 【重要】 `<3>` 处，调用 `SqlSessionTemplate#selectList(queryId, parameters)` 方法，执行查询列表。查询后，将结果添加到 `results` 中。

### 2.5 doJumpToPage

```java
// MyBatisPagingItemReader.java

@Override
protected void doJumpToPage(int itemIndex) {
    // Not Implemented
}
```

## 3. MyBatisCursorItemReader

`org.mybatis.spring.batch.MyBatisCursorItemReader` ，继承 `org.springframework.batch.item.support.AbstractItemCountingItemStreamItemReader` 抽象类，实现 InitializingBean 接口，基于 **Cursor** 的 MyBatis 的读取器。

### 3.1 示例

```java
// SpringBatchTest.java

@Test
@Transactional
void checkCursorReadingWithoutNestedInResultMap() throws Exception {
    // 打开 Cursor
    cursorNoNestedItemReader.doOpen();
    try {
        // Employee 数组
        List<Employee> employees = new ArrayList<>();
        // <x> 循环读取，写入到 Employee 数组中
        Employee employee = cursorNoNestedItemReader.read();
        while (employee != null) {
            employee.setSalary(employee.getSalary() * 2);
            employees.add(employee);
            employee = cursorNoNestedItemReader.read();
        }

        // 批量写入
        writer.write(employees);

        assertThat((Integer) session.selectOne("checkSalarySum")).isEqualTo(20000);
        assertThat((Integer) session.selectOne("checkEmployeeCount")).isEqualTo(employees.size());
    } finally {
        // 关闭 Cursor
        cursorNoNestedItemReader.doClose();
    }
}
```

- `<x>` 处，不断读取，直到为空。

### 3.2 构造方法

```java
// MyBatisCursorItemReader.java

/**
 * SqlSessionFactory 对象
 */
private SqlSessionFactory sqlSessionFactory;
/**
 * SqlSession 对象
 */
private SqlSession sqlSession;

/**
 * 查询编号
 */
private String queryId;
/**
 * 参数值的映射
 */
private Map<String, Object> parameterValues;

/**
 * Cursor 对象
 */
private Cursor<T> cursor;
/**
 * {@link #cursor} 的迭代器
 */
private Iterator<T> cursorIterator;

public MyBatisCursorItemReader() {
    setName(getShortName(MyBatisCursorItemReader.class));
}

// ... 省略 setting 方法
```

### 3.3 afterPropertiesSet

```java
// MyBatisCursorItemReader.java

@Override
public void afterPropertiesSet() throws Exception {
    notNull(sqlSessionFactory, "A SqlSessionFactory is required.");
    notNull(queryId, "A queryId is required.");
}
```

### 3.4 doOpen

`#doOpen()` 方法，打开 Cursor 。代码如下：

```java
// MyBatisCursorItemReader.java

@Override
protected void doOpen() throws Exception {
    // <1> 创建 parameters 参数
    Map<String, Object> parameters = new HashMap<>();
    if (parameterValues != null) {
        parameters.putAll(parameterValues);
    }

    // <2> 创建 SqlSession 对象
    sqlSession = sqlSessionFactory.openSession(ExecutorType.SIMPLE);

    // <3.1> 查询，返回 Cursor 对象
    cursor = sqlSession.selectCursor(queryId, parameters);
    // <3.2> 获得 cursor 的迭代器
    cursorIterator = cursor.iterator();
}
```

- `<1>` 处，创建 parameters 参数。
- `<2>` 处，调用 `SqlSessionFactory#openSession(ExecutorType.SIMPLE)` 方法，创建简单执行器的 SqlSession 对象。
- `<3.1>`处，调用`SqlSession#selectCursor(queryId, parameters)`方法，返回 Cursor 对象。
  - `<3.2>` 处，调用 `Cursor#iterator()` 方法，获得迭代器。

### 3.5 doRead

`#doRead()` 方法，读取下一条数据。代码如下：

```java
// MyBatisCursorItemReader.java

@Override
protected T doRead() throws Exception {
    // 置空 next
    T next = null;
    // 读取下一条
    if (cursorIterator.hasNext()) {
        next = cursorIterator.next();
    }
    // 返回
    return next;
}
```

### 3.6 doClose

`#doClose()` 方法，关闭 Cursor 和 SqlSession 对象。代码如下：

```java
// MyBatisCursorItemReader.java

@Override
protected void doClose() throws Exception {
    // 关闭 cursor 对象
    if (cursor != null) {
        cursor.close();
    }
    // 关闭 sqlSession 对象
    if (sqlSession != null) {
        sqlSession.close();
    }
    // 置空 cursorIterator
    cursorIterator = null;
}
```

## 4. MyBatisBatchItemWriter

`org.mybatis.spring.batch.MyBatisBatchItemWriter` ，实现 `org.springframework.batch.item.ItemWriter`、InitializingBean 接口，MyBatis 批量写入器。

### 4.1 示例

```java
// SpringBatchTest.java

@Test
@Transactional
void shouldDuplicateSalaryOfAllEmployees() throws Exception {
    // 批量读取 Employee 数组
    List<Employee> employees = new ArrayList<>();
    Employee employee = pagingNoNestedItemReader.read();
    while (employee != null) {
        employee.setSalary(employee.getSalary() * 2);
        employees.add(employee);
        employee = pagingNoNestedItemReader.read();
    }

    // <x> 批量写入
    writer.write(employees);

    assertThat((Integer) session.selectOne("checkSalarySum")).isEqualTo(20000);
    assertThat((Integer) session.selectOne("checkEmployeeCount")).isEqualTo(employees.size());
}
```

- `<x>` 处，执行**一次**批量写入到数据库中。

### 4.2 构造方法

```java
// SpringBatchTest.java

/**
 * SqlSessionTemplate 对象
 */
private SqlSessionTemplate sqlSessionTemplate;

/**
 * 语句编号
 */
private String statementId;

/**
 * 是否校验
 */
private boolean assertUpdates = true;

/**
 * 参数转换器
 */
private Converter<T, ?> itemToParameterConverter = new PassThroughConverter<>();

// ... 省略 setting 方法
```

### 4.3 PassThroughConverter

PassThroughConverter ，是 MyBatisBatchItemWriter 的内部静态类，实现 Converter 接口，直接返回自身。代码如下：

```java
// MyBatisBatchItemWriter.java

private static class PassThroughConverter<T> implements Converter<T, T> {

    @Override
    public T convert(T source) {
        return source;
    }

}
```

- 这是一个默认的 Converter 实现类。我们可以自定义 Converter 实现类，设置到 MyBatisBatchItemWriter 中。那么具体什么用呢？答案见 [「4.5 write」](http://svip.iocoder.cn/MyBatis/Spring-Integration-5/#) 。

### 4.4 afterPropertiesSet

```java
// MyBatisBatchItemWriter

@Override
public void afterPropertiesSet() {
    notNull(sqlSessionTemplate, "A SqlSessionFactory or a SqlSessionTemplate is required.");
    isTrue(ExecutorType.BATCH == sqlSessionTemplate.getExecutorType(), "SqlSessionTemplate's executor type must be BATCH");
    notNull(statementId, "A statementId is required.");
    notNull(itemToParameterConverter, "A itemToParameterConverter is required.");
}
```

### 4.5 write

`#write(List<? extends T> items)` 方法，将传入的 `items` 数组，执行一次批量操作，仅执行一次批量操作。代码如下：

```java
// MyBatisBatchItemWriter.java

@Override
public void write(final List<? extends T> items) {
    if (!items.isEmpty()) {
        LOGGER.debug(() -> "Executing batch with " + items.size() + " items.");

        // <1> 遍历 items 数组，提交到 sqlSessionTemplate 中
        for (T item : items) {
            sqlSessionTemplate.update(statementId, itemToParameterConverter.convert(item));
        }
        // <2> 执行一次批量操作
        List<BatchResult> results = sqlSessionTemplate.flushStatements();

        if (assertUpdates) {
            // 如果有多个返回结果，抛出 InvalidDataAccessResourceUsageException 异常
            if (results.size() != 1) {
                throw new InvalidDataAccessResourceUsageException("Batch execution returned invalid results. " +
                        "Expected 1 but number of BatchResult objects returned was " + results.size());
            }

            // <3> 遍历执行结果，若存在未更新的情况，则抛出 EmptyResultDataAccessException 异常
            int[] updateCounts = results.get(0).getUpdateCounts();
            for (int i = 0; i < updateCounts.length; i++) {
                int value = updateCounts[i];
                if (value == 0) {
                    throw new EmptyResultDataAccessException("Item " + i + " of " + updateCounts.length
                            + " did not update any rows: [" + items.get(i) + "]", 1);
                }
            }
        }
    }
}
```

- `<1>`处，遍历`items`数组，提交到`sqlSessionTemplate`中。
  - `<2>` 处，执行一次批量操作。
- `<3>` 处，遍历执行结果，若存在未更新的情况，则抛出 EmptyResultDataAccessException 异常。

## 5. Builder 类

在 `batch/builder` 包下，读取器和写入器都有 Builder 类，胖友自己去瞅瞅，灰常简单。

## 666. 彩蛋

> 在 `org.mybatis.spring.batch.SpringBatchTest` 单元测试类中，可以直接进行调试。
>
> 😈 调试起来把，胖友。

一篇靠体力活的文章，嘿嘿嘿。