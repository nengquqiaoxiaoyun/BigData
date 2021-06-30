大数据学习笔记，参考极客时间课程

![image-20210616154015957](README.assets/image-20210616154015957.png)

该图的所有这些框架、平台以及相关的算法共同构成了大数据的技术体系

# 官网

[Hadoop](http://hadoop.apache.org/)

# 存储

大数据主要解决海量数据的采集、存储和分析计算问题。如果我们要对数据进行分析计算，首先要解决的其实是海量数据的存储问题。

海量数据存储需要解决几个核心问题：

1. **数据存储容量问题**：如果存储海量数据？
2. **数据读写熟读问题**：一般磁盘连续读写速度为几时MB，对于大规模数据来讲就太慢了
3. **数据可靠性的问题**：磁盘是计算机设备中最易损坏的硬件，如果磁盘损坏，数据怎么办？

## RAID

大数据出现之前对应的解决方案就是RAID（独立磁盘冗余阵列）技术

该技术是将多块普通磁盘组成一个阵列，共同对外提供服务。主要是为了改善磁盘的存储容量、读写速度，增强磁盘的可用性和容错能力。在RAID之前，要使用大容量、高可用、高速访问的存储系统需要专门的存储设备，这类设备价格要比RAID的几块普通磁盘贵几十倍

### 解决方案

来看看RAID是如何解决关于存储的三个关键问题的

1. 数据存储容量的问题：RAID使用了N块磁盘构成一个存储阵列，数据被存储在多个磁盘上，空间得以扩大

2. 数据读写速度的问题：RAID根据可以使用的磁盘数量，将待写入的数据分成多片，并发同时向多块磁盘进行写入，显然写入的速度可以得到明显提高；同理，读取速度也可以得到明显提高
3. 数据可靠性的问题：当某块磁盘损坏的时候，可以通过其他磁盘上的数据和校验数据将丢失磁盘上的数据还原

在计算机领域，实现更强的计算能力和更大规模的数据存储有两种思路，一种是升级计算机，一种是用分布式系统。前一种也被称作“垂直伸缩”（scaling up），通过升级CPU、内存、磁盘等将一台计算机变得更强大；后一种是“水平伸缩”（scaling out），添加更多的计算机到系统中，从而实现更强大的计算能力

在计算机发展的早期，我们获得更强大计算能力的手段主要依靠**垂直伸缩**。垂直伸缩的方式成本巨大，所以互联网公司走向了一条新的道路：**水平伸缩**，在一个系统中不断添加计算机，以满足不断增长的用户和数据对计算资源的需求。这就是最近十几年引导技术潮流的分布式与大数据技术

RAID可以看作是一种垂直伸缩，一台计算机集成更多的磁盘实现数据更大规模、更安全可靠的存储以及更快的访问速度。而HDFS则是水平伸缩，通过添加更多的服务器实现数据更大、更快、更安全存储与访问

RAID技术只是在单台服务器的多块磁盘上组成阵列，大数据需要更大规模的存储空间和更快的访问速度。将RAID思想原理应用到分布式服务器集群上，就形成了Hadoop分布式文件系统HDFS的架构思想

# Hadoop

## 组成

![image-20210621094741314](README.assets/image-20210621094741314.png)

## HDFS（Hadoop Distributed File System）

# Hadoop环境搭建

操作系统：CentOs7

我们先搭建一个空的系统随后克隆出多个镜像即可，克隆出的镜像需要修改ip地址和主机名

```shell
sudo vim /etc/hostname
```

## 查看网络情况

```shell
ifconfig
```

![image-20210630085301461](assets/image-20210630085301461.png)

```shell
cat -n /etc/sysconfig/network-scripts/ifcfg-ens192
```

![image-20210630085441234](assets/image-20210630085441234.png)

## 添加用户和密码

```shell
adduser hadoop
passwd hadoop

# 切换为普通用户
login -f hadoop
```

## 添加用户权限

我们可以给账户配置root的权限，以后加上sudo即可执行root的权限命令

```shell
vim /etc/sudoers
```

```shell
## Allows people in group wheel to run all commands
%wheel ALL=(ALL) ALL
```

在*%wheel* 这行后面添加

```shell
usernmae ALL=(ALL) NOPASSWD:ALL
```

## 修改文件夹所属主和所属组

```shell
sudo chown username:group path
# exp sudo chown wentimei:wentimei /opt/module
```

## 删除自带的jdk

如果系统是非桌面（最小化系统）不需要执行此步骤，因为没有安装

```shell
# root
su
password
# execute
rpm -qa | grep -i java | xargs -n1 rpm -e --nodeps
# 重启
reboot
```

![image-20210621155027207](README.assets/image-20210621155027207.png)

## 配置主机映射

```shell
sudo vim /etc/hosts

# 将对应的ip和主机名写入hosts文件，后续的镜像ip、主机要与该映射保持一致
10.211.55.21 hadoop
10.211.55.18 hadoop2
10.211.55.23 hadoop3
10.211.55.22 hadoop4
```

## 安装JDK

之后我们要安装的软件只需在一台机上安装即可，其他的机子只要拷贝（scp等命令）即可

### 解压至指定目录

```shell
tar -zxvf jdk-8u212-linux-x64.tar.gz -C /opt/module/
```

### 配置环境变量

查看 `/etc/profile`文件，以往我们都是在该文件下进行配置，可以看到该文件的脚本其实就是在`/etc/profile.d/*.sh` 中循环读取依赖，所以我们可以在`profile.d`中加入自己的`.sh`文件

![image-20210621160035054](README.assets/image-20210621160035054.png)

```shell
vi /etc/profile.d/env.sh

#JAVA_HOME
export JAVA_HOME=/opt/module/jdk1.8.0_212
export PATH=$PATH:$JAVA_HOME/bin

#使配置生效
source /etc/profile

java -version
```

## 安装Hadoop

### 解压

```shell
# 将当前文件夹下的指定文件解压至指定目录
tar -zxvf hadoop-3.1.3.tar.gz -C /opt/module/
```

### 配置环境变量

```shell
sudo vim /etc/profile.d/env.sh

#HADOOP_HOME
export HADOOP_HOME=/opt/module/hadoop-3.1.3
export PATH=$PATH:$HADOOP_HOME/bin
export PATH=$PATH:$HADOOP_HOME/sbin
```

### 目录结构

![image-20210622093644939](README.assets/image-20210622093644939.png)

## 文件复制

### xsync脚本

循环复制文件到所有节点的相同目录下

```shell
# 在/home/username/bin下创建脚本文件xsync
cd ~
mkdir bin
cd bin
vim xsync

# 内容如下
#!/bin/bash
#1. 判断参数个数
if [ $# -lt 1 ]
then
echo Not Enough Arguement!
exit;
fi

#2. 遍历集群所有机器
for host in hadoop2 hadoop3 hadoop4
do
echo ==================== $host ====================
#3. 遍历所有目录，挨个发送
for file in $@
do
#4. 判断文件是否存在
if [ -e $file ]
then
#5. 获取父目录
pdir=$(cd -P $(dirname $file); pwd)
#6. 获取当前文件的名称
fname=$(basename $file)
ssh $host "mkdir -p $pdir"
rsync -av $pdir/$fname $host:$pdir
else
echo $file does not exists!
fi
done
done
```

修改xsync脚本的权限

```shell
chmod +x xsync
```

测试

```shell
# 在脚本目录下，将/home/wentimei/bin/ 拷贝到其他主机的相同目录
xsync /home/wentimei/bin/
```

全局调用

```shell
sudo cp /home/wentimei/bin/xsync /bin/
```

**Note: 如果在xsync前使用了sudo则需要加上xsync的全路径名**

```shell
# 在/home/wentimei路径下
sudo ./bin/xsync /home/wentimei/bin/
```

### scp 安全拷贝

基本语法

```shell
# 命令  递归  要拷贝的文件   目标主机和路径，该路径需要确保存在
  scp   -r    /opt/file    10.211.55.3:/opt/file
# 该命令可以从其他主机拉取文件到本地 参数交换即可，也可以将A的文件放置C
```

### rsync

用 *rsync* 做文件的复制要比 *scp* 的速度快，*rsync* 只对差异文件做更新而*scp* 是把所有文件都复制过去。

```shell
# 命令   归档拷贝、显示复制过程  要拷贝的文件   目标主机和路径，该路径需要确保存在
  rsync   -av                    /opt/file    10.211.55.3:/opt/file
```

## ssh免秘钥登录

- 生成公钥和私钥

```shell
cd ~
cd .ssh
ll
# 生成公钥和私钥
ssh-keygen -t rsa
```

- 将公钥拷贝至所有主机（包括自己，自己也要）

```shell
# .ssh目录下
ssh-copy-id hadoop
ssh-copy-id hadoop2
ssh-copy-id hadoop3
ssh-copy-id hadoop4
```

### 免密原理

![image-20210622165253895](README.assets/image-20210622165253895.png)

### .ssh目录结构

![image-20210622165412465](README.assets/image-20210622165412465.png)

![image-20210622165421782](README.assets/image-20210622165421782.png)

## 集群配置

### 1. 配置规划

![image-20210623135638873](README.assets/image-20210623135638873.png)

### 2. 配置文件

*Hadoop*有两类配置文件，默认配置文件和自定义配置文件

- 默认配置文件

默认配置文件路径为***$HADOOP_HOME/share/hadoop/xx.jar***，解析jar包获取对于的默认配置文件

![image-20210623140707575](README.assets/image-20210623140707575.png)

- 自定义配置文件

自定义配置文件存放在***$HADOOP_HOME/etc/hadoop***下包括：

**core-site.xml 、 hdfs-site.xml 、 yarn-site.xml 、 mapred-site.xml**

### 3. 修改自定义配置

#### 3.1 core-site.xml

```shell
# 进入配置文件目录
cd $HADOOP_HOME/etc/hadoop

vim core-site.xml
```

![image-20210623141737875](README.assets/image-20210623141737875.png)

文件内容如下，将*property*放入*configuration*中即可

```xml
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
<!-- 指定 NameNode 的地址 -->
<property>
<name>fs.defaultFS</name>
<value>hdfs://hadoop102:8020</value>
</property>
<!-- 指定 hadoop 数据的存储目录 -->
<property>
<name>hadoop.tmp.dir</name>
<value>/opt/module/hadoop-3.1.3/data</value>
</property>
<!-- 配置 HDFS 网页登录使用的静态用户为 hadoop -->
<property>
<name>hadoop.http.staticuser.user</name>
<value>hadoop</value>
</property>
</configuration>
```

#### 3.2 hdfs-site.xml

```shell
vim hdfs-site.xml
```

文件内容如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
<!-- nn web 端访问地址-->
<property>
<name>dfs.namenode.http-address</name>
<value>hadoop102:9870</value>
</property>
<!-- 2nn web 端访问地址-->
<property>
<name>dfs.namenode.secondary.http-address</name>
<value>hadoop104:9868</value>
</property>
</configuration>
```

#### 3.3 yarn-site.xml

```shell
vim yarn-site.xml
```

文件内容如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>

<configuration>
<!-- 指定 MR 走 shuffle -->
<property>
<name>yarn.nodemanager.aux-services</name>
<value>mapreduce_shuffle</value>
</property>
<!-- 指定 ResourceManager 的地址-->
<property>
<name>yarn.resourcemanager.hostname</name>
<value>hadoop103</value>
</property>
<!-- 环境变量的继承 该配置只针对3.1.3，高版本不需要配置-->
<property>
<name>yarn.nodemanager.env-whitelist</name>
<value>JAVA_HOME,HADOOP_COMMON_HOME,HADOOP_HDFS_HOME,HADOOP_CO
NF_DIR,CLASSPATH_PREPEND_DISTCACHE,HADOOP_YARN_HOME,HADOOP_MAP
RED_HOME</value>
</property>
</configuration>
```

#### 3.4 mapred-site.xml

```shell
vim mapred-site.xml
```

文件内容如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>

<configuration>
<!-- 指定 MapReduce 程序运行在 Yarn 上 -->
<property>
<name>mapreduce.framework.name</name>
<value>yarn</value>
</property>
  
<property>
  <name>yarn.app.mapreduce.am.env</name>
  <value>HADOOP_MAPRED_HOME=/opt/module/hadoop-3.1.3</value>
</property>
<property>
  <name>mapreduce.map.env</name>
  <value>HADOOP_MAPRED_HOME=/opt/module/hadoop-3.1.3</value>
</property>
<property>
  <name>mapreduce.reduce.env</name>
  <value>HADOOP_MAPRED_HOME=/opt/module/hadoop-3.1.3</value>
</property>
  
</configuration>
```

### 4. 配置分发

将所有机器的配置都改为一致

```shell
xsync /opt/module/hadoop-3.1.3/etc/hadoop/
```

## 启动

### 1. 配置workers并分发

```shell
vim /opt/module/hadoop-3.1.3/etc/hadoop/workers

# 将内容改为所有的主机名，注意该文件不能含有空格
hadoop2
hadoop3
hadoop4

xsync /opt/module/hadoop-3.1.3/etc/hadoop/workers
```

### 2. 第一次启动初始化 （仅第一次）

```shell
# hadoop目录下
hdfs namenode -format
```

格式化 NameNode，会产生新的集群 id，导致 NameNode 和 DataNode 的集群 id 不一致，集群找
不到已往数据。如果集群在运行过程中报错，需要重新格式化 NameNode 的话，一定要**先停止 namenode 和 datanode 进程**，并且要删除所有机器的 data 和 logs 目录，然后再进行格式
化

启动namenode会在data目录下生成name目录，name目录下有版本号和后面生成的data目录版本相对应，格式化之后namenode版本会产生变化不能和原来的data版本对应，所以需要删除data和logs目录

### 3. 启动HDFS

```shell
# hadoop目录下，在配置了NameNode节点启动
sbin/start-dfs.sh
```

访问地址：ip:9870 比如：10.211.55.18:9870

启动后应该和[配置规划](###1. 配置规划)的配置一致，下同

### 4. 启动yarn

```shell
# hadoop目录下，在配置了ResourceManager的节点启动
sbin/start-dfs.sh
```

访问地址：ip:8088比如：10.211.55.23:8088

### 5. 测试

```shell
# 注意：这里是在hdfs的路径上新建input文件夹
hadoop fs -mkdir /input

# 上传文件到hdfs的/input目录下
hadoop fs -put $HADOOP_HOME/wcinput/word.txt /input

# 这边路径同上，执行过程可在yarn页面上查看
hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-3.1.3.jar wordcount /input /output
```

## 配置历史服务器

### 配置mapred-site.xml

```shell
# $HADOOP_HOME/etc/hadoop
vim mapred-site.xml

# 添加如下配置
<!-- 历史服务器端地址 -->
<property>
<name>mapreduce.jobhistory.address</name>
<value>hadoop2:10020</value>
</property>
<!-- 历史服务器 web 端地址 -->
<property>
<name>mapreduce.jobhistory.webapp.address</name>
<value>hadoop2:19888</value>
</property>
```

### 分发

```shell
# $HADOOP_HOME/etc/hadoop
xsync mapred-site.xml
```

### 启动（Hadoop102）

```shell
 mapred --daemon start historyserver
 
 # 如果yarn已经启动，需要重新启动一下
```

## 开启日志聚集

![image-20210630091726755](assets/image-20210630091726755.png)

注意，**开启日志聚集功能需要重启yarn和历史服务器**

### 1. 配置yarn-site.xml

```shell
# $HADOOP_HOME/etc/hadoop/目录下
vim yarn-site.xml

# 添加如下配置
<!-- 开启日志聚集功能 -->
<property>
<name>yarn.log-aggregation-enable</name>
<value>true</value>
</property>
<!-- 设置日志聚集服务器地址 -->
<property> 
<name>yarn.log.server.url</name> 
<value>http://hadoop102:19888/jobhistory/logs</value>
</property>
<!-- 设置日志保留时间为 7 天 -->
<property>
<name>yarn.log-aggregation.retain-seconds</name>
<value>604800</value>
</property>
```

### 2. 分发

```shell
xsync $HADOOP_HOME/etc/hadoop/yarn-site.xml
```

### 3. 关闭服务后再次启动

```shell
# 关闭yarn (hadoop103)
sbin/stop-yarn.sh
# 关闭历史服务(hadoop102)
mapred --daemon stop historyserver
# 启动yarn (hadoop103)
sbin/start-yarn.sh
# 启动历史服务 （hadoop102）
mapred --daemon start historyserver
```

## 集群启停命令及脚本

### 1. 集群启停

##### 整体启/停 HDFS 和 YARN

```shell
start-dfs.sh/stop-dfs.sh

start-yarn.sh/stop-yarn.sh
```

### 2. 各组件单一启停

##### 分别启动/停止 HDFS 组件 和 YARN组件

```shell
hdfs --daemon start/stop namenode/datanode/secondarynamenode

yarn --daemon start/stop resourcemanager/nodemanager

# 历史服务启停
mapred --daemon start/stop historyserver
```

### 3. 集群启停脚本

```shell
cd ~
cd bin
vim cluster.sh

#!/bin/bash
if [ $# -lt 1 ]
then
echo "No Args Input..."
exit ;
fi
case $1 in
"start")
echo " =================== 启动 hadoop 集群 ==================="
echo " --------------- 启动 hdfs ---------------"
ssh hadoop102 "/opt/module/hadoop-3.1.3/sbin/start-dfs.sh"
echo " --------------- 启动 yarn ---------------"
ssh hadoop103 "/opt/module/hadoop-3.1.3/sbin/start-yarn.sh"
echo " --------------- 启动 historyserver ---------------"
ssh hadoop102 "/opt/module/hadoop-3.1.3/bin/mapred --daemon start 
historyserver"
;;
"stop")
echo " =================== 关闭 hadoop 集群 ==================="
echo " --------------- 关闭 historyserver ---------------"
ssh hadoop102 "/opt/module/hadoop-3.1.3/bin/mapred --daemon stop 
historyserver"
echo " --------------- 关闭 yarn ---------------"
ssh hadoop103 "/opt/module/hadoop-3.1.3/sbin/stop-yarn.sh"
echo " --------------- 关闭 hdfs ---------------"
ssh hadoop102 "/opt/module/hadoop-3.1.3/sbin/stop-dfs.sh"
;;
*)
echo "Input Args Error..."
;;
esac

```

##### 3.1 权限

```shell
chmod +x cluster.sh
```

### 4. 查看所有机器jps脚本并设置权限

```shell
cd ~
cd bin
vim jpsall

# input
#!/bin/bash
for host in hadoop102 hadoop103 hadoop104
do
echo =============== $host ===============
ssh $host jps 
done

chmod +x jpsall
```

### 5. 分发脚本给所有机器

```shell
xsync /home/hadoop/bin/
```

