# Kafka可靠性保证

## 1. 生产者可靠性保证

设置**acks=-1(all)&&retries=MAX**，一定不会丢，要求是，你的 leader 接收到消息，所有的 follower 都同步到了消息之后，才认为本次写成功了。如果没满足这个条件，生产者会自动不断的重试，重试无限次。

## 2. 消费者可靠性保证

### 2.1 消费者提交方式

- 自动提交
- 手动提交
  - 同步提交(阻塞线程)
  - 异步提交(非阻塞)

**无论是同步提交还是异步提交 offset，都有可能会造成数据的漏消费或者重复消费。**

**先提交 offset 后消费，有可能造成数据的漏消费；**

**而先消费后提交 offset，有可能会造成数据的重复消费**

### 2.2 消费者可靠性选择

**我们采取异步先消费后提交的方式来保证消费者端的可靠性,造成的重复消费我们使用幂等性来规避**



## 3 Broker可靠性保证

**这块比较常见的一个场景，就是 Kafka 某个 broker 宕机，然后重新选举 partition 的 leader。要是此时其他的 follower 刚好还有些数据没有同步，结果此时 leader 挂了，然后选举某个 follower 成 leader 之后，就会丢一些数据**

所以此时一般是要求起码设置如下 2 个参数：

- **给 topic 设置 replication.factor 参数：这个值必须大于 1，要求每个 partition 必须有至少 2 个副本。**
- 在 Kafka 服务端设置 **min.insync.replicas** 参数：**这个值必须大于 1，这个是要求一个 leader 至少感知到有至少一个 follower 还跟自己保持联系，没掉队，这样才能确保 leader 挂了还有一个 follower 吧。**

我们生产环境就是按照上述要求配置的，这样配置之后，至少在 Kafka broker 端就可以保证在 leader 所在 broker 发生故障，进行 leader 切换时，数据不会丢失。

## 4 总结

![](http://dist415.oss-cn-beijing.aliyuncs.com/kafkasis.png)







