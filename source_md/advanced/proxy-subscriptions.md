---
# 标题
title: 代理订阅
# 编写日期
date: 2020-02-07 17:15:26
# 作者 Github 名称
author: wivwiv
# 关键字
keywords:
# 描述
description:
# 分类
category: 
# 引用
ref: undefined
---

# 代理订阅

EMQ X Broker 的代理订阅功能使得客户端在连接建立时，不需要发送额外的 SUBSCRIBE 报文，便能自动建立用户预设的订阅关系。

{% hint style="info" %}
你可以使用管理监控 API 对处于连接状态的设备进行订阅、取消订阅操作，参见 [主题订阅](./http-api.md#endpoint-subscribe) [取消订阅](./http-api.md#endpoint-do-unsubscribe)
{% endhint %}


## 内置代理订阅

通过内置代理订阅模块可以通过配置文件指定代理订阅规则实现代理订阅，适用于有规律可循的静态的代理订阅需求。

### 开启代理订阅功能

代理订阅功能默认关闭，开启此功能需要修改 `etc/emqx.conf` 文件中的 `module.subscription` 配置项。默认 `off` 表示关闭，如需开启请修改为 `on`。

```bash
module.subscription = off
```

### 配置代理订阅规则

当然，仅仅开启并不意味代理订阅已经工作，你还需要配置相应的规则，EMQ X Broker 的代理订阅规则支持用户自行配置，用户可以自行添加多条代理订阅规则，每条代理订阅规则都需要指定 Topic 和 QoS，规则的数量没有限制，代理订阅规则的格式如下：

```bash
module.subscription.<number>.topic = <topic>
module.subscription.<number>.qos = <qos>
```

在配置代理订阅的主题时，EMQ X Broker 提供了 `%c` 和 `%u` 两个占位符供用户使用，EMQ X Broker 会在执行代理订阅时将配置中的 `%c` 和 `%u` 分别替换为客户端的 `Client ID` 和 `Username`，需要注意的是，`%c` 和 `%u` 必须占用一整个主题层级。

例如，在 `etc/emqx.conf` 文件中添加以下代理订阅规则：

```bash
module.subscription.1.topic = client/%c
module.subscription.1.qos = 1

module.subscription.2.topic = user/%u
module.subscription.2.qos = 2
```

当一个客户端连接 EMQ X Broker 的时候，假设客户端的 `Client ID` 为 `testclient`，`Username` 为 `tester`，根据上文的配置规则，代理订阅功能会主动帮客户端订阅 QoS 为 1 的 `client/testclient` 和 QoS 为 2 的 `user/tester` 这两个主题。


## 动态代理订阅

EMQ X Enterprise 版本中支持动态代理订阅，通过外部数据库设置主题列表在设备连接时读取列表实现代理订阅。

