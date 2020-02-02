# 第一节：Nginx

### 代理

#### 正向代理

正向代理是一个位于客户端和目标服务器之间的代理服务器(中间服务器)，为了从原始服务器取得内容，客户端向代理服务器发送一个请求，并且指定目标服务器，之后代理向目标服务器转交并且将获得的内容返回给客户端；需要你主动设置代理服务器ip或者域名进行访问，由设置的服务器ip或者域名去目标服务器获取访问内容并返回。

用途：翻墙

#### 反向代理

客户端向反向代理发送请求，接着反向代理判断请求走向何处，并将请求转交给客户端；对于客户端来说，反向代理就好像目标服务器，不需要你做任何设置，直接访问服务器真实ip或者域名，但是服务器内部会自动根据访问内容进行跳转及内容返回，你不知道它最终访问的是哪些机器。

用途：

- 保护和隐藏原始资源服务器
- 加密和SSL加速
- 负载均衡
- 缓存静态内容
- 安全



### 终端命令

#### 使用&后台运行程序

- 结果会输出到终端
- 使用Ctrl + C发送SIGINTI信号，程序免疫
- 关闭session（关闭终端）发送SIGHUP信号，程序关闭

#### 使用nohup运行程序

- 结果默认会输出到nohup.out
- 使用Ctrl + C发送SIGINT信号，程序关闭
- 关闭session发送SIGHUP信号，程序免疫
  平时可以使用nohup与&启动程序，同时免疫SIGINT与SIGHUP信号，开启守护进程



### Nginx实现高并发

nginx每进来一个request，会有一个worker进程去处理。但不是全程的处理，处理到什么程度呢？处理到可能发生阻塞的地方，比如向上游（后端）服务器转发request，并等待请求返回。那么，这个处理的worker不会这么傻等着，他会在发送完请求后，注册一个事件：“如果upstream返回了，告诉我一声，我再接着干”。于是他就休息去了。此时，如果再有request 进来，他就可以很快再按这种方式处理。而一旦上游服务器返回了，就会触发这个事件，worker才会来接手，这个request才会接着往下走。



### nginx配置
#### nginx.conf基础
- rewrite重定向url（可以美化URL）
- location优先级：(location =) > (location 完整路径) > (location ^~ 路径) > (location ~,~* 正则顺序，如果有两个相同正则，会直接按第一个正则为准，不会被下边的覆盖) > (location 部分起始路径) > (/)
- 如果直接访问ip:port（对应多个HOST），会访问第一个匹配的（引入文件顺序，文件内server配置顺序）；
  如果直接访问host（对应多个port）,会访问第一个匹配的（引入文件顺序，文件内server配置顺序）
#### nginx.conf配置结构
- main全局块：配置影响nginx全局的指令。一般有运行nginx服务器的用户组，nginx进程pid存放路径，日志存放路径，配置文件引入，允许生成worker process数等。
- events块：配置影响nginx服务器或与用户的网络连接。有每个进程的最大连接数，选取哪种事件驱动模型处理连接请求，是否允许同时接受多个网路连接，开启多个网络连接序列化等。
- http块：可以嵌套多个server，配置代理，缓存，日志定义等绝大多数功能和第三方模块的配置。如文件引入，mime-type定义，日志自定义，是否使用sendfile传输文件，连接超时时间，单连接请求数等。
- server块：配置虚拟主机的相关参数，一个http中可以有多个server。
- location块：配置请求的路由，以及各种页面的处理情况。
#### nginx.conf 基本配置模板
```
#配置用户或者组，默认为nobody nobody。
#user administrator administrators; 
#允许生成的进程数，默认为1
#worker_processes 2; 
#指定nginx进程运行文件存放地址
#pid /nginx/pid/nginx.pid; 
#制定错误日志路径，级别。这个设置可以放入全局块，http块，server块，级别依次为：debug|info|notice|warn|error|crit|alert|emerg
error_log log/error.log debug; 

#工作模式及连接数上限
events {
#设置网路连接序列化，防止惊群现象发生，默认为on
   accept_mutex on; 
#设置一个进程是否同时接受多个网络连接，默认为off
   multi_accept on; 
#事件驱动模型，select|poll|kqueue|epoll|resig|/dev/poll|eventport
#use epoll; 
#单个work进程允许的最大连接数，默认为512
   worker_connections 1024; 
}

#http服务器
http {
#文件扩展名与文件类型映射表。设定mime类型(邮件支持类型),类型由mime.types文件定义
#include /usr/local/etc/nginx/conf/mime.types;
   include mime.types; 
#默认文件类型，默认为text/plain
   default_type application/octet-stream; 

#取消服务访问日志
#access_log off;     
#自定义日志格式
   log_format myFormat '$remote_addr–$remote_user [$time_local] $request $status $body_bytes_sent $http_referer $http_user_agent $http_x_forwarded_for'; 
#设置访问日志路径和格式。"log/"该路径为nginx日志的相对路径，mac下是/usr/local/var/log/。combined为日志格式的默认值
   access_log log/access.log myFormat; 
   rewrite_log on;

#允许sendfile方式传输文件，默认为off，可以在http块，server块，location块。（sendfile系统调用不需要将数据拷贝或者映射到应用程序地址空间中去）
   sendfile on; 
#每个进程每次调用传输数量不能大于设定的值，默认为0，即不设上限。
   sendfile_max_chunk 100k; 

#连接超时时间，默认为75s，可以在http，server，location块。
   keepalive_timeout 65; 

#gzip压缩开关
#gzip on;

   tcp_nodelay on;

#设定实际的服务器列表
   upstream mysvr1 {   
     server 127.0.0.1:7878 max_fails=1 fail_timeout=10s; #在单位周期为fail_timeout设置的时间，中达到max_fails次数，那么接将节点标记为不可用，并等待下一个周期（同样时常为fail_timeout）再一次去请求
     server 192.168.10.121:3333 backup; #热备(其它所有的非backup机器down或者忙的时候，请求backup机器))
   }
   upstream mysvr2 {
#weigth参数表示权值，权值越高被分配到的几率越大
     server 192.168.1.11:80 weight=5;
     server 192.168.1.12:80 weight=1 down; #表示该服务器已经停用
     server 192.168.1.13:80 weight=6;
   }
   upstream https-svr {
#每个请求按访问ip的hash结果分配，这样每个访客固定访问一个后端服务器，可以解决session的问题
     ip_hash;
     server 192.168.1.11:90;
     server 192.168.1.12:90;
   }

#error_page 404 https://www.baidu.com; #错误页

#HTTP服务器

# 静态资源一般放在nginx所在主机
   server {
       listen 80; #监听HTTP端口
       server_name 127.0.0.1; #监听地址  
       keepalive_requests 120; #单连接请求上限次数
       set $doc_root_dir "/Users/doing/IdeaProjects/edu-front-2.0"; #设置server里全局变量
       #index index.html;  #定义首页索引文件的名称
       location ~*^.+$ { #请求的url过滤，正则匹配，~为区分大小写，~*为不区分大小写。
          root $doc_root_dir; #静态资源根目录
          proxy_pass http://mysvr1; #请求转向“mysvr1”定义的服务器列表
          #deny 127.0.0.1; #拒绝的ip
          #allow 172.18.5.54; #允许的ip           
       } 
   }

#http
   server {
       listen 80;
       server_name www.helloworld.com; #监听基于域名的虚拟主机。可有多个，可以使用正则表达式和通配符
       charset utf-8; #编码格式
       set $static_root_dir "/Users/doing/static";
       location /app1 { #反向代理的路径（和upstream绑定），location后面设置映射的路径 
           proxy_pass http://zp_server1;
       } 
       location /app2 {  
           proxy_pass http://zp_server2;
       } 
       location ~ ^/(images|javascript|js|css|flash|media|static)/ {  #静态文件，nginx自己处理
           root $static_root_dir;
           expires 30d; #静态资源过时间30天
       }
       location ~ /\.ht {  #禁止访问 .htxxx 文件
           deny all;
       }
       location = /do_not_delete.html { #直接简单粗暴的返回状态码及内容文本
           return 200 "hello.";
       }

# 指定某些路径使用https访问(使用正则表达式匹配路径+重写uri路径)
       location ~* /http* { #路径匹配规则：如localhost/http、localhost/httpsss等等
#rewrite只能对域名后边的除去传递的参数外的字符串起作用，例如www.c.com/proxy/html/api/msg?method=1&para=2只能对/proxy/html/api/msg重写。
#rewrite 规则 定向路径 重写类型;
#rewrite后面的参数是一个简单的正则。$1代表正则中的第一个()。
#$host是nginx内置全局变量，代表请求的主机名
#重写规则permanent表示返回301永久重定向
           rewrite ^/(.*)$ https://$host/$1 permanent;
       }
#重写类型：last：本条规则匹配完成后，继续向下匹配新的location；break：本条规则匹配完成即终止，不再匹配后#面的任何规则；redirect：返回302临时重定向，浏览器地址会显示跳转后的URL地址；permanent：返回301永久重
#定向，浏览器地址栏会显示跳转后的URL地址；
#错误处理页面（可选择性配置）
#error_page 404 /404.html;
#error_page 500 502 503 504 /50x.html;

#以下是一些反向代理的配置(可选择性配置)
#proxy_redirect off;
#proxy_set_header Host $host; #proxy_set_header用于设置发送到后端服务器的request的请求头
#proxy_set_header X-Real-IP $remote_addr;
#proxy_set_header X-Forwarded-For $remote_addr; #后端的Web服务器可以通过X-Forwarded-For获取用户真实IP
#proxy_connect_timeout 90; #nginx跟后端服务器连接超时时间(代理连接超时)
#proxy_send_timeout 90; #后端服务器数据回传时间(代理发送超时)
#proxy_read_timeout 90; #连接成功后，后端服务器响应时间(代理接收超时)
#proxy_buffer_size 4k; #设置代理服务器（nginx）保存用户头信息的缓冲区大小
#proxy_buffers 4 32k; #proxy_buffers缓冲区，网页平均在32k以下的话，这样设置
#proxy_busy_buffers_size 64k; #高负荷下缓冲大小（proxy_buffers*2）
#proxy_temp_file_write_size 64k; #设定缓存文件夹大小，大于这个值，将从upstream服务器传

#client_max_body_size 10m; #允许客户端请求的最大单文件字节数
#client_body_buffer_size 128k; #缓冲区代理缓冲用户端请求的最大字节数


   }

#https
#(1)HTTPS的固定端口号是443，不同于HTTP的80端口；
#(2)SSL标准需要引入安全证书，所以在 nginx.conf 中你需要指定证书和它对应的 key
   server {
     listen 443;
     server_name  www.hellohttps1.com www.hellohttps2.com;
     set $geek_web_root "/Users/doing/IdeaProjects/backend-geek-web";
     ssl_certificate      /usr/local/etc/nginx/ssl-key/ssl.crt; #ssl证书文件位置(常见证书文件格式为：crt/pem)
     ssl_certificate_key  /usr/local/etc/nginx/ssl-key/ssl.key; #ssl证书key位置
     location /passport {
       send_timeout 90;
       proxy_connect_timeout 50;
       proxy_send_timeout 90;
       proxy_read_timeout 90;
       proxy_pass http://https-svr;
     }
     location ~ ^/(res|lib)/ {
        root $geek_web_root; 
        expires 7d;
#add_header用于为后端服务器返回的response添加请求头，这里通过add_header实现CROS跨域请求服务器
        add_header Access-Control-Allow-Origin *; 
     }
#ssl配置参数（选择性配置）
     ssl_session_cache shared:SSL:1m;
     ssl_session_timeout 5m;
   }

#配置访问控制：每个IP一秒钟只处理一个请求，超出的请求会被delayed
#语法：limit_req_zone  $session_variable  zone=name:size  rate=rate (为session会话状态分配一个大小为size的内存存储区，限制了每秒（分、小时）只接受rate个IP的频率)
   limit_req_zone  $binary_remote_addr zone=req_one:10m   rate=1r/s nodelay;
   location /pay {
        proxy_set_header Host $http_host;
        proxy_set_header X-Real_IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
#访问控制：limit_req zone=name [burst=number] [nodelay];
        limit_req zone=req_one burst=5; #burst=5表示超出的请求(被delayed)如果超过5个，那些请求会被终止（默认返回503）
        proxy_pass http://mysvr1;
   }

#可以把子配置文件放到/usr/local/etc/nginx/servers/路径下，通过include引入
   include /usr/local/etc/nginx/servers/*.conf;

}
```
#### 内置全局变量

\$args ：这个变量等于请求行中的参数，同\$query_string
\$content_length ： 请求头中的Content-length字段。
\$content_type ： 请求头中的Content-Type字段。
\$document_root ： 当前请求在root指令中指定的值。
\$host ： 请求主机头字段，否则为服务器名称。
\$http_user_agent ： 客户端agent信息
\$http_cookie ： 客户端cookie信息
\$limit_rate ： 这个变量可以限制连接速率。
\$request_method ： 客户端请求的动作，通常为GET或POST。
\$remote_addr ： 客户端的IP地址。
\$remote_port ： 客户端的端口。
\$remote_user ： 已经经过Auth Basic Module验证的用户名。
\$request_filename ： 当前请求的文件路径，由root或alias指令与URI请求生成。
\$scheme ： HTTP方法（如http，https）。
\$server_protocol ： 请求使用的协议，通常是HTTP/1.0或HTTP/1.1。
\$server_addr ： 服务器地址，在完成一次系统调用后可以确定这个值。
\$server_name ： 服务器名称。
\$server_port ： 请求到达服务器的端口号。
\$request_uri ： 包含请求参数的原始URI，不包含主机名，如：”/foo/bar.php?arg=baz”。
\$uri ： 不带请求参数的当前URI，\$uri不包含主机名，如”/foo/bar.html”。
\$document_uri ： 与$uri相同。

#### 参数解析

##### root与alias

```nginx
# 如果一个请求的URI是/t/a.html时，web服务器将会返回服务器上的/www/root/html/t/a.html的文件。
location ^~ /t/ {
	root /www/root/html/;
}

# 如果一个请求的URI是/t/a.html时，web服务器将会返回服务器上的/www/root/html/new_t/a.html的文件。
# 注意这里是new_t，因为alias会把location后面配置的路径丢弃掉，把当前匹配到的目录指向到指定的目录。
location ^~ /t/ {
	alias /www/root/html/new_t/;
}
```





### 日志分割

#### 1.sh定时脚本

```sh
#设置log日志的存储地址
LOG_PATH=/home/soft/nginx/logs
#设置历史日志的存储地址
HISTORY_LOG_PATH=/home/soft/nginx/history_logs
#获取分割日志时所需要的时间当做日志文件名称
TIME=$(date +%Y-%m-%d)
#将当前日志备份到指定存储目录
mv ${LOG_PATH}/access.log ${HISTORY_LOG_PATH}/access_log/${TIME}_access.log
#发送信号重新打开日志文件
kill -USR1 $(cat ${LOG_PATH}/nginx.pid)
```

将sh脚本加入到crontab定时任务中，每天23:59执行：59 23 * * * /home/sh/backups_log.sh。

注意：在没有执行kill -USR1 nginx_pid 之前，即便已经对文件执行了mv命令也只是改变了文件的名称，nginx还是会向新命名的文件中照常写入日志数据。原因在于linux系统中，内核是根据文件描述符来找文件的；

#### 2.logrotate切割nginx日志

logrotate使用cron按时调度执行切割日志；

/etc/logrotate.conf：通用配置文件，可以定义全局默认使用的选项。
/etc/logrotate.d/xxx：一般需要切割的日子放到这里，编辑文件需要root权限

```nginx
/www/log_receiver/nginx/*.log {
    daily
    dateext
    compress
    rotate 7
    sharedscripts
    postrotate
        kill -USR1 `cat /usr/local/nginx_1.10.1/logs/nginx.pid`
    endscript
}
```

参数说明：

daily：每天切割一次文件

dateext：切割的旧文件会加上日期时间

compress：切割的旧文件会被压缩

rotate 7：最多保留多少次周期，这里7是指7天

sharedscripts：sharedscripts用于指明以下是执行轮转前和轮转后自定义执行的命令，比如postrotate和endscript表示，轮转后，执行nginx的重新加载配置文件，避免日志轮转后不写日志。如果要轮转前执行某个命令可以使用prerotate代替postrotate即可,两者可同时存在。



### Nginx进程结构

#### kill信号量

- TERM/INT：立刻快速的杀掉进程

  kill -TERM 'cat logs/nginx.pid' = nginx -s stop

- QUIT：优雅的关闭进程，即等请求结束后关闭进程

  kill -QUIT 'cat logs/nginx.pid' = nginx -s quit

- HUP：改变配置文件，平滑的重读配置文件

  kill -HUP 'cat logs/nginx.pid' = nginx -s reload

- USR1：重读日志，在日志按月/日分割时有用：对于做日志备份有用  首先对access.log做备份access.log.bak           然后新建access.log，最后 kill -USR1 PID，会把新生成的文件存到access.log里面

  kill -USR1 'cat  logs/nginx.pid` = nginx -s reopen

- USR2：平滑的升级；

- WINCH：优雅的关闭旧的worker进程(配合上USR2来进行升级)

#### 信号管理Nginx父子进程

master进程接收信号：TERM/INT，QUIT，HUP，USR1，USR2，WINCH

worker进程接收信号：TERM/INT，QUIT，USR1，WINCH

一般只给master进程发信号，master进程会将信号传递给worker进程。

#### 平滑重启（nginx -s reload）

1.向master进程发送HUP信号（reload命令）

2.master进程校验配置语法是否正确

3.master进程打开新的监听端口（子进程可以共享使用父进程已经打开的端口；新老worker进程因为是都是同一个master进程的子进程，所以可以同时监听80端口）

4.master进程用新配置启动新的worker子进程

5.master进程向老的worker子进程发送QUIT信号

6.老worker进程关闭监听句柄，处理完当前连接后结束进程

#### 平滑升级（二进制热升级）

一般有两种情况下需要升级Nginx，一种是确实要升级Nginx的版本，另一种是要为Nginx添加新的模块。

```shell
# 升级命令
# ./configure --prefix=/usr/local/nginx
# make
# 此时，源码目录 build/nginx-1.13.6/objs/下会有nginx二进制文件（及动态模块.so文件），之后用它覆盖掉旧的nginx二进制文件(如果更新动态模块，可以不用替换nginx文件)
# 热升级时不要执行 make install(只有首次安装时才需要执行)，否则会把旧的Nginx安装目录下文件覆盖掉
```

1.将旧Nginx文件换成新Nginx文件（/usr/local/openresty/nginx/sbin/nginx，注意备份）

2.向老master进程发送USR2信号

3.老master进程修改pid文件名，加后缀.oldbin（给新的master进程让路）

4.master进程用新Nginx文件启动新master进程（此时新老master、worker进程共存）

5.向老master进程PID发送QUIT信号，关闭老master进程

6.**回滚：**向老master发送HUP，向新master发送QUIT

**注意：**一个父进程退出，而它的一个或多个子进程还在运行，那么那些子进程将成为孤儿进程。孤儿进程将被init进程(进程号为1)所收养，并由init进程对它们完成状态收集工作。

##### 为什么不能在第2步执行nginx reload来热升级？

reload命令会根据配置文件把worker进程更换，但更换的只是配置文件内容，而不是二进制程序（/usr/local/openresty/nginx/sbin/nginx），且master进程也未更换（观察进程号），所以无法实现热升级（更换新版本nginx程序的内容）。

#### 优雅的关闭worker进程

1.设置定时器（在nginx.conf中配置worker_shutdown_timeout，默认不打开）

2.关闭监听句柄（保证worker不会处理新的连接）

3.关闭空闲连接（连接池中未断开的可复用连接）

4.在循环中等待全部连接关闭

5.退出进程（连接全部关闭 或者 时间达到worker_shutdown_timeout 进程立即退出）



### SSL协议

后面版本又称为TLS；此协议在ISO/OSI模型中的表示层工作；

CSR （证书签名申请）完成后，CA机构（负责颁发证书）会用自己的私钥加密网站的身份信息，生成一对公私钥，公钥会在CA证书中保存（又叫公钥证书），证书订阅人（站点的发布人，开发者）拿到公私钥之后，就会部署到web服务器，比如nginx。当浏览器第一步访问 http 站点时，会从服务器请求证书， nginx 就会把公钥证书发给浏览器，浏览器拿到网站的证书后，自己通过CA的公钥来解密公钥证书，得到网站的身份信息，来确定网站是否安全。

#### TLS加密过程

验证身份 =》 达成安全套件共识 =》 传递秘钥 =》 加密通讯；

HTTPS采用混合加密算法，即共享秘钥加密（对称加密）和公开秘钥加密（非对称加密）。
通信前准备工作：
A、数字证书认证机构的公开秘钥（CA公钥）已事先植入到浏览器里；
B、数字证书认证机构用自己的私有密钥对服务器的公开秘钥做数字签名，生成公钥证书，并颁发给服务器。

1、client hello
握手第一步是客户端向服务端发送 Client Hello 消息，这个消息里包含了一个客户端生成的随机数 Random1、客户端支持的加密套件（Support Ciphers）和 SSL Version 等信息。

2、server hello
服务端向客户端发送 Server Hello 消息，这个消息会从 Client Hello 传过来的 Support Ciphers 里确定一份加密套件，这个套件决定了后续加密和生成摘要时具体使用哪些算法，另外还会生成一份随机数 Random2。注意，至此客户端和服务端都拥有了两个随机数（Random1+ Random2），这两个随机数会在后续生成对称秘钥时用到。

3、Certificate
这一步是服务端将自己的公钥证书下发给客户端。

4、Server Hello Done
Server Hello Done 通知客户端 Server Hello 过程结束。

5、Certificate Verify
客户端收到服务端传来的公钥证书后，先从 CA 验证该证书的合法性（CA公钥去解密公钥证书），验证通过后取出证书中的服务端公钥，再生成一个随机数 Random3，再用服务端公钥非对称加密 Random3生成 PreMaster Key。

6、Client Key Exchange
上面客户端根据服务器传来的公钥生成了 PreMaster Key，Client Key Exchange 就是将这个 key 传给服务端，服务端再用自己的私钥解出这个 PreMaster Key 得到客户端生成的 Random3。至此，客户端和服务端都拥有 Random1 + Random2 + Random3，两边再根据同样的算法就可以生成一份秘钥，握手结束后的应用层数据都是使用这个秘钥进行对称加密。为什么要使用三个随机数呢？这是因为 SSL/TLS 握手过程的数据都是明文传输的，并且多个随机数种子来生成秘钥不容易被破解出来。

