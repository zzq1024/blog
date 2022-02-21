# 第五节：ElasticSearch

### 基础

ES不是数据库，它适合于海量数据、更新频率很低的数据(ES没有事务也不适合处理并行更改数据)，创建文档后，es不支持更新域（字段类型）。
ES是全文搜索引擎，能够对中文进行全文搜索，处理同义词和根据相关性给文档打分；能根据同一份数据生成分析和聚合的结果；在没有大量工作进程（线程）的情况下能做到对数据的实时处理。

#### 字段类型

ElasticSearch 5.0以后，字符串类型有重大变更，移除了string类型，string字段被拆分成两种新的数据类型， text和keyword：

**text**：会分词，然后进行索引，用于全文搜索；支持模糊、精确查询；不支持聚合（使用keyword进行聚合）。

**keyword**：不进行分词，直接索引，keyword用于关键词搜索；支持模糊、精确查询；支持聚合。

#### 检索

term会精准查询，match会进行分词查询；

index设置成false，不支持搜索，支持terms聚合；

enable设置成false，将无法进行搜索和聚合分析；

#### Mapping

Dynamic = true：一旦有新增字段写入，Mapping也会被更新

Dynamic = false：Mapping不会被更新，新增字段的数据无法被索引，但是信息会出现在_source中

Dynamic = strict：文档写入新字段时，直接失败

#### 集群
集群这个概念从业务层面来看，是聚合了某方面功能的节点的组合，一般由单个或多个节点组成。在ES中，具有相同的cluster.name的节点就为同一个集群， 一个集群就是由一个或多个拥有相同cluster.name的节点组成。
#### 节点
一个运行中的ES实例被成为一个节点，一个或多个节点（ES实例）则组成一个集群。
每个集群中会有主节点，ES会自动进行主节点选举，对用户透明。主节点的主要职责是管理集群范围內的数据变更，包括索引的增加／删除 ，节点的增加／删除，监控节点的状态等，它的工作负荷相对其他节点而言较轻，当集群流量增加时，它不会成为瓶颈。从节点负责具体的细节变更，比如文档级别的变更或搜索。各个节点能均匀地分摊集群的负载，实现负载均衡，以及数据冗余备份，当某个节点因为硬件故障丢失数据时，其他节点存放的冗余数据可以保证整个集群的数据完整性。

**客户端节点：**该节点只能处理路由请求，处理搜索，分发索引操作等；提供负载均衡功能；

**数据节点：**主要是存储索引数据的节点，**主要对文档进行增删改查操作，聚合操作**等。数据节点对cpu，内存，io要求较高， 在优化的时候需要监控数据节点的状态，当资源不够的时候，需要在集群中添加新的节点；

**主节点：**主要职责是和集群操作相关的内容，如**创建或删除索引，跟踪哪些节点是群集的一部分，并决定哪些分片分配给相关的节点**；稳定的主节点对集群的健康是非常重要的，默认情况下任何一个集群中的节点都有可能被选为主节点，索引数据和搜索查询等操作会占用大量的cpu，内存，io资源，为了确保一个集群的稳定，分离主节点和数据节点是一个比较好的选择。  

#### 分片
分片是一个底层的工作单元每个分片都是一个Lucene实例，每个Lucene实例都是一个独立的搜索引擎每个分片只存放索引全部数据中的一部分；
分片是节点中存储数据的核心单元，每个索引的文档数据是均匀地存放到多个分片中的（默认为5个），每个分片只能存储20亿个文档；一个集群是由一个或多个节点（ES实例）组成，一个节点又是由一个或多个分片（Lucene实例）组成；

分片有主分片和副本分片的区别，索引的内容均匀地放到各个主分片中，每个文档都属于某一个主分片；副本分片是主分片的冗余拷贝，和对应的主分片有完全一样的数据；主分片的数量在索引建立时会被定死，副本分片的数量没有限制，副本分片越多，集群的规模会相应的扩大，海量的检索请求会分散到各个副本上，系统的吞吐量会有成倍的提升。主分片数在索引创建后，（由于文档路由已经定了）不能修改，只能reindex；

#### 副本的作用

- 故障转移/集群恢复：海量的检索请求会分散到各个副本上，系统的吞吐量会有成倍的提升

- 通过副本进行负载均衡：此外副本策略提供了高可用和数据安全的保障，当分片所在的机器宕机，Elasticsearch可以使用其副本进行恢复，从而避免数据丢失。 

##### 分片及其生命周期

1）客户端发起数据写入请求，对你写的这条数据根据routing规则选择发给哪个Shard。
确认Index Request中是否设置了使用哪个Filed的值作为路由参数，
如果没有设置，则使用Mapping中的配置，
如果mapping中也没有配置，则使用_id作为路由参数，然后通过_routing的Hash值选择出Shard，最后从集群的Meta中找出出该Shard的Primary节点。
2）写入请求到达Shard后，先把数据写入到内存（buffer）中，同时会写入一条日志到translog日志文件中去。
当写入请求到shard后，首先是写Lucene，其实就是创建索引。
索引创建好后并不是马上生成segment，这个时候索引数据还在缓存中，这里的缓存是lucene的缓存，并非Elasticsearch缓存，lucene缓存中的数据是不可被查询的。
3）执行refresh操作：从内存buffer中将数据写入os cache(操作系统的内存)，产生一个segment file文件，buffer清空。
写入os cache的同时，建立倒排索引，这时数据就可以供客户端进行访问了。
默认是每隔1秒refresh一次的，所以es是准实时的，因为写入的数据1秒之后才能被看到。
buffer内存占满的时候也会执行refresh操作，buffer默认值是JVM内存的10%。
通过es的restful api或者java api，手动执行一次refresh操作，就是手动将buffer中的数据刷入os cache中，让数据立马就可以被搜索到。
若要优化索引速度, 而不注重实时性, 可以降低刷新频率。
4）translog会每隔5秒或者在一个变更请求完成之后，将translog从缓存刷入磁盘。
translog是存储在os cache中，每个分片有一个，如果节点宕机会有5秒数据丢失，但是性能比较好，最多丢5秒的数据。。
可以将translog设置成每次写操作必须是直接fsync到磁盘，但是性能会差很多。
可以通过配置增加transLog刷磁盘的频率来增加数据可靠性，最小可配置100ms，但不建议这么做，因为这会对性能有非常大的影响。
5）每30分钟或者当tanslog的大小达到512M时候，就会执行commit操作（flush操作），将os cache中所有的数据全以segment file的形式，持久到磁盘上去。
第一步，就是将buffer中现有数据refresh到os cache中去。
清空buffer 然后强行将os cache中所有的数据全都一个一个的通过segmentfile的形式，持久到磁盘上去。
将commit point这个文件更新到磁盘中，每个Shard都有一个提交点(commit point), 其中保存了当前Shard成功写入磁盘的所有segment。
把translog文件删掉清空，再开一个空的translog文件。
flush参数设置：
index.translog.flush_threshold_period:
index.translog.flush_threshold_size:
#控制每收到多少条数据后flush一次
index.translog.flush_threshold_ops:
6）Segment的merge操作：
随着时间，磁盘上的segment越来越多，需要定期进行合并。
Es和Lucene 会自动进行merge操作，合并segment和删除已经删除的文档。
我们可以手动进行merge：POST index/_forcemerge。一般不需要，这是一个比较消耗资源的操作。

#### 查询

**match：**基于全文的查询，先对输入的查询进行分词，然后对每个词逐个查询，最后将结果合并；

**match_phrase：**基于全文的查询，但是会精确匹配位置；

**term：**完全匹配，即不进行分词器分析，文档中必须包含整个搜索的词汇

注：如果字段设置了keyword，你用term查询，就会精确匹配。例如说keyword字段，索引时是“Iphone”，你的term查询必须是Iphone，输入“iphone”就无法匹配；而如果你的字段是“text”类型。你index时候，如果是“Iphone”，在term查询时，“iphone”可以匹配。但是，“Iphone”不会（背后的原因是，text类型的数据会分词，默认分词器会将输入一个个单词切开，并且转小写了，所以你 term查询时，必须用“iphone”）。

### 坑
#### 分页
ES默认的分页机制一个不足的地方是，比如有5010条数据，当你仅想取第5000到5010条数据的时候，ES也会将前5000条数据加载到内存当中，ES为了避免用户的过大分页请求造成ES服务所在机器内存溢出，默认对深度分页的条数进行了限制，默认的最大条数是10000条。
解决方案1：
修改配置，调整的新的窗口数：

```
curl -XPUT http://127.0.0.1:9200/my_index/_settings -d '{ "index" : { "max_result_window" : 500000}}'。
```
缺点：窗口值调大了后，虽然请求到分页的数据条数更多了，但它是用牺牲更多的服务器的内存、CPU资源来换取的。
解决方案2：
https://www.elastic.co/guide/en/elasticsearch/reference/5.6/search-request-search-after.html，利用search_after参数
scroll查询原理是在第一次查询的时候一次性生成一个快照，根据上一次的查询的id来进行下一次的查询，这个就类似于关系型数据库的游标，然后每次滑动都是根据产生的游标id进行下一次查询，这种性能比上面说的分页性能要高出很多，基本都是毫秒级的。
缺点：不支持跨页查询
解决方案3：
每次查询超过10000万条记录的时候，都会去更新一次index。
缺点：慢

### 命令

查看已装插件：bin/elasticsearch-plugin list

安装插件：bin/elasticsearch-plugin install analysis-icu

开启多节点（同一机器开启多节点，配置文件中不要指定端口，Elasticsearch 会取用9200~9299这个范围内的端口，如果9200被占用，就选择9201，依次类推）：

bin/elasticsearch -E node.name=node0 -E cluster.name=zzq -E path.data=node0_data -E http.port=9200 -d

bin/elasticsearch -E node.name=node1 -E cluster.name=zzq -E path.data=node1_data -E http.port=9201 -d

### cerebro（ES监控工具）

运行：bin/cerebro

指定端口运行：bin/cerebro -Dhttp.port=9100

浏览器打开：http://127.0.0.1:9100

### logstash（导入数据到ES，注：两者版本一致）

导入数据：sudo bin/logstash -f logstash.conf（注：需要使用 sudo ，否则命令执行完成后 没有权限操作相关文件）

### 版本更新

ES从5.X就引入了text和keyword，其中keyword用于不分词字段，搜索时只能完全匹配；到了6.X就彻底移除string了，“index”的值就只能是boolean变量了。

### 日常问题

#### 将分片分配给node节点

```
POST 10.66.96.204:9200/_cluster/reroute
{
	"commands":[{
		"allocate_stale_primary":{
			"index":"index_sms_long",//索引名称
			"shard": 22,//分片序号
			"node":"dvygzuZ5SYejNREJ7ohE3w",//node节点
			"accept_data_loss":true
		}
	}
		]
}
```

