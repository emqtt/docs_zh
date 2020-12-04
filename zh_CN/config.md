# 配置说明(Configuration)

emqttd 消息服务器通过 etc/目录下配置文件进行设置，主要配置文件包括:

| 配置文件           | 说明                      |
| ------------------ | ------------------------- |
| etc/vm.args        | Erlang 虚拟机的参数设置   |
| etc/emqttd.config  | emqttd 消息服务器参数设置 |
| etc/acl.config     | ACL(访问控制规则)设置     |
| etc/clients.config | 基于 ClientId 认证设置    |
| etc/rewrite.config | Rewrite 扩展模块规则配置  |
| etc/ssl/\*         | SSL 证书设置              |

## etc/vm.args

etc/vm.args 文件设置 Erlang 虚拟机参数:

    ##-------------------------------------------------------------------------
    ## Name of the node
    ##-------------------------------------------------------------------------
    -name emqttd@127.0.0.1

    ## Cookie for distributed erlang
    -setcookie emqttdsecretcookie

    ##-------------------------------------------------------------------------
    ## Flags
    ##-------------------------------------------------------------------------

    ## Heartbeat management; auto-restarts VM if it dies or becomes unresponsive
    ## (Disabled by default..use with caution!)
    ##-heart
    -smp true

    ## Enable kernel poll and a few async threads
    +K true

    ## 12 threads/core.
    +A 48

    ## max process numbers
    +P 8192

    ## Sets the maximum number of simultaneously existing ports for this system
    +Q 8192

    ## max atom number
    ## +t

    ## Set the distribution buffer busy limit (dist_buf_busy_limit) in kilobytes.
    ## Valid range is 1-2097151. Default is 1024.
    ## +zdbbl 8192

    ## CPU Schedulers
    ## +sbt db

    ##-------------------------------------------------------------------------
    ## Env
    ##-------------------------------------------------------------------------

    ## Increase number of concurrent ports/sockets, deprecated in R17
    -env ERL_MAX_PORTS 8192

    -env ERTS_MAX_PORTS 8192

    -env ERL_MAX_ETS_TABLES 1024

    ## Tweak GC to run more often
    -env ERL_FULLSWEEP_AFTER 1000

etc/vm.args 中两个最重要的参数:

| +P  | Erlang 虚拟机允许的最大进程数，一个 MQTT 连接会消耗 2 个 Erlang 进程，所以参数值 > 最大连接数 \* 2 |
| --- | -------------------------------------------------------------------------------------------------- |
| +Q  | Erlang 虚拟机允许的最大 Port 数量，一个 MQTT 连接消耗 1 个 Port，所以参数值 > 最大连接数           |

etc/vm.args 设置 Erlang 节点名、节点间通信 Cookie:

    -name emqttd@127.0.0.1

    ## Cookie for distributed erlang
    -setcookie emqttdsecretcookie

::: tip
Erlang/OTP 平台应用多由分布的 Erlang 节点(进程)组成，每个 Erlang 节点(进程)需指配一个节点名，用于节点间通信互访。
所有互相通信的 Erlang 节点(进程)间通过一个共用的 Cookie 进行安全认证。
:::

## etc/emqttd.config

etc/emqttd.config 是消息服务器的核心配置文件。Erlang 程序由多个应用(application)组成，每个应用(application)有自身的环境参数，

启动时候通过 etc/emqttd.config 文件加载。

etc/emqttd.config 文件采用的是 Erlang 数据格式，kernel, sasl,
emqttd 是 Erlang 应用(application)名称，'[]'内是应用的环境参数列表。

    [{kernel, [
        {start_timer, true},
        {start_pg2, true}
     ]},
     {sasl, [
        {sasl_error_logger, {file, "log/emqttd_sasl.log"}}
     ]},

     ...

     {emqttd, [
        ...
     ]}
    ].

emqttd.config 格式简要说明:

1. \[ ] : 列表，逗号分隔元素
2. \{ } : 元组，配置元组一般两个元素{Env, Value}
3. \% : 注释

### 日志级别设置

emqttd 消息服务器日志由 lager 应用(application)提供，日志相关设置在 lager 应用段落:

    {lager, [
      ...
    ]},

产品环境下默认只开启 error 日志，日志输出到 logs/emqttd_error.log 文件。'handlers'段落启用其他级别日志:

    {handlers, [
        {lager_console_backend, info},

        {lager_file_backend, [
            {formatter_config, [time, " ", pid, " [",severity,"] ", message, "\n"]},
            {file, "log/emqttd_info.log"},
            {level, info},
            {size, 104857600},
            {date, "$D0"},
            {count, 30}
        ]},

        {lager_file_backend, [
            {formatter_config, [time, " ", pid, " [",severity,"] ", message, "\n"]},
            {file, "log/emqttd_error.log"},
            {level, error},
            {size, 104857600},
            {date, "$D0"},
            {count, 30}
        ]}
    ]}

::: warning
过多日志打印严重影响服务器性能，产品环境下建议开启 error 级别日志。
:::

### 消息服务器参数配置

emqttd 消息服务器参数设置在 emqttd 应用段落，包括用户认证与访问控制设置，MQTT 协议、会话、队列设置，扩展模块设置，TCP 服务监听器设置:

    {emqttd, [
       %% 用户认证与访问控制设置
       {access, [
           ...
       ]},
       %% MQTT连接、协议、会话、队列设置
       {mqtt, [
           ...
       ]},
       %% 消息服务器设置
       {broker, [
           ...
       ]},
       %% 扩展模块设置
       {modules, [
           ...
       ]},
       %% 插件目录设置
       {plugins, [
           ...
       ]},

       %% TCP监听器设置
       {listeners, [
           ...
       ]},

       %% Erlang虚拟机监控设置
       {sysmon, [
       ]}
    ]}

### access 用户认证设置

emqttd 消息服务器认证由一系列认证模块(module)或插件(plugin)提供，系统默认支持用户名、ClientID、LDAP、匿名(anonymouse)认证模块:

    %% Authetication. Anonymous Default
    {auth, [
        %% Authentication with username, password
        %% Add users: ./bin/emqttd_ctl users add Username Password
        %% {username, [{"test", "public"}]},

        %% Authentication with clientid
        % {clientid, [{password, no}, {file, "etc/clients.config"}]},

        %% Authentication with LDAP
        % {ldap, [
        %    {servers, ["localhost"]},
        %    {port, 389},
        %    {timeout, 30},
        %    {user_dn, "uid=$u,ou=People,dc=example,dc=com"},
        %    {ssl, fasle},
        %    {sslopts, [
        %        {"certfile", "ssl.crt"},
        %        {"keyfile", "ssl.key"}]}
        % ]},

        %% Allow all
        {anonymous, []}
    ]},

系统默认采用匿名认证(anonymous)，通过删除注释可开启其他认证方式。同时开启的多个认证模块组成认证链:

               ----------------           ----------------           ------------
    Client --> | Username认证 | -ignore-> | ClientID认证 | -ignore-> | 匿名认证 |
               ----------------           ----------------           ------------
                      |                         |                         |
                     \|/                       \|/                       \|/
                allow | deny              allow | deny              allow | deny

::: tip
emqttd 消息服务器还提供了 MySQL、PostgreSQL、Redis、MongoDB 认证插件， 认证插件加载后认证模块失效。
:::

#### 用户名密码认证

    {username, [{test1, "passwd1"}, {test2, "passwd2"}]},

两种方式添加用户:

1. 直接在[]中明文配置默认用户:
   [{test1, "passwd1"}, {test2, "passwd2"}]
2. 通过'./bin/emqttd_ctl'管理命令行添加用户:
   $ ./bin/emqttd_ctl users add \<Username> \<Password>

#### ClientID 认证

    {clientid, [{password, no}, {file, "etc/clients.config"}]},

etc/clients.config 文件中添加 ClientID:

    testclientid0
    testclientid1 127.0.0.1
    testclientid2 192.168.0.1/24

#### LDAP 认证

    {ldap, [
       {servers, ["localhost"]},
       {port, 389},
       {timeout, 30},
       {user_dn, "uid=$u,ou=People,dc=example,dc=com"},
       {ssl, fasle},
       {sslopts, [
           {certfile, "ssl.crt"},
           {keyfile, "ssl.key"}]}
    ]},

#### 匿名认证

默认开启。允许任意客户端登录:

    {anonymous, []}

### access 用户访问控制(ACL)

emqttd 消息服务器支持基于 etc/acl.config 文件或 MySQL、PostgreSQL 插件的访问控制规则。

默认开启基于 etc/acl.config 文件的访问控制:

    %% ACL config
    {acl, [
        %% Internal ACL module
        {internal,  [{file, "etc/acl.config"}, {nomatch, allow}]}
    ]}

etc/acl.config 访问控制规则定义:

    允许|拒绝  用户|IP地址|ClientID  发布|订阅  主题列表

etc/acl.config 默认访问规则设置:

    {allow, {user, "dashboard"}, subscribe, ["$SYS/#"]}.

    {allow, {ipaddr, "127.0.0.1"}, pubsub, ["$SYS/#", "#"]}.

    {deny, all, subscribe, ["$SYS/#", {eq, "#"}]}.

    {allow, all}.

::: tip
默认规则只允许本机用户订阅'$SYS/#'与'#'
:::

emqttd 消息服务器接收到 MQTT 客户端发布(PUBLISH)或订阅(SUBSCRIBE)请求时，会逐条匹配 ACL 访问控制规则，

直到匹配成功返回 allow 或 deny。

### MQTT 报文(Packet)尺寸与 ClientID 长度限制

`packet` 段落设置最大报文尺寸、最大客户端 ID 长度:

    {packet, [

        %% ClientID长度, 默认1024
        {max_clientid_len, 1024},

        %% 最大报文长度，默认64K
        {max_packet_size,  65536}
    ]},

### MQTT 客户端(Client)连接闲置时间

'client'段落设置客户端最大允许闲置时间(Socket 连接建立，但未发送 CONNECT 报文):

    {client, [
        %% 单位: 秒
        {idle_timeout, 10}
    ]},

### MQTT 会话(Session)参数设置

'session'段落设置 MQTT 会话参数:

    {session, [
        %% Max number of QoS 1 and 2 messages that can be “in flight” at one time.
        %% 0 means no limit
        {max_inflight, 100},

        %% Retry interval for redelivering QoS1/2 messages.
        {unack_retry_interval, 20},

        %% Awaiting PUBREL Timeout
        {await_rel_timeout, 20},

        %% Max Packets that Awaiting PUBREL, 0 means no limit
        {max_awaiting_rel, 0},

        %% Statistics Collection Interval(seconds)
        {collect_interval, 20},

        %% Expired after 2 day (unit: minute)
        {expired_after, 2880}

    ]},

会话参数详细说明:

| max_inflight         | 飞行窗口。最大允许同时下发的 Qos1/2 报文数，0 表示没有限制。 窗口值越大，吞吐越高；窗口值越小，消息顺序越严格 |
| -------------------- | ------------------------------------------------------------------------------------------------------------- |
| unack_retry_interval | 下发 QoS1/2 消息未收到 PUBACK 响应的重试间隔                                                                  |
| await_rel_timeout    | 收到 QoS2 消息，等待 PUBREL 报文超时时间                                                                      |
| max_awaiting_rel     | 最大等待 PUBREL 的 QoS2 报文数                                                                                |
| collect_interval     | 采集会话统计数据间隔，默认 0 表示关闭统计                                                                     |
| expired_after        | 持久会话到期时间，从客户端断开算起，单位：分钟                                                                |

### MQTT 会话消息队列(MQueue)设置

emqttd 消息服务器会话通过队列缓存 Qos1/Qos2 消息:

1. 持久会话(Session)的离线消息
2. 飞行窗口满而延迟下发的消息

队列参数设置:

    {queue, [
        %% simple | priority
        {type, simple},

        %% Topic Priority: 0~255, Default is 0
        %% {priority, [{"topic/1", 10}, {"topic/2", 8}]},

        %% Max queue length. Enqueued messages when persistent client disconnected,
        %% or inflight window is full.
        {max_length, infinity},

        %% Low-water mark of queued messages
        {low_watermark, 0.2},

        %% High-water mark of queued messages
        {high_watermark, 0.6},

        %% Queue Qos0 messages?
        {queue_qos0, true}
    ]}

队列参数说明:

| type           | 队列类型。simple: 简单队列，priority: 优先级队列 |
| -------------- | ------------------------------------------------ |
| priority       | 主题(Topic)队列优先级设置                        |
| max_length     | 队列长度, infinity 表示不限制                    |
| low_watermark  | 解除告警水位线                                   |
| high_watermark | 队列满告警水位线                                 |
| queue_qos0     | 是否缓存 QoS0 消息                               |

### broker 消息服务器参数

'broker'段落设置消息服务器内部模块参数。

sys_interval 设置系统发布$SYS 消息周期:

    {sys_interval, 60},

### broker retained 消息设置

retained 设置 MQTT retain 消息处理参数:

    {retained, [
        %% retain消息过期时间，单位: 秒
        {expired_after, 0},

        %% 最大retain消息数量
        {max_message_num, 100000},

        %% retain消息payload最大尺寸
        {max_playload_size, 65536}
    ]},

| expired_after   | Retained 消息过期时间，0 表示永不过期 |
| --------------- | ------------------------------------- |
| max_message_num | 最大存储的 Retained 消息数量          |
| max_packet_size | Retained 消息 payload 最大允许尺寸    |

### broker pubsub 路由设置

发布/订阅(Pub/Sub)路由模块参数:

    {pubsub, [
        %% PubSub Erlang进程池
        {pool_size, 8},

        %% 订阅存储类型，true: 存储, false: 不存储
        {subscription, true},

        %% 路由老化时间
        {route_aging, 5}
    ]},

### broker bridge 桥接参数

桥接参数设置:

    {bridge, [
        %% 最大缓存桥接消息数
        {max_queue_len, 10000},

        %% 桥接节点宕机检测周期，单位: 秒
        {ping_down_interval, 1}
    ]}

### modules 扩展模块设置

emqtt 消息服务器支持简单的扩展模块，用于定制服务器功能。默认支持 presence、subscription、rewrite 模块。

'presence'扩展模块会向$SYS 主题(Topic)发布客户端上下线消息:

    {presence, [{qos, 0}]},

'subscription'扩展模块支持客户端上线时，自动订阅或恢复订阅某些主题(Topic):

    %% Subscribe topics automatically when client connected
    {subscription, [
        %% Subscription from stored table
        stored,

        %% $u will be replaced with username
        {"$Q/username/$u", 1},

        %% $c will be replaced with clientid
        {"$Q/client/$c", 1}
    ]}

'rewrite'扩展模块支持重写主题(Topic)路径, 重写规则定义在 etc/rewrite.config 文件:

    %% Rewrite rules
    %% {rewrite, [{file, "etc/rewrite.config"}]}

关于扩展模块详细介绍，请参考\<用户指南>文档。

### plugins 插件目录设置

    {plugins, [
        %% Plugin App Library Dir
        {plugins_dir, "./plugins"},

        %% File to store loaded plugin names.
        {loaded_file, "./data/loaded_plugins"}
    ]},

### listeners 监听器设置

emqttd 消息服务器开启的 MQTT 协议、HTTP 协议服务端，可通过 listener 设置 TCP 服务端口、最大允许连接数等参数。

emqttd 消息服务器默认开启的 TCP 服务端口包括:

| 1883 | MQTT 协议端口                  |
| ---- | ------------------------------ |
| 8883 | MQTT(SSL)端口                  |
| 8083 | MQTT(WebSocket), HTTP API 端口 |

    {listeners, [

        {mqtt, 1883, [
            %% Size of acceptor pool
            {acceptors, 16},

            %% Maximum number of concurrent clients
            {max_clients, 8192},

            %% Socket Access Control
            {access, [{allow, all}]},

            %% Connection Options
            {connopts, [
                %% Rate Limit. Format is 'burst, rate', Unit is KB/Sec
                %% {rate_limit, "100,10"} %% 100K burst, 10K rate
            ]},

            %% Socket Options
            {sockopts, [
                %Set buffer if hight thoughtput
                %{recbuf, 4096},
                %{sndbuf, 4096},
                %{buffer, 4096},
                %{nodelay, true},
                {backlog, 1024}
            ]}
        ]},

        {mqtts, 8883, [
            %% Size of acceptor pool
            {acceptors, 4},

            %% Maximum number of concurrent clients
            {max_clients, 512},

            %% Socket Access Control
            {access, [{allow, all}]},

            %% SSL certificate and key files
            {ssl, [{certfile, "etc/ssl/ssl.crt"},
                   {keyfile,  "etc/ssl/ssl.key"}]},

            %% Socket Options
            {sockopts, [
                {backlog, 1024}
                %{buffer, 4096},
            ]}
        ]},
        %% WebSocket over HTTPS Listener
        %% {https, 8083, [
        %%  %% Size of acceptor pool
        %%  {acceptors, 4},
        %%  %% Maximum number of concurrent clients
        %%  {max_clients, 512},
        %%  %% Socket Access Control
        %%  {access, [{allow, all}]},
        %%  %% SSL certificate and key files
        %%  {ssl, [{certfile, "etc/ssl/ssl.crt"},
        %%         {keyfile,  "etc/ssl/ssl.key"}]},
        %%  %% Socket Options
        %%  {sockopts, [
        %%      %{buffer, 4096},
        %%      {backlog, 1024}
        %%  ]}
        %%]},

        %% HTTP and WebSocket Listener
        {http, 8083, [
            %% Size of acceptor pool
            {acceptors, 4},
            %% Maximum number of concurrent clients
            {max_clients, 64},
            %% Socket Access Control
            {access, [{allow, all}]},
            %% Socket Options
            {sockopts, [
                {backlog, 1024}
                %{buffer, 4096},
            ]}
        ]}
    ]},

listener 参数说明:

| acceptors   | TCP Acceptor 池                                             |
| ----------- | ----------------------------------------------------------- |
| max_clients | 最大允许 TCP 连接数                                         |
| access      | 允许访问的 IP 地址段设置，例如: [{allow, "192.168.1.0/24"}] |
| connopts    | 连接限速配置，例如限速 10KB/秒: {rate_limit, "100,10"}      |
| sockopts    | Socket 参数设置                                             |

## etc/acl.config

emqttd 消息服务器默认访问控制规则配置在 etc/acl.config 文件。

访问控制规则采用 Erlang 元组格式，访问控制模块逐条匹配规则:

              ---------              ---------              ---------
    Client -> | Rule1 | --nomatch--> | Rule2 | --nomatch--> | Rule3 | --> Default
              ---------              ---------              ---------
                  |                      |                      |
                match                  match                  match
                 \|/                    \|/                    \|/
            allow | deny           allow | deny           allow | deny

etc/acl.config 文件默认规则设置:

    %% 允许'dashboard'用户订阅 '$SYS/#'
    {allow, {user, "dashboard"}, subscribe, ["$SYS/#"]}.

    %% 允许本机用户发布订阅全部主题
    {allow, {ipaddr, "127.0.0.1"}, pubsub, ["$SYS/#", "#"]}.

    %% 拒绝用户订阅'$SYS#'与'#'主题
    {deny, all, subscribe, ["$SYS/#", {eq, "#"}]}.

    %% 上述规则无匹配，允许
    {allow, all}.

## etc/rewrite.config

Rewrite 扩展模块的规则配置文件，示例配置:

    {topic, "x/#", [
        {rewrite, "^x/y/(.+)$", "z/y/$1"},
        {rewrite, "^x/(.+)$", "y/$1"}
    ]}.

    {topic, "y/+/z/#", [
        {rewrite, "^y/(.+)/z/(.+)$", "y/z/$2"}
    ]}.
