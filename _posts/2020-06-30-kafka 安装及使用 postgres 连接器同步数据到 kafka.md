---
layout: post
title: "kafka 安装及使用 postgres 连接器同步数据到 kafka"
date: 2020-06-27 20:00:00
tags: kafka
---

## 系统环境及软件版本

- window10 64位 系统
- kafka 2.3.0 版本(下载地址 http://kafka.apache.org/downloads)
- debezium-connector-postgres 1.1.2.Final 版本(下载地址 https://debezium.io/releases/1.1/)





## 启动并测试 kafka

- 下载 kafka 解压后目录结构
![](https://raw.githubusercontent.com/yupengj/yupengj.github.io/master/images/2020/kafka.png)

- 进入kafka文件夹下执行下面命令启动 ZooKeeper
```shell script
bin/windows/zookeeper-server-start.bat config/zookeeper.properties
```
- 启动 kafak server
```shell script
bin/windows/kafka-server-start.bat config/server.properties
```
- 创建 test 主题
```shell script
bin/windows/kafka-topics.bat --create --bootstrap-server localhost:9092 --replication-factor 1 --partitions 1 --topic test
```
- 查看主题,控制台会输出 test
```shell script
bin/windows/kafka-topics.bat --list --bootstrap-server localhost:9092 
```

### 创建 producer 和 consumer

创建生成者客户端并输入测试消息。注 2.5版本后需要把 --broker-list 改为 --bootstrap-server
 ```shell script
$ bin/windows/kafka-console-producer.bat --broker-list localhost:9092 --topic test
>nihao
>你好
```
创建消费者客户端接收消息。这里中文乱码暂时没有找到解决办法。
```shell script
$ bin/windows/kafka-console-consumer.bat --bootstrap-server localhost:9092 --topic test --from-beginning
>nihao
>锟斤拷锟
```
继续在生产值客户端发送消息，消费者客户端会自动同步消息。

### java 客户端测试消 producer 和 consumer

- Producer 类，向 test 主题发送10条消息
```java
@Slf4j
public class Producer {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        Properties props = new Properties();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, KafkaConfig.BOOTSTRAP_SERVERS_CONFIG);
        props.put(ProducerConfig.CLIENT_ID_CONFIG, "test_1");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        KafkaProducer<String, String> kafkaProducer = new KafkaProducer<>(props);
        String topic = "test";
        int count = 0;
        while (true) {
            String messageKey = "key_" + (++count) + "你好";
            String messageValue = "value_" + count + "你好";
            kafkaProducer.send(new ProducerRecord<>(topic, messageKey, messageValue)).get();
            log.info("send message topic {} ( {}, {})", topic, messageKey, messageValue);
            if (count > 10) {//只发送10条消息
                break;
            }
        }
    }
}
```
-  Consumer 类，拉取 test 主题的消息，连续10次没有拉取到数据时结束轮询
```java
@Slf4j
public class Consumer {
    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, KafkaConfig.BOOTSTRAP_SERVERS_CONFIG);// kafka 集群
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "test_1"); // 消费组id
        props.put(ConsumerConfig.CLIENT_ID_CONFIG, "test_1"); // 消费客户端id
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest"); // 从消息开始的位置读
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false"); // 不自动管理偏移量,即不记录消费者偏移量，可以重复读取数据方便测试
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);

		final String topic = "test";
        KafkaConsumer<String, String> kafkaConsumer = new KafkaConsumer<>(props);
        kafkaConsumer.subscribe(Collections.singletonList(topic));
        long start = System.currentTimeMillis();
        int count = 0, num = 10;// 有10次拉取的数据记录为 0 时 结束轮询
        while (true) {
            ConsumerRecords<String, String> records = kafkaConsumer.poll(Duration.ofSeconds(1));
            for (ConsumerRecord<String, String> record : records) {
                log.info("Received message topic {} : ({}, {}) at partition {} offset {}", record.topic(), record.key(), record.value(), record.partition(), record.offset());
            }
            count += records.count(); // 记录累加
            if (records.count() == 0) {
                num--;
                if (num < 0) {
                    break;
                }
            }
        }
        log.info("poll topic {} size {} time {} ms", topic, count, System.currentTimeMillis() - start);
    }
}
```

## 启动连接器 Worker 安装 debezium-connector-postgres 连接器插件

- 修改 kafka 文件夹下 config/connect-distributed.properties 文件增加以下配置
```properties
# rest 请求地址
rest.host.name=localhost
rest.port=8083
# 插件地址,为了清楚区分插件的 jar 和 kafka 的 jar 这里增加产品文件路径，启动连接器时会加载该目录下的 jar。
# 如果配置这个路径需要把插件的所有jar复制到 kafka 的 libs目录下
plugin.path=/software/kafka_2.12-2.3.0/plugins
```
- 下载 debezium-connector-postgres 插件 ![https://debezium.io/releases/1.1/](https://debezium.io/releases/1.1/). 把下载的文件解压后放到配置文件中定义的插件位置也就是 /software/kafka_2.12-2.3.0/plugins 中
![](https://raw.githubusercontent.com/yupengj/yupengj.github.io/master/images/2020/plugins.png)
- 启动连接器 Worker, 启动时有类着不到警告，暂时没有找到原因。
```shell script
bin/windows/connect-distributed.bat config/connect-distributed.properties 
```
- 查看所有的连接器插件 `curl localhost:8083/connector-plugins`. 第一个是新增加的，后面两个是 kafka 自带的。
![](https://raw.githubusercontent.com/yupengj/yupengj.github.io/master/images/2020/curl_plugins.png)

## 启动 debezium-connector-postgres 连接器同步数据到 kafka


## 参考地址
- [代码地址](https://github.com/yupengj/kafka-examples)
- [kafka 官方文档](http://kafka.apache.org/quickstart)
- [debezium-connector-postgres 官方文档](https://debezium.io/documentation/reference/1.1/connectors/postgresql.html)