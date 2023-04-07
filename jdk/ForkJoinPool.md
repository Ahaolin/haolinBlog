## ForkJoinPool解析

> 共享队列： 指的是该队列 是由外界提交的任务 所在的队列,索引地址为偶数。

## 一.简介

ThreadPoolExecutor应该都很了解了，就是一个基本的存储线程的线程池，需要执行任务的时候就从线程池中拿一个线程来执行。而ForkJoinPool则不仅仅是这么简单，同样也不是ThreadPoolExecutor的代替品，这种线程池是为了实现“分治法”这一思想而创建的，通过把大任务拆分成小任务，然后再把小任务的结果汇总起来就是最终的结果，和MapReduce的思想很类似
![](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/aHR0cHM6Ly9tbWJpei5xcGljLmNuL21tYml6L1lVZzZFRGtaNVRoVmdkbTFpYWFJeXRnMUdnVUY0dzhxblliUVNwUmZtVE1JeWFqVWdhWmJRa2c5WjZndGxyNnJuR1UwODM3YW5pYkJuQnNRR245d21wRVEvNjQw)

最核心的思想可以这样描述：

```apl
if(任务很小）{
    直接计算得到结果
}else{
    分拆成N个子任务
    调用子任务的fork()进行计算
    调用子任务的join()合并计算结果
}
```



继承体系如下：

<img src="https://img2018.cnblogs.com/blog/1648938/201911/1648938-20191109011139355-1945861823.png" style="zoom: 67%;" />

ForkJoinPool和ThreadPoolExecutor都是继承自AbstractExecutorService抽象类，所以它和ThreadPoolExecutor的使用几乎没有多少区别，除了任务变成了ForkJoinTask以外。

- `fork()`
  - fork()方法类似于线程的Thread.start()方法，但是它不是真的启动一个线程，而是将任务放入到工作队列中。
- `join()`
  - join()方法类似于线程的Thread.join()方法，但是它不是简单地阻塞线程，而是利用工作线程运行其它任务。当一个工作线程中调用了join()方法，它将处理其它任务，直到注意到目标子任务已经完成了。

###  1. 使用方式

#### 1.1 RecursiveAction  

```java
/** 
 *  
 *RecursiveAction代表没有返回值的任务。 
 */  
class PrintTask  extends RecursiveAction {    
     private  static  final  long serialVersionUID = -4335005084191316849L;

     // 每个"小任务"最多只打印20个数    
     private  static  final  int MAX =  20;    
    
     private  int start;    
     private  int end;    
    
    PrintTask( int start,  int end) {    
         this.start = start;    
         this.end = end;    
    }    
    
    @Override    
     protected  void compute() {    
         // 当end-start的值小于MAX时候，开始打印    
         if ((end - start) < MAX) {    
             for ( int i = start; i < end; i++) {    
                System.out.println(Thread.currentThread().getName() +  "的i值:"  + i);    
            }    
        }  else {    
             // 将大任务分解成两个小任务    
             int middle = (start + end) /  2;    
            PrintTask left =  new PrintTask(start, middle);    
            PrintTask right =  new PrintTask(middle, end);    
             // 并行执行两个小任务    
            left.fork();    
            right.fork();    
        }    
    }    
}    
    
public  class ForkJoinPoolTest {    
     /**  
     * @param args  
     * @throws Exception  
     */    
     public  static  void main( String[] args)  throws Exception {    
         // 创建包含Runtime.getRuntime().availableProcessors()返回值作为个数的并行线程的ForkJoinPool    
        ForkJoinPool forkJoinPool =  new ForkJoinPool();    
         // 提交可分解的PrintTask任务    
        forkJoinPool.submit( new PrintTask( 0,  1000));    
        
        forkJoinPool.awaitTermination( 2, TimeUnit.SECONDS); //阻塞当前线程直到 ForkJoinPool 中所有的任务都执行结束    
         // 关闭线程池    
        forkJoinPool.shutdown();    
    }
}
```

#### 1.2 RecursiveTask

```java
/**
 * 
 * RecursiveTask有返回值类型
 */
class SumTask  extends RecursiveTask<Integer> {
    private  static  final  long serialVersionUID = -7132529934386580711L;
    // 每个"小任务"最多只打印70个数
    private  static  final  int MAX =  70;
    private  int arr[];
    private  int start;
    private  int end;

    SumTask( int arr[],  int start,  int end) {
        this.arr = arr;
        this.start = start;
        this.end = end;
    }

    @Override
    protected Integer compute() {
        int sum =  0;
        // 当end-start的值小于MAX时候，开始打印
        if ((end - start) < MAX) {
            for ( int i = start; i < end; i++)  sum += arr[i];
            return sum;
        }  else {
            System.err.println( "=====任务分解======");
            // 将大任务分解成两个小任务
            int middle = (start + end) /  2;
            SumTask left =  new SumTask(arr, start, middle);
            SumTask right =  new SumTask(arr, middle, end);
            // 并行执行两个小任务
            left.fork();
            right.fork();
            // 把两个小任务累加的结果合并起来
            return left.join() + right.join();
        }
    }
}

public  class ForkJoinPoolTest2 {
    public  static  void main( String[] args)  throws Exception {
        int arr[] =  new  int[ 1000];
        Random random =  new Random();
        int total =  0;
        // 初始化100个数字元素
        for ( int i =  0; i < arr.length; i++) {
            int temp = random.nextInt( 100);
            // 对数组元素赋值,并将数组元素的值添加到total总和中
            total += (arr[i] = temp);
        }
        System.out.println( "初始化时的总和=" + total);
        // 创建包含Runtime.getRuntime().availableProcessors()返回值作为个数的并行线程的ForkJoinPool
        ForkJoinPool forkJoinPool =  new ForkJoinPool();
        // 提交可分解的PrintTask任务
        Future<Integer> future = forkJoinPool.submit( new SumTask(arr,  0, arr.length));
        System.out.println( "计算出来的总和=" + future.get());
        // 关闭线程池
        forkJoinPool.shutdown();
    }
}
```

#### 1.3 错误使用方式

[Java的Fork/Join任务，你写对了吗？ - 廖雪峰的官方网站 (liaoxuefeng.com)](https://www.liaoxuefeng.com/article/1146802219354112)

#### 1.4 阻塞Task

[Java ForkJoinPool.managedBlock方法代码示例 - 纯净天空 (vimsky.com)](https://vimsky.com/examples/detail/java-method-java.util.concurrent.ForkJoinPool.managedBlock.html)

### 2. 数据结构

![](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/20181111222804684.png)

### 3. 关键调用图

<img src="http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/20181111222837182.png" style="zoom:200%;" />

### 4. **内部原理**

#### 4.1 **work-stealing（工作窃取）**算法

ForkJoinPool 的另一个特性是它使用了**work-stealing（工作窃取）**算法：

线程池内的所有工作线程都尝试找到并执行已经提交的任务，或者是被其他活动任务创建的子任务（如果不存在就阻塞等待）。**这种特性使得 ForkJoinPool 在运行多个可以产生子任务的任务，或者是提交的许多小任务时效率更高。尤其是构建异步模型的 ForkJoinPool 时，对不需要合并（join）的事件类型任务也非常适用**。

在 ForkJoinPool 中，线程池中每个工作线程（ForkJoinWorkerThread）都对应一个任务队列（WorkQueue），工作线程优先处理来自自身队列的任务（LIFO或FIFO顺序，参数 mode 决定），然后以FIFO的顺序随机窃取其他队列中的任务。


![](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/aHR0cHM6Ly9tbWJpei5xcGljLmNuL21tYml6L1lVZzZFRGtaNVRoVmdkbTFpYWFJeXRnMUdnVUY0dzhxbkFvakxDSWxTbDRXVms5aWJWREwyRkkyYnh5VmljRGdDdWV2YU9GcWZxY2hnbWxnTzkyOWppY3hYUS82NDA)

#### 4.2 **ForkJoinPool中的任务**

ForkJoinPool 中的任务分为两种：

- **一种是本地提交的任务（Submission task，如 execute、submit 提交的任务）**；

- **另外一种是 fork 出的子任务（Worker task）**。

两种任务都会存放在 WorkQueue 数组中，但是这两种任务并不会混合在同一个队列里，ForkJoinPool 内部使用了一种随机哈希算法（有点类似 ConcurrentHashMap 的桶随机算法）将工作队列与对应的工作线程关联起来，**Submission 任务存放在 WorkQueue 数组的偶数索引位置，Worker 任务存放在奇数索引位**。

实质上，Submission 与 Worker 一样，只不过它被限制只能执行它们提交的本地任务，在后面的源码解析中，我们统一称之为“Worker”。


任务的分布情况如下图：

<img src="http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/aHR0cHM6Ly9tbWJpei5xcGljLmNuL21tYml6L1lVZzZFRGtaNVRoVmdkbTFpYWFJeXRnMUdnVUY0dzhxbjJ5c25sTEc2Y2dSRkhUa0FpYmljTVh6cVhmMzNjZjZZOXBpYmVHN2JmQXNjQWxLMWMzYWhuSjZuQS82NDA"  />

## 二.ForkJoinPooly源码解析

### 1. 总体介绍

在java中运行ForkJoinPool，经过对源码的分析，实际上，需要4个类来配合运行。这四个类分别是：

- `ForkJoinPool` 这是线程池的核心类，也是提供方法供我们使用的入口类。**基本上forkJoin的核心操作及线程池的管理方法都由这个类提供**。后面将详细介绍。
- `ForkJoinPool.WorkQueue` 这是ForkJoinPool类的内部类。也是线程池核心的组成部分。F**orkJoinPool线程池将由WorkQueue数组组成**。为了进一步提高性能，与ThreadPoolExecutor不一样的是，这没有采用外部传入的任务队列，而是作者自己实现了一个**阻塞**队列。
- `ForkJoinWorkerThread` 线程池中运行的thread也是作者重新定义的。这个类的目的是在于将**外部的各种形式的task都转换为统一的ForkJoinTask**格式。
- `ForkJoinTask`  这是ForkJoinPool支持运行的task抽象类，我们一般使用其子类如`RecursiveTask`或者`RecursiveAction`。这个类提供了任务拆分和结果合并的方法。

### 2. 常量和变量

#### 2.1 常量

##### 2.1.1 配置常量

```java
//定义IDLE的超时时间 默认位2秒
private static final long IDLE_TIMEOUT = 2000L * 1000L * 1000L; // 2sec

//idea的容忍度 20ms
private static final long TIMEOUT_SLOP = 20L * 1000L * 1000L;  // 20ms

//静态初始化commonMaxSpares的初始值。这个值远远超出了正常的要求，但也远远低于MAX_CAP和典型的OS线程限制，因此允许jvm在耗尽所需资源之前捕获异常以避免滥用。
private static final int DEFAULT_COMMON_MAX_SPARES = 256;

 //阻塞之前等待自旋的次数，这个默认值是0，以降低对CPU的使用。如果大于零，自旋值必须是2的幂次方，至少是4。值2048导致旋转的时间只有典型上下文切换时间的一小部分。
private static final int SPINS  = 0;

//这个是产生随机性的魔数，用于scan的时候进行计算
private static final int SEED_INCREMENT = 0x9e3779b9;
```

##### 2.1.2 二进制位运算的范围常量

这些常量实际上是用于后续进行二进制位运算的范围常量。
```java
//界限定义
static final int SMASK        = 0xffff;        // short bits == max index
static final int MAX_CAP      = 0x7fff;        // max #workers - 1
static final int EVENMASK     = 0xfffe;        // even short bits
static final int SQMASK       = 0x007e;        // max 64 (even) slots

// Masks and units for WorkQueue.scanState and ctl sp subfield
static final int SCANNING     = 1;             // false when running tasks
static final int INACTIVE     = 1 << 31;       // must be negative
static final int SS_SEQ       = 1 << 16;       // version count

// Mode bits for ForkJoinPool.config and WorkQueue.config
static final int MODE_MASK    = 0xffff << 16;  // top half of int
static final int LIFO_QUEUE   = 0;
static final int FIFO_QUEUE   = 1 << 16;
static final int SHARED_QUEUE = 1 << 31;       // must be negative
```

上述这些界限用二进制的方式表示如下：

![](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/3237432-7e4c4807f763e908.png)

##### 2.1.3 ctl控制常量

下面是进行ctl字段控制的一些字段和掩码：

```java
     
//ctl的计算字段和掩码 
// Lower and upper word masks
private static final long SP_MASK    = 0xffffffffL;
private static final long UC_MASK    = ~SP_MASK;

// Active counts
private static final int  AC_SHIFT   = 48;
private static final long AC_UNIT    = 0x0001L << AC_SHIFT;
private static final long AC_MASK    = 0xffffL << AC_SHIFT;

// Total counts
private static final int  TC_SHIFT   = 32;
private static final long TC_UNIT    = 0x0001L << TC_SHIFT;
private static final long TC_MASK    = 0xffffL << TC_SHIFT;
private static final long ADD_WORKER = 0x0001L << (TC_SHIFT + 15); // sign

```

![img](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/3237432-e0cfddb6a774113e.png)

##### 2.1.4 线程场运行状态常量

```java
// runState bits: SHUTDOWN must be negative, others arbitrary powers of two
private static final int  RSLOCK     = 1;
private static final int  RSIGNAL    = 1 << 1;
private static final int  STARTED    = 1 << 2;
private static final int  STOP       = 1 << 29;
private static final int  TERMINATED = 1 << 30;
private static final int  SHUTDOWN   = 1 << 31;
```

![img](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/3237432-e414e27cae0a3ae6.png)

#### 2.2 成员变量

```java
volatile long ctl;                   // 控制中心：非常重要，看下图解析
volatile int runState;               // 负数是shutdown，其余都是2的次方
final int config;                    // 配置：二进制的低16位代表 并行度（parallelism），
																				//高16位：mode可选FIFO_QUEUE（1 << 16）和LIFO_QUEUE（1 << 31），默认是LIFO_QUEUE

int indexSeed;                       // 生成worker的queue索引(随机种子，通过前面的魔数来实现) indexSeed += SEED_INCREMENT

volatile WorkQueue[] workQueues;     // 组成workQueue的数组
final ForkJoinWorkerThreadFactory factory; // 产生线程的工厂方法
final UncaughtExceptionHandler ueh;  // per-worker UEH 每个worker出现异常之后的处理办法，类似于前面ThreadPoolExecutor的拒绝策略

final String workerNamePrefix;       // to create worker name string 创建线程名称的前缀
volatile AtomicLong stealCounter;    // also used as sync monitor 用于监控steal的计数器
```

##### 2.2.1 ctl

`ctl`是``ForkJoinPool`中最重要的控制字段，将下面信息按16bit为一组封装在一个long中。

- AC: 活动的worker数量；
- TC: 总共的worker数量；
- SS: WorkQueue状态，第一位表示active的还是inactive，其余十五位表示版本号(对付ABA)；
- ID:  这里保存了一个WorkQueue在WorkQueue[]的下标，和其他worker通过字段stackPred组成一个TreiberStack。后文讲的栈顶，指这里下标所在的WorkQueue。

```apl
AC和TC初始化时取的是`parallelism`负数，后续代码可以直接判断正负，为负代表还没有达到目标数量。另外ctl低32位有个技巧可以直接用sp=(int)ctl取得，为负代表存在空闲worker。

long ctl < 0  表示活跃的线程数小于线程池初始化时设置的并行度要求
long ctl & ADD_WORKER != 0L   //不等于0，说明总的线程数小于parallelism属性，可以新增线程
int  (int)ctl == 0  //取低32位的值，如果为0，则活跃线程数等于总的线程数，即没有空闲的线程数
```

![image-20211111150140094](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/image-20211111150140094.png)

##### 2.2.2 runState

```apl
runState < 0  已关闭
runState & STARTED !=0 说明未启动   
```

##### 2.2.3 config

```api
config & SMASK  获取并行度
config & MODE_MASK  获取队列模式
```

###  3. 构造函数

#### 3.1 ForkJoinPool(parallelism)

创建一个具有**指定并行级别、默认线程工厂、无 UncaughtExceptionHandler 和非异步 LIFO** 处理模式的ForkJoinPool 

```java
    public ForkJoinPool(int parallelism) {
        // 默认工厂为  java.util.concurrent.ForkJoinPool.DefaultForkJoinWorkerThreadFactory
        this(parallelism, defaultForkJoinWorkerThreadFactory, null, false);
    }
```

####  3.2 ForkJoinPool(parallelism, factory,handler,asyncMode)

```java
public ForkJoinPool(int parallelism,
                    ForkJoinWorkerThreadFactory factory,
                    UncaughtExceptionHandler handler,
                    boolean asyncMode) {
    this(checkParallelism(parallelism),
         checkFactory(factory),
         handler,
         asyncMode ? FIFO_QUEUE : LIFO_QUEUE,
         "ForkJoinPool-" + nextPoolId() + "-worker-");
    checkPermission();
}
```

这个方法中会对并行度和factory进程check，以确保系统能支持。
 parallelism最大不允许大于MAX_CAP。最小必须大于0。
 factory则只是判断是否为空，如果为空则这个地方会出现NPE异常。

####  3.3 ForkJoinPool(parallelism,factory,handler,mode,workerNamePrefix)

使用给定的参数创建一个ForkJoinPool ，没有任何安全检查或参数验证。 由 `makeCommonPool()` 直接调用

```java
private ForkJoinPool(int parallelism,
                         ForkJoinWorkerThreadFactory factory,
                         UncaughtExceptionHandler handler,
                         int mode,
                         String workerNamePrefix) {
        this.workerNamePrefix = workerNamePrefix;
        this.factory = factory;
        this.ueh = handler;
        this.config = (parallelism & SMASK) | mode;
        long np = (long)(-parallelism); // offset ctl counts
        this.ctl = ((np << AC_SHIFT) & AC_MASK) | ((np << TC_SHIFT) & TC_MASK);
}
```

- **parallelism**默认是cpu核心数，ForkJoinPool里线程数量依据于它，但不表示最大线程数，不要等同于ThreadPoolExecutor里的corePoolSize或者maximumPoolSize。
- **factory**是线程工程，不是新东西了，默认实现是`DefaultForkJoinWorkerThreadFactory`。
- **mode**是类型。

- **workerNamePrefix**是其中线程名称的前缀，默认使用“ForkJoinPool-*”。

这个方法才是真正的初始化的构造方法。上述计算过程，假定parallelism 为7：

![img](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/3237432-934f6444104f0baa.png)

然后再计算np和ctl

![img](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/3237432-fb16a49de6b71009.png)

这样就完成了初始化。

### 4. 基本组成

   再了解了前面的变量之后，我们可以发现，`ForkJoinPool`的实际组成是，由一个`WorkQueue`的数组构成。但是这个数组比较特殊，**在偶数位，存放外部调用提交的任务，如通过execute和submit等方法。这种队列称为`SubmissionQueue`**。而**另外一种任务是前者在执行过程种通过fork方法产生的新任务。这种队列称为`workQueue`**。SubmissionQueue与WorkQueue的区别在于，WorkQueue的属性“final ForkJoinWorkerThread owner;”是有值的。也就是说，WorkQueue将有ForkJoinWorkerThread线程与之绑定。在运行过程中不断的从WorkQueue中获取任务。如果没有可执行的任务，则将从其他的SubmissionQueue和WorkQueue中窃取任务来执行。前面学习过了工作窃取算法，实际上载WorkQueue上的ForkJoinWorkerThread就是一个窃取者，它从SubmissionQueue中或者去偷的WorkQueue中，按FIFO的方式窃取任务。之后执行。也从自己的WorkQueue中安LIFO或者FIFO的方式执行任务。这取决于模式的设定。默认情况下是采用LIFO的方式在执行。组成如下图所示：

![img](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/3237432-55f1a2ccf326d444.png)

###  5. 外部提交的方法

#### 5.1 invoke / execute / submit / invokeAll

 `invoke`方法是ForkJoinPool的特有方法，会将指定任务提交到任务队列中并阻塞当前线程等待任务执行完成；execute，submit，invokeAll方法都是改写了父类AbstractExecutorService的实现，原实现是将Runnable或者Callable接口包装成FutureTask，这里统一包装成ForkJoinTask的某个子类。

```java
public <T> T invoke(ForkJoinTask<T> task) {
        if (task == null)
            throw new NullPointerException();
        //将新任务加入到任务队列中    
        externalPush(task);
        return task.join(); //等待任务执行完成
 }
 
public void execute(ForkJoinTask<?> task) {
        if (task == null)
            throw new NullPointerException();
        //将新任务加入到任务队列中     
        externalPush(task);
}
 
public void execute(Runnable task) {
        if (task == null)
            throw new NullPointerException();
        ForkJoinTask<?> job;
        if (task instanceof ForkJoinTask<?>) // avoid re-wrap
            job = (ForkJoinTask<?>) task;
        else
            //如果不是ForkJoinTask，则进行一层保证
            job = new ForkJoinTask.RunnableExecuteAction(task);
        //将新任务加入到任务队列中        
        externalPush(job);
  }
 
public <T> ForkJoinTask<T> submit(ForkJoinTask<T> task) {
        if (task == null)
            throw new NullPointerException();
        //将新任务加入到任务队列中     
        externalPush(task);
        return task;
}
 
public <T> ForkJoinTask<T> submit(Callable<T> task) {
        //将Callable进行包装
        ForkJoinTask<T> job = new ForkJoinTask.AdaptedCallable<T>(task);
        //将新任务加入到任务队列中 
        externalPush(job);
        return job;
 }
 
public <T> ForkJoinTask<T> submit(Runnable task, T result) {
        //将Runnable给包装成ForkJoinTask
        ForkJoinTask<T> job = new ForkJoinTask.AdaptedRunnable<T>(task, result);
        externalPush(job);
        return job;
 }
 
public ForkJoinTask<?> submit(Runnable task) {
        if (task == null)
            throw new NullPointerException();
        ForkJoinTask<?> job;
        if (task instanceof ForkJoinTask<?>) // avoid re-wrap
            job = (ForkJoinTask<?>) task;
        else
            //将Runnable给包装成ForkJoinTask
            job = new ForkJoinTask.AdaptedRunnableAction(task);
        externalPush(job);
        return job;
 }
 
 public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks) {
        ArrayList<Future<T>> futures = new ArrayList<>(tasks.size());
 
        boolean done = false;
        try {
            for (Callable<T> t : tasks) {
                //Callable包装成ForkJoinTask
                ForkJoinTask<T> f = new ForkJoinTask.AdaptedCallable<T>(t);
                futures.add(f);
                externalPush(f);
            }
            for (int i = 0, size = futures.size(); i < size; i++)
                ((ForkJoinTask<?>)futures.get(i)).quietlyJoin(); //等待任务执行完成
            done = true;
            return futures;
        } finally {
            if (!done)
                for (int i = 0, size = futures.size(); i < size; i++)
                    futures.get(i).cancel(false); //出现异常取消掉剩余的未执行任务
        }
    }
```

####  5.2 externalPush / tryExternalUnpush

ForkJoinPool跟ThreadPoolExecutor不同，后者只有一个支持阻塞的任务队列，前者是有多个任务队列，任务队列的数量是大于2*parallelism的最小的2的整数次幂的值，且任务队列是非阻塞的，多个任务队列的设计跟LongAdder的设计思想类似，可减少高并发提交任务时减少对同一个队列的竞争，减少因加锁解锁等同步操作的性能损耗。

将某个任务提交到任务队列时，是根据`m & r & SQMASK`的方式来计算所属的任务队列，其中**m等于任务队列数组的长度减1，r等于当前线程的probe属性，SQMASK是任务队列数组的最大个数**。如果所属的任务队列未初始化，则会创建一个新的任务队列，注意新创建的任务队列是`SHARED_QUEUE`共享模式，即不属于某个Worker线程，且是`INACTIVE`状态的。

`externalPush`方法是invoke / execute / submit / invokeAll等方法的核心实现，将某个任务提交到任务队列中，如果当前线程的`probe`属性未初始化，则初始化`probe`属性；如果线程池未初始化则初始化线程池，**初始化线程池就是根据parallelism属性初始化WorkQueue数组和stealCounter属性**；如果该task关联的WorkQueue未初始化，则创建一个新的WorkQueue，**新WorkQueue是SHARED_QUEUE和INACTIVE的**，最后将任务添加到这个新的WorkQueue中；如果关联的WorkQueue不为空，则将其添加到关联的WorkQueue中。只要添加到WorkQueue成功，**如果没有空闲的Worker线程且当前线程数小于parallelism属性则尝试创建一个新的Worker线程，否则尝试唤醒一个空闲的Worker线程。**

另外从其tryAddWorker方法的实现可知，ForkJoinPool下线程池的最大线程数就是parallelism，不可能大于parallelism，可参考ADD_WORKER的调用链，如下：

![image-20211111230301743](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/image-20211111230301743.png)

有一种情形除外，某个ForkJoinTask通过awaitJoin方法阻塞了当前Worker线程的执行，如果当前WorkQueue中还有其他未执行的任务，则会创建新的Worker线程，导致总的线程数超过了parallelism，可参考实际创建Worker线程的creatWorker方法的调用链路，如下：
![image-20211111230324656](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/image-20211111230324656.png)

代码实现如下：

#####  5.2.1 tryExternalUnpush

```java
// 如果某个task是位于关联的任务队列的顶端，即是最近一次插入的task，则将其从任务队列中移除，如果成功移除则tryExternalUnpush返回true，否则返回false。
final boolean tryExternalUnpush(ForkJoinTask<?> task) {
        WorkQueue[] ws; WorkQueue w; ForkJoinTask<?>[] a; int m, s;
        int r = ThreadLocalRandom.getProbe();
        if ((ws = workQueues) != null && (m = ws.length - 1) >= 0 &&  //线程池已初始化
            (w = ws[m & r & SQMASK]) != null && //关联的WorkQueue非空
            (a = w.array) != null && (s = w.top) != w.base) { // 关联的WorkQueue包含有未执行的任务
            //j是上一次插入到数组中的索引
            long j = (((a.length - 1) & (s - 1)) << ASHIFT) + ABASE;
            if (U.compareAndSwapInt(w, QLOCK, 0, 1)) { //队列加锁
                if (w.top == s && w.array == a && //top和array属性未变更
                    U.getObject(a, j) == task && //如果上一次插入到数组中的任务就是task
                    U.compareAndSwapObject(a, j, task, null)) { //成功将其修改为null
                    U.putOrderedInt(w, QTOP, s - 1); //top属性减1
                    U.putOrderedInt(w, QLOCK, 0); //队列解锁
                    return true;
                }
                U.compareAndSwapInt(w, QLOCK, 1, 0);//队列解锁
            }
        }
        return false;
 }
```

##### 5.2.2 externalPush

 实际上`externalPush`的逻辑只能处理简单的逻辑，对于其他复杂的逻辑，则需要通过`externalSubmit`提供，而这些简单的逻辑，实际上就是添加到任务队列。这个任务队列的索引一定是偶数：**i = m & r & SQMASK**。
 这个计算过程，实际上由于SQMASK最后一位为0，因此计算的index的最后一位一定为0，这样导致这个值为偶数。也就是说，**workQueues的偶数位存放的是外部提交的任务队列。之后提交成功之后，调用signalWork方法让其他的worker来窃取**。

```java
 /**
 * 尝试将任务添加到提交者当前的队列中，此方法只处理大多数情况，实际上是根据随机数指定一个workQueues的槽位，如果这个位置存在WorkQueue，则加入队列，然后调用signalWork通知其他工作线程来窃取。反之，则通过externalSubmit处理。这只适用于提交队列存在的普通情况。更复杂的逻辑参考externalSubmit。
 *
 * @param task the task. Caller must ensure non-null.
 */
final void externalPush(ForkJoinTask<?> task) {
    WorkQueue[] ws; WorkQueue q; int m;
    //通过ThreadLocalRandom产生随机数
    int r = ThreadLocalRandom.getProbe();
    //线程池的状态
    int rs = runState;
    //如果ws已经完成初始化，且根据随机数定位的index存在workQueue,且cas的方式加锁成功
    if ((ws = workQueues) != null && (m = (ws.length - 1)) >= 0 &&
        //此处先用随机数和wq的size取&，之后再取SQMASK，这些操作将多余的位的值去除
        (q = ws[m & r & SQMASK]) != null && 
        r != 0 && rs > 0 && // //r不等于0说明当前线程的probe属性已初始化，rs大于0说明线程池已初始化且是正常运行的
        U.compareAndSwapInt(q, QLOCK, 0, 1)) { //cas的方式加锁
        ForkJoinTask<?>[] a; int am, n, s;
        //判断q中的task数组是否为空，
        if ((a = q.array) != null &&
            //am为q的长度 这是个固定值，如果这个值大于n n就是目前队列中的元素，实际实这里是判断队列是否有空余的位置
            (am = a.length - 1) > (n = (s = q.top) - q.base)) {
            //j实际上是计算添加到workQueue中的index
            int j = ((am & s) << ASHIFT) + ABASE;
            //将task通过cas的方式插入a的index为j的位置
            U.putOrderedObject(a, j, task); 
            U.putOrderedInt(q, QTOP, s + 1); // top +1
            U.putIntVolatile(q, QLOCK, 0);  //cas 解锁
            //此处，如果队列中的任务小于等于1则通知其他worker来窃取。为什么当任务大于1的时候不通知。而且当没有任务的时候发通知岂不是没有意义？此处不太理解
            if (n <= 1)
                //这是个重点方法，通知其他worker来窃取
                //原任务队列是空的，此时新增了一个任务,则尝试新增Worker或者唤醒空闲的Worker
                signalWork(ws, q);
            return;
        }
        U.compareAndSwapInt(q, QLOCK, 1, 0);
    }
    //初始化调用线程的Probe，线程池和关联的WorkQueue
    //如果已初始化，则将任务保存到WorkQueue中
    externalSubmit(task);
}
```

##### 5.2.3 externalSubmit

```java
/**
 *externalPush的完整版本，处理哪些不常用的逻辑。如第一次push的时候进行初始化、此外如果索引队列为空或者被占用，那么创建一个新的任务队列。
 *
 * @param task the task. Caller must ensure non-null.
 */
private void externalSubmit(ForkJoinTask<?> task) {
        int r;                                    // initialize caller's probe
        if ((r = ThreadLocalRandom.getProbe()) == 0) { //初始化调用线程的Probe
            ThreadLocalRandom.localInit();
            r = ThreadLocalRandom.getProbe();
        }
        for (; ; ) {
            WorkQueue[] ws;
            WorkQueue q;
            int rs, m, k;
            boolean move = false;
            if ((rs = runState) < 0) { //如果线程池已关闭，则尝试终止，并抛出异常拒绝此次任务
                tryTerminate(false, false);     // help terminate
                throw new RejectedExecutionException();
            } else if ((rs & STARTED) == 0 ||     //如果线程池还未启动，未初始化
                    ((ws = workQueues) == null || (m = ws.length - 1) < 0)) {
                int ns = 0;
                //对线程池以CAS的方式加锁，从RUNSTATE变为RSLOCK，如果不为RUNSTATE则自旋
                rs = lockRunState(); 
                try {
                    if ((rs & STARTED) == 0) { //再次校验未初始化，
                        U.compareAndSwapObject(this, STEALCOUNTER, null,
                                new AtomicLong()); //初始化stealCounter属性
                        //获取并行度，计算对应的数组长度
                        int p = config & SMASK; // ensure at least 2 slots
                        int n = (p > 1) ? p - 1 : 1;
                        //p是3，即4核，n为2时，计算的结果是8
                        //p是7，即8核，n为6时，计算的结果是16，算出来的n是大于2n的最小的2的整数次幂的值
                        n |= n >>> 1;
                        n |= n >>> 2;
                        n |= n >>> 4;
                        n |= n >>> 8;
                        n |= n >>> 16;
                        n = (n + 1) << 1;
                        //初始化workQueues 这个数组的大小不会超过2^16
                        workQueues = new WorkQueue[n];
                        ns = STARTED;
                    }
                } finally {
                    //将线程池标记为已启动
                    unlockRunState(rs, (rs & ~RSLOCK) | ns);
                }
                //实际上这个分支只是创建了外层的workQueues数组，此时数组内的内容还是全部都是空的 
            }
           //线程池已启动 &&
          //根据随机数计算出来的槽位不为空，即索引处的队列已经创建，这个地方是外层死循环再次进入的结果
          //需要注意的是这个k的计算过程，SQMASK最低的位为0，无论随机数r怎么变化，得到的结果总是偶数。
            else if ((q = ws[k = r & m & SQMASK]) != null) {
                //关联的WorkQueue不为null
                if (q.qlock == 0 && U.compareAndSwapInt(q, QLOCK, 0, 1)) { //将WorkQueue加锁
                    ForkJoinTask<?>[] a = q.array;
                    int s = q.top;
                    boolean submitted = false; // initial submission or resizing
                    try {                      // locked version of push
                        if ((a != null && a.length > s + 1 - q.base) || //任务数组有剩余空间
                                (a = q.growArray()) != null) { //任务数组中没有剩余空间，则扩容
                            //计算任务数组中保存的位置并保存
                            int j = (((a.length - 1) & s) << ASHIFT) + ABASE;
                            U.putOrderedObject(a, j, task);
                            U.putOrderedInt(q, QTOP, s + 1);//top属性加1
                            submitted = true;
                        }
                    } finally {
                        //解锁
                        U.compareAndSwapInt(q, QLOCK, 1, 0);
                    }
                    if (submitted) { //如果任务已经提交到WorkQueue中
                        signalWork(ws, q); //新增Worker或者唤醒空闲的Worker
                        return;
                    }
                }
                //WorkQueue加锁失败
                move = true;                   // move on failure
            }
            //如果状态不为RSLOCK 上面两个分支都判断过了，说明关联的WorkQueue为null
            else if (((rs = runState) & RSLOCK) == 0) { //线程池未加锁
                q = new WorkQueue(this, null);
                q.hint = r; //使用当前线程的probe初始化hint
                // 计算config SHARED_QUEUE 将确保第一位为1 则这个计算出来的config是负数，这与初始化的方法是一致的
                q.config = k | SHARED_QUEUE; //模式是共享的，即不属于某个特定的Worker，k表示该WorkQueue在数组中的位置
                q.scanState = INACTIVE;
                rs = lockRunState();           //加锁
                if (rs > 0 && (ws = workQueues) != null && //线程池已启动且workQueues非空
                        k < ws.length && ws[k] == null) //再次校验关联的WorkQueue为null
                    ws[k] = q;                 //将新的WorkQueue保存起来
                unlockRunState(rs, rs & ~RSLOCK); //解锁，下一次for循环就将任务保存到该WorkQueue中
            } else
                move = true;                   // move if busy
            if (move) //如果WorkQueue加锁失败，则增加probe属性，下一次for循环则遍历下一个WorkQueue,即将该Task提交到其他的WorkQueue中
                r = ThreadLocalRandom.advanceProbe(r);
        }//for循环结束
    }
```

这个地方将产生的无owoner的workQueue放置在索引k的位置，需要注意的是k的计算过程，`k= r & m & SQMASK`。r是随机数，m是数组的长度，而SQMASK：![img](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/3237432-c13fdb115f754416.png)

##### 5.2.4 signalWork

上述两个方法如果提交成功，那么调用signalWork，通知工作线程运行。

```java
//如果当前总的线程数小于parallelism，则signalWork会尝试创建新的Worker线程
//如果当前有空闲的Worker线程，则尝试唤醒一个，如果没有空闲的则直接返回
final void signalWork(WorkQueue[] ws, WorkQueue q) {
        long c; int sp, i; WorkQueue v; Thread p;
        while ((c = ctl) < 0L) { //ctl初始化时使用-parallelism，为负值表示活跃的线程数小于线程池初始化时设置的并行度要求
            if ((sp = (int)c) == 0) {                  //取低32位的值，如果为0，则活跃线程数等于总的线程数，即没有空闲的线程数
                if ((c & ADD_WORKER) != 0L)    //不等于0，说明总的线程数小于parallelism属性，可以新增线程
                    tryAddWorker(c); //尝试创建新线程
                //只要没有空闲的线程，此处就break，终止while循环    
                break;
            }
            //sp不等于0，即有空闲的线程，sp等于ctl的低32位
            //ctl的最低16位保存最近空闲的Worker线程关联的WorkQueue在数组中的索引
            if (ws == null)                            //线程池未初始化或者已终止
                break;
            if (ws.length <= (i = sp & SMASK))         //线程池已终止
                break;
            if ((v = ws[i]) == null)                   //线程池在终止的过程中
                break;
            //sp加SS_SEQ，非INACTIVE就是最高位0，再加31个1
            //此处将空闲WorkQueue的INACTIVE表示去除，变成正常的激活状态   
            int vs = (sp + SS_SEQ) & ~INACTIVE;        // next scanState
            int d = sp - v.scanState;                  // screen CAS
            //AC加1，UC_MASK取高32位的值，SP_MASK取低32位的值
            //stackPred是v被标记成INACTIVE时ctl的低32位，可能保存着在v之前空闲的WorkQueue在数组中的索引
            long nc = (UC_MASK & (c + AC_UNIT)) | (SP_MASK & v.stackPred);
            if (d == 0 && U.compareAndSwapLong(this, CTL, c, nc)) {
                //d等于0说明WorkQueue v关联的Worker是最近才空闲的
                //修改ctl属性成功
                v.scanState = vs;                      //将WorkQueue v标记成已激活
                if ((p = v.parker) != null)
                    U.unpark(p); //唤醒等待的线程
                break;
            }
            //如果d不等于0或者上述CAS修改失败，即已经有一个空闲Worker被其他线程给激活了
            if (q != null && q.base == q.top)          //任务队列为空，没有待执行的任务了，不需要激活空闲Worker或者创建新线程了
                break;
            //如果if不成立则继续下一次for循环
        } //while循环结束
    }
```

可以看到，实际上signalWork就做了两件事情。

- 判断worker是否充足，如果不够，则创建新的worker。

- 判断worker的状态是否被park了，如果park则用unpark唤醒。这样worker就可以取scan其他队列进行窃取了。

#####  5.2.5 tryAddWorker

```java
/**
 * 尝试新增一个worker，然后增加ctl中记录的worker的数量
 *
 * @param c incoming ctl value, with total count negative and no
 * idle workers.  On CAS failure, c is refreshed and retried if
 * this holds (otherwise, a new worker is not needed).
 */
private void tryAddWorker(long c) {
    //传入的c为外层调用方法的ctl add标记为false
    boolean add = false;
    do {
        //AC和TC都加1
        long nc = ((AC_MASK & (c + AC_UNIT)) |
                   (TC_MASK & (c + TC_UNIT)));
        if (ctl == c) { //如果ctl未变更
            int rs, stop;                 // check if terminating
            if ((stop = (rs = lockRunState()) & STOP) == 0) //加锁成功且线程池未终止，则修改ctl
                //增加ctl的数量，如果成功 add为ture
                add = U.compareAndSwapLong(this, CTL, c, nc);
            unlockRunState(rs, rs & ~RSLOCK);   //解锁
            //如果stop不为0 则说明线程池停止 退出
            if (stop != 0)
                break;
            //如果前面增加ctl中的数量成功，那么此处开始创建worker
            if (add) {
                createWorker();
                break;
            }
        }
    //这个while循环， 前半部分与ADD_WORKER取并，最终只会保留第48位，这个位置为1，同时c的低32为为0，
    //重新读取ctl，如果依然可以新增线程且空闲线程数为0，则继续下一次for循环新增Worker
    } while (((c = ctl) & ADD_WORKER) != 0L && (int)c == 0);
}
```

实际上这个类中，只是做了一些准备过程，增加count，以及加锁判断，最终还是通过`createWorker`来进行。

##### 5.2.6 createWorker

创建worker的具体方法。

```java
//创建新的Worker，如果创建失败或者启动执行异常则通过deregisterWorker方法通知线程池将其解除注册
private boolean createWorker() {
    //创建线程的工厂方法
    ForkJoinWorkerThreadFactory fac = factory;
    Throwable ex = null;
    ForkJoinWorkerThread wt = null;
    try {
        //如果工厂方法不为空，则用这个工厂方法创建线程，之后再启动线程，此时newThread将与workQueue绑定
        if (fac != null && (wt = fac.newThread(this)) != null) {
            wt.start();
            return true;
        }
    //如果创建失败，出现了异常 则ex变量有值
    } catch (Throwable rex) {
        ex = rex;
    }
    deregisterWorker(wt, ex);
    return false;
}
```

此方法只是创建了一个`ForkJoinThread`,实际上worker还是没有创建。实际上这个创建过程是再`newThread(this)`中来进行的。

##### 5.2.7 ForkJoinWorkerThread()

```java
ForkJoinWorkerThread(ForkJoinPool pool, ThreadGroup threadGroup,
                     AccessControlContext acc) {
    super(threadGroup, null, "aForkJoinWorkerThread");
    U.putOrderedObject(this, INHERITEDACCESSCONTROLCONTEXT, acc);
    eraseThreadLocals(); // clear before registering
    this.pool = pool;
    this.workQueue = pool.registerWorker(this);
}
```

再线程创建成功之后调用`registerWorker`与之绑定。
如果线程创建失败或者出现异常就要调用`deregisterWorker`对count进行清理或者解除绑定。

##### 5.2.8 registerWorker

实际上是这个方法，完成`worker的创建和绑定（Queue）`。

registerWorker是在ForkJoinWorkerThread的构造方法中调用的，通知线程池新增了一个Worker，会配套的创建一个关联的WorkQueue，将其保存在WorkQueue数组中，保存的位置是根据indexSeed计算出来的，是一个**奇数**，如果对应位置的WorkQueue非空，则遍历WorkQueue数组找到一个WorkQueue为空且索引是奇数的位置。注意新创建的WorkQueue的模式跟线程池的模式一致，FIFO或者LIFO，且scanState初始就是激活状态，其值等于该WorkQueue数组中的位置。

```java
/**
 * 创建线程，并建立线程与workQueue的关系，此处只会在workQueues数组的奇数位操作
 *
 * @param wt the worker thread
 * @return the worker's queue
 */
final WorkQueue registerWorker(ForkJoinWorkerThread wt) {
    UncaughtExceptionHandler handler;
    //将线程设置为守护线程
    wt.setDaemon(true);                           // configure thread
    //如果没有handler则抛出异常
    if ((handler = ueh) != null)
        wt.setUncaughtExceptionHandler(handler);
    //创建一个workQueue，此时owoner为当前输入的ForkJoinThread
    WorkQueue w = new WorkQueue(this, wt);
    //定义i为0
    int i = 0;                                    // assign a pool index
    //获取线程池的队列模式 
    int mode = config & MODE_MASK; 
    //加锁 如果获取失败则阻塞当前线程
    int rs = lockRunState();
    try {
        WorkQueue[] ws; int n;                    // skip if no array
        if ((ws = workQueues) != null && (n = ws.length) > 0) {
             // 如果workQueues已初始化，indexSeed初始值为0
            int s = indexSeed += SEED_INCREMENT;  // unlikely to collide
            //m为n-1,而n为数组的初始化长度，m必为奇数
            int m = n - 1;
            //将s左移然后最后一位补上1，之后与奇数m求并集，那么得到的结果必然是奇数
            i = ((s << 1) | 1) & m;               // odd-numbered indices
            //判断i位置是否为空 如果为空，出现了碰撞，则计算步长向后移动 这个步长一定是偶数
            if (ws[i] != null) {                  // collision
                int probes = 0;                   // step by approx half n
                //最小步长为2  n是数组长度，比为偶数，那么如果n小于等于4，则步长为2，反之，则将n右移，将偶数最后一位的0清除，之后再和EVENMASK求并，这样就相当于将原来的长度缩小2倍，并确保是偶数。之后再加上2。那么假定n为16的话，此处计算的step就为10
                int step = (n <= 4) ? 2 : ((n >>> 1) & EVENMASK) + 2;
                //之后再通过while循环，继续判断增加步长之后是否碰撞，如果碰撞，则继续增加步长
                while (ws[i = (i + step) & m] != null) {
                    //如果还是碰撞，且probes增加1之后大于长度n，则会触发扩容，workQueues会扩大2倍 
                    if (++probes >= n) {
                        workQueues = ws = Arrays.copyOf(ws, n <<= 1);
                        m = n - 1;
                        //将probes置为0
                        probes = 0;
                    }
                }
            }
            //设置随机数seed
            w.hint = s;                           // use as random seed
            // 低16位保存w在WorkQueue数组中的位置，跟线程池的模式一致
            w.config = i | mode;
            //初始就是激活状态
            w.scanState = i;                      // publication fence
            //将w设置到i处
            ws[i] = w;
        }
    } finally {
       //cas的方式进行解锁
        unlockRunState(rs, rs & ~RSLOCK);
    }
    //此处设置线程name
    wt.setName(workerNamePrefix.concat(Integer.toString(i >>> 1)));
    return w;
}
```

##### 5.2.9 deregisterWorker

 `deregisterWorker`是在`ForkJoinWorkerThread`**执行任务的过程中出现未捕获异常导致线程退出时或者创建Worker线程异常时调用的，会将关联的WorkQueue标记为已终止**，将其steals属性累加到线程池的stealCounter属性中，取消掉所有未处理的任务，最后从WorkQueue数组中删除，将AC和TC都减1，如果有空闲的Worker线程则唤醒一个，如果没有且当前线程数小于parallelism则创建一个新线程，其调用链如下：

![image-20211112164217797](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/image-20211112164217797.png)

```java
/**
 * 此方法的主要目的是在创建worker或者启动worker失败之后的回调方法，此时将之前的ctl中增加的count减去。
 */
final void deregisterWorker(ForkJoinWorkerThread wt, Throwable ex) {
    WorkQueue w = null;
    //如果workQueue和thread不为空
    if (wt != null && (w = wt.workQueue) != null) {
        WorkQueue[] ws;                           // remove index from array
        //根据config计算index
        int idx = w.config & SMASK;
        //加锁
        int rs = lockRunState();
        // ws不为空且对应索引的元素是目标WorkQueue w，将其置为null
        if ((ws = workQueues) != null && ws.length > idx && ws[idx] == w)
            ws[idx] = null;
        //解锁 修改rs状态 
        unlockRunState(rs, rs & ~RSLOCK);
    }
    //后续对count减少
    long c;                                       // decrement counts
    // 将AC和TC都原子的减1，CAS失败则重试
    do {} while (!U.compareAndSwapLong
                 (this, CTL, c = ctl, ((AC_MASK & (c - AC_UNIT)) | //获取最高的16位
                                       (TC_MASK & (c - TC_UNIT)) | //获取紧挨着的16位
                                       (SP_MASK & c)))); //获取低32位                       
    if (w != null) {
        w.qlock = -1;      //表明该WorkQueue已终止工作 // ensure set
        w.transferStealCount(this); //将该WorkQueuen的steals累加到线程池的stealCounter属性中
        w.cancelAll();       //取消掉所有未完成的任务  cancel remaining tasks
    }
    for (;;) {                                    // possibly replace
        WorkQueue[] ws; int m, sp;
        if (tryTerminate(false, false) || w == null || w.array == null ||
            (runState & STOP) != 0 || (ws = workQueues) == null ||
            (m = ws.length - 1) < 0)    //如果线程池正在终止或者已经终止了 already terminating
            break;
        if ((sp = (int)(c = ctl)) != 0) {    //唤醒空闲的Worker线程 wake up replacement
            if (tryRelease(c, ws[sp & m], AC_UNIT))
                break;
        }
        //没有空闲的线程
        else if (ex != null && (c & ADD_WORKER) != 0L) { //如果是异常退出且线程总数小于parallelism
            tryAddWorker(c);    //增加一个新线程  create replacement
            break;
        }
        else                                      // don't need replacement
            break;
    }
    //异常处理
    if (ex == null)                               // help clean on way out
        ForkJoinTask.helpExpungeStaleExceptions(); //清空所有异常
    else                                          // rethrow
        ForkJoinTask.rethrow(ex); //重新抛出异常
}

//取消掉所有未执行完成的任务
 final void cancelAll() {
     ForkJoinTask<?> t;
     if ((t = currentJoin) != null) {
         currentJoin = null;
         ForkJoinTask.cancelIgnoringExceptions(t);
     }
     if ((t = currentSteal) != null) {
         currentSteal = null;
         ForkJoinTask.cancelIgnoringExceptions(t);
     }
     while ((t = poll()) != null)
         ForkJoinTask.cancelIgnoringExceptions(t);
  }
```

至此，我们通过外部线程push之后，将任务提交到submissionQueue队列之后，会根据并行度以及工作线程的需要创建workQueue。唤醒工作线程进行窃取的操作就完成了，外部线程的调用就结束了，会回到main方法中去等待结果。

####  5.3 外部提交的方法总结

上述代码调用的过程，我们可以形象的用如下图进行表示：
最开始，`workQueues`是null状态。在第一次执行的时候，`externalSubmit`方法中会初始化这个数组。

![img](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/3237432-c2a537cfacdeb471.png)

在这之后，还是在externalSubmit方法的for循环中，完成对任务队列的创建，将任务队列创建在偶数索引处。之后将任务写入这个队列：![img](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/3237432-c5e05088c2d4f343.png)



此后，任务添加完成，但是没有工作队列进行工作。因此在这之后调用`signalWork`，通知工作队列开始干活。但是在这个方法执行的过程中，由于工作队列并不存在，没有worker，所以调用`tryAddWorker`开始创建worker。在`createWorker`会真正创建一个worker线程：

![img](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/3237432-fac1e945e1396b65.png)



但是workQueue还没创建。这是在newthread之后，通过registerWorker创建的，在registerWorker方法中，会在奇数位创建一个workQueue，并将此前创建的线程与之绑定。这样一个worker就成功创建了。

![img](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/3237432-ab9ac22a1f447353.png)



这样就完成了worker创建的全过程。

### 6. workQueue工作过程

在workQueue创建完成之后，下一步，这些线程的run方法调用后被启动。之后就进入了worker线程的生命周期了。实际上run方方法如下：

```java
// java.util.concurrent.ForkJoinWorkerThread#run
public void run() {
    if (workQueue.array == null) { // only run once
        Throwable exception = null;
        try {
            onStart();
            pool.runWorker(workQueue);
        } catch (Throwable ex) {
            exception = ex;
        } finally {
            try {
                onTermination(exception);
            } catch (Throwable ex) {
                if (exception == null)
                    exception = ex;
            } finally {
                pool.deregisterWorker(this, exception);
            }
        }
    }
}
```

重点执行的式`runWorker`，此时一旦出错，可以调用`deregisterWorker`方法进行清理。下面来看看runWorker的详细过程。

#### 6.1 runWorker

**重点**:

​	`runWorker`是`ForkJoinWorkerThread`的`run`方法调用的，该方法是Worker线程执行任务的核心实现。该方法首先调用scan方法，scan方法会随机的从某个WorkQueue中获取未执行的任务，如果该WorkQueue为null或者没有未执行的任务，则继续遍历下一个WorkQueue，如果所有WorkQueue都遍历了，且遍历了三遍还是没有获取待执行的任务，且这期间没有新增的任务提交任何一个WorkQueue中，则会将该WorkQueue标记为INACTIVE并将AC减1，然后返回null，进入awaitWork方法。如果成功获取待执行的任务就调用runTask方法执行该任务，注意该方法不仅仅是执行该任务，还会将该WorkQueue中未执行的任务都执行完了，执行的过程中会将scanState的SCANNING标识去除，等所有任务都执行完成了再加上标识SCANNING。scan方法涉及WorkQueue中的两个关键属性，scanState和stackPred，某个Worker线程刚创建时其关联的WorkQueue的scanState就是该WorkQueue在数组中的索引，是一个非负数，参考registerWorker方法。当Worker线程执行scan方法无法获取到待执行的任务，会将关联的WorkQueue标记成INACTIVE，scanState属性的最高位变成1，其值变成一个负数，然后通过stackPred保存原来ctl属性的低32位的值，将变成负数的scanState写入ctl的低32位中并且将ctl中AC减1。当signalWork方法唤醒空闲的Worker线程时，会根据ctl属性的低32位获取Worker线程关联的WorkQueue，将其scanState的INACTIVE标识去除，scanState变成一个非负数，将AC加1，将stackPred属性写入ctl的低32位中，即将ctl恢复到该WorkQueue被置为INACTIVE时的状态。综上，ctl的低32位中保存着最近被置为INACTIVE的WorkQueue的索引，而该WorkQueue由保存着之前的ctl的低32位的值，据此可以不断向上遍历获取所有被置为INACTIVE状态的WorkQueue。可参考stackPred属性的调用链，如下：

![img](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202005101729202.png)

​	进入awaitWork方法，如果线程池准备关闭或者当前线程等待被激活超时，则返回false，终止外层的for循环，Worker线程退出，否则将当前线程阻塞指定的时间或者无期限阻塞，直到signalWork方法或者tryRelease方法将其唤醒，可参考parker属性的调用链，如下：

![img](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/20200510170345344.png)

​	参考awaitWork方法的实现可知，如果有多个Worker线程关联的WorkQueue依次进入此逻辑，则只有最后一个进入此逻辑的线程因为等待激活超时而退出，该线程退出后会唤醒之前的一个处于阻塞状态的WorkQueue，如果依然没有待执行的任务，则会继续等待并退出，直到最后所有线程退出，即ForkJoinPool中没有核心线程数的概念，如果长期没有待执行的任务，线程池中所有线程都会退出。

*这是worker工作线程的执行方法。通过死循环，不断scan是否有任务，之后窃取这个任务进行执行。*

```java
/**
 * 通过调用线程的run方法，此时开始最外层的runWorker
 */
final void runWorker(WorkQueue w) {
   //初始化队列，这个方法会根据任务进行判断是否需要扩容
    w.growArray();                   // allocate queue
    // hint初始值就是累加后的indexSeed
    int seed = w.hint;               // initially holds randomization hint
    int r = (seed == 0) ? 1 : seed;  // avoid 0 for xorShift
    for (ForkJoinTask<?> t;;) {
        if ((t = scan(w, r)) != null)
            // 将t和w中所有的未处理任务都执行完，t有可能是w中的，也可能是其他WorkQueue中的
            w.runTask(t);
       //如果scan返回null,说明所有的WorkQueue中都没有待执行的任务了,则调用awaitWork阻塞当前线程等待任务
       //此时w已经被置为非激活状态
       //awaitWork返回false，则终止for循环，线程退出，返回true则继续遍历
        else if (!awaitWork(w, r))
            break;
        r ^= r << 13; r ^= r >>> 17; r ^= r << 5; // xorshift
    }
}
```

​    `runWorker`是`ForkJoinWorkerThread`的run方法调用的，该方法是**Worker线程执行任务的核心实现**。该方法首先调用`scan`方法，进行获取任务。**如果成功获取待执行的任务就调用runTask方法执行该任务**。否则调用`awaitWork`阻塞线程。

#### 6.2 scan

此方法最大的特点就是，根据随机数计算一个k，然后根据k去遍历workQueues，之后看看这个位置是否有数据，如果不为空，则检查`checkSum`，根据**checkSum的状态缺认是否从这个队列中取数据**。按之前约定的FIFO或者LIFO取数。这意味着，工作队列对窃取和是否获得本队列中的任务之间并没有优先级，而是根据随机数得到的index，之后对数组进行遍历。

```java
//scan方法从r对应的一个WorkQueue开始遍历，查找待处理的任务，如果对应的WorkQueue为空或者没有待处理的任务，则遍历下一个WorkQueue
//直到所有的WorkQueue都遍历了两遍还没有找到待处理的任务，则返回null
// 遍历第一遍没有找到时，oldSum不等于checkSum，被赋值成checkSum，
// 遍历第二遍时，如果此期间没有新增的任务,则checkSum不变，oldSum等于checkSum，ss和scanState都变成负数，
// 遍历第三遍时如果发现了一个待处理任务，则将oldSum和checkSum都置为0，如果还发现了一个待处理任务，则将最近休眠的WorkQueue，可能就是当前WorkQueue置为激活状态，并更新ss。
// 如果第三遍遍历时还没有找到待处理的任务切期间没有新增的任务，则终止for循环，返回null，进入awaitWork方法。
    /**
     * 通过scan方法进行任务窃取，扫描从一个随机位置开始，如果出现竞争则通过魔数继续随机移动，反之则线性移动，
     * 直到所有队列上的相同校验连续两次出现为空，则说明没有任何任务可以窃取，因此worker会停止窃取，之后重新
     * 扫描，如果找到任务则重新激活，否则返回null，扫描工作应该尽可能少的占用内存，以减少对其他扫描线程的干
     * 扰。
     *
     * @param w the worker (via its WorkQueue)
     * @param r a random seed
     * @return a task, or null if none found
     */
    private ForkJoinTask<?> scan(WorkQueue w, int r) {
        WorkQueue[] ws; int m;
        //如果workQueues不为空且长度大于1，当前workQueue不为空
        if ((ws = workQueues) != null && (m = ws.length - 1) > 0 && w != null) {
            int ss = w.scanState; //scanState的初始值就是该元素在数组中的索引，大于0 initially non-negative
            //for循环 这是个死循环  origin将r与m求并，将多余位去除。然后赋值给k
            for (int origin = r & m, k = origin, oldSum = 0, checkSum = 0; ; ) {
                WorkQueue q; ForkJoinTask<?>[] a; ForkJoinTask<?> t;
                int b, n; long c;
                if ((q = ws[k]) != null) { //如果k处不为空
                    if ((n = (b = q.base) - q.top) < 0 &&
                            (a = q.array) != null) {   //如果有待处理的任务 non-empty
                        long i = (((a.length - 1) & b) << ASHIFT) + ABASE;
                        //得到i处的task
                        if ((t = ((ForkJoinTask<?>)
                                U.getObjectVolatile(a, i))) != null &&
                                q.base == b) { //base未变更且base对应的数组元素不是null
                            //如果扫描状态大于0
                            if (ss >= 0) {
                                //更改a中i的值为空 也就是此处将任务窃取走了
                                if (U.compareAndSwapObject(a, i, t, null)) {
                                    q.base = b + 1; //将底部的指针加1
                                    //还有待处理的任务，唤醒空闲的Worker线程或者新增一个Worker线程
                                    if (n < -1)       // signal others
                                        signalWork(ws, q);
                                    //将窃取的task返回
                                    return t;
                                }
                            }
                            //如果 scan状态小于0 说明w由激活状态流转成非激活状态了
                            else if (oldSum == 0 && //oldSum等于0说明是第一轮遍历 try to activate
                                    w.scanState < 0)
                                // 尝试激活最近一个被置为未激活的WorkQueue
                                tryRelease(c = ctl, ws[m & (int)c], AC_UNIT);
                        }
                        // 如果ss小于0
                        // base对应的元素为null或者base发生变更，说明其他某个线程正在处理这个WorkQueue
                        if (ss < 0)                   // refresh
                            ss = w.scanState; //上一步tryRelease会将scanState变成大于0,更新ss
                        r ^= r << 1; r ^= r >>> 3; r ^= r << 10;
                        // 重置k，遍历下一个元素，相当于重新调用scan方法  /*，下一次for循环就命中上面的ss小于0时的else if分支，*/
                        // 通过tryRelease方法将scanState变成一个非负数
                        origin = k = r & m;           // move and rescan
                        oldSum = checkSum = 0;
                        continue;
                    }
                    // 如果没有待处理的任务，checkSum将所有的WorkQueue的base值累加起来
                    checkSum += b;
                }
                //如果k对应的数组元素为null，或者不为null但是没有待处理的任务
                //会不断的k+1，往前遍历，再次等于origin时，相当于所有的WorkQueue都遍历了一遍
                if ((k = (k + 1) & m) == origin) {    // continue until stable
                    //ss大于等于0说明w还是激活状态
                    //如果ss小于0且跟scanState相等，说明w已经从激活状态变成非激活了
                    if ((ss >= 0 || (ss == (ss = w.scanState))) &&
                            //重新计算的checkSum没有改变，说明这期间没有新增任务
                            //第一次进入此方法时，因为oldSum初始值为0，所以跟checkSum不等，只有第二次进入时才可能相等
                            oldSum == (oldSum = checkSum)) {
                        //ss小于0表示w已经是非激活了
                        //w.qlock小于0表示关联的Worker线程准备退出了
                        if (ss < 0 || w.qlock < 0)    // already inactive
                            break;
                        //ss大于0，其值就是w在数组中的索引
                        int ns = ss | INACTIVE;       // try to inactivate 尝试将其标记成不活跃的
                        //SP_MASK取低32位，UC_MASK取高32位，计算新的ctl，获取nc的低32位时的值就会等于ns
                        long nc = ((SP_MASK & ns) |
                                (UC_MASK & ((c = ctl) - AC_UNIT))); // AC -1
                        w.stackPred = (int)c;   //保存上一次的ctl，取其低32位      // hold prev stack top
                        U.putInt(w, QSCANSTATE, ns); //修改w的scanState属性，再把WorkQueue数组遍历一遍后进入此分支，因为ss小于0而终止循环
                        if (U.compareAndSwapLong(this, CTL, c, nc))
                            ss = ns;
                        else
                            w.scanState = ss;      //如果cas修改失败，则恢复scanState  back out
                    }
                    //checkSum重置为0，下一轮遍历时会重新计算
                    checkSum = 0;
                }
            }
        }
        return null;
    }
```

​    **scan方法会随机的从某个WorkQueue中获取未执行的任务，如果该WorkQueue为null或者没有未执行的任务，则继续遍历下一个WorkQueue，如果所有WorkQueue都遍历了，且遍历了三遍还是没有获取待执行的任务，且这期间没有新增的任务提交任何一个WorkQueue中，则会将该WorkQueue标记为INACTIVE并将AC减1，然后返回null，进入awaitWork方法**。**如果成功获取待执行的任务就调用runTask方法执行该任务**，注意该方法不仅仅是执行该任务，还会将该`WorkQueue`中未执行的任务都执行完了，执行的过程中会将`scanState`的`SCANNING`标识去除，等**所有任务都执行完成了再加上标识SCANNING**。

  `scan`方法涉及`WorkQueue`中的两个关键属性，`scanState和stackPred`，某个Worker线程刚创建时其关联的`WorkQueue`的`scanState`就是该**WorkQueue在数组中的索引，是一个非负数**，参考`registerWorker`方法。当Worker线程执行`scan`方法无法获取到待执行的任务，会将**关联的WorkQueue**标记成`INACTIVE`，`scanState`属性的最高位变成1，其值变成一个**负数**，然后通过`stackPred`**保存原来ctl属性的低32位的值，将变成负数的scanState写入ctl的低32位中并且将ctl中AC减1**。当`signalWork`方法唤醒空闲的Worker线程时，会根据`ctl`属性的**低32位获取Worker线程关联的WorkQueue**，将其`scanState`的`INACTIVE`标识去除，`scanState`变成一个非负数，将AC加1，将`stackPred`**属性写入ctl的低32位中**，即**将ctl恢复到该WorkQueue被置为INACTIVE时的状态**。	综上，**ctl的低32位中保存着最近被置为INACTIVE的WorkQueue的索引**，而该WorkQueue由保存着之前的ctl的低32位的值，**据此可以不断向上遍历获取所有被置为`INACTIVE`状态的WorkQueue**。可参考`stackPred`属性的调用链，如下：![img](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202005101729202.png)

#### 6.3 tryRelease

对于执行完成的worker，则需要进行释放。

```java
/**
 * 如果worker处于空闲worker Stack的workQueue的顶部。则发信号对其进行释放。
 * 
 * @param c incoming ctl value
 * @param v if non-null, a worker
 * @param inc the increment to active count (zero when compensating)
 * @return true if successful
 */
//c就是当前的ctl，v表示空闲Worker线程关联的WorkQueue，inc就是AC_UNIT
private boolean tryRelease(long c, WorkQueue v, long inc) {
    //sp取c的低位，计算vs
    int sp = (int)c, vs = (sp + SS_SEQ) & ~INACTIVE; Thread p;
    // 如果v是最近被置为未激活的
    if (v != null && v.scanState == sp) {          // v is at top of stack
        // AC加1，计算v被置为未激活时的ctl
        long nc = (UC_MASK & (c + inc)) | (SP_MASK & v.stackPred);
        if (U.compareAndSwapLong(this, CTL, c, nc)) {
            // 修改ctl成功，唤醒parker
            v.scanState = vs;
            if ((p = v.parker) != null)
                U.unpark(p);
            return true;
        }
    }
    //如果v不是最近空闲的则返回false
    return false;
}
```

####  6.4 awaitWork

```java
/**
 * 如果不能窃取到任务，那么就将worker阻塞。如果停用导致线程池处于静止状态，则检查是否要关闭，如果这不是唯
 * 一的工作线程，则等待给定的持续时间，达到超时时间后，如果ctl没有更改，则将这个worker终止，之后唤醒另外
 * 一个其他的worker对这个过程进行重复。
 *
 * @param w the calling worker
 * @param r a random seed (for spins)
 * @return false if the worker should terminate
 */
//进入此方法表示当前WorkQueue已经被标记成INACTIVE状态
private boolean awaitWork(WorkQueue w, int r) {
    if (w == null || w.qlock < 0)   // w终止了  w is terminating
        return false;
    for (int pred = w.stackPred, spins = SPINS, ss;;) {
        //如果ss大于0 则返回
        if ((ss = w.scanState) >= 0) //w被激活了，终止for循环，返回true
            break;
        //w未激活 
        else if (spins > 0) {
           //计算r
            r ^= r << 6; r ^= r >>> 21; r ^= r << 7;
            if (r >= 0 && --spins == 0) {  //随机数大于等于0，则将spins减1，自旋   randomize spins
                //如果spins等于0
                WorkQueue v; WorkQueue[] ws; int s, j; AtomicLong sc;
                //如果pred不为0 且ws不为空
                if (pred != 0 && (ws = workQueues) != null &&
                     //j<ws.length
                    (j = pred & SMASK) < ws.length &&  //j是上一个被置为未激活的WorkQueue的索引
                    (v = ws[j]) != null &&        // see if pred parking
                    (v.parker == null || v.scanState >= 0)) //如果j对应的WorkQueue被激活了
                    spins = SPINS;                // 继续自旋等待 continue spinning
            }
        }
        else if (w.qlock < 0)    // 如果被终止 recheck after spins
            return false;
        else if (!Thread.interrupted()) { //线程未中断
            long c, prevctl, parkTime, deadline;
            //获取当前获取的线程数，(c = ctl) >> AC_SHIFT算出来的是一个负值，加上parallelism属性就大于等于0
            int ac = (int)((c = ctl) >> AC_SHIFT) + (config & SMASK);
            if ((ac <= 0 && tryTerminate(false, false)) ||
                (runState & STOP) != 0)   // 如果线程池被终止 pool terminating
                return false;
            // w是最近一个被置为未激活状态的WorkQueue
            if (ac <= 0 && ss == (int)c) {        // is last waiter
                //计算原来的ctl
                prevctl = (UC_MASK & (c + AC_UNIT)) | (SP_MASK & pred);
                //获取TC，总的线程数，当前实现下t最大为0，最小为-parallelism，不可能大于2
                int t = (short)(c >>> TC_SHIFT);  // shrink excess spares
                if (t > 2 && U.compareAndSwapLong(this, CTL, c, prevctl))
                    return false;   //如果修改ctl成功则返回false，让当前线程退出，即核心线程数为2，此处修改ctl将AC加1，在degisterWorker方法中会将AC减1  else use timed wait
                //如果修改t小于等于2或者修改ctl失败
                //计算阻塞的时间，t最大就等于0，此时线程数等于parallelism属性，如果小于0说明线程数不足，此时parkTime的值就更大
                parkTime = IDLE_TIMEOUT * ((t >= 0) ? 1 : 1 - t);
                deadline = System.nanoTime() + parkTime - TIMEOUT_SLOP;
            }
            else
                //有新的WorkQueue被置为未激活状态，parkTime为0表示无期限等待，只能被唤醒
                prevctl = parkTime = deadline = 0L;
            Thread wt = Thread.currentThread();
            U.putObject(wt, PARKBLOCKER, this);   // emulate LockSupport
            w.parker = wt;
            // 阻塞当前线程
            if (w.scanState < 0 && ctl == c) //再次检查w的状态，ctl是否改变 recheck before park
                U.park(false, parkTime);  //休眠指定的时间或者无期限休眠
            //当前线程被唤醒
            U.putOrderedObject(w, QPARKER, null);
            U.putObject(wt, PARKBLOCKER, null);
            // 无期限等待被唤醒肯定是w被激活了
            if (w.scanState >= 0) //w被激活了，终止for循环返回true
                break;
            // scanState依然小于0
            // 如果有多个Worker线程关联的WorkQueue依次进入此逻辑，则只有最后一个进入此逻辑的线程因为等待激活超时而退出，该线程退出后会唤醒之前的一个处于阻塞状态的WorkQueue，如果依然没有待执行的任务，则会继续退出，如此最后所有线程都会退出
            if (parkTime != 0L && ctl == c &&
                deadline - System.nanoTime() <= 0L &&
                U.compareAndSwapLong(this, CTL, c, prevctl))
                return false;                     // shrink pool
        	//如果if不成立则继续下一次for循环
        }
    }
    return true;
}
```

​    进入`awaitWork`方法，如果线程池准备关闭或者当前线程等待被激活超时，则返回false，终止外层的for循环，Worker线程退出，否则将当前线程阻塞指定的时间或者无期限阻塞，直到`signalWork`方法或者`tryRelease`方法将其唤醒，可参考parker属性的调用链，如下：
![img](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/20200510170345344.png)

   参考`awaitWork`方法的实现可知，**如果有多个Worker线程关联的WorkQueue依次进入此逻辑，则只有最后一个进入此逻辑的线程因为等待激活超时而退出，该线程退出后会唤醒之前的一个处于阻塞状态的WorkQueue，如果依然没有待执行的任务，则会继续等待并退出，直到最后所有线程退出，即ForkJoinPool中没有核心线程数的概念，如果长期没有待执行的任务，线程池中所有线程都会退出**。

#### 6.5 workQueue工作过程总结

   当`workQueue`上的thread启动之后，这个线程就会调用`runWorker`方法。之后再`runWorker`方法中有一个死循环，不断的通过`scan`方法去扫描任务，实际上就是执行窃取过程。如下图所示，这样通过遍历外层workQueues的方式将会从任务队列中窃取任务进行执行。

![img](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/3237432-ecd6c1392df9c6d7.png)

   当执行之后，会通过fork产生新的任务，将这些任务任务添加到工作队列中去。其他线程继续scan，执行。这个过程不断循环，直到任务全部都执行完成，这样就完成了整个计算过程。

### 7.其他方法

#### 7.1 lockRunState / unlockRunState

```
`lockRunState`将当前线程池加锁，如果当前runState已加锁或者CAS加锁失败则调用`awaitRunStateLock`等待锁释放并加锁；`awaitRunStateLock`方法会阻塞当前线程，直到锁释放被唤醒，然后尝试获取锁，如果获取失败则继续阻塞，直到获取成功为止；`unlockRunState`方法将当前线程池解锁，如果oldRunState跟当前的runstate属性不一致，说明有某个线程在等待获取锁，在runstate上加上了RSIGNAL标识，则直接修改runstate，去掉RSIGNAL标识，并唤醒所有等待获取锁的线程。
```

​	线程池加锁和解锁是成对出现的，用于线程池初始化或者终止，修改WorkerQueue数组，修改ctl属性时使用，参考lockRunState的调用链，如下：

![img](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/20200509145917706.png)

```java
//执行加锁
private int lockRunState() {
    int rs;
    return ((((rs = runState) & RSLOCK) != 0 || //如果已经加锁
             !U.compareAndSwapInt(this, RUNSTATE, rs, rs |= RSLOCK)) ? //如果未加锁则CAS加锁
            //如果加锁失败调用awaitRunStateLock，阻塞当前线程等待获取锁，否则返回rs
            awaitRunStateLock() : rs);
}
//执行解锁
private void unlockRunState(int oldRunState, int newRunState) {
    if (!U.compareAndSwapInt(this, RUNSTATE, oldRunState, newRunState)) {
        //如果cas修改失败，只能是oldRunState发生改变了，加上了RSIGNAL
        //所以此处是直接重置runState，去除RSLOCK和RSIGNAL，并唤醒所有等待获取锁的线程
        Object lock = stealCounter;
        runState = newRunState;              // clears RSIGNAL bit
        if (lock != null)
            //唤醒所有等待获取锁的线程
            synchronized (lock) { lock.notifyAll(); }
    }
}

//阻塞当前线程，直到锁释放被唤醒，然后尝试获取锁
private int awaitRunStateLock() {
    Object lock;
    boolean wasInterrupted = false;
    //SPINS默认是0
    for (int spins = SPINS, r = 0, rs, ns;;) {
        if (((rs = runState) & RSLOCK) == 0) { //如果未加锁
            if (U.compareAndSwapInt(this, RUNSTATE, rs, ns = rs | RSLOCK)) {
                //尝试加锁成功
                if (wasInterrupted) {
                    try {
                        //将当前线程标记为中断
                        Thread.currentThread().interrupt();
                    } catch (SecurityException ignore) {
                    }
                }
                return ns;
            }
        }
        //如果已加锁
        else if (r == 0)
            //初始化r
            r = ThreadLocalRandom.nextSecondarySeed();
        else if (spins > 0) {
            //自旋等待
            r ^= r << 6; r ^= r >>> 21; r ^= r << 7; // xorshift
            if (r >= 0)
                --spins;
        }
        //spins等于0
        else if ((rs & STARTED) == 0   //线程池未初始化
                 || (lock = stealCounter) == null)
            Thread.yield();   //执行yeild，等待线程池初始化
        //线程池已初始化    
        else if (U.compareAndSwapInt(this, RUNSTATE, rs, rs | RSIGNAL)) { //状态加上RSIGNAL
            synchronized (lock) {
                if ((runState & RSIGNAL) != 0) { //再次校验
                    try {
                        lock.wait(); //阻塞当前线程
                    } catch (InterruptedException ie) {
                        if (!(Thread.currentThread() instanceof
                              ForkJoinWorkerThread))
                            wasInterrupted = true;
                    }
                }
                else
                    //如果RSIGNAL标识没了，说明unlockRunState将该标识清除了
                    //此时还在synchronized代码块中未释放锁，所以unlockRunState中的synchronized会被阻塞
                    //唤醒所有等待获取锁的线程
                    lock.notifyAll();
            }
            //下一次for循环会尝试获取锁
        }
    }//for循环结束
}
```

#### 7.2 managedBlock

​	`ManagedBlocker`是一个接口，`block`方法用于将当前线程阻塞，`isReleasable`方法返回true表示不需要阻塞了。

```java
public static void managedBlock(ManagedBlocker blocker)
    throws InterruptedException {
    ForkJoinPool p;
    ForkJoinWorkerThread wt;
    Thread t = Thread.currentThread();
    if ((t instanceof ForkJoinWorkerThread) && //如果是ForkJoinWorkerThread
        (p = (wt = (ForkJoinWorkerThread)t).pool) != null) {
        WorkQueue w = wt.workQueue;
        while (!blocker.isReleasable()) { //如果需要阻塞
            if (p.tryCompensate(w)) { //进一步判断是否需要阻塞
                try {
                    //让当前线程阻塞
                    do {} while (!blocker.isReleasable() &&
                                 !blocker.block());
                } finally {
                    //活跃的线程数加1
                    U.getAndAddLong(p, CTL, AC_UNIT);
                }
                break;
            }
        }
    }
    else {
        //普通的JavaThread
        do {} while (!blocker.isReleasable() &&
                     !blocker.block());
    }
}
```

##  三. workQueue源码分析

### 1 .类结构和注释

WorkQueue是ForkJoinPool的核心内部类，是一个Contented修饰的静态内部类。

```java
/**
 * Queues supporting work-stealing as well as external task
 * submission. See above for descriptions and algorithms.
 * Performance on most platforms is very sensitive to placement of
 * instances of both WorkQueues and their arrays -- we absolutely
 * do not want multiple WorkQueue instances or multiple queue
 * arrays sharing cache lines. The @Contended annotation alerts
 * JVMs to try to keep instances apart.
 */
@sun.misc.Contended
static final class WorkQueue {}

workQUeue是一个支持任务窃取和外部提交任务的队列，其实现参考ForkJoinPool描述的算法。在大多数平台上的性能对工作队列及其数组的实例都非常敏感。我们不希望多个工作队列的实例和多个队列数组共享缓存。@Contented注释用来提醒jvm将workQueue在执行的时候与其他对象进行区别。
@Contented实际上就是采用内存对齐的方式，保证WorkQueue在执行的时候，其前后不会有其他对象干扰。
```

###  2. 常量

在WorkQueue中有两个重要的常量，分别是`INITIAL_QUEUE_CAPACITY`和`MAXIMUM_QUEUE_CAPACITY`。

#### 2.1 INITIAL_QUEUE_CAPACITY

```java
/**
 * Capacity of work-stealing queue array upon initialization.
 * Must be a power of two; at least 4, but should be larger to
 * reduce or eliminate cacheline sharing among queues.
 * Currently, it is much larger, as a partial workaround for
 * the fact that JVMs often place arrays in locations that
 * share GC bookkeeping (especially cardmarks) such that
 * per-write accesses encounter serious memory contention.
 */
static final int INITIAL_QUEUE_CAPACITY = 1 << 13;

这是工作窃取队列的初始化容量，这个容量必须是2的幂，而且至少是4。理论上应该比4更大，以减少在CPU执行的时候多个队列进行共享内存的情况。这个值目前是远大于4的。做为一种局部解决方案，jvm经常将数组放在共享GC的记录中，尤其是cardmarks，这样在每次访问的过程中都会出现严重的内存争用。
```

#### 2.2 MAXIMUM_QUEUE_CAPACITY

```java
/**
 * Maximum size for queue arrays. Must be a power of two less
 * than or equal to 1 << (31 - width of array entry) to ensure
 * lack of wraparound of index calculations, but defined to a
 * value a bit less than this to help users trap runaway
 * programs before saturating systems.
 */
static final int MAXIMUM_QUEUE_CAPACITY = 1 << 26; // 64M
MAXIMUM_QUEUE_CAPACITY是队列支持的最大容量，必须是2的幂小于或等于1<<（31-数组项的宽度），但定义为一个略小于此值的值，以帮助用户在饱和系统之前捕获失控的程序。
```

###  3. 变量

```java
volatile int scanState;    // versioned, <0: inactive; odd:scanning
int stackPred;             // pool stack (ctl) predecessor
int nsteals;               // number of steals
int hint;                  // randomization and stealer index hint
int config;                // pool index and mode
volatile int qlock;        // 1: locked, < 0: terminate; else 0
volatile int base;         // index of next slot for poll
int top;                   // index of next slot for push
ForkJoinTask<?>[] array;   // the elements (initially unallocated)
final ForkJoinPool pool;   // the containing pool (may be null)
final ForkJoinWorkerThread owner; // owning thread or null if shared
volatile Thread parker;    // == owner during call to park; else null
volatile ForkJoinTask<?> currentJoin;  // task being joined in awaitJoin
volatile ForkJoinTask<?> currentSteal; // mainly used by helpStealer
```

int `nsteals` ：表示当前队列从其他队列中偷过来并执行的任务数。

int `hint`： `hint`是在`WorkerQueue`创建时初始化的，初始值为当前线程的`Probe`。当从其他`WorkerQueue`获取未执行的任务时，用来临时保存被**偷取任务的WorkerQueue的索引**。

ForkJoinTask<?>[]  `array` : 实际保存ForkJoinTask的数组。

ForkJoinPool   `pool`：关联的线程池。

ForkJoinWorkerThread `owner`：拥有此队列的ForkJoinWorkerThread。如果为共享任务队列则没有所有者。

Thread `parker`: 如果队列长期为空则会被置为未激活状态，关联的`Worker`线程也会休眠，`parker`属性就是记录关联的Worker线程，跟上面的`owner`是一样的。

ForkJoinTask<?> `currentJoin` : 必须等待`currentJoin`执行完成而让当前线程阻塞。

ForkJoinTask<?> `currentSteal` ：当前线程通过`scan`方法获取待执行的任务，该任务可能是当前队列的，也可能是其他某个队列的。

####  3.1 scanState

int `scanState` :  当前队列的状态。`scanState`是会变化的，每次改变时是在当前ctl的基础上加上`SS_SEQ`

- **偶数表示RUNNING。**(关联的Worker线程正在执行任务)
- **奇数表示SCANING。**(关联的Worker线程正在扫描获取待执行的任务)
- **负数表示INACTIVE**。

####  3.2 stackPred

int `stackPred` : 当前WorkQueue被标记成未激活状态时ctl属性的低32位的值，其中低16位保存在此之间被标记为未激活状态的WorkQueue在数组中的索引。

####  3.3 config

int `config`： 高16位保存队列的类型，是先进先出还是后进先出，取决于线程池的配置。低16位保存当前队列在队列数组中的位置。**如果小于0表示该队列是共享的**。

####  3.4 qlock

int `qlock`： 1表示已加锁，负值表示当前队列已终止，不能接受新的任务，Worker退出或者线程池终止时会将qlock属性置为-1。正常是0

####  3.5 base && top

int `base`： 下一次`poll`时对应的数组索引

int `top` : 下一次`push`时对应的数组索引

```java
push:
 top  <=1 + base  //队列是空的
 top - base >= array.length-1 // 说明队列满了
groupArray:
 top  > base  // 数组中有任务
pop:
  top >=1 + base  //有待移除的元素
poll:
  base < top   // 说明存在task
  base + 1 == top // 说明无任务了    
```

###  4. 构造函数

```java
/**
 * 该函数一共两处调用地方
 *	 第一处是 ForkJoinPool#registerWorker() new WorkQueue(this, wt);
 *	 第二处是 ForkJoinPool#externalSubmit() q = new WorkQueue(this, null);
 * submit等提交的任务构建的队列是没有工作线程的
 */
WorkQueue(ForkJoinPool pool, ForkJoinWorkerThread owner) {
    this.pool = pool;
    this.owner = owner;
    // Place indices in the center of array (that is not yet allocated)
    base = top = INITIAL_QUEUE_CAPACITY >>> 1; // 8192 / 2 = 4096
}
```

如果该队列是共享队列，那么`owoner`此时是空的。`base`和`top`两个指针分别都指向了数组的中值，即为4096。





###  5. 重要的方法

#### 5.1 push

```java
/**
 * Pushes a task. Call only by owner in unshared queues.  (The
 * shared-queue version is embedded in method externalPush.)
 *
 * @param task the task. Caller must ensure non-null.
 * @throws RejectedExecutionException if array cannot be resized
 */
final void push(ForkJoinTask<?> task) {
    ForkJoinTask<?>[] a; ForkJoinPool p;
    int b = base, s = top, n;
    //执行runWorker方法时会通过growArray方法初始化array
    //一旦关联的ForkJoinWorkerThread解除注册了则会将其置为null
    if ((a = array) != null) {    // ignore if queue removed
        int m = a.length - 1;     // fenced write for task visibility
        // 将task保存到数组中，m & s的结果就是新元素在数组中的位置
        // 如果s大于m则求且的结果是0，又从0逐步增长到索引最大值 
        U.putOrderedObject(a, ((m & s) << ASHIFT) + ABASE, task);
        //top加1
        U.putOrderedInt(this, QTOP, s + 1);
        if ((n = s - b) <= 1) { //如果原来数组是空的
            if ((p = pool) != null)
                //唤醒因为待执行任务为空而阻塞的线程
                p.signalWork(p.workQueues, this);
        }
        else if (n >= m) //说明数组满了
            growArray();
    }
}
```

这个`push`方法是提供给工作队列自己push任务来使用的，共享队列push任务是在外部`externalPush`和`externalSubmit`等方法来进行初始化和push。
 **当队列中的任务数小于1的时候，才会调用signalWork**，说明工作队列空闲。如果这个条件不满足，那么工作队列中有很多任务需要工作队列来处理，就不会触发对这个队列的窃取操作。

#### 5.2 growArray

扩容的方法。实际上这个方法有两个作用，首先是初始化，其次是判断，是否需要扩容，如果需要扩容则容量加倍。

```java
// 数组扩容，会将原老数组的元素复制到新数组中，老数组中对应的位置置为null
/**
 * Initializes or doubles the capacity of array. Call either
 * by owner or with lock held -- it is OK for base, but not
 * top, to move while resizings are in progress.
 */
final ForkJoinTask<?>[] growArray() {
    ForkJoinTask<?>[] oldA = array;
    //获取扩容后的容量（*2），如果array未初始化则是INITIAL_QUEUE_CAPACITY
    int size = oldA != null ? oldA.length << 1 : INITIAL_QUEUE_CAPACITY;
    if (size > MAXIMUM_QUEUE_CAPACITY)
        throw new RejectedExecutionException("Queue capacity exceeded");
    int oldMask, t, b;
    ForkJoinTask<?>[] a = array = new ForkJoinTask<?>[size];
    if (oldA != null && (oldMask = oldA.length - 1) >= 0 &&
        (t = top) - (b = base) > 0) {  //老数组中还有未被移除的元素
        int mask = size - 1;
        // 从base往top遍历，将老数组中的元素拷贝到新数组中，老数组对应位置的数组元素置为null
        do { // emulate poll from old array, push to new array
            ForkJoinTask<?> x;
            int oldj = ((b & oldMask) << ASHIFT) + ABASE;
            int j    = ((b &    mask) << ASHIFT) + ABASE;
            //获取原数组oldj处的元素
            x = (ForkJoinTask<?>)U.getObjectVolatile(oldA, oldj);
            //判断  x不为空 则使用unsafe进行操作
            if (x != null &&
                U.compareAndSwapObject(oldA, oldj, x, null)) //将老数组oldj处的元素置为null，将x保存到新数组的j处
                U.putObjectVolatile(a, j, x);
        } while (++b != t);
    }
    return a;
}
```

需要注意的是，这个方法一旦调用进行扩容之后，无论是来自于外部`push`操作触发，还是有工作线程worker触发，都将被锁定，之后，**不能移动top指针，但是base指针是可以移动的**。这也就是说，一旦处于扩容的过程中，就**不能新增task**，但是可以**从base进行消费**，这就只支持FIFO。因此同步模式将在此时被阻塞。

#### 5.3 pop

同样，`pop`操作也仅限于工作线程，对于共享对立中则不允许使用pop方法。这个方法将按`LIFO`后进先出的方式从队列中。从top指针处取出task

```java
//弹出上一次push的元素
/**
 * Takes next task, if one exists, in LIFO order.  Call only
 * by owner in unshared queues.
 */
final ForkJoinTask<?> pop() {
    ForkJoinTask<?>[] a; ForkJoinTask<?> t; int m;
    if ((a = array) != null && (m = a.length - 1) >= 0) {
        for (int s; (s = top - 1) - base >= 0;) { // 有未移除的元素
            // 获取上一次push的元素在数组中的索引
            long j = ((m & s) << ASHIFT) + ABASE;
            //如果这个索引处的对象为空，则退出
            if ((t = (ForkJoinTask<?>)U.getObject(a, j)) == null)
                break;
            //将j处的元素置为null，修改top属性，返回j处的元素 
            if (U.compareAndSwapObject(a, j, t, null)) {
                U.putOrderedInt(this, QTOP, s);
                return t;
            }
        }
    }
    //数组未初始化或者数组为空
    return null;
}
```

#### 5.4 poll

`poll`方法将从队列中按`FIFO`的方式取出task。

```java
// 获取并移除base对应的数组元素
/**
 * Takes next task, if one exists, in FIFO order.
 */
final ForkJoinTask<?> poll() {
    ForkJoinTask<?>[] a; int b; ForkJoinTask<?> t;
    //判断 base-top小于0说明存在task && array不为空
    // 获取并移除base对应的数组元素
    while ((b = base) - top < 0 && (a = array) != null) {
         // 获取base对应数组索引处的元素
        int j = (((a.length - 1) & b) << ASHIFT) + ABASE;
        t = (ForkJoinTask<?>)U.getObjectVolatile(a, j);
        if (base == b) {  //如果base未发生改变
            if (t != null) {
                //通过cas将其置为null，base加1，返回该元素
                if (U.compareAndSwapObject(a, j, t, null)) {
                    base = b + 1;
                    return t;
                }
            }
            //在上述操作之后，如果base比top小1说明已经为空了 直接退出循环
            else if (b + 1 == top) // now empty
                break;
        } //如果base发生改变则下一次while循环重新读取base
    }
    //默认返回null
    return null;
}
```

####  5.5 pollAt

**这个方法将采用FIFO的方式，从 队列中获得task**。通常情况下，b指的是队列的`base`指针。那么从底部获取元素就能实现FIFO。特殊的版本出现在`scan`和`helpStealer`中用于对工作队列的窃取操作的实现。

```java
//获取指定数组索引的元素，要求b等于base，否则返回null
/**
 * Takes a task in FIFO order if b is base of queue and a task
 * can be claimed without contention. Specialized versions
 * appear in ForkJoinPool methods scan and helpStealer.
 */
final ForkJoinTask<?> pollAt(int b) {
    ForkJoinTask<?> t; ForkJoinTask<?>[] a;
    if ((a = array) != null) {
        int j = (((a.length - 1) & b) << ASHIFT) + ABASE;
        //只有base等于b的时候才会将对应的数组元素置为null并返回该元素
        if ((t = (ForkJoinTask<?>)U.getObjectVolatile(a, j)) != null &&
            base == b && U.compareAndSwapObject(a, j, t, null)) {
            base = b + 1;
            return t;
        }
    }
    return null;
}
```

####  5.6 nextLocalTask

这个方法中对之前的MODE会起作用，如果是`FIFO`则用`pop`方法，反之则用`poll`方法获得下一个task。

```java
/**
 * Takes next task, if one exists, in order specified by mode.
 */
final ForkJoinTask<?> nextLocalTask() {
    return (config & FIFO_QUEUE) == 0 ? pop() : poll();
}

```

#### 5.7 peek

`peek`则根据之前的mode定义，从队列的前面或者后面取得task。

```java
/**
 * Returns next task, if one exists, in order specified by mode.
 */
final ForkJoinTask<?> peek() {
    ForkJoinTask<?>[] a = array; int m;
    // 如果数组未初始化
    if (a == null || (m = a.length - 1) < 0)
        return null;
    // 根据mode决定从top还是base处获得task
    // 如果是LIFO则是top - 1 ，如果是FIFO则是base  
    int i = (config & FIFO_QUEUE) == 0 ? top - 1 : base;
    int j = ((i & m) << ASHIFT) + ABASE;
    //获取队列头或者队列尾的元素，取决于队列的类型
    return (ForkJoinTask<?>)U.getObjectVolatile(a, j);
}
```

#### 5.8 tryUnpush

这个方法是将之前`push`的任务撤回。这个操作仅仅只有**task位于top**的时候操能成功。

```JAVA
/**
 * Pops the given task only if it is at the current top.
 * (A shared version is available only via FJP.tryExternalUnpush)
*/
//如果t是上一次push的元素，则将top减1，对应的数组元素置为null，返回true
final boolean tryUnpush(ForkJoinTask<?> t) {
    ForkJoinTask<?>[] a; int s;
    if ((a = array) != null && (s = top) != base &&
        //将top位置的task与t比较
        U.compareAndSwapObject
         //如果t是上一次push插入的元素，则将其置为null，同时修改s
        (a, (((a.length - 1) & --s) << ASHIFT) + ABASE, t, null)) {
        //将top减1
        U.putOrderedInt(this, QTOP, s);
        return true;
    }

    return false;
}
```

####  5.9 runTask

   之前的文章分析外部提交task的时候，就提到了这个方法。实际上是`runWorker`调用的。
也就是说，线程在启动之后，一旦worker获取到task，就会运行。

`runTask`方法会执行指定的任务，并且将任务数组中包含的所有未处理任务都执行完成，注意指定的任务不一定是当前`WorkQueue`中包含的，有可能是从其他`WorkQueue`中偷过来的。

```java
/**
 * Executes the given task and any remaining local tasks.
 */
//执行特定任务，并执行完当前数组中包含的所有未处理任务
final void runTask(ForkJoinTask<?> task) {
    if (task != null) {
        //SCANNING的值就是1,将最后一位置为0，表示正在执行任务
        scanState &= ~SCANNING; // mark as busy
        //执行这个task
        (this.currentSteal = task).doExec();
        // 将currentSteal属性置为null
        U.putOrderedObject(this, QCURRENTSTEAL, null); // release for GC
        // 执行当前数组中包含的未处理的任务
        execLocalTasks();
        ForkJoinWorkerThread thread = owner;
        // nsteals已经超过最大值了，变成负值了，则将其累加到ForkJoinPool的stealCounter属性中 并重置为0
        if (++nsteals < 0)      // collect on overflow
            transferStealCount(pool);
        //将scanState改为1 这样就变得活跃可以被其他worker scan
        scanState |= SCANNING;
        //如果thread不为null说明为worker线程 则调用后续的exec方法
        if (thread != null)
            thread.afterTopLevelExec();
    }
}

```

####  5.10 execLocalTasks

调用这个方法，运行队列中的全部task，如果采用了`FIFO`模式，则调用`pollAndExecAll`，这是另外一种实现方法。直到将队列都执行到empty

```java
/**
 * Removes and executes all local tasks. If LIFO, invokes
 * pollAndExecAll. Otherwise implements a specialized pop loop
 * to exec until empty.
 */
final void execLocalTasks() {
    int b = base, m, s;
    ForkJoinTask<?>[] a = array;
    // 如果数组已经初始化且包含未处理的元素
    if (b - (s = top - 1) <= 0 && a != null &&
        (m = a.length - 1) >= 0) {
        //如果是LIFO，从top开始往前遍历直到base
        if ((config & FIFO_QUEUE) == 0) {
           //开始循环
            for (ForkJoinTask<?> t;;) {
               //从top开始取出task
                if ((t = (ForkJoinTask<?>)U.getAndSetObject
                     (a, ((m & s) << ASHIFT) + ABASE, null)) == null)
                    break;
                //修改top
                U.putOrderedInt(this, QTOP, s);
                t.doExec(); //执行任务
                //如果没有任务的了 则退出
                if (base - (s = top - 1) > 0)
                    break;
            }
        }
        else
           //FIFO的方式调用pollAndExecAll
            pollAndExecAll();
    }
}
```

####  5.11 pollAndExecAll

此方法将用poll，FIFO的方式获得task并执行。

```JAVA
  final void pollAndExecAll() {
        for (ForkJoinTask<?> t; (t = poll()) != null;)
            t.doExec();
    }
```

####  5.12 tryRemoveAndExec

从注释中可知，这个方法仅仅供`awaitJoin`方法调用，在await的过程中，将**task从workQueue中移除并执行**。

```JAVA
/**
 * If present, removes from queue and executes the given task,
 * or any other cancelled task. Used only by awaitJoin.
 *
 * @return true if queue empty and task not known to be done
 */
//如果队列为空且不知道task的状态，则返回true
//如果找到目标task，会将其从数组中移除并执行
//在遍历数组的过程中，如果发现任务被取消了，则会将其从数组中丢弃
final boolean tryRemoveAndExec(ForkJoinTask<?> task) {
    ForkJoinTask<?>[] a; int m, s, b, n;
    if ((a = array) != null && (m = a.length - 1) >= 0 &&
        task != null) { //数组和task非空
        while ((n = (s = top) - (b = base)) > 0) {  //还有未处理的任务
            //死循环 从top遍历到base
            for (ForkJoinTask<?> t;;) {      // traverse from s to b
                 //s先减1再求且
                long j = ((--s & m) << ASHIFT) + ABASE;
                 //如果索引为j的数组元素为nul
                if ((t = (ForkJoinTask<?>)U.getObject(a, j)) == null) 
                    return s + 1 == top;  //如果为true，说明数组是空的 shorter than expected
                 //找到了匹配的任务
                else if (t == task) {
                    boolean removed = false;
                    if (s + 1 == top) {   // 目标task就是上一次push的元素  pop
                        //pop的方式获取task  然后替换为null
                        if (U.compareAndSwapObject(a, j, task, null)) {
                            //将对应的数组元素置为null，然后修改top属性 top-1
                            U.putOrderedInt(this, QTOP, s);
                            removed = true;
                        }
                    }
                    // s+1不等于top，即目标task位于中间的位置，将用一个空的EmptyTask代替
                    else if (base == b)      // replace with proxy
                        removed = U.compareAndSwapObject(
                            a, j, task, new EmptyTask());
                    if (removed)  //如果成功移除该task，则执行该任务
                        task.doExec();
                    break;  //终止内存for循环，通过外层的while循环，重启for循环
                }
                //t不等于task
                //如果t被取消了，且t位于栈顶
                else if (t.status < 0 && s + 1 == top) {
                    if (U.compareAndSwapObject(a, j, t, null))
                        U.putOrderedInt(this, QTOP, s); //修改top属性，即将被取消掉的任务给丢掉
                    break;                  // was cancelled
                }
                //先减n减1，再判断n是否为0，如果为0，说明数组中所有元素都遍历完了，返回false
                if (--n == 0)
                    return false;
            } //for循环结束
            if (task.status < 0) //task被取消了，返回false
                return false;
        } //while循环结束
    }
    return true;
}
```

####  5.13 popCC

如果`pop`  `CountedCompleter`。这方法支持共享和worker的队列，但是仅仅通过`helpComplete`调用。
 `CountedCompleter`是jdk1.8中新增的一个ForkJoinTask的一个实现类。

```java
//如果栈顶的元素是CountedCompleter，则以该元素为起点向上遍历他的父节点，直到找到目标task元素，将栈顶的
//CountedCompleter从数组中移除并返回，top减1
/**
 * Pops task if in the same CC computation as the given task,
 * in either shared or owned mode. Used only by helpComplete.
 * @param Mode 为workQueue的config
 */
final CountedCompleter<?> popCC(CountedCompleter<?> task, int mode) {
    int s; ForkJoinTask<?>[] a; Object o;
    if (base - (s = top) < 0 && (a = array) != null) { //如果数组非空
        //从top处开始
        long j = (((a.length - 1) & (s - 1)) << ASHIFT) + ABASE;
        if ((o = U.getObjectVolatile(a, j)) != null &&
            // 如果栈顶的任务是CountedCompleter
            (o instanceof CountedCompleter)) {
            CountedCompleter<?> t = (CountedCompleter<?>)o;
            for (CountedCompleter<?> r = t;;) {
                if (r == task) { //如果找到目标task
                    if (mode < 0) { //该队列是共享的，会被多个线程修改，必须加锁 must lock
                        if (U.compareAndSwapInt(this, QLOCK, 0, 1)) { //加锁
                            if (top == s && array == a && //如果top属性和array属性都未改变
                                U.compareAndSwapObject(a, j, t, null)) {//索引为j元素设为null
                                U.putOrderedInt(this, QTOP, s - 1); //top减1
                                U.putOrderedInt(this, QLOCK, 0);  //释放锁
                                return t;
                            }
                             //上述if不成立，解锁
                            U.compareAndSwapInt(this, QLOCK, 1, 0);
                        }
                    }
                    //mode大于等于0，是某个Worker线程独享的，不需要加锁
                    else if (U.compareAndSwapObject(a, j, t, null)) { //将索引为j的元素置null
                        U.putOrderedInt(this, QTOP, s - 1);
                        return t;
                    }
                    break;
                }
                else if ((r = r.completer) == null) //遍历父节点，如果为null说明都遍历完了 try parent
                    break;
            }
        }
    }
    return null;
}
```

####  5.14 pollAndExecCC

窃取并运行与给定任务相同`CountedCompleter`计算任务（如果存在），并且可以在不发生争用的情况下执行该任务。否则，返回一个校验和/控制值，供`helpComplete`方法使用。

```java
//如果base对应的数组元素是CountedCompleter，则以该节点作为起点，往上遍历其父节点，如果找到目标节点task，则执行并返回1     
/**
 * Steals and runs a task in the same CC computation as the
 * given task if one exists and can be taken without
 * contention. Otherwise returns a checksum/control value for
 * use by method helpComplete.
 *
 * @return 1 if successful, 2 if retryable (lost to another
 * stealer), -1 if non-empty but no matching task found, else
 * the base index, forced negative.
 */
final int pollAndExecCC(CountedCompleter<?> task) {
    int b, h; ForkJoinTask<?>[] a; Object o;
    //判断array的合法性
    if ((b = base) - top >= 0 || (a = array) == null) //如果数组为空
        h = b | Integer.MIN_VALUE;  // to sense movement on re-poll
    else {
        //如果数组不为空，获取base对应的元素
        long j = (((a.length - 1) & b) << ASHIFT) + ABASE;
        if ((o = U.getObjectVolatile(a, j)) == null) //如果base对应的元素为null，返回2
            h = 2;                  // retryable
        else if (!(o instanceof CountedCompleter)) //如果base对应的元素不是CountedCompleter，返回-1
            h = -1;                 // unmatchable
        else {
            //base对应的元素是CountedCompleter
            CountedCompleter<?> t = (CountedCompleter<?>)o;
            //死循环
            for (CountedCompleter<?> r = t;;) {
                if (r == task) {
                    //匹配到目标元素
                    if (base == b &&
                        U.compareAndSwapObject(a, j, t, null)) {
                        //base未改变且成功将j对应的数组元素置为null，将base加1，并执行该任务，返回1
                        base = b + 1;
                        t.doExec();
                        h = 1;      // success
                    }
                    else
                        //base变了或者cas失败，返回2
                        h = 2;      // lost CAS
                    break;
                }
                else if ((r = r.completer) == null) { //往上遍历父节点，如果为null，则返回-1
                    h = -1;         // unmatched
                    break;
                }
            }
        }
    }
    return h;
}
```

###  6. 总结

本文对`workQueue`的源码进行了分析，我们需要注意的是，对于workQueue，定义了三个操作，分别是`push`，`poll`和`pop`。

- **push** : 主要是操作top指针，将top进行移动。

<img src="http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/3237432-b79d52280dfeed6a.png" alt="img" style="zoom: 80%;" />

- **poll** : 如果top和base不等，则说明队列有值，可以消费，那么poll就从base指针处开始消费。这个方法实现了队列的FIFO。消费之后对base进行移动。

<img src="http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/3237432-1bb16fe06ce795d3.png" alt="img" style="zoom:80%;" />

- **pop** : 同样，还可以从top开始消费，这就是pop。这个方法实际上实现了对队列的LIFO。消费之后将top减1。

<img src="http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/3237432-dc492ab819e31e15.png" alt="img" style="zoom:80%;" />

以上就是这三个方法对应的操作。但是我们还需要注意的是，在所有的unsafe操作中，通过cas进行设置或者获得task的时候，还有一个掩码。这个非常重要。
我们可以看在`push`方法中：

```java
 int m = a.length - 1;
 U.putOrderedObject(a, ((m & s) << ASHIFT) + ABASE, task);
```

在扩容的方法`growArray`中我们可以知道。每次扩容都是采用左移的方式来进行，这样就保证了数组的长度为2的幂。
 在这里，m=a.length-1，那就说明，m实际上其二进制格式将会有效位都为1,这个数字就可以做为掩码。当m再与s取&计算的时候。可以想象，s大于m的部分将被去除，只会保留比m小的部分。那么实际上，这就等价于，当我们一直再`push`元素到数组中的时候，实际上就从数组的索引底部开始：

![img](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/3237432-fbd6a19840ff0ebe.png)

```
	参考上面这个过程，也就是说，实际上这个数组，**base和top**实际指向的index并不重要。只有**二者的相对位移**才是重要的。这有点类似与RingBuffer的数据结构，但是还是有所不同。也就是说这个数组实际上是不会被浪费的。之前有很多不理解的地方，为什么top减去base可能出现负数。那么这样实际上就会导致负数的产生。
```
 这样的话，如果我们采用异步模式，`asyncMode`为true的时候，workQueue则会采用`FIFO_QUEUE`的mode，这样workQueue本身就使用的时`poll`方法。反之如果使用`LIFO_QUEUE`的同步模式，则workQueue使用`pop`方法。默认情况下采用**同步模式**。**同步的时候workQueue的指针都围绕在数组的初始化的中间位置波动。而共享队列则会一直循环**。
 		至此，我们分析了workQueue的源码，对其内部实现的双端队列本身的操作进行了分析。为什么作者会自己实现一个Deque，而不是使用juc中已存在的容器。这就是因为这个队列全程都是采用Unsafe来实现的，在开篇作者也说了，需要@Contented修饰，就是为了避免缓存的伪代共享。这样来实现一个高效的Deque，以供ForkJoinPool来操作。

## 四.ForkJoinTask源码分析

### 1. 类结构及其成员变量

​	`ForkJoinTask`是在`ForkJoinPool`中运行task的基础抽象类，ForkJoinTask是类似于线程的实体，其权重比普通线程要轻得多。大量的task或者task的子类可能由ForkJoinPool中实际的线程来托管，但以某些使用限制为代价。
 	一个main的`ForkJoinTask`被提交给ForkJoinPool的时候，**如果尚未参与ForkJoin计算，则通过ForkJoinPool＃commonPool()中fork或者invoke方法开始**。一旦启动，通过将依次启动其他子任务。如此类的名称所示，许多使用了`ForkJoinTask`的程序仅采用`fork`或者诸如`ivokeAll`。但是，此类还提供了许多其他可以在高级方法中使用的方法，以及允许支持新形式的fork/join处理的扩展机制。
 	`ForkJoinTask`是Future的轻量级形式，`ForkJoinTask`的效率源于一组限制条件，这些限制只能部分静态的强制执行，反映出它们的主要用途是作为计算纯函数或对纯函数隔离的对象进行的操作的计算任务。**主要协调机制是fork，用于安排异步执行和join，在计算任务结果之前不会执行**。理想情况下，**计算应避免使用sync方法块，并应用除加入其他任务或使用被宣传为fork/join的调度配合使用的诸如Phasers之类的同步器之外的其他最小化同步阻塞**。**可细分的任务也不应执行阻塞的I/O，并且理想情况下应访问与其他正在运行的任务访问的变量完全独立的变量**。不允许抛出诸如`IOExeption`之类的检查异常。从而松散的实现了这些准则，但是，计算可能任会遇到未经检查的异常，这些异常会被尝试加入它的调用者重新抛出。这些异常可能还包括源自内部资源耗尽，例如无法分配任务队列 `RejectedExecutionException`。重新引发的异常的行为与常规异常相同，但是在可能的情况下，包含启动计算的线程以及实际遇到的线程的堆栈跟踪（例如，使用ex.printStackTrace()显示）异常；最少只有后者。
 	可以定义和使用可能阻塞的`ForkJoinTasks`,但是这样还需要三点考虑：

1. 如果有other个任务，则应该完成少数几个依赖于在外部同步或者I/O，从未加入的事件样例的异常任务，例如，子类为CountedCompleter的哪些子任务通常属于此类。
2. 为了最大程度的减少资源的影响，任务应该很小。理想情况下，仅执行组织操作。
3. 除非使用`ForkJoinPoolManagedBlocker `API，或者已知可能被阻止的任务数小于pool的ForkJoinPool的getParallelism级别，否则pool无法保证有足够的线程可用来确保进度的良好表现。

**等待完成和提取任务结果的主要方法是join**，但是有几种变体，**get方法支持中断或定时等待完成，并使用Future约定**，**方法invoke在语义上等效于fork+join，当时始终尝试在当前线程中开始执行**，这些方法的quiet形式不会提取结果或报告异常，当执行一组任务的时候，这些选项可能有用，并且你需要将结果或异常的处理延时到所有任务为止。**方法invokeAll有多个版本，执行并调用的最常见的形式：分派一组任务将它们全部加入**。
    在最典型的用法中，fork-join对的作用类似于调用fork，并从并行递归中返回join，与其他形式的递归调用一样，返回应从最里面开始执行。例如:

```java
a.fork();
b.fork();
b.join();
a.join();
```

可能比在b之前加入a的效率更高。

​	可以在多个详细级别查询任务的执行状态：如果任务以任何方式完成（包括任务被取消而不执行的情况），则`isDone`为真； `isCompletedNormally`如果任务完成而没有取消或遇到异常，则为真； 如果任务被取消，则`isCancelled`为真（在这种情况下getException返回一个`CancellationException `）； 如果任务被取消或遇到异常，则`isCompletedAbnormally`为真，在这种情况下`getException`将返回遇到的异常或CancellationException 。

 `ForkJoinTask`类通常不直接子类化，相反，你可以将一个支持特定样势的fork/join处理的抽象类做为子类，对于大多数不返回结果的计算，通常使用RecursiveAction。对于哪些返回结果的计算，则通常使用RecursiveTask。并使用CountedCompleter 。其中已完成的操作会触发其他操作，通常一个具体的ForkJoinTask子类申明在构造函数中建立的包含其参数的字段，然后定义了一个compute方法。该方法以某种方式使用此基类提供的控制方法。
 方法join及其变体仅在完成依赖项是非循环的时候才适用，也就是说，并行计算可以描述为有向无环图DAG。否则，执行可能会遇到死锁，因为任务的周期性的互相等待。但是，此框架支持其他方法和技术，如Phaser的helpQuiesce和complete来构造针对非自定义问题的自定义子类，静态结构为DAG，为了支持此方法，可以适用setForkJoinTaskTag或者compareAndSetForkJoinTaskTag或者compareAndSetForkJoinTaskTag。适用short值自动标记ForkJoinTask并使用getForkJoinTaskTag进行检查。ForkJoinTask实现不出任何目的的使用这些protected的方法或标记，但是可以在构造函数中专门的子类中使用它们，例如，并行图遍历可以使用提供的方法来避免重访问已处理的节点/任务。用于标记的方法名称很大一部分是为了鼓励定义反映其使用方式的方法。
 大多数基本的支持的方法都是final，以防止覆盖与底层轻量级任务计划框架固有的联系的实现。创新的fork/join处理基本样式的开发人员应最小的实现protected的exec方法、setRawResult和getRawResult方法。同时还引入了一种抽象的计算方法，该方法可以在其子类中实现，可能依赖于此类提供的其他protected方法。
 ForkJoinTasks应该执行相对少量的计算，通常应该通过递归分解将大型任务分解为较小的子类，做为一个非常粗略的经验法则，任务执行100个以上且少于10000个基本计算步骤，并应避免无限循环。如果任务太大，则并行度无法提高吞吐量，如果太小，则内存和内部任务维护开销可能会使处理不堪重负。
 此类为Runnable和Callable提供了Adapt方法。这些方法将在ForkJoinTasks执行与其他类型的任务混合使用的时候可能会有用。当所有任务都具有这种形式的时候，请考虑使用asyncMode 中构造的池。
 ForkJoinTasks是可序列化的，使得它们可以在远程执行的框架中扩展使用，仅在执行前或者之后而不是之前期间序列化任务是明智的。执行本身不依赖于序列化。

```java
public abstract class ForkJoinTask<V> implements Future<V>, Serializable {}
```

在代码中，任有不少注释：
 有关通用的实现概述，请参见类ForkJoinPool的内部文档。在对ForkJoinWorkerThread和ForkJoinPool中的方法进行中继的过程中，ForkJoinTasks主要负责维护其“状态”字段。
 此类的大致方法分为：

- 1.基于状态维护
- 2.执行和等待完成
- 3.用户级方法（另外报告结果）

有时很难看到，因为此文件以在Javadocs中良好流动的方式对导出的方法进行排序。
 状态字段将运行控制状态位打包为单个int，以最大程度的减少占用空间并确保原子性，（通过cas）。Status最初的状态为0，并采用非负值，直到完成为止。此后，与DONE_MASK一起，状态的值保持为NORMAL，CANCELLED或者EXCEPTIONAL，其他线程正在等待阻塞任务，将SIGNAL位置设置为1。设置了SIGNAL的被窃取的任务完成后，将通过notifyAll唤醒任何等待的线程。 即使出于某些目的不是最优的，我们还是使用基本的内置的wait/notify来利用JVM中的Monitor膨胀。否则我们将需要模拟JVM以避免增加每个任务的记录开销。我们希望这些Monitor是fat的，即不需要使用偏向锁和细粒度的锁，因此要使用一些奇怪的编码习惯来避免它们，主要是安排每个同步块执行一个wait，notifyAll，或者两者都执行。
 这些控制位仅占用状态字段的上半部分中的一部分，低位用于用户定义的标签。

###  2.变量及常量

在`ForkJoinTask`中，主要的变量就一个，为`status`。

```java
// 任务的状态，初始值为0，大于等于0时表示任务未执行或者正在执行的过程中，小于0表示已执行完成
/** The run status of this task */
volatile int status; // accessed directly by pool
```

这是通过colatile修饰的int类型，用以标识每个task的运行状态。
这些状态主要是一下这些常量。

```java
 and workers
static final int DONE_MASK   = 0xf0000000;  // mask out non-completion bits
static final int NORMAL      = 0xf0000000;  // must be negative 正常完成
static final int CANCELLED   = 0xc0000000;  // must be < NORMAL 任务被取消了
static final int EXCEPTIONAL = 0x80000000;  // must be < CANCELLED 任务异常终止

// 某个线程在等待当前任务执行完成，需要在任务结束时唤醒等待的线程
static final int SIGNAL      = 0x00010000;  // must be >= 1 << 16
static final int SMASK       = 0x0000ffff;  // short bits for tags 获取低16位的值
```



![img](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/3237432-1195b9856d1d5baf.png)

### 3. 实现方法

#### 3.1 抽象方法

ForkJoinTask是ForkJoinPool中提交任务的主要实现类，需要注意的是，这个类是个抽象类。可扩展的方法如下：

```java

/**
 * Returns the result that would be returned by {@link #join}, even
 * if this task completed abnormally, or {@code null} if this task
 * is not known to have been completed.  This method is designed
 * to aid debugging, as well as to support extensions. Its use in
 * any other context is discouraged.
 *
 * @return the result, or {@code null} if not completed
 */
public abstract V getRawResult();

/**
 * Forces the given value to be returned as a result.  This method
 * is designed to support extensions, and should not in general be
 * called otherwise.
 *
 * @param value the value
 */
protected abstract void setRawResult(V value);

/**
 * Immediately performs the base action of this task and returns
 * true if, upon return from this method, this task is guaranteed
 * to have completed normally. This method may return false
 * otherwise, to indicate that this task is not necessarily
 * complete (or is not known to be complete), for example in
 * asynchronous actions that require explicit invocations of
 * completion methods. This method may also throw an (unchecked)
 * exception to indicate abnormal exit. This method is designed to
 * support extensions, and should not in general be called
 * otherwise.
 *
 * @return {@code true} if this task is known to have completed normally
 */
protected abstract boolean exec();
```

这些抽象方法，总结如下表：

| 方法           | 说明                                                         |
| -------------- | ------------------------------------------------------------ |
| getRawResult() | 此方法将返回`join`的返回结果，即使此任务出现异常，或者返回null（再该任务尚未完成的情况下返回），此方法旨在帮助调试以及支持扩展，不建议在其他任何上下文中使用 |
| setRawResult() | 强制将给定值做为结果返回，此方法旨在支持扩展，通常不应以其他的方式调用。 |
| exec()         | 立即执行此任务的基本操作，如果从该方法返回后，保证此任务已正常完成，则返回true，否则，此方法可能返回false，以表示此任务不一定完成，或不知道是完成的，例如在需要显示调用完成方法的异步操作中。此方法还可能引发未经检查的异常以指示异常退出。此方法旨在支持扩展，通常不应以其他方式调用。 |

​	实际上，这些抽象方法都是提供给子类进行实现的。如`RecursiveTask`和`RecursiveAction`就实现了这些方法，之后这**两个类将compute方法提供给子类继续扩展**。而上述的抽象方法，则不会提供给子类扩展。我们在使用的过程中，只需要直到这三个方法的大致含义即可。我们使用`RecursiveTask`和`RecursiveAction`这两个类只需要再实现`compute`方法。

#### 3.2 实现方法

`ForkJoinTask`是Future接口的实现类，因此，实现Future的方法，实际上就是当外部将ForkJoinTask当作Future的时候需要调用的方法。

##### 3.2.1 get

​	`get`方法是阻塞当前线程并等待任务执行完成，其效果和实现跟`join`方法基本一致，最大的区别在于**如果线程等待的过程中被中断了，get方法会抛出异常InterruptedException，而join方法不会抛出异常**，其实现如下：

```java
public final V get() throws InterruptedException, ExecutionException {
    //如果是ForkJoinWorkerThread执行doJoin 否则执行externalInterruptibleAwaitDone
    int s = (Thread.currentThread() instanceof ForkJoinWorkerThread) ?
        doJoin() :
        externalInterruptibleAwaitDone();
    Throwable ex;
    if ((s &= DONE_MASK) == CANCELLED) //任务被取消
        throw new CancellationException();
    if (s == EXCEPTIONAL && (ex = getThrowableException()) != null)//异常终止 并重新包装异常
        throw new ExecutionException(ex);
    //默认返回结果是getRawResult。
    return getRawResult();
}
```

##### 3.2.2 get(long timeout, TimeUnit unit)

根据传入的等待时间进行等待，之后再获取结果。

```java
/**
 * Waits if necessary for at most the given time for the computation
 * to complete, and then retrieves its result, if available.
 *
 * @param timeout the maximum time to wait
 * @param unit the time unit of the timeout argument
 * @return the computed result
 * @throws CancellationException if the computation was cancelled
 * @throws ExecutionException if the computation threw an
 * exception
 * @throws InterruptedException if the current thread is not a
 * member of a ForkJoinPool and was interrupted while waiting
 * @throws TimeoutException if the wait timed out
 */
public final V get(long timeout, TimeUnit unit)
    throws InterruptedException, ExecutionException, TimeoutException {
    int s;
    long nanos = unit.toNanos(timeout);
    if (Thread.interrupted()) //被中断抛出异常
        throw new InterruptedException();
    if ((s = status) >= 0 && nanos > 0L) {
        // //获取等待的终止时间
        long d = System.nanoTime() + nanos;
        long deadline = (d == 0L) ? 1L : d; // avoid 0
        Thread t = Thread.currentThread();
        // 如果是ForkJoinWorkerThread，通过awaitJoin方法等待任务执行完成
        if (t instanceof ForkJoinWorkerThread) {
            ForkJoinWorkerThread wt = (ForkJoinWorkerThread)t;
            s = wt.pool.awaitJoin(wt.workQueue, this, deadline);
        }
        // 如果是普通Java线程
        else if ((s = ((this instanceof CountedCompleter) ?
                       //如果是CountedCompleter，则通过externalHelpComplete等待其执行完成
                       ForkJoinPool.common.externalHelpComplete(
                           (CountedCompleter<?>)this, 0) :
                      //如果是普通的ForkJoinTask，尝试将其从任务队列中pop出来并执行
                       doExec() : 0)) >= 0) {
            //如果tryExternalUnpush返回false或者doExec方法返回值大于等于0，即任务未执行完成
            long ns, ms; // measure in nanosecs, but wait in millisecs
            while ((s = status) >= 0 &&  //任务已执行
                   (ns = deadline - System.nanoTime()) > 0L) { //等待超时
                if ((ms = TimeUnit.NANOSECONDS.toMillis(ns)) > 0L && 
                    U.compareAndSwapInt(this, STATUS, s, s | SIGNAL)) {
                    //cas修改状态加上SIGNAL
                    synchronized (this) {
                        if (status >= 0)
                            //阻塞当前线程指定时间，如果被中断则抛出异常
                            wait(ms); // OK to throw InterruptedException
                        else
                            notifyAll();
                    }
                }
            }
        }
    }
    if (s >= 0)
        s = status; //再次读取状态
    if ((s &= DONE_MASK) != NORMAL) { //不是正常执行
        Throwable ex;
        if (s == CANCELLED) //被取消
            throw new CancellationException();
        if (s != EXCEPTIONAL) //不是异常终止，则是等待超时
            throw new TimeoutException();
        if ((ex = getThrowableException()) != null) //异常终止
            throw new ExecutionException(ex);
    }
    return getRawResult();
}
```

###### 3.2.2.1 externalInterruptibleAwaitDone

此方法的目的是将非工作线程阻塞，直至执行完毕或者被打断。期间有可能会触发窃取操作。

```java
//逻辑同externalAwaitDone，区别在于如果被中断抛出异常
//externalAwaitDone不会抛出异常，如果被中断了会将当前线程标记为已中断
/**
 * Blocks a non-worker-thread until completion or interruption.
 */
private int externalInterruptibleAwaitDone() throws InterruptedException {
    int s; 
    if (Thread.interrupted()) //被中断则抛出异常
        throw new InterruptedException();
    if ((s = status) >= 0 &&
        (s = ((this instanceof CountedCompleter) ? 
              ForkJoinPool.common.externalHelpComplete((CountedCompleter<?>)this, 0) :
              ForkJoinPool.common.tryExternalUnpush(this) ? doExec() :
              0)) >= 0) {
        while ((s = status) >= 0) {
            if (U.compareAndSwapInt(this, STATUS, s, s | SIGNAL)) {
                synchronized (this) {
                    if (status >= 0)
                        wait(0L);
                    else
                        notifyAll();
                }
            }
        }
    }
    return s;
}
```

##### 3.2.4 其他实现方法

```java
/**
 * Attempts to cancel execution of this task. This attempt will
 * fail if the task has already completed or could not be
 * cancelled for some other reason. If successful, and this task
 * has not started when {@code cancel} is called, execution of
 * this task is suppressed. After this method returns
 * successfully, unless there is an intervening call to {@link
 * #reinitialize}, subsequent calls to {@link #isCancelled},
 * {@link #isDone}, and {@code cancel} will return {@code true}
 * and calls to {@link #join} and related methods will result in
 * {@code CancellationException}.
 *
 * <p>This method may be overridden in subclasses, but if so, must
 * still ensure that these properties hold. In particular, the
 * {@code cancel} method itself must not throw exceptions.
 *
 * <p>This method is designed to be invoked by <em>other</em>
 * tasks. To terminate the current task, you can just return or
 * throw an unchecked exception from its computation method, or
 * invoke {@link #completeExceptionally(Throwable)}.
 *
 * @param mayInterruptIfRunning this value has no effect in the
 * default implementation because interrupts are not used to
 * control cancellation.
 *
 * @return {@code true} if this task is now cancelled
 */
public boolean cancel(boolean mayInterruptIfRunning) {
    return (setCompletion(CANCELLED) & DONE_MASK) == CANCELLED;
}

public final boolean isDone() {
    return status < 0;
}

public final boolean isCancelled() {
    return (status & DONE_MASK) == CANCELLED;
}
```

实际上这些方法都是对status状态进行判断或者修改操作。当task执行完毕之后status小于0。`cancel`参照实际上是set status的状态为`CANCELLED`。

#### 3.3 外部调用方法

##### 3.3.1 fork

​	在当前任务正在执行的pool中异步执行此任务，如果不是在`ForkJoinPool`中执行，则使用`ForkJoinPool的commonPool`。尽管不一定要强制执行，但是如果任务已完成，并重新初始化，则多次fork任务是错误的。除非调用join或者相关方法。或者调用isDone，否则，执行该任务的状态或执行该操作的任何数据的后续修改不一定可由执行该任务的线程以外的任何线程一致地观察到。

```java
//fork方法并不是如其命名会创建一个新线程来执行任务，只是将任务提交到任务队列中而已
public final ForkJoinTask<V> fork() {
    Thread t;
    if ((t = Thread.currentThread()) instanceof ForkJoinWorkerThread)
        //如果当前线程是ForkJoinWorkerThread，将其提交到关联的WorkQueue中
        ((ForkJoinWorkerThread)t).workQueue.push(this);
    else
        //如果是普通线程，则提交到common线程池的任务队列中
        ForkJoinPool.common.externalPush(this);
    return this;
}
```

###### 3.3.1.1 测试用例

```java
 	@Test
    public void test() throws Exception {
        ForkJoinTask task=ForkJoinTask.adapt(new Runnable() {
            @Override
            public void run() {
                try {
                    Thread.sleep(2000);
                    System.out.println(Thread.currentThread().getName()+" exit");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
        //提交任务到common线程池
        task.fork();
        //阻塞等待任务执行完成，get方法会将该任务从任务队列中pop出来并执行
        //所以run方法打印出来的线程名就是main
        task.get();
        System.out.println("main thread exit");
    }

// 在AdaptedRunnableAction对应的exec()方法打下断点 然后debug，观察调用链如下：
```

![img](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/20200513161837494.png)

##### 3.3.2 join / quietlyJoin

​	`Join `: 返回`isDone`的时候的计算结果，此方法与get的不同之处在于，异常完成会导致`RuntimeException`。或者`Error`,而不是`ExecutionException`。且调用线程的中断不会导致方法通过抛出`InterruptedException`而突然返回。

`quietlyJoin` : 加入此任务，不返回结果或抛出异常。 当某些已被取消或以其他方式已知已中止时处理任务集合时，此方法可能很有用。

```java
 // join方法用于阻塞当前线程，等待任务执行完成，部分情形下会通过当前线程执行任务，如果异常结束或者被取消需要抛出异常；
/**
 * Returns the result of the computation when it {@link #isDone is
 * done}.  This method differs from {@link #get()} in that
 * abnormal completion results in {@code RuntimeException} or
 * {@code Error}, not {@code ExecutionException}, and that
 * interrupts of the calling thread do <em>not</em> cause the
 * method to abruptly return by throwing {@code
 * InterruptedException}.
 *
 * @return the computed result
 */
public final V join() {
    int s;
     //doJoin方法会阻塞当前线程直到任务执行完成并返回任务的状态
    if ((s = doJoin() & DONE_MASK) != NORMAL)
        //如果不是正常完成的，则报告异常
        reportException(s);
    //返回执行结果,该方法是抽象方法 
    return getRawResult();
}


// quietlyJoin方法只是阻塞当前线程等待任务执行完成，不会抛出异常
/**
 * Joins this task, without returning its result or throwing its
 * exception. This method may be useful when processing
 * collections of tasks when some have been cancelled or otherwise
 * known to have aborted.
 */
 public final void quietlyJoin() {
     doJoin();  //只是等待任务执行完成
 }

```

###### 3.3.2.1 doJoin

```java
/**
 * Implementation for join, get, quietlyJoin. Directly handles
 * only cases of already-completed, external wait, and
 * unfork+exec.  Others are relayed to ForkJoinPool.awaitJoin.
 *
 * @return status upon completion
 */
private int doJoin() {
    int s; Thread t; ForkJoinWorkerThread wt; ForkJoinPool.WorkQueue w;
    //status小于0说明任务已结束，直接返回 
    return (s = status) < 0 ? s :
        ((t = Thread.currentThread()) instanceof ForkJoinWorkerThread) ?
         //如果当前线程是ForkJoinWorkerThread，执行tryUnpush，返回true以后执行doExec
         //如果tryUnpush返回false或者doExec返回大于等于0，则执行awaitJoin
          //如果this在任务队列的顶端，tryUnpush会将其pop出来，返回true，否则返回false
        (w = (wt = (ForkJoinWorkerThread)t).workQueue).
        tryUnpush(this) && (s = doExec()) < 0 ? s :
        //awaitJoin也是阻塞当前线程，直到任务执行完成
        wt.pool.awaitJoin(w, this, 0L) :
        ///如果当前线程是普通的Java线程
        externalAwaitDone();
}
```

###### 3.3.2.2 doExce

```java
//执行任务，doExec方法的返回值取决于exec方法，如果exec返回true，则doExec返回值小于0
//如果返回false，则doExec返回值大于等于0
final int doExec() {
    int s; boolean completed;
    if ((s = status) >= 0) {
        try {
            //exec是子类实现的方法
            completed = exec();
        } catch (Throwable rex) {
            //执行异常
            return setExceptionalCompletion(rex);
        }
        if (completed)
            //正常完成
            s = setCompletion(NORMAL);
    }
    return s;
}
```

###### 3.3.2.3 测试用例

```java
@Test
public void test() throws Exception {
    ForkJoinTask task=ForkJoinTask.adapt(new Runnable() {
        @Override
        public void run() {
            try {
                Thread.sleep(2000);
                System.out.println(Thread.currentThread().getName()+" exit");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    });
    //提交任务到common线程池
    task.fork();
    //阻塞等待任务执行完成，join方法会将该任务从任务队列中pop出来并执行
    //所以run方法打印出来的线程名就是main
    task.join();
    System.out.println("main thread exit");
}
```

##### 3.3.3 invoke / quietlyInvoke

​	 `invoke`**会立即执行当前任务**，如果doExec方法返回值大于等于0说明还有其他的子任务未完成，则等待其他子任务执行完成，典型的应用场景就是`CountedCompleter`，`RecursiveAction`和`RecursiveTask`通常`doExec`返回值小于0，会在compute方法即执行exec方法时等待所有的子任务执行完成；`quietlyInvoke`和`invoke `都是基于`doInvoke`实现，**区别在于前者不关心执行的结果，不会抛出异常**。其实现如下：

```java
/**
 * Commences performing this task, awaits its completion if
 * necessary, and returns its result, or throws an (unchecked)
 * {@code RuntimeException} or {@code Error} if the underlying
 * computation did so.
 * 
 * 开始执行此任务，在必要的时候等待完成，然后返回其结果。如果基础计算执行了此操作，则抛出 
 *  RuntimeException`或者`Error`。
 *
 * @return the computed result
 */
public final V invoke() {
    int s;
    // 通过doExec立即执行任务，如果任务未完成则等待
    if ((s = doInvoke() & DONE_MASK) != NORMAL)
        reportException(s); //任务被取消或者异常终止则抛出异常
    return getRawResult();
}
    
/**
 * Commences performing this task and awaits its completion if
 * necessary, without returning its result or throwing its
 * exception.
 *
 * 开始执行此任务并在必要时等待其完成，而不返回其结果或抛出异常
 */    
public final void quietlyInvoke() { doInvoke();}  //不需要抛出异常
```

###### 3.3.3.1 doInvoke

```java

/**
 * Implementation for invoke, quietlyInvoke.
 *
 * @return status upon completion
 */
private int doInvoke() {
    int s; Thread t; ForkJoinWorkerThread wt;
    // 直接调用doExec方法执行任务，如果执行完成，则直接返回
    return (s = doExec()) < 0 ? s :
        ((t = Thread.currentThread()) instanceof ForkJoinWorkerThread) ?
        ///如果是ForkJoinWorkerThread，通过awaitJoin方法等待任务执行完成
        (wt = (ForkJoinWorkerThread)t).pool.
        awaitJoin(wt.workQueue, this, 0L) :
        //普通的Java线程，通过externalAwaitDone等待任务执行完成
        externalAwaitDone();
}
```

###### 3.3.3.2 测试用例

```java
@Test
public void test3() throws Exception {
    Thread thread=Thread.currentThread();
    ForkJoinTask task=new ForkJoinTask() {
        @Override
        public Object getRawResult() { return null;}

        @Override
        protected void setRawResult(Object value) {}

        @Override
        protected boolean exec() {
            System.out.println(Thread.currentThread().getName()+" start run");
            if(thread==Thread.currentThread()){
                return false;
            }else{
                try {
                    Thread.sleep(2000);
                    System.out.println(Thread.currentThread().getName()+" exit");
                    return true;
                } catch (InterruptedException e) {
                    return false;
                }
            }
        }
    };
    ForkJoinPool.commonPool().submit(task);
    //阻塞当前线程，等待任务执行完成
    task.invoke();
    System.out.println("main thread exit");
}
```

上述示例有两种运行结果:

- ![img](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/20200514111828570.png)

- ![img](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/20200514111904141.png)

​	第二种会无期限阻塞，第一种是正常退出，为啥会有这种情形了？main线程执行`invoke`方法，`invoke`方法调用`doExec`方法返回值等于0后就执行`externalAwaitDone`方法了，如果执行`ForkJoinPool.common.tryExternalUnpush`方法返回true，则再次执行`doExec`方法，因为返回值还是0，则通过wait方法等待了，因为没有其他线程唤醒该线程，就会无期限等待；如果执行`ForkJoinPool.common.tryExternalUnpush`方法返回false，说明**某个Worker线程已经将该任务从任务队列中移走了，Worker线程会负责执行该任务并修改任务执行状态，如果Worker线程正在执行的过程中则wait等待Worker线程执行完成，Worker执行完成会唤醒等待的main线程，main线程判断任务已完成就正常退出了**。

###### 3.3.3.3 externalAwaitDone

```java
 /**
  * Blocks a non-worker-thread until completion.
  * @return status upon completion
  * 阻塞非工作线程，直至执行完毕
  */
 private int externalAwaitDone() {
     int s = ((this instanceof CountedCompleter) ? // try helping
              //如果是CountedCompleter，则通过externalHelpComplete方法阻塞当前线程等待任务完成
              ForkJoinPool.common.externalHelpComplete(
                  (CountedCompleter<?>)this, 0) :
              //如果是普通的ForkJoinTask，则通过tryExternalUnpush尝试将其从任务队列中pop出来，如果该任务位于任务队列顶端则pop成功并返回true
               //pop成功后执行doExec方法，即通过当前线程完成任务
              ForkJoinPool.common.tryExternalUnpush(this) ? doExec() : 0);
     if (s >= 0 && (s = status) >= 0) {
         boolean interrupted = false;
         do {
              //修改status，加上SIGNAL标识，表示有线程等待了
             if (U.compareAndSwapInt(this, STATUS, s, s | SIGNAL)) {
                 synchronized (this) {
                     if (status >= 0) { //再次校验状态
                         try {
                            //0表示无期限等待，直到被唤醒
                            //任务执行完成，可通过setCompletion方法唤醒等待的线程
                             wait(0L);
                         } catch (InterruptedException ie) {
                             interrupted = true;
                         }
                     }
                     else
                         notifyAll();  //任务已执行完成，则唤醒所有等待的线程
                 }
             }
         } while ((s = status) >= 0);
         if (interrupted)  //等待时被中断，将当前线程标记为已中断
             Thread.currentThread().interrupt();
     }
     return s;
 }
```

##### 3.3.4 invokeAll

​	`invokeAll`方法有三个重载版本，都是**等待多个任务执行完成**，其中**第一个任务都是有当前线程执行，其他任务是提交到线程池执行**，多个任务时，如果有**一个任务执行异常**，则会**取消掉剩余未执行的任务**。其实现如下：

```java
/**
 * 分派给定的任务，在每个任务都保留isDone或者遇到未检查的异常的时候返回。在这种情况下，异常被重新抛出
 * 如果一个以上的任务遇到异常，则此方法中将引发这些异常中的任何一个。如果任何任务遇到异常，但其他任务可
 * 会被取消，但是，无法保证在异常返回的时候单个任务的执行状态，可以使用getException和相关方向来获取每个
 * 任务的状态，以检查它们是否已被取消，正常、或者未处理。
 *
 * Forks the given tasks, returning when {@code isDone} holds for
 * each task or an (unchecked) exception is encountered, in which
 * case the exception is rethrown. If more than one task
 * encounters an exception, then this method throws any one of
 * these exceptions. If any task encounters an exception, the
 * other may be cancelled. However, the execution status of
 * individual tasks is not guaranteed upon exceptional return. The
 * status of each task may be obtained using {@link
 * #getException()} and related methods to check if they have been
 * cancelled, completed normally or exceptionally, or left
 * unprocessed.
 *
 * @param t1 the first task
 * @param t2 the second task
 * @throws NullPointerException if any task is null
 */
public static void invokeAll(ForkJoinTask<?> t1, ForkJoinTask<?> t2) {
    int s1, s2;
    t2.fork(); //将t2提交到任务队列 用工作线程执行
    if ((s1 = t1.doInvoke() & DONE_MASK) != NORMAL) //执行t1并等待其执行完成
        t1.reportException(s1); //不是正常结束则抛出异常
    if ((s2 = t2.doJoin() & DONE_MASK) != NORMAL) //等待t2执行完成
        t2.reportException(s2);
}

//此时tasks相当于一个ForkJoinTask数组
public static void invokeAll(ForkJoinTask<?>... tasks) {
    Throwable ex = null;
    int last = tasks.length - 1;
    //除第0个之外的任务都会调用fork进行处理，而第0个会用当前线程进行处理
    for (int i = last; i >= 0; --i) {
        ForkJoinTask<?> t = tasks[i];
        if (t == null) {
            //某个ForkJoinTask为null
            if (ex == null)
                ex = new NullPointerException();
        }
        else if (i != 0) //i不等于0的，将其提交到任务队列
            t.fork();
        //i等于0，立即执行并等待其执行完成    
        else if (t.doInvoke() < NORMAL && ex == null)
            ex = t.getException();
    }
    //从1开始往前遍历
    for (int i = 1; i <= last; ++i) {
        ForkJoinTask<?> t = tasks[i];
        if (t != null) {
            if (ex != null) //ex不为空，则取消任务
                t.cancel(false); 
            else if (t.doJoin() < NORMAL) //等待任务执行完成，如果不是正常结束的则获取抛出的异常
                ex = t.getException();
        }
    }
    if (ex != null)
        rethrow(ex); //重新抛出异常
}

public static <T extends ForkJoinTask<?>> Collection<T> invokeAll(Collection<T> tasks) {
    if (!(tasks instanceof RandomAccess) || !(tasks instanceof List<?>)) {
        //如果没有实现RandomAccess接口或者不是List类型
        invokeAll(tasks.toArray(new ForkJoinTask<?>[tasks.size()]));
        return tasks;
    }
    //是List类型且实现了RandomAccess接口
    @SuppressWarnings("unchecked")
    List<? extends ForkJoinTask<?>> ts =
        (List<? extends ForkJoinTask<?>>) tasks;
    Throwable ex = null;
    //逻辑同上
    int last = ts.size() - 1;
    //从last处往前遍历
    for (int i = last; i >= 0; --i) {
        ForkJoinTask<?> t = ts.get(i);
        if (t == null) {
            if (ex == null)
                ex = new NullPointerException();
        }
        else if (i != 0)
            t.fork();
        else if (t.doInvoke() < NORMAL && ex == null)
            ex = t.getException();
    }
    //从1开始往后遍历
    for (int i = 1; i <= last; ++i) {
        ForkJoinTask<?> t = ts.get(i);
        if (t != null) {
            if (ex != null)
                t.cancel(false);
            else if (t.doJoin() < NORMAL)
                ex = t.getException();
        }
    }
    if (ex != null)
        rethrow(ex);
    return tasks;
}

public boolean cancel(boolean mayInterruptIfRunning) {
    return (setCompletion(CANCELLED) & DONE_MASK) == CANCELLED;
}
```

##### 3.3.6 complete / quietlyComplete/ completeExceptionally

​	这三个方法都是任务执行完成时调用的，其中`complete`方法用于保存任务执行的结果并修改状态，`quietlyComplete`方法只修改状态，`completeExceptionally`用于任务执行异常时保存异常信息并修改状态，其中只有`quietlyComplete`方法有调用方，都是`CountedCompleter`及其子类，其调用链如下：

![img](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/20200514143220384.png)

```java
//保存任务执行的结果并修改任务状态
/**
 * 完成此任务，如果尚未中止或取消，则返回给定值作为后续调用join和相关操作的结果。 此方法可用于为异步任务
 * 提供结果，或为无法正常完成的任务提供替代处理。 不鼓励在其他情况下使用它。 此方法是可覆盖的，但被覆盖的
 * 版本必须调用super实现来维护保证。 
 *
 * Completes this task, and if not already aborted or cancelled,
 * returning the given value as the result of subsequent
 * invocations of {@code join} and related operations. This method
 * may be used to provide results for asynchronous tasks, or to
 * provide alternative handling for tasks that would not otherwise
 * complete normally. Its use in other situations is
 * discouraged. This method is overridable, but overridden
 * versions must invoke {@code super} implementation to maintain
 * guarantees.
 *
 * @param value the result value for this task
 */
public void complete(V value) {
    try {
        //保存结果
        setRawResult(value);
    } catch (Throwable rex) {
        //出现异常，保存关联的异常
        setExceptionalCompletion(rex);
        return;
    }
    //修改状态，正常完成
    setCompletion(NORMAL);
}

/**
 * 正常完成此任务，无需设置值。 setRawResult建立的最新值（或默认为null ）将作为后续调用join和相关操作
 * 的结果返回。
 *
 * Completes this task normally without setting a value. The most
 * recent value established by {@link #setRawResult} (or {@code
 * null} by default) will be returned as the result of subsequent
 * invocations of {@code join} and related operations.
 *
 * @since 1.8
 */
public final void quietlyComplete() {
    //修改状态正常完成
    setCompletion(NORMAL);
}

//任务异常结束时，保存异常信息并修改状态
/**
 * 异常完成此任务，如果尚未中止或取消，将导致它在join和相关操作时抛出给定的异常。 此方法可用于在异步任务
 * 中引发异常，或强制完成原本不会完成的任务。 不鼓励在其他情况下使用它。 此方法是可覆盖的，但被覆盖的版本
 * 必须调用super实现来维护保证
 *
 * Completes this task abnormally, and if not already aborted or
 * cancelled, causes it to throw the given exception upon
 * {@code join} and related operations. This method may be used
 * to induce exceptions in asynchronous tasks, or to force
 * completion of tasks that would not otherwise complete.  Its use
 * in other situations is discouraged.  This method is
 * overridable, but overridden versions must invoke {@code super}
 * implementation to maintain guarantees.
 *
 * @param ex the exception to throw. If this exception is not a
 * {@code RuntimeException} or {@code Error}, the actual exception
 * thrown will be a {@code RuntimeException} with cause {@code ex}.
 */
public void completeExceptionally(Throwable ex) {
    //记录异常信息并更新任务状态
    setExceptionalCompletion((ex instanceof RuntimeException) ||
                             (ex instanceof Error) ? ex :
                             new RuntimeException(ex)); //如果不是RuntimeException或者Error，则将其用RuntimeException包装一层
}
```

###### 3.3.5.1 setCompletion

```java
/**
  * 标记完成并唤醒等待加入此任务的线程
  *
  * Marks completion and wakes up threads waiting to join this
  * task.
  *
  * @param completion one of NORMAL, CANCELLED, EXCEPTIONAL
  * @return completion status on exit
  */
private int setCompletion(int completion) {
    for (int s;;) {
        if ((s = status) < 0) //如果已完成，直接返回
            return s;
        if (U.compareAndSwapInt(this, STATUS, s, s | completion)) {
            //cas修改状态成功
            if ((s >>> 16) != 0) //如果status中有SIGNAL标识，即有线程在等待当前任务执行完成
                synchronized (this) { notifyAll(); } //唤醒等待的线程
            return completion;
        }
    }
}
```

###### 3.3.5.2 setExceptionalCompletion

```java
/**
 * 记录异常并可能传播
 *
 * Records exception and possibly propagates.
 *
 * @return status on exit
 */
private int setExceptionalCompletion(Throwable ex) {
    //记录异常信息并更新任务状态
    int s = recordExceptionalCompletion(ex);
    if ((s & DONE_MASK) == EXCEPTIONAL) //如果状态是异常完成，则执行钩子方法
        internalPropagateException(ex); //默认是空实现
    return s;
}
```

###### 3.3.5.3 internalPropagateException

```java
/**
 * 钩子为具有完成者的任务提供异常传播支持
 *
 * Hook for exception propagation support for tasks with completers.
 */
void internalPropagateException(Throwable ex) {}
```

##### 3.3.7 awaitJoin / helpComplete / externalHelpComplete

`awaitJoin`方法是ForkJoinTask使用的，用于**阻塞当前线程直到任务执行完成，如果该任务被其他某个线程偷走了，则会帮助其尽快的执行**，其调用链如下：![img](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/20200511154736259.png)

`helpComplete`和`externalHelpComplete`同样是阻塞当前线程等待任务执行完成，不过只适用于ForkJoinTask的特殊子类`CountedCompleter`。externalHelpComplete基于helpComplete实现，后者会**遍历WorkQueue数组找到目标task，然后执行该任务**，其调用链如下：

![img](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/20200511155155649.png)

```java
//task本来是属于w的，阻塞当前线程等待task执行完成
final int awaitJoin(WorkQueue w, ForkJoinTask<?> task, long deadline) {
    int s = 0;
    if (task != null && w != null) {
        //获取等待的Task
        ForkJoinTask<?> prevJoin = w.currentJoin;
        //将currentJoin修改成task
        U.putOrderedObject(w, QCURRENTJOIN, task);
        CountedCompleter<?> cc = (task instanceof CountedCompleter) ?
            (CountedCompleter<?>)task : null;
        for (;;) {
            if ((s = task.status) < 0) //如果任务已执行完成
                break;
            if (cc != null)
                //如果task是CountedCompleter实例
                helpComplete(w, cc, 0);
            //如果task不是CountedCompleter实例    
            else if (w.base == w.top        //w中没有待执行的任务
                     || w.tryRemoveAndExec(task)) //尝试去移除并执行task，如果队列是空的则返回true
                //task被其他某个线程从w中偷走了，帮助该线程执行完
                helpStealer(w, task);
            if ((s = task.status) < 0)
                break;
            long ms, ns;
            if (deadline == 0L)
                ms = 0L;
            else if ((ns = deadline - System.nanoTime()) <= 0L)
                break; //等待超时
            else if ((ms = TimeUnit.NANOSECONDS.toMillis(ns)) <= 0L)
                ms = 1L; //超过最大值
            if (tryCompensate(w)) { //如果需要阻塞
                task.internalWait(ms); //等待指定的时间
                U.getAndAddLong(this, CTL, AC_UNIT); //增加AC
            }
        }//for循环结束
        U.putOrderedObject(w, QCURRENTJOIN, prevJoin);
    }
    return s;
}


final int externalHelpComplete(CountedCompleter<?> task, int maxTasks) {
    WorkQueue[] ws; int n;
    int r = ThreadLocalRandom.getProbe();
    return ((ws = workQueues) == null || (n = ws.length) == 0) ? 0 :
    helpComplete(ws[(n - 1) & r & SQMASK], task, maxTasks);
}


//会在指定的任务队列w或者任务数组中其他任务队列中查找task，如果找到则将其从队列中移除并执行
final int helpComplete(WorkQueue w, CountedCompleter<?> task,
                       int maxTasks) {
    WorkQueue[] ws; int s = 0, m;
    if ((ws = workQueues) != null && (m = ws.length - 1) >= 0 && //workQueues已初始化
        task != null && w != null) {
        int mode = w.config;                 // for popCC
        //计算一个随机数，第一个扫描的WorkQueue
        int r = w.hint ^ w.top;              // arbitrary seed for origin
        int origin = r & m;                  // first queue to scan
        int h = 1;                           // 1:ran, >1:contended, <0:hash
        for (int k = origin, oldSum = 0, checkSum = 0;;) {
            CountedCompleter<?> p; WorkQueue q;
            if ((s = task.status) < 0) //如果任务已执行完成
                break;
            if (h == 1 && (p = w.popCC(task, mode)) != null) {
                p.doExec(); //执行task任务，下一次for循环因为status小于0了会终止循环
                if (maxTasks != 0 && --maxTasks == 0) 
                    break; //执行任务的次数达到最大值
                origin = k;                  // reset
                oldSum = checkSum = 0;
            }
            else {                           // poll other queues
                if ((q = ws[k]) == null)
                    h = 0; //置为0后就不会进入上面h==1的if分支了
                //ws[k]不等于null   
                else if ((h = q.pollAndExecCC(task)) < 0)
                    checkSum += h;
                if (h > 0) {
                    //h等于1表示查找并执行成功，将maxTasks减1
                    if (h == 1 && maxTasks != 0 && --maxTasks == 0)
                        break; //maxTasks达到最大值了
                    //h不等于1，表示查找失败，重新计算r，遍历下一个WorkQueue查找task    
                    r ^= r << 13; r ^= r >>> 17; r ^= r << 5; // xorshift
                    origin = k = r & m;      // move and restart
                    oldSum = checkSum = 0;
                }
                //如果h小于等于0，h应该是-1，表示ws[k]的base属性对应的Task不是CountedCompleter或者不包含目标任务task
                //k加1，遍历下一个任务队列，再次等于origin说明所有的WorkQueue都遍历了一遍
                else if ((k = (k + 1) & m) == origin) {
                    if (oldSum == (oldSum = checkSum))
                        break; //oldSum等于checkSum说明没有新的CountedCompleter提交到任务队列
                    checkSum = 0;
                }
            }
        }//for循环结束
    }
    return s;
}

//如果task被其他某个线程偷走了，则帮助这个线程执行完任务
private void helpStealer(WorkQueue w, ForkJoinTask<?> task) {
    WorkQueue[] ws = workQueues;
    int oldSum = 0, checkSum, m;
    if (ws != null && (m = ws.length - 1) >= 0 && w != null &&
        task != null) {
        do {                                       // restart point
            checkSum = 0;                          // for stability check
            ForkJoinTask<?> subtask;
            WorkQueue j = w, v;                    // v is subtask stealer
            descent: for (subtask = task; subtask.status >= 0; ) {
                for (int h = j.hint | 1, k = 0, i; ; k += 2) {
                    if (k > m)  
                    //所有WorkQueue都遍历过了，没有找到目标task则终止最外层的for循环，开始下一次的while循环
                        //通过while循环判断任务是否执行完成，如果已完成则返回
                        break descent; 
                    //h是一个奇数，k是一个偶数，h+k的结果就是一个奇数
                    //遍历的WorkQueue都是跟Worker线程绑定的
                    if ((v = ws[i = (h + k) & m]) != null) {
                        if (v.currentSteal == subtask) {
                            //如果找到目标task
                            j.hint = i;
                            break; //终止内层for循环，进入到下面的for循环
                        }
                        checkSum += v.base;
                    }
                }
                //找到偷走目标task的Worker线程关联的WorkQueue v
                for (;;) {                         // help v or descend
                    ForkJoinTask<?>[] a; int b;
                    checkSum += (b = v.base);
                    ForkJoinTask<?> next = v.currentJoin;
                    if (subtask.status < 0 || j.currentJoin != subtask ||
                        v.currentSteal != subtask) //如果任务已经执行完成
                        break descent; //终止最外层的for循环，开始下一次的while循环
                    if (b - v.top >= 0 || (a = v.array) == null) {//v中没有待执行的任务
                        if ((subtask = next) == null) // v.currentJoin为null，说明task可能已执行完成
                            break descent;
                        //v.currentJoin不为null，注意此时subtask被更新成v.currentJoin 
                        //v关联的Worker线程在等待currentJoin任务执行完成才能执行currentSteal任务
                        //所以这里需要向上遍历帮助所有currentJoin任务执行完成，才能最终执行currentSteal任务
                        j = v; //v对应的队列中没有待处理的任务
                        break; //终止内层for循环，开始外层descent对应的for循环
                    }
                    //v中有待执行的任务
                    int i = (((a.length - 1) & b) << ASHIFT) + ABASE;
                    //获取数组索引为i的元素
                    ForkJoinTask<?> t = ((ForkJoinTask<?>)
                                         U.getObjectVolatile(a, i));
                    if (v.base == b) { //base属性未发生变更
                        if (t == null)             //v的base属性发生变更了
                            break descent;
                        if (U.compareAndSwapObject(a, i, t, null)) {
                            //将数组索引为i的元素修改为null
                            v.base = b + 1; //base加1
                            ForkJoinTask<?> ps = w.currentSteal;
                            int top = w.top;
                            do {
                                //修改w的currentSteal属性，w对应的worker线程来执行v对应的Worker线程未执行的任务
                                U.putOrderedObject(w, QCURRENTSTEAL, t);
                                t.doExec();        //执行t
                            } while (task.status >= 0 &&
                                     w.top != top &&
                                     (t = w.pop()) != null); //执行w中的任务
                            //恢复w的currentSteal属性       
                            U.putOrderedObject(w, QCURRENTSTEAL, ps);
                            if (w.base != w.top) //w中添加了新任务
                                return;            // can't further help
                        }
                    }
                }//第二个内层for循环
            }//for循环结束
            //不断while循环，直到task执行完成，或者oldSum等于checkSum   
        } while (task.status >= 0 && oldSum != (oldSum = checkSum));
    }
}

//用于判断是否需要将当前线程阻塞，如果返回true且W非空，则会创建新的Worker线程代替当前阻塞的Worker线程继续执行w中其他待执行的任务
private boolean tryCompensate(WorkQueue w) {
    boolean canBlock;
    WorkQueue[] ws; long c; int m, pc, sp;
    if (w == null || w.qlock < 0 ||           // w终止了
        (ws = workQueues) == null || (m = ws.length - 1) <= 0 || //workQueues未初始化
        (pc = config & SMASK) == 0)           // parallelism为0
        canBlock = false;
    else if ((sp = (int)(c = ctl)) != 0)      //ctl的低32位不为0，尝试唤醒一个空闲的Worker线程
        //tryRelease返回true，表示成功唤醒一个Worker线程，可以替代当前线程继续执行任务
        //所以canBlock返回true，让当前线程阻塞
        canBlock = tryRelease(c, ws[sp & m], 0L);
    else {
        //没有空闲的Worker线程
        //获取ac和tc
        int ac = (int)(c >> AC_SHIFT) + pc;
        int tc = (short)(c >> TC_SHIFT) + pc;
        int nbusy = 0;                        // validate saturation
        for (int i = 0; i <= m; ++i) {        // two passes of odd indices
            WorkQueue v;
            //((i << 1) | 1)算出来的是一个奇数，即遍历Worker线程绑定的WorkQueue
            if ((v = ws[((i << 1) | 1) & m]) != null) { 
                if ((v.scanState & SCANNING) != 0)
                    break;  //如果正在扫描，则终止for循环
                ++nbusy;
            }
        }
        if (nbusy != (tc << 1) || ctl != c)
            canBlock = false;                 // unstable or stale
        else if (tc >= pc && ac > 1 && w.isEmpty()) {
            //没有待执行的任务了，需要休眠等待，将AC减1，修改ctl
            long nc = ((AC_MASK & (c - AC_UNIT)) |
                       (~AC_MASK & c));       // uncompensated
            canBlock = U.compareAndSwapLong(this, CTL, c, nc);
        }
        //w非空
        //总线程数超标
        else if (tc >= MAX_CAP ||
                 (this == common && tc >= pc + commonMaxSpares))
            throw new RejectedExecutionException(
            "Thread limit exceeded replacing blocked worker");
        else {                                // similar to tryAddWorker
            boolean add = false; int rs;      // CAS within lock
            //TC加1，createWorker方法中会新增TC
            //因为当前线程会被阻塞，所以新增线程后AC不会加1
            long nc = ((AC_MASK & c) |
                       (TC_MASK & (c + TC_UNIT)));
            if (((rs = lockRunState()) & STOP) == 0) //获取锁且线程池没有停止
                add = U.compareAndSwapLong(this, CTL, c, nc); //修改ctl，增加tc
            unlockRunState(rs, rs & ~RSLOCK); //解锁
            //如果cas修改ctl成功，则创建新的worker线程，该线程会在当前线程阻塞时继续执行WorkQueue中其他任务
            canBlock = add && createWorker(); // throws on exception
        }
    }
    return canBlock;
}
```

### 4.异常机制

​	`ForkJoinTask`在执行的时候可能会抛出异常，但是我们没办法在主线程里直接捕获异常，所以`ForkJoinTask`提供了`isCompletedAbnormally`()方法来检查任务是否已经抛出异常或已经被取消了，并且可以通过ForkJoinTask的`getException`方法获取异常。使用如下代码：

```java
if(task.isCompletedAbnormally()) System.out.println(task.getException());
```

#### 1. 异常table

`ForkJoinTask`维护了一个`ExceptionTable`,其相关常量如下：

```java
// Exception table support

//执行任务的时候抛出的异常表，使调用者能够进行报告，由于异常很少见，因此我们不直接将其保存在任务对象中，而是使用弱引用表。请注意，取消异常未出现在表中，而是记录状态值。注意，这些静态变量在下面的静态块中初始化。
private static final ExceptionNode[] exceptionTable;
private static final ReentrantLock exceptionTableLock;
private static final ReferenceQueue<Object> exceptionTableRefQueue;

//固定长度为32
private static final int EXCEPTION_MAP_CAPACITY = 32;
```

​	可以发现，维护了一个`static`的异常`ExceptionNode`的数组。这个数组长度为**32**位，其中内容是弱引用的`ExceptionNode`。也就是说，全局只会有一个异常表。

#### 2.ExceptionNode

​	ExceptionNode是异常表的链接节点，链接hash表使用身份比较、完全锁定和key的弱引用。该表具有固定容量，因为该表仅将任务异常维护的时间足够长，以使联接者可以访问它们，因此该表在持续时间内永远不会变得很大。但是，由于我们不知道最后一个连接器何时完成，因此必须使用弱引用并将其删除。我们对每个操作都执行此操作（因此完全锁定）。另外，任何ForkJoinPool中的某个线程在其池变为isQuiescent时都会调用`helpExpungeStaleExceptions`。

```java
static final class ExceptionNode extends WeakReference<ForkJoinTask<?>> {
    final Throwable ex;
    //链表结构
    ExceptionNode next;
    final long thrower;  // use id not ref to avoid weak cycles
    //hashCode
    final int hashCode;  // store task hashCode before weak ref disappears
    
    ExceptionNode(ForkJoinTask<?> task, Throwable ex, ExceptionNode next) {
       //初始化赋值
        super(task, exceptionTableRefQueue);
        this.ex = ex;
        //下一个记录
        this.next = next;
        //记录线程ID
        this.thrower =
        Thread.currentThread().getId();
        //赋值hashCode
        this.hashCode = System.identityHashCode(task);
    }
}
```

#### 3. 异常的调用方法

#####  3.1 recordExceptionalCompletion

**记录异常信息并修改任务状态**。

```java
/**
 * Records exception and sets status.
 *
 * @return status on exit
 */
final int recordExceptionalCompletion(Throwable ex) {
    int s;
    //如果s的状态大于0 说明task没有执行完
    if ((s = status) >= 0) {
        //获取hash值
        int h = System.identityHashCode(this);
        //定义锁
        final ReentrantLock lock = exceptionTableLock;
        lock.lock();
        try {
            //清理掉已经被GC回收掉的ExceptionNode
            expungeStaleExceptions();
            ExceptionNode[] t = exceptionTable;
             //计算该节点的索引
            int i = h & (t.length - 1);
            for (ExceptionNode e = t[i]; ; e = e.next) {
                if (e == null) {
                   //如果t[i]为null或者遍历完了没有找到匹配的，则创建一个新节点，插入到t[i]链表的前面
                    t[i] = new ExceptionNode(this, ex, t[i]);
                    break;
                }
                // 找到目标节点
                if (e.get() == this) // already present
                    break;
            }
        } finally {
            lock.unlock();
        }
        //设置状态，异常终止
        s = setCompletion(EXCEPTIONAL);
    }
    return s;
}
```

##### 3.2 clearExceptionalCompletion

`clearExceptionalCompletion`是`reinitialize`调用的，用于清理掉当前任务关联的异常信息

```java
//清理掉当前Task关联的ExceptionNode
private void clearExceptionalCompletion() {
    //识别hashCode
    int h = System.identityHashCode(this);
    //获得锁
    final ReentrantLock lock = exceptionTableLock;
    //加锁
    lock.lock();
    try {
        ExceptionNode[] t = exceptionTable;
        //计算所属的数组元素
        int i = h & (t.length - 1);
        ExceptionNode e = t[i];
        ExceptionNode pred = null;
        //遍历链表
        while (e != null) {
            ExceptionNode next = e.next;
            //将当前null清理
            if (e.get() == this) { 
                if (pred == null) //this是链表第一个节点
                    t[i] = next;
                else
                    pred.next = next; //this是链表中的一个节点
                break;
            }
            pred = e;  //遍历下一个节点
            e = next;
        }
        expungeStaleExceptions(); //清理掉已经被GC回收掉的ExceptionNode
        status = 0;
    } finally {
        lock.unlock();
    }
}
```

###### 3.2.1 reinitialize

```java
//将任务恢复至初始状态，然后可正常执行
public void reinitialize() {
    if ((status & DONE_MASK) == EXCEPTIONAL)
        clearExceptionalCompletion(); //如果是异常结束，则清除关联的异常信息
    else
        //状态恢复成0
        status = 0;
}
```

##### 3.3 expungeStaleExceptions

取出并删除过时的引用，只在锁定的时候执行

```java
//将exceptionTableRefQueue中已经被GC回收掉的节点从exceptionTable中移除
private static void expungeStaleExceptions() {
    //poll方法移除并返回链表头，链表中的节点是已经被回收掉了
    for (Object x; (x = exceptionTableRefQueue.poll()) != null;) {
        if (x instanceof ExceptionNode) {
            int hashCode = ((ExceptionNode)x).hashCode;
            ExceptionNode[] t = exceptionTable;
            //计算该节点的索引
            int i = hashCode & (t.length - 1);
            ExceptionNode e = t[i];
            ExceptionNode pred = null;
            while (e != null) {
                ExceptionNode next = e.next;
                if (e == x) { //找到目标节点
                    if (pred == null) //x就是链表第一个节点
                        t[i] = next;
                    else  //x是链表中某个节点
                        pred.next = next;
                    break;
                }
                //遍历下一个节点
                pred = e;
                e = next;
            }
        }
    }
}
```

##### 3.4 getException

`getException`方法获取当前任务关联的异常信息，如果任务是正常结束的则返回null，如果是被取消则返回`CancellationException`，如果异常结束则返回执行任务过程中抛出的异常。

```java
public final Throwable getException() {
    int s = status & DONE_MASK; 
    return ((s >= NORMAL)    ? null : //如果是正常完成，则返回null
            (s == CANCELLED) ? new CancellationException() : //任务被取消，则返回CancellationException
            getThrowableException()); //任务异常结束，获取之前异常结束时保存的异常信息
}
```

###### 3.4.1 getThrowableException

​		返回给定任务的可重试异常，为了提供准确的堆栈线索，如果异常不是又当前线程引起的，我们将尝试创建一个与引起异常类型相同的异常，但是记录的时候只会做为原因，如果没有这样的构造函数，我们会尝试使用一个无参的构造函数。后面跟着initCause，达到同样的效果。如果这些都不适用，或者由于其他异常而失败，我们返回记录的异常，它仍然是正确的，尽管它可能包含误导性的堆栈跟踪。

```java
private Throwable getThrowableException() {
    if ((status & DONE_MASK) != EXCEPTIONAL) //不是异常结束，返回null
        return null;
    int h = System.identityHashCode(this);
    ExceptionNode e;
    //加锁
    final ReentrantLock lock = exceptionTableLock;
    lock.lock();
    try {
        //清理掉已经被GC回收掉的ExceptionNode
        expungeStaleExceptions();
        ExceptionNode[] t = exceptionTable;
        //计算所属的数组元素
        e = t[h & (t.length - 1)];
        //遍历链表，e.get()方法返回该ExceptionNode关联的Task
        while (e != null && e.get() != this)
            e = e.next;
    } finally {
        lock.unlock();
    }
    Throwable ex;
    //没有找到当前Task 或者ex为null
    if (e == null || (ex = e.ex) == null) 
        return null;
    if (e.thrower != Thread.currentThread().getId()) {
        //如果保存异常信息的线程不是当前线程，创建一个同类型的异常实例包装原来的异常信息，从而提供准确的异常调用链
        Class<? extends Throwable> ec = ex.getClass();
        try {
            Constructor<?> noArgCtor = null;
            //获取构造函数
            Constructor<?>[] cs = ec.getConstructors();// public ctors only
            for (int i = 0; i < cs.length; ++i) {
                Constructor<?> c = cs[i];
                //获取构造函数的参数类型
                Class<?>[] ps = c.getParameterTypes();
                if (ps.length == 0)
                    noArgCtor = c; //默认的构造函数
                else if (ps.length == 1 && ps[0] == Throwable.class) {
                    //如果只有一个参数，且参数类型是Throwable，则创建一个新异常实例
                    Throwable wx = (Throwable)c.newInstance(ex);
                    return (wx == null) ? ex : wx;
                }
            }
            if (noArgCtor != null) {
                //有默认的无参构造函数，创建一个实例并设置ex
                Throwable wx = (Throwable)(noArgCtor.newInstance());
                if (wx != null) {
                    wx.initCause(ex);
                    return wx;
                }
            }
        } catch (Exception ignore) {
        }
    }
    return ex;
}
```

### 5. 适配其他类型task

此外，ForkJoinTask还提供了适配其他类型任务的适配器。如下：

![img](http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/20200424192257892.png)

带S的几个大都是**ForkJoinTask的内部类**，其实现比较简单，用于将`Runnable`等接口转换成`ForkJoinTask`类，对应于ForkJoinPool的adapt方法，以`AdaptedCallable`的实现为例说明，如下：

#### 5.1 adapt方法

适配器类通过适配器方法来调用

```java
public static <T> ForkJoinTask<T> adapt(Callable<? extends T> callable) {
    return new AdaptedCallable<T>(callable);
}

public static <T> ForkJoinTask<T> adapt(Runnable runnable, T result) {
    return new AdaptedRunnable<T>(runnable, result);
}

public static ForkJoinTask<?> adapt(Runnable runnable) {
    return new AdaptedRunnableAction(runnable);
}
```

```apl
RecursiveAction 无返回结果的ForkJoinTask实现Runnable
RecursiveTask<V> 有返回结果的ForkJoinTask实现Callable
CountedCompleter<T> 在任务完成执行后会触发执行一个自定义的钩子函数
```

##### 5.1.1 AdaptedRunnable

​	定义了一个适配器类，来适配`Runnable`
​	实际上就是定义一个类将`Runnable`进行包裹，参考`RunnableFuture`的实现思路，这个适配器需要返回结果，结果通过泛型来定义。

```java
/**
 * Adaptor for Runnables. This implements RunnableFuture
 * to be compliant with AbstractExecutorService constraints
 * when used in ForkJoinPool.
 */
static final class AdaptedRunnable<T> extends ForkJoinTask<T>
    implements RunnableFuture<T> {
    //需要适配的runnable
    final Runnable runnable;
    T result;
    AdaptedRunnable(Runnable runnable, T result) {
        if (runnable == null) throw new NullPointerException();
        this.runnable = runnable;
        this.result = result; // OK to set this even before completion
    }
    public final T getRawResult() { return result; }
    public final void setRawResult(T v) { result = v; }
    public final boolean exec() { runnable.run(); return true; }
    //调用invoke方法
    public final void run() { invoke(); }
    private static final long serialVersionUID = 5232453952276885070L;
}
```

##### 5.1.2 AdaptedRunnableAction

这个适配器不需要返回结果。与`AdapteRunnable`类似

```java
static final class AdaptedRunnableAction extends ForkJoinTask<Void>
    implements RunnableFuture<Void> {
    final Runnable runnable;
    AdaptedRunnableAction(Runnable runnable) {
        if (runnable == null) throw new NullPointerException();
        this.runnable = runnable;
    }
    public final Void getRawResult() { return null; }
    public final void setRawResult(Void v) { }
    public final boolean exec() { runnable.run(); return true; }
    public final void run() { invoke(); }
    private static final long serialVersionUID = 5232453952276885070L;
}
```

##### 5.1.3 RunnableExecuteAction

这是另外一种Runnable适配器，支持异常。

```java
static final class RunnableExecuteAction extends ForkJoinTask<Void> {
    final Runnable runnable;
    RunnableExecuteAction(Runnable runnable) {
        if (runnable == null) throw new NullPointerException();
        this.runnable = runnable;
    }
    public final Void getRawResult() { return null; }
    public final void setRawResult(Void v) { }
    public final boolean exec() { runnable.run(); return true; }
    void internalPropagateException(Throwable ex) {
        rethrow(ex); // rethrow outside exec() catches.
    }
    private static final long serialVersionUID = 5232453952276885070L;
}
```

##### 5.1.4 AdaptedCallable

适配Callables。有返回值。

```java
static final class AdaptedCallable<T> extends ForkJoinTask<T>
    implements RunnableFuture<T> {
    final Callable<? extends T> callable;
    T result;
    AdaptedCallable(Callable<? extends T> callable) {
        if (callable == null) throw new NullPointerException();
        this.callable = callable;
    }
    public final T getRawResult() { return result; }
    public final void setRawResult(T v) { result = v; }
    public final boolean exec() {
        try {
            result = callable.call();
            return true;
        } catch (Error err) {
            throw err;
        } catch (RuntimeException rex) {
            throw rex;
        } catch (Exception ex) {
            throw new RuntimeException(ex);
        }
    }
    public final void run() { invoke(); }
    private static final long serialVersionUID = 2838392045355241008L;
}
```

#### 5.2 总结

ForkJoinTask是ForkJoinPool的基本执行单位。这个类的设计并不复杂，做为理解ForkJoinPool的补充。我们需要直到fork、join、invoke、invokeAll的每个方法的用法。invoke方法会比使用fork节约资源。
 另外我们可以借鉴其异常处理的模式，采用了弱引用。
 适配器模式也得到了很好的应用。在我们写代码的过程中，值得借鉴。

## 五. ForkJoinThread源码分析

### 1.类结构及其成员变量

#### 1.1 类结构和注释

<img src="http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/3237432-6c2c411c0e8163b3.png" alt="img" style="zoom:50%;" />

`ForkJoinWorkerThread`继承了`Thread`类，其注释大意如下：

​	`ForkJoinWorkerThread`是由`ForkJoinPool`管理的线程，该线程执行`ForkJoinTask`。此类仅可**做为扩展功能的需要而被集成**，因为没有提供可以调度或者可重新的方法。但是，你可以覆盖主任务处理循环周围初始化和终止方法。如果确实创建了这个类的子类，还需要在`ForkJoinPool`中提供自定义的`ForkJoinWorkerThreadFactory`来使用。

#### 1.2 常量

​	`ForkJoinWorkerThreads`由`ForkJoinPools`管理，并执行`ForkJoinTasks`。请参见`ForkJoinPool`的内部文档。此类仅仅维护了指向`pool`和`WorkQueue`的链接。`pool`字段在构造的时候直接设置。但是直到对`registerWorker`调用完成之后，才设置`workQueue`字段。这将导致可见性竞争，可以通过要求`workQueue`字段仅由其所属线程访问来规避这个问题。对于`InnocuousForkJoinWorkerThread`子类的支持，要求我们在此处和子类中破坏很多封装，通过`Unsafe`以访问和设置`Thread`字段。
 这是两个final修饰的常量，只能初始化一次。

```java
final ForkJoinPool pool;                // the pool this thread works in  负责管理该线程的线程池实现
final ForkJoinPool.WorkQueue workQueue; // work-stealing mechanics 保存待执行任务的任务队列   
```

<img src="http://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/image-20211119150304400.png" alt="image-20211119150304400" style="zoom:80%;" />

### 2.构造函数

​	第一种是给`DefaultForkJoinWorkerThreadFactory`创建新线程使用，第二种是给子类`InnocuousForkJoinWorkerThread`使用。

```java
protected ForkJoinWorkerThread(ForkJoinPool pool) {
    //指定初始线程名，registerWorker方法中会修改线程名
    super("aForkJoinWorkerThread");
    this.pool = pool;
    //将当前线程注册到Pool，注册完成后返回WorkQueue实例
    this.workQueue = pool.registerWorker(this);
}

ForkJoinWorkerThread(ForkJoinPool pool, ThreadGroup threadGroup,
                     AccessControlContext acc) {
    //指定初始线程名和ThreadGroup，registerWorker方法中会修改线程名
    super(threadGroup, null, "aForkJoinWorkerThread");
    //修改属性inheritedAccessControlContext
    U.putOrderedObject(this, INHERITEDACCESSCONTROLCONTEXT, acc);
    //清空本地线程变量
    eraseThreadLocals(); // clear before registering
    this.pool = pool;
    //将当前线程注册到Pool，注册完成后返回WorkQueue实例
    this.workQueue = pool.registerWorker(this);
}

final void eraseThreadLocals() {
    //将保存本地线程变量的两个Map置为null，即清空可能残留的本地线程变量
    U.putObject(this, THREADLOCALS, null);
    U.putObject(this, INHERITABLETHREADLOCALS, null);
}
```

### 3. 重要方法

#### 3.1 run

​	`ThreadPoolExecutor`中的`Worker`是实现了`Runnable`接口，将`Worker`实例传入`Thread`的构造方法中执行`run`方法的，`ForkJoinWorkerThread`是继承`Thread`，直接改写其`run`方法，其实现如下：

```java
public void run() {
    if (workQueue.array == null) { // only run once
        Throwable exception = null;
        try {
            //启动前的回调方法，默认空实现
            onStart();
            //启动该线程，从任务队列中获取任务并执行
            pool.runWorker(workQueue);
        } catch (Throwable ex) {
            exception = ex;
        } finally {
            try {
                //线程退出时的回调方法
                onTermination(exception);
            } catch (Throwable ex) {
                if (exception == null)
                    exception = ex;
            } finally {
                //通知线程池当前线程退出
                pool.deregisterWorker(this, exception);
            }
        }
    }
}
```

​	`runWorker`方法实际上是对`workQueue`结合随机魔数，选择一个`workQueue`进行遍历，调用`scan`方法，如果不为空则执行，反之则`wait`。
 其中`registerWorker`与`deregisterWorker`方法 ，我们可以参考前面的ForkJoinPool源码解读。

#### 3.2 扩展方法(onStart / onTermination)

​	`onStart`用于`run`实际执行之前，执行一些**初始化操作**。`onTermination`用于**run**实际执行之后，执行一些**清理操作**。

```java
/**
 * Initializes internal state after construction but before
 * processing any tasks. If you override this method, you must
 * invoke {@code super.onStart()} at the beginning of the method.
 * Initialization requires care: Most fields must have legal
 * default values, to ensure that attempted accesses from other
 * threads work correctly even before this thread starts
 * processing tasks.
 */
protected void onStart() {
}

/**
 * Performs cleanup associated with termination of this worker
 * thread.  If you override this method, you must invoke
 * {@code super.onTermination} at the end of the overridden method.
 *
 * @param exception the exception causing this thread to abort due
 * to an unrecoverable error, or {@code null} if completed normally
 */
protected void onTermination(Throwable exception) {
}
```

#### 3.3 getPoolIndex

```java
public int getPoolIndex() {
    //注意workQueue是registerWorker后返回的，与线程是1对1的关系
    return workQueue.getPoolIndex();
}

// workQueue
final int getPoolIndex() {
    return (config & 0xffff) >>> 1; // ignore odd/even tag bit
}
```

### 4.InnocuousForkJoinWorkerThread

​	`InnocuousForkJoinWorkerThread`是`ForkJoinWorkerThread`的一个内部类，继承自`ForkJoinWorkerThread`，表示一个特殊的无任何访问权限的线程，不属于任何用户定义的线程组，当执行`afterTopLevelExec`后会将原线程的本地变量都清空，该类的定义如下：

> `ForkJoinPool#runWorker `-> `WorkQueue#runTask `-> `afterTopLevelExec`

```java
static final class InnocuousForkJoinWorkerThread extends ForkJoinWorkerThread {
    /** 创建一个全局的线程组 */
    private static final ThreadGroup innocuousThreadGroup =
        createThreadGroup();

    /** An AccessControlContext supporting no privileges */
    private static final AccessControlContext INNOCUOUS_ACC =
        new AccessControlContext(
        new ProtectionDomain[] {
            //无任务访问权限
            new ProtectionDomain(null, null)
        });

    InnocuousForkJoinWorkerThread(ForkJoinPool pool) {
        //此构造方法会清空本地线程变量
        super(pool, innocuousThreadGroup, INNOCUOUS_ACC);
    }

    @Override // to erase ThreadLocals
    void afterTopLevelExec() {
        //清空本地线程变量
        eraseThreadLocals();
    }

    @Override // to always report system loader
    public ClassLoader getContextClassLoader() {
        //默认是返回父线程的contextClassLoader，此处改写直接返回SystemClassLoader
        return ClassLoader.getSystemClassLoader();
    }

    @Override //无法设置该线程的异常处理器，即默认为null，出现异常后线程自动退出
    public void setUncaughtExceptionHandler(UncaughtExceptionHandler x) { }

    @Override // 不支持设置ClassLoader，直接抛出异常
    public void setContextClassLoader(ClassLoader cl) {
        throw new SecurityException("setContextClassLoader");
    }

    //Java中ThreadGroup只提供关联线程的管理维护功能，跟底层的Linux线程组无关联
    private static ThreadGroup createThreadGroup() {
        try {
            sun.misc.Unsafe u = sun.misc.Unsafe.getUnsafe();
            Class<?> tk = Thread.class;
            Class<?> gk = ThreadGroup.class;
            long tg = u.objectFieldOffset(tk.getDeclaredField("group"));
            long gp = u.objectFieldOffset(gk.getDeclaredField("parent"));
            //获取当前线程的线程组
            ThreadGroup group = (ThreadGroup)
                u.getObject(Thread.currentThread(), tg);
            while (group != null) {
                //获取父线程组
                ThreadGroup parent = (ThreadGroup)u.getObject(group, gp);
                if (parent == null)
                    //找到了最初的祖先级的线程组，使用该线程组创建一个新的线程组
                    return new ThreadGroup(group,
                                           "InnocuousForkJoinWorkerThreadGroup");
                group = parent;
            }
        } catch (Exception e) {
            throw new Error(e);
        }
        //正常不会进入此代码，直接抛出异常
        throw new Error("Cannot create ThreadGroup");
    }
}
```

### 5. ForkJoinPool中创建工作线程的过程

​	此时再来结合`ForkJoinPool`中的`ForkJoinWorkerThreadFactory`，就能明白`ForkJoinThread`的创建意义了。`ForkJoinPool`根据访问权限的需要，定义了采用**默认的创建方法**，还是创建`InnocuousForkJoinWorkerThread`。

#### 5.1 makeCommonPool创建过程

​	`ForkJoinPool`中的`makeCommPool`，有如下代码：

```java
if (factory == null) {
    if (System.getSecurityManager() == null)
        factory = defaultForkJoinWorkerThreadFactory;
    else // use security-managed default
        factory = new InnocuousForkJoinWorkerThreadFactory();
}
// 这里也就是说，如果System.getSecurityManager()为null，则返回默认的ThreadFactory,而不为null，则说， 使用了默认的安全管理级别，因此将创建InnocuousForkJoinWorkerThreadFactory。
```

#### 5.2 ForkJoinPool#createWorker

```java
ForkJoinWorkerThreadFactory fac = factory;
try {
        if (fac != null && (wt = fac.newThread(this)) != null) {
            wt.start();
            return true;
        }
    } catch (Throwable rex) {
        ex = rex;
    }
// 也就是说，createWorker根据ForkJoinWorkerThreadFactory的实现类来创建。
```

#### 5.3 ForkJoinWorkerThreadFactory

​	`ForkJoinWorkerThreadFactory`是`ForkJoinPool`内部的一个public接口类，用于创建`ForkJoinWorkerThread`，该接口有两个对应的实现类`DefaultForkJoinWorkerThreadFactory`和`InnocuousForkJoinWorkerThreadFactory`，分别用于创建**普通的ForkJoinWorkerThread**和**特殊的InnocuousForkJoinWorkerThread**，其实现如下：

```java
public static interface ForkJoinWorkerThreadFactory {
    public ForkJoinWorkerThread newThread(ForkJoinPool pool);
}

static final class DefaultForkJoinWorkerThreadFactory implements ForkJoinWorkerThreadFactory {

    public final ForkJoinWorkerThread newThread(ForkJoinPool pool) {
        return new ForkJoinWorkerThread(pool);
    }
}

static final class InnocuousForkJoinWorkerThreadFactory implements ForkJoinWorkerThreadFactory {

  /**
    * An ACC to restrict permissions for the factory itself.
    * The constructed workers have no permissions set.
    */
    private static final AccessControlContext innocuousAcc;
    static {
        Permissions innocuousPerms = new Permissions();
        //对应的RuntimePermission为modifyThread
        innocuousPerms.add(modifyThreadPermission);
        //允许改写获取ContextClassLoader的方法
        innocuousPerms.add(new RuntimePermission(
            "enableContextClassLoaderOverride"));
        //允许修改ThreadGroup
        innocuousPerms.add(new RuntimePermission(
            "modifyThreadGroup"));

        innocuousAcc = new AccessControlContext(new ProtectionDomain[] {
            new ProtectionDomain(null, innocuousPerms)
        });
    }

    public final ForkJoinWorkerThread newThread(ForkJoinPool pool) {
        return (ForkJoinWorkerThread.InnocuousForkJoinWorkerThread)
            java.security.AccessController.doPrivileged(
            new java.security.PrivilegedAction<ForkJoinWorkerThread>() {
                public ForkJoinWorkerThread run() {
                    return new ForkJoinWorkerThread.
                        InnocuousForkJoinWorkerThread(pool);
                }}, innocuousAcc);
    }
}
```

### 6.总结

​	`ForkJoinWorkerThread`实际上非常简单，就是结合`ForkJoinPool`，然后根据其需要，创建**合适的线程的过程**。这里面值得我们借鉴的是，如果**需要创建无其他访问权限的线程**，实际上这两种线程大部分内容都是相同的，因此可以通过继承来复用大部分代码。之后定义两个factory，让最终的用户根据需要选择factory。
