---
title: debezium+kafka实现oracle的数据同步方案
index_img: /img/default.png
banner_img: /img/default.png
date: 2024-03-14 18:09:15
tags: debezium
categories:
description:
---

debezium+kafka实现oracle的数据同步方案，类似与canal，但功能更为强大，支持的数据库类型更多

<!-- more -->

debezium最新稳定版本为2.5，当前文档采用的使用2.4版本

## 整体流程

> 说明： 方式一都是采用kafaka中配置 debezium-connector-oracle 插件然后启动插件方式,然后oracle连接器的，放在kafka的config目录中（connect-distributed.properties），这样启动好kafka 直接就配置好oracle连接器
>
> 方式二：采用的zookeeper+kafaka+connector 安装三个中间件，然后以restful方式请求创建oracle连接器
>
> 优缺点：方式一启动好久配置好参数，方式二以HTTP+JSON参数配置连接器，更为灵活

方式一：

- oracle 开启日志记录
  	查询是否开启 select name,log_mode from v$database; 
  	开启 ALTER DATABASE ARCHIVELOG;
  	补全日志 ALTER DATABASE ADD SUPPLEMENTAL LOG DATA (ALL) COLUMNS
- 下载debezium-connector-oracle-2.5.2.Final-plugin 插件 
- 配置kafka 中debezium插件地址
- 配置kafka中 **connect-distributed.properties**配置参数  包括oracle连接信息等
  	配置参数参考：https://www.cnblogs.com/Marydon20170307/p/17944940
- 先后启动zookeeper kafka
- 我们的java springboot应用监听kafka消息，做数据入库

方式二：

- oracle 开启日志记录
  	查询是否开启 select name,log_mode from v$database; 
  	开启 ALTER DATABASE ARCHIVELOG;
  	补全日志 ALTER DATABASE ADD SUPPLEMENTAL LOG DATA (ALL) COLUMNS

- 先后启动**zookeeper**、**kafka**、**kafka connect**中间件

- 最后以HTTP形式完成配置

  配置参数参考：https://www.cnblogs.com/Marydon20170307/p/17944940

- 我们的java springboot应用监听kafka消息，做数据入库

## 安装组件（下面方式二选一）

### 采用docker安装（当前测试环境的安装方式）

**zookeeper**中间件

```shell
docker run -d  --name zookeeper -p 2181:2181 -p 9011:9011 debezium/zookeeper:2.4 
```

**kafka**中间件

```shell
docker run -d --name kafka -p 9092:9092 --link zookeeper:zookeeper debezium/kafka:2.4
```

 **Kafka Connect**中间件

```shell
docker run -d --name connect -p 8083:8083  -e GROUP_ID=1 -e CONFIG_STORAGE_TOPIC=my_connect_configs -e OFFSET_STORAGE_TOPIC=my_connect_offsets -e STATUS_STORAGE_TOPIC=my_connect_statuses  -e CONNECT_MAX_REQUEST_SIZE=20000000  -e CONNECT_BUFFER_MEMORY=800000000    -e CONNECT_FETCH_MAX_BYTES=20000000  -e CONNECT_MAX_PARTITION_FETCH_BYTES=20000000 -e OFFSET_FLUSH_INTERVAL_MS=10000  -e OFFSET_FLUSH_TIMEOUT_MS=6000000 -e CONNECT_CONNECTIONS_MAX_IDLE_MS=6000000 -e  CONNECT_RECEIVE.BUFFER.BYTES=500000000 -e CONNECT_PRODUCER_MAX_REQUEST_SIZE=20000000  --link zookeeper:zookeeper --link kafka:kafka debezium/connect:2.4
```

oracle驱动拷贝到connect的目录下

```shell
docker cp ojdbc8.jar 容器ID:/kafka/libs 
# 重启connect 
docker restart connect容器ID
```

### 采用docker-compose安装

```shell
version: '3.7'

services:
  zookeeper:
    image: debezium/zookeeper:2.4
    container_name: zookeeper
    ports:
      - "2181:2181"
      - "9011:9011"

  kafka:
    image: debezium/kafka:2.4
    container_name: kafka
    ports:
      - "9092:9092"
    environment:
      ZOOKEEPER_CONNECT: zookeeper:2181

  connect:
    image: debezium/connect:2.4
    container_name: connect
    ports:
      - "8083:8083"
    environment:
      GROUP_ID: 1
      CONFIG_STORAGE_TOPIC: my_connect_configs
      OFFSET_STORAGE_TOPIC: my_connect_offsets
      STATUS_STORAGE_TOPIC: my_connect_statuses
      CONNECT_MAX_REQUEST_SIZE: 20000000
      CONNECT_BUFFER_MEMORY: 800000000
      CONNECT_FETCH_MAX_BYTES: 20000000
      CONNECT_MAX_PARTITION_FETCH_BYTES: 20000000
      OFFSET_FLUSH_INTERVAL_MS: 10000
      OFFSET_FLUSH_TIMEOUT_MS: 6000000
      CONNECT_CONNECTIONS_MAX_IDLE_MS: 6000000
      CONNECT_RECEIVE_BUFFER_BYTES: 500000000
      CONNECT_PRODUCER_MAX_REQUEST_SIZE: 20000000
    links:
      - "zookeeper:zookeeper"
      - "kafka:kafka"
    volumes:
      - ./connect/ojdbc8.jar:/kafka/libs/ojdbc8.jar
```

## Oracle连接器配置

> 我们这里采用以HTTP形式配置连接器，主要操作的就是对kafka-connect这个中间件操作
>

首先拿到connect连接器能够使用的暴露出来的ip+端口。公网也好内网也行，内网127.0.0.1为例

可以使用api工具 如apifox、postman

```shell
#查看所有的connector进程
GET http://127.0.0.1:8083/connectors

#新建connector连接器配置(需要携带配置器json参数)
POST http://127.0.0.1:8083/connectors

#删除connector连接器,{name}是新建时候name名字
DELETE http://127.0.0.1:8083/connectors/{name}

#查看connector连接器状态,{name}是新建时候name名字
GET http://127.0.0.1:8083/connectors/{name}/status

#目前没有从官网找到更新的接口，目前都是先删除再新增的方式实现修改连接器配置

```

####  参数说明

> 官方文档地址：https://debezium.io/documentation/reference/2.4/connectors/oracle.html

结构如下：

```json
{
    "name": "debezium-oracle",  
    "config": {
        "connector.class" : "io.debezium.connector.oracle.OracleConnector",
        "tasks.max" : "1",
         "topic.prefix" : "test", 
        	   "database.url": "jdbc:oracle:thin:@(DESCRIPTION=(ADDRESS_LIST=(LOAD_BALANCE=OFF)(FAILOVER=ON)(ADDRESS=(PROTOCOL=TCP)(HOST=<oracle ip 1>)(PORT=1521))(ADDRESS=(PROTOCOL=TCP)(HOST=<oracle ip 2>)(PORT=1521)))(CONNECT_DATA=SERVICE_NAME=)(SERVER=DEDICATED)))",
        "database.user" : "inventory",
        "database.password" : "123456",
        "database.dbname" : "xe",
        "schema.history.internal.kafka.bootstrap.servers" : "10.168.1.163:9092",
        "schema.history.internal.kafka.topic": "schema-changes.inventory"
    }
}
```

列举一些核心比较重要的参数

| 参数名                 | 说明                                                         |
| ---------------------- | ------------------------------------------------------------ |
| name属性               | 连接器名称，随意但必须唯一                                   |
| database.xxx           | 数据库连接参数，见名知意，不多解释                           |
| topic.prefix           | kafaka的topic前缀，重要，后续采集的数据推送到kafaka的时候，topic的命名就是**<u>prefix前缀+库名+表名</u>**，非常重要，java程序中监听此topic做后续业务 |
| table.include.list     | 要捕获的变更记录的表，多个逗号分割 格式是**库名.表名**       |
| log.mining.strategy    | redo_log_catalog(默认) 归档日志，<br />redolog写满才会生成归档日志，导致topic接收数据慢（DDL+DML<br />online_catalog 在线日志，即时读取日志（不包含DDL，只包含DML） |
| skipped.operations     | 默认值：t。不需要监控的操作，可选值：c(insert/create）,u（update）,d （delete）,t (truncate）,none。 |
| snapshot.mode          | 认值: initial, 可选值: [initial,initial_onlywhen_needed,never,schema_onlyschema_only_recovery] <br />initial(默认）（初始全量，后续增量）：连接器执行数据库的初始一致性快照，快照完成后，连接器开始为后续数据库更改流式传输事件记录。<br />initia_only（只全量，不增量）：连接器只执行数据库的初始一致性快照，不允许捕获任何后续更改的事件。<br />schema_only（只增量，不全量）：连接器只捕获所有相关表的表结构，不捕获初始数据，但是会同步后续数据库的更改记录。 |
| decimal.handlling.mode | 默认值: precise，可选值: [precise,double,string]  <br />说明：<br />如果你使用的不是debezium-connector-jdbc插件来接收数据，需将值设为：string，只有这样才能解决number类型被转成base64码的问题。<br />示例 <br />当decimal.handling.mode属性值为precise时："ZS_ID":("scale":0,"value":"Aw==};<br />当decimal.handling.mode属性值为string时："ZS_ID":"3"。 |
