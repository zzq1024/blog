# 第一节：PHP奥义

### 不足
TCP断开连接几次握手
LRU
strace、awk
redis分布式
PHP-cgi在PHP代码更新后怎么处理

### PHP基础篇

#### 基础类型

| 系统类型      | 16位 | 32位 | 64位 |
| ------------- | ---- | ---- | ---- |
| char          | 1    | 1    | 1    |
| short int     | 2    | 2    | 2    |
| int           | 2    | 4    | 4    |
| unsigned int  | 2    | 4    | 4    |
| float         | 4    | 4    | 4    |
| double        | 8    | 8    | 8    |
| long          | 4    | 4    | 8    |
| long long     | 8    | 8    | 8    |
| unsigned long | 4    | 4    | 8    |

在程序设计语言里面，做自动类型转换的时候，为了避免数据在转换过程中由于截断导致数据错误，也都是**“按数据长度增加的方向”**进行转换的。

#### 浮点数的精度

浮点数的精度有限。尽管取决于系统，PHP 通常使用 IEEE 754 双精度格式，则由于取整而导致的最大相对误差为 1.11e-16。非基本数学运算可能会给出更大误差，并且要考虑到进行复合运算时的误差传递。
此外，以十进制能够精确表示的有理数如 0.1 或 0.7，无论有多少尾数都不能被内部所使用的二进制精确表示，因此不能在不丢失一点点精度的情况下转换为二进制的格式。这就会造成混乱的结果：例如，floor((0.1+0.7)*10) 通常会返回 7 而不是预期中的 8，因为该结果内部的表示其实是类似 7.9999999999999991118...。
所以永远不要相信浮点数结果精确到了最后一位，也永远不要比较两个浮点数是否相等。如果确实需要更高的精度，应该使用任意精度数学函数（比如bcadd）或者 gmp 函数，可以先赋值给变量（自动给个精确值）再运算。

#### 数组
PHP的数组是用链地址法的哈希结构去实现的，链表是双向链表，这样既可以动态分配数组空间，也可以通过key值去计算hash值去访问对应的元素，是一种非常高效的数据结构。元素在链表的位置是根据插入顺序决定的，并不是根据下标（下标只是数据结构体中的一个值）。

foreach遍历数组要比for遍历要快的原因，因为for每次查找元素都要去做一次哈希映射查找对应下标的Bucket，而foreach只需要遍历Bucket链表就好了。pListLast与pListNext同理，只是指向前一个数组元素。

#### echo与print区别

echo 是语法结构，也就是关键字，不是函数。使用的时候不用加括号，加上也可以。显示多个值的时候可以用逗号隔开。只支持基本类型，布尔型除外，echo true的时候显示1，echo false的时候啥都没有。

#### 变量的生命周期

1. 局部变量的生命周期为其所在函数被调用的整个过程。当局部变量所在的函数结束时，局部变量的生命周期也随之结束。

2. 全局变量的生命周期为其所在的".php"脚本文件被调用的整个过程。当全局变量所在的脚本文件结束调用时，则全局变量的生命周期结束。

3. 静态变量的作用范围与局部变量相同，但是生命周期与全局变量相同。

#### 纯php脚本不写结束标签原因

因为在不写php结束标签时，默认从开始标签往后都是php代码，如果有其他代码，那就会报错。php只能运行在php标签里面的脚本，在脚本之外的所有字符，包括你看不见的空格或者回车，制表符号，都是作为输出内容会response到客户端的，这样就有可能会产生意想不到的事情。例如文件里面使用了header函数，这个文件同时又包含了另外一个文件，并且被包含的文件的php标签外有空字符，这个时候会报header already send的错误。我们查看一些网页的源代码看到的开头部分有很多空格和换行，就是因为这个原因导致的。

#### $_SERVER常量

* $_SERVER['SERVER_ADDR']//服务器IP

* $_SERVER['SERVER_NAME']//服务器主机名

* $_SERVER['HTTP_ACCEPT_LANGUAGE']//浏览器语言 

* $_SERVER['REMOTE_ADDR'] //当前用户 IP 

* $_SERVER['REMOTE_HOST'] //当前用户主机名 

* $_SERVER['REQUEST_URI'] //URL

  在PHP中所有的header中的自定义信息都会被加上HTTP_的开头，在获取的时候参数名称无论大小写全部转换成大写！


#### 运算符优先级

- **递增递减**>!>**算术运算符**>**大小比较**>不相等比较>引用>位运算符(^)>位运算符(|)>**逻辑与**>**逻辑或**>**三目**>**赋值**>and>xor>or
- 递增递减不影响布尔值=》++true=true
- 递增NULL为1=》--NULL=NULL;++NULL=1

#### 流程语句使用

* if……elseif语句尽量把可能性大的条件放前面
* switch……case条件只能是整型、浮点型、字符串，生成一个索引表，效率比if快



### PHP函数篇

[字符串函数](http://php.net/manual/zh/book.strings.php)

[数组函数](http://php.net/manual/zh/book.array.php)

#### call_user_func与call_user_func_array

* call_user_func — 把第一个参数作为回调函数调用，其余参数是回调函数的参数
* call_user_func_array — 把第一个参数作为回调函数调用，并把一个**数组**参数作为回调函数的参数

#### 匿名函数

匿名函数（Anonymous functions），也叫闭包函数（closures），允许 临时创建一个没有指定名称的函数。闭包函数也可以作为变量的值来使用，闭包可以从父作用域中继承变量，任何此类变量都应该用 use 语言结构传递进去。

#### 自动加载

**__autoload()**函数只能存在一次，**spl_autoload_register()**当然能注册多个函数，SPL函数很丰富，提供了更多功能，如spl_autoload_unregister()注销已经注册的函数、spl_autoload_functions()返回所有已经注册的函数等。_autoload 方法在 spl_autoload_register 后会失效，因为 autoload_func 函数指针已指向 spl_autoload 方法可以通过下面的方法来把 _autoload 方法加入 autoload_functions list ：spl_autoload_register( '__autoload' )

#### array_merry与运算符+合并数组的区别

* array_merge()将两个或多个数组的单元合并起来，一个数组中的值附加在前一个数组的后面。返回作为结果的数组。如果输入的数组中有相同的字符串键名，则该键名后面的值将覆盖前一个值。然而，如果数组包含数字键名，后面的值将不会覆盖原来的值，而是附加到后面。
* 对于使用+，如果数组中有相同的key(数字键名和字符串键名处理一样)，则会把最先出现的值作为最终结果返回，而把后面的数组拥有相同键名的那些值“抛弃”掉（注意：不是覆盖而是保留最先出现的那个值）

#### filter_var与filter_var_array
* filter_var($variable, $filter) //
* filter_var_array(array $data, $definition) //filter为FILTER_VALIDATE_REGEXP时,参数必须是关联数组

#### 序列化(json，serialize，igbinary，msgpack)比较

* 占用空间方面，igbinary节省空间明显优势，比如在json一个数组5.4k大小的数据，serialize方式要8.6k，而使用igbinary方式，仅需2.4k，近乎为serialize方式的1/4，但在小数组方面msgpack方式更具优势，igbinary占用空间123，而msgpack方式仅为102。但是在大数组情况下，明显igbinary方式优势更明显。大数组igbinary胜出，小数组msgpack胜出。
* 在序列化方面，msgpack方式性能最好，其次是json_encode的，再次是igbinary，这两者相差无几，最差的为原生serialize。
* 在反序列方面igbinary的比序列化过程更快，当然也是最快的，但是这种快也是有成本代价的（调用igbinary_unserialize时，传递非法数据，会导致整个php进程死掉），参见最后的注意事项，最慢的为json_decode方式，猜测原因可能在于PHP作为服务器端应用，最多的场景是encode，而decode的最常见的为js处理方式，性能不是很理想。
* 整体性能是序列化和反序列化之和，简单对比会发现，json是最差的，次之是原生serialize，再次为igbinary的方式，最优的为msgpack，不过igbinary和msgpack相差真的非常小，而在占用空间方面，小数据时msgpack胜出，大数据时igbinary胜出，算是各有千秋。所以，如果追求极致的性能，可以考虑使用msgpack，如果对是使用空间要求苛刻，那就选择igbinary方式，估计这也是PHPRedis选择igbinary作为内置序列化方式的原因之一，另外还有一个原因，考虑到Redis应用场景多是一写多读，要保证反序列化性能足够高，非igbinary莫属。

#### file_get_contents与curl的区别
* fopen /file_get_contents 每次请求都会重新做DNS查询，并不对 DNS信息进行缓存。但是CURL会自动对DNS信息进行缓存。对同一域名下的网页或者图片的请求只需要一次DNS查询。这大大减少了DNS查询的次数。所以CURL的性能比fopen /file_get_contents 好很多。
* fopen /file_get_contents 在请求HTTP时，使用的是http_fopen_wrapper，不会keeplive。而curl却可以。这样在多次请求多个链接时，curl效率会好一些。
* fopen / file_get_contents 函数会受到php.ini文件中allow_url_open选项配置的影响。如果该配置关闭了，则该函数也就失效了。而curl不受该配置的影响。
* curl 可以模拟多种请求，例如：POST数据，表单提交等，用户可以按照自己的需求来定制请求。而fopen / file_get_contents只能使用get方式获取数据。

#### 在定义函数中…$args用于把参数组合形成一个数组，在调用函数中…$arr用于把数组展开为函数参数。正好是相反的过程

```php
function add(...$args) {
    var_dump($args);
}

$numArr = [1, 2, 3];
// 作为一个数组参数输出如下
// 输出 array(1) { [0]=> array(3) { [0]=> int(1) [1]=> int(2) [2]=> int(3) } }
echo add($numArr);

echo '<hr />';
// 在调用的时候...$arr，把数组展开为函数参数
// 输出array(3) { [0]=> int(1) [1]=> int(2) [2]=> int(3) }
echo add(...$numArr);

echo '<hr />';
// 如果能看懂这个你就明白了
// array(4) { [0]=> int(0) [1]=> int(1) [2]=> int(2) [3]=> int(3) }
echo add(0, ...$numArr);
```



### 特性篇

#### 内存回收机制

PHP使用了引用计数(reference counting)这种单纯的垃圾回收(garbage collection)机制，php中的变量存储在变量容器zval中，zval中除了存储变量类型和值外，还有is_ref和refcount字段，refcount表示指向变量的元素个数，is_ref表示变量是否有别名。 unset() 只是断开这个变量对它原先指向的内存的引用,使变量本身成为没有定义过空引用,所在调用时发出了 Notice ,并且使那块内存在符号表中引用计数 减 1,并没有影响到其他指向这块内存的变量，当一块内存在符号表中的引用计数为 0 时, PHP 引擎才会将这块内存回收， 赋值 null 操作是相当猛的,它会直接将变量所指向的内存在符号号中的引用计数置 0。如果一个zval的refcount减1之后**大于**0，它就会进入垃圾缓冲区。当缓冲区达到最大值后，回收算法会循环遍历zval，判断其是否为垃圾，并进行释放处理。unset()函数只能在变量值占用内存空间超过256字节时才会释放内存空间。

**深拷贝**：赋值时值完全复制，完全的copy，将原有的数据拷贝一份放到独立分配一个地址空间，对其中一个作出改变，不会影响另一个；写时赋值（COW）就是对深拷贝的一种优化吧，意思是只有当发生写操作的时候才进行深拷贝；

**浅拷贝**：赋值时，引用赋值，相当于取了一个别名。对其中一个修改，会影响另一个；

PHP中， = 赋值时，普通对象是深拷贝，**但对对象来说，是浅拷贝。也就是说，对象的赋值是引用赋值**。

##### PHP优化内存

1.分段读取

2.尽可能减少静态变量的使用，在需要数据重用时，可以考虑使用引用(&)

3.数据库操作完成后，要马上关闭连接

4.一个对象使用完，要及时调用析构函数（__destruct()）

5.用过的变量及时销毁(unset())掉

6.可以使用memory_get_usage()函数,获取当前占用内存 根据当前使用的内存来调整程序

7.unset()函数只能在变量值占用内存空间超过256字节时才会释放内存空间。(PHP内核的gc垃圾回收机制决定)

8.有当指向该变量的所有变量（如引用变量）都被销毁后，才会释放内存



### 面向对象篇

#### OOP小常识

* 父类中的私有方法和属性，子类是不能重写的；子类重写父类的方法不能降低访问权限，可能造成方法不可访问，会影响多态。

* 静态方法中只能调用静态变量（原因：静态成员属于类,不需要生成对象就存在了.而非静态需要生成对象才产生，所以静态成员不能直接访问.  ）

* PHP中对象传值都是引用传值

  例如：$obj_b = $obj_a;改了$obj_b的元素$obj_a的也会变；可以使用$obj_b = clone $obj_a;来复制$obj_a，生成$obj_a 的备份

  **__clone**使用：
  在克隆操作期间执行，除了将所有现有对象成员复制到目标对象之外，还会执行__clone()方法指定的操作。php的__clone()方法对一个对象实例进行的浅复制,对象内的基本数值类型进行的是传值复制，而对象内的对象型成员变量,如果不重写__clone方法,显式的clone这个对象成员变量的话,这个成员变量就是传引用复制,而不是生成一个新的对象。

  ```php
  <?php
  
    class Account {
  
      public $balance;
  
      public function construct($balance) {
        $this->balance = $balance;
      }
    }
  
    class Person {
  
      private $id;
  
      private $name;
  
      private $age;
  
      public $account;
        
      public function construct($name, $age, Account $account) {
        $this->name = $name;
        $this->age = $age;
        $this->account = $account;
      }
      public function setId($id) {
        $this->id = $id;
      }
      public function clone() {  #复制方法,可在里面定义再clone是进行的操作
        $this->id = 0;
        $this->account = clone $this->account;  #不加这一句,account在clone是会只被复制引用,其中一个account的balance被修改另一个也同样会被修改
      }
    }
    $person = new Person("peter", 15, new Account(1000));
    $person->setId(1);
    $person2 = clone $person;
    $person2->account->balance = 250;
    var_dump($person, $person2);
   ?>
  ```

* PHP中static关键字以及与self关键字的区别

  self调用的就是本身代码片段这个类，而static调用的是从堆内存中提取出来，访问的是当前实例化的那个类，那么 static 就是调用的当前调用的类

  ```php
  <?php   
    class Boo {  
        protected static $str = "This is class Boo";   
        public static function get_info(){            
            echo get_called_class()."<br>";  
            echo self::$str; //下面函数调用结果：This is class Boo
            //echo static::$str; //下面函数调用结果：This is class Foo
        }      
    }  
    class Foo extends Boo{  
        protected static $str = "This is class Foo";      
    }  
     Foo::get_info();   
  ?>
  ```

* 同一个类的对象即使不是同一个实例也可以互相访问对方的私有与受保护成员。这是由于在这些对象的内部具体实现的细节都是已知的

  ```php
  <?php
  class Test
  {
      private $foo;
      public function __construct($foo)
      {
          $this->foo = $foo;
      }
      public function baz(Test $other)
      {
          // We can change the private property:
          $other->foo = 'hello';
          var_dump($other->foo);
      }
  }
  
  $test = new Test('test');
  $test->baz(new Test('other'));//输出 string(5) "hello"
  ?>
  ```

#### 抽象类与接口的区别

1. 对接口的使用是通过关键字implements。对抽象类的使用是通过关键字extends。当然接口也可以通过关键字extends继承。一个类可以同时实现多个接口，但一个类只能继承于一个抽象类

2. 接口中不可以声明成员变量（包括类静态变量），但是可以声明类常量。抽象类中可以声明各种类型成员变量，实现数据的封装

3. 接口没有构造函数，抽象类可以有构造函数

4. 接口中的方法默认都是public类型的，而抽象类中的方法可以使用private,protected,public来修饰

#### 设计模式原则（重点工厂模式、策略模式、适配器模式、单例模式、观察者模式）

- 单一职责：一个类应该承担一个职责。

- 开放封闭：类、模块、函数等等，应该可以扩展，但是不可修改。

- 依赖倒转：底层细节应该依赖于抽象。（有统一的接口，容易复用）

- 迪米特法则：如果两个类不必直接通信，那这两个类不比直接发生交互。如果一个类需要调用另一个类时，可用第三个类转发这个调用。

- 里氏代换原则：任何基类可以出现的地方，子类一定可以出现，基类与子类的继承关系就是抽象化的具体实现。

- 合成复用原则：尽量使用合成/聚合的方式，而不是使用继承。

  策略模式和简单工厂模式看起来非常相似，都是通过多态来实现不同子类的选取。如果从使用这两种模式的角度来看的话，我们会发现在简单工厂模式中我们只需要传递相应的条件就能得到想要的一个对象，然后通过这个对象实现算法的操作。而策略模式，使用时必须首先创建一个想使用的类对象，然后将该对象作为参数传递进去，通过该对象调用不同的算法。在简单工厂模式中实现了通过条件选取一个类去实例化对象，策略模式则将选取相应对象的工作交给模式的使用者，它本身不去做选取工作

#### 单例模式特征

1. 需要一个保存类的唯一实例的静态成员变量（通常为$_instance私有变量）

2. 构造函数和克隆函数必须声明为私有的，这是为了防止外部程序new类从而失去单例模式的意义

3. 必须提供一个访问这个实例的公共的静态方法（通常为getInstance方法），从而返回唯一实例的一个引用

```
class SingleInstance
{   
	private static $instance;
	
    private function _construct(){}

    private function _clone(){}
    
    public static function getInstance()
    {

        if(!self::$instance instanceof self){
            self::$instance=new self();
        }
        return self ::$instance;
    }
}
```
#### 依赖注入
当一个类的实例需要另一个类的实例协助时，在传统的程序设计过程中，通常由调用者来创建被调用者的实例。而采用依赖注入的方式，创建被调用者的工作不再由调用者来完成，因此叫控制反转，创建被调用者的实例的工作由IOC容器来完成，然后注入调用者，因此也称为依赖注入。
使用依赖注入，最重要的一点好处就是有效的分离了对象和它所需要的外部资源，使得它们松散耦合，有利于功能复用，更重要的是使得程序的整个体系结构变得非常灵活。

### 框架篇

#### MVC模式

* Model（模型）是用于处理应用程序数据逻辑的部分
* View（视图）是应用程序中处理数据显示的部分
* Controller（控制器）是应用程序中处理用户交互的部分，接收视图的请求传到Model层处理并返回结果数据在视图显示

#### MVC、smarty、框架区别

* MVC是model、controller、view 的缩写，是一种非常实用的网站开发框架，通过合理的逻辑处理以及任务分配，使得网站开发变得非常快速
* smarty是一种模板引擎（是庞大的完善的正则表达式替换库），主要是用来处理网页模板视图（MVC中的V）的，可以替换网页中的元素或者实现基于模板的网页输出。而且非常容易操作，效率也是非常快
* ThinkPHP是之后基于MVC的框架，用来开发网站。是国人自己的框架，用起来还是很好的，注释非常详细。目前的3.2.2版本还是很稳定的

* 单一入口

  优点：进行统一性的安全检查；集中处理程序

  缺点：URL不美观（可以重写）；处理效率稍低

#### restful、rpc、soap

* restful：一种架构设计风格，提供了设计原则和约束条件，而不是架构。而满足这些约束条件和原则的应用程序或设计就是 RESTful架构或服务。
* rpc：是一种允许分布式应用程序调用网络上不同计算机的可用服务的机制，从一台机器（客户端）上通过参数传递的方式调用另一台机器（服务器）上的一个函数或方法（可以统称为服务）并得到返回的结果。数据直接在传输层基于TCP协议传输
* soap：SOAP是一种数据交换协议规范，是一种轻量的、简单的、基于XML的协议的规范，易用，灵活，跨语言，跨平台
#### slim
##### 中间件调用顺序
slim添加中间件时会一层一层的套，先添加的在最里面，后添加的在最外面，这样request请求进来时也是从外层到里层再到外层（先添加的后执行）。应用级中间件先执行，执行到$route->run()时，会继续执行路由级中间件
```php
// 第一种方式（应用级中间件）
$app->add(AMiddleware::class)->add(BMiddleware::class);
// 第二种方式 （路由中间件）
$app->get('/hello/{name}', function ($request, $response, $args) {
    return $response->getBody()->write("Hello, " . $args['name']);
})->add(CMiddleware::class)->add(DMiddleware::class);
//执行顺序：BMiddleware->AMiddleware->DMiddleware->CMiddleware->控制器
```



### 服务篇

#### 守护进程

在linux或者unix操作系统中，守护进程（Daemon）是一种运行在后台的特殊进程，它独立于控制终端并且周期性的执行某种任务或等待处理某些发生的事件（不会因任何终端的关闭所打断）。

#### 消息队列

原理：把消息按照产生的顺序加入队列，而由另外的处理程序/模块将其从队列中取出，并加以处理，从而形成基本的消息队列

应用场景：

- 传统的方式系统的性能（并发量，吞吐量，响应时间）会有瓶颈，引入消息队列，在不影响业务流程前提下，**异步处理**
- 用户下单后，订单系统需要通知库存系统，假如库存系统无法访问，则订单减库存将失败，从而导致订单失败，订单系统写入消息队列实现订单系统与库存系统的**应用解耦**
- 秒杀活动，一般会因为流量过大，导致流量暴增，应用挂掉。为解决这个问题，一般需要在应用前端加入消息队列，秒杀业务根据消息队列中的请求信息，再做后续处理，假如消息队列长度超过最大数量，则直接抛弃用户请求或跳转到错误页面，此为**流量削锋**
- **日志处理**是指将消息队列用在日志处理中，比如Kafka的应用，解决大量日志传输的问题；日志采集客户端，负责日志数据采集，定时写受写入Kafka队列，Kafka消息队列负责日志数据的接收、存储和转发，日志处理应用订阅并消费kafka队列中的日志数据
- **消息通讯**是指，消息队列一般都内置了高效的通信机制，因此也可以用在纯的消息通讯，比如实现点对点消息队列，或者聊天室等

#### 负载均衡

系统的扩展可分为纵向（垂直）扩展和横向（水平）扩展。纵向扩展，是从单机的角度通过增加硬件处理能力，比如CPU处理能力，内存容量，磁盘等方面，实现服务器处理能力的提升，不能满足大型分布式系统（网站），大流量，高并发，海量数据的问题。因此需要采用横向扩展的方式，通过添加机器来满足大型网站服务的处理能力。解决访问统一入口问题，我们可以在集群前面增加负载均衡设备，实现流量分发。

Nginx的负载均衡模块目前支持4种调度算法，下面分别进行介绍，其中后两项属于第三方调度算法：

1. 轮询（默认） 每个请求按时间顺序逐一分配到不同的后端服务器，如果后端的某个服务器宕机，故障系统被自动剔除，使用户的访问不受影响，weight 指定轮询的权值  weight值越大，分配的访问的机率越高 主要用于后端的每个服务器性能不均的情况下
2. ip_hash：每个请求按访问Ip的hash结果分配，这样来自同一个Ip的访客固定访问到一个后端服务器，有效的解决了动态网页存在的session共享问题
3. fair：这是个比上面两个更加智能的负载均衡算法，此算法可以依据页面的大小和加载时间长短智能的进行负载均衡，也就是根据后端服务器的响应时间来分配请求，响应时间短的优先分配。Nginx本身不支持fair的，如果需要使用这种调度算法，必须下载Nginx的upstream_fair模块
4. url_hash：按访问的url的hash结果来分配请求，使每个url定向到同一个后端服务器，可以进一步提高后端缓存服务器的效率，Nginx本身是不支持url_hash的，如果需要使用这种调度算法，必须安装Nginx的hash软件包



### 常见问题篇

#### PHP执行流程

没有Opcache缓存的执行过程：test.php->经过zend编译->opcode->PHP解释器->机器码

启用Opcache缓存的执行过程：test.php->查找opcache缓存，如果没有则进行zend编译为opcode并缓存->opcode->PHP解释器->机器码

JIT流程：test.php->编译->机器码

JIT （Just-in-time），是一种编译器策略，JIT将为Zend VM生成的指令（视为中间码），生成依赖于体系结构的机器代码，所以代码执行不再需要Zend VM解释器，直接在CPU中执行。



#### nginx与PHP工作原理

nginx作为前端服务器，解析HTTP请求，通过配置文件找到server服务器，匹配合适的location，在location中的命令会启动不同的模块（nginx中模块是编译后自动加载的，比apache速度上有提升）完成工作；通过location指令，将所有以php为后缀的文件都交给 127.0.0.1:9000 来处理，而这里的IP地址和端口（127.0.0.1:9000）就是fastcgi（fastcgi是一个可伸缩地、高速地在http server和动态脚本语言间通信的接口）进程监听的IP地址和端口，主进程php-fpm主要是管理fastcgi子进程，监听9000端口，fastcgi子进程等待请求；php-fpm选择并连接到一个fastcgi子进程，并将环境变量和标准输入发送到fastcgi子进程；fastcgi子进程完成处理后将标准输出和错误信息返回，当fastcgi子进程关闭连接时，请求便告处理完成，等待下次处理。

**CGI**：公共网关接口，HTTP Server和一个PHP独立的进程之间的协议，以CGI方式运行时,web server将用户请求以消息的方式转交给PHP独立进程,PHP与web服务之间无从属关系。CGI有很多缺点，每接收一个请求就要fork一个进程处理，只能接收一个请求作出一个响应，请求结束后该进程就会释放。

**FastCGI**：是对CGI的改进，Fastcgi会先启一个master，解析配置文件，初始化执行环境，然后再启动多个worker，常驻内存的。当请求过来时，master会传递给一个worker，然后立即可以接受下一个请求。这样就避免了重复的劳动，效率自然是高。而且当worker不够用时，master可以根据配置预先启动几个worker等着；当然空闲worker太多时，也会停掉一些，这样就提高了性能，也节约了资源。它随着Nginx服务一同启动并驻留内存，当请求到达时，处理之，处理完之后并不结束这个进程，而是继续等待下一次连接。

**PHP-fpm**：是PHP一个实现了FastCGI的管理器，常驻内存，负责管理各PHP-cgi进程，可以提升运行效率，并实现平滑重启(修改php.ini后，php-fpm对此的处理机制是新的worker用新的配置，已经存在的worker处理完手上的活就可以歇着了)。

#### Nginx和PHP-FPM的进程间通信

在Linux上，nginx与php-fpm的通信有tcp socket和unix socket两种方式；

unix socket方式：
php-fpm.conf: listen = /tmp/php-fpm.sock;
nginx.conf: fastcgi_pass unix:/tmp/php-fpm.sock;
Unix socket 又叫 IPC(inter-process communication 进程间通信) socket，用于实现同一主机上的进程间通信;
Unix socket 不需要经过网络协议栈，不需要打包拆包、计算校验和、维护序号和应答等，只是将应用层数据从一个进程拷贝到另一个进程。所以其效率比 tcp socket 的方式要高，可减少不必要的 tcp 开销。不过，unix socket 高并发时不稳定，连接数爆发时，会产生大量的长时缓存，在没有面向连接协议的支撑下，大数据包可能会直接出错不返回异常。

tcp socket方式：
php-fpm.conf: listen = 127.0.0.1:9000
nginx.conf: fastcgi_pass 127.0.0.1:9000
tcp socket 的优点是可以跨服务器，当 nginx 和 php-fpm 不在同一台机器上时，只能使用这种方式；
tcp 这样的面向连接的协议，可以更好的保证通信的正确性和完整性，面临高并发业务，则考虑选择使用更可靠的 tcp socket，以负载均衡、内核优化等运维手段维持效率。


#### 进程与线程

**进程**：操作系统会以进程为单位，分配系统资源（CPU时间片、内存等资源），进程是资源分配的最小单位。

进程间通信：管道(Pipe)、命名管道(FIFO)、消息队列(Message Queue) 、信号量(Semaphore) 、共享内存（Shared Memory）；套接字（Socket）

**线程**：有时被称为轻量级进程(Lightweight Process，LWP），是操作系统调度（CPU调度）执行的最小单位。

一个线程只能属于一个进程，而一个进程可以有多个线程，但至少有一个线程；

进程是拥有资源的一个独立单位，线程不拥有系统资源，但可以访问隶属于进程的资源；

不同进程之间可以并发执行，同一个进程的多个线程之间也可并发执行；

进程有独立的地址空间，一个进程崩溃后，在保护模式下不会对其它进程产生影响，而线程只是一个进程中的不同执行路径，线程有自己的堆栈和局部变量，但线程之间没有单独的地址空间，一个进程死掉就等于所有的线程死掉，所以多进程的程序要比多线程的程序健壮，但在进程切换时，耗费资源较大，效率要差一些；

**协程**：协程，是一种比线程更加轻量级的存在，协程不是被操作系统内核所管理，而完全是由程序所控制，性能得到了很大的提升，不会像线程切换那样消耗资源；在协程中控制共享资源不加锁，只需要判断状态就好了，所以执行效率比多线程高很多。

**goroutine**:Goroutine是建立在线程之上的轻量级的抽象,它允许我们以非常低的代价在同一个地址空间中并行地执行多个函数或者方法。相比于线程，它的创建和销毁的代价要小很多，并且它的调度是独立于线程的。在golang中创建一个goroutine非常简单，使用“go”关键字即可，go的主函数也是跑在goroutine中。

Goroutine所需要的内存通常只有2kb，而线程则需要1Mb（500倍）



#### post与get区别：（HTTP协议中两种发送请求的方法）

* 最直观的是get将参数包含在url中，post通过request body传递参数。
* get传输数据长度2KB,POST没有长度限制。
* get产生一个TCP数据包，post产生两个TCP数据包。

#### session与cookie

* 存在的位置
  * cookie 存在于客户端，临时文件夹中
  * session：存在于服务器的内存中，一个session域对象为一个用户浏览器服务
* 安全性
  * cookie是以明文的方式存放在客户端的，安全性低，可以通过一个加密算法进行加密后存放
  * session存放于服务器的内存中，所以安全性好
* 网络传输量
  * cookie会传递消息给服务器
  * session本身存放于服务器，不会有传送流量
* 生命周期(以20分钟为例)
  * 如果不设置过期时间，则表示这个cookie生命周期为浏览器会话期间，只要关闭浏览器窗口，cookie就消失了，这种生命期为浏览会话期的cookie被称为会话cookie，会话cookie一般保存在内存里。如果设置了过期时间，浏览器就会把cookie保存到硬盘(临时文件)上，关闭后再次打开浏览器，这些cookie依然有效直到超过设定的过期时间。
  * session的生命周期是间隔的，从创建时，开始计时如在20分钟，没有访问session，那么session生命周期被销毁;但是，如果在20分钟内（如在第19分钟时）访问过session，那么，将重新计算session的生命周期
  * 关机会造成session生命周期的结束，但是对cookie没有影响
* 访问范围
  * session为一个用户浏览器独享
  * cookie为多个用户浏览器共享

#### 多服务器共享session的方法

* 基于NFS的Session共享:将共享目录服务器mount到各频道服务器的本地session目录即可，缺点是NFS依托于复杂的安全机制和文件系统，因此并发效率不高，尤其对于session这类高并发读写的小文件，会由于共享目录服务器的io-wait过高，最终拖累前端WEB应用程序的执行效率

* 基于数据库的Session共享:首选当然是大名鼎鼎的Mysql数据库，并且建议使用内存表Heap，提高session操作的读写效率。这个方案的实用性比较强，它的缺点在于session的并发读写能力取决于Mysql数据库的性能，同时需要自己实现session淘汰逻辑，以便定时从数据表中更新、删除 session记录，当并发过高时容易出现表锁，虽然我们可以选择行级锁的表引擎，但不得不否认使用数据库存储Session还是有些杀鸡用牛刀的架势

* 基于Cookie的Session共享:原理是将全站用户的Session信息加密、序列化后以Cookie的方式，统一种植在根域名下（如：.host.com），利用浏览器访问该根域名下的所有二级域名站点时，会传递与之域名对应的所有Cookie内容的特性，从而实现用户的Cookie化Session 在多服务间的共享访问（cookie长度4K，对存储数据的长度有限制）

* 基于Memcache的Session共享:Memcache的内存hash表所特有的Expires数据过期淘汰机制，正好和Session的过期机制不谋而合，降低了过期Session数据删除的代码复杂度，对比“基于数据库的存储方案”，仅这块逻辑就给数据表产生巨大的查询压力

  （实验：两个服务器设置相同的session_id，session_save_path设置在redis中）。

#### PHP实现单点登录

以http://a.com作为主域名，不论http://b.com还是http://c.com登录时请求都提交到http://a.com，所有的存储都在http://a.com，其他域名不允许访问。在http://a.com登录成功后，将登录信息存放在http://a.com域下的cookie中。此时发起广播，由client端通知http://b.com和http://c.com，说用户已经登录（仅仅是告诉他们已经登录），此时用户访问http://b.com或者http://c.com时有server端向http://a.com请求获取用户信息，并将非敏感信息存储到自己的存储当中。同理，登出时也是向http://a.com发起登出请求，http://a.com将用户cookie清掉，http://b.com和http://c.com再次向http://a.com请求验证用户身份时，则验证失败

#### 数据加密

* .数据表中敏感数据加密一般可采用aes+base64加密，先进行aes对称加密（明文会被分割成16字节逐个进行加密，最后一块若不足16字节, 则会被补齐到16字节，基本保证相同字段下数据长度一样），加密(用二进制中的或、与、异类、或等运算进行加密)后的密文是二进制不能正常显示，需要进行base64编码。
* 平常的密码加密：https://www.laravist.com/blog/post/php-password-hash-in-the-right-way

#### redis与memcache的区别

* 存储方式：memecache 把数据全部存在内存之中，断电后会挂掉，数据不能超过内存大小；redis有部份存在硬盘上，这样能保证数据的持久性，支持数据的持久化
* 数据支持类型：memcache只支持简单地k/v类型，redis在数据支持上要比memecache多的多
* 分布式存储：redis支持master-slave复制模式，memcache可以使用一致性hash做分布式
* Redis只使用单核，而Memcached可以使用多核，所以平均每一个核上Redis在存储小数据时比 
  Memcached性能更高。而在100k以上的数据中，Memcached性能要高于Redis
* redis目前官方只支持LINUX 上去行，从而省去了对于其它系统的支持，这样的话可以更好的把精力用于本系统 环境上的优化。

#### 网站高并发、大访问量问题解决

* 防盗链。以访问来源的HOST过滤；签名
* 前端优化，分为减少HTTP请求；异步请求；启用浏览器缓存和文件压缩
* CDN加速
* 图片服务器分离，大文件的下载会占用很大的流量，建议将大文件放在另外一台服务器上
* 前台实现完全的静态化最好，可以完全不用访问数据库
* 消息队列
* 数据库数据缓存到redis
* 数据库合理设计索引，分区，分库分表，读写分离，负载均衡
* Nginx负载均衡：根据某种负载策略把请求分发到集群中的每一台服务器上，让整个服务器群来处理网站的请求

#### HTTP状态码

100 （继续） 请求者应当继续提出请求。服务器返回此代码表示已收到请求的第一部分，正在等待其余部分。
200 （成功） 服务器已成功处理了请求。

304  表示此请求的本地缓存是最新的，可以直接使用

400 （错误请求） 服务器不理解请求的语法。
401 （未授权） 请求要求身份验证。 对于需要登录的网页，服务器可能返回此响应。
403 （禁止） 服务器拒绝请求(文件没有读权限)。
404 （未找到） 服务器找不到请求的网页。
499  (客户端断开连接) 接口处理时间过长，前端请求自带有超时时间，客户端主动关闭了连接
500 （服务器内部错误） 服务器遇到错误，无法完成请求。
502 （错误网关） 服务器作为网关或代理，从上游服务器收到无效响应（一般都是php-cgi进程数不够用、php执行时间长、或者是php-cgi进程死掉）。
503 （服务不可用） 服务器目前无法使用（由于nginx超载或停机维护）。通常，这只是暂时状态。
504 （网关超时） 服务器作为网关或代理，但是没有及时从上游服务器收到请求（可能DNS查询超时）。

#### TCP、UDP区别

* TCP是**有连接的**，两台主机在进行数据交互之前**必须先通过三次握手**建立连接；而UDP是无连接的，没有建立连接这个过程
* TCP是**可靠的传输**，TCP协议通过确认和重传机制来保证数据传输的可靠性；而UDP是不可靠的传输
* TCP还**提供了拥塞控制、滑动窗口等机制**来保证传输的质量，而UDP都没有
* TCP是基于**字节流**的，将数据看做无结构的字节流进行传输，当应用程序交给TCP的数据长度太长，超过MSS时，TCP就会对数据进行分段，因此TCP的数据是无边界的；而UDP是**面向报文**的，无论应用程序交给UDP层多长的报文，UDP都不会对数据报进行任何拆分等处理，因此UDP保留了应用层数据的边界 
##### TCP创建连接时三次握手：
- 第一次握手，A 向 B 发送信息，B 收到信息后可以确认自己的收信能力和 A 的发信能力没有问题。
- 第二次握手，B 向 A 发送信息，A 可以确认自己的发信能力和B的收信能力没有问题，但是 B 不知道自己的发信能力到底如何，所以就需要第三次通信。
- 第三次握手，A 向 B 发送信息，B 就可以确认自己的发信能力没有问题。
##### TCP断开时的四次握手：
- 第一次握手，A 向 B 发送消息，表明数据发送完成需要断开连接
- 第二次握手，B 向 A 发送消息，同意A的关闭
- 第三次握手，B 向 A 发送消息，数据已传完，可以断开了
- 第四次握手，A 向 B 发送消息，稍后会断开连接（等待2MSL后变成 CLOSED 状态）
##### TIME_WAIT
TCP 四次握手结束后，连接双方都不再交换消息，但主动关闭的一方保持这个连接在一段时间内不可用;
用处：
- 四次挥手中，A 发 FIN， B 响应 ACK，B 再发 FIN，A 响应 ACK 实现连接的关闭。而如果 A 响应的 ACK 包丢失，B 会以为 A 没有收到自己的关闭请求，然后会重试向 A 再发 FIN 包；如果没有 TIME_WAIT 状态，A 不再保存这个连接的信息，收到一个不存在的连接的包，A 会响应 RST 包，导致 B 端异常响应；此时， TIME_WAIT 是为了保证全双工的 TCP 连接正常终止。
- TCP 下的 IP 层协议是无法保证包传输的先后顺序的。如果双方挥手之后，一个网络四元组（src/dst ip/port）被回收，而此时网络中还有一个迟到的数据包没有被 B 接收，A 应用程序又立刻使用了同样的四元组再创建了一个新的连接后，这个迟到的数据包才到达 B，那么这个数据包就会让 B 以为是 A 刚发过来的；此时， TIME_WAIT 的存在是为了保证网络中迷失的数据包正常过期。
时长的确定：
TIME_WAIT 状态由主动关闭的 A 来保持，那么我们来考虑对于 A 来说，可能接到上一个连接的数据包的最大时长：A 刚发出的数据包，能保持 MSL（最大分段寿命，TCP 分段可以存在于互联网系统中的最大时间） 时长的寿命，它到了 B 端后，B 端由于关闭连接了，会响应 RST 包，这个 RST 包最长也会在 MSL 时长后到达 A，那么 A 端只要保持 TIME_WAIT 到达 2MS 就能保证网络中这个连接的包都会消失。

作者：JinyaX
链接：https://www.jianshu.com/p/275608bf7669
来源：简书
简书著作权归作者所有，任何形式的转载都请联系作者获得授权并注明出处。

作者：JinyaX
链接：https://www.jianshu.com/p/275608bf7669
来源：简书
简书著作权归作者所有，任何形式的转载都请联系作者获得授权并注明出处。

#### websocket
**Socket**是对TCP/IP协议的抽象，封装了应用层和传输控制层之间的一组接口，比如create、listen、connect、accept、send、read和write等等。
WebSocket protocol 是HTML5一种新的协议。它实现了浏览器与服务器全双工通信(full-duplex)。一开始的握手需要借助HTTP请求完成。WebSocket同HTTP一样也是应用层的协议，但是它是一种双向通信协议，是建立在TCP之上的。

目的：即时通讯，替代轮询。



### 外部扩展篇

#### PHP连接MySQL数据库的三种方式(mysql、mysqli、pdo)

* PHP的**MySQL**扩展是设计开发允许php应用与MySQL数据库交互的早期扩展，MySQL扩展提供了一个面向过程的接口，并不支持后期MySQL服务端提供的一些特性。由于太古老，又不安全，所以已被后来的mysqli完全取代。

* PHP的**mysqli**扩展，我们有时称之为MySQL增强扩展，可以用于使用 MySQL4.1.3或更新版本中新的高级特性。其特点为：面向对象接口 、prepared语句支持、多语句执行支持、事务支持 、增强的调试能力、嵌入式服务支持 、预处理方式完全解决了sql注入的问题。不过其也有缺点，就是只支持mysql数据库。如果你要是不操作其他的数据库，这无疑是最好的选择。

* **PDO**是PHP Data Objects的缩写，是PHP应用中的一个数据库抽象层规范。PDO提供了一个统一的API接口可以使得你的PHP应用不去关心具体要连接的数据库服务器系统类型，也就是说，如果你使用PDO的API，可以在任何需要的时候无缝切换数据库服务器，比如从Oracle 到MySQL，仅仅需要修改很少的PHP代码。其功能类似于JDBC、ODBC、DBI之类接口。同样，其也解决了sql注入问题，有很好的安全性。不过他也有缺点，某些多语句执行查询不支持（不过该情况很少）。

  总结：优先推荐msqli，其次是PDO 。而“民间”给出的结果很多是倾向于使用PDO，因为其不担有跨库的优点，更有读写速度快的特点。

#### URI与URL

URI，统一资源标识符，首先它是一个字符串。其次，它是一个可以唯一标识某一资源的字符串。
URL，统一资源定位符，首先，它是一种URI，其次，它可以标识资源的路径。

#### 版本控制

##### SVN

* 将文件checkout到本地目录：

  svn checkout path（path是服务器上的目录） //简写：svn co

* 将改动的文件提交到版本库：

  svn commit -m "LogMessage" [-N][--no-unlock] PATH(如果选择了保持锁，就使用--no-unlock开关) 

* 更新到某个版本:

  svn update -r m path  //简写：svn up

* 比较版本间差异：

  svn diff path(将修改的文件与基础版本比较) 

  svn diff -r m:n path(对版本m和版本n比较差异) 

* 合并

  svn merge -r m:n path

##### GIT

Workspace：工作区
Index / Stage：暂存区
Repository：仓库区（或本地仓库）
Remote：远程仓库

* 新建
  - 在当前目录新建一个Git代码库：git init
  - 新建一个目录，将其初始化为Git代码库：git init [project-name]
  - 下载一个项目和它的整个代码历史：git clone [url]
* 提交
  * 提交到暂存区：git add file/dir
  * 提交到本地仓库：git commit [file] -m [message]
* 分支
  * 列出所有本地分支：git branch
  * 列出所有远程分支：git branch -r
  * 列出所有远程分支和本地分支：git branch -a
  * 新建一个分支，但依然停留在当前分支：git branch [branchname]
  * 新建一个分支，并切换到该分支：git checkout -b [branch]
  * 切换到指定分支，并更新工作区：git checkout [branch]
  * 合并指定分支到当前分支：git merge [branch]
  * 显示有变更的文件:git status
  * 显示暂存区和工作区的差异：git diff
  * 下载远程仓库的所有变动:git fetch
  * 取回远程仓库的变化，并与本地分支合并:git pull remote branch
  * 上传本地指定分支到远程仓库:git push remote branch
  * 强行推送当前分支到远程仓库，即使有冲突:git push [remote] --force
  * 推送所有分支到远程仓库：git push [remote] --all

#### Xdebug配置

settings -> Languages -> php
settings -> Languages -> php -> debug
settings -> Languages -> php -> debug -> DBGp Proxy
settings -> Languages -> php -> servers
run -> edit configurations -> php web page
