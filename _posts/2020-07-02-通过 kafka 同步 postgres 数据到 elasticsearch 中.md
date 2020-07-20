---
layout: post
title: "通过 kafka 同步 postgres 数据到 elasticsearch 中"
date: 2020-07-02 20:00:00
tags: kafka elasticsearch
---

## 系统环境及软件版本

- window10 64位 系统
- kafka 2.3.0 版本(下载地址 http://kafka.apache.org/downloads)
- postgres 连接器 debezium-connector-postgres 1.1.2.Final 版本(下载地址 https://debezium.io/releases/1.1/)
- es 连接器 confluentinc-kafka-connect-elasticsearch-5.5.1 版本(下载地址 https://www.confluent.io/hub/confluentinc/kafka-connect-elasticsearch)




## 下载连接器插件

根据上面的下载地址下载这两个连接器插件，下载后解压放到 `/software/kafka_2.12-2.3.0/plugins` 文件夹中，因为下面连接器的会配置这个地址为插件地址，启动连接器时会自动加载文件夹下的 jar 包。

![](https://raw.githubusercontent.com/yupengj/yupengj.github.io/master/images/2020/plugins.png)

## 启动 kafka 连接器

- 首先启动连接器前要确保 kafka server 已经启动。修改 kafka 文件夹下 config/connect-distributed.properties 文件增加以下配置
```properties
# rest 请求地址
rest.host.name=localhost
rest.port=8083
# 插件地址,为了清楚区分插件的 jar 和 kafka 的 jar 在 kafka 下增加插件文件夹（plugins），启动连接器时会加载该目录下的 jar。
# 如果没有配置这个路径需要把插件的所有jar复制到 kafka 的 libs 目录下
plugin.path=/software/kafka_2.12-2.3.0/plugins
```

- 启动连接器 Worker, 启动时有类着不到警告，暂时没有找到原因。
```
bin/windows/connect-distributed.bat config/connect-distributed.properties 
```

- 查看所有的连接器插件 `curl localhost:8083/connector-plugins`. 一共有4个连机器，前两个是新增加的，后两个是 kafka 自带的
```json
[{
    "class": "io.confluent.connect.elasticsearch.ElasticsearchSinkConnector",
    "type": "sink",
    "version": "5.5.1"
}, {
    "class": "io.debezium.connector.postgresql.PostgresConnector",
    "type": "source",
    "version": "1.1.2.Final"
}, {
    "class": "org.apache.kafka.connect.file.FileStreamSinkConnector",
    "type": "sink",
    "version": "2.3.0"
}, {
    "class": "org.apache.kafka.connect.file.FileStreamSourceConnector",
    "type": "source",
    "version": "2.3.0"
}]
```

> 到这里说明连接器工作节点已启动，并且连接 postgresql 和 es 的连接器都已经被加载成功


## 创建 postgres 连接器

创建 postgres 连接器之前，需要在 postgres 中安装 wal2json 插件或者 decoderbufs 插件。
wal2json 插件安装可以参考我之前写的[文章](https://yupengj.github.io/2020/06/27/postgresql-%E5%AE%89%E8%A3%85-wal2json-%E6%8F%92%E4%BB%B6/)
decoderbufs 插件安装可以参考 github 上的[官网文档](https://github.com/debezium/postgres-decoderbufs)

### 准备同步数据

在 postgres 创建用于测试的 schema 名称为 "test"。并在 test scheme 下创建 test_slot 表，并写入100条数据
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

### postgres 连接器配置

创建名称为 `connect-postgres-source.json` 的json文件放在 kafka 的 config 目录下. 文件内容如下，其中 database 开头的配置是配置数据库连接的参数。`schema.whitelist` 是要同步 scheme 下所有的表
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

### 创建 postgres 连接器

-d 是连接器的配置文件路径。我这个请求是在kafka目录下执行的，所以配置文件相对路径就是config/connect-postgres-source.json。命令在哪里执行都可以只要配置文件路径正确就可以
```
curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" localhost:8083/connectors/ -d @config/connect-postgres-source.json
```
注意连接器创建前必须保证连接器 Worker 已启动。创建连接器后可通过 API 查看所有启动的连接器`curl localhost:8083/connectors`和连接器的状态`curl localhost:8083/connectors/pg-source/status`
![](https://raw.githubusercontent.com/yupengj/yupengj.github.io/master/images/2020/connectors.png)

### 测试数据同步

创建消费者客户端测试同步数据。连接器创建成功后会自动同步数据到 kafka 的指定主题中，主题的名称为配置中的`database.server.name`.`schemaName`.`tableName`，如这里主题的名称是 'test.test.test_slot'
```
bin/windows/kafka-console-consumer.bat --bootstrap-server localhost:9092 --topic test.test.test_slot --from-beginning
```

如果一切正常控制台会打印同步到的数据。之后在 test_slot 表中增加、修改、删除操作检查是否可以实时同步到数据。至于不同操作同步到的数据结构可以到 debezium-connector-postgres 官方文档中查看。文档中有详细介绍。
```postgresql
insert into test.test_slot (a, b, c) values ('test', 10, now());
update test.test_slot set a = 'test1' where a = 'test';
delete from test.test_slot where a ='test1';
```

## 创建 elasticsearch 连接器

### elasticsearch 连接器配置 

这里定义同步的主题是 `test.test.test_slot` 也就是 topics 参数的值，`connection` 开头的配置是 es 的连接参数，其他参数可选查询官网的说明文档
```json
{
  "name": "es-sink",
  "config": {
    "connector.class": "io.confluent.connect.elasticsearch.ElasticsearchSinkConnector",
    "tasks.max": "1",
    "connection.url": "http://localhost:9200",
    "connection.user": "",
    "connection.password": "",
    "topics": "test.test.test_slot",
    "type.name": "_doc",
	"schema.ignore": "true",
	"key.ignore": "false",
	"transforms": "renameTopic,extractId,extractAfter,createDateConverter,updateDateConverter",
	"transforms.renameTopic.type": "org.apache.kafka.connect.transforms.RegexRouter",
	"transforms.renameTopic.regex": ".*\\.(.*)",
	"transforms.renameTopic.replacement": "$1",
	"transforms.extractId.type": "org.apache.kafka.connect.transforms.ExtractField$Key",
	"transforms.extractId.field": "id",
	"transforms.extractAfter.type": "org.apache.kafka.connect.transforms.ExtractField$Value",
	"transforms.extractAfter.field": "after",
	"transforms.createDateConverter.type": "org.apache.kafka.connect.transforms.TimestampConverter$Value",
	"transforms.createDateConverter.target.type": "string",
	"transforms.createDateConverter.field": "create_date",
	"transforms.createDateConverter.format": "yyyy-MM-dd HH:mm:ss",
	"transforms.updateDateConverter.type": "org.apache.kafka.connect.transforms.TimestampConverter$Value",
	"transforms.updateDateConverter.target.type": "string",
	"transforms.updateDateConverter.field": "update_date",
	"transforms.updateDateConverter.format": "yyyy-MM-dd HH:mm:ss",
	"consumer.max.partition.fetch.bytes": "52428800"
  }
}
```

### 创建 elasticsearch 连接器

```
curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" localhost:8083/connectors/ -d @config/connect-es-sink.json
```

如一切正常在 postgres 中新增，修改，删除数据时 es 就会立即同步到变化的数据。


## 参考

文档地址 https://docs.confluent.io/current/connect/kafka-connect-elasticsearch/configuration_options.html
- [kafka 连接器官网文档](http://kafka.apache.org/documentation/#connect)
- [debezium-connector-postgres 官网文档](https://debezium.io/documentation/reference/1.1/connectors/postgresql.html)
- [kafka-connect-elasticsearch 官网地址](https://www.confluent.io/hub/confluentinc/kafka-connect-elasticsearch)