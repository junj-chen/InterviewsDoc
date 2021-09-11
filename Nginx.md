Nginx知识点

##### 1. Nginx的优点

Nginx是一个web服务器和方向代理服务器，用于HTTP、HTTPS、SMTP、POP3和IMAP协议。因它的稳定性、丰富的功能集、示例配置文件和低系统资源的消耗而闻名。  

Nginx本身就可以托管网站（类似于Tomcat一样），进行Http服务处理，也可以作为反向代理服务器 、负载均衡器和HTTP缓存  

Nginx的设计极具扩展性，它完全是由多个不同功能、不同层次、不同类型且耦合度极低的模块组成  



##### 2. Nginx是如何处理Http请求？

Nginx 是一个高性能的 Web 服务器，能够同时处理大量的并发请求。它结合多进程机制和异步机制 ，异步机制使用的是异步非阻塞方式 ，接下来就给大家介绍一下 Nginx 的多线程机制和异步非阻塞机制  

1. 多进程机制：

   Nginx服务器收到一个客户端的请求后，服务器主进程会生成一个子进程与客户端进行交互，直到断开，该子进程才结束

   使用进程的优点：

   	1. 进程之间相互独立，不需要进行加锁，减少了锁对性能的消耗，降低编程的复杂度。

   缺点：

   ​    生成一个子进程的时候会进行内存复制，在资源和内存上面有一定的损耗，当出现大量请求的时候，会导致系统性能下降

2. 异步非阻塞机制

每个工作进程 使用 异步非阻塞方式 ，可以处理 多个客户端请求 。当某个 工作进程 接收到客户端的请求以后，调用 IO 进行处理，如果不能立即得到结果，就去 处理其他请求 （即为 非阻塞 ）；

而 客户端 在此期间也 无需等待响应 ，可以去处理其他事情（即为 异步）。
当 IO 返回时，就会通知此 工作进程 ；该进程得到通知，暂时 挂起 当前处理的事务去 响应客户端请求  



##### 3. 正向代理与反向代理

代理服务器一般指局域网内部的机器通过代理服务器发送请求到互联网上的服务器，代理服务器一般作用在客户端。  我们的客户端在进行翻墙操作的时候，我们使用的正是正向代理，通过正向代理的方式，在我们的客户端运行一个软件，将我们的HTTP请求转发到其他不同的服务器端，实现请求的分发。  

反向代理服务器作用在服务器端，它在服务器端接收客户端的请求，然后将请求分发给具体的服务器进行处理，然后再将服务器的相应结果反馈给客户端。Nginx就是一个反向代理服务器软件。  



##### 4. Nginx应用场景？

1. http服务器。Nginx是一个http服务可以独立提供http服务。可以做网页静态服务器。
2. 虚拟主机。可以实现在一台服务器虚拟出多个网站，例如个人网站使用的虚拟机。
3. 反向代理，负载均衡。当网站的访问量达到一定程度后，单台服务器不能满足用户的请求时，需要用多台服务器集群可以使用nginx做反向代理。并且多台服务器可以平均分担负载，不会应为某台服务器负载高宕机而某台服务器闲置的情况。
4. Nginx 中也可以配置安全管理、比如可以使用Nginx搭建API接口网关,对每个接口服务进行拦截。



##### 5. Nginx配置文件nginx.conf有哪些属性模块?

~~~
worker_processes  1；                					# worker进程的数量
events {                              					# 事件区块开始
    worker_connections  1024；            				# 每个worker进程支持的最大连接数
}                                    					# 事件区块结束
http {                               					# HTTP区块开始
    include       mime.types；            				# Nginx支持的媒体类型库文件
    default_type  application/octet-stream；     		# 默认的媒体类型
    sendfile        on；       							# 开启高效传输模式
    keepalive_timeout  65；       						# 连接超时
      
    server {            								# 第一个Server区块开始，表示一个独立的虚拟主机站点
        listen       80；      							# 提供服务的端口，默认80
        server_name  localhost；       					# 提供服务的域名主机名
        location / {            						# 第一个location区块开始
            root   html；       						# 站点的根目录，相当于Nginx的安装目录
            index  index.html index.htm；      			# 默认的首页文件，多个用空格分开
        }          										# 第一个location区块结果
        error_page   500502503504  /50x.html；     		# 出现对应的http状态码时，使用50x.html回应客户
        location = /50x.html {          				# location区块开始，访问50x.html
            root   html；      							# 指定对应的站点目录为html
        }
    }  
    ......
}
~~~



##### 6. 虚拟主机配置

~~~
    #当客户端访问www.lijie.com,监听端口号为8080,直接跳转到data/www目录下文件
	 server {
        listen       8080;
        server_name  www.lijie.com;
        location / {
            root   data/www;
            index  index.html index.htm;
        }
    }
	
	#当客户端访问www.lijie.com,监听端口号为80直接跳转到真实ip服务器地址 127.0.0.1:8080
	server {
        listen       80;
        server_name  www.lijie.com;
        location / {
		 	proxy_pass http://127.0.0.1:8080;   // 
            index  index.html index.htm;
        }
	}

~~~



##### 7. location的作用是什么？

- location指令的作用是根据用户请求的URI来执行不同的应用，也就是根据用户请求的网站URL进行匹配，匹配成功即进行相关的操作。



##### 8. 限流怎么做？

- 流有3种
  1. 正常限制访问频率（正常流量）
  2. 突发限制访问频率（突发流量）
  3. 限制并发连接数
- Nginx的限流都是基于漏桶流算法，底下会说道什么是漏桶流

1. 正常限制访问频率（正常流量）

限制一个用户发送的请求，我Nginx多久接收一个请求。

Nginx中使用ngx_http_limit_req_module模块来限制的访问频率，限制的原理实质是基于漏桶算法原理来实现的。在nginx.conf配置文件中可以使用limit_req_zone命令及limit_req命令限制单个IP的请求处理频率。

~~~
	#定义限流维度，一个用户一分钟一个请求进来，多余的全部漏掉
	limit_req_zone $binary_remote_addr zone=one:10m rate=1r/m;

	#绑定限流维度
	server{
		
		location/seckill.html{
			limit_req zone=zone;	
			proxy_pass http://lj_seckill;
		}

	}
	
### 1r/s代表1秒一个请求，1r/m一分钟接收一个请求， 如果Nginx这时还有别人的请求没有处理完，Nginx就会拒绝处理该用户请求。
~~~



3. 限制并发连接数

- Nginx中的ngx_http_limit_conn_module模块提供了限制并发连接数的功能，可以使用limit_conn_zone指令以及limit_conn执行进行配置。接下来我们可以通过一个简单的例子来看下：

	http {
		limit_conn_zone $binary_remote_addr zone=myip:10m;
		limit_conn_zone $server_name zone=myServerName:10m;
	}
	
	server {
	    location / {
	        limit_conn myip 10;
	        limit_conn myServerName 100;
	        rewrite / http://www.lijie.net permanent;
	    }
	}
配置了单个IP同时并发连接数最多只能10个连接，并且设置了整个虚拟服务器同时最大并发数最多只能100个链接。当然，只有当请求的header被服务器处理后，虚拟服务器的连接数才会计数。刚才有提到过Nginx是基于漏桶算法原理实现的，实际上限流一般都是基于漏桶算法和令牌桶算法实现的。



##### 9. 限流算法

1. #### 漏桶算法

漏桶算法是网络世界中流量整形或速率限制时经常使用的一种算法，它的主要目的是控制数据注入到网络的速率，平滑网络上的突发流量。

漏桶算法提供了一种机制，通过它，突发流量可以被整形以便为网络提供一个稳定的流量。

漏桶算法提供的机制实际上就是刚才的案例：**突发流量会进入到一个漏桶，漏桶会按照我们定义的速率依次处理请求，如果水流过大也就是突发流量过大就会直接溢出，则多余的请求会被拒绝。所以漏桶算法能控制数据的传输速率。**

2. #### 令牌桶算法

令牌桶算法是网络流量整形和速率限制中最常使用的一种算法。典型情况下，令牌桶算法用来控制发送到网络上的数据的数目，并允许突发数据的发送。

令牌桶算法的机制如下：存在一个大小固定的令牌桶，会以恒定的速率源源不断产生令牌。如果令牌消耗速率小于生产令牌的速度，令牌就会一直产生直至装满整个令牌桶。

![image-20210911192242849](C:\Users\localuser\AppData\Roaming\Typora\typora-user-images\image-20210911192242849.png)



##### 10. Nginx中的负载均衡算法

Nginx负载均衡实现的策略有以下五种：

1. #### 轮询(默认)

   每个请求按时间顺序逐一分配到不同的后端服务器，如果后端某个服务器宕机，能自动剔除故障系统。

   ~~~
   upstream backserver { 
    server 192.168.0.12; 
    server 192.168.0.13; 
   } 
   ~~~

2. #### 权重 weight

- weight的值越大分配

- 到的访问概率越高，主要用于后端每台服务器性能不均衡的情况下。其次是为在主从的情况下设置不同的权值，达到合理有效的地利用主机资源。

  ~~~
  upstream backserver { 
   server 192.168.0.12 weight=2; 
   server 192.168.0.13 weight=8; 
  } 
  ### 权重越高，在被访问的概率越大，如上例，分别是20%，80%。
  ~~~

3. #### ip_hash( IP绑定)

- 每个请求按访问IP的哈希结果分配，使来自同一个IP的访客固定访问一台后端服务器，`并且可以有效解决动态网页存在的session共享问题`

  ~~~
  upstream backserver { 
   ip_hash; 
   server 192.168.0.12:88; 
   server 192.168.0.13:80; 
  } 
  ~~~

4. #### fair(第三方插件)

- 必须安装upstream_fair模块。
- 对比 weight、ip_hash更加智能的负载均衡算法，fair算法可以根据页面大小和加载时间长短智能地进行负载均衡，响应时间短的优先分配。

~~~
upstream backserver { 
 server server1; 
 server server2; 
 fair; 
} 
## 哪个服务器的响应速度快，就将请求分配到那个服务器上。
~~~

5. #### url_hash(第三方插件)

- 必须安装Nginx的hash软件包
- 按访问url的hash结果来分配请求，使每个url定向到同一个后端服务器，可以进一步提高后端缓存服务器的效率。

~~~
upstream backserver { 
 server squid1:3128; 
 server squid2:3128; 
 hash $request_uri; 
 hash_method crc32; 
} 
~~~



参考：https://blog.csdn.net/weixin_43122090/article/details/105461971

























































































































































