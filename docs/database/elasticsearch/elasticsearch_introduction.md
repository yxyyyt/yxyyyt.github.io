# 核心功能

## 节点角色

在Elasticsearch中，节点的角色分为三种，分别为master node，data node，ingest node。在一个ES节点中，每一个节点都可以充当一个或者多个角色，但是在较大的集群中，一般会将每一个角色都分开，这样可以保证集群的稳定性。

可以通过下面的配置：

```shell
node.master: true
node.data: true
node.ingest: true

# 当一个节点开启http选项之后，就有能力充当client的角色，即官方文档中提到的coordinating only node
http.enabled: true
```

- `master node` 负责更新和同步集群信息，如集群配置，集群健康情况，index的信息等等，对集群的稳定性十分重要。其中 `node.master`被设为true， 并不表示这个节点被设置为master，而是有资格被选为master，实际上**当前活动的master只有一个**，如果没有被选上的，只会执行其他角色的任务，一般建议将master和data分开，在较大的集群中，一般设置三个master节点，以提高稳定性
- `data node` 负责储存数据的节点
- `ingest node` 是专门对文档进行预处理的，有点像logstash的功能，是es 5.x新推出的功能
- `client/coordinating node` 指开启了http端口的节点，这类节点可以接收用户的请求，如搜索，索引等，然后将请求分发到不同的数据节点，接收到数据节点的结果后进行汇总，然后返回给用户，这类节点对内存比较敏感， 特别是大量请求的时候



## 读流程

- 查询（指定doc id查询）
  - 客户端发送请求到协调节点
  - 协调节点根据doc id轮询请求shard或replica数据节点（负载均衡）
  - 协调节点将数据返回客户端
- 搜索
  - 客户端发送请求到协调节点
  - 协调节点将搜索请求转发到所有shard对应的 primary shard 或 replica shard
  - 协调节点获取到与搜索内容匹配的doc id
  - 协调节点请求shard或replica数据节点拉取doc id对应的完整数据，对数据进行合并，排序，分页等操作
  - 协调节点将数据返回客户端



## 写流程

- 客户端发送请求到协调节点
- 协调节点对文档路由（id hash）转发到数据节点 primary shard
- primary shard 处理请求，将数据同步到所有的replica shard
- 协调节点发现 primary shard 和所有 replica shard 处理完成后，响应客户端



## 脑裂

Elasticsearch 6.x 通过 `discovery.zen.minimum_master_nodes` 参数防止脑裂。一般设置为 `N/2 + 1`，其中N是集群中标记为master节点的数量，默认是1；7.x 没有此参数，由系统控制。



## 性能优化

- 读性能优化

  ES会将读到的数据放到os cache中，因此读磁盘数据，或是读内存数据，有显著的性能差异。ES读性能严重依赖于内存，因此，内存容量要和数据容量大小相差不大，这样可以缓存大部分数据。

  - 数据尽量少，这样内存可以放尽量多的数据；通过ES+hbase一起实现存储，ES负责查询，符合条件的再去hbase中获取
  - 数据预热
  - 数据冷热分离
  - document设计为宽表，尽量避免多表关联操作
  - 不允许深度分页



# 分词与内置分词器

把文本转换为单词，称之为分词analysis。Elasticsearch默认只支持对英文语句的分词，中文不支持，每个中文字都会被拆分为独立的词。



## 内置分词器

- standard 默认分词，单词会被拆分。大写转换为小写
- simple 按照非字母分词。大写转换为小写
- whitespace 按照空格分词。忽略大小写
- stop 去除无意义单词。如 the/a/an/is ...
- keyword 不做分词。把整个文本作为一个单独的关键词。



## IK中文分词器

- ik_max_word 会将文本做最细粒度的拆分；尽可能多的拆分出词语
- ik_smart 会做最粗粒度的拆分；已被分出的词语将不会再次被其它词语占有



# Http操作

## 索引

### 集群健康

```shell
get http://node20:9200/_cluster/health
```



### 创建索引

```shell
put http://node20:9200/users
{
	"settings": {
		"index": {
			"number_of_shards": "2",
			"number_of_replicas": "0"
		}
	}
}
```



### 查看索引

```shell
get http://node20:9200/_cat/indices?v
```



### 删除索引

```shell
delete http://node20:9200/users
```



## 索引的mappings映射

### 创建索引同时创建mappings

- users是index，相当于数据库
  - 默认index的shard是5，replica是1
- user是type，相当于表
  - 6.x 用户可自定义
  - <font color=red>7.x 取消type，隐式为 _doc</font>
- properties包含了field，相当于表的字段
  - type
    - text 需要被分词，创建倒排索引；keyword不会被分词，不会创建倒排索引，精确匹配
    - long；integer；short；byte
    - double；float
    - boolean
    - date
    - object
    - 数组，类型需要一致
  - index：默认是true需要索引；false不需要索引

```shell
# 创建索引和mappings
put http://node20:9200/users

{
	"mappings": {
        "user":{
            "properties": {
                "realname": {
                    "type": "text",
                    "index": true
                },
                "username": {
                    "type": "keyword",
                    "index": false
                }
            }
	    }
    }
}


# 7.x 取消type
{
    "mappings": {
        "properties": {
            "realname": {
                "type": "text",
                "index": true
            },
            "username": {
                "type": "keyword",
                "index": false
            }
        }
    }
}
```



### 查询分词效果

```shell
get http://node20:9200/users/_analyze

{
	"field": "realname",
	"text": "openmall is good"
}
```



### 增加映射

- 已经定义好的field不可以修改；但可以新增不存在的field

```shell
# for 6.x
put http://node20:9200/users/_mapping/user

{
	"properties": {
				"id": {
   	        "type": "long"
  	    },
        "age": {
            "type": "integer"
        },
        "sex": {
            "type": "byte"
        },
        "birthday": {
            "type": "date"
        },
        "relationship": {
            "type": "object"
        }
	}
}

# for 7.x
put http://node20:9200/users/_mapping
```



## 文档

### 新增文档

- `/index/type/id` 
  - index 索引
  - type 文档类型
  - id 索引中的唯一id，可以和业务主键相同
- json数据是业务数据
- 如果index没有创建mappings，当插入文档时，会根据文档类型自动设置filed（动态映射）。

```shell
post http://node20:9200/users/user/1

{
    "id": 1001,
    "realname": "my realname is big rain",
    "username": "rain",
    "age": 18,
    "sex": 1,
    "birthday": "2020-11-11"
}


# for 7.x 必须加 _doc
post http://node20:9200/users/_doc/1
```



### 删除文档

- 文档删除不是立即删除，文档还是保存在磁盘上，当索引增长到一定程度，才会把标记删除的文档物理清除
- "_version" 会改变

```shell
delete http://node20:9200/users/user/1

# for 7.x 
delete http://node20:9200/users/_doc/2
```



### 修改文档

#### 部分修改文档

- "_version" 会改变
- 注意要有"doc"标识
- <font color=red>乐观锁</font>
  - < 6.7 `_version` 控制版本
    - ?version=x
  - `>=6.7` `_seq_no` 文档编号 和 `_primary_term` 文档所在位置，控制版本
    - ?if_seq_no=x&if_primary_term=y

```shell
post http://node20:9200/users/user/1/_update

{
    "doc": {
        "age": 28
    }
}

# for 7.x optimistic
post http://node20:9200/users/_doc/1/_update?if_seq_no=9&if_primary_term=1
```



#### 全量修改文档

- "_version" 会改变
- 全量替换原有doc的所有field

```shell
post http://node20:9200/users/user/1

{
    "id": 1001,
    "realname": "my realname is big rain, so please call me big rain, thx",
    "username": "rain"
}

# for 7.x
post http://node20:9200/users/_doc/1
```



### 查询文档

#### 常规查询

```shell
# 查询指定文档
get http://node20:9200/users/user/1

# 查询全部数据
get http://node20:9200/users/user/_search

# 查询指定field
get http://node20:9200/users/user/1?_source=id,realname

# for 7.x
get http://node20:9200/users/_doc/1
```



#### DSL搜索

##### 数据准备

```shell
# 增加映射
put http://node20:9200/users/_mapping/user

# for 7.x
put http://node20:9200/users/_mapping

{
	"properties": {
        "desc": {
            "type": "text",
            "analyzer": "ik_max_word"
        }
	}
}

# 录入数据
post http://node20:9200/users/user/1001

# for 7.x
post http://node20:9200/users/_doc/1001

{
    "id": 1001,
    "realname": "my realname is big rain",
    "username": "rain",
    "age": 18,
    "sex": 1,
    "birthday": "2020-11-11",
    "desc": "杨晓宇爱不爱学习"
}

{
    "id": 1002,
    "realname": "my realname is small rain",
    "username": "small rain",
    "age": 28,
    "sex": 1,
    "birthday": "2020-11-12",
    "desc": "杨晓宇爱不爱学习我是不知道的，他的小名是叫晓宇吗"
}

{
    "id": 1003,
    "realname": "my realname is old rain",
    "username": "old rain",
    "age": 38,
    "sex": 1,
    "birthday": "2020-11-13",
    "desc": "我的名字是杨晓宇，没有人知道我的名字"
}

{
    "id": 1004,
    "realname": "my realname is red rain",
    "username": "red rain",
    "age": 48,
    "sex": 1,
    "birthday": "2020-11-14",
    "desc": "从现在开始杨晓宇开始学习杨晓宇的知识"
}

{
    "id": 1005,
    "realname": "my realname is lucky",
    "username": "lucky dog",
    "age": 4,
    "sex": 0,
    "birthday": "2017-11-14",
    "desc": "我是一只白色的比熊晓宇狗狗"
}
```



##### QueryString查询

- 查询参数放在URL中

```shell
# realname的类型是text，全文检索
get http://node20:9200/users/user/_search?q=realname:rain

# 多项匹配
get http://node20:9200/users/user/_search?q=realname:rain&q=age:48

# for 7.x
get http://node20:9200/users/_doc/_search?q=realname:rain
get http://node20:9200/users/_doc/_search?q=realname:rain&q=age:48
```



##### DSL查询

- Domain Specific Language 特定领域语言
- 基于JSON格式的数据查询
- 查询更灵活，有利于复杂查询
- <font color=red>深度分页</font>
  - 原理：查询 `from=9999 size=10` 从9999到10009的记录。从每个分片（假设3个）都会查询10009条记录，然后汇集到一起，也就是10009*3条记录，然后再排序，最后再取从9999到10009的10条记录。因此，搜索的深度越深，就会越影响性能。
  - 解决
    - 从业务层面避免深度分页，限制分页的页数，如最多展示100页。因为用户的习惯而言，查询最多看10几页。如淘宝只能查询前100页
    - 从技术层面限制深度，Elasticsearch不支持一万条数据以上的分页查询

```shell
post http://node20:9200/users/user/_search

# for 7.x
post http://node20:9200/users/_doc/_search

# 全文检索，会先对 查询条件 分词
{
	"query": {
		"match": {
			"desc": "杨晓宇"
		}
	}
}

# 短语匹配，查询分词都存在，顺序必须相同
# slop 允许分词间跳过的数量，越小越苛刻
{
	"query": {
		"match_phrase": {
			"desc": {
				"query": "白色 狗狗",
				"slop": 4
			}
		}
	}
}

# 把查询条件完整的词汇做全文检索，不会对查询条件分词
{
	"query": {
		"term": {
			"desc": "杨晓宇"
		}
	}
}

# 可以用于tag查询
{
	"query": {
		"terms": {
			"desc": ["现在","名字"]
		}
	}
}

# 查询字段是否存在
{
	"query": {
		"exists": {
			"field": "desc"
		}
	}
}

# 查询所有doc的指定field
{
    "query": {
        "match_all": {}
    },
    "_source": [
        "id",
        "realname",
        "desc"
    ]
}

# 分页查询
{
    "query": {
        "match_all": {}
    },
    "_source": [
        "id",
        "realname",
        "desc"
    ],
    "from": 0,
    "size": 2
}

# 查询多个es主键
{
	"query": {
		"ids": {
			"type": "user",
			"values": ["1001","1005"]
		}
	}
}

# for 7.x
{
	"query": {
		"ids": {
			"type": "_doc",
			"values": ["1001","1005"]
		}
	}
}

# 匹配多个field
{
	"query": {
		"multi_match": {
			"query": "old狗狗",
			"fields": ["realname","desc"]
		}
	}
}

# 按权重搜索，权重高的排在前边，提高搜索体验
# desc^10 表示提升10倍相关性
{
	"query": {
		"multi_match": {
			"query": "old狗狗",
			"fields": ["realname","desc^10"]
		}
	}
}
```

