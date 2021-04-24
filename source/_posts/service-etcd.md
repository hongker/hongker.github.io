---
title: 服务化03-Etcd的服务注册与发现
date: 2021-04-24 06:48:21
tags: micro-service
---
本文介绍如何通过etcd实现服务注册与发现。

## Etcd
>一个高可用的分布式键值(key-value)数据库。etcd内部采用raft协议作为一致性算法，etcd基于Go语言实现。

## 服务注册与发现
![service](https://upload-images.jianshu.io/upload_images/5397496-d2afe8bb1544a0bf.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/994/format/webp)


### 服务注册   
A服务启动时，将当前服务运行的`IP`和`Port`提交到Etcd保存，key为`ServiceA`。
### 服务发现   
B服务需要调用A服务的接口时，去Etcd通过`ServiceA`这个key，找到存储的服务A的地址，即可发起调用。

## 实现
- 首先初始化etcd的客户端:
```go
import (
	client "go.etcd.io/etcd/clientv3"
	"time"
)

var cli *client.Client
// Connect 连接etcd
func Connect(endpoints []string, timeout time.Duration) (error) {
	instance, err := client.New(client.Config{
		Endpoints:   endpoints,
		DialTimeout: timeout,
	})

	if err != nil {
		return err
	}
	cli = instance
	return nil
}
func UseClient() client.KV {
	return client.NewKV(cli)
}
```