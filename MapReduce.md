# 1 概述

MapReduce是一种可用于数据处理的编程模型，Hadoop可以运行各种语言版本的MapReduce程序。**MapReduce既是一个编程模型，又是一个计算框架**。也就是说，开发人员必须基于MapReduce编程模型进行编程开发，然后将程序通过MapReduce计算框架分发到Hadoop集群中运行。

> 其编程模型只包含Map和Reduce两个过程，map的主要输入是一对<Key, Value>值，经过map计算后输出一对<Key, Value>值；然后将相同Key合并，形成<Key, Value集合>；再将这个<Key, Value集合>输入reduce，经过计算输出零个或多个<Key, Value>对。

# 2 序列化

序列化是指将结构化对象转化为字节流以便在网络上传输或写到磁盘进行永久存储的过程。反序列化是指将字节流转回结构化对象的逆过程

**序列化用于分布式数据处理的两大领域：进程间通行和永久存储**

在Hadoop中，系统中多个节点上进程间的通信是通过RPC（远程过程调用）实现的。RPC协议将消息序列化成二进制流后发送到远程节点，远程节点接着将二进制流反序列化为原始消息。通常情况下RPC有如下序列化格式

- 紧凑：充分利用网络带宽，进而高效使用存储空间
- 快速：尽量减少序列化和反序列化的性能开销
- 可扩展：可透明的读取老格式的数据
- 支持互操作：可以支持不同语言读/写永久存储的数据

这些属性对持久存储格式十分重要

## 2.1 Wirtable

![image-20210707102931723](assets/image-20210707102931723.png)

Hadoop使用的是自己的序列化格式Writable，它紧凑、速度快但不太容易用Java以外的语言进行扩展或使用

> 不使用Java Serialization是因为它太复杂了，Hadoop需要一个至精至简的机制，用于精确控制对象的读和写，这个机制是Hadoop的核心
>
> Java Serialization不满足前面的序列化格式标准：快速、紧凑、可扩展、支持互操作
>
> 不用RMI（远程方法调用）也是出于类似的考虑。高效、高性能的进程间通信是Hadoop的关键

## 2.2 自定义对象实现序列化接口

Hadoop提供的序列化类是有限的，但我们需要自定义对象时就需要实现序列化接口进行传输（原因上面已经说过[序列化](#2 序列化)）

自定义对象实现序列化接口时需要遵循如下步骤

- 实现Writable接口
- 反序列化是必须提供空参构造（反射）
- 重写序列化及反序列化方法

```java
@Override
public void write(DataOutput out) throws IOException {
  // 根据属性类型来决定写什么
  out.writeLong(upFlow);
	out.writeLong(downFlow);
	out.writeLong(sumFlow);
}

@Override
public void readFields(DataInput in) throws IOException {
	upFlow = in.readLong();
	downFlow = in.readLong();
	sumFlow = in.readLong();
}
```

- **反序列化的顺序和序列化写入顺序要一致**
- 重写toString方法用来格式化输出文件

```java
@Override
public String toString() {
    return upFlow + "\t" + downFlow + "\t" + sumFlow;
}
```

- 如果需要将自定义对象放在key中传输，则还需要实现 Comparable 接口，因为MapReduce中的 Shuffle过程要求对key必须能排序（考虑WritableComparable接口）

# 3 应用开发

MapReduce编程遵循一个特定的流程。首先写一个map函数和reduce函数，最好使用单元测试来确保函数的运行符合预期。然后写一个驱动程序来运行作业，看这个驱动程序是否可以正确运行

## 3.1 开发步骤

### Mapper

![image-20210706132653900](assets/image-20210706132653900.png)

### Reducer

![image-20210706132723535](assets/image-20210706132723535.png)

### Driver

相当于YARN集群的客户端，用于提交我们整个程序到YARN集群，提交的是封装了MapReduce程序相关运行参数的job对象

## 3.2 WordCount案列

如果写成内部类的形式注意一定要用static修饰

```java
package cn.huakai;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.Reducer;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;

import java.io.IOException;

/**
 * @author: huakaimay
 * @since: 2021-07-06
 */
public class WordCountDriver {

    public static void main(String[] args) throws IOException, InterruptedException, ClassNotFoundException {
        
        // 获取job实例
        Job job = Job.getInstance();

        // 关联Driver
        job.setJarByClass(WordCountDriver.class);

        // 关联Mapper和Reducer
        job.setMapperClass(WordCountMapper.class);
        job.setReducerClass(WordCountReducer.class);

        // 设置Map的输出kv
        job.setMapOutputKeyClass(Text.class);
        job.setMapOutputValueClass(LongWritable.class);

        // 设置最终输出kv
        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(LongWritable.class);

        // 输入和输出路径
        FileInputFormat.setInputPaths(job, new Path(args[0]));
        FileOutputFormat.setOutputPath(job, new Path(args[1]));

        // 提交job
        boolean result = job.waitForCompletion(true);
        System.exit(result ? 0 : 1);

    }

    /**
     * 文本输入类型key为偏移量，value为字符串
     */
    public static class WordCountMapper extends Mapper<LongWritable, Text, Text, LongWritable> {
        private Text text = new Text();
        private LongWritable longWritable = new LongWritable(1);

        @Override
        protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
            // 读取文本时的key
            String inKey = value.toString();
            // 每行可能含有空格
            String[] keyArr = inKey.split(" ");
            for (String word : keyArr) {
                text.set(word);
                context.write(text, longWritable);
            }
        }
    }

    public static class WordCountReducer extends Reducer<Text, LongWritable, Text, LongWritable> {
        private LongWritable longWritable = new LongWritable();

        @Override
        protected void reduce(Text key, Iterable<LongWritable> values, Context context) throws IOException, InterruptedException {

            long sum = 0;
            for (LongWritable value : values) {
                sum += value.get();
            }
            longWritable.set(sum);

            context.write(key, longWritable);
        }
    }


}

```

### 打包

maven打包（带有依赖）插件

```xml
 <build>
        <plugins>
            <plugin>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.6.1</version>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                </configuration>
            </plugin>
            <plugin>
                <artifactId>maven-assembly-plugin</artifactId>
                <configuration>
                    <descriptorRefs>
                        <descriptorRef>jar-with-dependencies</descriptorRef>
                    </descriptorRefs>
                </configuration>
                <executions>
                    <execution>
                        <id>make-assembly</id>
                        <phase>package</phase>
                        <goals>
                            <goal>single</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
```

### 集群测试

将jar包放置指定目录（/opt/modules/hadoop）下

```shell
# 全限定类名
hadoop jar xxx.jar xxx.WordCountDriver /input /output
```

## 3.3 序列化案例 

需求：统计每个手机号的上行流量、下行流量及总流量（可能含有重复手机号）

格式如下

```
1	13736230513	192.196.100.1	www.atguigu.com	2481	24681	200
2	13846544121	192.196.100.2			264	0	200
3 	13956435636	192.196.100.3			132	1512	200
4 	13966251146	192.168.100.1			240	0	404
5 	18271575951	192.168.100.2	www.atguigu.com	1527	2106	200
6 	84188413	192.168.100.3	www.atguigu.com	4116	1432	200
7 	13590439668	192.168.100.4			1116	954	200
8 	15910133277	192.168.100.5	www.hao123.com	3156	2936	200
9 	13729199489	192.168.100.6			240	0	200
10 	13630577991	192.168.100.7	www.shouhu.com	6960	690	200
11 	15043685818	192.168.100.8	www.baidu.com	3659	3538	200
12 	15959002129	192.168.100.9	www.atguigu.com	1938	180	500
13 	13560439638	192.168.100.10			918	4938	200
14 	13470253144	192.168.100.11			180	180	200
15 	13682846555	192.168.100.12	www.qq.com	1938	2910	200
16 	13992314666	192.168.100.13	www.gaga.com	3008	3720	200
17 	13509468723	192.168.100.14	www.qinghua.com	7335	110349	404
18 	18390173782	192.168.100.15	www.sogou.com	9531	2412	200
19 	13975057813	192.168.100.16	www.baidu.com	11058	48243	200
20 	13768778790	192.168.100.17			120	120	200
21 	13568436656	192.168.100.18	www.alibaba.com	2481	24681	200
22 	13568436656	192.168.100.19			1116	954	200
```

![image-20210707104232152](assets/image-20210707104232152.png)

期望输出如下 

![image-20210707104255873](assets/image-20210707104255873.png)

### 实体类

```java
package cn.huakai.writable;

import org.apache.hadoop.io.Writable;

import java.io.DataInput;
import java.io.DataOutput;
import java.io.IOException;

/**
 * 流量传输对象
 *
 * @author: huakaimay
 * @since: 2021-07-07
 */
public class FlowDTO implements Writable {

    /**
     * 上行流量
     */
    private Long upFlow;

    /**
     * 下行流量
     */
    private Long downFlow;

    /**
     * 总流量
     */
    private Long sumFlow;

    public Long getUpFlow() {
        return upFlow;
    }

    public void setUpFlow(Long upFlow) {
        this.upFlow = upFlow;
    }

    public Long getDownFlow() {
        return downFlow;
    }

    public void setDownFlow(Long downFlow) {
        this.downFlow = downFlow;
    }

    public Long getSumFlow() {
        return sumFlow;
    }

    public void setSumFlow(Long sumFlow) {
        this.sumFlow = sumFlow;
    }

    public void setSumFlow() {
        this.sumFlow = this.upFlow + this.downFlow;
    }

    @Override
    public void write(DataOutput out) throws IOException {
        out.writeLong(upFlow);
        out.writeLong(downFlow);
        out.writeLong(sumFlow);
    }

    @Override
    public void readFields(DataInput in) throws IOException {
        upFlow = in.readLong();
        downFlow = in.readLong();
        sumFlow = in.readLong();
    }

    @Override
    public String toString() {
        return upFlow + "\t" + downFlow + "\t" + sumFlow;
    }
}
```

### Driver

注意运行时要将mrunit从pom依赖中删除

```java
package cn.huakai.writable;

import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.Reducer;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
import org.apache.hadoop.mrunit.mapreduce.MapDriver;
import org.junit.Test;

import java.io.IOException;

/**
 * @author: huakaimay
 * @since: 2021-07-07
 */
public class FlowDriver {

    public static void main(String[] args) throws IOException, InterruptedException, ClassNotFoundException {
        Job job = Job.getInstance();

        // jar
        job.setJarByClass(FlowDriver.class);

        // mapper and reducer
        job.setMapperClass(FlowMapper.class);
        job.setReducerClass(FlowReducer.class);

        // map output key & value
        job.setMapOutputValueClass(Text.class);
        job.setMapOutputValueClass(FlowDTO.class);

        // output key & value
        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(FlowDTO.class);

        // fileinput & output
        FileInputFormat.setInputPaths(job, new Path("/Users/wentimei/Downloads/phone_data.txt"));
        FileOutputFormat.setOutputPath(job, new Path("/Users/wentimei/Downloads/output"));

        // submit
        System.exit(job.waitForCompletion(true) ? 0 : 1);

    }

    public static class FlowMapper extends Mapper<LongWritable, Text, Text, FlowDTO> {
        private FlowDTO flowDTO = new FlowDTO();
        private Text outKey = new Text();

        // 1   13736230513    192.196.100.1  www.atguigu.com    2481   24681  200
        @Override
        protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
            String[] split = value.toString().split("\t");
            String phone = split[1];
            // 因为前面的数据不全，切割后数据不一致，而后面数据完整，从后切割即可保证数据正确
            String upFlow = split[split.length - 3];
            String downFlow = split[split.length - 2];
            flowDTO.setUpFlow(Long.valueOf(upFlow));
            flowDTO.setDownFlow(Long.parseLong(downFlow));
            flowDTO.setSumFlow();
            outKey.set(phone);
            context.write(outKey, flowDTO);
        }
        @Test
        public void recoder() throws IOException {
            Text text = new Text("2\t13846544121\t192.196.100.2\t\t\t264\t0\t200");
            FlowDTO flowDTO = new FlowDTO();
            flowDTO.setUpFlow(264l);
            flowDTO.setDownFlow(0l);
            flowDTO.setSumFlow();
            new MapDriver<LongWritable, Text, Text, FlowDTO>()
                    .withMapper(new FlowMapper())
                    .withInput(new LongWritable(0), text)
                    .withOutput(new Text("13846544121"), flowDTO)
                    .runTest();
        }
    }

    public static class FlowReducer extends Reducer<Text, FlowDTO, Text, FlowDTO> {
        private FlowDTO flowDTO = new FlowDTO();
        @Override
        protected void reduce(Text key, Iterable<FlowDTO> values, Context context) throws IOException, InterruptedException {

            Long totalUp = 0l;
            Long totalDown = 0l;
            for (FlowDTO value : values) {
                totalUp += value.getUpFlow();
                totalDown += value.getDownFlow();
            }
            flowDTO.setUpFlow(totalUp);
            flowDTO.setDownFlow(totalDown);
            flowDTO.setSumFlow();

            context.write(key,flowDTO);
        }
    }
}
```

