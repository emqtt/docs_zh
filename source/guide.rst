
.. _guide:

=====================
用户指南 (User Guide)
=====================

.. _authentication:

-------------
MQTT 认证设置
-------------

*EMQ X* 消息服务器认证由一系列认证插件(Plugin)提供，系统支持按用户名密码、ClientID 或匿名认证。

系统默认开启匿名认证(anonymous)，通过加载认证插件可开启的多个认证模块组成认证链::

               ----------------           ----------------           ------------
    Client --> | Username认证 | -ignore-> | ClientID认证 | -ignore-> | 匿名认证 |
               ----------------           ----------------           ------------
                      |                         |                         |
                     \|/                       \|/                       \|/
                allow | deny              allow | deny              allow | deny

.. NOTE:: EMQ X 消息服务器提供 MySQL、PostgreSQL、Redis、MongoDB、HTTP、LDAP 认证插件。

------------
开启匿名认证
------------

etc/emqx.conf 配置启用匿名认证:

.. code-block:: properties

    ## Allow anonymous authentication by default if no auth plugins loaded.
    ## Notice: Disable the option in production deployment!
    ##
    ## Value: true | false
    allow_anonymous = true

EMQ X 3.0 版本提供的认证插件包括:

+----------------------------+---------------------------+
| 插件                       | 说明                      |
+============================+===========================+
| `emqx_auth_clientid`_      | ClientId 认证/鉴权插件    |
+----------------------------+---------------------------+
| `emqx_auth_username`_      | 用户名密码认证/鉴权插件   |
+----------------------------+---------------------------+
| `emqx_auth_ldap`_          | LDAP 认证/鉴权插件        |
+----------------------------+---------------------------+
| `emqx_auth_http`_          | HTTP 认证/鉴权插件        |
+----------------------------+---------------------------+
| `emqx_auth_mysql`_         | MySQ L认证/鉴权插件       |
+----------------------------+---------------------------+
| `emqx_auth_pgsql`_         | Postgre 认证/鉴权插件     |
+----------------------------+---------------------------+
| `emqx_auth_redis`_         | Redis 认证/鉴权插件       |
+----------------------------+---------------------------+
| `emqx_auth_mongo`_         | MongoDB 认证/鉴权插件     |
+----------------------------+---------------------------+
| `emqx_auth_jwt`_           | JWT 认证/鉴权插件         |
+----------------------------+---------------------------+



--------------
用户名密码认证
--------------

基于 MQTT 登录用户名(username)、密码(password)认证。

etc/plugins/emqx_auth_username.conf 中配置默认用户:

.. code-block:: properties

    auth.user.$N.username = admin
    auth.user.$N.password = public

启用 `emqx_auth_username`_ 插件:

.. code-block:: bash

    ./bin/emqx_ctl plugins load emqx_auth_username

使用 `./bin/emqx_ctl users` 命令添加用户::

   $ ./bin/emqx_ctl users add <Username> <Password>

-------------
ClientId 认证
-------------

基于 MQTT 客户端 ID 认证。

etc/plugins/emqx_auth_clientid.conf:

.. code-block:: properties

    auth.client.$N.clientid = clientid
    auth.client.$N.password = passwd

启用 `emqx_auth_clientid`_ 插件:

.. code-block:: bash

    ./bin/emqx_ctl plugins load emqx_auth_clientid

-------------
LDAP 插件认证
-------------

etc/plugins/emqx_auth_ldap.conf 配置 LDAP 参数:

.. code-block:: properties

    ## LDAP server list, seperated by ','.
    ## Value: String
    auth.ldap.servers = 127.0.0.1

    ## LDAP server port.
    ## Value: Port
    auth.ldap.port = 389

    ## LDAP Bind DN.
    ## Value: DN
    auth.ldap.bind_dn = cn=root,dc=emqtt,dc=com

    ## LDAP Bind Password.
    ## Value: String
    auth.ldap.bind_password = public

    ## LDAP query timeout.
    ## Value: Number
    auth.ldap.timeout = 30

    ## Authentication DN.
    ##  -%u: username
    ##  -%c: clientid
    ##
    ## Value: DN
    auth.ldap.auth_dn = cn=%u,ou=auth,dc=emqtt,dc=com

    ## Password hash.
    ## Value: plain | md5 | sha | sha256
    auth.ldap.password_hash = sha256

    ## Whether to enable SSL.
    ## Value: true | false
    auth.ldap.ssl = false

启用 LDAP 认证插件::

    ./bin/emqx_ctl plugins load emqx_auth_ldap

-------------
HTTP 插件认证
-------------

etc/plugins/emqx_auth_http.conf 配置 'super_req', 'auth_req':

.. code-block:: properties

    ## Variables: %u = username, %c = clientid, %a = ipaddress, %P = password, %t = topic

    auth.http.auth_req = http://127.0.0.1:8080/mqtt/auth
    auth.http.auth_req.method = post
    auth.http.auth_req.params = clientid=%c,username=%u,password=%P

    auth.http.super_req = http://127.0.0.1:8080/mqtt/superuser
    auth.http.super_req.method = post
    auth.http.super_req.params = clientid=%c,username=%u

启用 HTTP 认证插件::

    ./bin/emqx_ctl plugins load emqx_auth_http

------------
JWT 插件认证
------------

etc/plugins/emqx_auth_jwt.conf 配置 'secret', 'pubkey':

.. code-block:: properties

    ##--------------------------------------------------------------------
    ## JWT Auth Plugin
    ##--------------------------------------------------------------------

    ## HMAC Hash Secret.
    ##
    ## Value: String
    auth.jwt.secret = emqxsecret

    ## RSA or ECDSA public key file.
    ##
    ## Value: File
    ## auth.jwt.pubkey = etc/certs/jwt_public_key.pem

启用 JWT 认证插件::

    ./bin/emqx_ctl plugins load emqx_auth_jwt

--------------
MySQL 插件认证
--------------

通过 MySQL 数据库表认证，可创建如下的 'mqtt_user' 表:

.. code-block:: sql

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

etc/plugins/emqx_auth_mysql.conf 配置 'super_query', 'auth_query', 'password_hash':

.. code-block:: properties

    ## Mysql Server
    auth.mysql.server = 127.0.0.1:3306

    ## Mysql Pool Size
    auth.mysql.pool = 8

    ## Mysql Username
    ## auth.mysql.username = 

    ## Mysql Password
    ## auth.mysql.password = 

    ## Mysql Database
    auth.mysql.database = mqtt

    ## Variables: %u = username, %c = clientid

    ## Authentication Query: select password only
    auth.mysql.auth_query = select password from mqtt_user where username = '%u' limit 1

    ## Password hash.
    ##
    ## Value: plain | md5 | sha | sha256 | bcrypt
    auth.mysql.password_hash = sha256

    ## sha256 with salt prefix
    ## auth.mysql.password_hash = salt,sha256

    ## bcrypt with salt only prefix
    ## auth.mysql.password_hash = salt,bcrypt

    ## sha256 with salt suffix
    ## auth.mysql.password_hash = sha256,salt

    ## pbkdf2 with macfun iterations dklen
    ## macfun: md4, md5, ripemd160, sha, sha224, sha256, sha384, sha512
    ## auth.mysql.password_hash = pbkdf2,sha256,1000,20

    ## %% Superuser Query
    auth.mysql.super_query = select is_superuser from mqtt_user where username = '%u' limit 1

.. NOTE:: 如果系统已有MQTT认证表，可通过配置'auth_query'查询语句集成。

启用 MySQL 认证插件::

    ./bin/emqx_ctl plugins load emqx_auth_mysql

----------------
Postgre 插件认证
----------------

通过 PostgreSQL 数据库表认证，可创建如下的 'mqtt_user' 表:

.. code-block:: sql 

    CREATE TABLE mqtt_user (
      id SERIAL primary key,
      is_superuser boolean,
      username character varying(100),
      password character varying(100),
      salt character varying(40)
    );

etc/plugins/emqx_auth_pgsql.conf 配置 'auth_query'、'password_hash':

.. code-block:: properties

    ## Postgre Server
    auth.pgsql.server = 127.0.0.1:5432

    auth.pgsql.pool = 8

    auth.pgsql.username = root

    #auth.pgsql.password = 

    auth.pgsql.database = mqtt

    auth.pgsql.encoding = utf8

    auth.pgsql.ssl = false

    ## Variables: %u = username, %c = clientid, %a = ipaddress

    ## Authentication Query: select password only
    auth.pgsql.auth_query = select password from mqtt_user where username = '%u' limit 1

    ## Password hash: plain, md5, sha, sha256, pbkdf2, bcrypt
    auth.pgsql.password_hash = sha256

    ## sha256 with salt prefix
    ## auth.pgsql.password_hash = salt sha256

    ## sha256 with salt suffix
    ## auth.pgsql.password_hash = sha256 salt

    ## bcrypt with salt prefix
    ## auth.pgsql.password_hash = salt,bcrypt

    ## pbkdf2 with macfun iterations dklen
    ## macfun: md4, md5, ripemd160, sha, sha224, sha256, sha384, sha512
    ## auth.pgsql.password_hash = pbkdf2,sha256,1000,20

    ## Superuser Query
    auth.pgsql.super_query = select is_superuser from mqtt_user where username = '%u' limit 1

启用 Postgre 认证插件:

.. code-block:: bash

    ./bin/emqx_ctl plugins load emqx_auth_pgsql

--------------
Redis 插件认证
--------------

Redis 认证。MQTT 用户记录存储在 Redis Hash, 键值: "mqtt_user:<Username>"

etc/plugins/emqx_auth_redis.conf 设置 'super_cmd'、'auth_cmd'、'password_hash':

.. code-block:: properties

    ## Redis server address.
    ##
    ## Value: Port | IP:Port
    ##
    ## Redis Server: 6379, 127.0.0.1:6379, localhost:6379, Redis Sentinel: 127.0.0.1:26379

    ## Redis sentinel cluster name.
    ##
    ## Value: String
    ## auth.redis.sentinel = mymaster

    ## Redis pool size.
    ##
    ## Value: Number
    auth.redis.pool = 8

    ## Redis database no.
    ##
    ## Value: Number
    auth.redis.database = 0

    ## Redis password.
    ##
    ## Value: String
    ## auth.redis.password =

    ## Variables: %u = username, %c = clientid

    ## Authentication Query Command
    auth.redis.auth_cmd = HMGET mqtt_user:%u password

    ## Password hash: plain, md5, sha, sha256, pbkdf2, bcrypt
    auth.redis.password_hash = sha256

    ## sha256 with salt prefix
    ## auth.redis.password_hash = salt,sha256

    ## sha256 with salt suffix
    ## auth.redis.password_hash = sha256,salt

    ## bcrypt with salt prefix
    ## auth.redis.password_hash = salt,bcrypt

    ## pbkdf2 with macfun iterations dklen
    ## macfun: md4, md5, ripemd160, sha, sha224, sha256, sha384, sha512
    ## auth.redis.password_hash = pbkdf2,sha256,1000,20

    ## Superuser Query Command
    auth.redis.super_cmd = HGET mqtt_user:%u is_superuser

启用 Redis 认证插件:

.. code-block:: bash

    ./bin/emqx_ctl plugins load emqx_auth_redis

----------------
MongoDB 插件认证
----------------

按 MongoDB 用户集合认证，例如创建 'mqtt_user' 集合::

    {
        username: "user",
        password: "password hash",
        is_superuser: boolean (true, false),
        created: "datetime"
    }

etc/plugins/emqx_auth_mongo.conf 设置 'super_query'、'auth_query':

.. code-block:: properties

    ## MongoDB Topology Type.
    ##
    ## Value: single | unknown | sharded | rs
    auth.mongo.type = single

    ## The set name if type is rs.
    ##
    ## Value: String
    ## auth.mongo.rs_set_name =

    ## MongoDB server list.
    ##
    ## Value: String
    ##
    ## Examples: 127.0.0.1:27017,127.0.0.2:27017...
    auth.mongo.server = 127.0.0.1:27017

    ## Mongo Pool Size
    auth.mongo.pool = 8

    ## MongoDB login user.
    ##
    ## Value: String
    ## auth.mongo.login =

    ## MongoDB password.
    ##
    ## Value: String
    ## auth.mongo.password =

    ## MongoDB AuthSource
    ##
    ## Value: String
    ## Default: mqtt
    ## auth.mongo.auth_source = admin

    ## Mongo Database
    auth.mongo.database = mqtt

    ## auth_query
    auth.mongo.auth_query.collection = mqtt_user

    auth.mongo.auth_query.password_field = password

    auth.mongo.auth_query.password_hash = sha256

    auth.mongo.auth_query.selector = username=%u

    ## super_query
    ## Enable superuser query.
    auth.mongo.super_query = on

    auth.mongo.super_query.collection = mqtt_user

    auth.mongo.super_query.super_field = is_superuser

    auth.mongo.super_query.selector = username=%u

启用 MongoDB 认证插件:

.. code-block:: bash

    ./bin/emqx_ctl plugins load emqx_auth_mongo

.. _acl:

-------------
访问控制(ACL)
-------------

*EMQ X* 消息服务器通过 ACL(Access Control List) 实现 MQTT 客户端访问控制。

ACL 访问控制规则定义::

    允许(Allow)|拒绝(Deny) 谁(Who) 订阅(Subscribe)|发布(Publish) 主题列表(Topics)

MQTT 客户端发起订阅/发布请求时，EMQ X 消息服务器的访问控制模块，会逐条匹配 ACL 规则，直到匹配成功为止::

              ---------              ---------              ---------
    Client -> | Rule1 | --nomatch--> | Rule2 | --nomatch--> | Rule3 | --> Default
              ---------              ---------              ---------
                  |                      |                      |
                match                  match                  match
                 \|/                    \|/                    \|/
            allow | deny           allow | deny           allow | deny

----------------
默认访问控制设置
----------------

*EMQ X* 消息服务器默认访问控制，在 etc/emqx.conf 中设置:

.. code-block:: properties

    ## Allow or deny if no ACL rules matched.
    ##
    ## Value: allow | deny
    acl_nomatch = allow

    ## Default ACL File.
    ##
    ## Value: File Name
    acl_file = etc/acl.conf

ACL 规则定义在 etc/acl.conf，EMQ X 启动时加载到内存:

.. code-block:: erlang

    %% Allow 'dashboard' to subscribe '$SYS/#'
    {allow, {user, "dashboard"}, subscribe, ["$SYS/#"]}.

    %% Allow clients from localhost to subscribe any topics
    {allow, {ipaddr, "127.0.0.1"}, pubsub, ["$SYS/#", "#"]}.

    %% Deny clients to subscribe '$SYS#' and '#'
    {deny, all, subscribe, ["$SYS/#", {eq, "#"}]}.

    %% Allow all by default
    {allow, all}.

-----------------
HTTP 插件访问控制
-----------------

HTTP API 实现访问控制: https://github.com/emqtt/emq_auth_http

配置 etc/plugins/emqx_auth_http.conf, 启用 HTTP 认证插件后:

.. code-block:: properties

    ## 'access' parameter: sub = 1, pub = 2
    auth.http.acl_req = http://127.0.0.1:8080/mqtt/acl
    auth.http.acl_req.method = get
    auth.http.acl_req.params = access=%A,username=%u,clientid=%c,ipaddr=%a,topic=%t

------------------
MySQL 插件访问控制
------------------

MySQL 插件访问控制，通过 mqtt_acl 表定义 ACL 规则:

.. code-block:: sql

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

    INSERT INTO mqtt_acl (id, allow, ipaddr, username, clientid, access, topic)
    VALUES
        (1,1,NULL,'$all',NULL,2,'#'),
        (2,0,NULL,'$all',NULL,1,'$SYS/#'),
        (3,0,NULL,'$all',NULL,1,'eq #'),
        (5,1,'127.0.0.1',NULL,NULL,2,'$SYS/#'),
        (6,1,'127.0.0.1',NULL,NULL,2,'#'),
        (7,1,NULL,'dashboard',NULL,1,'$SYS/#');

etc/plugins/emqx_auth_mysql.conf 配置 'acl_query' 与 'acl_nomatch':

.. code-block:: properties

    ## ACL Query Command
    ## Variables:
    ##  - %a: ipaddr
    ##  - %u: username
    ##  - %c: clientid
    auth.mysql.acl_query = select allow, ipaddr, username, clientid, access, topic from mqtt_acl where ipaddr = '%a' or username = '%u' or username = '$all' or clientid = '%c'

--------------------
Postgre 插件访问控制
--------------------

PostgreSQL 插件访问控制，通过 mqtt_acl 表定义 ACL 规则:

.. code-block:: sql

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

etc/plugins/emqx_auth_pgsql.conf 设置 'acl_query' 与 'acl_nomatch':

.. code-block:: properties

    ## ACL Query. Comment this query, the acl will be disabled.
    ## Variables:
    ##  - %a: ipaddress
    ##  - %u: username
    ##  - %c: clientid
    auth.pgsql.acl_query = select allow, ipaddr, username, clientid, access, topic from mqtt_acl where ipaddr = '%a' or username = '%u' or username = '$all' or clientid = '%c'

------------------
Redis 插件访问控制
------------------

Redis Hash 存储一个 MQTT 客户端的访问控制规则::

    HSET mqtt_acl:<username> topic1 1
    HSET mqtt_acl:<username> topic2 2
    HSET mqtt_acl:<username> topic3 3

etc/plugins/emqx_auth_redis.conf 配置 'acl_cmd' 与 'acl_nomatch':

.. code-block:: properties

    ## ACL Query Command
    auth.redis.acl_cmd = HGETALL mqtt_acl:%u

--------------------
MongoDB 插件访问控制
--------------------

MongoDB 数据库创建 `mqtt_acl` 集合::

    {
        username: "username",
        clientid: "clientid",
        publish: ["topic1", "topic2", ...],
        subscribe: ["subtop1", "subtop2", ...],
        pubsub: ["topic/#", "topic1", ...]
    }

`mqtt_acl` 集合插入数据，例如::

    db.mqtt_acl.insert({username: "test", publish: ["t/1", "t/2"], subscribe: ["user/%u", "client/%c"]})
    db.mqtt_acl.insert({username: "admin", pubsub: ["#"]})

etc/plugins/emqx_auth_mongo.conf 配置 'acl_query' 与 'acl_nomatch':

.. code-block:: properties

    ## acl_query
    auth.mongo.acl_query.collection = mqtt_user

    auth.mongo.acl_query.selector = username=%u

-------------
MQTT 发布订阅
-------------

MQTT 是为移动互联网、物联网设计的轻量发布订阅模式的消息服务器:

.. image:: ./_static/images/pubsub_concept.png

*EMQ X* 消息服务器安装启动后，任何设备或终端的 MQTT 客户端，可通过 MQTT 协议连接到服务器，发布订阅消息方式互通。

MQTT 协议客户端库: https://github.com/mqtt/mqtt.github.io/wiki/libraries

例如，mosquitto_sub/pub 命令行发布订阅消息::

    mosquitto_sub -t topic -q 2
    mosquitto_pub -t topic -q 1 -m "Hello, MQTT!"

MQTT V5.0 版本协议规范: http://docs.oasis-open.org/mqtt/mqtt/v5.0/mqtt-v5.0.html

*EMQ X* 消息服务器的 MQTT 协议 TCP 监听器，可在 etc/emqx.conf 文件中设置:

.. code-block:: properties

    ## TCP Listener: 1883, 127.0.0.1:1883, ::1:1883
    listener.tcp.external = 0.0.0.0:1883

    ## Size of acceptor pool
    listener.tcp.external.acceptors = 8

    ## Maximum number of concurrent clients
    listener.tcp.external.max_connections = 1024000
    ## Maximum external connections per second.
    ##
    ## Value: Number
    listener.tcp.external.max_conn_rate = 1000

MQTT/SSL 监听器，缺省端口8883:

.. code-block:: properties

    ## SSL Listener: 8883, 127.0.0.1:8883, ::1:8883
    listener.ssl.external = 8883

    ## Size of acceptor pool
    listener.ssl.external.acceptors = 16

    ## Maximum number of concurrent clients
    listener.ssl.external.max_connections = 102400

    ## Maximum MQTT/SSL connections per second.
    ##
    ## Value: Number
    listener.ssl.external.max_conn_rate = 500

.. _http_publish:

-------------
HTTP 发布接口
-------------

*EMQ X* 消息服务器提供了一个 HTTP 发布接口，应用服务器或 Web 服务器可通过该接口发布 MQTT 消息::

    HTTP POST http://host:8080/mqtt/publish

Web 服务器例如 PHP/Java/Python/NodeJS 或 Ruby on Rails，可通过 HTTP POST 请求发布 MQTT 消息:

.. code-block:: bash

    curl -v --basic -u user:passwd -H "Content-Type: application/json" -d '{"qos":1, "retain": false, "topic":"world", "payload":"test" , "client_id": "C_1492145414740"}'  -k http://localhost:8080/api/v3/mqtt/publish

HTTP 接口参数:

+---------+----------------------+
| 参数    | 说明                 |
+=========+======================+
| client  | MQTT 客户端 ID       |
+---------+----------------------+
| qos     | QoS: 0 | 1 | 2       |
+---------+----------------------+
| retain  | Retain: true | false |
+---------+----------------------+
| topic   | 主题(Topic)          |
+---------+----------------------+
| message | 消息载荷             |
+---------+----------------------+

.. NOTE::

    HTTP 发布接口采用 Basic 认证

.. NOTE::

    该接口在 v3.0-beta.1 版本变更为: 'api/v3/mqtt/publish', 详见文档: :doc:`/rest`

-------------------
MQTT WebSocket 连接
-------------------

*EMQ X* 消息服务器支持 MQTT WebSocket 连接，Web 浏览器可直接通过 MQTT 协议连接服务器:

+-------------------------+----------------------------+
| WebSocket URI:          | ws(s)://host:8083/mqtt     |
+-------------------------+----------------------------+
| Sec-WebSocket-Protocol: | 'mqttv3.1' or 'mqttv3.1.1' |
+-------------------------+----------------------------+

Dashboard 插件提供了一个 MQTT WebSocket 连接的测试页面::

    http://127.0.0.1:18083/websocket.html

*EMQ X* 通过内嵌的 HTTP 服务器，实现 MQTT/WebSocket，etc/emqx.conf 设置:

.. code-block:: properties

    ## MQTT/WebSocket Listener
    listener.ws.external = 8083
    listener.ws.external.acceptors = 4
    ## Maximum number of concurrent MQTT/WebSocket connections.
    ##
    ## Value: Number
    listener.ws.external.max_connections = 102400

    ## Maximum MQTT/WebSocket connections per second.
    ##
    ## Value: Number
    listener.ws.external.max_conn_rate = 1000

.. _sys_topic:

-------------
$SYS-系统主题
-------------

*EMQ X* 消息服务器周期性发布自身运行状态、MQTT 协议统计、客户端上下线状态到 `$SYS/` 开头系统主题。

$SYS 主题路径以 "$SYS/brokers/{node}/" 开头，'${node}' 是 Erlang 节点名称::

    $SYS/brokers/emqttd@127.0.0.1/version

    $SYS/brokers/emqttd@host2/uptime

.. NOTE:: 默认只允许 localhost 的 MQTT 客户端订阅 $SYS 主题，可通过 etc/acl.config 修改访问控制规则。

$SYS 系统消息发布周期，通过 etc/emqx.conf 配置:

.. code-block:: properties

    ## System interval of publishing $SYS messages.
    ##
    ## Value: Duration
    ## Default: 1m, 1 minute
    broker.sys_interval = 1m

.. _sys_brokers:

服务器版本、启动时间与描述消息
------------------------------

+--------------------------------+-----------------------+
| 主题                           | 说明                  |
+================================+=======================+
| $SYS/brokers                   | 集群节点列表          |
+--------------------------------+-----------------------+
| $SYS/brokers/${node}/version   | EMQ 服务器版本        |
+--------------------------------+-----------------------+
| $SYS/brokers/${node}/uptime    | EMQ 服务器启动时间    |
+--------------------------------+-----------------------+
| $SYS/brokers/${node}/datetime  | EMQ 服务器时间        |
+--------------------------------+-----------------------+
| $SYS/brokers/${node}/sysdescr  | EMQ 服务器描述        |
+--------------------------------+-----------------------+

.. _sys_clients:

MQTT 客户端上下线状态消息
-------------------------

$SYS 主题前缀: $SYS/brokers/${node}/clients/

+--------------------------+--------------------------------------------+------------------------------------+
| 主题(Topic)              | 数据(JSON)                                 | 说明                               |
+==========================+============================================+====================================+
| ${clientid}/connected    | {ipaddress: "127.0.0.1", username: "test", | Publish when a client connected    |
|                          |  session: false, version: 3, connack: 0,   |                                    |
|                          |  ts: 1432648482}                           |                                    |
+--------------------------+--------------------------------------------+------------------------------------+
| ${clientid}/disconnected | {reason: "keepalive_timeout",              | Publish when a client disconnected |
                           |  username: "test", ts: 1432749431}         |                                    |
+--------------------------+--------------------------------------------+------------------------------------+

'connected' 消息 JSON 数据:

.. code-block:: json

    {
        clientid:    "test"
        username:    "test",
        ipaddress:   "127.0.0.1",
        clean_start: true,
        proto_ver:   4,
        proto_name:  "MQTT",
        keepalive:   60,
        connack:   0,
        ts:        1432648482
    }

'disconnected' 消息 JSON 数据:

.. code-block:: json

    {
        clientid:   "test"
        username:   "test",
        reason:     "normal",
        ts:         1432648486
    }

.. _sys_stats:

Statistics - 系统统计消息
--------------------------

系统主题前缀: $SYS/brokers/${node}/stats/

Clients - 客户端统计
....................

+---------------------+---------------------------------------------+
| 主题(Topic)         | 说明                                        |
+---------------------+---------------------------------------------+
| connections/count   | 当前客户端总数                              |
+---------------------+---------------------------------------------+
| connections/max     | 最大客户端数量                              |
+---------------------+---------------------------------------------+

Sessions - 会话统计
...................

+-----------------------------+---------------------------------------------+
| 主题(Topic)                 | 说明                                        |
+-----------------------------+---------------------------------------------+
| sessions/count              | 当前会话总数                                |
+-----------------------------+---------------------------------------------+
| sessions/max                | 最大会话数量                                |
+-----------------------------+---------------------------------------------+
| sessions/persistent/count   | 当前持久会话总数                            |
+-----------------------------+---------------------------------------------+
| sessions/persistent/max     | 最大持久会话数量                            |
+-----------------------------+---------------------------------------------+

Subscriptions - 订阅统计
........................

+---------------------------------+---------------------------------------------+
| 主题(Topic)                     | 说明                                        |
+---------------------------------+---------------------------------------------+
| subscriptions/shared/max        | 当前共享订阅最大数量                        |
+---------------------------------+---------------------------------------------+
| subscriptions/max               | 最大订阅数量                                |
+---------------------------------+---------------------------------------------+
| subscribers/max                 | 最大订阅者总数                              |
+---------------------------------+---------------------------------------------+
| subscriptions/shared/count      | 当前共享订阅总数                            |
+---------------------------------+---------------------------------------------+
| subscriptions/count             | 当前订阅总数                                |
+---------------------------------+---------------------------------------------+
| subscribers/count               | 当前订阅者数量                              |
+---------------------------------+---------------------------------------------+

Topics - 主题统计
................

+---------------------+---------------------------------------------+
| 主题(Topic)         | 说明                                        |
+---------------------+---------------------------------------------+
| topics/count        | 当前 Topic 总数                             |
+---------------------+---------------------------------------------+
| topics/max          | 最大 Topic 数量                             |
+---------------------+---------------------------------------------+

Retained - 保留标志统计
.......................

+---------------------+---------------------------------------------+
| 主题(Topic)         | 说明                                        |
+---------------------+---------------------------------------------+
| retained/count      | 当前 Retained 总数                          |
+---------------------+---------------------------------------------+
| retained/max        | 最大 Retained 数量                          |
+---------------------+---------------------------------------------+

Routes - 路由统计
.................

+---------------------+---------------------------------------------+
| 主题(Topic)         | 说明                                        |
+---------------------+---------------------------------------------+
| routes/count        | 当前 Routes 总数                            |
+---------------------+---------------------------------------------+
| routes/max          | 最大 Routes 数量                            |
+---------------------+---------------------------------------------+

Metrics - 收发流量/报文/消息统计
--------------------------------

系统主题(Topic)前缀: $SYS/brokers/${node}/metrics/

收发流量统计
............

+---------------------+---------------------------------------------+
| 主题(Topic)         | 说明                                        |
+---------------------+---------------------------------------------+
| bytes/received      | 累计接收流量                                |
+---------------------+---------------------------------------------+
| bytes/sent          | 累计发送流量                                |
+---------------------+---------------------------------------------+

MQTT报文收发统计
................

+--------------------------+---------------------------------------------+
| 主题(Topic)              | 说明                                        |
+--------------------------+---------------------------------------------+
| packets/received         | 累计接收 MQTT 报文                          |
+--------------------------+---------------------------------------------+
| packets/sent             | 累计发送 MQTT 报文                          |
+--------------------------+---------------------------------------------+
| packets/connect          | 累计接收 MQTT CONNECT 报文                  |
+--------------------------+---------------------------------------------+
| packets/connack          | 累计发送 MQTT CONNACK 报文                  |
+--------------------------+---------------------------------------------+
| packets/publish/received | 累计接收 MQTT PUBLISH 报文                  |
+--------------------------+---------------------------------------------+
| packets/publish/sent     | 累计发送 MQTT PUBLISH 报文                  |
+--------------------------+---------------------------------------------+
| packets/puback/received  | 累计接收 MQTT PUBACK 报文                   |
+--------------------------+---------------------------------------------+
| packets/puback/sent      | 累计发送 MQTT PUBACK 报文                   |
+--------------------------+---------------------------------------------+
| packets/puback/missed    | 累计丢失 MQTT PUBACK 报文                   |
+--------------------------+---------------------------------------------+
| packets/pubrec/received  | 累计接收 MQTT PUBREC 报文                   |
+--------------------------+---------------------------------------------+
| packets/pubrec/sent      | 累计发送 MQTT PUBREC 报文                   |
+--------------------------+---------------------------------------------+
| packets/pubrec/missed    | 累计丢失 MQTT PUBREC 报文                   |
+--------------------------+---------------------------------------------+
| packets/pubrel/received  | 累计接收 MQTT PUBREL 报文                   |
+--------------------------+---------------------------------------------+
| packets/pubrel/sent      | 累计发送 MQTT PUBREL 报文                   |
+--------------------------+---------------------------------------------+
| packets/pubrel/missed    | 累计丢失 MQTT PUBREL 报文                   |
+--------------------------+---------------------------------------------+
| packets/pubcomp/received | 累计接收 MQTT PUBCOMP 报文                  |
+--------------------------+---------------------------------------------+
| packets/pubcomp/sent     | 累计发送 MQTT PUBCOMP 报文                  |
+--------------------------+---------------------------------------------+
| packets/pubcomp/missed   | 累计丢失 MQTT PUBCOMP 报文                  |
+--------------------------+---------------------------------------------+
| packets/subscribe        | 累计接收 MQTT SUBSCRIBE 报文                |
+--------------------------+---------------------------------------------+
| packets/suback           | 累计发送 MQTT SUBACK 报文                   |
+--------------------------+---------------------------------------------+
| packets/unsubscribe      | 累计接收 MQTT UNSUBSCRIBE 报文              |
+--------------------------+---------------------------------------------+
| packets/unsuback         | 累计发送 MQTT UNSUBACK 报文                 |
+--------------------------+---------------------------------------------+
| packets/pingreq          | 累计接收 MQTT PINGREQ 报文                  |
+--------------------------+---------------------------------------------+
| packets/pingresp         | 累计发送 MQTT PINGRESP 报文                 |
+--------------------------+---------------------------------------------+
| packets/disconnect       | 累计接收 MQTT DISCONNECT 报文               |
+--------------------------+---------------------------------------------+
| packets/auth             | 累计接收Auth 报文                           |
+--------------------------+---------------------------------------------+

MQTT 消息收发统计
..................

+--------------------------+---------------------------------------------+
| 主题(Topic)              | 说明                                        |
+--------------------------+---------------------------------------------+
| messages/received        | 累计接收消息                                |
+--------------------------+---------------------------------------------+
| messages/sent            | 累计发送消息                                |
+--------------------------+---------------------------------------------+
| messages/expired         | 累计发送消息                                |
+--------------------------+---------------------------------------------+
| messages/retained        | Retained 消息总数                           |
+--------------------------+---------------------------------------------+
| messages/dropped         | 丢弃消息总数                                |
+--------------------------+---------------------------------------------+
| messages/forward         | 节点转发消息总数                            |
+--------------------------+---------------------------------------------+
| messages/qos0/received   | 累计接受QoS0消息                            |
+--------------------------+---------------------------------------------+
| messages/qos0/sent       | 累计发送QoS0消息                            |
+--------------------------+---------------------------------------------+
| messages/qos1/received   | 累计接受QoS1消息                            |
+--------------------------+---------------------------------------------+
| messages/qos1/sent       | 累计发送QoS1消息                            |
+--------------------------+---------------------------------------------+
| messages/qos2/received   | 累计接受QoS2消息                            |
+--------------------------+---------------------------------------------+
| messages/qos2/sent       | 累计发送QoS2消息                            |
+--------------------------+---------------------------------------------+
| messages/qos2/expired    | QoS2过期消息总数                            |
+--------------------------+---------------------------------------------+
| messages/qos2/dropped    | QoS2丢弃消息总数                            |
+--------------------------+---------------------------------------------+

.. _sys_alarms:

Alarms - 系统告警
-----------------

系统主题(Topic)前缀: $SYS/brokers/${node}/alarms/

+------------------+------------------+
| 主题(Topic)      | 说明             |
+------------------+------------------+
| ${alarmId}/alert | 新产生告警       |
+------------------+------------------+
| ${alarmId}/clear | 清除告警         |
+------------------+------------------+

.. _sys_sysmon:

Sysmon - 系统监控
-----------------

系统主题(Topic)前缀: $SYS/brokers/${node}/sysmon/

+------------------+--------------------+
| 主题(Topic)      | 说明               |
+------------------+--------------------+
| long_gc          | GC 时间过长警告    |
+------------------+--------------------+
| long_schedule    | 调度时间过长警告   |
+------------------+--------------------+
| large_heap       | Heap 内存占用警告  |
+------------------+--------------------+
| busy_port        | Port 忙警告        |
+------------------+--------------------+
| busy_dist_port   | Dist Port 忙警告   |
+------------------+--------------------+

.. _trace:

----
追踪
----

EMQ X 消息服务器支持追踪来自某个客户端(Client)的全部报文，或者发布到某个主题(Topic)的全部消息。

追踪客户端(Client):

.. code-block:: bash

    ./bin/emqx_ctl trace client "clientid" "trace_clientid.log"

追踪主题(Topic):

.. code-block:: bash

    ./bin/emqx_ctl trace topic "topic" "trace_topic.log"

查询追踪:

.. code-block:: bash

    ./bin/emqx_ctl trace list

停止追踪:

.. code-block:: bash

    ./bin/emqx_ctl trace client "clientid" off

    ./bin/emqx_ctl trace topic "topic" off

.. _emqx_auth_clientid: https://github.com/emqtt/emq_auth_clientid
.. _emqx_auth_username: https://github.com/emqtt/emq_auth_username
.. _emqx_auth_ldap:     https://github.com/emqtt/emq_auth_ldap
.. _emqx_auth_http:     https://github.com/emqtt/emq_auth_http
.. _emqx_auth_mysql:    https://github.com/emqtt/emq_auth_mysql
.. _emqx_auth_pgsql:    https://github.com/emqtt/emq_auth_pgsql
.. _emqx_auth_redis:    https://github.com/emqtt/emq_auth_redis
.. _emqx_auth_mongo:    https://github.com/emqtt/emq_auth_mongo
.. _emqx_auth_jwt:      https://github.com/emqtt/emq-auth-jwt
