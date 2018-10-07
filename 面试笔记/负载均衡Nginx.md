---
title: Nginx 负载均衡
---

# 概念

+ 模块划分
 + Core 核心模块
 + Events事件模块
 + HTTP 模块
 + Mail邮件模块
+ <a href = "http://seanlook.com/2015/05/17/nginx-location-rewrite/"> Nginx 中的正则表达式语法</a>
 + 第一优先级：  已=开头表示精确匹配
 + 第二优先级：^~ 开头表示uri以某个常规字符串开头，不是正则匹配
 + 第三优先级：~ 开头表示区分大小写的正则匹配;~* 开头表示不区分大小写的正则匹配 
 + 第四优先级：/ 通用匹配, 如果没有其它匹配,任何请求都会匹配到
 
<pre>
location = / {
	# 仅仅匹配请求 /  **第一优先级，因为好含有 "="**
	[ configuration A ]
}
location / {
	# 匹配所有以 / 开头的请求。但是如果有更长的同类型的表达式，则选择更长的表达式。如果有正则表达式可以匹配，则	
	# 优先匹配正则表达式。 **常规匹配**
	[ configuration B ]
}
location /documents/ {
	# 匹配所有以 /documents/ 开头的请求。但是如果有更长的同类型的表达式，则选择更长的表达式。
	#如果有正则表达式可以匹配，则优先匹配正则表达式。
	[ configuration C ]
}
location ^~ /images/ { 
	# 匹配所有以 /images/ 开头的表达式，如果匹配成功，则停止匹配查找。所以，即便有符合的正则表达式location，也
	# 不会被使用 **第二优先级 ，**^~ 所有 以image 开头的url 利用这个**
	[ configuration D ]
}
location ~* \.(gif|jpg|jpeg)$ {
	# 匹配所有以 gif jpg jpeg结尾的请求。但是 以 /images/开头的请求，将使用 Configuration D
	[ configuration E ]
}
</pre>

# 配置



## 负载均衡配置



+ **注意当你的负责均衡服务器也要做为服务器时，负载监听端口一定不能和服务接口一致**
+ **Nginx 负载均衡有几种配置方案**
 1. **轮询(Round Robin)：** 根据Nginx配置文件中的顺序，`依次`将客户端的请求分发到后端服务器上
 2. **最少连接(Least_Conn)：** 将客户端的web请求分发到分发到连接最少的服务器上
 3. **IP地址Hash(Ip_hash) ： 如果处理的连接比较复杂，通过IP计算的hash值将统一客户端的Web请求分发到统一服务器上去处理**
 	<pre>
 	upstream backend {
  	 	server 127.0.0.1:8080 ;
		server 127.0.0.1:9090 ;
   		ip_hash;
	}
</pre>
 4. **基于权重的负载均衡（Weight）** ，下面是一个基于权重的负载均衡配置
   
	<pre>
	upstream Server_Name { # 这个Server_Name 就是你的配置的Nginx 服务器的Servern_Name，  
  
	#ip_hash;   
 	# down 表示这个指定的Server 暂时不参与负载均衡
	server 127.0.0.1:8083 down;    
  	# Weight 表示当前的负载权重
	server 127.0.0.1:8084 weight=3;   
  
	server 127.0.0.1:8001;   
  	@
	server 127.0.0.1:8002 backup;
	}
</pre>

## 代理设置

### 反向代理设置

反向代理，其实是将代理服务器作为客户端可见的服务器作为 客户端的相应，所以你在访问的时候其实是在访问这个代理服务器！
在下面的这个例子中其实是访问 的 `http:localhost:85`
<pre>
   # 定义上游服务器,并提供负载均衡
	upstream  web{   
		server localhost:8084;  
		server localhost:8083;  
		server localhost:8082;  
	}  

    server {
		# Http 监听的端口
        listen       85;
		# Host name ，Server name 即为 服务器的 地址，确保能被发现
        server_name  localhost;  
		# 
        charset utf-8;
		# 访问日志存储目录
        access_log  logs/host.access.log;
			
        location / {
            # 当前这个Server 反向代理的 上游服务器 看上面的 UpStream
            proxy_pass  http://web;
            root   html;
            index  index.html index.htm;
            #  Nginx默认是不转发client的 Host 名字的
            #  这个利用这个变量 Nginx会转发 <a href = "http://www.pureage.info/2014/02/22/host-variable-#in-nginx.html">Client Host</a> 
            proxy_set_header Host $host;  
            # 转发时隐藏中 指定的Client 请求中的Header 信息   
            proxy_hide_header Cache-Control;
			# proxy_next_upstream 表示 当当前的上游服务器发生故障时，去找下一个上游服务器  
			#proxy_next_upstream
			# proxy_pass_request_body 是否向上游服务器转发Request的请求体部分
			# proxy_pass_request_body on|off
        }
		
</pre>
	
		

+ proxy_redirect
 一个典型的配置 `proxy_redirect http://localhost:8000/two/ http://frontend/one/;`如果上游 服务器返回302状态重定向到   http://localhost:8000/two/ 通过这个 配置 其实是定向到 http://frontend/one/some/uri/。




### 正向代理设置
正向代理，其实是将代理服务器作为客户端的一个DNS 解析器而已,一个正先发代理的例子

<pre>
server {
 listen 8090;
 location / {
	
 	resolver 8.8.8.8; //设置一个DNS 服务器作为Client 端请求服务器的解析服务器
 	resolver_timeout 30s;
 	proxy_pass http://$host$request_uri;
 }
 access_log  /data/httplogs/proxy-$host-aceess.log;      
}
</pre>

# 怎样对Nginx 运行状态进行监控

# Nginx 的安全过滤

首选 认识Nginx 中的这些内置百变量

<pre><code class="hljs lang-gams"><span class="hljs-meta"><span class="hljs-meta-keyword">$args</span> ：这个变量等于请求行中的参数，同$query_string</span>
<span class="hljs-meta"><span class="hljs-meta-keyword">$content</span>_length ： 请求头中的Content-length字段。</span>
<span class="hljs-meta"><span class="hljs-meta-keyword">$content</span>_type ： 请求头中的Content-Type字段。</span>
<span class="hljs-meta"><span class="hljs-meta-keyword">$document</span>_root ： 当前请求在root指令中指定的值。</span>
<span class="hljs-meta"><span class="hljs-meta-keyword">$host</span> ： 请求主机头字段，否则为服务器名称。</span>
<span class="hljs-meta"><span class="hljs-meta-keyword">$http</span>_user_agent ： 客户端agent信息</span>
<span class="hljs-meta"><span class="hljs-meta-keyword">$http</span>_cookie ： 客户端cookie信息</span>
<span class="hljs-meta"><span class="hljs-meta-keyword">$limit</span>_rate ： 这个变量可以限制连接速率。</span>
<span class="hljs-meta"><span class="hljs-meta-keyword">$request</span>_method ： 客户端请求的动作，通常为GET或POST。</span>
<span class="hljs-meta"><span class="hljs-meta-keyword">$remote</span>_addr ： 客户端的IP地址。</span>
<span class="hljs-meta"><span class="hljs-meta-keyword">$remote</span>_port ： 客户端的端口。</span>
<span class="hljs-meta"><span class="hljs-meta-keyword">$remote</span>_user ： 已经经过Auth Basic Module验证的用户名。</span>
<span class="hljs-meta"><span class="hljs-meta-keyword">$request</span>_filename ： 当前请求的文件路径，由root或alias指令与URI请求生成。</span>
<span class="hljs-meta"><span class="hljs-meta-keyword">$scheme</span> ： HTTP方法（如http，https）。</span>
<span class="hljs-meta"><span class="hljs-meta-keyword">$server</span>_protocol ： 请求使用的协议，通常是HTTP/1.0或HTTP/1.1。</span>
<span class="hljs-meta"><span class="hljs-meta-keyword">$server</span>_addr ： 服务器地址，在完成一次系统调用后可以确定这个值。</span>
<span class="hljs-meta"><span class="hljs-meta-keyword">$server</span>_name ： 服务器名称。</span>
<span class="hljs-meta"><span class="hljs-meta-keyword">$server</span>_port ： 请求到达服务器的端口号。</span>
<span class="hljs-meta"><span class="hljs-meta-keyword">$request</span>_uri ： 包含请求参数的原始URI，不包含主机名，如：”/foo/bar.php?arg=baz”。</span>
<span class="hljs-meta"><span class="hljs-meta-keyword">$uri</span> ： 不带请求参数的当前URI，$uri不包含主机名，如”/foo/bar.html”。</span>
<span class="hljs-meta"><span class="hljs-meta-keyword">$document</span>_uri ： 与$uri相同。</span>
</code></pre>


一些内置的条件判断：

+ -f和!-f用来判断是否存在文件
+ -d和!-d用来判断是否存在目录
+ -e和!-e用来判断是否存在文件或目录
+ -x和!-x用来判断文件是否可执行

例子
<pre># 如果文件不存在则返回400
if (!-f $request_filename) {
    return 400;
}

# 如果host不是xuexb.com，则301到xuexb.com中
if ( $host != "xuexb.com" ){
    rewrite ^/(.*)$ https://xuexb.com/$1 permanent;
}

# 如果请求类型不是POST则返回405
if ($request_method = POST) {
    return 405;
}

# 如果参数中有 a=1 则301到指定域名
if ($args ~ a=1) {
    rewrite ^ http://example.com/ permanent;
}
在某种场景下可结合location规则来使用，如：

# 访问 /test.html 时
location = /test.html {
    # 默认值为xiaowu
    set $name xiaowu;

    # 如果参数中有 name=xx 则使用该值
    if ($args ~* name=(\w+?)(&|$)) {
        set $name $1;
    }

    # 301
    rewrite ^ /$name.html permanent;
}
上面表示：

/test.html => /xiaowu.html
/test.html?name=ok => /ok.html?name=ok</pre>

# 页面缓存
