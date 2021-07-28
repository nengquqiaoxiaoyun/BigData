# 1 概念

[HBase官网](https://hbase.apache.org/)

> [Apache](https://www.apache.org/) HBase™ is the [Hadoop](https://hadoop.apache.org/) database, a distributed, scalable, big data store.
>
> Use Apache HBase™ when you need random, realtime read/write access to your Big Data. This project's goal is the hosting of very large tables -- billions of rows X millions of columns -- atop clusters of commodity hardware. Apache HBase is an open-source, distributed, versioned, non-relational database modeled after Google's [Bigtable: A Distributed Storage System for Structured Data](https://research.google.com/archive/bigtable.html) by Chang et al. Just as Bigtable leverages the distributed data storage provided by the Google File System, Apache HBase provides Bigtable-like capabilities on top of Hadoop and HDFS.

从Google的BigTable开始，一系列的可以进行海量数据存储与访问的数据库被设计出来，更进一步说，NoSQL这一概念被提了出来

NoSQL主要指**非关系的、分布式的、支持海量数据存储的数据库设计模式**。也有许多专家将 NoSQL解读为Not Only SQL，表示NoSQL只是关系数据库的补充，而不是替代方案。其中，HBase是这一类NoSQL系统的杰出代表

## 1.1 数据模型

![image-20210727151817391](assets/image-20210727151817391.png)

### NameSpace

命名空间，类似于关系型数据库的DataBase概念，每个命名空间下有多个表。HBase有两个自带的命名空间：`hbase`和`defalut`，`hbase`中存放HBase自带的表，`default`是用户默认使用的命名空间

### Cell

表由行和列组成，表格的单元格（Cell）由行和列的坐标交叉决定，是有版本的。默认情况下，版本号是自动分配的，为HBase插入单元格时的时间戳。单元格中的数据是没有类型的，其内容是字节数组

### Column Family

行中的列被分成列族（column family），同一个列族的所有成员具有相同的前缀。如上图`info:format`和`info:geo`都是列族`info`的成员

一个表的列族必须作为表模式定义的一部分预先给出，而列限定符（ column qualifier，可以看作是单列）无需预先给定，但是新的列族成员（列限定符，也就是单列）可以随后按需加入

物理上，所有的列族成员都一起存放在文件系统中。所以，HBase被描述为一个面向列的存储器，实际上更准确的说法是：**它是一个面向列族的存储器**

![image-20210727153249993](assets/image-20210727153249993.png)

### Region

HBase自动把表水平划分成区域（Region），每个区域都由表中行的子集构成

一开始一个表只有一个区域，随着区域开始变大（行数增多想象为高表）等到它超出设定的大小阈值，便会在某行的边界上把表分成两个大小基本相同的新分区

第一次划分之前，所有加载的数据都放在原始区域所在的服务器上，随着表的增大，区域个数也会增加。**区域是在HBase集群上分布数据的最小单位**。使用这种方式，一个因为太大而无法放在单台服务器上的表会被放到服务器集群上，其中每个节点都负责管理表所有区域的一个子集

![image-20210727153415168](assets/image-20210727153415168.png)

## 1.2 实现

正如HDFS和YARN是有客户端、从属机（slave）和协调主控机（master）组成，HBase也采用相同的模型，它用一个`master`节点协调管理一个或多个`regionserver`从属机

![image-20210727154322480](assets/image-20210727154322480.png)

![image-20210727154756818](assets/image-20210727154756818.png)

# 2 部署

HBase依赖于ZooKeeper和Hadoop，安装HBase之前先将ZooKeeper和Hadoop部署并启动

将HBase解压至指定目录

```shell
tar -zxvf hbase-1.3.1-bin.tar.gz -C /opt/module
```

修改配置文件(这边使用$hbase表示hbase主目录)

`$hbase/conf/hbase-env.sh`

```shell
# $JAVA_HOME根据自身情况替换为实际路径
export JAVA_HOME=$JAVA_HOME
export HBASE_MANAGES_ZK=false
```

注释掉如下两行（仅JDK7需要用到）

![image-20210728090510213](assets/image-20210728090510213.png)

`$hbase/conf/hbase-site.xml`**根据实际情况修改！**

```xml
<configuration>
  <!-- 对应hdfs -->
<property>
	<name>hbase.rootdir</name>
	<value>hdfs://hadoop102:9000/HBase</value>
</property>
<property>
	<name>hbase.cluster.distributed</name>
	<value>true</value>
</property>
<!-- 0.98 后的新变动，之前版本没有.port,默认端口为 60000 -->
<property>
	<name>hbase.master.port</name>
	<value>16000</value>
</property>
<property> 
	<name>hbase.zookeeper.quorum</name>
	<value>hadoop102, hadoop103, hadoop104</value>
</property>
  <!-- 指定zk的dataDir 要和zk对应 -->
<property> 
	<name>hbase.zookeeper.property.dataDir</name>
	<value>/opt/module/zookeeper-3.6.3/zkData</value>
</property>
</configuration>
```

`$hbase/conf/regionservers`添加所有需要的hbase节点，准确来说应该是`regionserver`的节点（类似于Hadoop的wookers）

```
hadoop102
hadoop103
hadoop104
```

将Hadoop配置[软件接](https://www.cnblogs.com/kex1n/p/5193826.html)到HBase

> 当 我们需要在不同的目录，用到相同的文件时，我们不需要在每一个需要的目录下都放一个必须相同的文件，我们只要在某个固定的目录，放上该文件，然后在其它的 目录下用ln命令链接（link）它就可以，不必重复的占用磁盘空间。例如：ln -s /bin/less /usr/local/bin/less
> -s 是代号（symbolic）的意思。
> 这里有两点要注意：第一，ln命令会保持每一处链接文件的同步性，也就是说，不论你改动了哪一处，其它的文件都会发生相同的变化；第二，ln的链接又软链接 和硬链接两种，软链接就是ln -s ,它只会在你选定的位置上生成一个文件的镜像，不会占用磁盘空间，硬链接ln ,没有参数-s, 它会在你选定的位置上生成一个和源文件大小相同的文件，无论是软链接还是硬链接，文件都保持同步变化

```shell
# core-site 这边我的Hadoop路径为/opt/module/hadoop-3.1.3
# 我的实际操作位 ln -s /opt/module/hadoop-3.1.3/etc/hadoop/core-site.xml /opt/module/hbase-1.3.1/conf/core-site.xm
# 下同
ln -s $HADOOP_HOME/etc/hadoop/core-site.xml $hbase/conf/core-site.xm
# hdfs-site
# ln -s /opt/module/hadoop-3.1.3/etc/hadoop/hdfs-site.xml /opt/module/hbase-1.3.1/conf/hdfs-site.xml
ln -s $HADOOP_HOME/etc/hadoop/hdfs-site.xml $hbase/conf/hdfs-site.xml


[hadoop@hadoop102 conf]$ ln -s /opt/module/hadoop-3.1.3/etc/hadoop/hdfs-site.xml /opt/module/hbase-1.3.1/conf/hdfs-site.xml
[hadoop@hadoop102 conf]$ ll
总用量 40
lrwxrwxrwx. 1 hadoop hadoop   49 7月  28 09:22 core-site.xm -> /opt/module/hadoop-3.1.3/etc/hadoop/core-site.xml
-rw-r--r--. 1 hadoop hadoop 1811 9月  21 2016 hadoop-metrics2-hbase.properties
-rw-r--r--. 1 hadoop hadoop 4537 11月  7 2016 hbase-env.cmd
-rw-r--r--. 1 hadoop hadoop 7513 7月  28 09:06 hbase-env.sh
-rw-r--r--. 1 hadoop hadoop 2257 9月  21 2016 hbase-policy.xml
-rw-r--r--. 1 hadoop hadoop  934 9月  21 2016 hbase-site.xml
lrwxrwxrwx. 1 hadoop hadoop   49 7月  28 09:22 hdfs-site.xml -> /opt/module/hadoop-3.1.3/etc/hadoop/hdfs-site.xml
-rw-r--r--. 1 hadoop hadoop 4722 4月   5 2017 log4j.properties
-rw-r--r--. 1 hadoop hadoop   30 7月  28 09:15 regionservers
```

**配置完成后分发服务到各节点**

## 2.1 启动

