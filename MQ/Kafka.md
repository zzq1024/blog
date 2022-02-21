### 基础

#### 消息传递模式

##### 点对点传递模式：

消息持久化到一个队列中，将有一个或多个消费者消费队列中的数据，但是一条消息只能被消费一次。当一个消费者消费了队列中的某条数据之后，该条数据则从消息队列中删除。该模式即使有多个消费者同时消费数据，也能保证数据处理的顺序；**生产者发送一条消息到queue，只有一个消费者能收到**。

##### 发布-订阅消息传递模式

在发布-订阅消息系统中，消息被持久化到一个topic中，与点对点消息系统不同的是，消费者可以订阅一个或多个topic，消费者可以消费该topic中所有的数据，同一条数据可以被多个消费者消费，数据被消费后不会立马删除。在发布-订阅消息系统中，消息的生产者称为发布者，消费者称为订阅者；**发布者发送到topic的消息，只有订阅了topic的订阅者才会收到消息**。

#### 结构

**Producer**：生产者即数据的发布者，该角色将消息 **推送** 到Kafka的topic中。broker接收到生产者发送的消息后，存储到一个partition中，生产者也可以指定数据存储的partition。

**broker：**Kafka 集群包含一个或多个服务器，服务器节点称为broker；broker存储topic的数据。

**topic：**每条发布到Kafka集群的消息都有一个类别，这个类别被称为Topic。

**partition：**topic中的数据分割为一个或多个partition（每个topic至少有一个partition），partition中的数据是有序的，分区中的每个消息都有一个连续的序列号叫做offset，用来在分区中唯一的标识这个消息。如果topic有多个partition，消费数据时就不能保证数据的顺序。在需要严格保证消息的消费顺序的场景下，需要将partition数目设为1；每个partition有多个副本，其中有且仅有一个作为Leader（负责数据的读写），其他为followers；**followers像普通的consumer那样从leader那里拉取消息并保存在自己的日志文件中。**

**Consumer：**消费者可以从broker中 **拉取** 数据，消费者可以消费多个topic中的数据；consumer保存消息在日志中的偏移量（offset），就可以消费从这个位置开始的消息，customer拥有了offset的控制权，可以向后回滚去重新消费之前的消息。

#### topic分区的好处

- 每个分区都有一个服务器作为“leader”，零或若干服务器作为“followers”，leader负责处理消息的读和写，followers则去复制leader.如果leader down了，followers中的一台则会自动成为leader；集群中的每个服务都会同时扮演两个角色：作为它所持有的一部分分区的leader，同时作为其他分区的followers，这样集群就会据有较好的负载均衡。
- 每个分区在Kafka集群的若干服务中都有副本，使Kafka具备了容错能力。
- 对于消费者来说，可以提高并发度，提高效率；

#### 消费者拉取

**推送：**

缺点：push模式下，由broker决定消息推送的速率，消息系统都致力于让consumer以最大的速率消费消息，当broker推送的速率远大于consumer消费的速率时，consumer恐怕就要崩溃了；

**拉取：**

优点：consumer可以自主控制拉取速率，并决定是否批量的从broker拉取数据；

缺点：如果broker没有可供消费的消息，将导致consumer不断在循环中轮询，直到新消息到t达。为了避免这点，Kafka有个参数可以让consumer阻塞知道新消息到达（当然也可以阻塞知道消息的数量达到某个特定的量这样就可以批量发送）；



### 数据持久化

Kafka大量依赖文件系统去存储和缓存消息；如果将消息以随机写的方式存入磁盘，寻址过程会消耗大量时间，导致磁盘的写入速度会很慢；为了规避随机写带来的时间消耗，KAFKA采取顺序写的方式存储数据，新来的消息只能追加到已有消息的末尾，并且已经生产的消息不支持随机删除以及随机访问，但是消费者可以通过重置offset的方式来访问已经消费过的数据；

### 性能优化

#### 消息集

kafka将消息组织到一起，以消息集为单位处理消息。Producer把消息集一块发送给服务端，而不是一条条的发送；服务端把消息集一次性的追加到日志文件中，这样减少了琐碎的I/O操作。consumer也可以一次性的请求一个消息集；

#### 字节拷贝

kafka采用由Producer，broker和consumer共享的标准化二进制消息格式，这样数据块就可以在它们之间自由传输，无需转换，降低了字节复制的成本开销。也就是说，Kafka的所有消息都需要在生产者端先转化为byte[]数组，而后在消费者端再转换回来，降低了字节复制的成本开销。

#### 零拷贝

先了解将数据从文件传输到socket的公共数据路径：

1.操作系统将数据从磁盘读入到内核空间的页缓存

2.应用程序将数据从内核空间读入到用户空间缓存中

3.应用程序将数据写回到内核空间到socket缓存中

4.操作系统将数据从socket缓冲区复制到网卡缓冲区，以便将数据经网络发出

这里有四次拷贝，两次系统调用，这是非常低效的做法；使用sendfile，只需要一次拷贝就行：允许操作系统将数据直接从页缓存发送到网络上。所以在这个优化的路径中，只有最后一步将数据拷贝到网卡缓存中是需要的。

#### 数据压缩

很多情况下，系统的瓶颈不是CPU或磁盘，而是网络带宽，对于需要在广域网上的数据中心之间发送消息的数据流水线尤其如此。所以数据压缩就很重要。可以每个消息都压缩，但是压缩率相对很低。所以KAFKA使用了批量压缩，即将多个消息一起压缩而不是单个消息压缩；

Kafka采用了端到端的压缩：因为有“消息集”的概念，客户端的消息可以一起被压缩后送到服务端，并以压缩后的格式写入日志文件，以压缩的格式发送到consumer，消息从producer发出到consumer拿到都被是压缩的，只有在consumer使用的时候才被解压缩，所以叫做“端到端的压缩”。

##### 