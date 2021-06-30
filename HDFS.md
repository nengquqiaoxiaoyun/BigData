# 1. 概述

## 1.1 定义

**HDFS（Hadoop Distributed File System）**是一个分布式文件系统，适用于一次写入多次读取的场景

## 1.2 组成

![image-20210630153708563](assets/image-20210630153708563.png)

HDFS的关键组件有NameNode和DataNode

### 1.2.1 NameNode

**NameNode负责整个分布式文件系统的元数据（MetaData）管理，也就是文件路径名、数据块的ID以及存储位置等信息**

HDFS将一个数据块复制为多份（默认3份），并将多份相同数据存储在不同的服务器甚至机架上来保持数据的高可用。当有磁盘损坏或者DataNode服务宕机时，客户端会查找备份数据进行访问

NameNode是整个HDFS的核心，记录着HDFS文件分配表信息，所有的文件路径和数据块存储信息都保存在NameNode，如果NameNode故障，整个HDFS系统集群都无法使用；如果NameNode上记录的数据丢失，整个集群所有DataNode存储的数据也就没用了

因此NameNode高可用容错能力非常重要。NameNode采用主从热备的方式提供高可用服务，如下图

![image-20210630160033527](assets/image-20210630160033527.png)

集群部署两台NameNode服务器，一台作为主服务器提供服务，一台作为从服务器进行热备，两台服务器通过ZooKeeper选举，主要是通过争夺znode锁资源，决定谁是主服务器。而DataNode则会向两个NameNode同时发送心跳数据，但是只有主NameNode才能向DataNode返回控制信息

正常运行期间，主从NameNode之间通过一个共享存储系统shared edits来同步文件系统的元数据信息。当主NameNode服务器宕机，从NameNode会通过ZooKeeper升级成为主服务器，并保证HDFS集群的元数据信息，也就是文件分配表信息完整一致

### 1.2.2 DataNode

**DataNode负责文件数据的存储和读写操作，HDFS将文件数据分割成若干数据块（Block），每个DataNode存储一部分数据块，这样文件就分布存储在整个HDFS服务器集群中**

### 1.2.3 Client

- Client在文件上传至HDFS会将文件切分成多个BLock然后进行上传
- 与NameNode交互获取文件位置信息
- 与DataNode交互进行读写
- 通过命令访问或管理HDFS，比如对HDFS的CRUD、NameNode格式化

### 1.2.4 Secondary NameNode

- 辅助NameNode分担其工作量
- 在紧急情况下可辅助恢复部分NameNode

## 1.3 HDFS文件块大小

HDFS默认块大小为128M（Hadoop2.X/3.X，Hadoop1.X为64M）

**HDFS块大小的设置主要取决于磁盘传输速率（100MB/s时考虑128M，200-300MB/s考虑256M）**

若HDFS块设置过小会导致块的增多，从而增加了寻址时间，相反，块设置太大会导致磁盘传输数据的时间增长，处理数据过慢