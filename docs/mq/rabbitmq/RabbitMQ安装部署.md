# 单机

## 环境准备

```shell
sudo yum install -y build-essential openssl openssl-devel unixODBC unixODBC-devel make gcc gcc-c++ kernel-devel m4 ncurses-devel tk tc xz 

sudo yum install -y tcp_wrappers
```



## 安装

```shell
wget www.rabbitmq.com/releases/erlang/erlang-18.3-1.el7.centos.x86_64.rpm
wget http://repo.iotti.biz/CentOS/7/x86_64/socat-1.7.3.2-1.1.el7.lux.x86_64.rpm
wget www.rabbitmq.com/releases/rabbitmq-server/v3.6.5/rabbitmq-server-3.6.5-1.noarch.rpm

# 拷贝
scp erlang-18.3-1.el7.centos.x86_64.rpm openmall@node100:/openmall/software/rabbitmq
scp socat-1.7.3.2-1.1.el7.x86_64.rpm openmall@node100:/openmall/software/rabbitmq
scp rabbitmq-server-3.6.5-1.noarch.rpm openmall@node100:/openmall/software/rabbitmq

# 安装
sudo rpm -ivh erlang-18.3-1.el7.centos.x86_64.rpm 
sudo rpm -ivh socat-1.7.3.2-1.1.el7.x86_64.rpm
sudo rpm -ivh rabbitmq-server-3.6.5-1.noarch.rpm

# 卸载
rpm -qa | grep rabbitmq
rpm -e --allmatches rabbitmq-server-3.6.5-1.noarch
rpm -qa | grep erlang
rpm -e --allmatches erlang-18.3-1.el7.centos.x86_64
rpm -qa | grep socat
rpm -e --allmatches socat-1.7.3.2-5.el7.lux.x86_64
rm -rf /usr/lib/rabbitmq/ /etc/rabbitmq/ /var/lib/rabbitmq/
```



## 配置

- `vi /usr/lib/rabbitmq/lib/rabbitmq_server-3.6.5/ebin/rabbit.app`
  - loopback_users 中的 <<"guest">>，修改为 `{loopback_users, []}` （用于用户登录）
  - heartbeat 为10（用于心跳连接）



## 启动

```shell
# 启动服务 stop | status | restart
sudo /etc/init.d/rabbitmq-server start
# 检查
netstat -tunlp | grep 5672

# 启用控制台
sudo rabbitmq-plugins enable rabbitmq_management
# 检查
netstat -tunlp | grep 15672
```



## 测试

访问控制台 http://node100:15672 ，输入用户名和密码：guest / guest



# 集群

## 镜像模式（待验证）

### 服务节点分配

| 服务器IP      | hostname | 节点说明           | 端口 | 管控台地址                                                   |
| ------------- | -------- | ------------------ | ---- | ------------------------------------------------------------ |
| 192.168.11.71 | bhz71    | rabbitmq master    | 5672 | http://192.168.11.71:15672                                   |
| 192.168.11.72 | bhz72    | rabbitmq slave     | 5672 | http://192.168.11.72:15672                                   |
| 192.168.11.73 | bhz73    | rabbitmq slave     | 5672 | http://192.168.11.73:15672                                   |
| 192.168.11.74 | bhz74    | haproxy+keepalived | 8100 | [http://192.168.11.74:8100/rabbitmq-stats](http://192.168.1.27:8100/rabbitmq-stats) |
| 192.168.11.75 | bhz75    | haproxy+keepalived | 8100 | [http://192.168.11.75:8100/rabbitmq-stats](http://192.168.1.28:8100/rabbitmq-stats) |

### 文件同步

71、72、73分别安装rabbitmq，选择71作为Master

```shell
scp /var/lib/rabbitmq/.erlang.cookie 192.168.11.72:/var/lib/rabbitmq/
scp /var/lib/rabbitmq/.erlang.cookie 192.168.11.73:/var/lib/rabbitmq/
```

### 集群组建

```shell
# 停止3个节点
rabbitmqctl stop

# 3个节点分别启动
rabbitmq-server -detached

# slave加入集群
# 192.168.11.72
rabbitmqctl stop_app
rabbitmqctl join_cluster --ram rabbit@bhz71
rabbitmqctl start_app
# 192.168.11.73
rabbitmqctl stop_app
rabbitmqctl join_cluster rabbit@bhz71
rabbitmqctl start_app

# 在另外其他节点上操作要移除的集群节点
# rabbitmqctl forget_cluster_node rabbit@bhz71

# 修改集群名称
rabbitmqctl set_cluster_name rabbitmq_cluster1

# 查看集群状态
rabbitmqctl cluster_status

# 设置镜像队列
rabbitmqctl set_policy ha-all "^" '{"ha-mode":"all"}'
```

### 安装Haproxy

#### 安装

```shell
# 下载
wget http://www.haproxy.org/download/1.6/src/haproxy-1.6.5.tar.gz

# 解压
tar -zxvf haproxy-1.6.5.tar.gz -C /usr/local

# 进入目录、进行编译、安装
cd /usr/local/haproxy-1.6.5
make TARGET=linux31 PREFIX=/usr/local/haproxy
make install PREFIX=/usr/local/haproxy
mkdir /etc/haproxy

# 赋权
groupadd -r -g 149 haproxy
useradd -g haproxy -r -s /sbin/nologin -u 149 haproxy

# 创建haproxy配置文件
touch /etc/haproxy/haproxy.cfg
```

#### 配置

vim /etc/haproxy/haproxy.cfg

```shell
# logging options
global
	log 127.0.0.1 local0 info
	maxconn 5120
	chroot /usr/local/haproxy
	uid 99
	gid 99
	daemon
	quiet
	nbproc 20
	pidfile /var/run/haproxy.pid

defaults
	log global
	# 使用4层代理模式，”mode http”为7层代理模式
	mode tcp
	# if you set mode to tcp,then you nust change tcplog into httplog
	option tcplog
	option dontlognull
	retries 3
	option redispatch
	maxconn 2000
	contimeout 10s
  # 客户端空闲超时时间为 60秒 则HA 发起重连机制
  clitimeout 10s
  # 服务器端链接超时时间为 15秒 则HA 发起重连机制
  srvtimeout 10s	
  # front-end IP for consumers and producters

listen rabbitmq_cluster
	bind 0.0.0.0:5672
	# 配置TCP模式
	mode tcp
	# balance url_param userid
	# balance url_param session_id check_post 64
	# balance hdr(User-Agent)
	# balance hdr(host)
	# balance hdr(Host) use_domain_only
	# balance rdp-cookie
	# balance leastconn
	# balance source //ip
	# 简单的轮询
	balance roundrobin
	# rabbitmq集群节点配置 #inter 每隔五秒对mq集群做健康检查，2次正确证明服务器可用，2次失败证明服务器不可用，并且配置主备机制
  server bhz71 192.168.11.71:5672 check inter 5000 rise 2 fall 2
  server bhz72 192.168.11.72:5672 check inter 5000 rise 2 fall 2
  server bhz73 192.168.11.73:5672 check inter 5000 rise 2 fall 2
  # 配置haproxy web监控，查看统计信息

listen stats
	bind 192.168.11.74:8100
	mode http
	option httplog
	stats enable
	# 设置haproxy监控地址为http://localhost:8100/rabbitmq-stats
	stats uri /rabbitmq-stats
	stats refresh 5s
```

#### 启动

```shell
/usr/local/haproxy/sbin/haproxy -f /etc/haproxy/haproxy.cfg

# 关闭
# killall haproxy
# ps -ef | grep haproxy
# netstat -tunpl | grep haproxy
# ps -ef |grep haproxy |awk '{print $2}'|xargs kill -9
```

#### 测试

http://192.168.11.74:8100/rabbitmq-stats



### 安装Keepalived

#### 安装

``` shell
# 安装所需软件包
yum install -y openssl openssl-devel
# 下载
wget http://www.keepalived.org/software/keepalived-1.2.18.tar.gz
# 解压、编译、安装
tar -zxvf keepalived-1.2.18.tar.gz -C /usr/local/
cd ..
cd keepalived-1.2.18/ && ./configure --prefix=/usr/local/keepalived
make && make install
# 将keepalived安装成Linux系统服务
mkdir /etc/keepalived
cp /usr/local/keepalived/etc/keepalived/keepalived.conf /etc/keepalived/
cp /usr/local/keepalived/etc/rc.d/init.d/keepalived /etc/init.d/
cp /usr/local/keepalived/etc/sysconfig/keepalived /etc/sysconfig/
ln -s /usr/local/sbin/keepalived /usr/sbin/
# 如果存在则进行删除: rm /sbin/keepalived
ln -s /usr/local/keepalived/sbin/keepalived /sbin/
# 设置开机启动
chkconfig keepalived on
```

#### 配置

vim /etc/keepalived/keepalived.conf

192.168.11.74节点

```shell
! Configuration File for keepalived

global_defs {
   router_id bhz74  ##标识节点的字符串，通常为hostname

}

vrrp_script chk_haproxy {
    script "/etc/keepalived/haproxy_check.sh"  ##执行脚本位置
    interval 2  ##检测时间间隔
    weight -20  ##如果条件成立则权重减20
}

vrrp_instance VI_1 {
    state MASTER  ## 主节点为MASTER，备份节点为BACKUP
    interface eno16777736 ## 绑定虚拟IP的网络接口（网卡），与本机IP地址所在的网络接口相同（我这里是eth0）
    virtual_router_id 74  ## 虚拟路由ID号（主备节点一定要相同）
    mcast_src_ip 192.168.11.74 ## 本机ip地址
    priority 100  ##优先级配置（0-254的值）
    nopreempt
    advert_int 1  ## 组播信息发送间隔，俩个节点必须配置一致，默认1s
authentication {  ## 认证匹配
        auth_type PASS
        auth_pass bhz
    }

    track_script {
        chk_haproxy
    }

    virtual_ipaddress {
        192.168.11.70  ## 虚拟ip，可以指定多个
    }
}
```

192.168.11.75节点

```shell
! Configuration File for keepalived

global_defs {
   router_id bhz75  ##标识节点的字符串，通常为hostname

}

vrrp_script chk_haproxy {
    script "/etc/keepalived/haproxy_check.sh"  ##执行脚本位置
    interval 2  ##检测时间间隔
    weight -20  ##如果条件成立则权重减20
}

vrrp_instance VI_1 {
    state BACKUP  ## 主节点为MASTER，备份节点为BACKUP
    interface eno16777736 ## 绑定虚拟IP的网络接口（网卡），与本机IP地址所在的网络接口相同（我这里是eno16777736）
    virtual_router_id 74  ## 虚拟路由ID号（主备节点一定要相同）
    mcast_src_ip 192.168.11.75  ## 本机ip地址
    priority 90  ##优先级配置（0-254的值）
    nopreempt
    advert_int 1  ## 组播信息发送间隔，俩个节点必须配置一致，默认1s
authentication {  ## 认证匹配
        auth_type PASS
        auth_pass bhz
    }

    track_script {
        chk_haproxy
    }

    virtual_ipaddress {
        192.168.1.70  ## 虚拟ip，可以指定多个
    }
}
```

#### 执行脚本

```shell
# 脚本编写
/etc/keepalived/haproxy_check.sh

#!/bin/bash
COUNT=`ps -C haproxy --no-header |wc -l`
if [ $COUNT -eq 0 ];then
    /usr/local/haproxy/sbin/haproxy -f /etc/haproxy/haproxy.cfg
    sleep 2
    if [ `ps -C haproxy --no-header |wc -l` -eq 0 ];then
        killall keepalived
    fi
fi

# 授权
chmod +x /etc/keepalived/haproxy_check.sh
```

#### 启动

```shell
service keepalived start 

# 查看状态
# ps -ef | grep haproxy
# ps -ef | grep keepalived
```



# 插件

## 延迟队列插件

### 下载

```shell
# 从http://www.rabbitmq.com/community-plugins.html下载 rabbitmq_delayed_message_exchange-0.0.1.ez

# 上传到目录 /usr/lib/rabbitmq/lib/rabbitmq_server-3.6.5/plugins
```

### 启用

```shell
# 停止
sudo /etc/init.d/rabbitmq-server stop

# 启用
sudo rabbitmq-plugins enable rabbitmq_delayed_message_exchange

# q
sudo /etc/init.d/rabbitmq-server stop
```

### 添加延迟队列

```shell
# 添加延迟交换机
Type: x-delayed-message
Arguments: x-delayed-type topic
```

