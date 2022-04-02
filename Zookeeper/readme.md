# Zookeeper

## 一、Zookeeper介绍

### 1. 什么是Zookeeper

ZooKeeper是一种分布式协调服务，用于管理大型主机。在分布式环境中协调和管理服务是一个复杂的过程。Zookeeper通过其简单的架构和API解决了这个问题。Zookeeper允许开发人员专注于核心应用程序逻辑，而不必担心应用程序的分布式特性。

### 2. Zookeeper的应用场景

- 分布式协调组件
![20220401201605](https://raw.githubusercontent.com/neicun1024/PicBed/main/images_for_markdown/20220401201605.png)

在分布式系统中，需要有Zookeeper作为分布式协调组件，协调分布式系统中的状态。

- 分布式锁
Zookeeper在实现分布式锁上，可以做到强一致性，关于分布式锁相关的知识，在之后的ZAB协议中介绍

- 无状态化的实现
![20220401202953](https://raw.githubusercontent.com/neicun1024/PicBed/main/images_for_markdown/20220401202953.png)

每个系统不需要关心登录的状态，而是在Zookeeper中保存登录信息，从而实现分布式系统的无状态

## 二、 搭建Zookeeper服务器


## 三、 Zookeeper内部的数据模型

### 1. zk是如何保存数据的
zk中的数据是保存在节点上的，节点就是Znode，多个znode之间构成一棵树的目录结构。
Zookeeper的数据模型很像数据结构中的树，也很像文件系统。
![20220401213203](https://raw.githubusercontent.com/neicun1024/PicBed/main/images_for_markdown/20220401213203.png)

这样的层级结构，让每一个Znode节点拥有唯一的路径，就像命名空间一样对不同信息作出清晰的隔离。

### 2. zk中的znode是什么样的结构

zk中的znode，包含了四个部分：
- data：保存数据
- acl：权限（定义了什么样的用户能够操作这个节点，且能够进行怎样的操作）
  - c：create 创建权限，允许在该节点下创建子节点
  - w：write 更新权限，允许更心该节点的数据
  - r：read 读取权限，允许读取该节点的内容以及子节点的列表信息
  - d：delete 删除权限，允许删除该节点的子节点
  - a：admmin 管理者权限，允许对该节点进行acl权限设置
- stat：描述当前znode的元数据
- child：当前节点的子节点

### 3. zk中节点znode的类型

- 持久节点：创建出的节点，在会话结束后仍然存在。保存数据
- 持久序号节点：-s （sequential的缩写）创建出的节点，根据先后顺序，会在节点之后带上一个数值，越后执行数值越大，适用于分布式锁的应用场景-单调递增
- 临时节点：-e （ephemeral的缩写）临时节点是在会话结束后，通过这个特性，zk可以实现服务注册与发现的效果。那么临时节点时如何维持心跳呢？
![20220401235636](https://raw.githubusercontent.com/neicun1024/PicBed/main/images_for_markdown/20220401235636.png)
- 临时序号节点：-e -s 跟持久序号节点相同，适用于临时的分布式锁。
- 容器节点（3.5.3版本新增）：-c （Container的缩写） Container容器节点，当容器中没有任何子节点，该容器节点会被zk定期删除（60s）
- TTL节点：-ttl 可以指定节点的到期时间，到期后被zk定时删除。只能通过系统配置*zookeeper.extendedTypesEnabled=true*开启

### 4. zk的数据持久化
zk的数据是运行在内存中，zk提供了两种在能够持久化机制：
- 事务日志
  - zk把执行的命令以日志形式保存在dataLogDir指定的路径中的文件中（如果没有指定dataLogDir，则按dataDir指定的路径）。
- 数据快照
  - zk会在一定的时间间隔内做一次内存数据的快照，把该时刻的内存数据保存在快照文件中。
zk通过两种形式的持久化，在恢复时先恢复快照文件中的数据到内存中，再用日志文件中的数据做增量恢复，这样的恢复速度更快。


## 四、Zookeeper客户端（zkCli）的使用

### 1. 多节点类型创建
- 创建持久节点：creat 
- 创建持久序号节点：create -s
- 创建临时节点：create -e
- 创建临时序号节点：create -e -s
- 创建容器节点：create -c

### 2. 查询节点
- 普通节点：get
- 查询节点详细信息：get -s
  - 数据
  - cZxid：创建节点的事务ID
  - ctime：创建的时间
  - mZxid：修改节点的事务ID
  - mtime：修改的时间
  - pZxid：添加和删除子节点的事务ID
  - cversion：版本号
  - dataVersion：节点内数据的版本，每更新一次数据，版本会+1
  - aclVersion：此节点的权限版本
  - ephemeralOwner：是否是临时节点，如果是临时，该值是当前节点所有者的session id，不是则为0
  - dataLength：节点内数据的长度
  - numChildren：该节点的子节点个数

### 3. 删除节点
- 普通删除
- 乐观锁删除
  - 乐观锁：整个系统乐观地认为当前并发不是很严重，很多地方不用上锁，但是也实现了上锁的效果，乐观锁只在真正有并发出现的时候才上锁（乐观锁是验证而不是阻止）

### 4. 权限设置
- 注册当前会话的账号和密码
```
addauth digest xiaoming:123456
```
- 创建节点并设置权限
```
create /test-node abc auth:xiaoming:123456:cdwra
```
在另一个会话中必须先使用账号密码，才能拥有操作该节点的权限

## 五、Curator客户端的使用

### 1. Curator介绍
Curator是Netflix公司开源的一套Zookeeper客户端框架，Curator是对Zookeeper支持最好的客户端框架。Curator封装了大部分Zookeeper的功能，比如Leader选举、分布式锁等，减少了技术人员在使用Zookeeper时的底层细节开发工作。

1. 引入Curator


## 六、Zookeeper实现分布式锁

### 1. zk中锁的种类：
- 读锁：大家都可以读，要想上读锁的前提：之前的锁没有写锁
- 写锁：只有得到写锁的才能写。要想上写锁的前提是，之前没有任何锁

### 2. zk如何上读锁
- 创建一个临时序列节点，节点的数据是read，表示是读锁
- 获取当前zk中序号比自己小的所有节点
- 判断最小节点是否是读锁：
  - 如果是读锁的话，则上锁成功（因为如果最小节点是读锁的话，它之后肯定没有写锁）
  - 如果不是读锁的话，则上锁失败，为最小节点设置监听。阻塞等待，zk的watch机制会当最小节点发生变化时通知当前节点，于是再执行第二步
![20220402105804](https://raw.githubusercontent.com/neicun1024/PicBed/main/images_for_markdown/20220402105804.png)

### 3. zk如何上写锁
- 创建一个临时序号节点，节点的数据是write，表示是写锁
- 获取zk中所有的子节点
- 判断自己是否是最小的节点：
  - 如果是，则上写锁成功
  - 如果不是，说明前面还有锁，则上锁失败，监听最小的节点，如果最小节点有变化，则回到第二步
![20220402110020](https://raw.githubusercontent.com/neicun1024/PicBed/main/images_for_markdown/20220402110020.png)

### 4. 羊群效应

如果用上述的上锁方式，只要节点发生变化，就会触发其它节点的监听事件，这样的话对zk的压力非常大，这就是羊群效应（惊群效应，只要有一个节点释放，其它节点都“受到惊吓”）。可以调整成链式监听来解决这个问题。
![20220402124142](https://raw.githubusercontent.com/neicun1024/PicBed/main/images_for_markdown/20220402124142.png)

### 5. curator实现读写锁
1. 获取读锁
2. 获取写锁

## 七、Zookeeper的watch机制

### 1. Watch机制介绍
我们可以把Watchc理解成是注册在特定Znode上的触发器。当这个Znode发生改变，也就是调用了create，delete，setData方法的时候，将会触发Znode上注册的对应事件，请求Watch的客户端会接收到异步通知。

具体交互过程如下：
- 客户端调用getData方法，watch参数是true。服务端接到请求，返回数据，并且在对应的哈希表里插入被Watch的Znode路径，以及Watcher列表。
![20220402125124](https://raw.githubusercontent.com/neicun1024/PicBed/main/images_for_markdown/20220402125124.png)
- 当被Watch的Znode已删除，服务端会查找哈希表，找到该Znode对应的所有Watcher，异步通知客户端，并且删除哈希表中对应的Key-Value。
![20220402125111](https://raw.githubusercontent.com/neicun1024/PicBed/main/images_for_markdown/20220402125111.png)
客户端使用了NIO通信模式监听服务端的调用。

### 2. zkCli客户端使用watch

```
create /test xxx
get -w /test 一次性监听节点
ls -w /test 监听目录，创建和删除子节点会收到通知。但子节点中新增节点不会收到通知
ls -R -w /test 对于子节点中子节点的变化，但内容的变化不会收到通知
```
### 3. curator客户端使用watch


## 八、Zookeeper集群实战

### 1. Zookeeper集群角色
Zookeeper集群中的节点有三种角色（这里的节点指服务器节点，而不是树中的节点）
- Leader：处理集群的所有事务请求，集群中只有一个Leader
- Follower：只能处理读请求，可以参与Leader选举
- Observer：只能处理读请求，提升集群读的性能，但不能参与Leader选举

### 2. 集群搭建
搭建4个节点，其中一个节点为Observer

1. 创建4个节点的myid，并设值（myid作为节点的唯一表示，用于选票）
2. 编写4个zoo.cfg
3. 启动4台Zookeeper

### 3. 连接Zookeeper集群


## 九、ZAB协议

### 1. 什么是ZAB协议
Zookeeper作为非常重要的分布式协调组件，需要进行集群部署，集群中会以一主多从的形式进行部署。Zookeeper为了保证数据的一致性，使用了ZAB（Zookeeper Atomic Broadcast）协议，这个协议解决了Zookeeper的崩溃恢复和主从数据同步的问题。
![20220402133952](https://raw.githubusercontent.com/neicun1024/PicBed/main/images_for_markdown/20220402133952.png)

### 2. ZAB协议定义的四种节点状态
- Looking：选举状态
- Following：Follower节点（从节点）所处的状态
- Leading：Leader节点（主节点）所处的状态
- Observing：Observer节点所处的状态

### 3. 集群上线时的Leader选举过程
Zookeeper集群中的节点在上线时，将会进入到Looking状态，也就是选举Leader的状态，这个状态具体会发生什么？

当有两台节点上线时，会开始选举。
![20220402135534](https://raw.githubusercontent.com/neicun1024/PicBed/main/images_for_markdown/20220402135534.png)
第一轮投票：
1. 生成一张自己的选票（此时Node-1中有选票（1，0），Node-2中有选票（2，0））
2. 把选票投给对方（此时Node-1中有选票（1，0）和（2，0），Node-2中的选票都是（2，0））
3. 把zxid/myid更大的选票投到投票箱中（Node-1和Node-2投的都是（2，0））

第二轮投票：
1. 将手上较大的选票投出去（上一轮较大的选票，对于Node-1和Node-2来说都是（2，0））
2. 把选票投给对方（此时Node-1和Node-2手中的选票都是（2，0））
3. 把zxid/myid更大的选票投到投票箱中（Node-1和Node-2投的都是（2，0））

此时投票箱中有票数超过集群半数的服务器节点，该节点确定为Leader，选举结束（服务器节点数为非Observer节点的数量，为3，可以从配置文件中获取，票数超过一半就是至少有两票）。

当第三台节点上线时，发现集群已经选举出了Leader，于是把自己作为Follower。

### 4. 崩溃恢复时的Leader选举
Leader建立完后，Leader周期性地不断向Follower发送心跳（ping命令，没有内容的socket），Follower为周期性地读socket数据。当Leader崩溃后，就停止了心跳的发送，Follower在尝试读socket数据的时候发现socket通道已关闭，于是Follower开始进入到Looking状态，重新回到上一节中的Leader选举状态，此时集群不能对外提供服务。

### 5. 主从服务器之间的数据同步
![20220402141246](https://raw.githubusercontent.com/neicun1024/PicBed/main/images_for_markdown/20220402141246.png)

### 6. Zookeeper中NIO与BIO的应用
