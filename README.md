RocketMQ-Research
==================

RocketMQ源代码学习研究(包括代码注释、文档、用于代码分析的测试用例)


## 使用的RocketMQ版本

rocketmq-all-4.4.0

## NameSrv

1. Broker信息是由客户端主动来获取的，NameSrv并不会主动通知客户端。
2. 客户端如何从NameSrv获取topic信息？Producer和Consumer，以及Broker。
答：GET_ROUTEINTO_BY_TOPIC 请求类型 - 根据topic获取route信息
3. NameSrv是如何收集TopicRouteData这些信息的，以及Producer获取到这些信息都是如何转换成自己想要的。Consumer和Broker还没看。