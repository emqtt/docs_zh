.. _bridge:

=================
节点桥接 (Bridge)
=================

.. _bridge_emqx:

----------------
EMQ X 节点间桥接
----------------

所谓 **桥接** 的概念是指 EMQ X 支持将自身某类主题的消息通过某种方式转发到另一个 MQTT Broker

**桥接** 与 **集群** 的不同在于：桥接不会复制主题树与路由表，只根据桥接规则转发 MQTT 消息。

目前 EMQ X 支持的桥接方式有:

- RPC 桥接：RPC 桥接只支持消息转发，不支持订阅远程节点的主题去同步数据
- MQTT 桥接：MQTT 桥接同时支持转发和通过订阅主题来实现数据同步两种方式

其概念如下图所示:

.. image:: ./_static/images/bridge.png

此外 *EMQ X* 消息服务器支持多节点桥接模式互联::

                  ---------                     ---------                     ---------
                  Publisher --> | Node1 | --Bridge Forward--> | Node2 | --Bridge Forward--> | Node3 | --> Subscriber
                  ---------                     ---------                     ---------

在 EMQ X 中，通过修改 ``etc/plugins/emqx_bridge_mqtt.conf`` 来配置 bridge。EMQ X 会根据不同的 name 来区分不同的 bridge。例如::

    ## Bridge address: node name for local bridge, host:port for remote.
    bridge.mqtt.aws.address = 127.0.0.1:1883

该项配置声明了一个名为 ``aws`` 的 bridge 并指定以 MQTT 的方式桥接到 ``127.0.0.1:1883`` 这台 MQTT 服务器

在需要创建多个 bridge 时，可以先复制其全部的配置项，在通过使用不同的 name 来标示（比如 bridge.$name.address 其中 $name 指代的为 bridge 的名称）


接下来俩个小节，表述了如何创建 RPC/MQTT 方式的桥接，并创建一条转发传感器(sensor)主题消息的转发规则。假设在俩台主机上启动了两个 EMQ X 节点：

+---------+---------------------+-----------+
| 名称    | 节点                | MQTT 端口 |
+---------+---------------------+-----------+
| emqx1   | emqx1@192.168.1.1   | 1883      |
+---------+---------------------+-----------+
| emqx2   | emqx2@192.168.1.2   | 1883      |
+---------+---------------------+-----------+


EMQ X 节点 RPC 桥接配置
---------------------------

以下是 RPC 桥接的基本配置，最简单的 RPC 桥接只需要配置以下三项就可以了::

    ## 桥接地址： 使用节点名（nodename@host）则用于 rpc 桥接，使用 host:port 用于 mqtt 连接
    bridge.mqtt.emqx2.address = emqx2@192.168.1.2

    ## 转发消息的主题
    bridge.mqtt.emqx2.forwards = sensor1/#,sensor2/#

    ## 桥接的 mountpoint(挂载点)
    bridge.mqtt.emqx2.mountpoint = bridge/emqx2/${node}/

本地 emqx1 节点接收到的消息如果匹配主题 ``sersor1/#``, ``sensor2/#``, 这些消息会被转发到远程 emqx2 节点的 ``sensor1/#``, ``sensor2/#`` 主题上。

forwards 用于指定主题，转发到本地节点指定 forwards 上的消息都会被转发到远程节点上。

mountpoint 用于在转发消息时加上主题前缀，该配置选项须配合 forwards 使用，转发主题为 `sensor1/hello` 的消息, 到达远程节点时主题为 `bridge/emqx2/emqx1@192.168.1.1/sensor1/hello` 。

RPC 桥接的局限性：

1. emqx 的 RPC 桥接只能将本地的消息转发到远程桥接节点上，无法将远程桥接节点的消息同步到本地节点上；

2. RPC 桥接只能将两个 emqx 桥接在一起，无法桥接 emqx 到其他的 mqtt broker 上。


EMQ X 节点 MQTT 桥接配置
------------------------

emqx 3.0 正式引入了 mqtt bridge，使 emqx 可以桥接任意 mqtt broker ，同时由于 mqtt 协议本身的特性， emqx 可以通过 mqtt bridge 去订阅远程 mqtt broker 的主题，再将远程 mqtt broker 的消息同步到本地。

EMQ X MQTT 的桥接原理: 通过在 emqx broker 上创建一个 mqtt 客户端，mqtt 客户端会连接远程的 mqtt broker，因此在 mqtt bridge 的配置中，需要去设置 emqx 作为 mqtt 客户端连接时所必须用到的字段::

    ## 桥接地址： 使用节点名则用于 rpc 桥接，使用 host:port 用于 mqtt 连接
    bridge.mqtt.emqx2.address = 192.168.1.2:1883

    ## 桥接的协议版本
    ## 枚举值: mqttv3 | mqttv4 | mqttv5
    bridge.mqtt.emqx2.proto_ver = mqttv4

    ## mqtt 客户端的 client_id
    bridge.mqtt.emqx2.client_id = bridge_emq

    ## mqtt 客户端的 clean_start 字段
    ## 注: 有些 MQTT Broker 需要将 clean_start 值设成 `true`
    bridge.mqtt.emqx2.clean_start = true

    ## mqtt 客户端的 username 字段
    bridge.mqtt.emqx2.username = user

    ## mqtt 客户端的 password 字段
    bridge.mqtt.emqx2.password = passwd

    ## mqtt 客户端是否使用 ssl 来连接远程服务器
    bridge.mqtt.emqx2.ssl = off

    ## 客户端 SSL 连接的 CA 证书 (PEM格式)
    bridge.mqtt.emqx2.cacertfile = etc/certs/cacert.pem

    ## 客户端 SSL 连接的 SSL 证书
    bridge.mqtt.emqx2.certfile = etc/certs/client-cert.pem

    ## 客户端 SSL 连接的密钥文件
    bridge.mqtt.emqx2.keyfile = etc/certs/client-key.pem

    ## SSL 加密方式
    bridge.mqtt.emqx2.ciphers = ECDHE-ECDSA-AES256-GCM-SHA384,ECDHE-RSA-AES256-GCM-SHA384

    ## TLS PSK 的密码
    ## 注意 'listener.ssl.external.ciphers' 和 'listener.ssl.external.psk_ciphers' 不能同时配置
    ##
    ## See 'https://tools.ietf.org/html/rfc4279#section-2'.
    ## bridge.mqtt.emqx2.psk_ciphers = PSK-AES128-CBC-SHA,PSK-AES256-CBC-SHA,PSK-3DES-EDE-CBC-SHA,PSK-RC4-SHA

    ## 客户端的心跳间隔
    bridge.mqtt.emqx2.keepalive = 60s

    ## 支持的 TLS 版本
    bridge.mqtt.emqx2.tls_versions = tlsv1.2,tlsv1.1,tlsv1

    ## 转发消息的主题
    bridge.mqtt.emqx2.forwards = sensor1/#,sensor2/#

    ## 桥接的 mountpoint(挂载点)
    bridge.mqtt.emqx2.mountpoint = bridge/emqx2/${node}/

    ## 用于桥接的订阅主题
    bridge.mqtt.emqx2.subscription.1.topic = cmd/topic1

    ## 用于桥接的订阅 qos
    bridge.mqtt.emqx2.subscription.1.qos = 1

    ## 用于桥接的订阅主题
    bridge.mqtt.emqx2.subscription.2.topic = cmd/topic2

    ## 用于桥接的订阅 qos
    bridge.mqtt.emqx2.subscription.2.qos = 1

    ## 桥接的重连间隔
    ## 默认: 30秒
    bridge.mqtt.emqx2.reconnect_interval = 30s

    ## QoS1 消息的重传间隔
    bridge.mqtt.emqx2.retry_interval = 20s

    ## Inflight 大小.
    bridge.mqtt.emqx2.max_inflight_batches = 32


EMQ X 桥接缓存配置
-----------------------

EMQ X 的 bridge 拥有消息缓存机制，缓存机制同时适用于 RPC 桥接和 MQTT 桥接，当 bridge 断开（如网络连接不稳定的情况）时，可将 forwards 主题的消息缓存到本地的消息队列上。等到桥接恢复时，再把消息重新转发到远程节点上。关于缓存队列的配置如下::

    ## emqx_bridge 内部用于 batch 的消息数量
    bridge.mqtt.emqx2.queue.batch_count_limit = 32

    ## emqx_bridge 内部用于 batch 的消息字节数
    bridge.mqtt.emqx2.queue.batch_bytes_limit = 1000MB

    ## 放置 replayq 队列的路径，如果没有在配置中指定该项，那么 replayq
    ## 将会以 `mem-only` 的模式运行，消息不会缓存到磁盘上。
    bridge.mqtt.emqx2.queue.replayq_dir = data/emqx_emqx2_bridge/
    
    ## Replayq 数据段大小
    bridge.mqtt.emqx2.queue.replayq_seg_bytes = 10MB

``bridge.mqtt.emqx2.queue.replayq_dir`` 是用于指定 bridge 存储队列的路径的配置参数。

``bridge.mqtt.emqx2.queue.replayq_seg_bytes`` 是用于指定缓存在磁盘上的消息队列的最大单个文件的大小，如果消息队列大小超出指定值的话，会创建新的文件来存储消息队列。


EMQ X 桥接的命令行使用
-----------------------

启动 emqx_bridge_mqtt 插件:

    $ cd emqx1/ && ./bin/emqx_ctl plugins load emqx_bridge_mqtt
    ok

桥接 CLI 命令:

.. code-block:: bash

    $ ./bin/emqx_ctl bridges
    bridges list                                    # List bridges
    bridges start <Name>                            # Start a bridge
    bridges stop <Name>                             # Stop a bridge
    bridges forwards <Name>                         # Show a bridge forward topic
    bridges add-forward <Name> <Topic>              # Add bridge forward topic
    bridges del-forward <Name> <Topic>              # Delete bridge forward topic
    bridges subscriptions <Name>                    # Show a bridge subscriptions topic
    bridges add-subscription <Name> <Topic> <Qos>   # Add bridge subscriptions topic

列出全部 bridge 状态

.. code-block:: bash

    $ ./bin/emqx_ctl bridges list
    name: emqx     status: Stopped

启动指定 bridge

.. code-block:: bash

    $ ./bin/emqx_ctl bridges start emqx
    Start bridge successfully.

停止指定 bridge

.. code-block:: bash

    $ ./bin/emqx_ctl bridges stop emqx
    Stop bridge successfully.

列出指定 bridge 的转发主题

.. code-block:: bash

    $ ./bin/emqx_ctl bridges forwards emqx
    topic:   topic1/#
    topic:   topic2/#

添加指定 bridge 的转发主题

.. code-block:: bash

    $ ./bin/emqx_ctl bridges add-forwards emqx topic3/#
    Add-forward topic successfully.

删除指定 bridge 的转发主题

.. code-block:: bash

    $ ./bin/emqx_ctl bridges del-forwards emqx topic3/#
    Del-forward topic successfully.

列出指定 bridge 的订阅

.. code-block:: bash

    $ ./bin/emqx_ctl bridges subscriptions emqx
    topic: cmd/topic1, qos: 1
    topic: cmd/topic2, qos: 1

添加指定 bridge 的订阅主题

.. code-block:: bash

    $ ./bin/emqx_ctl bridges add-subscription emqx cmd/topic3 1
    Add-subscription topic successfully.

删除指定 bridge 的订阅主题

.. code-block:: bash

    $ ./bin/emqx_ctl bridges del-subscription emqx cmd/topic3
    Del-subscription topic successfully.

注: 如果有创建多个 bridge 的需求，需要复制默认的 bridge 配置，再拷贝到 emqx_bridge_mqtt.conf 中，根据需求重命名 bridge.mqtt.${name}.config 中的 name 即可。

.. _bridge_mosquitto:

-----------------------
mosquitto 桥接到 EMQ X
-----------------------

mosquitto 本身支持以普通 MQTT 连接方式，桥接到 emqx 消息服务器::

                 -------------             -----------------
    Sensor ----> | mosquitto | --Bridge--> |               |
                 -------------             |      EMQ X    |
                 -------------             |    Cluster    |
    Sensor ----> | mosquitto | --Bridge--> |               |
                 -------------             -----------------

mosquitto.conf
--------------

本机 （192.168.1.1）1883 端口启动 emqx 进程，远端服务器（192.168.1.2）1883 端口启动 mosquitto 并创建桥接。

mosquitto.conf 配置::

    connection emqx
    address 192.168.1.1:1883
    topic sensor/# out 2

    # Set the version of the MQTT protocol to use with for this bridge. Can be one
    # of mqttv31 or mqttv311. Defaults to mqttv31.
    bridge_protocol_version mqttv311

.. _bridge_rsmb:

-------------------
rsmb 桥接到 EMQ X
-------------------

本机（192.168.1.1）1883 端口启动 emqx 消息服务器，远端服务器（192.168.1.2）1883 端口启动 rsmb 并创建桥接。

broker.cfg 桥接配置::

    connection emqx
    addresses 192.168.1.1:1883
    topic sensor/#

