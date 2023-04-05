# ELK技术选型

## 1.概念

> - es + kibnana : 4核8G 300GB
>
> - logstash : 4核4G  100G
>

### 1.1 日志打印级别

日志打印通常有四种级别，从高到底分别是:ERROR、WARN、INFO、DEBUG。

日志级别中的优先级是什么意思？在你的系统中如果开启了某一级别的日志后，就不会打印比它级别 低的日志。例如，程序如果开启了 INFO 级别日志，DEBUG 日志就不会打印，通常在生产环境中开启 INFO 日志。

1. DEBUG：DEBUG可以打印出最详细的日志信息，主要用于开发过程中打印一些运行信息。
2. INFO：INFO 可以打印一些你感兴趣的或者重要的信息，这个可以用于生产环境中输出程序运行的一些重要 信息，但是不能滥用，避免打印过多的日志。
3. WARNING： WARNING 表明发生了一些暂时不影响运行的错误，会出现潜在错误的情形，有些信息不是错误信息，但是也要给程序员的一些提示
4. ERROR：ERROR可以打印错误和异常信息，如果不想输出太多的日志，可以使用这个级别，这一级就是比较重要的错误了，软件的某些功能已经不能继续执行了。

### 1.2 组件介绍

> 　“ELK”是三个开源项目的缩写：Elasticsearch，Logstash和Kibana。也称ELK Stack，能够可靠，安全地从任何来源以任何格式获取数据，然后进行实时搜索，分析和可视化。
>
> - **Elasticsearch是搜索和分析引擎,开源的，分布式，RESTful，基于JSON的搜索引擎。它易于使用，可扩展且灵活**。
> - **Logstash是服务器端的数据处理管道，它同时从多个源中提取数据，进行转换，然后将其发送到类似Elasticsearch的“存储”中**。**Kibana允许用户在Elasticsearch中使用图表将数据可视化**。
> - **Beats 是一个免费且开放的平台，集合了多种单一用途数据采集器**。
>
> 它们从成百上千或成千上万台机器和系统向 Logstash或Elasticsearch发送数据。Beats可以将数据直接发送到Elasticsearch或通过 Logstash 发送，然后在Logstash中可以进一步处理和过滤数据，然后再在Kibana中进行可视化 。Beats架构图如下：
>
> ![img](https://img2020.cnblogs.com/blog/1527426/202009/1527426-20200925102409605-650553513.png)

#### 1.2.1 filebeat组件介绍

##### 1.2.1.1 filebeat是什么？

`Filebeat `是用于转发和收集日志数据的轻量级传送工具。Filebeat监视你指定的日志文件或位置，收集日志事件，并将它们转发到 Elasticsearch或Logstash中。

`Filebeat `的工作方式如下：启动 Filebeat 时，它将启动一个或多个输入，这些输入将在为日志数据指定的位置中查找。对于 Filebeat所找到的每个日志，Filebeat 都会启动收集器。每个收集器都读取单 个日志以获取新内容，并将新日志数据发送到 libbeat，libbeat将聚集事件，并将聚集的数据发送到为 Filebeat配置的输出。

> 工作流程图：
>
> ![img](https://img2022.cnblogs.com/blog/1523753/202208/1523753-20220811101343061-667332116.png)
>
> Filebeat有两个主要组件：
>
> - `harvester`：一个harvester负责读取一个单个文件的内容。harvester逐行读取每个文件，并把这些内容发送到输出。每个文件启动一个harvester。
>
> - `Input`：一个input负责管理harvesters，并找到所有要读取的源。如果input类型log，则input查找驱动器上与已定义的log日志路径匹配的所有文件，并为每个文件启动一个harvester。

##### 1.2.1.2 filebeat和beat关系？

`filebeat `是 Beats 中的一员。Beats 是一个轻量级日志采集器，Beats 家族有 6 个成员，早期的 ELK 架构中使用 Logstash 收集、 解析日志，但是 `Logstash `对内存、cpu、io 等资源消耗比较高。相比 Logstash，Beats 所占系统的 CPU 和内存几乎可以忽略不计。

目前 Beats 包含六种工具：

1. Packetbeat：网络数据（收集网络流量数据）
2. Metricbeat：指标（收集系统、进程和文件系统级别的 CPU 和内存使用情况等数据）
3. Filebeat：日志文件（收集文件数据）
4. Winlogbeat：windows 事件日志（收集 Windows 事件日志数据）
5. Auditbeat：审计数据（收集审计日志）
6. Heartbeat：运行时间监控（收集系统运行时的数据）

##### 1.2.1.3  **Filebeat工作原理**

在任何环境下,应用程序都有停机的可能性。Filebeat读取并转发日志行,如果中断,则会记住所有事件恢复联机状态时所在位置。

- `Filebeat带有内部模块`（auditd，Apache，Nginx，System 和 MySQL），可通过一个指定命令来简化 通用日志格式的收集，解析和可视化。
- `FileBeat 不会让你的管道超负荷`。FileBeat如果是向Logstash传输数据，当 Logstash忙于处理数据,会通知FileBeat放慢读取速度。一旦拥塞得到解决,FileBeat将恢复到原来的速度并继续传播。
- `Filebeat 保持每个文件的状态，并经常刷新注册表文件中的磁盘状态`。状态用于记住harvester正在读取的最后偏移量，并确保发送所有日志行。Filebeat 将每个事件的传递状态存储在注册表文件中。所以它能保证事件至少传递一次到配置的输出，没有数据丢失。

 

## 2.技术方案

### 2.1 ELK 

![img](https://img2022.cnblogs.com/blog/1523753/202208/1523753-20220811100351949-1820489271.png)

> `应用程序（AppServer）–>Logstash–>ElasticSearch–>Kibana–>浏览器（Browser）`
>
> Logstash 收集 AppServer 产生的 Log，并存放到 ElasticSearch 集群中，而 Kibana 则从 ElasticSearch集群中查询数据生成图表，再返回给 Browser。
>
> 考虑到聚合端（日志分析处理、清洗等）负载问题和采集端传输效率，一般在日志量比较大的时候在采集端和聚合端增加队列，以用来实现日志消峰。

### 2.2 ELK+filebeat（使用比较多）

![img](https://img2022.cnblogs.com/blog/1523753/202208/1523753-20220811100509622-2087086165.png)

> `Filebeat（采集）—> Logstash（聚合、处理）—> ElasticSearch （存储）—>Kibana （展 示）`

### 2.3 其他方案

> ELK 日志流程可以有多种方案（不同组件可自由组合，根据自身业务配置），常见有以下：
>
> - `Logstash（采集、处理）—> ElasticSearch （存储）—>Kibana（展示）`
> - `Logstash（采集）—> Logstash（聚合、处理）—> ElasticSearch （存储）—>Kibana （展示）`
> - `Filebeat（采集、处理）—> ElasticSearch （存储）—>Kibana （展示）`
> - **`Filebeat（采集）—> Kafka/Redis(消峰) —> Logstash（聚合、处理）—> ElasticSearch （存 储）—>Kibana （展示）`**

## 3.方案优劣

### 3.1 常见日志采集工具对比分析

> 常见的日志采集工具有 Logstash、Filebeat、Fluentd 等等，那么他们之间 有什么区别呢?什么情况下我们应该用哪一种工具

|    名称    |                             备注                             |                             优势                             |                             劣势                             |                         典型应用场景                         |
| :--------: | :----------------------------------------------------------: | :----------------------------------------------------------: | :----------------------------------------------------------: | :----------------------------------------------------------: |
| `Logstash` | Logstash 是一个开源数据收集引擎，具有实时管道功能。Logstash 可以动态地将来自不同数据源的数据统一起来，并将数据标准化到你所选择的目的地。 | Logstash 主要的优点就是它的灵活性，主要因为它有很多插件，详细的文档以及直白的配置格式 让它可以在多种场景下应用。我们基本上可以在网上找到很多资源，几乎可以处理任何问题。 | `Logstash 致命的问题是它的性能以及资源消耗(默认的堆大小是 1GB)`。尽管它的性能在近几年已 经有很大提升，与它的替代者们相比还是要慢很多的。它在大数据量的情况下会是个问题。 另一个问题是它目前不支持缓存，目前的典型替代方案是将 Redis 或 Kafka 作为中心缓冲池 | 因为Logstash自身的灵活性以及网络上丰富的资料，Logstash适用于原型验证阶段使用，或者`解析非常的复杂的时候`。在不考虑服务器资源的情况下，如果服务器的性能足够好，我们也可以为每台 服务器安装Logstash。我们也不需要使用缓冲，因为文件自身就有缓冲的行为，而Logstash也会记住上次处理的位置。**`如果服务器性能较差，并不推荐为每个服务器安装 Logstash ，这样就需要一个轻量的日志传输工具，将数据从服务器端经由一个或多个 Logstash 中心服务器传输到 Elasticsearch`**。 |
| `Filebeat` | 作为Beats家族的一员，Filebeat是一个轻量级的日志传输工具，它的存在正弥补了`Logstash`的缺点：**`Filebeat 作为一个轻量级的日志传输工具可以将日志推送到中心 Logstash`**。在版本5.x 中，Elasticsearch具有解析的能力(像Logstash过滤器)— Ingest。这也就意味 着可以将数据直接用 Filebeat 推送到 Elasticsearch，并让 Elasticsearch 既做解析的事情，又做存储的事情。也不需要使用缓冲，因为 Filebeat 也会和 Logstash 一样记住上次读取的偏移，如果需要缓冲(例如，不希望将日志服务器的文件系统填满)，可以使用 Redis/Kafka，因为 Filebeat 可以与它们进行通信 | `Filebeat只是一个二进制文件没有任何依赖。`它占用资源极少，尽管它还十分年轻，正式因为它简单，所以几乎没有什么可以出错的地方，所以它的可靠性还是很高的。它也为我们提供了很多可以调 节的点，例如：它以何种方式搜索新的文件，以及当文件有一段时间没有发生变化时，何时选择关闭文 件句柄。 | `Filebeat 的应用范围十分有限，所以在某些场景下我们会碰到问题`。例如，如果使用 Logstash 作为下游管道，我们同样会遇到性能问题。正因为如此，Filebeat 的范围在扩大。开始时，它只能将日志发送到 Logstash 和 Elasticsearch，而现在它可以将日志发送给 Kafka 和 Redis，在 5.x 版本 中，它还`具备过滤的能力`。 |                     轻量级的日志传输工具                     |
| `Fluentd`  | Fluentd 创建的初衷主要是尽可能的使用JSON 作为日志输出，所以传输工具及其下游的传输线 不需要猜测子字符串里面各个字段的类型。这样，它为几乎所有的语言都提供库，这也意味着，我们可以将它插入到我们自定义的程序中。 | 多数应用场景下，我们会通过Fluentd 得到结构化的数据，它的灵活性并不好。但仍然可以通过正则表达式，来解析非结构化的数据。性能在大多数场景下都很好，但它并不是最好的，大的节点下性能是受限的，资源消耗在大多数场景下是可以接受的。 |                              -                               | `Fluentd 在日志的数据源和目标存储各种各样时非常合适`，因为它有很多插件。而且，如果大多数数据源都是自定义的应用，所以可以发现用 fluentd 的库要比将日志库与其他传输工具结合起来要容易很多。特别是在我们的应用是多种语言编写的时候，即我们使用了多种日志库，日志的行为也不太一 样。 |

## 4.安装ELK

> ## 下载安装包
>
> - [elasticsearch](https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-8.5.3-linux-x86_64.tar.gz)
>
> - [kibana](https://artifacts.elastic.co/downloads/kibana/kibana-8.5.3-linux-x86_64.tar.gz)
>
> - [logstash](https://artifacts.elastic.co/downloads/logstash/logstash-8.5.3-linux-x86_64.tar.gz)
>
> - [filebeat](https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-8.5.3-linux-x86_64.tar.gz)
>
> - [metricbeat](https://artifacts.elastic.co/downloads/beats/metricbeat/metricbeat-8.5.3-linux-x86_64.tar.gz)
>
> 下载完成如下：
>
> ```shell
> [esuser@VM-12-16-centos elk]$ pwd
> /data/elk
> [esuser@VM-12-16-centos elk]$ ll
> total 1195028
> -rw-r--r-- 1 root root 580678137 Dec  9 01:01 elasticsearch-8.5.3-linux-x86_64.tar.gz
> -rw-r--r-- 1 root root  40125224 Dec  9 00:52 filebeat-8.5.3-linux-x86_64.tar.gz
> -rw-r--r-- 1 root root 223252007 Dec  9 01:13 kibana-8.5.3-linux-x86_64.tar.gz
> -rw-r--r-- 1 root root 331310588 Dec  9 01:17 logstash-8.5.3-linux-x86_64.tar.gz
> -rw-r--r-- 1 root root  48322398 Dec  9 00:56 metricbeat-8.5.3-linux-x86_64.tar.gz
> ```

### 4.1 ES安装

> 安全原因，ES不能使用root用户启动，所以先创建新用户。

```powershell
useradd -m  esuser
passwd esuser    回车输入密码: yhl19980410

授予esuser用户目录权限
chown -R esuser:esuser /data/elk

切换到esuser
su - esuser
解压
tar -zxvf elasticsearch-8.5.3-linux-x86_64.tar.gz

#查看已开启的端口
firewall-cmd --list-ports
# 防火墙状态
firewall-cmd --state 
# 如果是running 执行开启端口
firewall-cmd --zone=public --add-port=9200/tcp --permanent
# 重启防火墙
systemctl restart firewalld.service
# 重新加载防火墙
firewall-cmd --reload
```

解压后config目录下有三个配置文件

```
elasticsearch.yml   es相关设置
jvm.options         Java虚拟机相关设置，主要就堆大小设置，可以先默认。
log4j2.properties   日志相关设置，可以先默认。
```

#### 4.1.1 elasticsearch.yml

```yaml
cluster.name: elk-cluster
network.host: 0.0.0.0
ingest.geoip.downloader.enabled: false

#----------------------- BEGIN SECURITY AUTO CONFIGURATION -----------------------
#
# The following settings, TLS certificates, and keys have been automatically      
# generated to configure Elasticsearch security features on 20-12-2022 14:19:28
#
# --------------------------------------------------------------------------------

# Enable security features
xpack.security.enabled: false  ## 改为false，该字段为true,页面访问9200时需要输入密码。
xpack.security.enrollment.enabled: true

# Enable encryption for HTTP API client connections, such as Kibana, Logstash, and Agents
xpack.security.http.ssl:
  enabled: false   ## 改为false
  keystore.path: certs/http.p12
  
# Enable encryption and mutual authentication between cluster nodes
xpack.security.transport.ssl:
  enabled: true
  verification_mode: certificate
  keystore.path: certs/transport.p12
  truststore.path: certs/transport.p12
# Create a new cluster with the current node only
# Additional nodes can still join the cluster later
cluster.initial_master_nodes: ["localhost.localdomain"]

#----------------------- END SECURITY AUTO CONFIGURATION -------------------------
```

#### 4.1.2 jvm.options

```powershell
########################################### 只需要改这个就行 其余默认  
-Xms3g  ## 测试环境 给3G
-Xmx3g
###########################################

-XX:+UseG1GC
-Djava.io.tmpdir=${ES_TMPDIR}
-XX:+HeapDumpOnOutOfMemoryError
-XX:+ExitOnOutOfMemoryError
-XX:HeapDumpPath=data
-XX:ErrorFile=logs/hs_err_pid%p.log
-Xlog:gc*,gc+age=trace,safepoint:file=logs/gc.log:utctime,pid,tags:filecount=32,filesize=64m
```

>切换回root用户，对系统做如下配置，否则启动es可能报错
>
>/etc/sysctl.conf
>
>```powershell
>#不设置可能报这个错，虚拟内存不够。
>#max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]
>vm.max_map_count=262144
>
>保存然后执行命令刷新
>sysctl -p
>```
>
>/etc/security/limits.conf
>
>```powershell
>#不设置可能报这个错，用户能创建的线程数不够
>#max number of threads [1024] for user [esuser] is too low, increase to at least [4096]
>esuser - nproc 4096
>
>#不设置可能报这个错，用户能打开的最大文件数不够
>#max file descriptors [65535] for elasticsearch process is too low, increase to at least [65536]
>esuser - nofile 65536
>
>* soft nofile 2048
>* hard nofile 131072
>
>保存后退出账号重新登录即可生效
>```
>
>使用esuser账号在bin目录下 执行如下命令 启动es
>
>```powershell
>./elasticsearch 
>
>#要将Elasticsearch作为守护程序运行，命令如下，第一次建议按上述命令启动，可查看报错信息以及密码令牌等信息
>./elasticsearch -d -p pid
>```
>
>- ES首次启动有如下默认行为
>
> ```apl
> - 启用身份验证和授权，并生成密码 内置超级用户。`elastic`
> - TLS 的证书和密钥是为传输层和 HTTP 层生成的， 并且 TLS 已启用并配置了这些密钥和证书。
> - 将为 Kibana 生成一个注册令牌，有效期为 30 分钟。
> ```
>
>输出如下：

```xx
✅ Elasticsearch security features have been automatically configured!
✅ Authentication is enabled and cluster connections are encrypted.

ℹ️  Password for the elastic user (reset with `bin/elasticsearch-reset-password -u elastic`):
  dZd+BYQfgJjRI+6PBz8r

ℹ️  HTTP CA certificate SHA-256 fingerprint:
  e0154bb9c7b494b45356a0a12ec2d3d9029f27aa3b6a7c2411c2844bb14314da

ℹ️  Configure Kibana to use this cluster:
• Run Kibana and click the configuration link in the terminal when Kibana starts.
• Copy the following enrollment token and paste it into Kibana in your browser (valid for the next 30 minutes):
  eyJ2ZXIiOiI4LjUuMyIsImFkciI6WyIxNzIuMTguMC4xOjkyMDAiXSwiZmdyIjoiZTAxNTRiYjljN2I0OTRiNDUzNTZhMGExMmVjMmQzZDkwMjlmMjdhYTNiNmE3YzI0MTFjMjg0NGJiMTQzMTRkYSIsImtleSI6ImZ2em5MNFVCdXhoNm00UWZ4T0F2OnFOcHpVT2JSUVc2NktlT3kwOXRrc0EifQ==

ℹ️ Configure other nodes to join this cluster:
• Copy the following enrollment token and start new Elasticsearch nodes with `bin/elasticsearch --enrollment-token <token>` (valid for the next 30 minutes):
  eyJ2ZXIiOiI4LjUuMyIsImFkciI6WyIxNzIuMTguMC4xOjkyMDAiXSwiZmdyIjoiZTAxNTRiYjljN2I0OTRiNDUzNTZhMGExMmVjMmQzZDkwMjlmMjdhYTNiNmE3YzI0MTFjMjg0NGJiMTQzMTRkYSIsImtleSI6ImdQem5MNFVCdXhoNm00UWZ4T0EyOjNFTi1hdlNTUmd1SlZwN1JKN3I1blEifQ==

  If you're running in Docker, copy the enrollment token and run:
  `docker run -e "ENROLLMENT_TOKEN=<token>" docker.elastic.co/elasticsearch/elasticsearch:8.5.3`
```

- 所以浏览器用https访问9200端口号(云服务器注意防火墙端口号是否打开)

#### 4.1.3 组件安全

 vim /data/elk/elasticsearch-8.5.3/config/elasticsearch.yml

```powershell
xpack.security.enabled: true   ## 组件安全这里需要开启
xpack.security.transport.ssl:
  enabled: true  ## 组件安全这里需要开启
  
 ## 完整如下: 



#----------------------- BEGIN SECURITY AUTO CONFIGURATION -----------------------
#
# The following settings, TLS certificates, and keys have been automatically      
# generated to configure Elasticsearch security features on 20-12-2022 14:19:28
#
# --------------------------------------------------------------------------------

# Enable security features
xpack.security.enabled: true   ## 组件安全这里需要开启
xpack.security.enrollment.enabled: true

# Enable encryption for HTTP API client connections, such as Kibana, Logstash, and Agents
xpack.security.http.ssl:
  enabled: false   ## 改为false
  keystore.path: certs/http.p12
  
# Enable encryption and mutual authentication between cluster nodes
xpack.security.transport.ssl:
  enabled: true  ## 组件安全这里需要开启
  verification_mode: certificate
  keystore.path: certs/transport.p12
  truststore.path: certs/transport.p12
# Create a new cluster with the current node only
# Additional nodes can still join the cluster later
cluster.initial_master_nodes: ["localhost.localdomain"]

#----------------------- END SECURITY AUTO CONFIGURATION -------------------------

```

> kibana的登陆用户不能使用 `elastic`用户登录。所以使用es新建用户。
>
> ```powershell
> /data/elk/elasticsearch-8.5.3/bin/elasticsearch-users useradd  moduleAdmin
> zjdsj@401
> 
> 
> /data/elk/elasticsearch-8.5.3/bin/elasticsearch-users roles -a superuser moduleAdmin
> /data/elk/elasticsearch-8.5.3/bin/elasticsearch-users roles -a kibana_system  moduleAdmin
>  /data/elk/elasticsearch-8.5.3/bin/elasticsearch-users roles -a logstash_system  moduleAdmin
> ```

使用该功能[^nginx代理免密登陆]，否则需要弹出框。

### 4.2 kianan安装

> kibana默认也不允许用root账号启动，这里也用esuser账号操作

1. 解压

   ```powershell
   tar -zxvf kibana-8.5.3-linux-x86_64.tar.gz
   ```

2. config/kibana.yml 添加如下配置

    ```powershell
    #设置端口
    server.port: 5601
    #设置主机ip,不设置默认localhsot,远程无法访问，只能本机访问
    server.host: "0.0.0.0"
    #设置中文
    i18n.locale: "zh-CN"
    
    ## 好像不行 研究研究
    elasticsearch.username: "moduleAdmin"
    elasticsearch.password: "zjdsj@401"
    ```

3. 在bin目录下启动

    ```powershell
    ./kibana
    ```

4. 访问浏览器，端口 5601 即可。

### 4.3 Logstash安装

- ~[^Logstash Guides]

- 添加配置文件

  > logstash-simple.conf内容如下，表示从标准输入，filebeat输入，两个过滤器，然后从标准出书和es输出。
  >
  > - 分页查询 某个库下面的某张表记录，记录其 `更新时间`供下次使用（注意：自行创建所需的目录及文件）。
  > - 同时，同样的对`type以oper_log`结尾的类型，需要对`@timestamp`进行处理。（es默认搜索的指定时间，默认为导入时间，格式为`UTC`）.
  > - 将 `es的index中的文档id`指定为 表的主键。

  ```yaml
  # Sample Logstash configuration for creating a simple
  # Beats -> Logstash -> Elasticsearch pipeline.
  input {
          jdbc {
                  jdbc_driver_library => "/data/elk/logstash-8.5.3/logstash-core/lib/jars/mysql-connector-java-8.0.18.jar" #注意该jar包 logstash本身并没有，需要手动拷入
                  jdbc_driver_class => "com.mysql.cj.jdbc.Driver"
                  jdbc_connection_string => "jdbc:mysql://192.168.115.30:3306/sys?useUnicode=true&characterEncoding=UTF-8&autoReconnect=true&useSSL=false&zeroDateTimeBehavior=convertToNull"
                  jdbc_user => "root"
                  jdbc_password => "austin_pwd+-"
                  
                  # 正式环境必须关闭
                  clean_run => true
                  # 指定多久执行一次数据输出  目前是1分钟
                  schedule => "* * * * *"
                  statement => "SELECT oper_id,set_by,`value`,variable,set_time as oper_time FROM sys_config WHERE set_time > :sql_last_value" # 注意 数据库字段必须驼峰命名，可以as指定别名
                  
                  # 值为true 开启追踪
                  use_column_value => true
                  tracking_column => "oper_time"
                  # 追踪字段类型 目前只有数字（numeric）和 时间类型（timestamp）  默认数字
                  tracking_column_type => 'timestamp'
                  type => "sys_config_oper_log"
                  
                  #JDBC启用分页
                  jdbc_paging_enabled => true
                  # 每页查询多少条
                  jdbc_page_size => "500"
                  last_run_metadata_path => "/data/elk/logstash-8.5.3/sysMetadata/sysConfig-jdbcPosition.txt"
                  
                  # 开启记录最后一次运行的结果
                  record_last_run => true
          }
  }
  
  filter {
          mutate {
                  add_field => {
                          "[@metadata][oper_id]" => "%{oper_id}"
                  }
          }
          # 操作记录 修改记录时间（oper_log）
          if [type] =~ /oper_log$/ {
                  mutate { add_field => {"tmp_timestamp" => "%{oper_time}"}}
                  # 对oper_time格式化，然后赋值给 @timestamp
                  date {
                          match => ["tmp_timestamp","ISO8601"]
                          target => "@timestamp"
                  }
                  mutate { remove_field => ["tmp_timestamp"]}
          }
  }
  
  output {
          if [type] == "sys_config_oper_log" {
                  elasticsearch {
                          hosts => ["192.168.115.30:9200"]
                          index => "sys_config"
                          action => "index"
                          document_id => "%{[@metadata][oper_id]}"
                          #user => "moduleAdmin"
  		      #password => "zjdsj@401"
                  }
          }
  }
  ```

  - 一个简单的示例。

    - ![image-20230321145625818](https://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202303211456911.png)

    - ```shell
      input {stdin{}} #从控制台输入
      
      filter {
      	grok {
      		match =>{
      			"message" =>"%{TIMESTAMP_ISO8601:oper_time}"
      		}
      	}
      	date {
      		match =>["oper_time","yyyy-MM-dd HH:mm:ss,SSS"]
      		target =>"@timestamp"
      	}
      	# 此处需要开启东八区
      	ruby {
      		code =>"event.set('temp', event.get('@timestamp').time.localtime + 8*60*60); event.set('@timestamp', event.get('temp'))"
      	}
      	mutate {remove_field =>["temp","message"]}
      }
      
      output {stdout{codec=>rubydebug}}
      ```

  - 启动如下：

    ```shell
    # 启动：
    bin/logstash -f logstash-custom.conf
    # 检查配置文件是否有问题
    bin/logstash -f logstash-custom.conf --config.test_and_exit
    # 重载配置文件
    bin/logstash -f logstash-custom.conf --config.reload.automatic
    ```

### 4.4 安装filebeat

> 某些特殊情况，需要解析日志文件获取`操作日志`。

- ~[^Filebeats Guides]

- > ~[^filebeat 配置]
  >
  > ~[^Beats:避免重复数据导入]

- 添加配置文件

  ```shell
  # filebeat-config.conf
  filebeat.inputs:
  - type: log
    paths:
     - /home/admin/logs/user-access.log.* # T+1取记录  当天的为user-access.log，前天为user-access.log.2023-03-01
    tags: ["drp_test_run_log"]   # 自定义标识
    exclude_lines :[“^gatway”]   #排除行
    close_eof : true
    max_bytes: 104857600 # 单个文件读取的最大字节数，默认10M，此处100M
    fields:
    	from: drp.test  # 添加自定义属性
    	itcase: "11.48.22.xx"
  out.logstash:
    	host: ["192.168.115.30"]
    	pretty: true # 打印数据
    	enable: true
  ```

- 启动如下,使用详细见[^filebeat use]。

  ```shell
  ./filebeat -e -c filebeat配置文件
  ```

- 对应的logstash如下：

  ```shell
  input {
  	beats {
  		host => "0.0.0.0"
  		port => 5044
  	}
  }
  
  filter {
  	#日志如下： 
  	#2023-03-20 14:53:59.712 [http-nio-8080-exec-6] INFO  USER-ACCESS - ms=499614|traceId=1e0db3b51679295239163102014376|serverIp=30.13.179.181|clientIp=0:0:0:0:0:0:0:1|userId=regulatory_admin|userName=监管 管理员|m=POST|url=http://localhost/drp/reportSummary/reportSummaryList.json|referer=http://localhost:8080/|paras=|resp=200|reqPL={"pageNo":1,"pageSize":10,"reportDisplayName":null,"orderDirection":"ASC","sortBy":null,"startTime":null,"endTime":null,"reportStatus":null,"visualType":"REPORT","forceRefresh":true}
  	grok {
  		#DEF_STR [a-z0-9]+
  		#DEF_B_STR [A-Z0-9]+
  		#URL (http(s)?:\/\/)?%{URIHOST}%{URIPATH}
  		patterns_dir => ["./patterns"] # 在logstash根目录下建一个文件夹，创建文件即可.
  		match => {
  			# [T ]?(?<userId>[\S+[T ]?]+)  userId、userName存在‘ ’的情况，\S+无法识别空格。
                  "message"  =>"^%{TIMESTAMP_ISO8601:oper_time}[T ]+\[%{DATA:current_thread}\][T ]+%{LOGLEVEL:level}[T ]+USER-ACCESS[T ]+-[T ]+ms=(?<ms>[0-9.]+)\|traceId=%{DEF_STR:traceId}\|serverIp=%{IPV4:serverIp}\|clientIp=%{URIHOST:clientIp}\|userId=[T ]?(?<userId>[\S+[T ]?]+)\|userName=[T ]?(?<userName>[\S+[T ]?]+)\|m=%{WORD:req_method}\|url=%{URL:url}\|(?<param>\S+)"
  		}
  	}
  	date {
  		match => ["oper_time","yyyy-MM-dd HH:mm:ss.SSS"]
  		target => "@timestamp"
  	}
  	# 东8区 只有控制台输入才需要添加
  	#ruby {
  	#	code => "event.set('timestamp', event.get('@timestamp').time.localtime + 8*60*60); event.set('@timestamp', event.get('timestamp'))"
  	#}
  	
  	## urlPath为 `接口名称`，需要配置为`真实模块名称`，此处使用ruby处理 【moduleName】
  	ruby {
  		path => "./config/extraDrp.r"  # 这里的相对路径 相对于bin/logstash 来说的	
  	}
  	mutate {remove_field => ["timestamp","message","current_thread","level"]}
  }
  
  output {
  	if ("drp_test_run_log" in [tags]){
  		elastisearch {
  			hosts => ["192.168.15.30"]
  			index => "drp_test_run_log-%{+YYYY.MM.dd}" 
  		}
  	}
  }
  ```
  
  > Filebeat传过来的数据，其中`urlPath`为`接口名称`，类似于 `/upload/data.json`.
  >
  > 但是在es真实使用过程中，是需要显示为`模块名称`：
  >
  > 1. 如果指定`urlPath`的字段格式配置为 `静态查找`，可以解决`Discover`中关于表格的显示问题。
  > 2. 但是`Dashboard`等图表需要 将 字段配置为`keyword`，如果为`keyword`，此时就算配置了`静态查找`都不管作用。（指`urlPath`、`urlPath.keyword`都配置了`静态查找`）
  >
  > 所以，可以**使用ruby代码将`urlPath`转换为另一个字段`moduleName`，图表展示的时候，使用该字段进行 `聚合显示`**.
  >
  > - extraDrp.r如下
  >
  >   ```ruby
  >   # the value of `params` is the value of the hash passed to `script_params`
  >   # in the logstash configuration
  >   def register(params)
  >       # 这里通过params获取的参数是在logstash文件中通过script_params传入的
  >       @message = params["message"]
  >   	  
  >   	@map = Hash.new('系统内置')
  >   	@map.store("/upload/data.json", "【系统上传接口】")
  >   	  
  >   end
  >     
  >   # the filter method receives an event and must return a list of events.
  >   # Dropping an event means not including it in the return array,
  >   # while creating new ones only requires you to add a new instance of
  >   # LogStash::Event to the returned array
  >   # 这里通过参数event可以获取到所有input中的属性
  >   def filter(event)
  >   	urlPath = event.get('urlPath')
  >   	if event.include?('urlPath')
  >   		event.set('moduleName', @map[urlPath])
  >   	else
  >   		event.set('moduleName', '系统内置')
  >   	end	
  >   	#event.set('for_time', (event.get('@timestamp').time.localtime + 8*60*60).strftime('%Y-%m-%d %H:%M:%S'))
  >   	#event.set('for_date', (event.get('@timestamp').time.localtime + 8*60*60).strftime('%Y-%m-%d'))
  >       return [event]
  >   end
  >     
  >   ```
  
  
  
  
  
  - ![image-20230321113848477](https://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202303211138561.png)
  - ~[^Grok use]。可以在kibana中的 `Management -》 开发工具 -》Grok Debugger`查看。
  - Logstash Grok[^Grok详解]

## 5.总结

- [^Elastic开发指南总览]:[ Elastic：开发者上手指南](https://elasticstack.blog.csdn.net/article/details/102728604)

- 构建

  - [^EBLK]:[EBLK之2---EBLK - 原因与结果 - 博客园 (cnblogs.com)](https://www.cnblogs.com/backups/p/EBLK_2.html)

  - [^EFK+Logstash]: [EFK+logstash构建日志收集平台 - 杨梅冲 - 博客园 (cnblogs.com)](https://www.cnblogs.com/yangmeichong/p/16574693.html)

  - [^ELK+Kafaka+Beats]:[ELK+Kafka+Beats实现海量日志收集平台（一） - tianjh - 博客园 (cnblogs.com)](https://www.cnblogs.com/jhtian/p/13728590.html)

- Elastic Search

  - [^nginx代理免密登陆]:[[Nginx\]反向代理kibana添加basic认证](https://www.shuzhiduo.com/A/B0zqKoQ85v/)、[elasticsearch设置登录用户名和密码 & nginx代理免密登录](https://blog.csdn.net/paulluo0739/article/details/114835823)

- Logstash

  - [^Logstash:Apache导入到es中]:[Logstash：把 Apache 日志导入到 Elasticsearch_Elastic 中国社区官方](https://blog.csdn.net/UbuntuTouch/article/details/100727051)

  - [^Logstash Guides]:[Logstash官方配置参考](https://www.elastic.co/guide/en/logstash/current/configuration.html)

  - [^Logstash 配置详解]:[`Logstash配置详解_梦想的征途的博客`](https://blog.csdn.net/u012017645/article/details/127319892)

  - [^Grok use]:[Grok正则使用](https://blog.csdn.net/qq_36025814/article/details/108825120)

  - [^Grok详解]:[Logstash Grok详解_叱咤少帅（少帅）的博客](https://blog.csdn.net/knight_zhou/article/details/104954098)

- Filebeat

  - [^filebeat use]:[Filebeat的基本使用_码农的进阶之路的博客](https://blog.csdn.net/zyxwvuuvwxyz/article/details/108831962)

  - [^Filebeats Guides]:[Filebeats 官方配置参考](https://www.elastic.co/guide/en/beats/filebeat/7.17/filebeat-input-log.html#fields-under-root-log)

  - [^filebeat 配置]:[`Filebeat的一些重要配置`](https://blog.csdn.net/u013613428/article/details/113480948)、[`Filebeat常见配置参数解释`](https://blog.51cto.com/ghl1024/5340920)

  - [^Beats:避免重复数据导入]:[Beats：如何避免重复的导入数据_add_id processor_Elastic 中国社区](https://blog.csdn.net/UbuntuTouch/article/details/114007425)

  - [^Filebeat摄入网络服务数据]:[Beats：使用 Filebeat 中的 HTTP JSON input 来摄入网络服务数据_Elastic 中国社区](https://elasticstack.blog.csdn.net/article/details/121333075)

### 5.1 kibana免密登录[^nginx代理免密登陆]

> 内部开启默认的登陆验证，直接访问es或kibana也需要密码。通过nginx代理就不需要了。

#### 5.1.1 nginx代理

docker-compose.yml

```yml
version: '3.1'
services:
    nginx:
        image: nginx     # 镜像名称
        container_name: nginx     # 容器名字
        restart: always     # 开机自动重启
        ports:     # 端口号绑定（宿主机:容器内）
            - '8080:80'
            - '443:443'
        volumes:      # 目录映射（宿主机:容器内）
            - ./conf/nginx.conf:/etc/nginx/nginx.conf ###配置文件
            - ./log:/var/log/nginx  ###这里面放置日志
            - ./html:/html         ###这里面
```

config/nginx.conf

```shell
worker_processes  1;
events {
   worker_connections  1024;
}
http {
   include       mime.types;
   default_type  application/octet-stream;
   sendfile        on;
   keepalive_timeout  65;
   server {
       listen       80;
       #该ip是虚拟机地址，根据实际情况修改。
       server_name 192.168.115.30;
     #  location / {
     #     # Nginx解决浏览器跨域问题
     #     add_header Access-Control-Allow-Origin *;
     #     add_header Access-Control-Allow-Headers X-Requested-With;
     #     add_header Access-Control-Allow-Methods GET,POST,PUT,DELETE,PATCH,OPTIONS;
     #     root   /html;
     #     index  index.html index.htm;
     #  }
	
	 
#server_name 以下都为 '/kibana'
	## rewrite_path : true
	## 此时js访问 ： http://192.168.115.30:5601/【kibana】/57217/bundles/plugin/unifiedSearch/1.0.0/unifiedSearch.chunk.7.js
    location ^~ /kibana  {
			proxy_pass   http://192.168.115.30:5601/kibana/;
			proxy_set_header  Host $host;
			proxy_set_header  X-Real-IP $remote_addr;
			proxy_set_header  X-Forwarded-For $proxy_add_x_forwarded_for;
			proxy_set_header  Authorization "Basic bW9kdWxlQWRtaW46empkc2pANDAx";  # Basic base64(username:password)
			rewrite ^/(.*)$ /$1 break;
	}
	 
   }
}

echo -n moudlueAdmin:zjdsj@401 | base64
```

#### 5.1.2 es配置

/data/elk/elasticsearch-8.5.3/config/elasticsearch.yml

```yaml
cluster.name: elk-cluster
#path.data: /path/to/data
#path.logs: /path/to/logs
bootstrap.memory_lock: true

network.host: 0.0.0.0
ingest.geoip.downloader.enabled: false
cluster.initial_master_nodes: ["localhost.localdomain"]

xpack.security.enabled: true  ##
xpack.security.enrollment.enabled: true
xpack.security.http.ssl:
  enabled: false
  keystore.path: certs/http.p12
xpack.security.transport.ssl:
  enabled: true  ###
  verification_mode: certificate
  keystore.path: certs/transport.p12
  truststore.path: certs/transport.p12
```

#### 5.1.3 kibana配置

 /data/elk/kibana-8.5.3/config/kibana.yml

```yaml
server.port: 5601
server.name: kibana
server.basePath: "/kibana"
server.rewriteBasePath: true

server.host: "0.0.0.0"
i18n.locale: "zh-CN"
elasticsearch.username: "moduleAdmin"
elasticsearch.password: "zjdsj@401"
```

#### 5.1.4 logstassh配置

/data/elk/logstash-8.5.3/config/logstash-custom.conf

```yaml
## ... 省略了一些
output {
        if [type] == "sys_config_oper_log" {
                elasticsearch {
                        hosts => ["192.168.115.30:9200"]
                        index => "sys_config"
                        action => "index"
                        document_id => "%{[@metadata][oper_id]}"
                        user => "moduleAdmin"  ## 放开
		       password => "zjdsj@401" ## 放开
                }
        }
```

#### 5.1.5 效果图

> 1. 可以新建个用户、角色
>
>    <img src="https://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202303291621939.jpeg" style="zoom: 5%;" />
>
>    <img src="https://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202303291622074.png" alt="image-20230329162229011" style="zoom:15%;" />
>
> 2. 将nginx中的 认证改为 新建用户的认证，这样`viewUser`可以免密访问。
>
> 3. 在`隐私页`中通过`http://192.168.115.30:5601/kibana`指定账号登陆。

- 直接访问`es:9200`没有问题

  <img src="https://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202303131025370.png" alt="image-20230313100417082" style="zoom:25%;" />

- [`直接访问`](http://192.168.115.30:5601/kibana/)也没有问题

  <img src="https://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202303291119086.png" alt="image-20230329111923015" style="zoom: 25%;" />

- 访问 [`nginx代理地址`](http://192.168.115.30:8080/kibana/)也没有问题

  <img src="https://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202303131025858.png" alt="image-20230313100753550" style="zoom:25%;" />

### 5.2 索引生命周期管理

[Elasticsearch：Index 生命周期管理入门_Elastic 中国社区官方博客](https://elasticstack.blog.csdn.net/article/details/102728987)

[Logstash：为 Logstash 日志启动索引生命周期管理_Elastic 中国社区官方博客](https://blog.csdn.net/UbuntuTouch/article/details/110816948)

### 5.3 回滚更新

> 本来打算使用`索引生命周期`管理解决该问题，发现有点难理解。替代方案。
>
> > 目前项目 存在两种 index
> >
> > - `项目稀少`：大批量的操作记录。（日增40w+）
> > - `项目比较多`: 操作日志较少。
>
> 针对这两种情况，决定对这两种索引的执行方式不太一样。
>
> 1. logstash => es, index后面添加 `日期`后缀
> 2. logstash => es, 全部数据放入到一个 index中。

#### 5.3.1 logstash.conf

```shell
input {
	jdbc {
        id=> "operLog"
		jdbc_driver_library => "/data/elk/logstash-8.5.3/logstash-core/lib/jars/mysql-connector-java-8.0.18.jar"
		jdbc_driver_class => "com.mysql.cj.jdbc.Driver"
		jdbc_connection_string => "jdbc:mysql://192.168.115.30:3306/sys?useUnicode=true&characterEncoding=UTF-8&autoReconnect=true&useSSL=false&zeroDateTimeBehavior=convertToNull"
		jdbc_user => "root"
		jdbc_password => "austin_pwd+-"
		
		# 正式环境必须关闭
		clean_run => true
		# 指定多久执行一次数据输出  目前是1分钟
		schedule => "*/10 * * * *"
		statement => "SELECT oper_id,set_by,`value`,variable,set_time as oper_time FROM sys_config WHERE set_time > :sql_last_value"
		
		# 值为true 开启追踪
		use_column_value => true
		tracking_column => "oper_time"
		# 追踪字段类型 目前只有数字（numeric）和 时间类型（timestamp）  默认数字
		tracking_column_type => 'timestamp'
		type => "sys_config_oper_log"
		
		#JDBC启用分页
		jdbc_paging_enabled => true
		# 每页查询多少条
		jdbc_page_size => "500"
		last_run_metadata_path => "/data/elk/logstash-8.5.3/sysMetadata/sysConfig-jdbcPosition.txt"
		
		# 开启记录最后一次运行的结果
		record_last_run => true
	}
}

# 大数据导入 此时根据id进行追踪
input {
	jdbc {
		id => "big_oper_log"
		jdbc_driver_library => "/data/elk/logstash-8.5.3/logstash-core/lib/jars/mysql-connector-java-8.0.18.jar"
		jdbc_driver_class => "com.mysql.cj.jdbc.Driver"
		jdbc_connection_string => "jdbc:mysql://192.168.115.30:3306/sys?useUnicode=true&characterEncoding=UTF-8&autoReconnect=true&useSSL=false&zeroDateTimeBehavior=convertToNull"
		jdbc_user => "root"
		jdbc_password => "austin_pwd+-"
		
		# 正式环境必须关闭
		clean_run => false
		# 指定多久执行一次数据输出  目前是5分钟
		schedule => "* * * * *"  # 限制查询最大条数 10w
		statement => "SELECT a.*, b.`name` as oper_real_name  FROM sys_user b  RIGHT JOIN sys_oper_log a  ON a.oper_name = b.user_name  WHERE a.oper_id > :sql_last_value LIMIT 100000"
		
		# 值为true 开启追踪
		use_column_value => true
		tracking_column => "oper_id"
		# 追踪字段类型 目前只有数字（numeric）和 时间类型（timestamp）  默认数字
		tracking_column_type => 'numeric'
		type => "big_oper_log"
		
		#JDBC启用分页
		jdbc_paging_enabled => true
		# 每页查询多少条
		jdbc_page_size => "1000"
		last_run_metadata_path => "/data/elk/logstash-8.5.3/sysMetadata/bigOperLog-jdbcPosition.txt"
		
		# 开启记录最后一次运行的结果
		record_last_run => true
	}
}

filter {
	mutate {
		add_field => {
			"[@metadata][oper_id]" => "%{oper_id}"
		}
	}
	# 操作记录 修改记录时间（oper_log）
	if [type] =~ /oper_log$/ {
		mutate { add_field => {"tmp_timestamp" => "%{oper_time}"}}
		# 对oper_time格式化，然后赋值给 @timestamp
		date {
			match => ["tmp_timestamp","ISO8601"]
			target => "@timestamp"
		}
		mutate { remove_field => ["tmp_timestamp"]}
	}
	
	if [type] == "big_oper_log" {
		ruby {
			# 此处同一个操作时间段的数据  会进入同一个索引
			code => "event.set('est_time', event.get('[@timestamp]').time.localtime.strftime('%Y-%m-%d'))" #将转换后的时间set到一个字段中
		}
	}
}

output {
	if [type] == "sys_config_oper_log" {
		elasticsearch {
			hosts => ["192.168.115.30:9200"]
			index => "sys_config"
			action => "index"
			document_id => "%{[@metadata][oper_id]}"
			user => "moduleAdmin"
		   	   password => "zjdsj@401"
		}
	}
	
	if [type] == "big_oper_log" {
		elasticsearch {
			hosts => ["192.168.115.30:9200"]
			index => "big_oper_log-%{est_time}" # big_oper_log-%{+YYYY.MM.dd}
			action => "index"
			document_id => "%{[@metadata][oper_id]}"
			user => "moduleAdmin"
		        password => "zjdsj@401"
		}
	}
}
```

![image-20230328101338441](https://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202303281013499.png)

#### 5.3.2 回滚定时任务

> 1. 直接删除`索引+日期`就好了 。【删除索引】
> 2. 使用[`Delete By Query API`](http://www.manongjc.com/detail/58-oyzlqqqmmktxmif.html)删除数据。【删除文档】[[通过查询 API 删除](https://www.elastic.co/guide/en/elasticsearch/reference/8.6/docs-delete-by-query.html) 

```java
GET sys_config/_search
{"query":{"range":{"oper_time":{"gte":"2023-01-01","lte":"2023-01-21"}}}}

POST sys_config/_delete_by_query
{"query":{"range":{"oper_time":{"gte":"2023-01-01"}}}}

POST /sys_config/_forcemerge?only_expunge_deletes=true
```



```shell
5 8 * * * sh /weblogic/elk/es-index-clear.sh
#!/bin/bash

DATA=`date -d "1 year ago" +%Y.%m.%d`
# 当前日期
time=`date +%F`
# 删除一年前的日志
Res=$(curl -X DELETE 'http://192.168.115.30:9200/big_oper_log-'${DATA}'' \
  -H 'Authorization: Basic bW9kdWxlQWRtaW46empkc2pANDAx' \
  -H 'Content-Type: application/json')

if [ $? -eq 0 ]; then
	echo "$time--> DELETE $DATA log success.. ["$Res"]" >> /data/elk/shell/clearLogs/es-index-clear.log	  
else
	echo "$time--> DELETE $DATA log fail.. ["$Res"]" >> /data/elk/shell/clearLogs/es-index-clear.log	
fi
echo -e '\n' >> /data/elk/shell/clearLogs/es-index-clear.log
```

![滚动更新：针对索引+日期的数据](https://ahaolin-public-img.oss-cn-hangzhou.aliyuncs.com/img/202303141550585.webp)

```sh
## 滚动更新：针对单一索引的数据

#!/bin/bash

DATA=`date -d "1 year ago" +%Y-%m-%d`
# 当前日期
time=`date +%F`
# 指定需要删除的 索引名称 
indexName="sys_config,big_oper_log-2023.03.31" # 使用,分隔
# 删除一年前的日志
Res=$(curl -X POST 'http://192.168.115.30:9200/sys_config/_delete_by_query' \
  -H 'Authorization: Basic bW9kdWxlQWRtaW46empkc2pANDAx' \
  -H 'Content-Type: application/json' \
  --data '{"query":{"range":{"oper_time":{"gte":"'$DATA'"}}}}')

if [ $? -eq 0 ]; then
	echo "["$indexName"]$time--> del $DATA log success.. ["$Res"]" >> /data/elk/shell/clearLogs/es-index-clear.log
        # 执行curl 进行命令行的合并
	Res=$(curl -X POST  'http://192.168.115.30:9200/'$indexName'/_forcemerge?only_expunge_deletes=true' \
          -H 'Authorization: Basic bW9kdWxlQWRtaW46empkc2pANDAx' \
          -H 'Content-Type: application/json; charset=UTF-8') 	
	echo "["$indexName"]$time--> merge $DATA log success.. ["$Res"]" >> /data/elk/shell/clearLogs/es-index-clear.log		  
else
	echo "["$indexName"]$time--> del $DATA log fail.. ["$Res"]" >> /data/elk/shell/clearLogs/es-index-clear.log	
fi
echo -e '\n' >> /data/elk/shell/clearLogs/es-index-clear.log # 换行

#http://172.16.96.*:9200/index/_delete_by_query?slices=3&wait_for_completion=false&scroll_size=5000&conflicts=proceed
```

> - `slices`：线程数（根据CPU的数量设置）
>   `wait_for_completion`:如果设置为true，则导致 API 阻塞，直到索引器状态完全停止。如果设置为false，API 立即返回，并且索引器在后台异步停止。默认为 false。如果请求包含wait_for_completion=false，则 Elasticsearch 将执行一些预检检查，启动请求，然后返回一个task 可用于任务 API 以取消或获取任务状态的内容。Elasticsearch 还将创建此任务的记录作为文档，位于.tasks/task/${taskId}. 这是您认为合适的保留或删除。完成后，将其删除，以便 Elasticsearch 可以回收它使用的空间。
>
> - `scroll_size`：游标查询，根据index.max_result_window值设置，scroll_size应当小于index.max_result_window值，默认是10000
>
> - `conflicts`：在_delete_by_query执行过程中，依次执行多个搜索请求，以便找到所有匹配的文档进行删除。每找到一批文档，就会执行相应的批量请求，删除所有这些文档。如果搜索或批量请求被拒绝，_delete_by_query 则依靠默认策略重试被拒绝的请求（最多 10 次，指数回退）。达到最大重试限制会导致_delete_by_query 中止，并且所有失败都在failures响应中返回。已执行的删除仍然存在。换句话说，该过程没有回滚，只是中止。当第一次失败导致中止时，失败的批量请求返回的所有失败都在failures 元素; 因此，可能会有相当多的失败实体。如果您想计算版本冲突而不是导致它们中止，请conflicts=proceed在 url 或"conflicts": "proceed"请求正文中设置。

#### 5.3.3 日志滚动更新

创建脚本 /data/elk/auto-del.sh

```sh
find /data/elk/elasticsearch-8.5.3/logs/  -mtime +30 -name "*.gz" -exec rm -rf {} \;
find /data/elk/logstash-8.5.3/logs/  -mtime +30 -name "*.gz" -exec rm -rf {} \;
# 脚本记得赋予权限

# 使用crontab 定时任务执行定时删除
[root@localhost elk]# crontab -e
no crontab for root - using an empty one
crontab: installing new crontab
[root@localhost elk]# crontab -l
00 23 * * * /data/elk/auto-del.sh >/dev/null 2>&1
```

### 5.4 额外链接

[集群身份认证与用户鉴权，集群内部安全通信，集群与外部间的安全通信 - 博客园](https://www.cnblogs.com/hahaha111122222/p/12046640.html)

[ElasticSearch结合LDAP实现权限、用户管控 - 简书](https://www.jianshu.com/p/7154e80490ad)

[【elastic系列】4、es集群认证及传输加密_海军同学](https://blog.51cto.com/root/3169828)

[ES LDAP 集成 - 简书](http://events.jianshu.io/p/5b10475c605a)

[CentOS7安装elasticsearch-8.5.3集群、kibana-8.5.3_义明的博客](https://blog.csdn.net/ZYMruanjianjishu/article/details/128700599)

- [windows部署elk8.5.3记录 - ENU - 博客园](https://www.cnblogs.com/ENU7/p/16995033.html)
- [Linux部署elk 8.5.3 教程 - ENU - 博客园 ](https://www.cnblogs.com/ENU7/p/17003589.html)

> [ElasticSearch实战系列](https://www.cnblogs.com/xuwujing/tag/elasticsearch/):
>
> - [ElasticSearch实战系列一: ElasticSearch集群+Kinaba安装教程](https://www.cnblogs.com/xuwujing/p/11385255.html)
> - [ElasticSearch实战系列二: ElasticSearch的DSL语句使用教程---图文详解](https://www.cnblogs.com/xuwujing/p/11567053.html)
> - [ElasticSearch实战系列三: ElasticSearch的JAVA API使用教程](https://www.cnblogs.com/xuwujing/p/11645630.html)
> - [ElasticSearch实战系列四: ElasticSearch理论知识介绍](https://www.cnblogs.com/xuwujing/p/12093933.html)
> - [ElasticSearch实战系列四: ElasticSearch理论知识介绍](https://www.cnblogs.com/xuwujing/p/12093933.html)
> - [ElasticSearch实战系列五: ElasticSearch的聚合查询基础使用教程之度量(Metric)聚合](https://www.cnblogs.com/xuwujing/p/12385903.html)
> - [ElasticSearch实战系列六: Logstash快速入门](https://www.cnblogs.com/xuwujing/p/13412108.html)
> - [ElasticSearch实战系列七: Logstash实战使用-图文讲解](https://www.cnblogs.com/xuwujing/p/13520666.html)

[ELK入门——elk索引清空如何加载数据_Netceor的博客-CSDN博客](https://blog.csdn.net/Netceor/article/details/112992149)
