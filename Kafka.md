# Kafka

Kafka是一种分布式的，基于发布/订阅的消息系统。主要设计目标如下：

- 以时间复杂度为O(1)的方式提供消息持久化能力，即使对TB级以上数据也能保证常数时间复杂度的访问性能
- 高吞吐率。即使在非常廉价的商用机器上也能做到单机支持每秒100K条以上消息的传输
- 支持Kafka Server间的消息分区，及分布式消费，同时保证每个Partition内的消息顺序传输
- 同时支持离线数据处理和实时数据处理
- Scale out：支持在线水平扩展



## 为什么要用消息队列

- 解耦：
- 冗余：
- 扩展性：
- 削峰填谷：
- 可恢复:
- 缓冲：
- 异步：





## 消息队列的两种模式

**点对点模式：**

​	一对一，消费者主动拉取，消息收到后消息清除



**发布/订阅模式：**

​	一对多，消费者消费数据后，不会清除消息



## Kafka的基础架构

**Producer:**消息生产者，向Kafka broker发送消息

**Consumer:**消息的消费者，向Kafka broker取消息的客户端

**Consumer Group(CG):**消费者组，多个Consumer组成。消费者组内每个消费者负责消费不同分区的数据，一个分区只能由一个消费者消费；消费者组之间互不影响。所有的消费者都是属于一个消费者组，因此逻辑上来讲，一个消费者组抽象为一个订阅者。

**Broker:**一台Kafka服务器就是一个broker。一个集群由多个broker组成。一个broker可以容纳多个topic。

**Topic：**可以理解为一个队列，生产者和消费者直接面对的就是话题Topic



**Partition:**为了实现扩展性，一个非常大的topic可以分布到多个broker上，一个topic包含多个partition,每个partition是一个有序队列。



**Replica:**副本，为了保证集群中的某个节点发生故障时，该节点上的partition数据不丢失，提高容错率，每一个partition都会有若干个副本，一个leader和几个flower.



**Leader:**一个分区多个副本中的领导者，生产者发送数据的对象，消费者消费数据的对象都是leader。



**Follower:**一个分区多个副本中的小弟，实施的从领导者哪里同步数据，保持和leader的数据同步。leader发生故障，其中一个小弟会成为新的leader



## Kafka的工作流程





## Kafka文件存储机制

一个Topic里分为多个partition。每个partition可能有多个副本，一个partition分为多个segment。

一个segment对应一个.log和.index文件

![1568796508178](img\1568796508178.png)

生产者生产的消息不断地追加，为了防止文件过大导致检索效率低下。Kafka采用**分片**和**索引**的机制。一个partition分为多个segment。segment包含了`.index`和`.log`文件。这些文件的命名规则：topic+分区序号



![1568796568557](img\1568796568557.png)



## Kafka生产者

#### 分区策略

**分区原因**

1.方便再集群中扩展，每个partition可以通过调整以适应它所在的机器，而一个topic又可以有多个partition组成。所以整个集群可以适应各种大小的数据

2.可以提高并发，可以一partition为单位来读写。partition是最小的并发粒度



**分区的原则**

- 指明partition的情况下，直接指定分区的话，将使用指定的值作为partition
- 没有指定partition但是由key的情况下，将key的hash值与topic的partition数进行取余得到partition值
- 既没有partition值有没有key值的情况下，第一次调用随机生成一个整数，以后调用每次自增，将这个值与partition数两取余，得到对应的partition号。即round-robin算法



#### 数据可靠性如何保证

生产者发送数据给topic，topic的每一个partition收到生产者发送的数据后，都需要向producer发**送ack，如果producer收到ack，确定这一轮消息发送成功，否则重新发送。**

**副本同步策略**

全部完成同步才发送ack，此时n个副本选取新leader时可以容忍n-1个副本宕掉。缺点是延迟高

**ISR（即in-sync Replica）**：leaderhi跟踪与其保持同步的Replica列表，如果Follwer宕机，或者过于落后，Leader会将其从ISR中删除，落后的阈值可以设置可以设置条数`replica.lag.max.messages`=4000，也可以设置超时时间`replica.lag.time.max.ms=10000`



**ACK应答机制**

acks:

0的时候producer不需要等待broker的ack，此项延迟最低，broker已接受到 还没有写入到磁盘就已经返回，但是当broker宕掉的时候有可能数据丢失；

1的时候producer等待broker的ack，partition的leader落盘成功后返回ack，如果再follwer同步之前leader宕掉了，数据会丢失。

-1的时候procucer等待broker的ack，partition的leader和follower全部落盘成功后返回ack，但是如果再落盘成功后，返回ack之前，leader发生故障，会导致数据重复



#### Replica之间是如何选举新的leader

所有的Follower都会在Zookeeper上设置一个Watch。一单leader宕机，其对应的ephemeral znode会删除，此时所有的follower都会尝试建立这个节点，但是zookeeper只会允许其中一个建立成功，因此成功那个follower当选新的leader,其他的成为follower。但是容易造成脑裂、zookeeper压力过大、羊群效应。0.8中，他在所有的broker中选出一个controller,所有的partition的leader选举都是由controller决定。



## Broker failover过程简介









## Kafka高效率读写数据

Kafka的product生产数据，要写入log文件，写的过程是一直追加到文件末端，即顺序写,也就是一个文件。而机械硬盘的顺序写比随机写要快很多，ranid5阵列的磁盘顺序写有600MB/s，随机写只有100k/s。

除此之外，Kafka会借助Page Cache减少甚至不适用磁盘读写，也就是零拷贝。

![1568856544901](img\1568856544901.png)



## Zookeeper在Kafka中的的作用

所有Broker会选举出一个Broker作为Controller,负责管理集群的broker的上下限，所有的topic的分区副本分配和leader选举的工作。controller的工作依赖于Zookeeper

每台broker都会向Zookeeper注册ids，/broker/ids [0,1,2]

Kafka Controller会监听/broker/ids,选举出一个leader,其他都是replica

此时会生成一个/broker/topics/topicname/partition/0/state:"leader":1,"isr"[0,1,2]

此时如果leader挂掉了，/broker/ids中会剔除对应id，Controller监听到后会获取ISR选举新的

然后更新leader和ISR列表。

所有的broker都会在zk中生成一个watch目录，并不断尝试把该目录重新命名为Controller,但是只允许有一个，如果controller挂掉了，自然会有其他的broker成功成为controller，然后继续监听。





## Producer API

生产者发送消息的方式采用异步的方式。发送消息的时候，会存在两个线程一个是main线程，一个是sender线程，以及共享变量Record Accumulator。main线程不断地将消息发送给record Accumulator，sender不断地从Record Accumulator 中拉取,发送给broker。

这个过程不是将记录一个一个传过去的，而是以批次的形式，可以设置一个batchsize,如果达到了batchsize sender才会发送，除此之外还有一个限定时间linger.ms。如果这个记录的批次一直没有达到batchsize，但是到了这个时间也会被sender发送。



![1568859861290](img\1568859861290.png)





## Consumer API

Consumer主要考虑的问题是offset的问题，可以手动或者自动提交offset

`enable.auto.commit`:是否开启自动提交offset的功能

`auto.commit.interval.ms`:自动提交offset的时间间隔





### Offset维护

Kafka在0.8版本中，consumer的offset值保存在zookeeper中。0.9版本之后存在Kafka的topic中

内置该topic为**__consumer_offsets**





## 自定义拦截器Interceptor

拦截器可以在消息发送之前对消息做一些定制化的需求，比如修改消息。producer允许用户允许多个interceptor按序作用域同一条消息，从而形成一个拦截链（interceptor chain）。Interceptor的实现接口是，`org.apache.kafka.clients.producer.ProducerInterceptor`

**config**:获取配置信息

**onSend**:用户可以在该方法中对消息做任何操作

**onAckKnowledgement**:如果方法会在sender发送消息成功或者失败后调用

**close**:关闭Interceptor,清理资源







## Kafka面试题

1.Kafka中的ISR、AR又代表什么？

ISR(In-sync Replica)，跟leader保持同步的folower集合，leader中记录的副本，如果leader挂掉了会优先从这里面选举leader

AR：分区中的所有副本

2.Kafka中的HW、LEO等分别代表什么？

LEO(last end offset):replica当前最后的offset值

HW(High watermark)：所有replica中最小的LEO

HW之前的记录是对consumer可见的，之后的是不可见

3.Kafka中是怎么体现消息顺序性的？

每个分区都有一个offset。分区内有序，每个分区之间是无序的

4.Kafka中的分区器、序列化器、拦截器是否了解？它们之间的处理顺序是什么？

producer=>拦截器=>序列化器=》分区器

拦截器可以对消息做任何操作，比如加上时间戳之类的

序列化器：

分区器是用来分发的

5.Kafka生产者客户端的整体结构是什么样子的？使用了几个线程来处理？分别是什么？

两个线程main、sender。一个共享变量：recordAccumulator

producer =>拦截器=>序列化器=>分区器 发送给recordAccumulator 以recordBatch。达到batchsize的之后sender会发送这个到topic对应的分区，如果没达到超时也会发送，这个阈值可以设置linger.ms



6.“消费组中的消费者个数如果超过topic的分区，那么就会有消费者消费不到数据”这句话是否正确？

这句话是正确的，因为同一个组内一个消费者一个分区，超过了就会有消费者空闲

7.消费者提交消费位移时提交的是当前消费到的最新消息的offset还是offset+1？

offset+1

8.有哪些情形会造成重复消费？





9.那些情景会造成消息漏消费？

如果消费之前提交了offset，就有可能造成消息漏了

10.当你使用kafka-topics.sh创建（删除）了一个topic之后，Kafka背后会执行什么逻辑？

​    1）会在zookeeper中的/brokers/topics节点下创建一个新的topic节点，如：/brokers/topics/first

​    2）触发Controller的监听程序

​    3）kafka Controller 负责topic的创建工作，并更新metadata cache

11.topic的分区数可不可以增加？如果可以怎么增加？如果不可以，那又是为什么？

可以增加，通过--alter --topic topic-config --partition 

bin/kafka-topics.sh --zookeeper
localhost:2181/kafka --alter --topic topic-config --partitions 3

12.topic的分区数可不可以减少？如果可以怎么减少？如果不可以，那又是为什么？

不可以，减少的分区数据是很难处理的



13.Kafka有内部的topic吗？如果有是什么？有什么所用？

有，__consumer_offset 这个是消费者消费数据时，保存offset用的

14.Kafka分区分配的概念？

一个topic有多个分区，消费者组会有多个消费者，将分区分配给消费者

两种方式：roudrobin分发， range平均分



15.简述Kafka的日志目录结构？

/log/[topic]/index和log

index时记录的log的索引



16.如果我指定了一个offset，Kafka Controller怎么查找到对应的消息？

二分，找到index文件，然后进入文件查找对应的值，在进入log文件查找对应的消息



17.聊一聊Kafka Controller的作用？

管理broker，以及所有topic的副本，还有leader的选举等待

18.Kafka中有那些地方需要选举？这些地方的选举策略又有哪些？

分区的leader（ISR的方式），controller（抢占）

19.失效副本是指什么？有那些应对措施？

即ISR中被踢出的，不与leader同步，等到追上leader之后才会重新加入



20.Kafka的那些设计让它有如此高的性能？

分区：分区多提高了整体的并行的，分区也是并行的最小粒度

顺序读写，append的方式，写一个大文件比写一堆小文件快很多

`pageCache`与`sendfile`实现0copy