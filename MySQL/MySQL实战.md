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

