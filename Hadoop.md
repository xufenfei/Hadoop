http://www.cloudera.com/documentation/enterprise/latest/topics/cdh_ig_cdh5_cluster_deploy.html     // 

https://www.ibm.com/developerworks/cn/opensource/os-cn-hadoop-name-node/        //NN主备切换，经典 

## 1, HDFS

namenode只能有两个，一个Active,一个Standby, 只有active NN才能对外提供读写服务。
主备切换控制器ZKFailoverController：作为独立的进程运行，对NN的主备切换进行总体控制。
ZKFailoverController能及时检测到NN的健康状况，在主NN故障时借助Zookeeper实现自动主备选举和切换，但也支持不依赖于zookeeper的手动主备切换。
Zookeeper集群：为主备切换控制器提供主备选举支持。
共享存储系统：是实现NN高可用最为关键的部分，共享存储系统保存了NN在运行过程中所产生的HDFS的元数据，主NN与备NN通过共享存储实现元数据同步。在进行主备切换时，新的主NN在确认元数据完全同步之后才能继续对外提供服务。

除了通过共享存储系统共享HDFS的元数据信息之外，主NN和备NN还需要共享HDFS的数据块和DataNode之间的映射关系，DN会同时向主NN和备NN上报数据块的位置信息。

NN主备切换主要由ZKFC, HealthMonitor, ActiveStandbyElector这3个组件来协同实现。
ZKFC启动时会创建HealthMonitor和ActiveStandbyElector这两个主要内部组件。
HealthMonitor:主要负责检测nn的健康状态，如果检测到nn的状态发生变化，会回调ZKFC的相应方法进行自动主备选举。
ActiveStandbyElector: 主要负责完成自动的主备选举，内部封装了zookeeper的处理逻辑，一是zookeeper主备选举完成，会回调ZKFC的相应方法来进行NameNode的主备状态切换。

NN主备切换流程：
HealthMonitor 初始化完成之后会启动内部的线程来定时调用对应 NameNode 的 HAServiceProtocol RPC 接口的方法，对 NameNode 的健康状态进行检测。
HealthMonitor 如果检测到 NameNode 的健康状态发生变化，会回调 ZKFailoverController 注册的相应方法进行处理。
如果 ZKFailoverController 判断需要进行主备切换，会首先使用 ActiveStandbyElector 来进行自动的主备选举。
ActiveStandbyElector 与 Zookeeper 进行交互完成自动的主备选举。
ActiveStandbyElector 在主备选举完成后，会回调 ZKFailoverController 的相应方法来通知当前的 NameNode 成为主 NameNode 或备 NameNode。
ZKFailoverController 调用对应 NameNode 的 HAServiceProtocol RPC 接口的方法将 NameNode 转换为 Active 状态或 Standby 状态。

 


HealthMonitor 主要检测NN的两类状态，分别是HealthMonitor.State和HAServiceStatus，

HealthMonitor.State 是通过 HAServiceProtocol RPC 接口的 monitorHealth 方法来获取的，反映了 NameNode 节点的健康状况，主要是磁盘存储资源是否充足，有如下状态：
NITIALIZING：HealthMonitor 在初始化过程中，还没有开始进行健康状况检测；
SERVICE_HEALTHY：NameNode 状态正常；
SERVICE_NOT_RESPONDING：调用 NameNode 的 monitorHealth 方法调用无响应或响应超时；
SERVICE_UNHEALTHY：NameNode 还在运行，但是 monitorHealth 方法返回状态不正常，磁盘存储资源不足；
HEALTH_MONITOR_FAILED：HealthMonitor 自己在运行过程中发生了异常，不能继续检测 NameNode 的健康状况，会导致 ZKFailoverController 进程退出；


 HAServiceStatus 则是通过 HAServiceProtocol RPC 接口的 getServiceStatus 方法来获取的，主要反映的是 NameNode 的 HA 状态，包括：
INITIALIZING：NameNode 在初始化过程中；
ACTIVE：当前 NameNode 为主 NameNode；
STANDBY：当前 NameNode 为备 NameNode；
STOPPING：当前 NameNode 已停止；
HAServiceStatus 在状态检测之中只是起辅助的作用，在 HAServiceStatus 发生变化时，HealthMonitor 也会回调 ZKFailoverController 的相应方法来进行处理，





  
自动故障切换是通过Zookeeper quorum与ZKFC来实现在的。

Failure detection: 集群中的每个NN机器都在ZooKeeper里维护着持久的session，当机器crashes,ZooKeeper session就会到期，从而通知另外的NN，此时切换可以被触发。
Active NN election: Zookeepers提供一种简单的机制来唯一选择一个active, 如果当前的活动NN挂了，另外 一个NN将启用排他锁，表明自己将成为下一个Active NN.

ZKFailoverController : ZKFC , 它是一个zookeeper客户端，它同时监视/管理NN的状态，运行NN的主机同时也运行着ZKFC，ZKFC可以做如下事情：
1，Health monitoring:  通过health-check来检查NN的状态。
2， Zookeeper session management:  当NN状态为healthy时，ZKFC在Zookeeper里保持着open的状态，如果NN是acitve时，它会同时把持着一个特殊的锁，znode, 如果SESSION到期，锁节点会自己被删除。
3， Zookeeper-base election:    

配置HDFS HA 

1， NameNode hosts:    ANN与SNN物理配置要一样。
2， JournalNode hosts:   日志主机，就是运行JournalNode的主机，JournalNode进程可以运行在NN,SNN,JobTracker上，这样可以确保本地存储的可靠性。
3， 如果共存在一台主机上，每个JournalNode进程和每个NN进程都需要有自己的独享盘，你不能使用SAN/NAS作用它们的目录。
4， 必须有至3个JournalNode进程，应该有奇数个JournalNodes,(3,5,,7等），如果运行N个JouralNode,则可以允许容忍（N-1）/2个失败，如果必须的quorum不可用了，NN就不会启动，于是，你就会收到ERROR消息。


通过Cloudera Manager来启用HDFS HAs

在CM5里，HA是通过使用基于quorum的存储来实现的，基于quorum的存储依赖于一系列JounalNode，每个journalnode维护一个本地修改目录 ，里面记录着修改名字空间的源数据。

-----------------------------------------------------------------------
 

自动故障转移配置：

新增组件：zookeeper仲裁  ZKFailoverController进程（缩写ZKFC）
http://www.cloudera.com/content/www/zh-CN/documentation/enterprise/5-3-x/topics/cdh_hag_hdfs_ha_enabling.html#topic_2_4_3_unique_3


## 2, Zookeeper

zookeeper分布式服务框架，它主要用来解决分布式应用经常遇到的一些数据管理的问题，如：统一命名服务，状态同步服务，分布式应用配置项的管理等。

名称服务：将一个名称映射到与该名称有关联的一些信息的服务。
锁定：为了允许在分布式系统中对共享资源进行有序的访问，可能需要实现分布式互斥。
同步：
配置管理：
领导者选举：


 

在 ZooKeeper 集合体中，三、五或七是最典型的节点数量。请记住，ZooKeeper 集合体的大小与分布式系统中的节点大小没有什么关系。


单机模式：
tickTime：zookeeper服务器之间或客户端与服务器之间维持心跳的时间间隔。也就是每个tickTime时间就会发送一个心跳。
dataDir：数据目录，默认清况下，写数据的日志文件也保存在这个目录中
clientPort：客户端连接zookeeper服务器的端口，zookeeper会监听这个端口，接受客户端的访问请求。
netstat -ano命令查看端口。

集群模式：
initLimit=5
syncLimit=2
server.1=192.168.211.1:2888:3888
server.2=192.168.211.2:2888:3888

initLimit：这个配置项是用来配置zookeeper接受客户端初始化连接时最长能忍受多少个心跳时间间隔数，这里的客户端不是用户连接zookeeper服务器的客户端，而是zookeeper服务器集群中连接到Leader的follower服务器）， 5 * 2000 = 10s,也就是说10s内没有收到客户端的返回信息，就表明客户端连接失败。

syncLimit：标识leader和follower之间发送消息，请求和应答时间长度，最长不能超过多少个tickTime的时间长度，总的时间长度就是2*2000=4s.

server.A=B:C:D   A：表示第几号服务器。
                             B：本服务器的IP地址。
                             C：本服务器与集群中的leader服务器交换信息的端口。
                             D：万一集群的leader挂了，需要一个端口来重新进行选举，选出一个新的leader,而这个端口就是用来执行选举时服务器相互通信的端口。


数据模型：

* 每个子目录如NameService都被称为znode，这个znode是被它所在的路径唯一标识，如Server1这个znode的标识为：/NameService/Server1.

* znode可以有子节点目录，并且每个znode可以存储数据，ephemeral类型的目录节点不能有子节点目录。

* znode是有版本的，每个znode中存储的数据可以有多个版本，也就是一个访问路径中可以存储多份数据。

*  znode可以是临时节点，一量创建这个znode的客户端与服务器失去联系，这个znode就会自动删除，客户端与服务端采用长连接方式，能过心跳来保持连接，此连接状态称为session，如果znode是临时节点，当session失效 ，znode也就删除了。

*  znode的目录名可以自动编号。

*  znode可以被监控，包括这个目录节点中的存储数据的修改，子节点目录的变化等，一量变化可以通知设置监控的客户端，这zookeeper的核心特性，zookeeper的很多功能都是基于这个特性实现的。

Zookeeper集群最少有需要有3个zookeeper server，分开部署，zookeeper server进程应具有其专用磁盘存储。


 https://www.ibm.com/developerworks/cn/opensource/os-cn-zookeeper/  分布式服务框架zookeeper.

如果zookeeper server遇到无法恢复的错误，则它将 “ 快速失败” ，立即退出进程。


http://www.ibm.com/developerworks/cn/data/library/bd-zookeeper/
ZooKeeper 的应用程序
由于 ZooKeeper 在分布式系统中提供了一些多功能的用例，ZooKeeper 有一组不同的实用应用程序。我们将在这里列出部分这些应用程序。这些应用程序大多取自 Apache ZooKeeper 维基，那里还提供了一个更完整的最新列表。请参阅 参考资料，获得这些技术的链接：
Apache Hadoop 依靠 ZooKeeper 来实现 Hadoop HDFS NameNode 的自动故障转移，以及 YARN ResourceManager 的高可用性。
Apache HBase 是构建于 Hadoop 之上的分布式数据库，它使用 ZooKeeper 来实现区域服务器的主选举（master election）、租赁管理以及区域服务器之间的其他通信。
Apache Accumulo 是构建于 Apache ZooKeeper（和 Apache Hadoop）之上的另一个排序分布式键/值存储。
Apache Solr 使用 ZooKeeper 实现领导者选举和集中式配置。
Apache Mesos 是一个集群管理器，提供了分布式应用程序之间高效的资源隔离和共享。Mesos 使用 ZooKeeper 实现了容错的、复制的主选举。
Neo4j 是一个分布式图形数据库，它使用 ZooKeeper 写入主选择和读取从协调（read slave coordination）。
Cloudera Search 使用 ZooKeeper（通过 Apache Solr）集成了搜索功能与 Apache Hadoop，以实现集中式配置管理。






## 3, Hive 
            
Hive：数据仓库应用，你可以使用Hive QL访问你的数据

HiveServer2：Hive Server的改进版，它支持JDBC/ODBC Thrift API应用， 

sudo service hive-server2 start/stop

beeline： CLI工具

/usr/lib/hive/bin/beeline

beeline>!connect jdbc:hive2://localhost:10000

show tables;

## 4, impala

http://www.cloudera.com/content/www/zh-CN/documentation/enterprise/5-3-x/topics/impala_proxy.html    
//HA与代理配置 

impala statestore机制不包括代理与负载平衡功能。

impala:分布式，大规模并行处理MPP数据库引擎，由不同的daemon进程组成。

impalad daemon: 核心组件，运行在每个节点上，它从/向数据文件读取/写入，接受impala-shell指令，JDBC查询指令，分配工作到impala集群中的其它节点，将中间查询结果传回中央协调器节点。

对于运行生产工作负载的群集，可以通过以循环调度样式，使用JDBC/ODBC接口提交每个查询到不同的impala daemon,以在节点之间平衡工作负载。impala daemon通过与statestore通信，以确认哪些节点正常运行并可接受新工作。

-  impalad statestored: 检查集群众中所有节点的impala daemon运行情况，并持续转发其获取结果到各个daemon，仅需要在一个节点上运行些进程。
     statestore目的是在出现问题时提供帮助，它不是集群至关重要的部分，如果statestored未运行，或无法访问，其它节点可以继续运行并在节点之间正常分配工作；如果statestore离线，则集群会越来越不坚固，当statestore重新联机后，会重新建立与其它节点间的通信并恢复其监控功能。

-  impalad catalogd：称为目录服务，将impala sql语句的元数据更改传到集群中的所有节点，只需在一个节点上使用此进程，可在同一个节点上运行staestored和catalogd服务。


imapla的一个主要目标是使用hadoop上的SQL运行的更快更高效，impala可以访问由HIVE定义或加载的表，只要所有列使用impala支持的数据类型、文件格式和压缩编解码器。

查询优化程序： compute stats

安装和配置hive metastore是impala的要求，如果没有metastore数据库，impala将无法工作

1.安装MYSQL

2.下载mysql连接插件，并将其放在/usr/share/java/目录中

3.为数据库使用合适的命令行工具以创建metastore数据库。

4.为数据库使用合适的命令行工具向hive用户授予metastore数据库的权限。

5.修改hive-site.xml以包含与特定数据库匹配的信息。



DDL语句：create, drop, alter 

DML语句：
                insert：
                load data：将现有数据文件复制到impala表的目录中，使用其立即供impala使用，


update子句在impala中是没有的。

impala-shell配置选项：
-o filename 
-f:
-r:


load balance for impala
监听端口
21000   应用连接端口
21050   impala-shell连接端口

yum install haproxy       (可以直接使用，在openldap.succez.com上测试过）
cat /etc/haproxy/haproxy.cfg
/usr/sbin/haproxy -f /etc/haproxy/haproxy.cfg

25003 impala-shell连接此端口。
25002  HA管理界面


配置HA

目的： 为impala jdbc提供统一的接口，作用参照http://www.cloudera.com/content/cloudera/en/documentation/core/latest/topics/impala_proxy.html 

步骤： 

安装haproxy

选择一台非impalad的机器安装haproxy 
<code>

yum install haproxy 

</code>

编辑/etc/haproxy/haproxy.cfg，参考 

<code>

global 
    # To have these messages end up in /var/log/haproxy.log you will 
    # need to: 
    # 
    # 1) configure syslog to accept network log events. This is done 
    # by adding the '-r' option to the SYSLOGD_OPTIONS in 
    # /etc/sysconfig/syslog 
    # 
    # 2) configure local2 events to go to the /var/log/haproxy.log 
    # file. A line like the following can be added to 
    # /etc/sysconfig/syslog 
    # 
    # local2.* /var/log/haproxy.log 
    # 
    log 127.0.0.1 local0 
    log 127.0.0.1 local1 notice 
    chroot /var/lib/haproxy 
    pidfile /var/run/haproxy.pid 
    maxconn 4000 
    user haproxy 
    group haproxy 
    daemon 

    # turn on stats unix socket 
    #stats socket /var/lib/haproxy/stats 

#--------------------------------------------------------------------- 
# common defaults that all the 'listen' and 'backend' sections will 
# use if not designated in their block 
# 
# You might need to adjust timing values to prevent timeouts. 
#--------------------------------------------------------------------- 
defaults 
    mode http 
    log global 
    option httplog 
    option dontlognull 
    option http-server-close 
    #option forwardfor except 127.0.0.0/8 
    option redispatch 
    retries 3 
    maxconn 3000 
    timeout connect 5000 
    timeout client 50000 
    timeout server 50000 

# 
# This sets up the admin page for HA Proxy at port 25002. 
# 
listen stats :25002 
    balance 
    mode http 
    stats enable 
    stats auth username:password 

# This is the setup for Impala. Impala client connect to load_balancer_host:25003. 
# HAProxy will balance connections among the list of servers listed below. 
# The list of Impalad is listening at port 21000 for beeswax (impala-shell) or original ODBC driver. 
# For JDBC or ODBC version 2.x driver, use port 21050 instead of 21000. 
listen impala :25003 
    mode tcp 
    option tcplog 
    balance leastconn 
    server cdhslave1 cdhslave1.yeahmobi.com:21050 check 
    server cdhslave2 cdhslave2.yeahmobi.com:21050 check 
    server cdhslave3 cdhslave3.yeahmobi.com:21050 check 

启动haproxy 
haproxy -f /etc/haproxy/haproxy.cfg 
关闭service haproxy stop 

</code>


impala配置 
impala daemon group->advanced->Impala Daemon Command Line Argument Advanced Configuration Snippet (Safety Valve) 

-principal=impala/cdhmaster.yeahmobi.com@YEAHMOBI.COM 

-be_principal=impala/_HOST@YEAHMOBI.COM 

测试 
private static final String IMPALAD_HOST = "cdhmaster.yeahmobi.com";//haproxy server的hostname 

private static final String IMPALAD_JDBC_PORT = "25003";//端口选择haproxy的代理端口 

private static final String CONNECTION_URL = "jdbc:hive2://" + IMPALAD_HOST + ':' + IMPALAD_JDBC_PORT + "/data_system;user=hive;password=111111";//ldap用户及密码 


使用时发现，impala-shell不好使了，不能正常连接，去除如上粗体步骤后，impala-shell正常，jdbc HAproxy也正常使用。


来源： http://blog.csdn.net/lookqlp/article/details/52096329











