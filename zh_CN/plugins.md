# 扩展插件(Plugins)

emqttd 消息服务器通过模块注册和钩子(Hooks)机制，支持用户开发扩展插件定制服务器认证鉴权与业务功能。

emqtt 项目组开发维护的插件包括:

| 插件                                                                        | 说明                       |
| --------------------------------------------------------------------------- | -------------------------- |
| [ emqttd_dashboard ](https://github.com/emqtt/emqttd_dashboard)             | Web 控制台插件(默认加载)   |
| [ emqttd_auth_http ](https://github.com/emqtt/emqttd_auth_http)             | HTTP 认证/访问控制         |
| [ emqttd_plugin_mysql ](https://github.com/emqtt/emqttd_plugin_mysql)       | MySQL 认证/访问控制        |
| [ emqttd_plugin_pgsql ](https://github.com/emqtt/emqttd_plugin_pgsql)       | PostgreSQL 认证/访问控制   |
| [ emqttd_plugin_redis ](https://github.com/emqtt/emqttd_plugin_redis)       | Redis 认证/访问控制        |
| [ emqttd_plugin_mongo ](https://github.com/emqtt/emqttd_plugin_mongo)       | MongoDB 认证/访问控制      |
| [ emqttd_stomp ](https://github.com/emqtt/emqttd_stomp)                     | Stomp 协议支持             |
| [ emqttd_sockjs ](https://github.com/emqtt/emqttd_sockjs)                   | Stomp over SockJS 协议支持 |
| [ emqttd_recon ](https://github.com/emqtt/emqttd_recon)                     | Recon 性能调试             |
| [ emqttd_reloader ](https://github.com/emqtt/emqttd_reloader)               | Reloader 代码热加载插件    |
| [ emqttd_plugin_template ](https://github.com/emqtt/emqttd_plugin_template) | 插件开发模版               |

## emqttd_plugin_template: 插件开发模版

emqttd 插件实际是一个 Erlang 应用，带自身的配置文件'etc/plugin.config"，置于'emqttd/plugins'目录下。

plugins/emqttd_plugin_template 是一个模版插件，典型目录结构:

| 目录/文件         | 说明             |
| ----------------- | ---------------- |
| etc/plugin.config | 插件配置文件     |
| ebin/             | 插件程序文件目录 |

### 加载、卸载插件

管理命令行'./bin/emqttd_ctl'加载卸载插件。

加载插件:

    ./bin/emqttd_ctl plugins load \<PluginName>

卸载插件:

    ./bin/emqttd_ctl plugins unload \<PluginName>

查询插件:

    ./bin/emqttd_ctl plugins list

## emqttd_dashboard: Dashboard 插件

emqttd 消息服务器的 Web 管理控制台。插件项目地址: [ https://github.com/emqtt/emqttd_dashboard
](https://github.com/emqtt/emqttd_dashboard)

emqttd 消息服务器默认加载 Dashboard 插件。URL 地址: http://localhost:18083 ，缺省用户名/密码: admin/public。

Dashboard 插件可查询 emqttd 基本信息、统计数据、度量数据，查询系统客户端(Client)、会话(Session)、主题(Topic)、订阅(Subscription)。

![image](./_static/images/dashboard.png)

### Dashboard 插件设置

plugins/emqttd_dashboard/etc/plugin.config:

    [
      {emqttd_dashboard, [
        {listener,
          {emqttd_dashboard, 18083, [
            {acceptors, 4},
            {max_clients, 512}]}
        }
      ]}
    ].

## emqttd_auth_http: HTTP 认证/访问控制插件

HTTP 认证/访问控制插件: [ https://github.com/emqtt/emqttd_auth_http
](https://github.com/emqtt/emqttd_auth_http)

::: tip
1.1 版本支持
:::

### 配置插件

plugins/emqttd_auth_http/etc/plugin.config:

    [

      {emqttd_auth_http, [

        %% Variables: %u = username, %c = clientid, %a = ipaddress, %t = topic

        {super_req, [
          {method, post},
          {url, "http://localhost:8080/mqtt/superuser"},
          {params, [
            {username, "%u"},
            {clientid, "%c"}
          ]}
        ]},

        {auth_req, [
          {method, post},
          {url, "http://localhost:8080/mqtt/auth"},
          {params, [
            {clientid, "%c"},
            {username, "%u"},
            {password, "%P"}
          ]}
        ]},

        %% 'access' parameter: sub = 1, pub = 2

        {acl_req, [
          {method, post},
          {url, "http://localhost:8080/mqtt/acl"},
          {params, [
            {access,   "%A"},
            {username, "%u"},
            {clientid, "%c"},
            {ipaddr,   "%a"},
            {topic,    "%t"}
          ]}
        ]}
      ]}

    ].

### HTTP API

认证/ACL 成功，API 返回 200

认证/ACL 失败，API 返回 4xx

### 加载插件

    ./bin/emqttd_ctl plugins load emqttd_auth_http

## emqttd_plugin_mysql: MySQL 认证/访问控制插件

MySQL 认证/访问控制插件，基于 MySQL 库表认证鉴权: [ https://github.com/emqtt/emqttd_plugin_mysql
](https://github.com/emqtt/emqttd_plugin_mysql)

### MQTT 用户表

    CREATE TABLE `mqtt_user` (
      `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
      `username` varchar(100) DEFAULT NULL,
      `password` varchar(100) DEFAULT NULL,
      `salt` varchar(20) DEFAULT NULL,
      `is_superuser` tinyint(1) DEFAULT 0,
      `created` datetime DEFAULT NULL,
      PRIMARY KEY (`id`),
      UNIQUE KEY `mqtt_username` (`username`)
    ) ENGINE=MyISAM DEFAULT CHARSET=utf8;

::: tip
MySQL 插件可使用系统自有的用户表，通过'authquery'配置查询语句。
:::

### MQTT 访问控制表

    CREATE TABLE `mqtt_acl` (
      `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
      `allow` int(1) DEFAULT NULL COMMENT '0: deny, 1: allow',
      `ipaddr` varchar(60) DEFAULT NULL COMMENT 'IpAddress',
      `username` varchar(100) DEFAULT NULL COMMENT 'Username',
      `clientid` varchar(100) DEFAULT NULL COMMENT 'ClientId',
      `access` int(2) NOT NULL COMMENT '1: subscribe, 2: publish, 3: pubsub',
      `topic` varchar(100) NOT NULL DEFAULT '' COMMENT 'Topic Filter',
      PRIMARY KEY (`id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8;

    INSERT INTO `mqtt_acl` (`id`, `allow`, `ipaddr`, `username`, `clientid`, `access`, `topic`)
    VALUES
        (1,1,NULL,'$all',NULL,2,'#'),
        (2,0,NULL,'$all',NULL,1,'$SYS/#'),
        (3,0,NULL,'$all',NULL,1,'eq #'),
        (5,1,'127.0.0.1',NULL,NULL,2,'$SYS/#'),
        (6,1,'127.0.0.1',NULL,NULL,2,'#'),
        (7,1,NULL,'dashboard',NULL,1,'$SYS/#');

### 配置插件

plugins/emqttd_plugin_mysql/etc/plugin.config:

    [

      {emqttd_plugin_mysql, [

        {mysql_pool, [
            %% ecpool options
            {pool_size, 8},
            {auto_reconnect, 3},

            %% mysql options
            {host,     "localhost"},
            {port,     3306},
            {user,     ""},
            {password, ""},
            {database, "mqtt"},
            {encoding, utf8}
        ]},

        %% Variables: %u = username, %c = clientid, %a = ipaddress

        %% Superuser Query
        {superquery, "select is_superuser from mqtt_user where username = '%u' limit 1"},

        %% Authentication Query: select password only
        {authquery, "select password from mqtt_user where username = '%u' limit 1"},

        %% hash algorithm: plain, md5, sha, sha256, pbkdf2?
        {password_hash, sha256},

        %% select password with salt
        %% {authquery, "select password, salt from mqtt_user where username = '%u'"},

        %% sha256 with salt prefix
        %% {password_hash, {salt, sha256}},

        %% sha256 with salt suffix
        %% {password_hash, {sha256, salt}},

        %% '%a' = ipaddress, '%u' = username, '%c' = clientid
        %% Comment this query, the acl will be disabled
        {aclquery, "select allow, ipaddr, username, clientid, access, topic from mqtt_acl where ipaddr = '%a' or username = '%u' or username = '$all' or clientid = '%c'"},

        %% If no ACL rules matched, return...
        {acl_nomatch, allow}

      ]}

    ].

### 加载插件

    ./bin/emqttd_ctl plugins load emqttd_plugin_mysql

## emqttd_plugin_pgsql: PostgreSQL 认证/访问控制插件

PostgreSQL 认证/访问控制插件，基于 PostgreSQL 库表认证鉴权: [
https://github.com/emqtt/emqttd_plugin_pgsql
](https://github.com/emqtt/emqttd_plugin_pgsql)

### MQTT 用户表

    CREATE TABLE mqtt_user (
      id SERIAL primary key,
      is_superuser boolean,
      username character varying(100),
      password character varying(100),
      salt character varying(40)
    );

### MQTT 访问控制表

    CREATE TABLE mqtt_acl (
      id SERIAL primary key,
      allow integer,
      ipaddr character varying(60),
      username character varying(100),
      clientid character varying(100),
      access  integer,
      topic character varying(100)
    );

    INSERT INTO mqtt_acl (id, allow, ipaddr, username, clientid, access, topic)
    VALUES
        (1,1,NULL,'$all',NULL,2,'#'),
        (2,0,NULL,'$all',NULL,1,'$SYS/#'),
        (3,0,NULL,'$all',NULL,1,'eq #'),
        (5,1,'127.0.0.1',NULL,NULL,2,'$SYS/#'),
        (6,1,'127.0.0.1',NULL,NULL,2,'#'),
        (7,1,NULL,'dashboard',NULL,1,'$SYS/#');

### 配置插件

plugins/emqttd_plugin_pgsql/etc/plugin.config:

    [

      {emqttd_plugin_pgsql, [

        {pgsql_pool, [
            %% ecpool options
            {pool_size, 8},
            {auto_reconnect, 3},

            %% pgsql options
            {host, "localhost"},
            {port, 5432},
            {ssl, false},
            {username, "feng"},
            {password, ""},
            {database, "mqtt"},
            {encoding,  utf8}
        ]},

        %% Variables: %u = username, %c = clientid, %a = ipaddress

        %% Superuser Query
        {superquery, "select is_superuser from mqtt_user where username = '%u' limit 1"},

        %% Authentication Query: select password only
        {authquery, "select password from mqtt_user where username = '%u' limit 1"},

        %% hash algorithm: plain, md5, sha, sha256, pbkdf2?
        {password_hash, sha256},

        %% select password with salt
        %% {authquery, "select password, salt from mqtt_user where username = '%u'"},

        %% sha256 with salt prefix
        %% {password_hash, {salt, sha256}},

        %% sha256 with salt suffix
        %% {password_hash, {sha256, salt}},

        %% Comment this query, the acl will be disabled. Notice: don't edit this query!
        {aclquery, "select allow, ipaddr, username, clientid, access, topic from mqtt_acl
                     where ipaddr = '%a' or username = '%u' or username = '$all' or clientid = '%c'"},

        %% If no rules matched, return...
        {acl_nomatch, allow}
      ]}
    ].

### 加载插件

    ./bin/emqttd_ctl plugins load emqttd_plugin_pgsql

## emqttd_plugin_redis: Redis 认证/访问控制插件

基于 Redis 认证/访问控制: [ https://github.com/emqtt/emqttd_plugin_redis
](https://github.com/emqtt/emqttd_plugin_redis)

### 配置插件

plugins/emqttd_plugin_redis/etc/plugin.config:

    [
      {emqttd_plugin_redis, [

        {eredis_pool, [
          %% ecpool options
          {pool_size, 8},
          {auto_reconnect, 2},

          %% eredis options
          {host, "127.0.0.1"},
          {port, 6379},
          {database, 0},
          {password, ""}
        ]},

        %% Variables: %u = username, %c = clientid

        %% HMGET mqtt_user:%u is_superuser
        {supercmd, ["HGET", "mqtt_user:%u", "is_superuser"]},

        %% HMGET mqtt_user:%u password
        {authcmd, ["HGET", "mqtt_user:%u", "password"]},

        %% Password hash algorithm: plain, md5, sha, sha256, pbkdf2?
        {password_hash, sha256},

        %% SMEMBERS mqtt_acl:%u
        {aclcmd, ["SMEMBERS", "mqtt_acl:%u"]},

        %% If no rules matched, return...
        {acl_nomatch, deny},

        %% Load Subscriptions form Redis when client connected.
        {subcmd, ["HGETALL", "mqtt_subs:%u"]}
      ]}
    ].

### 用户 Hash

默认基于用户 Hash 认证:

    HSET mqtt_user:\<username> is_superuser 1
    HSET mqtt_user:\<username> password "passwd"

### ACL 规则 SET

默认采用 SET 存储 ACL 规则:

    SADD mqtt_acl:\<username> "publish topic1"
    SADD mqtt_acl:\<username> "subscribe topic2"
    SADD mqtt_acl:\<username> "pubsub topic3"

### 订阅 Hash

插件还支持 Redis 中创建 MQTT 订阅。当 MQTT 客户端连接成功，会自动从 Redis 加载订阅:

    HSET mqtt_subs:\<username> topic1 0
    HSET mqtt_subs:\<username> topic2 1
    HSET mqtt_subs:\<username> topic3 2

### 加载插件

    ./bin/emqttd_ctl plugins load emqttd_plugin_redis

## emqttd_plugin_mongo: MongoDB 认证/访问控制插件

基于 MongoDB 认证/访问控制: [ https://github.com/emqtt/emqttd_plugin_mongo
](https://github.com/emqtt/emqttd_plugin_mongo)

### 配置插件

plugins/emqttd_plugin_mongo/etc/plugin.config:

    [
      {emqttd_plugin_mongo, [

        {mongo_pool, [
          {pool_size, 8},
          {auto_reconnect, 3},

          %% Mongodb Driver Opts
          {host, "localhost"},
          {port, 27017},
          %% {login, ""},
          %% {password, ""},
          {database, "mqtt"}
        ]},

        %% Variables: %u = username, %c = clientid

        %% Superuser Query
        {superquery, [
          {collection, "mqtt_user"},
          {super_field, "is_superuser"},
          {selector, {"username", "%u"}}
        ]},

        %% Authentication Query
        {authquery, [
          {collection, "mqtt_user"},
          {password_field, "password"},
          %% Hash Algorithm: plain, md5, sha, sha256, pbkdf2?
          {password_hash, sha256},
          {selector, {"username", "%u"}}
        ]},

        %% ACL Query: "%u" = username, "%c" = clientid
        {aclquery, [
          {collection, "mqtt_acl"},
          {selector, {"username", "%u"}}
        ]},

        %% If no ACL rules matched, return...
        {acl_nomatch, deny}

      ]}
    ].

### MongoDB 数据库

    use mqtt
    db.createCollection("mqtt_user")
    db.createCollection("mqtt_acl")
    db.mqtt_user.ensureIndex({"username":1})

::: tip
数据库、集合名称可自定义
:::

### 用户集合(User Collection)

    {
        username: "user",
        password: "password hash",
        is_superuser: boolean (true, false),
        created: "datetime"
    }

示例:

    db.mqtt_user.insert({username: "test", password: "password hash", is_superuser: false})
    db.mqtt_user:insert({username: "root", is_superuser: true})

### ACL 集合(ACL Collection)

    {
        username: "username",
        clientid: "clientid",
        publish: ["topic1", "topic2", ...],
        subscribe: ["subtop1", "subtop2", ...],
        pubsub: ["topic/#", "topic1", ...]
    }

示例:

    db.mqtt_acl.insert({username: "test", publish: ["t/1", "t/2"], subscribe: ["user/%u", "client/%c"]})
    db.mqtt_acl.insert({username: "admin", pubsub: ["#"]})

### 加载插件

    ./bin/emqttd_ctl plugins load emqttd_plugin_mongo

## emqttd_stomp: Stomp 协议插件

Stomp 协议插件。支持 STOMP 1.0/1.1/1.2 协议客户端连接 emqttd，发布订阅 MQTT 消息。

### 配置插件

::: tip
Stomp 协议端口: 61613
:::

plugins/emqttd_stomp/etc/plugin.config:

    [
      {emqttd_stomp, [

        {default_user, [
            {login,    "guest"},
            {passcode, "guest"}
        ]},

        {allow_anonymous, true},

        %%TODO: unused...
        {frame, [
          {max_headers,       10},
          {max_header_length, 1024},
          {max_body_length,   8192}
        ]},

        {listeners, [
          {emqttd_stomp, 61613, [
            {acceptors,   4},
            {max_clients, 512}
          ]}
        ]}

      ]}
    ].

### 加载插件

    ./bin/emqttd_ctl plugins load emqttd_stomp

## emqttd_sockjs: Stomp/Sockjs 插件

### 配置插件

::: tip
缺省端口: 61616
:::

    [
      {emqttd_sockjs, [

        {sockjs, []},

        {cowboy_listener, {stomp_sockjs, 61616, 4}},

      ]}
    ].

### 加载插件

::: tip
需先加载 emqttd_stomp 插件
:::

    ./bin/emqttd_ctl plugins load emqttd_stomp

    ./bin/emqttd_ctl plugins load emqttd_sockjs

### 插件演示页面

> [ http://localhost:61616/index.html ](http://localhost:61616/index.html)

## emqttd_recon: Recon 性能调试插件

emqttd_recon 插件集成 recon 性能调测库，'./bin/emqttd_ctl'命令行注册 recon 命令。

### 加载插件

    ./bin/emqttd_ctl plugins load emqttd_recon

### recon 命令

    ./bin/emqttd_ctl recon

    recon memory                 #recon_alloc:memory/2
    recon allocated              #recon_alloc:memory(allocated_types, current|max)
    recon bin_leak               #recon:bin_leak(100)
    recon node_stats             #recon:node_stats(10, 1000)
    recon remote_load Mod        #recon:remote_load(Mod)

## emqttd_reloader: 代码热加载插件

用于开发调试的代码热升级插件。加载该插件后，emqttd 会自动热升级更新代码。

::: tip
产品部署环境不建议使用该插件
:::

### 加载插件

    ./bin/emqttd_ctl plugins load emqttd_reloader

### reload 命令

    ./bin/emqttd_ctl reload

    reload \<Module>             # Reload a Module

## emqttd 插件开发

### 创建插件项目

github 下载 emqttd 源码库，plugins/目录下创建插件应用。

模版代码请参考: emqttd_plugin_templage

### 注册认证/访问控制模块

认证演示模块 - emqttd_auth_demo.erl

    -module(emqttd_auth_demo).

    -behaviour(emqttd_auth_mod).

    -include("../../../include/emqttd.hrl").

    -export([init/1, check/3, description/0]).

    init(Opts) -> {ok, Opts}.

    check(#mqtt_client{client_id = ClientId, username = Username}, Password, _Opts) ->
        io:format("Auth Demo: clientId=~p, username=~p, password=~p~n",
                  [ClientId, Username, Password]),
        ok.

    description() -> "Demo Auth Module".

访问控制演示模块 - emqttd_acl_demo.erl

    -module(emqttd_acl_demo).

    -include("../../../include/emqttd.hrl").

    %% ACL callbacks
    -export([init/1, check_acl/2, reload_acl/1, description/0]).

    init(Opts) ->
        {ok, Opts}.

    check_acl({Client, PubSub, Topic}, Opts) ->
        io:format("ACL Demo: ~p ~p ~p~n", [Client, PubSub, Topic]),
        allow.

    reload_acl(_Opts) ->
        ok.

    description() -> "ACL Module Demo".

注册认证、访问控制模块 - emqttd_plugin_template_app.erl

    ok = emqttd_access_control:register_mod(auth, emqttd_auth_demo, []),
    ok = emqttd_access_control:register_mod(acl, emqttd_acl_demo, []),

### 注册扩展钩子(Hooks)

通过钩子(Hook)处理客户端上下线、主题订阅、消息收发。

emqttd_plugin_template.erl:

    %% Called when the plugin application start
    load(Env) ->
        emqttd:hook('client.connected', fun ?MODULE:on_client_connected/3, [Env]),
        emqttd:hook('client.disconnected', fun ?MODULE:on_client_disconnected/3, [Env]),
        emqttd:hook('client.subscribe', fun ?MODULE:on_client_subscribe/3, [Env]),
        emqttd:hook('client.subscribe.after', fun ?MODULE:on_client_subscribe_after/3, [Env]),
        emqttd:hook('client.unsubscribe', fun ?MODULE:on_client_unsubscribe/3, [Env]),
        emqttd:hook('message.publish', fun ?MODULE:on_message_publish/2, [Env]),
        emqttd:hook('message.delivered', fun ?MODULE:on_message_delivered/3, [Env]),
        emqttd:hook('message.acked', fun ?MODULE:on_message_acked/3, [Env]).

扩展钩子(Hook):

| 钩子                   | 说明               |
| ---------------------- | ------------------ |
| client.connected       | 客户端上线         |
| client.subscribe       | 客户端订阅主题前   |
| client.subscribe.after | 客户端订阅主题后   |
| client.unsubscribe     | 客户端取消订阅主题 |
| message.publish        | MQTT 消息发布      |
| message.delivered      | MQTT 消息送达      |
| message.acked          | MQTT 消息回执      |
| client.disconnected    | 客户端连接断开     |

### 注册扩展命令行

扩展命令行演示模块 - emqttd_cli_demo.erl

    -module(emqttd_cli_demo).

    -include("../../../include/emqttd_cli.hrl").

    -export([cmd/1]).

    cmd(["arg1", "arg2"]) ->
        ?PRINT_MSG("ok");

    cmd(_) ->
        ?USAGE([{"cmd arg1 arg2",  "cmd demo"}]).

注册命令行模块 - emqttd_plugin_template_app.erl

    emqttd_ctl:register_cmd(cmd, {emqttd_cli_demo, cmd}, []).

插件加载后，'./bin/emqttd_ctl'新增命令行:

    ./bin/emqttd_ctl cmd arg1 arg2
