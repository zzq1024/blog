# 第三节：MySQL实战

### mysql慢查询导致故障

##### 原因：

慢查询导致连接数过多，CPU资源耗尽，服务宕机

##### 快速排查及处理：

查看MySQL监控，用top命令分析服务器负载；show full processlist，查看当前mysql连接执行内容，发现有慢查询SQL经常出现，占用很多CPU；

先将过长线程kill掉，等服务稳定，再根据mysqld_slow.log的信息优化慢查询。

##### 长期解决方案：

使用pdo连接MySQL时，使用长连接，并设置超时时间

```php
<?php
$dbh = new PDO('mysql:host=localhost;dbname=test', $user, $pass, array(
    PDO::ATTR_PERSISTENT => true,//开启长连接
    PDO::ATTR_TIMEOUT    => 60 //超时时间，秒
));
?>
```



### 减库存的方法

```sql
start transaction;
update t set num=num-1 where num>0;//防止并发库存减成负数
affected_rows;//判断上边更新是否产生影响
```



### MySQL分库分表环境下全局ID生成方案

#### Sequence ID

数据库自增长序列或字段，最常见的方式，由数据库维护，数据库唯一。设置的起始数字不一样，步长一样，比如：Master1 生成的是 1，4，7，10，Master2生成的是2,5,8,11 Master3生成的是 3,6,9,12。这样就可以有效生成集群中的唯一ID，也可以大大降低ID生成数据库操作的负载。

缺点：要做数据库拓展时，极其麻烦。

#### UUID

常见的方式,128位。可以利用数据库也可以利用程序生成，一般来说全球唯一。

优点：

1. 简单，代码方便。
2. 全球唯一，在遇见数据迁移，系统数据合并，或者数据库变更等情况下，可以从容应对。

缺点：

1. 没有排序，无法保证趋势递增。
2. UUID往往是使用字符串存储，查询的效率比较低。
3. 存储空间比较大，如果是海量数据库，就需要考虑存储量的问题。
4. 传输数据量大
5. 不可读。

优化方案：

为了解决UUID不可读，可以使用UUID to Int64的方法。

#### Twitter的snowflake算法

snowflake是Twitter开源的分布式ID生成算法，结果是一个long型的ID。其核心思想是：使用41bit作为毫秒数，10bit作为机器的ID（5个bit是数据中心，5个bit的机器ID），12bit作为毫秒内的流水号（意味着每个节点在每毫秒可以产生 4096 个 ID），最后还有一个符号位，永远是0。snowflake算法可以根据自身项目的需要进行一定的修改。比如估算未来的数据中心个数，每个数据中心的机器数以及统一毫秒可以能的并发数来调整在算法中所需要的bit数。

优点：

1. 不依赖于数据库，灵活方便，且性能优于数据库。
2. ID按照时间在单机上是递增的。

缺点：

在单机上是递增的，但是由于涉及到分布式环境，每台机器上的时钟不可能完全同步，也许有时候也会出现不是全局递增的情况。



### 分区分表

#### 分表

##### 实现分表的方法

水平分表：手动创建多张表，通过PHP算法判断实现读写（方法：ID取模、按年份）
垂直分表：将表字段拆分到其他表中（将表中字段分为常用字段和不常用字段，分别存入两个表中）

##### 缺点

水平分表或垂直分表，虽然可以加快查询速度，但却需要手动在应用层写逻辑代码，比较麻烦，增加了代码复杂度。

#### 分区

##### 分区算法

- key分区

  ```sql
  #创建数据库db
  create database db;
  #选择数据库
  use db;
  #创建表
  create table articles(
    id int unsigned primary key auto_increment,
    title varchar(50) not null,
    content text
  ) engine = myisam charset = utf8
  partition by key(id) partitions 3;
  #通过key算法求余id字段，分3个区
  ```

- hash分区

  ```sql
  #创建表
  create table articles(
    id int unsigned primary key auto_increment,
    title varchar(50) not null,
    content text
  ) engine = myisam charset = utf8
  partition by hash (id) partitions 4;
  ```

- list分区

  ```sql
  #创建表
  create table articles(
    id int unsigned auto_increment,
    title varchar(50) not null,
    content text,
    cid int unsigned,
  primary key (id,cid)
  ) engine = myisam charset = utf8
  partition by list(cid) (
  partition c1 values in (1,3),
  partition c2 values in (2,4),
  partition c3 values in (5,6,7)
  );
  ```

- range分区

  ```sql
  #创建数据表并实现range分区
  create table user(
    id int not null auto_increment,
    name varchar(40) not null,
    birthday date not null default '0000-00-00',
    primary key(id,birthday)
  ) engine = myisam default charset = utf8
  partition by range(year(birthday)) (
    partition 70hou values less than (1980),
    partition 80hou values less than (1990),
    partition 90hou values less than (2000),
    partition 00hou values less than (2010)
  );
  ```

##### 分区特点

- MySQL 在第一次打开分区表的时候，需要访问所有的分区；

- 在 server 层，认为这是同一张表，因此所有分区共用同一个MDL（元数据）锁；
- 在引擎层，认为这是不同的表，因此 MDL 锁之后的执行过程，会根据分区表规则，只访问必要的分区;