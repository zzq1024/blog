### 优势

#### 详尽的文档和测试用例

- 通过shell查看文档：restydoc [函数名称]
- 测试样例：<https://github.com/openresty/lua-nginx-module/tree/master/t>

#### 同步非阻塞

- 同步异步说的是代码，调用就有返回是同步，反之是异步。
- 阻塞非阻塞说的是cpu，cpu要等待就是阻塞，反之非阻塞。
- 非阻塞并不能缩减rt时间，其最大的优点是可以服务更多的请求，达到c100k.

#### 动态

通过OpenResty中的lua-nginx-module模块中提供的Lua Api，我们可以动态的控制路由、上游、SSL证书、请求、响应等，你可以在不重启OpenResty，（不开启lua_code_cache的前提下），修改代码。



### 库

Lua世界的库很可能会带来阻塞，让原本高性能的服务，直接下降几个数量级，而是应该使用lua-resty-*(使用cosocket实现)的库。

#### 包管理

- OPM（<https://opm.openresty.org/>）：OpenResty自带的包管理工具，查找http请求的库：opm search http。

- LUAROCKS（<https://luarocks.org/>）：OpenResty自带的另一个包管理工具，只包含OpenResty相关的包及Lua相关的库，查找http的库：luarocks search http，Lua可以编译C代码进行使用。

  OPM与LUAROCKS都不支持私有包。

- AWESOME-RESTY（<https://github.com/bungle/awesome-resty>）：维护了几乎所有OpenResty可用的包。

#### 库优先级

OpenResty的API （ngx.*）> LuaJIT的库函数 > 标准Lua的函数，对性能有影响。

#### lua-resty-core

lua-resty-core 中不仅重新实现了部分 lua-nginx-module 项目中的 API，新的API也通过 FFI 的方式（可以被luajit优化），在 lua-resty-core 仓库中实现，可以提升性能；在 OpenResty 2019 年 5 月份发布的 1.15.8.1 版本前，lua-resty-core 默认是不开启的，需要在 init_by_lua 阶段：require "resty.core"。



### Lua与LuaJIT

标准 Lua 和 LuaJIT 是两回事儿，LuaJIT 只是兼容了 Lua 5.1 的语法。

OpenResty现在已经去掉了对标准 Lua 的支持，只支持 LuaJIT；OpenResty 维护了自己的 LuaJIT 分支，并扩展了很多独有的 API。

#### Lua

运行时，标准 Lua 出于性能考虑，内置了虚拟机，先由 Lua 编译器编译为字节码（Byte Code），然后再由 Lua 虚拟机执行；

#### LuaJIT

不同于Lua，LuaJIT 的解释器会在执行字节码的同时，记录一些运行时的统计信息，比如每个 Lua 函数调用入口的实际运行次数，还有每个 Lua 循环的实际执行次数；当这些次数超过某个随机的阈值时，便认为对应的 Lua 函数入口或者对应的 Lua 循环足够热，这时便会触发 JIT 编译器开始工作（注：这里的字节码是 LuaJIT 虚拟机执行的一种指令格式；机器码是指 CPU 可以读取的指令格式。）。

JIT 编译器会从热函数的入口或者热循环的某个位置开始，尝试编译对应的 Lua 代码路径；编译的过程，是把 LuaJIT 字节码先转换成 LuaJIT 自己定义的中间码（IR），然后再生成针对目标体系结构的机器码。

所谓 LuaJIT 的性能优化，本质上就是让尽可能多的 Lua 代码可以被 JIT 编译器生成机器码，而不是回退到 Lua 解释器的解释执行模式。



### API

#### ngx.re.split

syntax: res, err = ngx_re.split(subject, regex, options?, ctx?, max?, res?)

字符串切割，lua-resty-core/lib/ngx/re.md 这个第三级目录的文档中出现;

```lua
local ngx_re = require "ngx.re"
local res, err = ngx_re.split("a,b,c,d", ",")
ngx.say(res)
-- res is now {"a", "b", "c", "d"}
```

#### ngx.now

这些返回当前时间的 API，如果没有非阻塞网络 IO 操作来触发，便会一直返回缓存的值，而不是像我们想的那样，能够返回当前的实时时间。

使用 Lua 的阻塞函数 sleep 了 1 秒钟，但从打印的结果来看，这两次返回的时间戳却是一模一样的:

```lua
ngx.say(ngx.now())
os.execute("sleep 1")
ngx.say(ngx.now())
```

非阻塞的 sleep 函数，就会打印出不同的时间戳了；

```lua
ngx.say(ngx.now())
ngx.sleep(1)
ngx.say(ngx.now())
```

ngx_cached_time 的值只在函数 ngx_time_update 中会更新，非阻塞函数会打印出不同时间戳，说明协程挂起（yield）后会执行ngx_time_update；

####ngx.sleep

这个非阻塞的 sleep 函数，这个函数除了可以休眠指定的时间外---比如你有一段正在做密集运算的代码，需要花费比较多的时间，那么在这段时间内，这段代码对应的请求就会一直占用着 worker 和 CPU 资源，导致其他请求需要排队，无法得到及时的响应。这时，我们就可以在其中穿插 ngx.sleep(0)，使这段代码让出控制权，让其他请求也可以得到处理。

#### table

- table的赋值，其实传递的是地址，操作被赋值的表内的元素，会影响原来的table:

  ```lua
  --备用，打印table的元素
  function printT(t)       
      print('{'..table.concat(t,',')..'}')
  end
  local t1 = {1,2,3}
  local t2 = t1
  t2[1] = 3
  printT(t1)                  -- {3,2,3}
  printT(t2)                  -- {3,2,3}
  --此时若打印t1,t2的地址，会发现是一样的
  print(t1)                    -- table: 008D3A48
  print(t2)                    -- table: 008D3A48
  ```

- \#t：获取table长度，时间复杂度O(n)

#### string

- string.sub("abcd", 1, 1)：截取字符串，这个函数会有隐蔽的临时字符串的产生；可替代的方法：string.char(string.byte("abcd", 1,1)));
- string.format：这种拼接字符串消耗大，效率低，可以用数组(table.concat)的方式拼接；

#### lua-rapidjson

[lua-rapidjson](https://github.com/xpol/lua-rapidjson) 是基于 [RapidJSON](http://rapidjson.org/) 的 Lua 模块，由于 RapidJSON 性能的优化，目前在许多情况下 lua-rapidjson 都要比 lua-cjson 快；与cjson不同之处是支持 JSON Schema：

```lua
local rapidjson = require('rapidjson')--引入第三方模块

local rules = {
    type = "object",
    properties = {
        count = {type = "integer", minimum = 0},
        time_window = {type = "integer",  minimum = 0},
        key = {type = "string", enum = {"remote_addr", "server_addr"}},
        rejected_code = {type = "integer", minimum = 200, maximum = 600},
    },
    additionalProperties = false,
    required = {"count", "time_window", "key", "rejected_code"},
}
-- JSON Schema
local schema = rapidjson.SchemaDocument(rules)
local validator = rapidjson.SchemaValidator(schema)
-- 获取body中参数
ngx.req.read_body()
local params = rapidjson.decode(ngx.req.get_body_data())
-- 参数验证
local ret, err = validator:validate(rapidjson.Document(params))
ngx.say(ret, err) --true,nil
```



### 数据共享

#### Nginx变量

ngx.var 的生命周期和请求一致，它可以在 Nginx C 模块之间共享数据，自然的，也可以在 C 模块和 OpenResty 提供的 lua-nginx-module 之间共享数据；不过，使用 Nginx 变量这种方式来共享数据是比较慢的，因为它涉及到 hash 查找和内存分配。同时，这种方法有其局限性，只能用来存储字符串，不能支持复杂的 Lua 类型。

```nginx
location /foo {
     set $my_var ''; # this line is required to create $my_var at config time
     content_by_lua_block {
         ngx.var.my_var = 123;
         ...
     }
 }
```

ngx.var可以在子请求中有效，也就是说可以跨location使用，而ngx.ctx不可以；

#### ngx.ctx

**可以在同一个请求的不同阶段之间共享数据**，速度很快，还可以存储各种 Lua 的对象，它的生命周期是请求级别的，当一个请求结束的时候，ngx.ctx 也会跟着被销毁掉；我们可以用 ngx.ctx 来缓存 Nginx 变量（ngx.var）这种昂贵的调用，并在不同阶段都可以使用到它：

```nginx
location /test {
     rewrite_by_lua_block {
         ngx.ctx.host = ngx.var.host
     }
     access_by_lua_block {
        if (ngx.ctx.host == 'openresty.org') then
            ngx.ctx.host = 'test.com'
        end
     }
     content_by_lua_block {
         ngx.say(ngx.ctx.host)
     }
 }
```

需要特别注意的是，正因为 ngx.ctx 的生命周期是请求级别的，所以它并不能在模块级别进行缓存，错误示例：

```lua
local ngx_ctx = ngx.ctx

local function bar()
    ngx_ctx.host =  'test.com'
end
```

我们应该在函数级别进行调用和缓存：

```lua
local ngx = ngx

local function bar()
    ngx.ctx.host =  'test.com'
end
```

#### 模块级别的变量

**在同一个 worker 内的所有请求之间共享数据**，示例中的mydata 就是一个模块，它只会被 worker 进程加载一次，之后，这个 worker 处理的所有请求，都会共享 mydata 模块的代码和数据，mydata 模块中的 data 这个变量，就是 模块级别的变量，它位于模块的 top level，也就是模块最开始的位置，所有函数都可以访问到它，可以把需要在请求间共享的数据，放在模块的 top level 变量中；

不过，需要特别注意的是，一般我们只用这种方式来保存**只读的数据**（建议只读）；如果涉及到写操作，你就要非常小心了，因为可能会有 race condition（竞争条件），这是非常难以定位的 bug。

```lua
-- mydata.lua
 local _M = {}

 local data = {
     dog = 3,
     cat = 4,
     pig = 5,
 }

 function _M.incr_age(name)
     data[name]  = data[name] + 1
    return data[name]
 end

 return _M
```

```nginx
location /lua {
     content_by_lua_block {
         local mydata = require "mydata"
         ngx.say(mydata. incr_age("dog"))
         ngx.sleep(5) -- yield API
         ngx.say(mydata. incr_age("dog"))
     }
 }
```

在模块中，有 incr_age 这个函数，它会对 data 这个表的数据进行修改；在调用的代码中，我们增加了最关键的一行 ngx.sleep(5)，这个 sleep 是一个 yield 操作（非阻塞操作，会产生竞争，其他还有：访问 Redis 等）， sleep 的 5 秒钟内，也很可能就有其他请求调用了mydata. incr_age 函数，修改了变量的值，从而导致最后输出的数字不连续。

#### ngx.shared.DICT

只能存储字符串类型的数据，**多个worker之间共享数据**。

- ngx.share.dict.get(key)：如果键不存在或已过期，则将返回nil；如果出现错误，将返回nil和一个描述错误的字符串；当它们被插入字典时，返回的值将具有原始数据类型，例如Lua布尔值、数字或字符串。

- ngx.share.dict.set(key, value, exptime?, flags?)：

  返回：success（返回值）；err（错误信息）；forcible（一个布尔值，指示在共享内存区域中的存储空间不足时是否已强行删除了其他有效项）；

  插入的值参数可以是Lua布尔值、数字、字符串或nil；可选的exptime参数指定插入的键值对的过期时间（以秒为单位）。时间分辨率为0.001秒。如果exptime采用值0（这是默认值），则该项将永不过期；optional flags参数指定与要存储的项关联的用户标志值，稍后也可以使用该值检索它；当未能分配内存时，SET将尝试根据最近最少使用的（LRU）算法删除存储中的现有项，LRU优先于到期时间；如果多达几十个现有项目已经被删除，并且存储剩下的仍然不够（要么由于LuaHySydDyDICT或内存分段指定的总容量限制），然后success将是false，err将是no memory；

- ngx.share.dict.add(key, value, exptime?, flags?)：只有当key不存在时才会插入值；如果key存在，success=>false，err=>exists；

- ngx.share.dict.replace(key, value, exptime?, flags?)：仅在键确实存在的情况下才将键值对存储到字典ngx.shared.DICT中，如果key不存在，success=>false，err=>not found；

- ngx.share.dict.add(key)：在基于ngx.shared.DICT中，将key值增加步长值， 如果操作成功完成，则返回新的结果编号，否则返回nil，否则返回错误消息。如果密钥在共享字典中不存在或已过期，如果未指定init参数或值为nil，则此方法将返回nil和错误字符串“未找到”，如果init参数采用数字值，则此方法将使用init + value值创建一个新键（可选的init_ttl参数指定通过init参数初始化值时的到期时间（以秒为单位））。

- ngx.share.dict.lpush(key,value)：将指定的（数字或字符串）值插入基于字典ngx.shared.DICT中名为key的列表的开头， 返回推送操作后列表中的元素数。当共享内存区域中的存储空间用完时，它永远不会覆盖存储中（最近使用最少的）未过期项目；

- ngx.share.dict.rpush(key,value)：与lpush方法类似，但是将指定的（数字或字符串）值插入名为key的列表的末尾；

- ngx.share.dict.lpop(key):删除并返回基于字典ngx.shared.DICT中名为key的列表的第一个元素。

- ngx.share.dict.rpop(key):删除并返回基于字典ngx.shared.DICT中名为key的列表的最后一个元素。

- ngx.share.dict.llen(key)：返回列表长度。

- ngx.share.dict.ttl(key)：检索基于shm的字典ngx.shared.DICT中的键值对的剩余TTL（生存时间（以秒为单位））；

- ngx.share.dict.expire(key，expiretime)：更新基于shm的字典ngx.shared.DICT中的键/值对的exptime（以秒为单位）；

- ngx.share.dict.flush_all()：清除字典中的所有项目。 此方法实际上不会释放字典中的所有存储块，而只是将所有现有项标记为已过期。

- ngx.share.dict.expire_all()：清除字典中的过期项目，直到可选的max_count参数指定的最大数目。 如果将max_count参数设置为0或根本不设置参数，则表示无限制。 返回实际已刷新的项目数

- ngx.share.dict.get_keys()：从字典中获取max_count个键列表，默认情况下，仅返回前1024个键（如果有）， 当<max_count>参数的值设置为0时，即使字典中有1024个以上的键，也将返回所有键；注意避免在具有大量键的字典上调用此方法，因为它可能会锁定字典相当长的时间，并阻止尝试访问字典的Nginx工作进程。

##### 缺点

1.worker进程之间会有锁竞争，在高并发的情况下会增加性能开销

2.只支持Lua布尔值、数字、字符串和nil类型的数据，无法支持table类型的数据

3.在读取数据时有反序列化操作，会增加CPU开销

#### lua-resty-lrucache

一个使用 LuaJIT FFI 实现的 LRU 缓存库，这个库为OpenResty和ngx_lua模块实现了一个简单的LRU缓存，可以在 worker 内缓存各种类型的数据（而无需序列化开销）；在大多数实际情况下，lrucache 作为一级缓存，shared dict 作为二级缓存。

这个库以两个类的形式提供了两个不同的实现：resty.lrucache和resty.lrucache.pureffi。两者都实现相同的API。唯一的区别是前者使用本地Lua tables，而后者是一个纯FFI实现，它还实现了一个用于缓存查找的基于FFI的哈希表。由于Lua tables不擅长频繁地删除键，如果缓存命中率相对较高，则应使用比resty.lrucache.pureffi更快的resty.lrucache类；如果缓存命中率相对较低，并且在缓存中插入和删除的键可能有很多变化，那么应该使用resty.lrucache.purefi代替（如果在这样的用例中使用resty.lrucache类而不是resty.lrucache.purefi，那么您可能会看到LuaJIT运行时中的resizetab函数调用在CPU flame图上非常热）。

- syntax: cache, err = lrucache.new(max_items [, load_factor])
- syntax: cache:set(key, value, ttl?, flags?)
- syntax: data, stale_data, flags = cache:get(key)
- syntax: cache:delete(key)
- syntax: count = cache:count()
- syntax: size = cache:capacity()
- syntax: keys = cache:get_keys(max_count?, res?)
- syntax: cache:flush_all()

注：开启lua_code_cache才生效；

##### 缺点

1.因为数据不在worker之间共享，所以无法保证在更新数据时，数据在同一时间的不同worker进程上完全一致。

2.虽然可以支持复杂的数据结构，但可使用的指令却很少，如不支持消息队列功能

3.重载Nginx配置时，缓存数据会丢失。如果使用lua_shared_dict，则不会如此

#### lua-resty-mlcache

缓存级别层次结构为：

L1 缓存就是 lua-resty-lrucache。每一个 worker 中都有自己独立的一份，有 N 个 worker，就会有 N 份数据，自然也就存在数据冗余。由于在单 worker 内操作 lrucache 不会触发锁，所以它的性能更高，适合作为第一级缓存。
L2 缓存是 shared dict。所有的 worker 共用一份缓存数据，在 L1 缓存没有命中的情况下，就会来查询 L2 缓存。ngx.shared.DICT 提供的 API，使用了自旋锁来保证操作的原子性，所以这里我们并不用担心竞争的问题；
L3 则是在 L2 缓存也没有命中的情况下，需要执行回调函数去外部数据库等数据源查询后，再缓存到 L2 中。在这里，为了避免缓存风暴，它会使用 lua-resty-lock ，来保证只有一个 worker 去数据源获取数据。

mlcache提供了 `l1_serializer`，专门用于处理 shared dict 提升到 lru cache 时候对数据的处理，可以尽可能的减少序列化。



### 执行阶段

#### init_by_lua_*

当Nginx master进程加载Nginx配置文件时，在全局Lua VM级别上运行由参数<Lua script str>指定的Lua代码；当Nginx接收到HUP信号并开始重新加载配置文件时，Lua VM也将被重新创建，init_by_Lua将在新的Lua VM上再次运行。如果lua_code_cache指令被**关闭**（默认为打开），init_by_lua处理程序将在每个请求时运行，因为在这种特殊模式下，总是为每个请求创建一个独立的lua VM；您可以通过这个阶段在服务器启动时预加载Lua模块，并利用现代操作系统的copy-on-write（COW）优化，由于此阶段中的Lua代码在Nginx fork其worker进程之前运行，因此此处加载的数据或代码将享受所有worker进程中操作系统提供的写时拷贝（COW）功能，从而节省大量内存。

不要在此阶段初始化自己的Lua全局变量，因为使用Lua全局变量会降低性能，并可能导致全局命名空间污染；建议的方法是使用正确的Lua模块文件（但不要使用标准的Lua函数module（）来定义Lua模块，因为它也会污染全局命名空间），并调用require（）在init_by_Lua或其他上下文中加载自己的模块文件（require（）会将加载的Lua模块缓存在全局包中。

#### init_worker_by_lua_*

当master进程被启动后，每个worker进程都会执行Lua代码。如果Nginx禁用了master进程，init_by_lua*将会直接运行。

#### log_by_lua_*

log阶段是在返回给客户端之后才会执行的一个阶段，会话完成后本地异步完成。



### cosocket

cosocket 是 OpenResty 中的专有名词，是把协程和网络套接字的英文拼在一起形成的，即 cosocket = coroutine + socket；cosocket 不仅需要 Lua 协程特性的支持，也需要 Nginx 中非常重要的事件机制的支持，这两者结合在一起，最终实现了非阻塞网络 I/O。另外，cosocket 支持 TCP、UDP 和 Unix Domain Socket。

遇到网络 I/O 时，它会交出控制权（yield），把网络事件注册到 Nginx 监听列表中，并把权限交给 Nginx；当有 Nginx 事件达到触发条件时，便唤醒对应的协程继续处理（resume）;

而在 init_by_lua* 和 init_worker_by_lua* 中暂时不能用，可以用ngx.timer绕过；



###ngx.timer

#### ngx.timer.at

用来执行一次性的定时任务；

可以在回调函数的最后，再创建另外一个新的 timer，来实现周期性运行；

#### ngx.time.every

用来执行固定周期的定时任务；

定时任务是在后台运行的，并且无法取消，如果定时任务的数量很多，就很容易耗尽系统资源；OpenResty 提供了 lua_max_pending_timers （默认1024）和 lua_max_running_timers （默认256）这两个指令，来对其进行限制，前者代表等待执行的定时任务的最大值，后者代表当前正在运行的定时任务的最大值。



### 性能优化

#### 阻塞函数

##### 用io.open写日志，优化方案：

- 使用 lua-io-nginx-module 这个第三方的 C 模块，lua-io-nginx-module 利用了 Nginx 的线程池，把磁盘 I/O 操作从主线程转移到另外一个线程中处理，这样，主线程就不会因为磁盘 I/O 操作而被阻塞。使用这个库时，你需要重新编译 Nginx，因为它是一个 C 模块。

  ```lua
  local ngx_io = require "ngx.io"
  local path = "/conf/apisix.conf"
   local file, err = ngx_io.open(path, "rb")
   local data, err = file: read("*a") 
  file:close()
  ```

- 整个请求中的日志先缓存起来，在log_by_lua阶段写日志。