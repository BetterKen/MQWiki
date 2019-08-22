# RabbitMQ集群模式

- 主备模式(了解)
- 远程模式(了解)
- 镜像模式(推荐使用)
- 多活模式(推荐使用)

## 1 主备模式 

使用Haproxy做的主备模式

![](http://dist415.oss-cn-beijing.aliyuncs.com/rmqwarren.png)



## 2 远程模式

远距离通信和复制，所谓Shovel就是我们可以把消息进行不同数据中心的复制工作，我们可以跨地域的让两个mq集群互联。

在使用了shovel插件后，**模型变成了近端同步确认，远端异步确认方式**，大大提高了订单确认速度，并且还能保证可靠性。

我们下面看一下Shovel架构模型：

![](http://dist415.oss-cn-beijing.aliyuncs.com/rmqshovel.png)



## 3 镜像模式

- 镜像模式：集群模式非常经典的就是Mirror镜像模式，保证100%数据不丢失，在实际工作中用的最多的。并且实现集群非常的简单，**一般互联网大厂都会构建这种镜像集群模式**。

- Mirror镜像队列，目的是为了保证rabbitmq数据的高可靠性解决方案，主要就是实现数据的同步，一般来讲是3-5个实现数据同步（对于100%数据可靠性解决方案一般是3个节点）集群架构如下：

  ![](http://dist415.oss-cn-beijing.aliyuncs.com/rmqmirror.png)

  

  

## 4 多活模式

- 多活模式：这种模式也是实现异地数据复制的主流模式，因为Shovel模式配置比较复杂，所以一般来说实现异地集群都是使用双活或者多活模式来实现的。这种模式需要依赖rabbitmq的**federation**插件，可以实现继续的可靠AMQP数据通信，多活模式在实际配置与应用非常的简单。

- RabbitMQ部署架构采用双中心模式（多中心），那么在两套（或多套）数据中心中各部署一套RabbitMQ集群，各中心之间还需要实现部分队列消息共享。多活集群架构如下：

  ![](http://dist415.oss-cn-beijing.aliyuncs.com/rmqfeder.png)

- **Federation**插件是一个不需要构建Cluster，而在Brokers之间传输消息的高性能插件，Federation插件可以在Brokers或者Cluster之间传输消息，连接双方可以使用**不同的users和vistual hosts**，双方也可以使用版本不同的RabbitMQ和Erlang。Federation插件使用AMQP协议通信，可以接收不连续的传输。

- Federation Exchanges,可以看成**Downstream从Upstream主动拉取消息**，但并不是拉取所有消息，必须是在Downstream上已经明确定义Bindings关系的Exchange，也就是有实际的物理Queue来接收消息，才会从Upstream拉取消息到Downstream。使用AMQP协议实施代理间通信，Downstream会将绑定关系组合在一起，绑定/解绑命令将会发送到Upstream交换机。因此，FederationExchange只接收具有订阅的消息。

![](http://dist415.oss-cn-beijing.aliyuncs.com/rmqfederation.png)

