# 1 关于Yarn

​	Apache YARN（Yet Another Resource Negotiator）是Hadoop的集群资源管理系统，负责为运算程序提供服务器运算资源，相当于一个分布式的操作系统平台，而MapReduce等运算程序则相当于运行于操作系统之上的应用程序

# 2 运行机制

​	Yarn通过两类长期运行的守护进程提供自己的核心服务：管理集群上资源使用的资源管理器（resource manager）、运行在集群中所有节点上且能够启动和监控容器（container）的节点管理器（node manager）。容器用于执行特定应用程序的进程，每个容器都有资源限制（内存、CPU等）。一个容器可以是一个Unix进程，也可以是一个Linux cgroup，取决于Yarn的配置

![image-20210719133525003](assets/image-20210719133525003.png)

