RocketMQ-Research
==================

RocketMQ源代码学习研究(包括代码注释、文档、用于代码分析的测试用例)


## 使用的RocketMQ版本

rocketmq-all-4.4.0

## NameSrv

问题列表：

1. Broker信息是由客户端主动来获取的，NameSrv并不会主动通知客户端。
2. 客户端如何从NameSrv获取topic信息？Producer和Consumer，以及Broker。
答：GET_ROUTEINTO_BY_TOPIC 请求类型 - 根据topic获取route信息
3. NameSrv是如何收集TopicRouteData这些信息的，以及Producer获取到这些信息都是如何转换成自己想要的。Consumer和Broker还没看。
4. NameSrv会和Broker进行长连接，保持心跳；客户端（Producer和Consumer）只是定期发送请求来获取信息而已。
答：客户端通过scheduleAtFixedRate定期线程来与NameSrv通信。
5. NameSrv配置集群，目前看代码发现，客户端并没有与所有NameSrv通信，只是选择一条进行交互，当这条不可用时再选择下一条。
6. Topic对应BrokerName，BrokerName对应多个BrokerAddr。Producer是选择一个BrokerAddr进行消息发送，那Consumer如何知道是哪个Broker。

整理：

NameSrv启动后开始监听来自客户端的请求，这里的客户端分为两类，一类是Producer/Consumer获取信息的请求，另一类是Broker上报信息的请求。对于上
报来的信息会存储在集合中（5个table），当接收到获取信息的请求后将集合内容返回。

为了防止broker节点过去，NameSrv会定期扫描这些集合。以及与每个Broker Channel保持长连接，监听这些连接，一旦断开也会清理集合。

NameSrv整体上设计还是比较简单的，相当于一个协同角色。

## Broker

1. Broker启动了一个服务端和一个客户端，服务端用于监听来自Producer或Consumer的请求，客户端（BrokerOuterAPI）用于向NameSrv上报信息用的。
答：那slave和master如何通信？ 
2. Topic是通过命令向Broker来创建的，如果找不多指定的topic或者有默认的topic吧。topic在broker上创建成功后，就会上报到NameSrv，这样topic就
可用了。
3. Broker的事件处理程序是哪个？
答：有多个，registerProcessor()
* PullMessageProcessor 拉取消息
* SendMessageProcessor 发送消息
* QueryMessageProcessor 消息查询
* ClientManageProcessor 客户端处理
* ConsumerManageProcessor 消费者处理
* AdminBrokerProcessor 默认处理，其他所有的请求都在这里，比如UPDATE_AND_CREATE_TOPIC请求
4. 如果Producer发送了一个新的Topic，那Broker和NameSrv是如何处理的？
答：这里有几个知识点要解答，Producer每次发送都要先去NameSrv获取broker吗？是的，发送前会先去topicTable获取该topic，如果不存在，就向NameSrv
拉取该topic信息，如果也没有就会报错。按这个逻辑肯定是不行的，那是如何自动创建的呢？
5. Broker的Topic信息是如何维护的？
答：接收到创建新的Topic请求后，会将该topic信息添加到table中，然后上报到NameSrv，最后进行一些本地处理。
6. Slave角色需要与NameSrv保持心跳吗？
答：需要，Slave可以提供读，所以消费者可使用。Master提供读和写，所以Producer发送消息都是发送到Master节点上。Consumer要能获取到Slave节点，那
NameSrv就必须也要维护Slave的节点信息了。
7. Broker是如何接受Consumer订阅请求的，以及当接收到消息后又是如何发给Consumer的。
答： PULL_MESSAG这个方法很重要，可以认真看下。
8. 同一个Group下的集群和广播是如何处理的，也就是要保存offset，这里应该会很精彩。
答：集群，是Group下的Consumer分担。广播会不会简单点，只要保存一个offset，然后，那什么时候更改offset。
看了分析，集群模式下，一个Group下的Consumer会分别持有不同的queueId，通过这样的方式来分担。那是由谁来分配的呢？以及何时分配？应该是Broker来做的，
以及只要有新的Consumer加入。

每个Consumer都有一个rebalance，定期对消息队列进行负载均衡处理。有集群和广播消费模式，集群模式可以根据策略选择，同一个Group下的Consumers平均
分摊，还是随机分摊，还是一致性哈希等来处理。

9. 消息存储模型！
答：难道每个Broker都有一个CommitLog。为什么会有多个ConsumerQueue，好像ConsumerQueue是根据Topic来划分的。


## Consumer

1. 现在先来看看Consumer是怎么获取消息的？
答：使用push方式，订阅指定topic，那broker收到消息后，就会主动把消息发送给客户端。那客户端就要与XXX先建立连接了？

MQClientInstance要从Consumer角度来分析了。

2. Consumer订阅消息方式？
答：这里还没看明白。一直在想有没有可能是先拉取消息再过滤。能不能这解释，Consumer只有pull_message这一种请求类型
3. 这里就可以展开，offset是谁来保存？

## Producer

1. Producer会跟每个Broker建立连接？
答：应该不是吧，看了源码，好像是在send的时候选择一个Broker进行发送，这时候才建立连接的。