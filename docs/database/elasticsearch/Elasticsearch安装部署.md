# Elasticsearch

## 单机

### 安装

```shell
# 上传
scp elasticsearch-7.6.2-linux-x86_64.tar.gz openmall@node20:/openmall/software/elasticsearch

# 解压
cd /openmall/software/elasticsearch
tar -zxvf elasticsearch-7.6.2-linux-x86_64.tar.gz

mkdir -p /openmall/install/elasticsearch
mv elasticsearch-7.6.2 /openmall/install/elasticsearch
```



### 配置

- elasticsearch.yml

  ```shell
  cd /openmall/install/elasticsearch/elasticsearch-7.6.2/config/
  vi elasticsearch.yml
  
  # 集群名称
  cluster.name: openmall-es
  # 节点名称
  node.name: node20
  # 索引目录
  path.data: /openmall/install/elasticsearch/elasticsearch-7.6.2/data
  # 日志目录
  path.logs: /openmall/install/elasticsearch/elasticsearch-7.6.2/logs
  # 对外可访问
  network.host: 0.0.0.0
  # default discovery settings
  cluster.initial_master_nodes: ["node20"]
  
  # 创建目录
  mkdir -p /openmall/install/elasticsearch/elasticsearch-7.6.2/data
  mkdir -p /openmall/install/elasticsearch/elasticsearch-7.6.2/logs
  ```

- jvm.options

  ```shell
  cd /openmall/install/elasticsearch/elasticsearch-7.6.2/config/
  vi jvm.options
  
  -Xms256m
  -Xmx256m
  ```

- limits.conf

  - <font color=red>注意修改后需要用户重新登录才会生效</font>

  ```shell
  cd /etc/security/
  sudo vi limits.conf
  
  * soft nofile 65536
  * hard nofile 131072
  * soft nproc 2048
  * hard nproc 4096
  ```

- sysctl.conf

  ```shell
  cd /etc/
  sudo vi sysctl.conf
  
  vm.max_map_count=262145
  
  # 生效
  sudo sysctl -p
  ```

  

### 启动

```shell
cd /openmall/install/elasticsearch/elasticsearch-7.6.2
# -d 后台启动
bin/elasticsearch -d
```



### 测试

- 浏览器测试

  - 9200 http协议，外部通讯
  - 9300 tcp协议，es集群间通讯

  ```shell
  http://node20:9200/
  ```

- <font color=red>Postman测试（强烈推荐）</font>

- Chrome安装插件 `Elasticsearch-head`



## 集群

### 分发

- 清空node20目录data和logs

```shell
scp -r /openmall/install/elasticsearch openmall@node21:/openmall/install/

scp -r /openmall/install/elasticsearch openmall@node22:/openmall/install/
```



### 配置

- elasticsearch.yml

  ```shell
  cd /openmall/install/elasticsearch/elasticsearch-7.6.2/config/
  vi elasticsearch.yml
  
  ####
  # node20
  ####
  cluster.name: openmall-es
  node.name: node20
  path.data: /openmall/install/elasticsearch/elasticsearch-7.6.2/data
  path.logs: /openmall/install/elasticsearch/elasticsearch-7.6.2/logs
  network.host: 0.0.0.0
  
  node.master: true	# master节点，管理其他data节点
  node.data: true # data节点
  
  discovery.seed_hosts: ["192.168.1.20","192.168.1.21","192.168.1.22"]	# 集群列表
  cluster.initial_master_nodes: ["node20"] # 初始master节点
  
  
  ####
  # node21
  ####
  # 其他与node20相同
  node.name: node21
  
  ####
  # node22
  ####
  # 其他与node20相同
  node.name: node22
  ```



### 启动

- 三个node分别启动

```shell
cd /openmall/install/elasticsearch/elasticsearch-7.6.2
# -d 后台启动
bin/elasticsearch -d
```



### 测试

#### 浏览器测试

```shell
http://node20:9200/
```



#### 宕机测试

- **node21**（master）宕机
  - **node22**选举为新的master，可以正常操作。
  - 启动node21后，成为data节点，master会重新分配分片
- node20（data）宕机
  - **node22**（master）不变
  - 启动node20后，仍为data节点，master会重新分配分片



## 插件

### 安装IK中文分词器

#### 安装IK

```shell
# 下载对应版本 git@github.com:medcl/elasticsearch-analysis-ik.git
scp elasticsearch-analysis-ik-7.6.2.zip openmall@node20:/openmall/software/elasticsearch

# 安装
sudo yum install -y unzip
cd /openmall/software/elasticsearch
unzip elasticsearch-analysis-ik-7.6.2.zip -d ik
mv ik /openmall/install/elasticsearch/elasticsearch-7.6.2/plugins/

# 重启elasticsearch
```



#### 自定义中文库

```shell
mkdir -p /openmall/install/elasticsearch/elasticsearch-7.6.2/plugins/ik/config/custom
cd /openmall/install/elasticsearch/elasticsearch-7.6.2/plugins/ik/config/custom
vi openmall.dic

# 添加内容
杨晓宇
晓宇

# 修改
cd /openmall/install/elasticsearch/elasticsearch-7.6.2/plugins/ik/config
vi IKAnalyzer.cfg.xml
# 多个用 ; 隔开
<entry key="ext_dict">custom/openmall.dic</entry>
```



#### 测试

```shell
post http://node20:9200/_analyze

{
    "analyzer": "ik_max_word",
    "text": "我的名字是杨晓宇"
}
```



# logstash

## 安装

```shell
# 上传
mkdir -p /openmall/software/logstash
scp logstash-7.6.2.tar.gz openmall@node100:/openmall/software/logstash/

# 解压
cd /openmall/software/logstash
tar -zxvf logstash-7.6.2.tar.gz

mkdir -p /openmall/install/logstash
mv logstash-7.6.2 /openmall/install/logstash
```



## 配置

### 环境准备

```shell
# 创建自定义目录
mkdir -p /openmall/install/logstash/logstash-7.6.2/sync
cd /openmall/install/logstash/logstash-7.6.2/sync

# 创建 logstash-db-sync.conf
vi logstash-db-sync.conf

# 拷贝mysql驱动
scp mysql-connector-java-5.1.47.jar openmall@node100:/openmall/install/logstash/logstash-7.6.2/sync

# 创建同步数据脚本 openmall-items.sql
vi openmall-items.sql

# 创建logstash-ik.json支持ik分词器
vi logstash-ik.json
```



### 创建 logstash-db-sync.conf

- `manage_template => true` 由logstash管理安装自定义模板
- 创建索引
  - 通过指定 `template` 由logstash自动创建
  - 手动创建索引

```shell
input {
    jdbc {
        # 设置 MySql/MariaDB 数据库url以及数据库名称
        jdbc_connection_string => "jdbc:mysql://192.168.1.100:3306/openmall?useUnicode=true&characterEncoding=UTF-8&autoReconnect=true"
        # 用户名和密码
        jdbc_user => "root"
        jdbc_password => "root"
        # 数据库驱动所在位置，可以是绝对路径或者相对路径
        jdbc_driver_library => "/openmall/install/logstash/logstash-7.6.2/sync/mysql-connector-java-5.1.47.jar"
        # 驱动类名
        jdbc_driver_class => "com.mysql.jdbc.Driver"
        # 开启分页
        jdbc_paging_enabled => "true"
        # 分页每页数量，可以自定义
        jdbc_page_size => "1000"
        # 执行的sql文件路径
        statement_filepath => "/openmall/install/logstash/logstash-7.6.2/sync/openmall-items.sql"
        # 设置定时任务间隔  含义：分、时、天、月、年，全部为*默认含义为每分钟跑一次任务
        schedule => "* * * * *"
        # 索引类型
        type => "_doc"
        # 是否开启记录上次追踪的结果，也就是上次更新的时间，会记录到 last_run_metadata_path 目录
        use_column_value => true
        # 记录上一次追踪的结果值
        last_run_metadata_path => "/openmall/install/logstash/logstash-7.6.2/sync/track_time"
        # 如果 use_column_value 为true，配置本参数追踪的 column 名，可以是自增id或者时间
        tracking_column => "updated_time"
        # tracking_column 对应字段的类型
        tracking_column_type => "timestamp"
        # 是否清除 last_run_metadata_path 的记录，true则每次都从头开始查询所有的数据库记录
        clean_run => false
        # 数据库字段名称大写转小写
        lowercase_column_names => false
    }
}
output {
    elasticsearch {
        # es地址
        hosts => ["192.168.1.100:9200"]
        # 同步的索引名
        index => "openmall-items"
        # 设置_id和数据库相同
        document_id => "%{id}"
        # 定义模板名称
        template_name => "logstash-ik"
        # 模板所在位置
        template => "/openmall/install/logstash/logstash-7.6.2/sync/logstash-ik.json"
        # 重写模板
        template_overwrite => true
        # logstash自动管理模板
        manage_template => true
    }
    # 日志输出
    stdout {
        codec => json_lines
    }
}
```



### 创建同步数据脚本 openmall-items.sql

- 由MySQL表的updated_time字段控制，支持数据插入和更新

```sql
SELECT
	i.id,
	i.item_name as itemName,
	i.sell_counts as sellCounts,
	ii.url,
	tempSpec.price_discount as priceDiscount,
	i.updated_time as updated_time
FROM
	items i
LEFT JOIN item_images ii on i.id = ii.item_id
LEFT JOIN (SELECT item_id,MIN(price_discount) as price_discount from item_specs GROUP BY item_id) tempSpec on i.id = tempSpec.item_id
WHERE ii.is_main = 1 and i.updated_time >= :sql_last_value
```



### 创建logstash-ik.json

- `"fielddata": true` 支持排序

```json
{
    "order": 0,
    "version": 1,
    "index_patterns": [
        "*"
    ],
    "settings": {
        "index": {
            "number_of_replicas": 0,
            "number_of_shards": 1,
            "refresh_interval": "5s"
        }
    },
    "mappings": {
        "dynamic_templates": [
            {
                "message_field": {
                    "path_match": "message",
                    "match_mapping_type": "string",
                    "mapping": {
                        "type": "text",
                        "norms": false
                    }
                }
            },
            {
                "string_fields": {
                    "match": "*",
                    "match_mapping_type": "string",
                    "mapping": {
                        "type": "text",
                        "fielddata": true,
                        "norms": false,
                        "analyzer": "ik_max_word",
                        "fields": {
                            "keyword": {
                                "type": "keyword",
                                "ignore_above": 256
                            }
                        }
                    }
                }
            }
        ],
        "properties": {
            "@timestamp": {
                "type": "date"
            },
            "@version": {
                "type": "keyword"
            },
            "geoip": {
                "dynamic": true,
                "properties": {
                    "ip": {
                        "type": "ip"
                    },
                    "location": {
                        "type": "geo_point"
                    },
                    "latitude": {
                        "type": "half_float"
                    },
                    "longitude": {
                        "type": "half_float"
                    }
                }
            }
        }
    },
    "aliases": {}
}
```



## 启动

```shell
bin/logstash -f sync/logstash-db-sync.conf
```

