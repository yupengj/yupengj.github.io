---
layout: post
title: "docker 搭建本地 spark 测试环境"
date: 2019-04-25 20:00:00
tags: spark
---

1. 虚拟 vmware 中安装 ubuntu
2. 虚拟机中 ubuntu 系统安装 docker
3. 安装并运行 spark
4. scala 控制台版本 wordCount.
5. IDEA 版本 JavaWordCount
6. 利用 spark-submit 命令提交作业 JavaWordCount




## 虚拟 vmware 中安装 ubuntu

> 我这里 vmware 和 ubuntu 下载都是在官网下的最新的，文件很大需要等一段时间。

- 下载 [VMware Workstation Pro](https://my.vmware.com/cn/web/vmware/downloads)
- 下载 [ubuntu ios](https://www.ubuntu.com/download/desktop)
- 下载后在 VMware 中安装 Ubuntu [参考地址](https://blog.csdn.net/stpeace/article/details/78598333)
- 安装完成后，为了实现虚拟机和物理主机之间文件复制安装 vmware tools [参考地址](https://jingyan.baidu.com/article/597a0643356fdc312b5243f6.html)
- ubuntu ios 中默认没有安装 vim。 安装 vim [参考地址](https://jingyan.baidu.com/article/046a7b3efd165bf9c27fa915.html)
- 安装 vim 遇到一个问题：“无法获得锁 /var/lib/dpkg/lock-frontend - open (11: 资源暂时不可用)” 解决方法 [参考地址](https://jingyan.baidu.com/article/a65957f435f60b24e67f9b07.html)

## 虚拟机中 ubuntu 系统安装 docker

- [参考官网](https://docs.docker.com/install/linux/docker-ce/ubuntu/) 

## 安装并运行 spark

- 拉取镜像 `sudo docker pull mesosphere/spark:2.4.0-2.2.1-3-hadoop-2.6`, ":" 后是镜像的版本。[版本列表](https://hub.docker.com/r/mesosphere/spark/tags/)
- 运行镜像 `sudo docker run --name spark1 --hostname spark1 -it mesosphere/spark:2.4.0-2.2.1-3-hadoop-2.6 /bin/bash`
- 启动后会进入到 bash 终端，设置环境变量
    ```bash
    export PATH=$JAVA_HOME/bin:$PATH
    export SPARK_HOME=/opt/spark
    export PATH=$SPARK_HOME/bin:$PATH
    ```
启动 spark-shell 启动成功后如下图，会进入 scala 控制台

![启动spark](https://github.com/yupengj/yupengj.github.io/blob/master/images/spark_start.png?raw=true)


### scala 控制台版本 wordCount. 输入以下代码

| 下面的例子中统计的是 spark 的 README.md 文件中出现频率前10的单词

```scala
// 读取文件每行记录
val lines = sc.textFile("file:///opt/spark/README.md")
// 用空格分割提取每个单词并统计出现的次数
val words = lines.flatMap(_.split(" ")).map(x => (x,1)).reduceByKey(_+_)
// 按照单词出现的次数降序
val wordsort = words.map(x => (x._2, x._1)).sortByKey(false).map(x => (x._2, x._1))
// 取出前 10 个记录
wordsort.take(10)
```
运行结果如图

![wordCount](https://github.com/yupengj/yupengj.github.io/blob/master/images/spark_wordCount.png?raw=true)

### IDEA 版本 JavaWordCount
```
package org.jyp.graphx;

import java.util.List;
import java.util.regex.Pattern;
import java.util.stream.Collectors;
import java.util.stream.Stream;

import org.apache.spark.SparkConf;
import org.apache.spark.SparkContext;
import org.apache.spark.api.java.JavaPairRDD;
import org.apache.spark.api.java.JavaRDD;
import scala.Tuple2;

public class JavaWordCount {
	private static final Pattern SPACE = Pattern.compile(" ");

	public static void main(String[] args) throws Exception {
	    if (args.length < 1) {
			System.err.println("Usage: JavaWordCount <file>");
			System.exit(1);
		}
		SparkConf sparkConf = new SparkConf().setAppName("JavaWordCount").setMaster("local[*]");
		SparkContext spark = new SparkContext(sparkConf);
		JavaRDD<String> lines = spark.textFile(args[0], 2).toJavaRDD();
		JavaRDD<String> words = lines.flatMap(s -> Stream.of(SPACE.split(s)).collect(Collectors.toList()).iterator());
		JavaPairRDD<String, Integer> ones = words.mapToPair(s -> new Tuple2<>(s, 1));
		JavaPairRDD<String, Integer> counts = ones.reduceByKey(Integer::sum);
		JavaPairRDD<String, Integer> newCounts = counts.mapToPair(row -> new Tuple2<>(row._2, row._1)).sortByKey(false).mapToPair(row -> new Tuple2<>(row._2, row._1));
		List<Tuple2<String, Integer>> output = newCounts.take(10);
		System.out.println(output.toString());
		spark.stop();
	}
}
```
运行结果 [(,72), (the,24), (to,17), (Spark,16), (for,12), (and,10), (##,9), (a,9), (can,7), (is,7)]

**具体项目的配置可以查看[这里](https://github.com/yupengj/spark-graphx)**

### 利用 spark-submit 命令提交作业 JavaWordCount

- 把 IDEA 版本的 JavaWordCount 打成 jar 
- 在 ubuntu 系统中新建文件夹，并把 jar 复制到新建的文件夹中，我这里新建的文件夹路径是/home/jyp/spark_jar
- 利用命令 `docker exec -it spark1 /bin/bash` 进入前面启动的 docker 容器，并创建文件夹 /home/spark_jar
- 利用命令 `docker cp /home/jyp/spark_jar/spark-graphx-1.0-SNAPSHOT.jar spark1:/home/spark_jar`(在 ubuntu 系统中执行该命令) 复制jar 到 docker 容器中。
- 在 docker 容器中提交作业 `./bin/spark-submit --class org.jyp.graphx.JavaWordCount /home/spark_jar/spark-graphx-1.0-SNAPSHOT.jar /opt/spark/README.md`
- 看输出结果是否与 scala 控制台版本的结果相同