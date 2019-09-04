# Kafka幂等性保证

## 1 Kafka生产端自带幂等性

为了实现Producer的幂等性，Kafka引入了Producer ID（即PID）和Sequence Number。

- PID。每个新的Producer在初始化的时候会被分配一个唯一的PID，这个PID对用户是不可见的。
- Sequence Numbler。（对于每个PID，该Producer发送数据的每个<Topic, Partition>都对应一个从0开始单调递增的Sequence Number

Kafka可能存在多个生产者，会同时产生消息，但对Kafka来说，**只需要保证每个生产者内部的消息幂等就可以了**，所有引入了PID来标识不同的生产者。

对于Kafka来说，要解决的是生产者发送消息的幂等问题。也即需要区分每条消息是否重复。
Kafka通过为每条消息增加一个Sequence Numbler，通过Sequence Numbler来区分每条消息。每条消息对应一个分区，不同的分区产生的消息不可能重复。所有Sequence Numbler对应每个分区

Broker端在缓存中保存了这seq number，对于接收的每条消息，如果其序号比Broker缓存中序号大于1则接受它，否则将其丢弃。这样就可以实现了消息重复提交了。

**但是，只能保证单个Producer对于同一个<Topic, Partition>的Exactly Once语义。不能保证同一个Producer一个topic不同的partion幂等。**

## 2 消费端幂等性

将在消息队列幂等性文章中统一说明

