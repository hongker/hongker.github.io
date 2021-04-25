---
title: 服务化03-Etcd的服务注册与发现
date: 2021-04-24 06:48:21
tags: micro-service
---
本文介绍如何通过etcd实现服务注册与发现。

## Etcd
>一个高可用的分布式键值(key-value)数据库。etcd内部采用raft协议作为一致性算法，etcd基于Go语言实现。

## 一致性算法raft
节点的三种角色：
- Leader: 负责接收客户端的请求，将日志复制到其他节点。
- Candidate: 用于选举Leader的一种角色。
- Follower: 普通节点，负责响应Leader和Candidate节点的请求。

## 服务注册与发现
![service](http://processon.com/chart_image/6084cf02637689195242cbff.png?_=1619390448374)


### 服务注册   
A服务启动时，将当前服务运行的`IP`和`Port`提交到Etcd保存，key为`ServiceA`。
### 服务发现   
B服务需要调用A服务的接口时，去Etcd通过`ServiceA`这个key，找到存储的服务A的地址，即可发起调用。
### 服务注销
A服务现在要重启，就要在停止前主动注销。等待重启后再次注册即可。


接下来我们来通过etcd实现：服务注册/服务发现/服务注销
<!--more-->
## 实现
- 首先初始化etcd的客户端:
```go

import (
	 "go.etcd.io/etcd/clientv3"
	"time"
)

var cli *clientv3.Client // etcd客户端
// Connect 连接
func Connect(endpoints []string, timeout time.Duration) (error) {
	instance, err := clientv3.New(clientv3.Config{
		Endpoints:   endpoints,
		DialTimeout: timeout,
	})

	if err != nil {
		return err
	}
	cli = instance
	return nil

}

// KV 键值存储
func KV() clientv3.KV {
	return clientv3.NewKV(cli)
}
// Lease 租约，控制过期时间
func Lease() clientv3.Lease {
	return clientv3.NewLease(cli)
}
```

- 服务组件： 实现服务注册、发现、注销的功能
```go

import (
	"context"
	"encoding/json"
	"fmt"
	"go.etcd.io/etcd/clientv3"
	"log"
)

// Service
type Service struct {
	// 前缀
	prefix string
}

// Node 服务节点
type Node struct {
	Name    string `json:"name"`    // 服务名称,比如:order
	ID      int    `json:"id"`      // 节点ID: 建议从1开始
	Address string `json:"address"` // 服务地址，如：127.0.0.1:8080
	Ttl     int64  // 检测时间，单位:秒。比如设置5秒，那如果节点宕机5秒，这个节点就自动注销了
}

// Key 节点的唯一键名
func (node Node) Key() string {
	return fmt.Sprintf("%s/%d", node.Name, node.ID)
}

// String 节点的存储数据，通过json序列化实现
func (node Node) String() string {
	b, _ := json.Marshal(node)
	return string(b)
}

// withPrefix 拼接前缀
func (service *Service) withPrefix(name string) string {
	return fmt.Sprintf("/%s/%s", service.prefix, name)
}

// Register 服务注册
func (service *Service) Register(node Node) error {
	ctx := context.Background()
	// 申请lease
	leaseResp, err := Lease().Grant(ctx, node.Ttl)
	if err != nil {
		return fmt.Errorf("unable to grant lease: %v", err)
	}

	// 保存数据
	_, err = KV().Put(ctx, service.withPrefix(node.Key()), node.String(), clientv3.WithLease(leaseResp.ID))
	if err != nil {
		return fmt.Errorf("unable to put value: %v", err)
	}

	// 保持连接
	if err := service.keepAlive(leaseResp.ID); err != nil {
		return fmt.Errorf("unable to keep alive: %v", err)
	}
	return nil
}

// Unregister 服务注销
func (service *Service) Unregister(node Node) error {
	if node.Name == "" {
		return fmt.Errorf("unknown service name")
	}

	_, err := KV().Delete(context.TODO(), service.withPrefix(node.Key()))
	if err != nil {
		return err
	}

	return nil
}

// Discovery 服务发现
func (service *Service) Discovery(name string) ([]Node, error) {
	resp, err := KV().Get(context.Background(), service.withPrefix(name), clientv3.WithPrefix())
	if err != nil {
		return nil, err
	}

	nodes := make([]Node, 0, resp.Count)
	if resp.Count == 0 {
		return nodes, nil
	}

	for _, kv := range resp.Kvs {
		var node Node
		if err := json.Unmarshal(kv.Value, &node); err != nil {
			continue
		}
		nodes = append(nodes, node)
	}

	return nodes, err
}

// keepAlive 保持会话
func (service *Service) keepAlive(leaseId clientv3.LeaseID) error {
	keepAliveRes, err := Lease().KeepAlive(context.TODO(), leaseId)
	if err != nil {
		return err
	}

	go func() {
		for {
			select {
			case ret := <-keepAliveRes:
				if ret != nil {
					log.Println("keep alive success")
				}
			}
		}
	}()
	return nil
}

// NewService 实例化
func NewService(prefix string) *Service {
	return &Service{prefix: prefix}
}
```

- 使用
```go
func main() {
	// 连接etcd
	if err := Connect([]string{"127.0.0.1:2379"}, time.Second*10); err != nil {
		panic(err.Error())
	}
	service := NewService("service")

	// 服务注册：注册一个叫order的服务节点
	err := service.Register(Node{
		Name:    "order",
		ID:      1,
		Address: "http://127.0.0.1:8081",
		Ttl:     10,
	})

	// 服务发现：获取到节点信息，返回的是节点数组
	node, err := service.Discovery("order")

	// 服务注销: 程序关闭会自动注销，但是有延迟，如果想要立马注销，就调用Unregister方法
	err = service.Unregister(Node{Name: "order", ID: 1})
}
```

- 优化
服务发现不需要每次都去查询etcd里的数据，可以通过定期查询的方式去获取数据。但是如果某个节点已注销，这样会导致信息更新滞后，请求到已注销的节点上，出现异常。所以我们通过监听的方式来保持信息同步，这样可以做到实时性更新。

```go

// Watcher 监听器
type Watcher struct {
	service *Service // service
	name string // 服务名称
	once sync.Once // once,保证只有第一次需要通过Discovery获取节点信息
	nodes []Node // 缓存
}
// NewWatcher 实例化watcher
func NewWatcher(service *Service, name string) *Watcher {
	return &Watcher{
		service: service,
		name: name,
		nodes:   make([]Node, 10),
	}
}
// 获取节点
func (watcher *Watcher) GetNodes() []Node {
	watcher.once.Do(func() {
		watcher.nodes, _ = watcher.service.Discovery(watcher.name)
	})
	return watcher.nodes
}

// Watch 监听服务节点的注册与注销
func (watcher *Watcher) Watch() {
	rch := cli.Watch(context.Background(), watcher.service.withPrefix(watcher.name), clientv3.WithPrefix())
	for resp := range rch {
		for _, ev := range resp.Events {

			switch ev.Type {
			case mvccpb.PUT: // 新增或更新事件
				// 解析node节点
				var node Node
				if err := json.Unmarshal(ev.Kv.Value, &node); err != nil {
					continue
				}

				if ev.Kv.Version == 1 { // 首次创建,直接append
					watcher.nodes = append(watcher.nodes, node)
					continue
				}
				for idx, n := range watcher.nodes {
					if n.ID == node.ID {
						watcher.nodes[idx].Address = node.Address
						break
					}
				}
			case mvccpb.DELETE: // 删除事件
				nodes := make([]Node, 0, len(watcher.nodes))
				for _, n := range watcher.nodes {
					if string(ev.Kv.Key) != watcher.service.withPrefix(n.Key()) {
						nodes = append(nodes, n)
					}
				}
				watcher.nodes = nodes
			}
		}
	}
}

func main() {
	watcher := NewWatcher(service, "order")
	// 开启协程来负责监听
	go watcher.Watch()
	for { // 模拟每个一秒获取一次,来观察服务节点的注册与注销是否生效
		time.Sleep(time.Second)
		fmt.Println(watcher.GetNodes())
	}
}
```