# 常见问题(FAQ)

Q1. 无法订阅$SYS/开头的系统主题?

\$SYS/#系统主题默认只允许本机订阅，访问控制规则设置在etc/acl.config:
```
{allow, {ipaddr, "127.0.0.1"}, pubsub, ["$SYS/#", "#"]}.
```
  1. emqttd消息服务器适用于什么项目? 
     - 物联网、移动互联网.
     - 不适合做企业内部MQ

  2. 使用emqttd服务器，需要了解Erlang语言吗？ 

