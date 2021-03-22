---
title: 服务化01-Elasticsearch集群
date: 2021-03-21 22:02:27
tags: micro-service
---
本文介绍下Elasticsearch集群的相关知识。
## 什么是Elasticsearch
>Elasticsearch 是一个分布式、高扩展、高实时的搜索与数据分析引擎。它能很方便的使大量数据具有搜索、分析和探索的能力。充分利用Elasticsearch的水平伸缩性，能使数据在生产环境变得更有价值。

## 为什么要用集群
用ElasticSearch集群，将单个索引的分片到多个不同分布式物理机器上存储，从而实现高可用、容错性等。ElasticSearch会自动选举实现主备服务器。

达到基本高可用，一般要至少启动3个节点，3个节点互相连接，单个节点包括所有角色，其中任意节点停机集群依然可用。

![image](https://pics3.baidu.com/feed/94cad1c8a786c91736f0a154e533f0c93ac75784.jpeg?token=1dd5eac3f42387f2cab5f579d6e99605)

## 关键概念
### 节点角色
- Master，集群管理
- Voting，投票选举节点
- Data，数据节点
- Ingest，数据编辑节点
- Coordinate，协调节点
- Machine Learning，集群学习节点

### 索引
一般意义上的索引是一种基于文档（数据）生成、建立的，用于快速定位指定文档的工具。而 ElasticSearch 对索引的定义有所不同，ElasticSearch 中的索引对应 MySQL 中的 Database ，也就说 ElasticSearch 中的索引更像是一种数据存储集合，即用于存储文档。

### 分片
分片（Shard），是Elasticsearch中的最小存储单元。
- Shards分片：代表索引分片，ElasticSearch可以把一个完整的索引分成多个分片，这样的好处是可以把一个
大的索引拆分成多个，分布到不同的节点上。构成分布式搜索。分片的数量只能在索引创建前指定，并且索引
创建后不能更改。
- Replicas分片：代表副本分片，ElasticSearch可以设置多个索引的副本，副本的作用一是提高系统的容错性，
当某个节点某个分片损坏或丢失时可以从副本中恢复。二是提高ElasticSearch的查询效率，ElasticSearch
会自动对搜索请求进行负载均衡。

## 数据路由
- 当客户端发起创建document的时候，ElasticSearch需要确定这个document放在该索引哪个shard上。这个过程就是数据路由。
- 路由算法：分片位置shard=hash(routing) % number_of_primary_shards
- 如果number_of_primary_shards在查询的时候取余发生变化，则无法获取到该数据。即：在查询的时候，底层根据文档id%主分片数量获取分片位置


![image](https://upload-images.jianshu.io/upload_images/7017386-85f5867459bca717)
```
1.客户端向 ES1节点（协调节点）发送写请求，通过路由计算公式得到值为0，则当前数据应被写到主分片 S0 上。
2.ES1 节点将请求转发到 S0 主分片所在的节点 ES3，ES3 接受请求并写入到磁盘。
3.并发将数据复制到两个副本分片 R0 上，其中通过乐观并发控制数据的冲突。一旦所有的副本分片都报告成功，则节点 ES3 将向协调节点报告成功，协调节点向客户端报告成功。
```


### 脑裂现象
正常情况下，集群中的所有节点，应该对主节点的选择是一致的，即一个集群中只有一个选举出来的主节点。然而，在某些情况下，比如网络通信出现问题、主节点因为负载过大停止响应等等，就会导致重新选举主节点，此时可能会出现集群中有多个主节点的现象，即节点对集群状态的认知不一致，称之为脑裂现象。


为了尽量避免此种情况的出现，可以通过`discovery.zen.minimum_master_nodes`来设置最少可工作的候选主节点个数，建议设置为(候选主节点数 / 2) + 1。保证集群中有半数以上的候选主节点。

## 安装
现在开始介绍如何通过`docker`搭建一个简单的Elasticsearch集群服务

<!--more-->

- 项目结构
```
.
├── config  # 配置
│   ├── es01
│   ├── es02
│   └── es03
├── data    # 数据
│   ├── es01
│   ├── es02
│   └── es03
├── logs    # 日志
│   ├── es01
│   ├── es02
│   └── es03
└── docker-componse.yml
```

- docker-compose.yml
```
version: '2'
services:
  es01:
    image: elasticsearch:7.6.1
    container_name: es01
    restart: always
    environment:
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - "TAKE_FILE_OWNERSHIP=true"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - ./data/es01:/usr/share/elasticsearch/data
      - ./config/es01/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
      - ./logs/es01:/usr/share/elasticsearch/logs
    ports:
      - 9200:9200
      - 9300:9300
    networks:
      - esnet
  es02:
    image: elasticsearch:7.6.1
    container_name: es02
    restart: always
    environment:
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - "TAKE_FILE_OWNERSHIP=true"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - ./data/es02:/usr/share/elasticsearch/data
      - ./config/es02/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
      - ./logs/es02:/usr/share/elasticsearch/logs
    ports:
      - 9201:9200
      - 9301:9300
    depends_on:
      - es01
    networks:
      - esnet
  es03:
    image: elasticsearch:7.6.1
    container_name: es03
    restart: always
    environment:
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - "TAKE_FILE_OWNERSHIP=true"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - ./data/es03:/usr/share/elasticsearch/data
      - ./config/es03/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
      - ./logs/es03:/usr/share/elasticsearch/logs
    ports:
      - 9202:9200
      - 9302:9300
    depends_on:
      - es01
    networks:
      - esnet

networks:
  esnet:
```

- config/es01/elasticsearch.yml
```
# 集群名称
cluster.name: es-cluster

# 名称
node.name: es01

# 主节点
node.master: true

# 需要存储数据
node.data: true

# 数据文件
path.data: /usr/share/elasticsearch/data

# 日志文件
path.logs: /usr/share/elasticsearch/logs

# 锁定物理内存地址，防止es内存被交换出去，也就是避免es使用swap交换分区，频繁的交换，会导致IOPS变高
bootstrap.memory_lock: true

# 服务地址
network.host : 0.0.0.0

# 端口,默认为9200
http.port : 9200

# 内部节点之间沟通端口
transport.tcp.port : 9300

# 服务发现种子主机，这里的172.17.0.1是我的docker网关地址，只有配置这个可访问的地址，es才能发现各个节点
discovery.seed_hosts: ["172.17.0.1:9300" ,"172.17.0.1:9301", "172.17.0.1:9302"]
# 初始主节点,用于选举
cluster.initial_master_nodes: ["es01", "es02", "es03"]

# 集群中自动发现其它节点时ping连接超时时间，默认为3秒，对于比较差的网络环境可以高点的值来防止自动发现时出错
discovery.zen.ping_timeout: 120s

# 设置这个参数来保证集群中的节点可以知道其它N个有master资格的节点
discovery.zen.minimum_master_nodes: 2

# 跨域设置
http.cors.enabled: true
http.cors.allow-origin: "*"
http.cors.allow-methods: OPTIONS, HEAD, GET, POST, PUT, DELETE
http.cors.allow-headers: "X-Requested-With, Content-Type, Content-Length, X-User"

```

- config/es02/elasticsearch.yml
```
# 拷贝es01的配置，只更改以下配置接口
node.name: es02
```
- config/es03/elasticsearch.yml
```
# 拷贝es01的配置，只更改以下配置接口
node.name: es03
```

- 启动服务
```
docker-compose -f docker-compose.yml up -d
```

- 查看集群状态   
现在可以通过curl来查看集群的一些状态，如下：

```
# 查看状态
curl -XGET 'http://localhost:9200/_cluster/health'

# 查看节点
curl -XGET 'http://localhost:9200/_cat/nodes?pretty'

# 查看索引
curl -XGET 'localhost:9200/_cat/indices?v'
```

## 冷热分离
通过设置`node.attr.temperature`为`hot`或者`warm`区分冷热节点。

- 将索引迁移到冷节点：
```
curl -XPUT localhost:9200/index_name/_settings -H "Content-Type: application/json" -d '{"index.routing.allocation.require.temperature":"warm"}'
```

- 将索引恢复到热节点:
```
curl -XPUT localhost:9200/index_name/_settings -H "Content-Type: application/json" -d '{"index.routing.allocation.require.temperature":"hot"}'
```

- 查看索引的状态
```
curl -XGET "localhost:9200/_cat/shards/index_name?v&h=index,shard,prirep,node&s=node"
```