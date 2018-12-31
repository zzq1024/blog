# 第一节：Redis

### 基础

#### Redis如何淘汰过期的keys

Redis keys过期有两种方式：被动和主动方式。
当一些客户端尝试访问它时，key会被发现并主动的过期。
当然，这样是不够的，因为有些过期的keys，永远不会访问他们。 无论如何，这些keys应该过期，所以定时随机测试设置keys的过期时间。所有这些过期的keys将会从密钥空间删除。

#### phpredis:connect与pconnect

* connect：脚本结束之后连接就释放了。

* pconnect：脚本结束之后连接不释放，连接保持在php-fpm进程中。

  所以使用pconnect代替connect，可以减少频繁建立redis连接的消耗。

  [pconnect弊端](https://www.v2ex.com/t/95635)