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

![20220402155640](https://raw.githubusercontent.com/neicun1024/PicBed/main/images_for_markdown/20220402155640.png)

| 名称 | 解释 |
| ---- | ---- |
| Broker | 消息中间件处理节点，一个Kafka节点就是一个broker，一个或者多个Broker可以组成一个Kafka集群 |
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
- 消费时可以指明偏移量进行消费


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


## 五、Kafka集群操作

### 1. 搭建kafka集群（三个broker）

- 创建三个**server.properties**文件
```
// 0 1 2
broker.id=2
// 9092 9093 9094
listeners=PLAINTEXT://172.26.73.44:9094
// kafka-logs kafka-logs-1 kafka-logs-2
log.dir=/usr/local/data/kafka-logs-2
```

- 通过命令来启动三台broker
```
./kafka-server-start.sh -daemon ../config/server.properties
./kafka-server-start.sh -daemon ../config/server1.properties
./kafka-server-start.sh -daemon ../config/server2.properties
```

- 校验是否启动成功
进入到zk中查看/brokers/ids中是否有三个znode（0，1，2）

### 2. 副本的概念

在创建主题时，除了指明了主题的分区数以外，还指明了副本数，那么副本是一个什么概念呢？

副本是用来为主题中的分区创建多个备份，多个副本在Kafka集群的多个broker中，会有一个副本作为leader，其它是follower。

副本是对分区的备份。在集群中，不同的副本会被部署在不同的broker上。下面例子：创建1个主题，2个分区，3个副本。

```
./kafka-topics.sh --create --zookeeper 172.26.73.44 -replication-factor 3 --partitions 2 --topic my-replicated-topic
```
![20220404102443](https://raw.githubusercontent.com/neicun1024/PicBed/main/images_for_markdown/20220404102443.png)

- leader：
  - kafka的写和读的操作，都发生在leader上。leader负责把数据同步给follower。当leader挂了，经过主从选举，从多个follower中选举产生一个新的leader
- follower
  - 接收leader的同步数据
- isr
  - 可以同步和已同步的节点会被存入到isr集合中。这里有一个细节：如果isr中的节点性能较差，会被踢出isr集合。

此时，broker、主题、分区、副本，这些概念就全部展现了。

集群中有多个broker，创建主题时可以指明主题有多个分区（把消息拆分到不同的分区中存储），可以为分区创建多个副本，不同的副本存放在不同的broker里。

### 3. 关于集群消费
1. 向集群发送消息：
```
./kafka-console-producer.sh --broker-list 172.26.73.44:9092,172.26.73.44:9093,172.26.73.44:9094 -topic my-replicated-topic
```

2. 从集群中消费消息：
```
./kafka-console-consumer.sh --bootstrap-server 172.26.73.44:9092,172.26.73.44:9093,172.26.73.44:9094 --from-beginning -topic my-
replicated-topic
```

3. 指定消费组消费消息：
```
./kafka-console-consumer.sh --bootstrap-server 172.26.73.44:9092,172.26.73.44:9093,172.26.73.44:9094 --from-beginning --consumer-property group.id=testGroup1 -topic my-
replicated-topic
./kafka-console-consumer.sh --bootstrap-server 172.26.73.44:9092,172.26.73.44:9093,172.26.73.44:9094 --from-beginning --consumer-property group.id=testGroup2 -topic my-
replicated-topic
```

4. 分区消费组的集群消费中的细节
![20220404105544](https://raw.githubusercontent.com/neicun1024/PicBed/main/images_for_markdown/20220404105544.png)

图中Kafka集群有两个broker，每个broker中有多个partition
   - 一个partition只能被一个消费组中的一个消费者消费，目的是为了保证消费的顺序性，但是多个partition的多个消费者消费的总的顺序性是得不到保证的，那怎么做到消费的总顺序性呢？
   - partition的数量决定了消费组中消费者的数量，建议同一个消费组中消费者的数量不要超过partition的数量，否则多的消费者消费不到消息
   - 如果消费者挂了，那么会触发rebalance机制（后面介绍）



## 六、Kafka的Java客户端-生产者的实现

### 1. 生产者的基本实现
- 引入依赖
- 具体实现

### 2. 生产者的同步发送消息
![20220404130518](https://raw.githubusercontent.com/neicun1024/PicBed/main/images_for_markdown/20220404130518.png)

如果生产者发送消息没有收到ack，生产者会阻塞，阻塞到3s的时间，如果还没有收到消息，会进行重试。重试的次数为3次。

### 3. 生产者的异步发送消息
![20220404131407](https://raw.githubusercontent.com/neicun1024/PicBed/main/images_for_markdown/20220404131407.png)

异步发送，生产者发送完消息后就可以执行之后的业务，broker在收到消息后异步调用生产者提供的callback回调方法。

### 4. 生产者中的ack的配置
![20220404134726](https://raw.githubusercontent.com/neicun1024/PicBed/main/images_for_markdown/20220404134726.png)
在同步发送的前提下，生产者在获得集群返回的ack之前会一直阻塞。那么集群什么时候返回ack呢？此时ack有3个配置：
- acks=0
  - kafka-cluster不需要任何的broker收到消息，就立即返回ack给生产者，这种方式是最容易丢消息的，效率是最高的
- acks=1
  - 多副本之间的leader已经收到消息，并把消息写入到本地的log中，才会返回ack给生产者，这种方式的性能和安全性是最均衡的
- acks=-1/all
  - 里面有默认的配置min.insync.replicas=1（默认为1，推荐配置大于等于2），此时就需要leader和一个follower同步完后，才会返回ack给生产者（此时集群中有2个broker已完成数据同步）这种方式最安全，但性能最差

### 5. 关于消息发送的缓冲区
![20220404135504](https://raw.githubusercontent.com/neicun1024/PicBed/main/images_for_markdown/20220404135504.png)

- kafka默认会创建一个消息缓冲区，用来存放要发送的消息，缓冲区大小为32MB
- kafka本地线程会去缓冲区中一次拉16KB的数据，发送到broker
- 如果线程拉不到16KB的数据，间隔10ms也会将已拉到的数据发到broker

*Java中的CountDownLatch的作用：可以用CountDownLatch来启动多线程，合并多线程的查询结果（我的理解：CountDownLatch是一种满足线程安全的计数器）*


## 七、Kafka的Java客户端-消费者的实现

### 1. 消费者的基本实现

### 2. 消费者offset的自动提交和手动提交

1. 提交的内容

消费者无论是自动提交还是手动提交，都需要把所属的消费组+消费的某个主题+消费的某个分区及消费的偏移量，这样的信息提交到集群的__consumer_offsets主题里面。

![20220404154024](https://raw.githubusercontent.com/neicun1024/PicBed/main/images_for_markdown/20220404154024.png)

1. 自动提交

消费者poll到消息后就会自动提交offset。

注意：自动提交可能会丢消息。因为消费者在消费前提交offset，有可能提交完后还没消费时消费者挂了。

3. 手动提交

需要把自动提交的配置改成false

手动提交又分成了两种：
- 手动同步提交
  - 在消费完消息后调用同步提交的方法，当集群返回ack前一直阻塞，返回ack后表示提交成功，执行之后的逻辑
- 手动异步提交
  - 在消息消费完后提交，不需要等到集群ack，直接执行之后的逻辑，可以设置一个回调方法，供集群调用

### 3. 长轮询poll消息

- 默认情况下，消费者一次会poll 500条消息。
- 代码中设置了长轮询的时间是1000毫秒

意味着：
  - 如果一次poll到500条，就直接执行for循环（处理这500条消息）
  - 如果这一次没有poll到500内，且poll的累计时间在1秒内，那么长轮询继续poll，要么到500条消息，要么到1秒
  - 如果多次poll都没达到500条，且poll的累计时间达到1秒，那么直接执行for循环（处理目前poll到的消息）

如果两次poll的时间间隔超过了30秒的时间间隔（消费者消费消息所用的时间超过了30秒），kafka会认为其消费能力过弱，将其踢出消费组，将分区分配给其它消费者。然后触发rebalance机制，但是rebalance机制会造成性能开销。所以可以通过设置参数让一次poll的消息条数少一点（从500条到100条）。

### 4. 消费者的健康状态检查

消费者每隔1秒向kafka集群发送心跳以续约，如果集群发现有超过10秒没有续约的消费者，就将其踢出消费组，触发该消费组的rebalance机制，将该分区交给消费组里的其它消费者进行消费。

### 5. 指定分区和偏移量、时间消费

- 指定分区消费（消费指定partition的消息）
- 从头消费（消费partition从头开始的消息）
- 指定offset消费（消费partition的指定offset之后的消息）
- 指定时间消费（在所有的partition中找到该时间对应的offset，然后在所有的partition中消费该offset之后的消息）

### 6. 新消费组的消费offset规则
新消费组中的消费者在启动以后，默认会从当前分区的最后一条消息的offest+1开始消费（消费新消息）。可以通过设置来让新的消费者第一次消费是从头开始消费。之后的消费是消费新消息（最后消费的位置的偏移量+1）


## 八、Springboot中使用Kafka

### 1. 引入依赖

### 2. 编写配置文件

### 3. 编写消息生产者

### 4. 编写消费者

### 5. 消费者中配置主题、分区、偏移量


## 九、Kafka集群Controller、Rebalance和HW

### 1. Controller
每个broker启动时会向zk创建一个临时序号节点，获得的序号最小的那个broker（最先创建的节点）将会作为集群中的Controller，负责管理整个集群中的所有分区和副本的状态：
- 当某个分区的leader副本出现故障时，由控制器负责为该分区选举新的leader，选举的规则是从isr集合中最左边获得
- 当检测到某个分区的isr集合发生变化时，由控制器负责通知所有broker更新其元数据信息
- 当使用kafka-topics.sh脚本为某个topic增加或减少分区时，同样还是由控制器负责让新分区被其它节点感知到

### 2. Rebalance机制
- 前提：消费者没有指明分区消费
- 触发的条件：当消费组里消费者和分区的关系发生变化
- 分区分配的策略：在触发Rebalance机制之前，消费者消费哪个分区有三种策略
  - range：通过公式来计算每个消费者消费哪几个分区，前面的消费者是分区总数/消费者数量+1，之后的消费者是分区总数/消费者数量
    ![20220404185242](https://raw.githubusercontent.com/neicun1024/PicBed/main/images_for_markdown/20220404185242.png)
  - 轮询：所有分区轮流分配给所有消费者
    ![20220404184405](https://raw.githubusercontent.com/neicun1024/PicBed/main/images_for_markdown/20220404184405.png)
  - sticky：粘合策略，在触发了Rebalance后，会在之前已分配的基础上进行调整，不会改变之前的分配情况。如果这个策略没有开启，那么所有分区都需要重新分配。建议开启。

### 3. HW、LEO、LSO
- LSO（LogStartOffset）是某个副本第一条消息的offset

- LEO（LogEndOffset）是某个副本下一条待写入的消息的offset

- HW （High Watermark）俗称高水位，它标识了一个特定的消息偏移量（offset），消费者只能消费这个offset之前的消息

![20220404200200](https://raw.githubusercontent.com/neicun1024/PicBed/main/images_for_markdown/20220404200200.png)

在这幅图中，这个日志文件中有9条消息，第一条消息的offset（LSO）为0，最后一条消息的offset为8，下一条待写入的消息的offset（LEO）为9，日志文件的HW为6，表示消费者只能消费offset在0-5之间的消息，offset为6的消息对消费者而言是看不见的。

分区ISR集合中的每个副本都会维护自身的LEO，而ISR集合中最小的LEO即为分区的HW。

举例：某个分区的ISR集合中有3个副本，即1个leader副本和2个follower副本，此时分区的LEO和HW都分别为3。消息3和消息4从生产者出发后先被存入leader副本。

![20220404201747](https://raw.githubusercontent.com/neicun1024/PicBed/main/images_for_markdown/20220404201747.png)

在消息被写入副本之后，follower副本会发送拉取请求来拉取消息3和消息4，进行消息同步。

![20220404201833](https://raw.githubusercontent.com/neicun1024/PicBed/main/images_for_markdown/20220404201833.png)

在同步过程中不同的副本同步的效率不尽相同，在某一时刻follower1完全跟上了leader副本而follower2只同步了消息3，如此leader副本的LEO为5，follower1的LEO为5，follower2的LEO 为4，那么当前分区的HW取最小值4，此时消费者可以消费到offset0至3之间的消息。

![20220404201947](https://raw.githubusercontent.com/neicun1024/PicBed/main/images_for_markdown/20220404201947.png)

当所有副本都成功写入消息3和消息4之后，整个分区的HW和LEO都变为5，因此消费者可以消费到offset为4的消息了。

![20220404202029](https://raw.githubusercontent.com/neicun1024/PicBed/main/images_for_markdown/20220404202029.png)

由此可见kafka的复制机制既不是完全的同步复制，也不是单纯的异步复制。
- 事实上，同步复制要求所有能工作的follower副本都复制完，这条消息才会被确认已成功提交，这种复制方式极大的影响了性能。
- 而在异步复制的方式下，follower副本异步的从leader副本中复制数据，数据只要被leader副本写入就会被认为已经成功提交。在这种情况下，如果follower副本都还没有复制完而落后于leader副本，然后leader副本宕机，则会造成数据丢失。

kafka使用这种ISR的方式有效的权衡了数据可靠性和性能之间的关系。


## 十、Kafka线上问题优化（这一块内容千锋讲的有问题）

### 1. 如何防止消息丢失
- 发送方：ack是1或者-1/all可以防止丢失，如果要做到99.9999%，ack设成all，把min.insync.replicas配置成分区备份数
- 消费方：有时候考虑到消息处理消耗时间较长，会单独启动线程拉取消息存储到本地内存队列，然后再搞个线程池并行处理业务逻辑。这种情况下，是拉取到消息就提交offset，可能导致消息丢失。解决方法就是在消息处理完之后再提交offset。

### 2. 如何防止消息的重复消费
在防止消息丢失的方案中，如果生产者发送完消息后，因为网络抖动没有收到ack，但实际上broker已经收到了消息。

此时生产者会进行重试，于是broker会收到多条相同的消息，而造成消费者的重复消费。

解决方法：生产者关闭重试：可能造成消息丢失（不建议）

一个消费者可能已经消费了若干个消息，但是由于是自动提交（每隔几秒提交一次offset），导致offset还没有变化，这时如果它挂了，那么另一个消费者在消费的时候会从之前的offset开始消费，从而导致重复消费。

解决方法：将自动提交改为手动提交。

![20220407235129](https://raw.githubusercontent.com/neicun1024/Interview/main/images_for_markdown/20220407235129.png)

### 3. 如何做到顺序消费
- 发送方：在发送时将ack不能设置0，关闭重试，使用同步发送，等到发送成功再发送下一条。
- 接收方：消息时发送到一个分区中，只能有一个消费组的消费者来接收消息。

### 4. 解决消息积压问题
消息积压会导致很多问题，比如磁盘被打满、生产端发消息导致kafka性能过慢，就容易出现服务雪崩，就需要有相应的手段
- 方案一：提升一个消费者的消费能力：在一个消费者中启动多个线程，让多个线程同时消费；
- 方案二：充分利用服务器的CPU资源：如果方案一还不够的话，可以启动多个消费者，多个消费者部署在不同的服务器上。其实多个消费者部署在同一个服务器上也可以提高消费能力；
- 方案三：让一个消费者去把收到的消息往另外一个topic上发，另一个topic设置多个分区和多个消费者，进行具体的业务消费。

### 5. 实现延时队列的效果

#### 1. 应用场景

订单创建后超过30分钟没有支付，则需要取消订单，这种场景可以通过延时队列来实现。


#### 2. 具体方案

![20220408123514](https://raw.githubusercontent.com/neicun1024/PicBed/main/images_for_markdown/20220408123514.png)

- kafka中创建相应的主题
- 消费者消费该主题的消息（轮询）
- 消费者消费消息时判断消息的创建时间和当前时间是否超过30分钟（前提是订单没支付）
  - 如果是：去数据库中修改订单状态为已取消
  - 如果否：记录当前消息的offset，并不再继续消费之后的消息。等待1分钟后，再次向kafka拉取该offset及之后的消息，继续进行判断，以此反复