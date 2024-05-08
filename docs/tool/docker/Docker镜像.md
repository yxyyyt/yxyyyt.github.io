# 下载镜像

## MySQL

```shell
# 下载镜像
docker pull mysql:5.7

# 启动
docker run -p 3306:3306 --name mysql \
-v /root/data/mysql/log:/var/log/mysql \
-v /root/data/mysql/data:/var/lib/mysql \
-v /root/data/mysql/conf:/etc/mysql/conf.d \
-e MYSQL_ROOT_PASSWORD=root  \
-d mysql:5.7

# 进入容器
docker exec -it mysql /bin/bash

# 打开客户端
mysql -uroot -proot --default-character-set=utf8
```



## Redis

```shell
# 下载镜像
docker pull redis:7

# 启动
docker run -p 6379:6379 --name redis \
-v /root/data/redis/data:/data \
-d redis:7 redis-server --appendonly yes

# 进入容器，使用redis-cli命令连接redis服务
docker exec -it redis redis-cli
```



## Nginx

```shell
# 下载镜像
docker pull nginx:1.22

# 首次运行，为了获得配置
docker run -p 80:80 --name nginx \
-v /root/data/nginx/html:/usr/share/nginx/html \
-v /root/data/nginx/logs:/var/log/nginx  \
-d nginx:1.22

# 拷贝nginx配置到容器
docker container cp nginx:/etc/nginx /root/data/nginx/

# 修改配置名称
cd /root/data/nginx
mv nginx conf

# 删除首次运行的容器
docker stop nginx
docker rm nginx

# 正式启动，映射配置
docker run -p 80:80 --name nginx \
-v /root/data/nginx/html:/usr/share/nginx/html \
-v /root/data/nginx/logs:/var/log/nginx  \
-v /root/data/nginx/conf:/etc/nginx \
-d nginx:1.22
```



## RabbitMQ

```shell
# 下载镜像
docker pull rabbitmq:3.9-management

# 启动
docker run -p 5672:5672 -p 15672:15672 --name rabbitmq \
-v /root/data/rabbitmq/data:/var/lib/rabbitmq \
-d rabbitmq:3.9-management

# 开启虚拟机防火墙，使得主机可以访问 http://192.168.8.100:15672（guest/guest）
firewall-cmd --zone=public --add-port=15672/tcp --permanent
firewall-cmd --reload
```



## ELK

### Elasticsearch

```shell
# 下载镜像
docker pull elasticsearch:7.17.3

# 修改虚拟机内存大小
sysctl -w vm.max_map_count=262144

# 修改数据目录权限
mkdir -p /root/data/elasticsearch/data
chmod 777 /root/data/elasticsearch/data/

# 启动
docker run -p 9200:9200 -p 9300:9300 --name elasticsearch \
-e "discovery.type=single-node" \
-e "cluster.name=elasticsearch" \
-e "ES_JAVA_OPTS=-Xms512m -Xmx1024m" \
-v /root/data/elasticsearch/plugins:/usr/share/elasticsearch/plugins \
-v /root/data/elasticsearch/data:/usr/share/elasticsearch/data \
-d elasticsearch:7.17.3

# 安装中文分词器IKAnalyzer，注意下载与Elasticsearch对应的版本，下载地址 https://github.com/medcl/elasticsearch-analysis-ik/releases/tag/v7.17.3
# 下载完成后解压到Elasticsearch的/root/data/elasticsearch/plugins目录下
mkdir -p /root/software/elasticsearch/elasticsearch-analysis-ik-7.17.3
scp D:\download\elasticsearch-analysis-ik-7.17.3.zip root@192.168.8.100:/root/software/elasticsearch/
unzip elasticsearch-analysis-ik-7.17.3.zip -d elasticsearch-analysis-ik-7.17.3
mv elasticsearch-analysis-ik-7.17.3 /root/data/elasticsearch/plugins/

# 重启服务
docker restart elasticsearch

# 开启虚拟机防火墙，使得主机可以访问 http://192.168.8.100:9200
firewall-cmd --zone=public --add-port=9200/tcp --permanent
firewall-cmd --reload
```



### Logstash

```shell
# 下载镜像
docker pull logstash:7.17.3

# 创建目录和文件logstash.conf
mkdir /root/data/logstash
touch logstash.conf

# 拷贝内容到logstash.conf
input {
  tcp {
    mode => "server"
    host => "0.0.0.0"
    port => 4560
    codec => json_lines
    type => "debug"
  }
  tcp {
    mode => "server"
    host => "0.0.0.0"
    port => 4561
    codec => json_lines
    type => "error"
  }
  tcp {
    mode => "server"
    host => "0.0.0.0"
    port => 4562
    codec => json_lines
    type => "business"
  }
  tcp {
    mode => "server"
    host => "0.0.0.0"
    port => 4563
    codec => json_lines
    type => "record"
  }
}
filter{
  if [type] == "record" {
    mutate {
      remove_field => "port"
      remove_field => "host"
      remove_field => "@version"
    }
    json {
      source => "message"
      remove_field => ["message"]
    }
  }
}
output {
  elasticsearch {
    hosts => "es:9200"
    index => "xmall-%{type}-%{+YYYY.MM.dd}"
  }
}

# 启动
# --link 连接容器elasticsearch，可以互相通信
docker run --name logstash -p 4560:4560 -p 4561:4561 -p 4562:4562 -p 4563:4563 \
--link elasticsearch:es \
-v /root/data/logstash/logstash.conf:/usr/share/logstash/pipeline/logstash.conf \
-d logstash:7.17.3

# 进入容器
docker exec -it logstash /bin/bash

# 在容器内安装json_lines插件
logstash-plugin install logstash-codec-json_lines
```



### Kibana

```shell
# 下载镜像
docker pull kibana:7.17.3

# 启动
docker run --name kibana -p 5601:5601 \
--link elasticsearch:es \
-e "elasticsearch.hosts=http://es:9200" \
-d kibana:7.17.3

# 开启虚拟机防火墙，使得主机可以访问 http://192.168.8.100:5601
firewall-cmd --zone=public --add-port=5601/tcp --permanent
firewall-cmd --reload
```



## MongoDB

```shell
# 下载镜像
docker pull mongo:4

# 启动
docker run -p 27017:27017 --name mongo \
-v /root/data/mongo/db:/data/db \
-d mongo:4
```



## MinIO

```shell
# 下载镜像
docker pull minio/minio

# 启动
docker run -p 9090:9000 -p 9001:9001 --name minio \
-v /root/data/minio/data:/data \
-e MINIO_ROOT_USER=minioadmin \
-e MINIO_ROOT_PASSWORD=minioadmin \
-d minio/minio server /data --console-address ":9001"

# 主机可以访问 http://192.168.8.100:9090
```



## Nexus

```shell
# 下载镜像
docker pull sonatype/nexus3

# 创建目录
mkdir -p /root/data/nexus/data
chmod 777 -R /root/data/nexus/data

# 启动
docker run -d --name nexus3 -p 8081:8081 --restart always \
-v /root/data/nexus/data:/nexus-data \
sonatype/nexus3

# 查看日志，出现 "Started Sonatype Nexus OSS 3.64.0-04" 启动成功；主机可以访问 http://192.168.8.100:8081
docker logs -f nexus3

# 首次登录查看并在管理界面修改密码（admin/密文）
cat /root/data/nexus/data/admin.password
```

