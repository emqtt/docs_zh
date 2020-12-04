# 高级特性 (Advanced Features)

EMQ 2.0 版本新增了本地订阅与共享订阅功能。

## 本地订阅 (Local Subscription)

本地订阅(Local Subscription) 只在本节点创建订阅与路由表，不会在集群节点间广播全局路由，非常适合物联网数据采集应用:

    mosquitto_sub -t '$local/topic'

    mosquitto_pub -t 'topic'

使用方式: 订阅者在主题(Topic)前增加 '$local/' 前缀。

## 共享订阅 (Shared Subscription)

共享订阅(Shared Subscription)支持在多订阅者间采用分组负载平衡方式派发消息:

                                ---------
                                |       | --Msg1--> Subscriber1
    Publisher--Msg1,Msg2,Msg3-->|  EMQ  | --Msg2--> Subscriber2
                                |       | --Msg3--> Subscriber3
                                ---------

共享订阅支持两种使用方式:

| 订阅前缀          | 使用示例                              |
| ----------------- | ------------------------------------- |
| \$queue/          | mosquitto_sub -t '$queue/topic'       |
| \$share/\<group>/ | mosquitto_sub -t '$share/group/topic' |
