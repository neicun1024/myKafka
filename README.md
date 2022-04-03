# myKafka

## 一、为什么要使用消息队列

### 1. 同步的方式存在的问题
- 造成的系统开销-响应时间是比较大的（2-5s）
- 在同步的过程中要保证每个服务都顺利执行完，整个链路才执行完，因为网络等其它问题，整个链路成功执行完的成功率会受影响，导致用户体验较差

同步的通信方式会存在性能和稳定性的问题。

### 2. 异步的优势
- 明显提升系统的吞吐量
- 即使有服务失败，也可以通过分布式事务解决方案来保证最终是成功的

针对同步的通信方式来说，异步的方式，可以让上游快速成功，极大提高了系统的吞吐量。并且在分布式系统中，通过下游多个服务的分布式事务的保障，也能保障业务执行之后的最终一致性。


### 3. 消息队列解决的是什么问题
异步通信问题


## 二、消息队列的流派

目前消息队列中间件选型有很多种：
- RabbitMQ：比较简单，但是内部的可玩性（功能性）使非常强的
- RocketMQ：阿里内部一个大神，根据Kafka的内部执行原理，手写的一个消息队列中间件。性能与Kafka相比肩，除此之外，相比Kafka封装了更多的功能
- Kafka：全球消息处理性能最快的一款MQ
- ZeroMQ：

这些消息队列中间件有什么区别？

### 1. 有Broker的MQ
这个流派通常有一台服务器作为Broker，所有的消息都通过它中转。生产者把消息发送给它就结束自己的任务了，Broker则把消息主动推送给消费者（或者消费者主动轮询）。

#### 重Topic（Kafka、RocketMQ、ActiveMQ）
生产者会发送key和数据到Broker，由Broker比较key之后决定给哪个消费者。这种模式是我们最常见的模式，是我们对MQ最多的印象。在这种模式下，一个topic往往是一个比较大的概念，甚至一个系统中就可能只有一个topic，topic某种意义上就是queue，生产者发送key相当于说：“hi，把数据放到key的队列中”。

整个Broker依据topic来进行消息的中转。在重topic的消息队列里必然需要topic的存在。

#### 轻Topic（RabbitMQ）
生产者发送key和数据，消费者定义订阅的队列，Broker收到数据之后会通过一定的逻辑计算出key对应的队列，然后把数据交给队列。

内部有很多种模式，topic只是其中一种中转模式。

### 2. 无Broker的MQ（ZeroMQ）
ZeroMQ被设计成一个“库”而不是一个中间件，这种实现也可以达到没有Broker的目的。

节点之间通讯的消息都是发送到彼此的队列中，每个节点都既是生产者又是消费者。ZeroMQ做的事情就是封装出一套类似于Socket的API，可以完成发送数据，读取数据。


## 三、Kafka的基本知识

### 1. Kafka的安装
- 部署一台Zookeeper服务器
- 安装jdk
- 下载kafka的安装包
- 上传到kafka服务器上：```/usr/local/kafka```
- 解压缩压缩包
- 进入到config目录内，修改server.properties
```
# broker.id属性在kafka集群中必须要是唯一
broker.id=0
# kafka部署的机器ip和提供服务的端口号
Listeners=PLAINTEXT://172.26.73.44:9092
# kafka的消息存储文件
Log.dir=/usr/local/data/kafka-logs
# kafka连接的Zookeeper地址
zookeeper.connect=172.26.73.44:2181
```
- 进入到bin目录内，执行以下命令来启动kafka服务器（带着配置文件）
```
./kafka-server-start.sh -daemon ../config/server.properties
```
- 校验kafka是否启动成功
进入到zk内查看是否有kafka的节点：```/brokers/ids/0```

### 2. kafka中的一些基本概念
kafka中有这么些复杂的概念
![20220402155640](https://raw.githubusercontent.com/neicun1024/PicBed/main/images_for_markdown/20220402155640.png)

| 名称 | 解释 |
| ---- | ---- |
| Broker | ixoaxi中间件处理节点，一个Kafka节点就是一个broker，一个或者多个Broker可以组成一个Kafka集群 |
| Topic  | kafka根据topic对消息进行归类，发布到kafka集群的每条消息都需要指定一个topic |
| Producer | 消息生产者，向Broker发送消息的客户端 |
| Consumer | 消息消费者，从Broker读取消息的客户端 |
| | |
| | |


### 3. 创建topic
topic可以实现消息的分类，不同消费者订阅不同的topic
![20220402160415](https://raw.githubusercontent.com/neicun1024/PicBed/main/images_for_markdown/20220402160415.png)

执行以下命令创建名为“test”的topic，这个topic只有一个partition，并且备份因子也设置为1：
```
./kafka-topics.sh --create --zookeeper 172.26.73.44:2181 --replication-factor 1 --partitions 1 --topic test
```
查看当前kafka内有哪些topic：
```
./kafka-topics.sh --list --zookeeper localhost:2181
```

### 4. 发送消息
kafka自带了一个producer命令客户端，可以从本地文件中读取内容，或者我们也可以在命令行中直接输入内容，并将这些内容以消息的形式发送到kafka集群中。在默认情况下，每一个行会被当做一个独立的消息。使用kafka的发送消息的客户端，指定发送到的kafka服务器地址和topic
```
./kafka-console-producer.sh --broker-list 172.26.73.44:9092 --topic test
```

### 5. 消费消息
对于consumer，kafka同样也携带了一个命令行客户端，会将获取到内容在命令中进行输出，默认是消费最新的消息。使用kafka的消费者消息的客户端，从指定kafka服务器的指定topic中消费消息
- 方式一：从当前主题中的最后一条消息的offset（偏移量位置）+1开始消费
```
./kafka-console-consumer.sh --bootstrap-server 172.26.73.44:9092 --topic test
```
- 方式二：从当前主题中的第一条消息开始消费
```
./kafka-console-consumer.sh --bootstrap-server 172.26.73.44:9092 --from-beginning --topic test
```

几个注意点：
- 消息会被存储
- 消息是顺序存储的
- 消息是有偏移量的
- 消费时可以致命偏移量进行消费


### 6. 关于消息的细节
![20220402162707](https://raw.githubusercontent.com/neicun1024/PicBed/main/images_for_markdown/20220402162707.png)

- 生产者将消息发送给broker，broker会将消息保存在本地的日志文件中
```
/usr/local/kafka/data/kafka-logs/主题-分区/00000000.log
```
- 消息的保存是有序的，通过offset偏移量来描述消息的有序性
- 消费者消费消息时也是通过offset来描述当前要消费的那条消息的位置


### 7. 单播消息
在一个kafka的topic中，启动两个消费者，一个生产者，问：生产者发送消息，这条消息是否同时被两个消费者消费？

如果多个消费者在同一个消费组，那么只有一个消费者可以收到订阅的topic中的消息。换言之，同一个消费组中只能有一个消费者收到一个topic中的消息。
```
./kafka-console-consumer.sh --bootstrap-server 172.26.73.44:9092 --consumer-property group.id=testGroup --topic test
```

### 8. 多播消息
不同的消费组订阅同一个topic，那么每个消费组中只有一个消费者能收到消息。实际上也是多个消费组中的多个消费者收到了同一个消息。
```
./kafka-console-consumer.sh --bootstrap-server 172.26.73.44:9092 --consumer-property group.id=testGroup1 --topic test
./kafka-console-consumer.sh --bootstrap-server 172.26.73.44:9092 --consumer-property group.id=testGroup2 --topic test
```

下图就是描述多播和单薄消息的区别
![20220402194357](https://raw.githubusercontent.com/neicun1024/PicBed/main/images_for_markdown/20220402194357.png)


### 9. 查看消费组的详细信息
通过以下命令可以查看到消费组的详细信息
```
./kafka-consumer-groups.sh --bootstrap-server 172.26.73.44:9092 --describe --group testGroup
```
![20220402195037](https://raw.githubusercontent.com/neicun1024/PicBed/main/images_for_markdown/20220402195037.png)

重点关注以下几个信息：
- current-offset：最后被消费的消息的偏移量
- Log-end-offset：消息总量（最后一条消息的偏移量）
- Lag：积压了多少条消息


## 四、主题和分区的概念

### 1. 主题Topic
主题-topic在kafka中是一个逻辑的概念，kafka通过topic将消息进行分类。不同的topic会被订阅该topic的消费者消费。

但是有一个问题，如果说这个topic中的消息非常非常多，多到需要几T来存，因为消息是会被保存到log日志文件中的。为了解决这个文件过大的问题，kafka提出了Patition分区的概念。

### 2. 分区Partition

通过Partition将一个topic中的消息分区来存储。这样的好处有多个：
- 分区存储，可以解决统一存储文件过大的问题
- 提供了读写的吞吐量：读和写可以同时在多个分区中进行
![20220402195620](https://raw.githubusercontent.com/neicun1024/PicBed/main/images_for_markdown/20220402195620.png)

创建多分区的主题
```
./kafka-topics.sh --create --zookeeper 172.26.73.44:2181 --replication-factor 1 --partitions 2 --topic test1
```

查看topic的分区信息
```
./kafka-topics.sh --describe --zookeeper localhost:2181 --topic test1
```

分区的作用：
- 可以分布式存储
- 可以并行写

### 3. kafka中消息日志文件中保存的内容
- 00000.log：这个文件中保存的就是消息
- __consumer_offsets-49：
  
  kafka内部自己创建了__consumer_offsets主题，包含了50个分区。这个主题用来存放消费者消费某个主题的偏移量。因为每个消费者都会自己维护着消费的主题的偏移量，也就是说每个消费者会把消费的主题的偏移量自主上报给kafka中的默认主题：consumer_offsets。因此kafka为了提升这个主题的并发性，默认设置了50个分区。
    - 提交到哪一个分区：通过hash函数：hash(consumerGroupId)%__consumer_offsets主题的分区数
    - 提交到该主题中的内容是：key是consumerGroupId+topic+分区号，value就是当前offset的值
- 文件中保存的消息，默认保存7天。七天到后消息会被删除