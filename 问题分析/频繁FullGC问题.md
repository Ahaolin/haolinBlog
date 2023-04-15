# 频繁FullGC问题分析

## 1. 场景

使用top查看时  cpu飙到了 398.8% ，紧急重启，解决问题。耗时40min（包括通勤）。

## 2.日志分析

> 鉴于此系统安装再docker中，需要先将软件复制到容器中。
>
> 1. 拷贝jdk下bin中`jstack`到doker容器中jre的bin下。
>
>    - ```shell
>      docker cp jstack 容器id:/usr/local/jre/bin
>      
>      # 其他如下（jstack、jps、jmap、jstat、jinfo）
>      jmap -dump:file=a.dump  进程ID
>      # 打印GC信息
>      jstat -gcutil 进程ID 1000
>      ```
>
> 2. 文件jstack尚不是可执行文件，修改文件权限，直接暴力chmod 777 jstack
>
> 3. 拷贝jdk下lib中tools.jar到docker容器中jre的lib下
>
> 4. 再次执行 jstack -l 1，命令正常
>
> [(47条消息) 常见的几款JVM监控工具_wh柒八九的博客-CSDN博客_jvm检测工具](https://blog.csdn.net/qq_31960623/article/details/120541633)
>
> [(47条消息) jvm 内存查看与分析工具_普通网友的博客-CSDN博客_jvm内存分析工具](https://blog.csdn.net/geejkse_seff/article/details/124276085?spm=1001.2101.3001.6650.9&utm_medium=distribute.pc_relevant.none-task-blog-2~default~BlogCommendFromBaidu~Rate-9-124276085-blog-120541633.pc_relevant_aa_2&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2~default~BlogCommendFromBaidu~Rate-9-124276085-blog-120541633.pc_relevant_aa_2&utm_relevant_index=17)
>
> [(47条消息) jvm 性能调优工具之 jstat 命令详解_java领域的博客-CSDN博客_jstat](https://blog.csdn.net/javalingyu/article/details/124800644)



### 2.1 日志显示 jvm参数

|          名称          |          值          | 运算值（MB） |                             作用                             |
| :--------------------: | :------------------: | :----------: | :----------------------------------------------------------: |
|    InitialHeapSize     |      130023424       |     124      |                     JVM初始占用的堆内存                      |
|      MaxHeapSize       |      2051014656      |     1956     |                          最大堆内存                          |
|       MaxNewSize       |      683671552       |     652      |                        最大新生代内存                        |
|        OldSize         |       87031808       |      83      |                  JVM启动时分配的老年代内存                   |
|        NewSize         |       42991616       |      41      |                  JVM启动时分配的新生代内存                   |
|        NewRatio        |          2           |              | 指定老年代/新生代的堆内存比例。在hotspot虚拟机中，堆内存 = 新生代 + 老年代。如果-XX:NewRatio=4表示年轻代与年老代所占比值为1:4,年轻代占整个堆内存的1/5。`在设置了-XX:MaxNewSize的情况下，-XX:NewRatio的值会被忽略`，老年代的内存=堆内存 - 新生代内存。老年代的最大内存 = 堆内存 - 新生代 最大内存。 |
|     SurvivorRatio      |          8           |              | 设置新生代中1个Eden区与1个Survivor区的大小比值。在hotspot虚拟机中，新生代 = 1个Eden + 2个Survivor。Eden:S0:S1=8:1:1 |
|  MaxTenuringThreshold  |          15          |              | 设置新生代gc最大年龄。如果设置为0的话，则新生代对象直接进入老年代设置对象直接进入老年代 |
| PretenureSizeThreshold |          0           |              |   对象大小大于`N`字节的直接在老年代分配对象，0表示不做处理   |
|    `MetaspaceSize`     |       21807104       |              |                                                              |
|     `MaxPermSize`      | 18446744073709547520 |              |     **`永久代不属于堆内存，堆内存只包含新生代和老年代`**     |

- ```
  UseParallelGC
  UseParallelOldGC
  ```

#### 2.1.1 分析

``` mermaid
%% initHeadpSize=maxHeapSize =2051014656   = 【1956M】符合预期
%% metaspaceSize=134217728   = 【128M】符合预期

%% 老年代:新生代 = 2:1    1956 / 3  = 652
%% eden:s1:s2 = 8:1:1      652  / 10 = 65.2
pie
    title 堆内存分析(all 1956M)
    "surovier0" : 65.2
    "surovier1" : 65.2
    "eden" : 521.6
      
    "最大老年代内存" : 1304
%%【GC1： 2.996】
%% [GC(Allocatio Failure)  [PSYoungGen : 500736K->47274K(584192K)]  500736K -> 47370K(1919488K)]
%% [GC(Allocatio Failure)  [PSYoungGen : 489M -> 46.17M(570.5M)]  489M -> 46.25M(1874.5M)]

%% - 500736K(年轻代垃圾回收前内存占用大小[489M]) -->  47274KK(年轻代垃圾回收后内存占用大小[46M])       (584192K【570.5M】)标识年轻代总大小；
%% - 500736K(堆内存垃圾回收前内存占用大小[489M]) -->  47370K(堆内存垃圾回收后内存占用大小[46.2M])     (1919488K【1,874.5M】)表示堆内存总大小；
    
%%【GC2： 6.045】
%% [GC(Allocatio Failure)  [PSYoungGen : 548010K->45951K(584192K)]   548106K -> 46079K(1919488K)]
%% [GC(Allocatio Failure)  [PSYoungGen : 535.17M -> 44.87M(570.5M)]    535.26M -> 45M(1874.5M)]

%%【GC3： 7.523】
%% [GC(Allocatio Failure)  [PSYoungGen : 546687K->53683K(584192K)]   546815K -> 53827K(1919488K)]
%% [GC(Allocatio Failure)  [PSYoungGen : 533.87M -> 52.42M(570.5M)]    534M -> 52.57M(1874.5M)]

%%【GC3： 8.938】
%% [GC(Allocatio Failure)  [PSYoungGen : 554419K->62084K(584192K)]   554563K -> 62236K(1919488K)]
%% [GC(Allocatio Failure)  [PSYoungGen : 541.42M -> 60.63M(570.5M)]    541.57M -> 60.78M(1874.5M)]

%%【GC4： 11.039】
%% [GC(Allocatio Failure)  [PSYoungGen : 562820K->83449K(584192K)]   562972K -> 85107K(1919488K)]
%% [GC(Allocatio Failure)  [PSYoungGen : 549.63M -> 81.49M(570.5M)]    549.78M -> 83.11M(1874.5M)]
```

#### 2.1.2 本地测试导出 堆栈

图片如下：

#### 2.1.3 本地内存分配

```shell
jmap -heap pid  
# 无法查看 本地看不见
```

### 2.2 存在以下问题

> 已经排除的问题：
>
> - `ThreadLocal`未进行remove 导致的内存泄露。
> - `Stream`未释放  导致的内存泄露。

1. Logger打印日志时，日志级别为info, 还是会打印 某些内容，导致堆中该内容很多。需要使用Logger.isDebugAble()方法判断。

2. 以下耗时接口需要前端添加页面限制

   - 企业画像  需要限制查询导出（以及详情的导出）。
   - 主题列表 查看详情时 也需要限制。
   - 企业信息维护：
     - 导出时  搜索条件没有带上。
     - 导入时  导入成功没有提示。（全部更新成功需要提示， 否则弹出框  下载错误数据）
   - 以上

3. 大数据量导出时 需要 `Thread.sleep(0)` ，设置gc的安全点。

4. **发现`ODPS`大数据量时，使用Tunnel，其中`TunnelRecordReader`没有调用`close()`方法。**

5. 添加JVM调优：

   - 设置 Xms 和 Xmx。（使得最大堆内存 =  初始堆内存）

   - 添加各种gc日志和gcDump文件。

     ```java
     -XX:+HeapDumpOnOutOfMemoryError 
     -XX:HeapDumpPath=/Users/peng
     -XX:ErrorFile=/home/app/foo/hs_err_pid%p.log    
     
     -XX:+PrintGC -XX:+PrintGCDetails -XX:+PrintGCTimestamps -XX:+PrintGCApplicationStopedTime   
     jvm将会按照这些参数顺序输出gc概要信息，详细信息，gc时间信息，gc造成的应用暂停时间。
     加入：-Xloggc:文件路径，gc信息将会输出到指定的文件中 
     ```

[JVM参数配置详解及调优](https://blog.csdn.net/lpw_cn/article/details/84594859)

[heap 查看当前jvm堆栈信息_JVM性能调优常用命令](https://blog.csdn.net/weixin_28728443/article/details/113076413)

[Java 垃圾收集日志和如何分析它们 - Sematext](https://sematext.com/blog/java-garbage-collection-logs/)



[♥满满的一整篇，全是 JVM 核心知识点！ (baidu.com)](https://baijiahao.baidu.com/s?id=1687079316127790177&wfr=spider&for=pc)
