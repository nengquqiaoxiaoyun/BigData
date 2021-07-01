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

## 1.4 总结

- 文件数据以数据块的方式进行切分，数据块可以存储在集群任意DataNode服务器上，所以HDFS存储的文件可以非常大，一个文件理论上可以占据整个HDFS服务器集群上的所有磁盘，实现了大容量存储
- HDFS一般的访问模式是通过MapReduce程序在计算时读取，MapReduce对输入数据进行分片读取，通常一个分片就是一个数据块，每个数据块分配一个计算进程，这样就可以同时启动很多进程对一个HDFS文件的多个数据块进行并发访问，从而实现数据的高速访问
- DataNode存储的数据块会进行复制，使每个数据块在集群里有多个备份，保证了数据的可靠性，并通过一系列的故障容错手段实现HDFS系统中主要组件的高可用，进而保证数据和整个系统的高可用

# 2. HDFS的基本操作

## 2.1 基本语法

```shell
hadoop fs ...
hdfs dfs ...
# 两个命令完全相同
```

## 2.2 常用命令

###  -help

查看命令参数

```shell
 hadoop fs -help rm
 
 -rm [-f] [-r|-R] [-skipTrash] [-safely] <src> ... :
  Delete all files that match the specified file pattern. Equivalent to the Unix
  command "rm <src>"
                                                                                 
  -f          If the file does not exist, do not display a diagnostic message or 
              modify the exit status to reflect an error.                        
  -[rR]       Recursively deletes directories.                                   
  -skipTrash  option bypasses trash, if enabled, and immediately deletes <src>.  
  -safely     option requires safety confirmation, if enabled, requires          
              confirmation before deleting large directory with more than        
              <hadoop.shell.delete.limit.num.files> files. Delay is expected when
              walking over large directory recursively to count the number of    
              files to be deleted before the confirmation.  
```

###  -mkdir

创建文件夹

```shell
 hadoop fs -mkdir /sanguo
```

###  -moveFromLocal

将本地文件剪切到HDFS

```shell
# create file and input
vim shuguo.txt
input: shuguo

hadoop fs -moveFromLocal ./shuguo.txt /sanguo
```

###  -copyFromLocal/-put

将本地文件复制到HDFS

```shell
# create file and input
vim weiguo.txt
input: weiguo

hadoop fs -copyFromLocal/-put ./weiguo.txt /sanguo
```

### -appendToFile

追加文件到已有文件末尾 

```shell
 vim liubei.txt
 input: liubei
 
 hadoop fs -appendToFile ./liubei.txt /sanguo/shuguo.txt
```

### -copyToLocal/-get 

从HDFS拷贝到本地

```shell
hadoop fs -copyToLocal /sanguo/shuguo.txt ./

# 可更改名字
hadoop fs -get /sanguo/shuguo.txt ./shuguo2.txt
```

### -ls

显示目录信息

```shell
hadoop fs -ls /sanguo
```

### -cat 

显示文件内容

```shell
 hadoop fs -cat /sanguo/shuguo.txt
```

### -chgrp、-chmod、-chown

修改文件属性

```shell
hadoop fs -chgrp hadoop /sanguo/shuguo.txt

hadoop fs -chmod 666 /sanguo/shuguo.txt

hadoop fs -chown atguigu:atguigu /sanguo/shuguo.txt
```

### -cp

从 HDFS 的一个路径拷贝到 HDFS 的另一个路径

```shell
 hadoop fs -cp /sanguo/shuguo.txt /jinguo
```

### -mv 

在HDFS目录中移动文件

```shell
hadoop fs -mv /sanguo/wuguo.txt /jinguo
hadoop fs -mv /sanguo/weiguo.txt /jinguo
```

### -tail

显示文件末尾1kb的数据

```shell
hadoop fs -tail /jinguo/shuguo.txt
```

### -rm

删除文件或文件夹

```shell
hadoop fs -rm /sanguo/shuguo.txt
```

#### -rm -r

递归删除目录及目录中的内容

```shell
 hadoop fs -rm -r /sanguo
```

### -du 

统计文件大小信息

```shell
hadoop fs -du -s -h /jinguo
27 81 /jinguo

hadoop fs -du -h /jinguo
14 42 /jinguo/shuguo.txt
7 21 /jinguo/weiguo.txt
6 18 /jinguo/wuguo.tx
```

### -setrep

设置HDFS中文件的副本数量

Note：副本超出当前机器时，假设只有三台机器，那么只能有三个副本，后续增加机器后会再创建副本。**即节点数足够时才能使设置的副本数量满足需求**

```shell
hadoop fs -setrep 10 /jinguo/shuguo.txt
```

