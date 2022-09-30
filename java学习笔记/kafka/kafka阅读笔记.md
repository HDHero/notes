学习笔记

## 1、kafka

day-2022-09-14

### 04 我应该选择哪种kaka？

kafka manager 帮助监控kafka

### 05 kafka版本号

### 06 kafka线上部署这么做

需要多大硬盘？

假设 每天向kafka集群发送1亿条消息。每条消息保存2份放丢失。消息默认保存时间2周。假设消息的平均大小1kb。则需要1亿 * 1kb *2  / 1000 /1000 = 200GB。索引数据，预留10%磁盘空间，大概220GB。保存2周，220GB * 2 约等于 3TB。kafka支持数据压缩，假设压缩比是0.75 则规范0.75 *3 = 2.25TB。

### 07 最最最重要的集群参数配置

#### 7.1 broker端参数

> **log.dirs:** 重要，指定Broker需要使用的若干个文件目录路径。必填。
>
> log.dir: 单个路径，补充上一个参数。

只要设置log.dirs即可，线上生成环境一定要位log.dirs配置多个路径，格式位CSV格式，如/home/kafka1,/home/kafak2,/home/kafka3，有条件保证这些目录挂载不同物理磁盘上。这样有2个好处：

1、提示读写性能，比起单块磁盘，多个物理磁盘同事读写有更高的吞吐量。

2、能够实现故障转移，从1.1开始，坏掉的磁盘上的数据会自动转移到其它正常磁盘上，而且BBroker能正常工作。

> listeners:监听器，告诉外部连接者通过什么协议访问指定主机名和端口开发的kafka访问
>
> advertised.listeners: advertised公布的，这组监听器是Broker用于对外发布的
>
> host.name/port：不需要指定

监听器，比如CONTROLLER：//localhost:9092

#### 7.2 Zookeeper相关配置

Zookeeper分布式协调框架，负责协调管理并保存kafka集群所有元数据将信息，比如集群都有哪些Broker在运行，创建了哪些Topic，每个topic都有多少分区以及这些分区的leader副本都在哪些机器上等信息。

> zookeeper.connect:zookeeper连接地址

多套可以这样指定：zk1:2181,zk2:2181,zk3:2181

多个kafak机器使用同一套Zookeeper机器，使用chroot，类似于别名

例如：zk1:2181,zk2:2181,zk3:2181/kafka1和zk1:2181,zk2:2181,zk3:2181/kafka2

#### 7.3 TOPIC管理

> auto.create.topics.enable:是否允许自动创建Topic
>
> unclean.leader.election.enable:是否允许Unclean Leader选举
>
> auto.leader.rebalance.enable:是否允许定期进行leader选举

auto.create.topics.enable最好设置false

unclean.leader.election.enable关闭Unclear选举，何谓Unclean，kafka有多个副本，每个分区都有多个副本提供高可用，这些副本中只能有一个副本对外提供服务，即所谓Leader副本，只有保存数据比较多的那些副本能竞选，假设设置了false，坚持之前的原则，坚决不让落后太多副本竞选leader，这样后果就是这个分区不可用，因为没有Leader。如果为true，可以从跑的慢的副本中选出来一个当Leader，这样的后果是数据可能就丢失了，因为这些副本保存的数据不全。

auto.leader.rebalance.enable，假设为ture，表示允许kafka定期的对一些kafka分区进行Leader重选举，代价搞。建议设置false。

 

如果同时设置了Topicj级别参数和全局Broker参数，Topic级别参数会覆盖全局Broker参数的值。

> retention.ms：规定了该Topic消息被保存的时长，默认是7天，一旦设置了这个值，它会覆盖掉Broker端的全局参数值。
>
> retention.bytes：规定了要为该Topic预留多大的磁盘空间。默认值-1，表示无线

#### 7.4 其它

> log.retention.{hour|minutes|ms}:控制一条消息数据保存多长时间，ms设置最高，minutes随后。
>
> log.retention.bytes:指定broker为消息保存的总磁盘大小
>
> message.max.bytes:控制Broker能够接受最大消息大小

log.retention.hour =168 表示保存7天数据

log.retention.bytes 默认-1.表明这台Broker保存多少数据多可以



#### 7.5 JVM参数。

JDK版本尽量选择JDK8.

JVM 堆内存设置为6GB



### 09 kafka分区策略

所谓分区策略是决定生产者将消息发送到哪个分区的算法。

自定义分区是策略需要配置生产者端的参数partitioner.class,实现org.apache.kafka.clients.producer.Partitioner接口。

```java
int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster);
```

​		这里的topic，key，keyBytes，value和valueBytes都属于消息数据，cluster为集群信息（比如当前kafka集群公用多少个主题，多少个broker等）。

​		只要你自己的实现类定义奥了partition方法，同时设置partitioner.class参数为你自己实现类的Full Qualified Name，那么生产者程序就会按照你的代码逻辑对消息进行分区。

#### 9.1 轮询策略

  	 Round-robin策略，即顺序分配，比如一个主题下有3个分区，那么第一条消息被发送到分区0，第二条被发送到分区1，第三条被发送到分区2，以此类推。当生产第四条消息，又被分配到分区0。如果未指定partitioner.class，那么默认按照轮询。轮询策略有非常优秀的负载均衡表现，最大限度地被平均分配到所有分区上。

#### 9.2 随机策略

​		也称Randomness策略，所谓随机就是我们随意地将消息放置到任意一个分区上。

实现随机策略版的partition方法，只需要如下：

```java
// 先技术出该主题总的分区数
List<PartitionInfo> partitions = cluster.paritionsForTopic(topic);
// 然后随机地返回一个小于它的正整数
return ThreadLocalRandom.current().nextInt(partitions.size);
```

#### 9.3 按消息键保序策略

​		也称Key-ordering策略，kafka允许为每条消息定义消息键，简称为key，它可以是一个有着明确业务含义的字符串，比如客户代码、部门编号等。一旦消息被定义了key，那么可以保证同一个key的所有消息都进入到相同的分区里，由于每个分区下的消息处理都是有顺序的，故这个策略被称为按消息键保序策略。

如：

| 分区0     | k1   | k1   | k1   |
| --------- | ---- | ---- | ---- |
| **分区1** | k3   | k3   | k3   |
| **分区2** | k2   | k2   | k2   |

实现这个策略，代码：

```java
List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
return Math.abs(key.hashCode()) % partitions.size();
```

#### 9.4案例 如何实现消息的顺序问题

​		某个企业发送的kafka消息是有因果关系的，故处理因果关系也必须要保证有序，否则先处理了"果"后处理"因"必然造成业务上的混乱。

​		当时这个公司的做法是给kafka主题设置单分区，也就是一个分区。这样所有的消息都只在一个分区内读写，因此保证了全局的顺序性。这样虽然实现了因果关系的顺序性，但也丧失了Kafka多分区带来的高吞吐和负载均衡的优势。经过调研，具有因果关系的消息都有一定特点，比如消息体中都封装了固定的标志位，建议对此标志位设定专门的分区策略，保证同一标志位的所有消息都发送到同一分区。这一思想本质是按消息键保序思想。

#### 9.5 其它分区策略

基于地理位置的分区策略，一般针对大规模的kafka集群，跨城市、跨国家的集群。

根据Broker所在的IP地址实现定制话的分区策略

```java
List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
// 从所有分区中找出那些Leader副本在南方的所有分区，然后随机挑选一个进行消息发送。
return partitions.stream.filter(p->isSouth(p.leader().host))).map(PartitionInfo::partition).findAny();
```

#### 9.5 源码

​		Java客户端默认的生产者分区策略的实现类为org.apace.kafka.clients.producer.internals.DefaultPatitioner。默认策略为：如果指定了partition就直接发送到该分区；如果没有指定分区但是指定了key，就按照key的hash值选择分区；如果partition和key都没有指定就使用轮询策略。<font color='red'>**TODO**</font>

### 10 生产者压缩算法

​		压缩(compression)，用时间来换空间的经典trade-off思想，用CPU时间来换磁盘空间或网络I/O传输量。

#### 10.1 怎么压缩？

kafka消息层次都分为两层：

> 消息集合（message set）一个消息集合中包含若干日志项（record item），儿日志项才是真正封装消息的地方。
>
> 消息（message）

#### 10.2 何时压缩？

压缩可能发生的两个地方：生产者端和Broker端

生产者程序中配置compression.type参数即表示启用指定类型的压缩算法

```java
Properties props = new Properties();
props.put("bootstrap.servers","localhost:9092");
props.put("acks","all");
props.put("key.serializer","org.apache.kafka.common.serialization.StringSerializer");
// 开启GZIP压缩
props.put("compression.type","gizp");
Producer<String,String> producer = new KafkaProducer<>(props);
```

大部分情况，Broker从Producer端接受到消息后仅原封不动地保存儿不会对其进行任何修改。以下情况例外：

情况1：broker端指定了和producer端不同的压缩算法。

情况2：broker端发生了消息格式转换。

#### 10.3 何时解压缩

producer发送压缩消息到broker后，broker会原样保存起来。当consumer程序请求到这部分消息时，Broker依然原样发送出去，当消息到达Consumer端后，由Consumer自行解压缩还原之前的消息。Consumer怎么知道这些消息是用何种压缩算法？答案在消息中，Kafka会将启用了哪注压缩算法封装到消息集合中。

#### 10.4 总结

​		启用压缩的一个条件就是Producer程序允许机器上的CPU资源要充足。如果Producer允许机器本身CPU已经消耗殆尽，那么启用消息压缩无疑是雪上加霜。

​		第二个如果环境中带宽资源有限，建议开启压缩。千兆网络中kafka机器带宽资源耗尽比较容易出现。如果CPU资源富足，建议开启ZSTD压缩，极大节省网络资源消耗。

### 11 无消息丢失配置怎么实现

​		kafka只对“已提交”的消息(committed message)做有限度的持久化保证。

​		**已提交的消息**：当kafka的若干个Broker成功地接受到第一条消息并写入到日志文件后，它们会告诉生产者程序这条消息已成功提交。可以选择只要有一个Broker成功保存该消息就是是已提交，也可以是令所有Broker都成功保存消息才算是已提交。

​		**有限度的持久化保证**：kafka不可能保证在任何情况下做的不丢失消息，举个极端例子，如果地球不存在了，kafka还能保存任何消息吗？其实就是说Kafka不丢消息是有前提条件的，假如你的消息保存在N个Kakfa Broker上，那么这个前提条件就是这个N个Broker中至少有一个存活，只要这个条件成立，kafka就能保证这条消息永远不丢失。

#### 11.1 生产者程序丢失数据

​		Producer程序丢失消息，一个Producer应用向Kafka发送消息，最后发现Kafka没有保存，还不告诉我，目前kafka Producer是异步发送消息的，也就是说如果调用的是<font color='red'>**producer.send(msg)这个api**</font>，那么它通常会立即返回，这种发送方式有趣的名字“fire and forget”，发射后不管。执行完一个操作后不去管它的结果是否成功，producer.send(msg) 如果出现消息丢失，我们是无法知晓的。

​		**因素：**例如网络抖动，导致消息压根就没有发送到Broker端，或者消息本身不合格导致Broker拒绝接受（消息太大等）

​		**解决：**<font color='red'>Proucer永远要使用带有回调通知的发送API</font>，不要使用producer.send(msg)，而要使用producer.send(msg, callback); callback(回调)它能准确地告诉你消息是否真正的提交成功，一旦出现消息提交失败的情况，你就可以针对性地进行处理。重试等。

​		总之，处理发送失败的责任在Producer端而非Broker端。

#### 11.2 消费者丢失数据

**案例1**

​		Consumer端丢失数据主要体现在Consumer端要消费的消息不见了。Consumer程序有个"位移"的概念，表示的是这个Consumer当前消费到的Topic分区的位置。

这里的"位移"类似于我们看书使用的书签，它会标记我们当前阅读了多少也，下次翻书时候直接跳到书签也继续阅读。可能出现的场景：当前书签页是90也，我先将书签也放到100页，之后开始读书，当阅读95页，临时停止阅读。下次直接跳到书签页阅读时，就会丢失96~99页的内容，即这些消息就丢失了。

**解决：**解决上述消息丢失，方法：维持消费消息(阅读)，再更新位移（书签）的顺序即可。这样最大限度保证消息不丢失。当然这种处理方式可能带来问题的是消息的重复处理，类似于同一页书被读了很多遍，后续补充。

**案例2**

​		还有一种，已看书为例，假设租了一本共有10章内容的电子书，该电子数的有效阅读时间是1天。过期后该电子数就无法打开。为加快阅读速度，把书10个章节给10个盆友，并拜托他们告诉你主旨大意。对于kafka而言，这就好不Consumer程序从kafka获取到消息后开启了多个线程异步处理消息，而consumer程序自动地向前更新位移，假如某个线程运行失败了，它负责的消息没有被成功处理，但位移已经被跟新了，因此这条消息对于Consumer而已丢失了。

**解决**

​		如果是多线程异步处理消费消息，Consumer程序不要开启自动提交位移，二手要应用程序手动提交位移。但很难正确的处理位移更新，就是说避免无消费者消息丢失很简单，但是极易出现消息被消费了多次的情况。

#### 11.3 最佳实战

>1、不要使用producer.send(msg);而要使用producer.send(msg, callback);
>
>2、设置acks =all，acks是Producer的一个参数，代表你对"已提交"消息的定义，如果设置成all，则表明所有副本Broker都要接受到消息，该消息才算是”已提交“。
>
>3、设置retries为一个较大的值，这里的retries是producer的参数，对应自动重试。当出现网络的瞬时抖动时，消息发送可能会失败。
>
>4、设置unclean.leader.election.enable =false,这是broker端的参数，它控制的是哪些Broker有资格竞选分区leader，如果一个Broker落后原先的Leader太多的话，那它一旦成为新的Leader，必然会造成消息的丢失，故一般设置false。
>
>5、设置replication.factor>=3，Broker端参数，最后消息多保存几份，数据冗余。
>
>6、设置min.insync.replicas>1，Broker端参数，控制消息至少要写入多少个副本才算是”已提交“，设置大于1可以提示消息持久性。在在实际环境中千万不要使用默认值1.
>
>7、确保replication.factor>min.insync.replicas，如果两者相等，那么只要有一个副本挂机，整个分区就无法正常工作了，推荐设置replication.factor=min.insync.replicas+1
>
>8、确保消息消费完成后再提交。Consumer端有个参数enable.auto.commit，最好设置false，并采用手动提交位移的方式。但Consumer多线程处理的场景而言至关重要。
>
>

### 12 客户端都有哪些不常见但是很高级的功能？

#### 12.1 Kafka拦截器

kafka拦截器分为生产者拦截器和消费者拦截器。生产者拦截器允许发送消息前以及消息提交后植入。通过参数配置完成，interceptor.class

```java
Properties props = new Properties();
List<String> interceptors = new ArrayList<>();
interceptors.add("com.yourcompany.kafkaproject.interceptors.AddTimneInterceptor");// 拦截器1
interceptors.add("com.yourcompany.kafkaproject.interceptors.UpdateTimneInterceptor");// 拦截器2
props.put(ProducerConfig.INTERCEPTOR_CLASSES_CONFIG,interceptors);

```

举个例子，假设你在生产消息钱执行两个“前置”动作；第一个是为消息增加一个头信息，封装发送该消息的时间，第二个是更新发消息字段。那么当你将这两个拦截器串联在一起统一指定给producer后，会按顺序执行上面的动作。

```java
// 生产者拦截器统一继承org.apache.kafka.clients.producer.ProducerInterceptor接口
// 该方法会在消息发送之前被调用，
public ProducerRecord onSend(ProducerRecord<K,V> record);
// 该方法在消息成功提交或发送失败之后被调用，并早于发送回调通知callback
// 此方法处在Producer发送的主路经，减少太重逻辑，否则TSP直线下降
onAcknowledgement();
```

```java
// 消费者拦截器同一继承org.apache.kafka.clients.consumer.ConsumerIntercpetor
// 该方法在消息返回给Consumer程序之前调用，处理消息之前
onConsume();
// Consumer在提交位移之后调用该方法，通常在该方法做一些记账类的动作，打印日志等
onCommit();
```

**使用场景**

kafka拦截器可以应用与包括客户端监控，端到端系统性能检测，消息审计等多种功能。

如今kafka默认提供的监控指标都是针对单个客户端或broker的，很难从具体的消息维度去追踪集群键消息的流转路径。同时，如何监控一条消息从生产者到最后消费的端到端的延迟也是很多用户需要解决的问题。



**案例分析：**

某个业务只有一个Producer和一个Consumer，想知道该业务消息从被生产到最后被消费平均总时长是多少，但是目前kafka并没有提供这种端到端的延迟统计。计算总延迟，一定有个公共的地方保存，让生产者和消费者程序都能访问。假设数据被保存在Redis中。



我们真正消费第一批消息前，首先更新了它们的总延迟，方法就是用当前的时钟减去封装在消息中创建的时间，然后累计到这批消息总的的端到端处理延迟并更新到redis中。之后，便从Redis中读取更新过的总延迟和总消息数，两者相除得到端到端消息的平均延迟。



### 13 Java生产者是如何管理TCP连接的

Apache kafka所有通信都是基于TCP的，而不是基于HTTP或其它协议。

作为一个基于报文的协议，TCP能够被用于多路复用连接的场景前提是，上层的应用协议（比如HTTP）允许发送多条消息。多路复用，指将两个或多个数据流合并到底层单一物理连接中的过程。TCP的多路复用请求会在一条物理连接上创建若干干虚拟连接，每个虚拟连接负责流转各自对应的数据流。

#### 13.1 kafka生产者程序概述

kafka的Java生产者API主要的对象就是KafkaProducer，通常我们开发一个生产者的步骤有4步

- 构造生产者对象所需要的参数对象
- 利用第一步的参数对象，创建KafkaProducer对象实例
- 使用kafkaProducer的send方法发送消息
- 调用kafkaProducer的close方法关闭生产者并释放各种系统资源

Java 代码

```java
Properties props = new Properties();
props.put("参数1","参数1的值");
props.put("参数2","参数2的值");
.....
try (Producer<String, String> producer = new KafkaProducer<>(props)) {
    producer.send(new ProducerRecord<String, String>(), callback);
}
```

#### 13.2 何时创建TCP连接

上述可能创建TCP连接的有：Producer producer = new KafkaProducer(props)和producer.send(msg,callback);

生产者应用在创建KafkaProducer时会创建与Broker的TCP连接。**在创建KafkaProducer实例，生产者应用在后台创建并启动一个名为Sender的线程，该Sender线程开始运行时会首先创建与Broker的连接**.

集群中所有的Broker信息配置到bootstrap.server中，

Producer更新集群元数据信息的两个场景

**场景一：**当Producer尝试给一个不存在的主题发送消息时，Broker会告诉Producer说这个主题不存在，此时Producer会发送到METADATA请求给Kafka集群，尝试获取最新的元数据信息。

**场景二：**Producer通过metadata.max.age.ms参数定期更新元数据信息。不管集群是否有变化，producer每5分钟强制更新一次元数据以保证它是最及时的数据。

问题：在一个有着1000台Broker的集群中，producer可能只会与其中的3-5台Broker长期通信，但是Producer启动后依次创建与这1000台Broker的TCP连接。一段时间后，大约995给TCP连接又被强制关闭，这其实一种资源的浪费。

#### 13.3 何时关闭TCP连接

Producer端关闭TCP连接的方式有2种：一种是用户主动关闭；一种是kafka自动关闭。

第一种： 主动关闭 调用producer.close()方法，kill -9 杀掉进程。

第二种：kafka帮你关闭，kafka端参数：connections。max.idle.ms，默认9分钟，如果9分钟内没有任何请求“流过”某个TCP连接，那么Kafka会主动帮你把TCP连接关闭。用户可以在connections.max.idle.ms =-1禁用掉，变成永久长连接。

### 14 幂等生产者和事务生产者是一回事吗？

所谓的消息交付可靠性保障，是指kafka对Producer和Consumer要处理的消息提供什么样的承若。常见的有3种：

- 最多一次（at most once）：消息可能会丢失，但绝不会被重复消费。
- 至少一次（at least once）：消息不会丢失，但有可能被重复发送。
- 精确一次（exactly once）：消息不会丢失，也不会被重复发送。

​     目前，kafka默认提供的交互可靠性保障是第二种，至少一次。之前提过”已提交“的含义：即Broker成功"提交"消息且Producer端接到Broker的应答才会认为消息成功发送。不过倘若消息成功”提交“，但Broker的应答没有成功发送回producer端（比如网络出现瞬时抖动），那么Producer就无法确定消息是否真的提交成功了。因此它只能选择重试，也就是再次发送相同的消息。这就是Kakfa默认提供至少一次可靠性保障的原因，不过回导致消息重复发送。

如何做到精确一次？

- 幂等性（Idempotence）
- 事务（Transaction）

#### 14.1 幂等性

指某些操作或函数能够执行多次，但每次得到的结果都是不变的。

- 幂等性Producer

  在kafka中，Producer默认不是幂等性的，但是可以创建幂等性Producer

   ```java
  props.put("enable.idempotence","true");
  // 或者
  props.put(PrpducerConfig.ENABLE_IDEMPOTENCE_CONFIG,true);
   ```

  设置完后kafka自动做消息去重。底层实现思想：用空间去换时间，在Broker端多保存一些字段，当Producer发送了具有相同字典值的消息后，Broker能够知晓消息已经重复了，于是可以在后台丢弃。

- 作用范围

  它只能保证单分区上的幂等性，即一个幂等性Producer能够保证某个主题的一个分区上不出现重复消息，它无法实现多个分区幂等性。第二点，它只能实现单回合上的幂等性，不能实现跨会话的幂等性。这里的会话，可以是Producer进程一次运行。当你重启了Producer进程之后，这种幂等性保证就丧失了。

#### 14.2 事务

幂等性无法实现多分区以及多会话上的消息无重复，应该怎么做？答案就是事务，或者依赖事务型Producer。这就是幂等性Producer和事务型Producer的最大区别。

原子性、一致性、隔离性、持久性。

- 事务型Producer

事务性Producer能够保证消息原子性地写入多个分区中。这批消息要么全部写入成功，要么全部失败。另外事务性Producer支持进程的重启，做到精确一次处理。

- 和幂等性Producer一样，开启enable.idepotence = true

- 设置Producer端参数transactionl.id。最后为其设置一个有意义的名字。

  也可以

  ```java
  // 要么全部提交成功，要么全部写入失败
  producer.initTransactions();
  try {
      producer.beginTransaction();
      producer.send(record1);
      producer.send(record2);
      producer.commitTransaction();
  } catch(KafkaException e) {
      producer.abortTransaction();
  }
  ```

  

- 读取事务型Producer发送消息，设置isolation.level参数。

  > 1、 read_uncommitted：默认值，表明Consumer能够读取kafka写入的任何消息，不论事务性Producer提交事务还是终止事务，其写入的消息都可以读取。如果使用室温下Producer，那么对应的Consumer就不要使用这个值。
  >
  > 2、read_commited：表明Consumer只会读取事务性Producer成功提交事务写入消息。当然，可以看到非事务型Producer写入的所有消息。

- 比起幂等性Producer，事务型Producer的性能要更差。

### 15 消费组到底是什么