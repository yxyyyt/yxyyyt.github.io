# Docker

## 单机

```shell
# 下载最新镜像
docker pull redis

# 查看redis版本 -i 忽略大小写
# docker image inspect redis | grep -i 'version'

# 创建容器并运行
docker run -itd --name redis-test -p 6379:6379 redis

# 显示容器运行时的完整命令
# 	docker-entrypoint.sh redis-server
#		默认行为 Entrypoint + Cmd，其中Cmd可以被docker run命令中image后的CMD参数替换
# docker ps --no-trunc

# 进入容器
docker exec -it redis-test /bin/bash

# 进入redis命令交互模式
redis-cli

# 性能测试
# redis-benchmark -n 100000 -c 32 -t SET,GET,INCR,HSET,LPUSH,MSET -q
```



## 集群

### 主从复制

#### 配置网络

- 使用 `tunnelblick` + `mac-network` 方案使得宿主机可以直接访问docker容器，docker容器通过bridge组网。因为宿主机可以独立访问容器的IP和端口，所以使用此种方式，可以不需要向宿主机映射端口

```shell
# 创建专用网络
docker network create --driver=bridge --subnet=172.82.0.0/24 redisnet
```



#### 配置redis.conf

```properties
# redis01 主
port 6379
logfile "/log/redis.log"

# redis02 从
port 6379
replicaof 172.82.0.100 6379
logfile "/log/redis.log"

# redis03 从
port 6379
replicaof 172.82.0.100 6379
logfile "/log/redis.log"
```



#### 启动redis服务

```shell
# 主
docker run -p 6379:6379 --name redis01 --hostname redis01 --net=redisnet --ip=172.82.0.100 \
-v /Users/yangxiaoyu/work/test/redisdatas/redis01/redis.conf:/etc/redis/redis.conf \
-v /Users/yangxiaoyu/work/test/redisdatas/redis01/data:/data \
-v /Users/yangxiaoyu/work/test/redisdatas/redis01/log:/log \
-v /Users/yangxiaoyu/work/test/redisdatas/exchange:/exchange \
-d redis redis-server /etc/redis/redis.conf

# 从
docker run -p 6380:6379 --name redis02 --hostname redis02 --net=redisnet --ip=172.82.0.101 \
-v /Users/yangxiaoyu/work/test/redisdatas/redis02/redis.conf:/etc/redis/redis.conf \
-v /Users/yangxiaoyu/work/test/redisdatas/redis02/data:/data \
-v /Users/yangxiaoyu/work/test/redisdatas/redis02/log:/log \
-v /Users/yangxiaoyu/work/test/redisdatas/exchange:/exchange \
-d redis redis-server /etc/redis/redis.conf

# 从
docker run -p 6381:6379 --name redis03 --hostname redis03 --net=redisnet --ip=172.82.0.102 \
-v /Users/yangxiaoyu/work/test/redisdatas/redis03/redis.conf:/etc/redis/redis.conf \
-v /Users/yangxiaoyu/work/test/redisdatas/redis03/data:/data \
-v /Users/yangxiaoyu/work/test/redisdatas/redis03/log:/log \
-v /Users/yangxiaoyu/work/test/redisdatas/exchange:/exchange \
-d redis redis-server /etc/redis/redis.conf
```



#### 验证

- 参数

  ```shell
  # redis01
  info replication
  # role:master
  # connected_slaves:2
  # slave0:ip=172.82.0.101,port=6379,state=online,offset=809,lag=1
  # slave1:ip=172.82.0.102,port=6379,state=online,offset=809,lag=1
  
  # redis02
  info replication
  # role:slave
  # master_host:172.82.0.100
  # master_port:6379
  
  # redis03
  info replication
  # role:slave
  # master_host:172.82.0.100
  # master_port:6379
  ```

- 日志

  ```shell
  1:M 18 Feb 2021 09:12:41.091 * Synchronization with replica 172.82.0.101:6379 succeeded
  1:M 18 Feb 2021 09:21:18.626 * Synchronization with replica 172.82.0.102:6379 succeeded
  ```

- 命令

  - redis01（主）对key 增 / 删 / 改 会同步到redis02（从）和redis03（从）上。redis01可以查询数据
  - redis02（从）和redis03（从）只读



### 高可用

以主从复制的三个主从节点为基础，搭建sentinel。

#### 配置sentinel.conf

```shell
# sentinel01
port 26379
logfile "/log/sentinel.log"
# 配置主，不需要配置从，可以通过主获取到需要监控的从
# 最后一个2表示两台sentinel判定主被动下线后，就进行failover(故障转移)
sentinel monitor mymaster 172.82.0.100 6379 2
# 3s内mymaster无响应，则认为mymaster宕机了
sentinel down-after-milliseconds mymaster 3000
# 如果10秒后mymaster仍没启动过来，则启动failover  
sentinel failover-timeout mymaster 10000
# 限制同时同新主同步的从数量，即执行故障转移时最多有1个从同新的主进行同步，此时这个从不可用；之后其他从轮询同主进行同步；因此值越小，同步的时间就越久，但值越大，也会造成一段时间内多个从不可用的问题
sentinel parallel-syncs mymaster 1

# sentinel02
port 26379
logfile "/log/sentinel.log"
sentinel monitor mymaster 172.82.0.100 6379 2
sentinel down-after-milliseconds mymaster 3000
sentinel failover-timeout mymaster 10000
sentinel parallel-syncs mymaster 1

# sentinel03
port 26379
logfile "/log/sentinel.log"
sentinel monitor mymaster 172.82.0.100 6379 2
sentinel down-after-milliseconds mymaster 3000
sentinel failover-timeout mymaster 10000
sentinel parallel-syncs mymaster 1
```



#### 启动sentinel服务

```shell
# sentinel01
docker run -p 26379:26379 --name sentinel01 --hostname sentinel01 --net=redisnet --ip=172.82.0.200 \
-v /Users/yangxiaoyu/work/test/redisdatas/sentinel01/sentinel.conf:/etc/redis/sentinel.conf \
-v /Users/yangxiaoyu/work/test/redisdatas/sentinel01/data:/data \
-v /Users/yangxiaoyu/work/test/redisdatas/sentinel01/log:/log \
-v /Users/yangxiaoyu/work/test/redisdatas/exchange:/exchange \
-d redis redis-sentinel /etc/redis/sentinel.conf

# sentinel02
docker run -p 26380:26379 --name sentinel02 --hostname sentinel02 --net=redisnet --ip=172.82.0.201 \
-v /Users/yangxiaoyu/work/test/redisdatas/sentinel02/sentinel.conf:/etc/redis/sentinel.conf \
-v /Users/yangxiaoyu/work/test/redisdatas/sentinel02/data:/data \
-v /Users/yangxiaoyu/work/test/redisdatas/sentinel02/log:/log \
-v /Users/yangxiaoyu/work/test/redisdatas/exchange:/exchange \
-d redis redis-sentinel /etc/redis/sentinel.conf

# sentinel03
docker run -p 26381:26379 --name sentinel03 --hostname sentinel03 --net=redisnet --ip=172.82.0.202 \
-v /Users/yangxiaoyu/work/test/redisdatas/sentinel03/sentinel.conf:/etc/redis/sentinel.conf \
-v /Users/yangxiaoyu/work/test/redisdatas/sentinel03/data:/data \
-v /Users/yangxiaoyu/work/test/redisdatas/sentinel03/log:/log \
-v /Users/yangxiaoyu/work/test/redisdatas/exchange:/exchange \
-d redis redis-sentinel /etc/redis/sentinel.conf
```



#### 验证

- 日志

  ```shell
  1:X 18 Feb 2021 12:56:14.055 # +monitor master mymaster 172.82.0.100 6379 quorum 2
  # 两从
  1:X 18 Feb 2021 12:56:14.060 * +slave slave 172.82.0.101:6379 172.82.0.101 6379 @ mymaster 172.82.0.100 6379
  1:X 18 Feb 2021 12:56:14.072 * +slave slave 172.82.0.102:6379 172.82.0.102 6379 @ mymaster 172.82.0.100 6379
  # 另外两个sentinel
  1:X 18 Feb 2021 12:56:23.955 * +sentinel sentinel cf2c4ed31c1d46f68bc2fd73ece3b7af96043bd0 172.82.0.201 26379 @ mymaster 172.82.0.100 6379
  1:X 18 Feb 2021 12:56:30.776 * +sentinel sentinel 6f2cd400137500c1a248e6a7074e568c082f02a9 172.82.0.202 26379 @ mymaster 172.82.0.100 6379
  ```

- 命令

  - 主redis01下线
    - redis03被选举为主，redis02为从，符合主从复制约定限制
    - 主redis01上线，成为从，同步主数据
  - 从redis02下线
    - 从redis02上线，仍为从，同步主数据

- 参数

  ```shell
  # 连接到sentinel服务
  redis-cli -h localhost -p 26379
  
  # master清单
  sentinel masters
  
  # 指定master的slave清单
  sentinel slaves mymaster
  
  # 指定master的sentinel清单
  sentinel sentinels mymaster
  ```

  

### 集群

#### 配置redis.conf

```shell
# redis7000 | redis7001 | redis7002 | redis7003 | redis7004 | redis7005
port 7000
cluster-enabled yes	# 启用集群支持
cluster-config-file nodes.conf # 由redis集群维护，记录集群节点状态等信息
cluster-node-timeout 5000 # 集群不可用最长时间
appendonly yes
daemonize no
protected-mode no
pidfile  /data/redis.pid
```



#### 启动redis服务

```shell
# redis7000
docker run --name redis7000 --net=redisnet --ip=172.82.0.70 -v /Users/yangxiaoyu/work/test/redisdatas/cluster/7000:/data -d redis redis-server /data/redis.conf

# redis7001
docker run --name redis7001 --net=redisnet --ip=172.82.0.71 -v /Users/yangxiaoyu/work/test/redisdatas/cluster/7001:/data -d redis redis-server /data/redis.conf

# redis7002
docker run --name redis7002 --net=redisnet --ip=172.82.0.72 -v /Users/yangxiaoyu/work/test/redisdatas/cluster/7002:/data -d redis redis-server /data/redis.conf

# redis7003
docker run --name redis7003 --net=redisnet --ip=172.82.0.73 -v /Users/yangxiaoyu/work/test/redisdatas/cluster/7003:/data -d redis redis-server /data/redis.conf

# redis7004
docker run --name redis7004 --net=redisnet --ip=172.82.0.74 -v /Users/yangxiaoyu/work/test/redisdatas/cluster/7004:/data -d redis redis-server /data/redis.conf

# redis7005
docker run --name redis7005 --net=redisnet --ip=172.82.0.75 -v /Users/yangxiaoyu/work/test/redisdatas/cluster/7005:/data -d redis redis-server /data/redis.conf
```



#### 创建集群

- 副本数为1，6台机器，即3主3从

```shell
# -p 指定连接 Redis 的端口
# --cluster 使用 Redis 集群模式命令
# create 创建 Redis 集群
# --cluster-replicas 指定副本数（slave 数量）
docker exec -it redis7000 \
redis-cli -p 7000 --cluster create \
172.82.0.70:7000 172.82.0.71:7000 172.82.0.72:7000 \
172.82.0.73:7000 172.82.0.74:7000 172.82.0.75:7000 \
--cluster-replicas 1
```

输出关键信息

```shell
Master[0] -> Slots 0 - 5460
Master[1] -> Slots 5461 - 10922
Master[2] -> Slots 10923 - 16383
Adding replica 172.82.0.74:7000 to 172.82.0.70:7000
Adding replica 172.82.0.75:7000 to 172.82.0.71:7000
Adding replica 172.82.0.73:7000 to 172.82.0.72:7000
```



####验证

- 查看集群信息

  登录任意节点

  ```shell
  # -p：指定连接 Redis 的端口；
  # -c：使用集群模式；
  docker exec -it redis7000 redis-cli -p 7000 -c
  ```

  cluster info

  ```shell
  cluster_state:ok
  cluster_slots_assigned:16384
  cluster_slots_ok:16384
  cluster_slots_pfail:0
  cluster_slots_fail:0
  cluster_known_nodes:6
  cluster_size:3
  cluster_current_epoch:6
  cluster_my_epoch:1
  cluster_stats_messages_ping_sent:1014
  cluster_stats_messages_pong_sent:1031
  cluster_stats_messages_sent:2045
  cluster_stats_messages_ping_received:1026
  cluster_stats_messages_pong_received:1014
  cluster_stats_messages_meet_received:5
  cluster_stats_messages_received:2045
  ```

  cluster nodes 查看主从关联情况

  ```shell
  1f1bfcb5d7f8da84b6b2ce418260b19edb4b4731 172.82.0.73:7000@17000 slave a223227673365bab768d52bce81fe57c74a70c78 0 1613826416777 3 connected
  a1a0e1df50d7b217f0a6e81de7d549168cd158d3 172.82.0.71:7000@17000 master - 0 1613826417000 2 connected 5461-10922
  a223227673365bab768d52bce81fe57c74a70c78 172.82.0.72:7000@17000 master - 0 1613826417593 3 connected 10923-16383
  c9cb4de4f0980694190f8096fe12e21503dfa7ea 172.82.0.75:7000@17000 slave a1a0e1df50d7b217f0a6e81de7d549168cd158d3 0 1613826417000 2 connected
  f31b9855b0314eca9c4b6a431b19a1be5f953b74 172.82.0.74:7000@17000 slave bb49e8c8ff1c7c8c5b58f6bfae09d16ad5a249ab 0 1613826417801 1 connected
  bb49e8c8ff1c7c8c5b58f6bfae09d16ad5a249ab 172.82.0.70:7000@17000 myself,master - 0 1613826415000 1 connected 0-5460
  ```

- 命令

  以单机模式运行master客户端

  ```shell
  docker exec -it redis7000 redis-cli -p 7000
  
  # 提示 (error) MOVED 6257 172.82.0.71:7000	（由71的slot管理key）
  # 需要在redis7001执行该命令才会成功；同样，也只有在redis7001才可以查询
  set msg helloworld
  ```

  以集群模式运行master客户端

  - <font color=red>在集群模式下任意客户端可以写入，也可以读取；redis客户端会自动重定向，不需要切换客户端</font>

  ```shell
  docker exec -it redis7000 redis-cli -p 7000 -c
  
  # 在redis7000执行，请求被重定向到redis7001
  # redis7000可以查询，也可以读取
  # -> Redirected to slot [6257] located at 172.82.0.71:7000
  # OK	（自动切换，客户端通过第一次返回的正确主机，与正确主机进行第二次通讯）
  set msg helloworld
  ```

  以单机模式运行slave客户端

  - <font color=red>当设置为 READONLY 模式时，可以只读查询；当不设置此参数时，slave只作为备份节点，不提供操作</font>

  ```shell
  # redis7005是redis7001的slave
  docker exec -it redis7005 redis-cli -p 7000
  
  # 无法查询
  get msg
  ```

  以集群模式运行slave客户端

  - <font color=red>注意当重定向后，keys * 只能查询到重定向节点的数据</font>

  ```shell
  docker exec -it redis7005 redis-cli -p 7000 -c
  
  # 请求被重定向（自动切换，客户端通过第一次返回的正确主机，与正确主机进行第二次通讯）
  # -> Redirected to slot [6257] located at 172.82.0.71:7000
  # "helloworld"
  get msg
  ```



- 测试高可用

  ```shell
  # 目前关联情况
  # master -> slave
  172.82.0.70 -> 172.82.0.74
  172.82.0.71 -> 172.82.0.75
  172.82.0.72 -> 172.82.0.73
  
  # 172.82.0.70下线，172.82.0.74升级为master，slots保持不变
  # 172.82.0.74下线，(error) CLUSTERDOWN The cluster is down 导致集群不可用
  
  # 172.82.0.74上线为master，处理数据正常
  # 172.82.0.70上线为slave，成为备份节点
  
  # 目前关联情况
  # master -> slave
  172.82.0.74 -> 172.82.0.70
  172.82.0.71 -> 172.82.0.75
  172.82.0.72 -> 172.82.0.73
  ```

  

