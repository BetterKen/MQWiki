# RabbitMQ介绍

## 1 RabbitMQ是什么

RabbitMQ 是一个由 Erlang 语言开发的 AMQP 的开源实现。

> AMQP ：Advanced Message Queue，高级消息队列协议。它是应用层协议的一个开放标准，为面向消息的中间件设计，基于此协议的客户端与消息中间件可传递消息，并不受产品、开发语言等条件的限制。

## 2 RabbitMQ特点

- 可靠性
- 灵活路由机制
- 消息集群
- 高可用

### 2.1 可靠性

RabbitMQ 使用一些机制来保证可靠性，如持久化、传输确认、发布确认。

### 2.2 灵活路由机制

在消息进入队列之前，通过 Exchange 来路由消息的。对于典型的路由功能，RabbitMQ 已经提供了一些内置的 Exchange 来实现。针对更复杂的路由功能，可以将多个 Exchange 绑定在一起，也通过插件机制实现自己的 Exchange 。

### 2.3 消息集群

多个 RabbitMQ 服务器可以组成一个集群，形成一个逻辑 Broker 。

### 2.4 高可用

队列可以在集群中的机器上进行镜像，使得在部分节点出问题的情况下队列仍然可用。

## 3 AMQP协议

### 3.1 什么是AMQP协议

是具有现代特征的二进制协议。是一个提供统一消息服务的应用层标准高级消息队列协议，是应用层协议的一个开放标准，为面向消息的中间件设计。

### 3.2 协议模型

![](http://dist415.oss-cn-beijing.aliyuncs.com/amqpprot.png)

### 3.3 核心概念

- **Server**:又称**Broker**,接受客户端的连接，实现AMQP实体服务
- **Connection**:连接，应用程序与Broker的网络连接
- **Channel**:网络信道，几乎所有的操作都在Channel中进行，Channel是进行消息读写的通道。客户端可建立多个Channel,每个Channel代表一个会话任务
- **Message**:消息，服务器和应用程序之间传送的数据，由Properties和Body组成。Properties可以对消息进行修饰，比如消息的优先级、延迟等高级特性；Body则就是消息体内容
- **Virtual host**:虚拟地址，用于进行逻辑隔离，最上层的消息路由。—个Virtual Host里面可以有若干个Exchange和Queue,**同一个 VHost里面不能有相同名称的Exchange或Queue**
- **Exchange**:交换机，接收消息，根据路由键转发消息到绑定的队列
- **Binding**: Exchange和Queue之间的虚拟连接，binding中可以包含routing key
- **Routing key**: 一个路由规则,虚拟机可用它来确定如何路由一个特定消息
- **Queue**:也称为Message Queue,消息队列，保存消息并将它们转发给消费者

#### 3.3.1 channel

信道是生产消费者与rabbit通信的渠道，生产者publish或是消费者subscribe一个队列都是通过信道来通信的。信道是建立在TCP连接上的虚拟连接，rabbitmq在一条TCP上建立成百上千个信道来达到多个线程处理，这个TCP被多个线程共享，每个线程对应一个信道，信道在rabbit都有唯一的ID ,保证了信道私有性，对应上唯一的线程使用。

>疑问：为什么不建立多个TCP连接呢？原因是rabbit保证性能，系统为每个线程开辟一个TCP是非常消耗性能，每秒成百上千的建立销毁TCP会严重消耗系统。所以rabbitmq选择建立多个信道（建立在tcp的虚拟连接）连接到rabbit上。

#### 3.3.2 queue

1. 推模式：通过AMQP的basic.consume命令订阅，有消息会自动接收，吞吐量高
2. 拉模式：通过AMQP的bsaic.get命令

注：当队列拥有多个消费者时，队列收到的消息将以循环的方式发送给消费者。每条消息只会发送给一个订阅的消费者

## 4 RabbitMQ整体结构

![](http://dist415.oss-cn-beijing.aliyuncs.com/rabbitserver.png)

![](http://dist415.oss-cn-beijing.aliyuncs.com/rabbitmqtxt.png)

## 5 工作模式

### 5.1 simple模式

![](http://dist415.oss-cn-beijing.aliyuncs.com/rmqsimple.png)

1. 消息产生消息，将消息放入队列
2. 消息的消费者(consumer) 监听 消息队列,如果队列中有消息,就消费掉,消息被拿走后,自动从队列中删除

### 5.2 工作模式



