# 安装

```shell
# 递归创建目录
mkdir -p /openmall/software/nginx

# 上传安装包
scp -r nginx-1.16.1.tar.gz openmall@192.168.1.20:/openmall/software/nginx

# 安装gcc环境
sudo yum install -y gcc-c++
# 安装PCRE库，用于解析正则表达式
sudo yum install -y pcre pcre-devel
# zlib压缩和解压缩依赖
sudo yum install -y zlib zlib-devel
# SSL 安全的加密的套接字协议层，用于HTTP安全传输，也就是https
sudo yum install -y openssl openssl-devel

# 解压源码
tar -zxvf nginx-1.16.1.tar.gz
# 编译前创建临时目录
mkdir /var/temp/nginx -p

# 在nginx目录，输入如下命令进行配置，目的是为了创建makefile文件
cd /openmall/software/nginx/nginx-1.16.1

./configure --prefix=/openmall/install/nginx --pid-path=/var/run/nginx/nginx.pid --lock-path=/var/lock/nginx.lock --error-log-path=/var/log/nginx/error.log --http-log-path=/var/log/nginx/access.log --with-http_ssl_module --with-http_stub_status_module --with-http_gzip_static_module --http-client-body-temp-path=/var/temp/nginx/client --http-proxy-temp-path=/var/temp/nginx/proxy --http-fastcgi-temp-path=/var/temp/nginx/fastcgi --http-uwsgi-temp-path=/var/temp/nginx/uwsgi --http-scgi-temp-path=/var/temp/nginx/scgi
# --prefix 指定nginx安装目录
# --pid-path 指向nginx的pid
# --lock-path 锁定安装文件，防止被恶意篡改或误操作
# --error-log-path 错误日志
# --http-log-path http日志
# --with-http_gzip_static_module 启用gzip模块，在线实时压缩输出数据流
# --http-client-body-temp-path 设定客户端请求的临时目录
# --http-proxy-temp-path 设定http代理临时目录
# --http-fastcgi-temp-path 设定fastcgi临时目录
# --http-uwsgi-temp-path 设定uwsgi临时目录
# --http-scgi-temp-path 设定scgi临时目录

# 修改拥有者
sudo chown openmall /var/log/nginx
sudo chown openmall /var/temp/nginx

# make编译
make

# 安装
sudo make install
```



# 命令

- nginx 启动
- nginx -s stop 停止
- nginx -s quit 等待用户完成请求，之后再停止
- nginx -s reload 修改配置后重新加载
- nginx -t 测试配置文件
- nginx -? 帮助
- nginx -V 查看安装时的配置信息



# 配置文件核心参数解析

nginx/conf/nginx.conf

- `error_log  logs/error.log;` logs目录需要提前创建

- `pid logs/nginx.pid;` 需要 `nginx -c /openmall/install/nginx/conf/nginx.conf` 重新指定配置文件，重新生成新的pid

- `log_format`

  - $remote_addr 客户端ip
  - $remote_user 远程客户端用户名，一般为：’-’
  - $time_local 时间和时区
  - $request 请求的url以及method
  - $status 响应状态码
  - $body_bytes_send 响应客户端内容字节数
  - $http_referer 记录用户从哪个链接跳转过来的
  - $http_user_agent 用户所使用的代理，一般都是浏览器
  - $http_x_forwarded_for 通过代理服务器来记录客户端的ip

- server 中 `listen ip:port` 和 `server_name` ，先匹配ip，再匹配server_name主机名，最后匹配port。

  - 客户端输入 `www.test.com:90` 时，解析对应域名的IP发送请求，请求会携带IP和域名发送至对应的Nginx服务器，Nginx匹配到如下server，然后在Nginx服务器上查找hosts文件中域名配置的IP，这时Nginx就会把请求转发给此IP相应的端口

    `listen       90;`

    `server_name www.test.com;`

- server 中 `location` 匹配规则 `location [=|~|~*|^~] /uri/ { … }`

  - `=` 开头表示精确匹配
  - `^~` （前缀匹配，即正则匹配取反）开头表示 uri 以某个常规字符串开头，理解为匹配 url 路径即可。nginx 不对 url 做编码，因此请求为/static/20%/aa，可以被规则^~ /static/ /aa匹配到（注意是空格）
  - `~` 开头表示区分大小写的正则匹配
  - `~*` 开头表示不区分大小写的正则匹配
  - `/` 通用匹配，任何请求都会匹配到
  
- 负载均衡upstream配置

  - 默认轮询算法
  - 加权轮询算法 `server 192.168.1.190:8080 weight=5;` ，weight值越大权重越大
  - IP哈希算法 `ip_hash`
    - 默认哈希IP地址的前三位，所以同一个网段的IP可能会请求到一台服务器上
    - 只要IP不变就会请求到一台server上（会话保持）
    - 注意热点问题
    - 当某一台server不可用时，要标记为down，避免hash算法改变影响其他请求导致session缓存失效（解决：一致性hash算法保证绝大多数节点的session可用，请求环上的就近server，只影响down机的session或新增节点之后原节点之前的server，其他server不影响）
  - URL哈希算法 `url_hash`，根据每次请求的url地址，哈希后访问到固定的server
  - 最少连接算法 `least_conn`，请求落在最少连接数的server上

```shell
# 设置worker进程的用户，指的linux中的用户，会涉及到nginx操作目录或文件的一些权限，默认为 nobody
user  root;
# worker进程工作数设置，一般来说CPU有几个，就设置几个，或者设置为N-1
worker_processes  2;

# nginx 日志级别 debug | info | notice | warn | error | crit | alert | emerg
# 可以覆盖编译时的配置
# server操作日志，默认 error_log logs/error.log error;
error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

# 设置nginx进程 pid
pid        logs/nginx.pid;

# 设置工作模式
events {
    # 默认使用epoll
    use epoll;
    # 每个worker允许连接的客户端最大连接数
    worker_connections  1024;
}

# http指令块
http {
    # 引入外部配置，提高可读性
    include       mime.types;
    default_type  application/octet-stream;

    # 设置日志格式
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    # http访问日志，默认 access_log logs/access.log combined;
    access_log  logs/access.log  main;

    # 使用高效文件传输，提升传输性能
    sendfile        on;
    # 当数据累积到一定大小后才发送，提高效率
    #tcp_nopush     on;

    # 设置客户端与服务端请求的超时时间，保证客户端多次请求的时候不会重复建立新的连接，节约资源损耗
    #keepalive_timeout  0;
    keepalive_timeout  65;

    # 启动压缩
    #gzip  on;
    
    # 匹配的server
    server {
        # 监听端口
        listen       80;
        # 匹配的server name，域名
        server_name  localhost;
        
        # 跨域配置支持
        # 允许跨域请求的域，*代表所有
        add_header 'Access-Control-Allow-Origin' *;
        # 允许带上cookie请求
        add_header 'Access-Control-Allow-Credentials' 'true';
        # 允许请求的方法，比如 GET/POST/PUT/DELETE
        add_header 'Access-Control-Allow-Methods' *;
        # 允许请求的header 
        add_header 'Access-Control-Allow-Headers' *;
        
        # 防盗链配置支持
        valid_referers *.sciatta.com; 
        # 非法引入会进入下方判断
        if ($invalid_referer) { 
        	return 404; 
        }

        # charset koi8-r;

        # access_log  logs/host.access.log  main;

        # 路由映射
        location / {
            root   html; # 请求位置
            index  index.html index.htm; # 首页设置
        }

        #error_page  404              /404.html;

        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }
    
    # 负载均衡配置，默认轮询算法
    upstream tomcats { 
				server 192.168.1.190:8080;
				server 192.168.1.191:8080; 
		}
		server {
				listen 80;
				server_name www.tomcats.com;
				location / {
						proxy_pass http://tomcats;
				}
		}
}
```

