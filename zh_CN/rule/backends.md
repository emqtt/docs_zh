## 保存数据到 MySQL

搭建 MySQL 数据库，并设置用户名密码为 root/public，以 MacOS X 为例:

```bash
$ brew install mysql

$ brew services start mysql

$ mysql -u root -h localhost -p

ALTER USER 'root'@'localhost' IDENTIFIED BY 'public';
```

初始化 MySQL 表:

```bash
$ mysql -u root -h localhost -ppublic
```

创建 “test” 数据库:
```bash
CREATE DATABASE test;
```
创建 t_mqtt_msg 表:

```sql
USE test;
CREATE TABLE `t_mqtt_msg` (
`id` int(11) unsigned NOT NULL AUTO_INCREMENT,
`msgid` varchar(64) DEFAULT NULL,
`topic` varchar(255) NOT NULL,
`qos` tinyint(1) NOT NULL DEFAULT '0',
`payload` blob,
`arrived` datetime NOT NULL,
PRIMARY KEY (`id`),
INDEX topic_index(`id`, `topic`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8MB4;
```

![image](./assets/rule-engine/mysql_init_1@2x.png)

创建规则:

打开 [EMQ X Dashboard](http://127.0.0.1:18083/#/rules)，选择左侧的 “规则” 选项卡。

填写规则 SQL:

```sql
SELECT * FROM "t/#"
```

![image](./assets/rule-engine/rule_sql.png)

关联动作:

在 “响应动作” 界面选择 “添加”，然后在 “动作” 下拉框里选择 “保存数据到 MySQL”。

![image](./assets/rule-engine/rule_action_1@2x.png)

填写动作参数:

“保存数据到 MySQL” 动作需要两个参数：

1). SQL 模板。这个例子里我们向 MySQL 插入一条数据，SQL
​    模板为:

```sql
insert into t_mqtt_msg(msgid, topic, qos, payload, arrived) values (${id}, ${topic}, ${qos}, ${payload}, FROM_UNIXTIME(${timestamp}/1000))
```

![image](./assets/rule-engine/rule_action_2@2x.png)

2). 关联资源的 ID。现在资源下拉框为空，可以点击右上角的 “新建资源” 来创建一个 MySQL 资源:

填写资源配置:

数据库名填写 “mqtt”，用户名填写 “root”，密码填写 “123456”

![image](./assets/rule-engine/rule_action_3@2x.png)

点击 “新建” 按钮。

返回响应动作界面，点击 “确认”。

![image](./assets/rule-engine/rule_action_4@2x.png)

返回规则创建界面，点击 “创建”。

![image](./assets/rule-engine/rule_overview_1@2x.png)

在规则列表里，点击 “查看” 按钮或规则 ID 连接，可以预览刚才创建的规则:

![image](./assets/rule-engine/rule_overview_2@2x.png)

规则已经创建完成，现在发一条数据:

```bash
Topic: "t/a"
QoS: 1
Payload: "hello"
```

然后检查 MySQL 表，新的 record 是否添加成功:

![image](./assets/rule-engine/mysql_result_1@2x.png)

## 保存数据到 PostgreSQL

搭建 PostgreSQL 数据库，以 MacOS X 为例:

```bash
$ brew install postgresql
$ brew services start postgresql
```

```bash
## 使用用户名 root 创建名为 'mqtt' 的数据库
$ createdb -U root mqtt

$ psql -U root mqtt

mqtt=> \dn;
List of schemas
Name  | Owner
--------+-------
public | shawn
(1 row)
```

初始化 PgSQL 表:

```bash
$ psql -U root mqtt
```

创建 `t_mqtt_msg` 表:

```sql
CREATE TABLE t_mqtt_msg (
    id SERIAL primary key,
    msgid character varying(64),
    sender character varying(64),
    topic character varying(255),
    qos integer,
    payload text,
    arrived timestamp without time zone
);
```

创建规则:

打开 [EMQ X Dashboard](http://127.0.0.1:18083/#/rules)，选择左侧的 “规则” 选项卡。

填写规则 SQL:

```bash
SELECT * FROM "t/#"
```

![image](./assets/rule-engine/rule_sql.png)

关联动作:

在 “响应动作” 界面选择 “添加”，然后在 “动作” 下拉框里选择 “保存数据到 PostgreSQL”。

![image](./assets/rule-engine/pgsql-action-0@2x.png)

填写动作参数:

“保存数据到 PostgreSQL” 动作需要两个参数：

1). 关联资源的 ID。现在资源下拉框为空，可以点击右上角的 “新建资源” 来创建一个 PostgreSQL 资源:

![image](./assets/rule-engine/pgsql-resource-0@2x.png)

选择 “PostgreSQL 资源”。

填写资源配置:

数据库名填写 “mqtt”，用户名填写 “root”，其他配置保持默认值，然后点击 “测试连接” 按钮，确保连接测试成功。

最后点击 “新建” 按钮。

![image](./assets/rule-engine/pgsql-resource-1@2x.png)

返回响应动作界面，点击 “确认”。

2).SQL 模板。这个例子里我们向 PostgreSQL 插入一条数据，SQL
​    模板为:

```bash
insert into t_mqtt_msg(msgid, topic, qos, payload, arrived) values (${id}, ${topic}, ${qos}, ${payload}, to_timestamp(${timestamp}::double precision /1000))
```

插入数据之前，SQL 模板里的 ${key} 占位符会被替换为相应的值。

![image](./assets/rule-engine/pgsql-action-2@2x.png)

返回规则创建界面，点击 “创建”。

![image](./assets/rule-engine/pgsql-rulesql-2@2x.png)

规则已经创建完成，现在发一条数据:

```bash
Topic: "t/1"
QoS: 0
Payload: "hello1"
```

然后检查 PostgreSQL 表，新的 record 是否添加成功:

![image](./assets/rule-engine/pgsql-result-1@2x.png)

在规则列表里，可以看到刚才创建的规则的命中次数已经增加了 1:

![image](./assets/rule-engine/pgsql-rulelist-1@2x.png)

## 保存数据到 MongoDB

搭建 MongoDB 数据库，并设置用户名密码为 root/public，以 MacOS X 为例:

```bash
$ brew install mongodb
$ brew services start mongodb

## 新增 root/public 用户
$ use mqtt;
$ db.createUser({user: "root", pwd: "public", roles: [{role: "readWrite", db: "mqtt"}]});

## 修改配置，关闭匿名认证
$ vim /usr/local/etc/mongod.conf

    security:
    authorization: enabled

$ brew services restart mongodb
```

初始化 MongoDB 表:

```bash
$ mongo 127.0.0.1/mqtt -uroot -ppublic
db.createCollection("t_mqtt_msg");
```

创建规则:

打开 [EMQ X Dashboard](http://127.0.0.1:18083/#/rules)，选择左侧的 “规则” 选项卡。

填写规则 SQL:

```bash
SELECT id as msgid, topic, qos, payload, publish_received_at as arrived FROM "t/#"
```

![image](./assets/rule-engine/mongodb_data_to_store1.png)

关联动作:

在 “响应动作” 界面选择 “添加”，然后在 “动作” 下拉框里选择 “保存数据到 MongoDB”。

![image](./assets/rule-engine/mongo-action-0@2x.png)

填写动作参数:

“保存数据到 MongoDB” 动作需要三个参数：

1). 关联资源的 ID。初始状况下，资源下拉框为空，现点击右上角的 “新建资源” 来创建一个 MongoDB 单节点 资源。

![image](./assets/rule-engine/mongodb_data_to_store2.png)

填写资源配置:

数据库名称 填写 “mqtt”，用户名填写 “root”，密码填写 “public”，连接认证源填写 “mqtt”
其他配置保持默认值，然后点击 “测试连接” 按钮，确保连接测试成功。

![image](./assets/rule-engine/mongo-resoure-1.png)

点击 “新建” 按钮，完成资源的创建。

2). Collection 名称。这个例子我们向刚刚新建的 collection 插入数据，填 “t_mqtt\_msg”

3). Payload Tmpl 模板。这个例子里我们向 MongoDB 插入一条数据，模板为空, 插入的数据是上面SQL语句select出来的结果用json格式写入到MongoDB中


![](./assets/rule-engine/mongodb_data_to_store3.png)

在点击 “新建” 完成规则创建

![image](./assets/rule-engine/mongodb_data_to_store4.png)

现在发送一条数据，测试该规则:

```bash
Topic: "t/mongo"
QoS: 1
Payload: "hello"
```

然后检查 MongoDB 表，可以看到该消息已成功保存:

![image](./assets/rule-engine/mongo-rule-result@2x.png)

在规则列表里，可以看到刚才创建的规则的命中次数已经增加了 1:

![image](./assets/rule-engine/mongo-rule-result@3x.png)

## 保存数据到 Redis

搭建 Redis 环境，以 MacOS X 为例:

```bash
 $ wget http://download.redis.io/releases/redis-4.0.14.tar.gz
$ tar xzf redis-4.0.14.tar.gz
$ cd redis-4.0.14
$ make && make install

# 启动 redis
$ redis-server
```

创建规则:

打开 [EMQ X Dashboard](http://127.0.0.1:18083/#/rules)，选择左侧的 “规则” 选项卡。

填写规则 SQL:

```bash
SELECT * FROM "t/#"
```

![image](./assets/rule-engine/rule_sql.png)

关联动作:

在 “响应动作” 界面选择 “添加”，然后在 “动作” 下拉框里选择 “保存数据到 Redis”。

![image](./assets/rule-engine/redis-action-0@2x.png)

填写动作参数:

“保存数据到 Redis 动作需要两个参数：

1). 关联资源。现在资源下拉框为空，可以点击右上角的 “新建资源” 来创建一个 Redis 资源:

![image](./assets/rule-engine/redis-resource-0@2x.png)

选择 Redis 单节点模式资源”。

填写资源配置:

   填写真实的 Redis 服务器地址，其他配置保持默认值，然后点击 “测试连接” 按钮，确保连接测试成功。

最后点击 “新建” 按钮。

![image](./assets/rule-engine/redis-resource-2@2x.png)

返回响应动作界面，点击 “确认”。

2). Redis 的命令:

```bash
HMSET mqtt:msg:${id} id ${id} from ${client_id} qos ${qos} topic ${topic} payload ${payload} ts ${timestamp}
```

![image](./assets/rule-engine/redis-action-1@2x.png)

返回规则创建界面，点击 “新建”。

![image](./assets/rule-engine/redis-rulesql-1@2x.png)

规则已经创建完成，现在发一条数据:

```bash
Topic: "t/1"

QoS: 0

Payload: "hello"
```

然后通过 Redis 命令去查看消息是否生产成功:

```bash
$ redis-cli

KEYS mqtt:msg\*

hgetall Key
```

![image](./assets/rule-engine/redis-cli.png)

在规则列表里，可以看到刚才创建的规则的命中次数已经增加了 1:

![image](./assets/rule-engine/redis-rulelist-0@2x.png)

## 保存数据到 Cassandra

搭建 Cassandra 数据库，并设置用户名密码为 root/public，以 MacOS X 为例:

```bash
$ brew install cassandra
## 修改配置，关闭匿名认证
$  vim /usr/local/etc/cassandra/cassandra.yaml

    authenticator: PasswordAuthenticator
    authorizer: CassandraAuthorizer

$ brew services start cassandra

## 创建 root 用户
$ cqlsh -ucassandra -pcassandra

create user root with password 'public' superuser;
```

初始化 Cassandra 表:

```bash
$ cqlsh -uroot -ppublic
```

创建 "test" 表空间:

```bash
CREATE KEYSPACE test WITH replication = {'class': 'SimpleStrategy', 'replication_factor': '1'}  AND durable_writes = true;
```

创建 “t_mqtt_msg” 表:

```sql
USE test;
CREATE TABLE t_mqtt_msg (
    msgid text,
    topic text,
    qos int,
    payload text,
    arrived timestamp,
    PRIMARY KEY (msgid, topic)
);
```

创建规则:

打开 [EMQ X Dashboard](http://127.0.0.1:18083/#/rules)，选择左侧的 “规则” 选项卡。

填写规则 SQL:
```bash
SELECT * FROM "t/#"
```

![image](./assets/rule-engine/rule_sql.png)

关联动作:

在 “响应动作” 界面选择 “添加”，然后在 “动作” 下拉框里选择 “保存数据到 Cassandra”。

![image](./assets/rule-engine/cass-action-0@2x.png)

填写动作参数:

“保存数据到 Cassandra” 动作需要两个参数：

1). 关联资源的 ID。初始状况下，资源下拉框为空，现点击右上角的 “新建资源” 来创建一个 Cassandra 资源。

![image](./assets/rule-engine/cass-resoure-0.png)

填写资源配置:

Keysapce 填写 “test”，用户名填写 “root”，密码填写 “public” 其他配置保持默认值，然后点击 “测试连接”
按钮，确保连接测试成功。

![image](./assets/rule-engine/cass-resoure-1.png)

点击 “新建” 按钮，完成资源的创建。

2). SQL 模板。这个例子里我们向 Cassandra 插入一条数据，SQL
​    模板为:

```sql
insert into t_mqtt_msg(msgid, topic, qos, payload, arrived) values (${id}, ${topic}, ${qos}, ${payload}, ${timestamp})
```

插入数据之前，SQL 模板里的 ${key} 占位符会被替换为相应的值。

![image](./assets/rule-engine/cass-resoure-2.png)

在点击 “新建” 完成规则创建

![image](./assets/rule-engine/cass-rule-overview.png)

现在发送一条数据，测试该规则:

```bash
Topic: "t/cass"
QoS: 1
Payload: "hello"
```

然后检查 Cassandra 表，可以看到该消息已成功保存:

![image](./assets/rule-engine/cass-rule-result@2x.png)

在规则列表里，可以看到刚才创建的规则的命中次数已经增加了 1:

![image](./assets/rule-engine/cass-rule-result@3x.png)

## 保存数据到 DynamoDB

搭建 DynamoDB 数据库，以 MacOS X 为例:

```bash
$ brew install dynamodb-local
$ dynamodb-local
```

创建 DynamoDB 表定义文件 mqtt\_msg.json :

<!-- end list -->

```json
{
   "TableName": "mqtt_msg",
   "KeySchema": [
       { "AttributeName": "msgid", "KeyType": "HASH" }
   ],
   "AttributeDefinitions": [
       { "AttributeName": "msgid", "AttributeType": "S" }
   ],
   "ProvisionedThroughput": {
       "ReadCapacityUnits": 5,
       "WriteCapacityUnits": 5
   }
}
```

初始化 DynamoDB 表:

```bash
$ aws dynamodb create-table --cli-input-json file://mqtt_msg.json --endpoint-url http://localhost:8000
```

创建规则:

打开 [EMQ X Dashboard](http://127.0.0.1:18083/#/rules)，选择左侧的 “规则” 选项卡。

填写规则 SQL:

```sql
SELECT id, topic, payload FROM "#"
```

![image](./assets/rule-engine/dynamo-rulesql-0.png)

关联动作:

在 “响应动作” 界面选择 “添加”，然后在 “动作” 下拉框里选择 “保存数据到 DynamoDB”。

![image](./assets/rule-engine/dynamo-action-0.png)

填写动作参数:

“保存数据到 DynamoDB” 动作需要两个参数：

1). DynamoDB 表名。这个例子里我们设置的表名为 "mqtt\_msg"

2). DynamoDB Hash Key。这个例子里我们设置的 Hash Key 要与表定义的一致

3). DynamoDB Range Key。由于我们表定义里没有设置 Range Key。这个例子里我们把 Range Key 设置为空。

![image](./assets/rule-engine/dynamo-action-1.png)

4). 关联资源的 ID。现在资源下拉框为空，可以点击右上角的 “新建资源” 来创建一个 DynamoDB 资源:

填写资源配置:

![image](./assets/rule-engine/dynamo-resource-1.png)

点击 “新建” 按钮。

返回响应动作界面，点击 “确认”。

![image](./assets/rule-engine/dynamo-action-2.png)

返回规则创建界面，点击 “新建”。

![image](./assets/rule-engine/dynamo-rulesql-1.png)

规则已经创建完成，现在发一条数据:

```bash
Topic: "t/a"
QoS: 1
Payload: "hello"
```

然后检查 DynamoDB 的 mqtt\_msg 表，新的 record 是否添加成功:

![image](./assets/rule-engine/dynamo-result-0.png)

在规则列表里，可以看到刚才创建的规则的命中次数已经增加了 1:

![image](./assets/rule-engine/dynamo-result-1.png)


## 保存数据到 ClickHouse

搭建 ClickHouse 数据库，并设置用户名密码为 default/public，以 CentOS 为例:

```bash
## 安装依赖
sudo yum install -y epel-release

## 下载并运行packagecloud.io提供的安装shell脚本
curl -s https://packagecloud.io/install/repositories/altinity/clickhouse/script.rpm.sh | sudo bash

## 安装ClickHouse服务器和客户端
sudo yum install -y clickhouse-server clickhouse-client

## 启动ClickHouse服务器
clickhouse-server

## 启动ClickHouse客户端程序
clickhouse-client
```

创建 “test” 数据库:
```bash
create database test;
```
创建 t_mqtt_msg 表:

```sql
use test;
create table t_mqtt_msg (msgid Nullable(String), topic Nullable(String), clientid Nullable(String), payload Nullable(String)) engine = Log;
```

![](./assets/rule-engine/clickhouse_0.png)

创建规则:

打开 [EMQ X Dashboard](http://127.0.0.1:18083/#/rules)，选择左侧的 “规则” 选项卡。

填写规则 SQL:

```sql
SELECT * FROM "#"
```

![image](./assets/rule-engine/clickhouse_1.png)

关联动作:

在 “响应动作” 界面选择 “添加”，然后在 “动作” 下拉框里选择 “保存数据到 ClickHouse”。

![image](./assets/rule-engine/clickhouse_2.png)

填写动作参数:

“保存数据到 ClickHouse” 动作需要两个参数：

1). 关联资源的 ID。现在资源下拉框为空，可以点击右上角的 “新建资源” 来创建一个 ClickHouse 资源:

![image](./assets/rule-engine/clickhouse_3.png)

选择 “ClickHouse 资源”。

填写资源配置:

![image](./assets/rule-engine/clickhouse_4.png)

点击 “新建” 按钮。

2). SQL 模板。这个例子里我们向 ClickHouse 插入一条数据，SQL
​    模板为:

```sql
insert into test.t_mqtt_msg(msgid, clientid, topic, payload) values ('${id}', '${clientid}', '${topic}', '${payload}')
```

![image](./assets/rule-engine/clickhouse_5.png)

返回响应动作界面，点击 “确认”。

![image](./assets/rule-engine/clickhouse_6.png)

在规则列表里，点击 “查看” 按钮或规则 ID 连接，可以预览刚才创建的规则:

![image](./assets/rule-engine/clickhouse_7.png)

规则已经创建完成，现在发一条数据:

```bash
Topic: "t/a"
QoS: 1
Payload: "hello"
```

然后检查 ClickHouse 表，新的 record 是否添加成功:

![image](./assets/rule-engine/clickhouse_8.png)


## 保存数据到 OpenTSDB

搭建 OpenTSDB 数据库环境，以 MacOS X 为例:

```bash
$ docker pull petergrace/opentsdb-docker

$ docker run -d --name opentsdb -p 4242:4242 petergrace/opentsdb-docker
```

创建规则:

打开 [EMQ X Dashboard](http://127.0.0.1:18083/#/rules)，选择左侧的 “规则” 选项卡。

填写规则 SQL:

```sql
SELECT
    payload as p,
    p.metric as metric, p.tags as tags, p.value as value
FROM
    "#"
```

![image](./assets/rule-engine/opentsdb-rulesql-0@2x.png)

关联动作:

在 “响应动作” 界面选择 “添加”，然后在 “动作” 下拉框里选择 “保存数据到 OpenTSDB”。

![image](./assets/rule-engine/opentsdb-action-0@2x.png)

填写动作参数:

“保存数据到 OpenTSDB” 动作需要六个参数:

1). 详细信息。是否需要 OpenTSDB Server 返回存储失败的 data point 及其原因的列表，默认为 false。

2). 摘要信息。是否需要 OpenTSDB Server 返回 data point 存储成功与失败的数量，默认为 true。

3). 最大批处理数量。消息请求频繁时允许 OpenTSDB 驱动将多少个 Data Points 合并为一次请求，默认为 20。

4). 是否同步调用。指定 OpenTSDB Server 是否等待所有数据都被写入后才返回结果，默认为 false。

5). 同步调用超时时间。同步调用最大等待时间，默认为 0。

6). 关联资源。现在资源下拉框为空，可以点击右上角的 “新建资源” 来创建一个 OpenTSDB 资源:

![image](./assets/rule-engine/opentsdb-action-1@2x.png)

选择 “OpenTSDB 资源”:

填写资源配置:

本示例中所有配置保持默认值即可，点击 “测试连接” 按钮，确保连接测试成功。

最后点击 “新建” 按钮。

![image](./assets/rule-engine/opentsdb-resource-1@2x.png)

返回响应动作界面，点击 “确认”。

![image](./assets/rule-engine/opentsdb-action-2@2x.png)

返回规则创建界面，点击 “新建”。

![image](./assets/rule-engine/opentsdb-rulesql-1@2x.png)

规则已经创建完成，现在发一条消息:

```bash
Topic: "t/1"

QoS: 0

Payload: "{"metric":"cpu","tags":{"host":"serverA"},"value":12}"
```

我们通过 Postman 或者 curl 命令，向 OpenTSDB Server 发送以下请求:

```bash
POST /api/query HTTP/1.1
Host: 127.0.0.1:4242
Content-Type: application/json
cache-control: no-cache
Postman-Token: 69af0565-27f8-41e5-b0cd-d7c7f5b7a037
{
    "start": 1560409825000,
    "queries": [
        {
            "aggregator": "last",
            "metric": "cpu",
            "tags": {
                "host": "*"
            }
        }
    ],
    "showTSUIDs": "true",
    "showQuery": "true",
    "delete": "false"
}
------WebKitFormBoundary7MA4YWxkTrZu0gW--
```

如果 data point 存储成功，将会得到以下应答:

```json
[
  {
      "metric": "cpu",
      "tags": {
          "host": "serverA"
      },
      "aggregateTags": [],
      "query": {
          "aggregator": "last",
          "metric": "cpu",
          "tsuids": null,
          "downsample": null,
          "rate": false,
          "filters": [
              {
                  "tagk": "host",
                  "filter": "*",
                  "group_by": true,
                  "type": "wildcard"
              }
          ],
          "index": 0,
          "tags": {
              "host": "wildcard(*)"
          },
          "rateOptions": null,
          "filterTagKs": [
              "AAAC"
          ],
          "explicitTags": false
      },
      "tsuids": [
          "000002000002000007"
      ],
      "dps": {
          "1561532453": 12
      }
  }
]
```

在规则列表里，可以看到刚才创建的规则的命中次数已经增加了 1:

![image](./assets/rule-engine/opentsdb-rulelist-1@2x.png)


## 保存数据到 InfluxDB

启动 InfluxDB，请确保启用了相应的 Listener（我们假设您已经成功安装了 InfluxDB 环境）

```bash
$ influxd -config /usr/local/etc/influxdb.conf
```

创建规则:

打开 [EMQ X Dashboard](http://127.0.0.1:18083/#/rules)，选择左侧的 “规则” 选项卡。

填写规则 SQL:

```bash
SELECT
    payload.host as host,
    payload.location as location,
    payload.internal as internal,
    payload.external as external,
    *
FROM
    "#"
```

![image](./assets/rule-engine/influxdb-rulesql-0@2x.png)

关联动作:

在 “Action” 界面选择 “Add action”，然后在 “Action Type” 下拉框里选择 “Data to InfluxDB”。

![image](./assets/rule-engine/influxdb-action-0@2x.png)

填写动作参数:

“Data to InfluxDB” 动作有以下参数：

![image](./assets/rule-engine/influxdb-action-1@2x.png)

1). Measurement。指定写入到 InfluxDB 的 data point 的 Measurement，支持固定字符串和占位符两种设置方式。

2). Fields。指定写入到 InfluxDB 的 data point 的 Fields 的 Key 和 Value，支持固定字符串和占位符两种设置方式。

3). Tags。指定写入到 InfluxDB 的 data point 的 Tags 的 Key 和 Value，支持固定字符串和占位符两种设置方式。

4). Timestamp Key。指定写入到 InfluxDB 的 data point 的 Timestamp 的值从哪里获取。

5). Use of reources。指定动作关联的资源，现在资源下拉框为空，可以点击右上角的 “新建资源” 来创建一个 InfluxDB 资源，目前支持 HTTP/HTTPS 和 UDP 两种资源。

![image](./assets/rule-engine/influxdb-resource-0@2x.png)

填写资源配置:

InfluxDB HTTP 资源包括以下配置：

   ![image](./assets/rule-engine/influxdb-resource-1@2x.png)

   1). Resource Name。资源名称，支持以易读的形式唯一标识资源。

   2). InfluxDB Host。InfluxDB HTTP 主机地址。

   3). InfluxDB Port。InfluxDB HTTP 主机端口。

   4). InfluxDB Database。InfluxDB 数据库名。

   5). InfluxDB Username。InfluxDB 用户名。

   6). InfluxDB Password。InfluxDB 密码。

   7). Precision of timestamp。时间戳精度。

   8). Batch Size。单次写入能够收集的最大数据点数量，用于提升高并发时的性能。

   9). Pool Size。InfluxDB 写进程池大小，合适的进程池大小可以在一定程度上提升写入性能。

InfluxDB UDP 资源包括以下配置：

   ![image](./assets/rule-engine/influxdb-resource-2@2x.png)

   1). Resource Name。资源名称，支持以易读的形式唯一标识资源。

   2). InfluxDB Host。InfluxDB UDP 主机地址。

   3). InfluxDB Port。InfluxDB UDP 主机端口。

   8). Batch Size。单次写入能够收集的最大数据点数量，用于提升高并发时的性能。

   9). Pool Size。InfluxDB 写进程池大小，合适的进程池大小可以在一定程度上提升写入性能。

本示例中所有配置保持默认值即可，点击 “测试连接” 按钮，确保连接测试成功。

最后点击 “Confirm” 按钮。

返回响应动作界面，选择刚刚创建的 InfluxDB 资源，填写其余配置后点击 “Confirm” 按钮。

最后返回规则创建界面，点击页面底部的 “Create” 按钮，完成规则创建。 

![image](./assets/rule-engine/influxdb-rulelist-0@2x.png)

规则已经创建完成，现在发一条消息:

```bash
Topic: "t/1"

QoS: 0

Payload:
"{"host":"serverA","location":"roomA","internal":25,"external":37}"
```

然后检查 InfluxDB，新的 data point 是否添加成功:

```bash
$ influx -precision rfc3339

> use db
Using database db
> select * from "temperature"
name: temperature
time                external  from            host    internal location
----                --------  ----            ----    -------- --------
1561535778444457348 37        mqttjs_46355e19 serverA 25       roomA
```

在规则列表里，可以看到刚才创建的规则的命中次数已经增加了 1:

![image](./assets/rule-engine/influxdb-rulelist-1@2x.png)

## 保存数据到 TimescaleDB

搭建 TimescaleDB 数据库环境，以 MacOS X 为例:

```bash
$ docker pull timescale/timescaledb

$ docker run -d --name timescaledb -p 5432:5432 -e POSTGRES_PASSWORD=password timescale/timescaledb:latest-pg11

$ docker exec -it timescaledb psql -U postgres

## 创建并连接 tutorial 数据库
CREATE database tutorial;

\c tutorial

CREATE EXTENSION IF NOT EXISTS timescaledb CASCADE;
```

初始化 TimescaleDB 表:

```bash
$ docker exec -it timescaledb psql -U postgres -d tutorial
```

创建 `conditions` 表:

```sql
CREATE TABLE conditions (
    time        TIMESTAMPTZ       NOT NULL,
    location    TEXT              NOT NULL,
    temperature DOUBLE PRECISION  NULL,
    humidity    DOUBLE PRECISION  NULL
);

SELECT create_hypertable('conditions', 'time');
```

创建规则:

打开 [EMQ X Dashboard](http://127.0.0.1:18083/#/rules)，选择左侧的 “规则” 选项卡。

填写规则 SQL:

```sql
SELECT
    payload as p,
    p.temp as temp,
    p.humidity as humidity,
    p.location as location
FROM
    "#"
```

![image](./assets/rule-engine/timescaledb-rulesql-0@2x.png)

关联动作:

在 “响应动作” 界面选择 “添加”，然后在 “动作” 下拉框里选择 “保存数据到 TimescaleDB”。

![image](./assets/rule-engine/timescaledb-action-0@2x.png)

填写动作参数:

“保存数据到 TimescaleDB” 动作需要两个参数：

1). SQL 模板。这个例子里我们向 TimescaleDB 插入一条数据，SQL
​    模板为:

```sql
insert into conditions(time, location, temperature, humidity) values (NOW(), ${location}, ${temp}, ${humidity})
```

插入数据之前，SQL 模板里的 ${key} 占位符会被替换为相应的值。

2). 关联资源。现在资源下拉框为空，可以点击右上角的 “新建资源” 来创建一个 TimescaleDB 资源:

![image](./assets/rule-engine/timescaledb-resource-0@2x.png)

选择 “PostgreSQL 资源”。

填写资源配置:

数据库名填写 “mqtt”，用户名填写 “root”，其他配置保持默认值，然后点击 “测试连接” 按钮，确保连接测试成功。

最后点击 “新建” 按钮。

![image](./assets/rule-engine/pgsql-resource-1@2x.png)

返回响应动作界面，点击 “确认”。

![image](./assets/rule-engine/timescaledb-action-1@2x.png)

返回规则创建界面，点击 “新建”。

![image](./assets/rule-engine/timescaledb-rulesql-1@2x.png)

规则已经创建完成，现在发一条数据:

```bash
Topic: "t/1"

QoS: 0

Payload: "{"temp":24,"humidity":30,"location":"hangzhou"}"
```

然后检查 TimescaleDB 表，新的 record 是否添加成功:

```bash
tutorial=# SELECT * FROM conditions LIMIT 100;
            time              | location | temperature | humidity
-------------------------------+----------+-------------+----------
2019-06-27 01:41:08.752103+00 | hangzhou |          24 |       30
```

在规则列表里，可以看到刚才创建的规则的命中次数已经增加了 1:

![image](./assets/rule-engine/timescaledb-rulelist-0@2x.png)


## 保存数据到 TDengine

[TDengine](https://github.com/taosdata/TDengine) 是[涛思数据](https://www.taosdata.com/cn/)推出的一款开源的专为物联网、车联网、工业互联网、IT 运维等设计和优化的大数据平台。除核心的快 10 倍以上的时序数据库功能外，还提供缓存、数据订阅、流式计算等功能，最大程度减少研发和运维的复杂度。

EMQ X 支持通过 **发送到 Web 服务** 的方式保存数据到 TDengine，也在企业版上提供原生的 TDengine 驱动实现直接保存。

使用 Docker 安装 TDengine 或在 [Cloud](https://marketplace.huaweicloud.com/product/OFFI454488918838128640) 上部署：

```bash
docker run --name TDengine -d -p 6030:6030 -p 6035:6035 -p 6041:6041 -p 6030-6040:6030-6040/udp TDengine/TDengine 
```

进入 Docker 容器：

```bash
docker exec -it TDengine bash
taos
```

创建 “test” 数据库:
```bash
create database test;
```
创建 t_mqtt_msg 表，关于 TDengine 数据结构以及 SQL 命令参见 [TAOS SQL](https://www.taosdata.com/cn/documentation/taos-sql/#表管理)：

```sql
use test;
CREATE TABLE t_mqtt_msg (
  ts timestamp,
 	msgid NCHAR(64),
  topic NCHAR(255),
  qos TINYINT,
  payload BINARY(1024),
  arrived timestamp
);
```

![image-20200729163951206](./assets/rule-engine/image-20200729163951206.png)


创建规则:

打开 [EMQ X Dashboard](http://127.0.0.1:18083/#/rules)，选择左侧的 “规则” 选项卡。

填写规则 SQL:

```sql
SELECT * FROM "t/#"
```

![image](./assets/rule-engine/rule_sql.png)

后续动作创建操作可以根据你的 EMQ X 版本灵活选择。

### 原生方式（企业版）

关联动作:

在 “响应动作” 界面选择 “添加”，然后在 “动作” 下拉框里选择 “保存数据到 TDengine”。

> 仅限企业版 4.1.1 及以后版本。

填写动作参数:

“保存数据到 TDengine” 动作需要两个参数：

1). SQL 模板。这个例子里我们向 TDengine 插入一条数据，注意我们应当在 SQL 中指定数据库名，字符类型也要用单引号括起来，SQL 模板为：

```sql
insert into test.t_mqtt_msg(ts, msgid, topic, qos, payload) values (now, '${id}', '${topic}', ${qos}, '${payload}')
```

![image-20200729164158454](./assets/rule-engine/image-20200729164158454.png)

2). 关联资源的 ID。现在资源下拉框为空，可以点击右上角的 “新建资源” 来创建一个 TDengine资源:

填写资源配置:

用户名填写 “root”，密码填写缺省密码 “taosdata”，**TDengine 不在资源中配置数据库名，请在 SQL 中自行配置。**

![image-20200729165651951](./assets/rule-engine/image-20200729165651951.png)

点击 “新建” 按钮。

返回响应动作界面，点击 “确认”。

![image-20200729174211581](./assets/rule-engine/image-20200729174211581.png)

返回规则创建界面，点击 “创建”。


### 通过发送数据到 Web 服务写入

为支持各种不同类型平台的开发，TDengine 提供符合 REST 设计标准的 API。通过 [RESTful Connector](https://www.taosdata.com/cn/documentation/connector/#RESTful-Connector) 提供了最简单的连接方式，即使用 HTTP 请求携带认证信息与要执行的 SQL 操作 TDengine。

EMQ X 规则引擎中有功能强大的**发送数据到 Web 服务功能**，可以实现无缝实现上述操作。


关联动作:

在 “响应动作” 界面选择 “添加”，然后在 “动作” 下拉框里选择 “保存数据到 Web 服务“。

EMQ X 规则引擎中有功能强大的***\*发送数据到 Web 服务功能\****，可以实现无缝实现上述操作。

填写动作参数:

“保存数据到 Web 服务” 动作需要两个参数：

1). 消息内容模板，即 HTTP 请求体。这个例子里我们向 TDengine 插入一条数据，应当在请求体内拼接携带 INSERT SQL。注意我们应当在 SQL 中指定数据库名，字符类型也要用单引号括起来， 消息内容模板为：

```sql
insert into test.t_mqtt_msg(ts, msgid, topic, qos, payload) values (now, '${id}', '${topic}', ${qos}, '${payload}')
```

2). 关联资源的 ID。现在资源下拉框为空，可以点击旁边的 “新建” 来创建一个 Web 服务资源:

填写资源配置:

请求 URL 填写 http://127.0.0.1:6041/rest/sql ，请求方法选择 POST;
**还需添加 Authorization 请求头作为认证信息**，请求头的值为 Basic + TDengine {username}:{password} 经过Base64 编码之后的字符串, 例如默认的 root:taosdata 编码后为 `cm9vdDp0YW9zZGF0YQ==`，
填入的值为 `Basic cm9vdDp0YW9zZGF0YQ==`。

![image-20200730093728092](./assets/rule-engine/tdengine-webhook.png)

点击 “新建” 按钮。

返回响应动作界面，点击 “确认”。

![image-20200730093457366](./assets/rule-engine/image-20200730093457366.png)

返回规则创建界面，点击 “创建”。


### 测试

在规则列表里，点击 “查看” 按钮或规则 ID 连接，可以预览刚才创建的规则:

![image-20200729165826748](./assets/rule-engine/image-20200729165826748.png)

规则已经创建完成，现在发一条数据:

```bash
Topic: "t/a"
QoS: 1
Payload: "hello"
```

然后检查 TDengine 表，新的 record 是否添加成功:

```sql
select * from t_mqtt_msg;
```

![image-20200729174914518](./assets/rule-engine/image-20200729174914518.png)

## 保存数据到 Oracle DB

创建 t_mqtt_msg 表:

```sql
CREATE TABLE t_mqtt_msg (msgid VARCHAR2(64),topic VARCHAR2(255), qos NUMBER(1), payload NCLOB)
```

![image](./assets/rule-engine/oracle_action_1.png)

创建规则:

打开 [EMQ X Dashboard](http://127.0.0.1:18083/#/rules)，选择左侧的 “规则” 选项卡。

填写规则 SQL:

```sql
SELECT * FROM "t/#"
```

![image](./assets/rule-engine/rule_sql.png)

关联动作:

在 “响应动作” 界面选择 “添加”，然后在 “动作” 下拉框里选择 “保存数据到 Oracle Database”。

![image](./assets/rule-engine/oracle_action_2.png)

填写动作参数:

“保存数据到 Oracle Database” 动作需要两个参数：

1). SQL 模板。这个例子里我们向 Oracle Database 插入一条数据，SQL 模板为:

```sql
INSERT INTO T_MQTT_MSG (MSGID, TOPIC, QOS, PAYLOAD) values ('${id}', '${topic}', '${qos}', '${payload}');
```

2). 关联资源的 ID。现在资源下拉框为空，可以点击右上角的 “新建资源” 来创建一个 Oracle Database  资源:

填写资源配置:

![image](./assets/rule-engine/oracle_action_3.png)

点击 “新建” 按钮。

返回响应动作界面，点击 “确认”。

![image](./assets/rule-engine/oracle_action_4.png)

返回规则创建界面，点击 “创建”。

![image](./assets/rule-engine/oracle_action_5.png)

在规则列表里，点击 “查看” 按钮或规则 ID 连接，可以预览刚才创建的规则:

![image](./assets/rule-engine/oracle_action_6.png)

规则已经创建完成，现在发一条数据:

```bash
Topic: "t/a"
QoS: 1
Payload: "hello"
```

查看规则命中次数

![image](./assets/rule-engine/oracle_action_7.png)

## 保存数据到 SQLServer

搭建 SQLServer 数据库，并设置用户名密码为 sa/mqtt_public，以 MacOS X 为例:

```bash
docker run -e 'ACCEPT_EULA=Y' -e 'SA_PASSWORD=mqtt_public' -p 1433:1433 -d mcr.microsoft.com/mssql/server:2017-latest
```

进入SQLServer容器， 初始化 SQLServer 表:

```bash
$ /opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P mqtt_public -d master
$ mysql -u root -h localhost -ppublic
```

创建 “test” 数据库:
```bash
CREATE DATABASE test;
go;
```
创建 t_mqtt_msg 表:

```sql
USE test;
go;
CREATE TABLE t_mqtt_msg (id int PRIMARY KEY IDENTITY(1000000001,1) NOT NULL,
                         msgid   VARCHAR(64) NULL,
                         topic   VARCHAR(100) NULL,
                         qos     tinyint NOT NULL DEFAULT 0,
                         payload NVARCHAR(100) NULL,
                         arrived DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP);
go;
```

配置 odbc 驱动:
```
$ brew install unixodbc freetds
$ vim /usr/local/etc/odbcinst.ini

[ms-sql]
Description = ODBC for FreeTDS
Driver      = /usr/local/lib/libtdsodbc.so
Setup       = /usr/local/lib/libtdsodbc.so
FileUsage   = 1
```

创建规则:

打开 [EMQ X Dashboard](http://127.0.0.1:18083/#/rules)，选择左侧的 “规则” 选项卡。

填写规则 SQL:

```sql
SELECT * FROM "t/#"
```

![image](./assets/rule-engine/sqlserver1.png)

关联动作:

在 “响应动作” 界面选择 “添加”，然后在 “动作” 下拉框里选择 “保存数据到 SQLServer”。

![image](./assets/rule-engine/sqlserver2.png)

填写动作参数:

“保存数据到 SQLServer” 动作需要两个参数：
4
2). SQL 模板。这个例子里我们向 SQLServer 插入一条数据，SQL
​    模板为:

```sql
insert into t_mqtt_msg(msgid, topic, qos, payload) values ('${id}', '${topic}', ${qos}, '${payload}')
```

![image](./assets/rule-engine/sqlserver4.png)

1). 关联资源的 ID。现在资源下拉框为空，可以点击右上角的 “新建资源” 来创建一个 SQLServer 资源:

填写资源配置:
数据库名填写 “mqtt”，用户名填写 “sa”，密码填写 “mqtt_public”

![image](./assets/rule-engine/sqlserver3.png)

点击 “新建” 按钮。

返回响应动作界面，点击 “确认”。

![image](./assets/rule-engine/sqlserver5.png)

返回规则创建界面，点击 “创建”。

![image](./assets/rule-engine/sqlserver6.png)

在规则列表里，点击 “查看” 按钮或规则 ID 连接，可以预览刚才创建的规则:

![image](./assets/rule-engine/sqlserver7.png)

规则已经创建完成，现在发一条数据:

```bash
Topic: "t/a"
QoS: 1
Payload: "hello"
```

然后检查 SQLServer 表，新的 record 是否添加成功:

![image](./assets/rule-engine/sqlserver8.png)

## 保存数据到 DolphinDB

[DolphinDB](https://www.dolphindb.cn) 是由浙江智臾科技有限公司研发的一款高性能分布式时序数据库，集成了功能强大的编程语言和高容量高速度的流数据分析系统，为海量结构化数据的快速存储、检索、分析及计算提供一站式解决方案，适用于量化金融及工业物联网等领域。

EMQ X 用 Erlang 实现了 DolphinDB 的客户端 API，它通过 TCP 的方式将数据传输到 DolphinDB 进行存储。

### 搭建 DolphinDB

目前，EMQ X 仅适配 DolphinDB 1.20.7 的版本。

以 Linux 版本为例，前往官网下载社区最新版本的 Linux64 安装包：https://www.dolphindb.cn/downloads.html

将安装包的 server 目录上传至服务器目录 `/opts/app/dolphindb`，并测试启动是否正常：

```bash
chmod +x ./dolphindb
./dolphindb

## 启动成功后，会进入到 dolphindb 命令行，执行 1+1
>1+1
2
```

启动成功，并得到正确输出，表示成功安装 DolphinDB。然后使用 `<CRTL+D>` 关闭 DolphinDB。

现在，我们需要打开 DolphinDB 的 StreamTable 的发布/订阅的功能，并创建相关数据表，以实现 EMQ X 消息存储并持久化的功能：

1. 修改 DolphinDB 的配置文件 `vim dolphindb.cfg` 加入以下配置项，以打开 发布/订阅 的功能：
``` properties
## Publisher for streaming
maxPubConnections=10
persistenceDir=/ddb/pubdata/
#persistenceWorkerNum=
#maxPersistenceQueueDepth=
#maxMsgNumPerBlock=
#maxPubQueueDepthPerSite=

## Subscriber for streaming
subPort=8000
#subExecutors=
#maxSubConnections=
#subExecutorPooling=
#maxSubQueueDepth=
```

2. 后台启动 DolphinDB 服务：
```bash
## 启动完成后，DolphinDB 会监听 8848 端口供客户端使用。
nohup ./dolphindb -console 0 &
```

3. 前往 DolphinDB 官网，下载合适的 GUI 客户端连接 DolphinDB 服务：
    - 前往 [下载页](http://www.dolphindb.cn/alone/alone.php?id=10) 下载 `DolphinDB GUI`
    - DolphinDB GUI 客户端依赖 Java 环境，先确保已经安装 Java
    - 前往 DolphinDB GUI 目录中执行 `sh gui.sh` 启动客户端
    - 在客户端中添加 Server 并创建一个 Project，和脚本文件。

4. 创建分布式数据库，和 StreamTable 表；并将 StreamTable 的数据持久化到分布式表中：
```ruby
// 创建一个名为 emqx 的 分布式文件数据库
// 并创建一张名为 `msg` 表，按 `clientid` 和 `topic` 的 HASH 值进行分区：
schema = table(1:0, `clientid`topic`qos`payload, [STRING, STRING, INT, STRING])
db1 = database("", HASH, [STRING, 8])
db2 = database("", HASH, [STRING, 8])
db = database("dfs://emqx", COMPO, [db1, db2])
db.createPartitionedTable(schema, "msg",`clientid`topic)

// 创建名为 `st_msg` 的 StreamTable 表，并将数据持久化到 `msg` 表。
share streamTable(10000:0,`clientid`topic`qos`payload, [STRING,STRING,INT,STRING]) as st_msg
msg_ref= loadTable("dfs://emqx", "msg")
subscribeTable(, "st_msg", "save_msg_to_dfs", 0, msg_ref, true)

// 查询 msg_ref；检查是否创建成功
select * from msg_ref;
```
完成后，可以看到一张空的 `msg_ref` 已创建成功：

![Create DolphinDB Table](./assets/rule-engine/dolphin_create_tab.jpg)

至此，DolphinDB 的配置已经完成了。

详细的 DolphinDB 使用文档请参考：
- 用户指南：https://github.com/dolphindb/Tutorials_CN/blob/master/dolphindb_user_guide.md
- IoT 场景示例：https://gitee.com/dolphindb/Tutorials_CN/blob/master/iot_examples.md
- 流处理指南：https://github.com/dolphindb/Tutorials_CN/blob/master/streaming_tutorial.md
- 编程手册：https://www.dolphindb.cn/cn/help/index.html

### 配置规则引擎

创建规则:

打开 [EMQ X Dashboard](http://127.0.0.1:18083/#/rules)，选择左侧的 “规则” 选项卡。

填写规则 SQL:

```sql
SELECT * FROM "t/#"
```

![image](./assets/rule-engine/rule_sql.png)

关联动作:

在 “响应动作” 界面选择 “添加”，然后在 “动作” 下拉框里选择 “保存数据到 DolphinDB”。

![image](./assets/rule-engine/dolphin_action_1.jpg)

填写动作参数:

“保存数据到 DolphinDB” 动作需要两个参数：

1). SQL 模板。这个例子里我们向流表 `st_msg` 中插入一条数据，SQL 模板为:

```sql
insert into st_msg values(${clientid}, ${topic}, ${qos}, ${payload})
```

2). 关联资源的 ID。现在资源下拉框为空，可以点击右上角的 “新建资源” 来创建一个DolphinDB 资源:

填写资源配置:

服务器地址填写对应上文部署的 DolphinDB 的服务器，用户名为 `admin` 密码为 `123456`

![image](./assets/rule-engine/dolphin_res_1.jpg)

点击 “确定” 按钮。

返回响应动作界面，点击 “确定”。

![image](./assets/rule-engine/dolphin_action_2.jpg)

返回规则创建界面，点击 “创建”。

![image](./assets/rule-engine/dolphin_action_3.jpg)

在规则列表里，点击 “查看” 按钮或规则 ID 连接，可以预览刚才创建的规则:

![image](./assets/rule-engine/dolphin_overview.jpg)

规则已经创建完成，现在发一条数据:

```bash
Topic: "t/a"
QoS: 1
Payload: "hello"
```

然后检查持久化的 `msg_dfs` 表，新的数据是否添加成功:

![image](./assets/rule-engine/dolphin_result.jpg)