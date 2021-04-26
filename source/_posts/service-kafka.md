---
title: 服务化04-消息队列kafka
date: 2021-04-27 06:50:17
tags:  micro-service
---
本文介绍在微服务架构中，常用于服务与服务之间的异步通信组件kafka。

## kafka
>Kafka是由Apache软件基金会开发的一个开源流处理平台，是一种高吞吐量的分布式发布订阅消息系统。

## 使用场景
- 在系统或应用程序之间构建可靠的用于传输实时数据的管道，消息队列功能
- 构建实时的流数据处理程序来变换或处理数据流，数据处理功能

![image](https://images2017.cnblogs.com/blog/760273/201711/760273-20171108181426763-1692750478.png)

## 通过docker部署kafka
- 首先需要安装zookeeper
```
docker run -d --name zookeeper -p 2181:2181 -v /etc/localtime:/etc/localtime wurstmeister/zookeeper
```

- 然后启动kafka容器
```
 docker run  -d --name kafka -p 9092:9092 \
 -e KAFKA_BROKER_ID=0 \
 -e KAFKA_ZOOKEEPER_CONNECT=172.17.0.1:2181 \
 -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://172.17.0.1:9092 \
  -e KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:9092 \
  -t wurstmeister/kafka
```

## 使用kafka-go
- 安装
```
go get github.com/segmentio/kafka-go
```

- 封装客户端
```go

import (
	"context"
	"fmt"
	"github.com/segmentio/kafka-go"
	"time"
)

type Client struct {
	host string // kafka地址,一般是:localhost:9092
	instance *kafka.Conn // 
}

// Connect 连接kafka服务
func (client *Client) Connect( topic string) error {
	conn, err := kafka.DialLeader(context.Background(), "tcp", client.host, topic, 0)
	if err != nil {
		return fmt.Errorf("failed to dial learder: %v", err)
	}

	_ = conn.SetWriteDeadline(time.Now().Add(10*time.Second))

	client.instance = conn
	return nil
}

// NewClient 实例化
func NewClient(host string) *Client {
	return &Client{host: host}
}
```

<!--more-->
假设现在有个电商系统，分别有订单和仓库两个服务，当客户创建订单成功后，订单服务需要通知仓库发货，那么如下：
- 订单服务(生产者)：
```go
// Producer 生产者
type Producer struct {
	client *Client
}

// Send 发送信息
func (producer Producer) Send() error {
	// 通过kafka发送一条信息，通知发货
	_, err := producer.client.instance.Write([]byte("new order created, please deliver goods"))

	if err != nil {
		return fmt.Errorf("failed to write message: %v", err)
	}

	return nil
}
```

- 仓库服务(生产者)
```go

type Consumer struct {
	client *Client
}
// Receive 接收
func (consumer Consumer) Receive()  {
    // 保持监听
	for {
		message, err := consumer.client.instance.ReadMessage(10e3) 
		if err != nil {
			log.Println("unable to read message:", err)
			return
        }
        // 接收到新消息
        log.Println("receive message:",string( message.Value), message.Offset)
        // 模拟实际业务执行
        time.Sleep(time.Second)
	}

}
```

## 扩展
思考如何基于kafka实现异步事件。