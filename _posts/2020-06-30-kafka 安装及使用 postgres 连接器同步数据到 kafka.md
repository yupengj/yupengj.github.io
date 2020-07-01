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
```
bin/windows/zookeeper-server-start.bat config/zookeeper.properties
```

- 启动 kafak server
```
bin/windows/kafka-server-start.bat config/server.properties
```

- 创建 test 主题
```
bin/windows/kafka-topics.bat --create --bootstrap-server localhost:9092 --replication-factor 1 --partitions 1 --topic test
```

- 查看主题,控制台会输出 test
```
bin/windows/kafka-topics.bat --list --bootstrap-server localhost:9092 
```

### 创建 producer 和 consumer 测试消息同步

- 创建生成者客户端并输入测试消息。注 2.5版本后需要把 --broker-list 改为 --bootstrap-server
 ```
$ bin/windows/kafka-console-producer.bat --broker-list localhost:9092 --topic test
>nihao
>你好
```

- 创建消费者客户端接收消息。这里中文乱码暂时没有找到解决办法。
```
$ bin/windows/kafka-console-consumer.bat --bootstrap-server localhost:9092 --topic test --from-beginning
>nihao
>锟斤拷锟
```

- 继续在生产者客户端发送消息，消费者客户端会自动同步消息。

### java 客户端创建 producer 和 consumer 测试消息同步[java客户端源码地址](https://github.com/yupengj/kafka-examples)

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
# 插件地址,为了清楚区分插件的 jar 和 kafka 的 jar 在 kafka 下增加插件文件夹（plugins），启动连接器时会加载该目录下的 jar。
# 如果没有配置这个路径需要把插件的所有jar复制到 kafka 的 libs 目录下
plugin.path=/software/kafka_2.12-2.3.0/plugins
```

- 下载 debezium-connector-postgres 插件 [https://debezium.io/releases/1.1/](https://debezium.io/releases/1.1/)。 把下载的文件解压后放到配置文件中定义的插件位置也就是 /software/kafka_2.12-2.3.0/plugins 中
![](https://raw.githubusercontent.com/yupengj/yupengj.github.io/master/images/2020/plugins.png)

- 启动连接器 Worker, 启动时有类着不到警告，暂时没有找到原因。
```
bin/windows/connect-distributed.bat config/connect-distributed.properties 
```

- 查看所有的连接器插件 `curl localhost:8083/connector-plugins`. 第一个是新增加的，后面两个是 kafka 自带的。
![](https://raw.githubusercontent.com/yupengj/yupengj.github.io/master/images/2020/curl_plugins.png)

## 创建 debezium-connector-postgres 连接器同步数据到 kafka

创建 postgres 连接器之前，需要在 postgres 中安装 wal2json 插件或者 decoderbufs 插件。
wal2json 插件安装可以参考我之前写的[文章](https://yupengj.github.io/2020/06/27/postgresql-%E5%AE%89%E8%A3%85-wal2json-%E6%8F%92%E4%BB%B6/)
decoderbufs 插件安装可以参考 github 上的[官网文档](https://github.com/debezium/postgres-decoderbufs)

### 准备同步数据

在 postgres 创建用于测试的 schema 名称为 "test"。并在 test scheme 下创建 test 表，并写入100条数据
```postgresql
create schema test;
create table test.test_slot(id serial8 primary key, a text, b float8, c timestamp);
do
$$
    declare
        num integer := 1;
    begin
        while num <= 100
            loop
                insert into test.test_slot (a, b, c) values (num||'', num, now());
                num := num + 1;
            end loop;
    end
$$;
```

### 创建 postgres 连接器

- 编写 postgres 连接器配置文件
创建名称为 `connect-postgres-source.json` 的json文件放在 kafka 的 config 目录下. 文件内容如下，其中 database 开头的配置是配置数据库连接的参数。
其他配置就不一一解释请参考官方文档。最下方有文档连接
```json
{
  "name": "pg-source",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "tasks.max": "1",
    "database.hostname": "loaclhost",
    "database.port": "5432",
    "database.user": "postgres",
    "database.password": "app",
    "database.dbname": "postgres",
    "database.server.name": "test",
    "schema.whitelist": "test",
    "table.blacklist": "",
    "plugin.name": "wal2json",
    "snapshot.mode": "initial",
    "slot.name": "test"
  }
}
```

- 创建 postgres 连接器
-d 是连接器的配置文件路径。我这个请求是在kafka目录下执行的，所以配置文件相对路径就是config/connect-postgres-source.json。命令在哪里执行都可以只要配置文件路径正确就可以
```
curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" localhost:8083/connectors/ -d @config/connect-postgres-source.json
```
注意连接器创建前必须保证连接器 Worker 已启动。创建连接器后可通过 API 查看所有启动的连接器`curl localhost:8083/connectors`和连接器的状态`curl localhost:8083/connectors/pg-source/status`
![](https://raw.githubusercontent.com/yupengj/yupengj.github.io/master/images/2020/connectors.png)

- 创建消费者客户端测试同步数据。连接器创建成功后会自动同步数据到 kafka 的指定主题中，主题的名称为配置中的`database.server.name`.`schemaName`.`tableName`，如这里主题的名称是 'test.test.test_slot'
```
bin/windows/kafka-console-consumer.bat --bootstrap-server localhost:9092 --topic test.test.test_slot --from-beginning
```

如果一切正常控制台会打印同步到的数据。之后在 test_slot 表中增加、修改、删除操作检查是否可以实时同步到数据。至于不同操作同步到的数据结构可以到 debezium-connector-postgres 官方文档中查看。文档中有详细介绍。
```postgresql
insert into test.test_slot (a, b, c) values ('test', 10, now());
update test.test_slot set a = 'test1' where a = 'test';
delete from test.test_slot where a ='test1';
```

## 参考地址
- [java客户端源码地址](https://github.com/yupengj/kafka-examples)
- [kafka 快速开始官方文档](http://kafka.apache.org/quickstart)
- [kafka 连接器官方文档](http://kafka.apache.org/documentation/#connect)
- [debezium-connector-postgres 官方文档](https://debezium.io/documentation/reference/1.1/connectors/postgresql.html)