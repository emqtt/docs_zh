
.. _lwm2m:

==========
LWM2M 协议
==========

LWM2M 是由 Open Mobile Alliance(OMA) 定义的一套适用于物联网的协议，它提供了设备管理和通讯的功能。协议可以在 `这里 <http://www.openmobilealliance.org/wp/>`_ 下载。

LWM2M 使用 CoAP 作为底层的传输协议，承载在 UDP 或者 SMS 上。

LWM2M 定义了两种服务器

- 一种是 LWM2M BOOTSTRAP SERVER，emqx-lwm2m 插件并未实现该服务器的功能。
- 一种是 LWM2M SERVER，emqx-lwm2m 实现该服务器在 UDP 上的功能，SMS 并没有实现。

LWM2M 把设备上的服务抽象为 Object 和 Resource, 在 XML 文件中定义各种 Object 的属性和功能。可以在 `这里 <http://www.openmobilealliance.org/wp/OMNA/LwM2M/LwM2MRegistry.html>`_ 找到 XML 的各种定义。

---------------
EMQX-LWM2M 插件
---------------

EMQX-LWM2M 是 EMQ X 服务器的一个网关插件，实现了 LWM2M 的大部分功能。MQTT 客户端可以通过 EMQX-LWM2M 访问支持 LWM2M 的设备。设备也可以往 EMQX-LWM2M 上报 notification，为 EMQ X后端的服务采集数据。

-------------------
MQTT 和 LWM2M的转换
-------------------

从 MQTT 客户端可以发送 Command 给 LWM2M 设备。MQTT 到 LWM2M 的命令使用如下的 topic

.. code-block::

    "lwm2m/{?device_end_point_name}/command".

其中 MQTT Payload 是一个 json 格式的字符串，指定要发送的命令，更多的细节请参见 emqx-lwm2m 的文档。

LWM2M 设备的回复用如下 topic 传送
    
.. code-block::

    "lwm2m/{?device_end_point_name}/response".

MQTT Payload 也是一个 json 格式的字符串，更多的细节请参见 emqx-lwm2m 的文档。
    
配置参数
--------

File: etc/emqx_lwm2m.conf::

    lwm2m.port = 5683
       
    lwm2m.certfile = etc/certs/cert.pem

    lwm2m.keyfile = etc/certs/key.pem

    lwm2m.xml_dir =  etc/lwm2m_xml

+-----------------------------+---------------------------------------------------------------------------+
| lwm2m.port                  | 指定 lwm2m 监听的端口号，为了避免和 emqx-coap 冲突，使用了非标准的5783端口|
+-----------------------------+---------------------------------------------------------------------------+
| lwm2m.certfile              | DTLS 使用的证书                                                           |
+-----------------------------+---------------------------------------------------------------------------+
| lwm2m.keyfile               | DTLS 使用的秘钥                                                           |
+-----------------------------+---------------------------------------------------------------------------+
| lwm2m.xml_dir               | 存放 XML 文件的目录，这些 XML 用来定义 LWM2M Object                       |
+-----------------------------+---------------------------------------------------------------------------+

启动 emqx-lwm2m
---------------

.. code-block::

    ./bin/emqx_ctl plugins load emqx_lwm2m

----------------
LWM2M 的客户端库
----------------

- https://github.com/eclipse/wakaama
- https://github.com/OpenMobileAlliance/OMA-LWM2M-DevKit 
- https://github.com/AVSystem/Anjay
- http://www.eclipse.org/leshan/

