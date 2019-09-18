# Flume

高可用、高可靠的分布式海量日志采集、聚合和传输的系统。流式架构，灵活简单



Flume可以实时监控本地磁盘上的数据，并将数据写入到HDFS



## Flume组成架构

Agent是Flume数据传输的基本单元，以事件的形式传输数据

Agent内部有三个核心：**Source、Channel、Sink**



**Source:**负责接收数据到Flume Agent的组件。Source组件可以处理的数据类型有很多，baokuo：avro、thrift、exec、jms、spooling directory、netcat、syslog、http、legacy、sequence generator



**Sink:**不断地轮询Channel中的事件，且批量地移除它们，并将这些事键批量写入到存储或索引系统、或者发送到另一个Flume Agent。组件包括：HDFS、Logger、Avro、Thrift、ipc、file、HBase、solr、自定义.。

Sink是事务性的，每次从Channel批量删除数据之前，每个Sink用Channel启动一个事件，事件一旦写出成功，Sink酒用Channel提交事务。事务提交后，Channel会将其删除



**Channel:**介于Source和Sink之间的缓冲区。Flume自带两种Channel，分别是：Memory Channel和FIle Channel

Memory Channel是在内存中的队列，适合在不需要关心数据丢失的情况下使用

FIle Channel会将所有的事件写入磁盘，可以在发生宕机或者程序关闭的情况后恢复数据



## Flume的拓扑结构

Flume=>Flume 使用Avro Sink 和Avro Source

单个Source多个channel、sink：有两种Selector replicating(default)和Multiplexing 。replicating会将事件发送给所有的channel，而Multiplexing是可以通过mapping配置事件发往哪些Channel

除此之外：负载均衡的拓扑，和 flume Agent 聚合

一个flume 分三个sink，分别到三个flume，再写入HDFS或者进入Kafka消费。负载均衡

多个Flume分别监听不同Server的日志，通过Avro Sink传输给一个Flume (Consolidation合并)。Agent聚合



## Flume Agent的内部原理

![1568642380785](img\1568642380785.png)





## Flume拦截器

Source 将 Event 写入到 Channel 之前可以使用拦截器对 Event 进行各种形式的处理，Source 和 Channel 之间可以有多个拦截器，不同拦截器使用不同的规则处理 Event，包括时间、主机、UUID、正则表达式等多种形式的拦截器。







## Flume选择器（Selector）

Source 发送的 Event 通过 Channel 选择器来选择以哪种方式写入到 Channel 中，Flume 提供三种类型 Channel 选择器，分别是复制Selector replicating(default)和Multiplexing 复用和自定义选择器。

1. 复制选择器: 一个 Source 以复制的方式将一个 Event 同时写入到多个 Channel 中，不同的 Sink 可以从不同的 Channel 中获取相同的 Event，比如一份日志数据同时写 Kafka 和 HDFS，一个 Event 同时写入两个 Channel，然后不同类型的 Sink 发送到不同的外部存储。
2. 复用选择器: 需要和拦截器配合使用，根据 Event 的头信息中不同键值数据来判断 Event 应该写入哪个 Channel 中。





## Flume的负载均衡和故障转移

### Flume Sink Processors

Sink groups允许用户在一个代理中对多个sink进行分组。Sink processor能够实现分组内的sink负载均衡。以及组内sink容错，实现当组内一个sink失败时，切换至其他的sink。

| Property Name      | Default   | Description                                                  |
| :----------------- | :-------- | :----------------------------------------------------------- |
| **sinks**          | –         | Space-separated list of sinks that are participating in the group |
| **processor.type** | `default` | The component type name, needs to be`default`, `failover` or `load_balance` |

示例:



```
a1.sinkgroups=g1a1.sinkgroups.g1.sinks=k1 k2a1.sinkgroups.g1.processor.type=load_balancea1.sinkgroups.g1.processor.backoff=truea1.sinkgroups.g1.processor.selector=random
```



#### Default Sink Processor

​    默认的sink processor仅接受单独一个sink。不必对单个sink使用processor。对单个sink可以使用source-channel-sink的方式。

#### Failorver Sink Processor

​    Failover Sink Processor（故障转移处理器）拥有一个sink的优先级列表，用来保证只有一个sink可用。

​    容错机制将失败的sink放入一个冷却池中，并给他设置一个冷却时间，如果重试中不断失败，冷却时间将不断增加。一旦sink成功的发送event，sink将被重新保存到一个可用sink池中。在这个可用sink池中，每一个sink都有一个关联优先级值，值越大优先级越高。当一个sink发送event失败时，剩下的sink中优先级最高的sink将试着发送event。例如：在选择发送event的sink时，优先级100的sink将优先于优先级80的sink。如果没有设置sink的优先级，那么优先级将按照设置的顺序从左至右，由高到低来决定。

​    设置sink组的processor为failover，并且为每个独立的sink配置优先级，优先级不能重复。通过设置参数maxpenalty，来设置冷却池中的sink的最大冷却时间。

​    示例：

```
a1.sinkgroups=g1a1.sinkgroups.g1.sinks=k1 k2a1.sinkgroups.g1.processor.type=failovera1.sinkgroups.g1.processor.priority.k1=5a1.sinkgroups.g1.processor.priority.k2=10a1.sinkgroups.g1.processor.maxpenalty=10000
```

#### Load balancing Sink Processor

​    Load balancing Sink processor（负载均衡处理器）在多个sink间实现负载均衡。数据分发到多个活动的sink，处理器用一个索引化的列表来存储这些sink的信息。处理器实现两种数据分发机制，轮循选择机制和随机选择机制。默认的分发机制是轮循选择机制，可以通过配置修改。同时我们可以通过继承AbstractSinkSelector来实现自定义数据分发选择机制。

​    选择器按照我们配置的选择机制执行选择sink。当sink失败时，处理器将根据我们配置的选择机制，选择下一个可用的sink。这种方式中没有黑名单，而是主动常识每一个可用的sink。如果所有的sink都失败了，选择器将把这些失败传递给sink的执行者。

​    如果设置backoff为true，处理器将会把失败的sink放进黑名单中，并且为失败的sink设置一个在黑名单驻留的时间，在这段时间内，sink将不会被选择接收数据。当超过黑名单驻留时间，如果该sink仍然没有应答或者应答迟缓，黑名单驻留时间将以指数的方式增加，以避免长时间等待sink应答而阻塞。如果设置backoff为false，在轮循的方式下，失败的数据将被顺序的传递给下一个sink，因此数据分发就变成非均衡的了。



| Property Name                 | Default       | Description                                                  |
| :---------------------------- | :------------ | :----------------------------------------------------------- |
| **processor.sinks**           | –             | Space-separated list of sinks that are participating in the group |
| **processor.type**            | `default`     | The component type name, needs to be`load_balance`           |
| processor.backoff             | false         | Should failed sinks be backed off exponentially.             |
| processor.selector            | `round_robin` | Selection mechanism. Must be either`round_robin`, `random` or FQCN of custom class that inherits from`AbstractSinkSelector` |
| processor.selector.maxTimeOut | 30000         | Used by backoff selectors to limit exponential backoff (in milliseconds) |

## Flume参数调优

**Source:**

​		增加Source个数，使用Tair Dir Source 时可以增加FIleGroup个数 可以增大Source的读取数据的能力。

​		调整batchSize参数决定Source往Channel运输的每个批次的事件条数



**Channel:**

​		type选择memory Channel时性能好，但是无法保证数据安全性，掉电会丢失。选择File channel容错性好，但是性能差一些。

​		使用file Channel 时dataDir配置不同磁盘的目录，可以提高性能。（多个磁盘并行，有点类似raid）。

​		Capacity 参数决定Channel可容纳最大的event条数。

​		transactionCapacity参数决定每次Source往channel里面写的最大event条数和每次Sink从channel里面读的最大的event条数。这个参数必须要大于Source和Sink的batchSize。



**Sink:**

​		增加Sink个数可以直接增加消费event的能力，batchSize决定了Sink每次从Channel中获取的event条数





## Flume事务机制

Flume的事务机制，可以类比数据库里的事务，Flume有两个独立的事务，分别是从Source到Channel，以及从Channel到Sink的事件传递。事务具有一致性：source为每一个批次创建一个事务，一旦事务中所有的事件全部提交成功，则Source标记该事务完成。从Channel到Sink端也是同理，一旦有一个事件失败，则整个事务失败并回滚。所有的事件都会保存到Channel中等待重新传递







## Flume监控文件夹

#### Spooling Directory Source

Example for an agent named agent-1:

```
a1.channels = ch-1
a1.sources = src-1

a1.sources.src-1.type = spooldir
a1.sources.src-1.channels = ch-1
a1.sources.src-1.spoolDir = /var/log/apache/flumeSpool
a1.sources.src-1.fileHeader = true
```

#### Exec

执行指定的Unix命令，且该命令在持续生成数据，当flume停止时，该source命令也会停止

Example for agent named a1:

```
a1.sources = r1
a1.channels = c1
a1.sources.r1.type = exec
a1.sources.r1.command = tail -F /var/log/secure
a1.sources.r1.channels = c1
```

The ‘shell’ config is used to invoke the ‘command’ through a command shell (such as Bash or Powershell). The ‘command’ is passed as an argument to ‘shell’ for execution. This allows the ‘command’ to use features from the shell such as wildcards, back ticks, pipes, loops, conditionals etc. In the absence of the ‘shell’ config, the ‘command’ will be invoked directly. Common values for ‘shell’ : ‘/bin/sh -c’, ‘/bin/ksh -c’, ‘cmd /c’, ‘powershell -Command’, etc.

```
a1.sources.tailsource-1.type = exec
a1.sources.tailsource-1.shell = /bin/bash -c
a1.sources.tailsource-1.command = for i in /path/*.txt; do cat $i; done
```

Spooling directory Source 与Exec的区别

```
Warning The problem with ExecSource and other asynchronous sources is that the source can not guarantee that if there is a failure to put the event into the Channel the client knows about it. In such cases, the data will be lost. As a for instance, one of the most commonly requested features is the tail -F [file]-like use case where an application writes to a log file on disk and Flume tails the file, sending each line as an event. While this is possible, there’s an obvious problem; what happens if the channel fills up and Flume can’t send an event? Flume has no way of indicating to the application writing the log file that it needs to retain the log or that the event hasn’t been sent, for some reason. If this doesn’t make sense, you need only know this: Your application can never guarantee data has been received when using a unidirectional asynchronous interface such as ExecSource! As an extension of this warning - and to be completely clear - there is absolutely zero guarantee of event delivery when using this source. For stronger reliability guarantees, consider the Spooling Directory Source, Taildir Source or direct integration with Flume via the SDK.
```

使用exec没办法保证事件的完整性，如果执行的过程中flume出了问题，着这段时间的时间会丢失，如果需要保证数据完整可以使用Spooling directory Source。

但是Spooling directory Source监控目录时，不能修改文件的名字，不能出现同名覆盖文件，不要出现只有一半内容的文件。传输完成后，文件会被重命名为.COMPLETED，需要定时脚本将其删除。重启FLume会导致event重复,因为没有完成传输的文件未被重新命名。

因为使用Spooling directory Source的时候不能同时读写，如果写入文件较大，过程中flume察觉到的时候会报错，此时可以用正则忽略掉正在传输中的文件，比如：

```
test.sources.source1.ignorePattern = ^(.)*\\.tmp$
```









## Flume订阅Kafka消息

Example for topic subscription by comma-separated topic list.

```
tier1.sources.source1.type = org.apache.flume.source.kafka.KafkaSource
tier1.sources.source1.channels = channel1
tier1.sources.source1.batchSize = 5000
tier1.sources.source1.batchDurationMillis = 2000
tier1.sources.source1.kafka.bootstrap.servers = localhost:9092
tier1.sources.source1.kafka.topics = test1, test2
tier1.sources.source1.kafka.consumer.group.id = custom.g.id
```

Example for topic subscription by regex

```
tier1.sources.source1.type = org.apache.flume.source.kafka.KafkaSource
tier1.sources.source1.channels = channel1
tier1.sources.source1.kafka.bootstrap.servers = localhost:9092
tier1.sources.source1.kafka.topics.regex = ^topic[0-9]$
# the default kafka.consumer.group.id=flume is used
```





## Flume的Kafka Sink

![1568722993095](img\1568722993095.png)



另外不建议指定brokerList、Topic 、BatchSize 、requiredAcks 

An example configuration of a Kafka sink is given below. Properties starting with the prefix `kafka.producer` the Kafka producer. The properties that are passed when creating the Kafka producer are not limited to the properties given in this example. Also it is possible to include your custom properties here and access them inside the preprocessor through the Flume Context object passed in as a method argument.

```
a1.sinks.k1.channel = c1
a1.sinks.k1.type = org.apache.flume.sink.kafka.KafkaSink
a1.sinks.k1.kafka.topic = mytopic
a1.sinks.k1.kafka.bootstrap.servers = localhost:9092
a1.sinks.k1.kafka.flumeBatchSize = 20
a1.sinks.k1.kafka.producer.acks = 1
a1.sinks.k1.kafka.producer.linger.ms = 1
a1.sinks.k1.kafka.producer.compression.type = snappy
```



## 说说 Kafka Channel

Memory Channel 有很大的丢数据风险，而且容量一般，File Channel 虽然能缓存更多的消息，**但如果缓存下来的消息还没写入 Sink**，此时 Agent 出现故障则 File Channel 中的消息一样不能被继续使用，直到该 Agent 恢复。而 Kafka Channel 容量大，容错能力强。

有了 Kafka Channel 可以在日志收集层只配置 Source 组件和 Kafka 组件，不需要再配置 Sink 组件，减少了日志收集层启动的进程数，有效降低服务器内存、磁盘等资源的使用率。而日志汇聚层，可以只配置 Kafka Channel 和 Sink，不需要再配置 Source。

`kafka.consumer.auto.offset.reset`，当 Kafka 中没有 Consumer 消费的初始偏移量或者当前偏移量在 Kafka 中不存在（比如数据已经被删除）情况下 Consumer 选择从 Kafka 拉取消息的方式，earliest 表示从最早的偏移量开始拉取，latest 表示从最新的偏移量开始拉取，none 表示如果没有发现该 Consumer 组之前拉取的偏移量则抛出异常。





## Flume的HDFS Sink

Example for agent named a1:

```
a1.channels = c1
a1.sinks = k1
a1.sinks.k1.type = hdfs
a1.sinks.k1.channel = c1
a1.sinks.k1.hdfs.path = /flume/events/%y-%m-%d/%H%M/%S
a1.sinks.k1.hdfs.filePrefix = events-
a1.sinks.k1.hdfs.round = true
a1.sinks.k1.hdfs.roundValue = 10
a1.sinks.k1.hdfs.roundUnit = minute
```

The above configuration will round down the timestamp to the last 10th minute. For example, an event with timestamp 11:54:34 AM, June 12, 2012 will cause the hdfs path to become `/flume/events/2012-06-12/1150/00`.

![1568722765087](img\1568722765087.png)











## Flume方面的面试问题（具体看项目）

***Q:flume怎么配置的？***

source channel  sink



***Q:介绍你项目中的数据流向，并说出在数据流向中遇到的主要问题有哪些？你的flume源码是如何修改的？***

从优化exec监控文件扇出复用？



***Q:为什么用flume拉取日志，别的不行吗？***

Flume更加适合日志的采集，采集源更加丰富



***Q:为什么用flume从kafka拉取日志到HDFS？***

离线处理，flume更适合做日志采集



***Q:flume是如何将数据落地到kafka的，如何保证不会数据丢失***

Source端使用的时Spooling Directory 的方式，可以数据写入到Channel的时候发生丢失

Channel使用的是File Channel，Sink



***Q:flume是如何将数据从kafka中拉取到HDFS的，不会发生数据丢失***





***Q: flume的官网看过吗?简单说一下***





***Q:flume能详细讲解一下吗(非组件)?***





 ***Q:flume如何进行调优?***





***Q:flume source几种方式，分别是什么***





***Q:Flume的source用的什么方式？***





***Q:Flume怎么sink到两个下沉地***





***Q:Flume拉取的文件有多少个***





***Q:Flume源码***





***Q:flume的source的类型,flume是否高可用***





***Q:Flume和kafka采集日志区别，采集日志中间停了，怎么记录之前的日志***