---
title: 服务化02-mongodb集群
date: 2021-03-22 22:36:03
tags: micro-service
---
本文介绍mongodb集群的相关知识。

## 概念
Replica Set 副本集：一个副本集就是一组 MongoDB 实例组成的集群，由一个主（Primary）服务器和多个备份（Secondary）服务器构成
- 节点（master）：主节点接收所有写入操作。主节点将对其数据集所做的所有更改记录到其 oplog。
- 副节点（secondary）：复制主节点的 oplog 并将操作应用到其数据集，如果主节点不可用，一个合格的副节点将被选为新的主节点。
- 仲裁节点（arbiter）：负载选举，当主节点不可用，它将从副节点中选一个作为主节点。

<!--more-->
![图片](cluster.jpg)

## 目录结构
```
.
├── data    # 数据
│   ├── mongo1
│   ├── mongo2
│   └── mongo3
└── docker-componse.yml
```

## 安装
docker-compose.yml：
```yml
version: '3'
services:
  mongo1:
    container_name: mongo1
    image: mongo:4.4.4
    restart: always
    ports:
      - '27017:27017'
    environment:
      - TZ=Asia/Shanghai
    volumes:
      - $PWD/mongo1/data:/data/db
    command:  mongod --replSet mongo_cluster
  mongo2:
    container_name: mongo2
    image: mongo:4.4.4
    restart: always
    ports:
      - '27018:27017'
    environment:
      - TZ=Asia/Shanghai
    volumes:
      - $PWD/mongo2/data:/data/db
    command:  mongod --replSet mongo_cluster
  mongo3:
    container_name: mongo3
    image: mongo:4.4.4
    restart: always
    ports:
      - '27019:27017'
    environment:
      - TZ=Asia/Shanghai
    volumes:
      - $PWD/mongo3/data:/data/db
    command:  mongod --replSet mongo_cluster
```

启动:
```
docker-compose up -d
```

## 配置
```
docker exec -ti mongo_master bash
mongo

cfg={
  "_id": "mongo_cluster",
  "members":[{
  	_id:0,
  	host:"172.17.0.1:27017",
  	priority: 2
  },{
  	_id:1,
  	host:"172.17.0.1:27018",
  	priority: 1
  },{
  	_id:2,
  	host:"172.17.0.1:27019",
  	arbiterOnly: true
  }]
}
rs.initiate(cfg)
```
最后得到结果:
```json
{
	"ok" : 1,
	"$clusterTime" : {
		"clusterTime" : Timestamp(1616575307, 1),
		"signature" : {
			"hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
			"keyId" : NumberLong(0)
		}
	},
	"operationTime" : Timestamp(1616575307, 1)
}
```

同时也可以通过`rs.status()`查看集群状态