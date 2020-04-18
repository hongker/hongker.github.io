---
title: 微服务02-服务注册与发现
date: 2020-04-12 19:53:45
tags: 微服务
---
本文介绍微服务核心模块，服务注册与发现。

## 什么是服务注册
>将服务运行的IP与端口发送到服务中心注册，注册中心将运行服务的节点信息记录。

## 什么是服务发现
>当需要调用某个服务提供的接口时，去注册中心，获取到对应服务的节点信息，发起请求调用服务。

## 注册中心
常用的服务注册中心有Etcd,Consul,Zookeeper等。下面分别介绍

## 目录结构
```
soa
├─consul
│      consul.go
│
├─etcd
│      etcd.go
│
└─service
        service.go
```

### Serivce
公共的服务定义
```go
package service

import (
	"math/rand"
	"net"
	"strconv"
	"time"
)

// Node 节点
type Node struct {
	// 服务ID
	ID string
	// 服务名称
	Name string
	// 服务地址
	Address string
	// 服务端口
	Port int
	// 服务标签
	Tags []string
}

// GetHost 获取服务完整的地址
func (Node Node) GetHost() string {
	return net.JoinHostPort(Node.Address, strconv.Itoa(Node.Port))
}

// Group
type Group struct {
	items  []Node
	cursor int
}

// Count return node count
func (group *Group) Count() int {
	return len(group.items)
}

// First return fist node
func (group *Group) First() Node {
	return group.items[0]
}

// Rand return a rand node
func (group *Group) Rand() Node {
	rand.Seed(time.Now().Unix()) // initialize global pseudo random generator

	return group.items[rand.Intn(group.Count())]
}

// Next return the next node
func (group *Group) Next() Node {
	if group.cursor == group.Count() {
		group.cursor = 0
	}

	Node := group.items[group.cursor]
	group.cursor++
	return Node
}

// Add add node
func (group *Group) Add(node Node)  {
	group.items = append(group.items, node)
}

```

<!--more-->

### Etcd
ETCD是一个高可用的分布式键值数据库，可用于共享配置、服务的注册和发现。ETCD采用Raft一致性算法，基于Go语言实现。ETCD作为后起之秀，又非常大的优势。

- 部署
通过docker 部署
```
// 启动一个服务
docker run -d -p 2379:2379 -p 2380:2380 --name etcd-server quay.io/coreos/etcd:v3.3.20

// 进入服务内部
docker exec -ti etcd-server /bin/sh

// 将一个为hello的key，设置为world
etcdctl set hello world

// 获取
etcdctl get hello
```

安装官方包
```
go get github.com/coreos/etcd/client
```

- Demo
```go
package etcd

import (
	"context"
	"fmt"
	"github.com/coreos/etcd/client"
	"github.com/ebar-go/ego/component/soa/service"
	"net"
	"strconv"
)

var instance client.Client

// InitClient 初始化Client
func InitClient(config client.Config) error   {
	var err error
	instance , err = client.New(config)
	return err
}

// Register 服务注册
func Register(node service.Node) error {
	kapi := client.NewKeysAPI(instance)

	key := node.Name + "/" + node.ID
	val := net.JoinHostPort(node.Address, strconv.Itoa(node.Port))
	resp, err := kapi.Set(context.Background(), key, val, nil)

	fmt.Println(resp)
	return err
}

// Deregister 服务注销
func Deregister(node service.Node) error {
	kapi := client.NewKeysAPI(instance)

	key := node.Name + "/" + node.ID
	resp, err := kapi.Delete(context.Background(), key, nil)

	fmt.Println(resp)
	return err
}

// Discover 服务发现
func Discover(name string) error {
	kapi := client.NewKeysAPIWithPrefix(instance, name)
	resp, err := kapi.Get(context.Background(), "*", nil)
    fmt.Println(resp)
    
    // TODO 待完成，因为我用docker拉取3.3.13版本以上的镜像，死都拉不下来。。特么的
	return err
}

```
### Consul
Consul是一个高可用的分布式服务注册中心，由HashiCorp公司推出，Golang实现的开源共享的服务工具。Consul在分布式服务注册与发现方面有自己的特色，解决方案更加“一站式”，不再需要依赖其他工具。

- 部署
通过docker部署
```
docker run -d -p 8500:8500 --name consul-server consul agent -server -bootstrap -ui -client='0.0.0.0'
```

安装扩展包
```
go get github.com/hashicorp/consul/api
```

- Demo
```go
/**
集成consul组件，包含实例化consul客户端,服务发现,服务注册,服务注销,负载均衡等方法
*/
package consul

import (
	"fmt"
	"github.com/ebar-go/ego/component/soa/service"
	consulapi "github.com/hashicorp/consul/api"
)


var client *consulapi.Client

// InitClient 初始化consul客户端
func InitClient(config *consulapi.Config) error {
	var err error
	client, err =  consulapi.NewClient(config)

	return err
}

// DefaultConfig 默认配置
func DefaultConfig() *consulapi.Config {
	return consulapi.DefaultConfig()
}

// Discover 服务发现
func Discover(name string) (*service.Group, error) {
	services, _, err := client.Health().Service(name, "", true, &consulapi.QueryOptions{})
	if err != nil {
		return nil, fmt.Errorf("service: %s not found,%s", name, err.Error())
	}

	if len(services) == 0 {
		return nil, fmt.Errorf("service name : %s not found", name)
	}

	group := new(service.Group)
	for _, item := range services {
		group.Add(service.Node{
			ID:      item.Service.ID,
			Name:    item.Service.Service,
			Address: item.Service.Address,
			Port:    item.Service.Port,
			Tags:    item.Service.Tags,
		})
	}

	return group, nil
}


// Register 注册服务
func Register(node service.Node) error {
	registration := new(consulapi.AgentServiceRegistration)
	registration.ID = node.ID
	registration.Name = node.Name
	registration.Port = node.Port
	registration.Tags = node.Tags
	registration.Address = node.Address

	check := new(consulapi.AgentServiceCheck)
	check.HTTP = fmt.Sprintf("http://%s:%d%s", registration.Address, registration.Port, "/check")
	check.Timeout = "3s"
	check.Interval = "3s"
	check.DeregisterCriticalServiceAfter = "30s" //check失败后30秒删除本服务
	registration.Check = check

	return client.Agent().ServiceRegister(registration)
}

// Deregister 注销服务
func Deregister(node service.Node) error {
	return client.Agent().ServiceDeregister(node.ID)
}
```