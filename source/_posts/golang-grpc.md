---
title: Golang系列(37)-grpc的负载均衡
date: 2022-12-03 19:52:26
tags: golang
---

本文介绍如何实现grpc服务的负载均衡

## 什么是grpc
>gRPC 是一个高性能、开源、通用的 RPC 框架，面向移动和 HTTP/2 设计，是由谷歌发布的首款基于 Protocol Buffers 的 RPC 框架。 gRPC 基于 HTTP/2 标准设计，带来诸如双向流、流控、头部压缩、单 TCP 连接上的多复用请求等特性。

## 问题
由于grpc底层是通过保持长连接来实现链路复用，避免重复创建连接导致的开销。但是当服务端存在多个实例时，会导致新的请求始终指到某一个实例。

为此我们需要一些特殊的手段来实现负载均衡。



## 实现方式
我们可以通过客户端负载均衡以及服务端负载均衡两种方式来实现，对比如下：

类型 | 优点 | 缺点
---|---|---
客户端| 性能好，不会有转发请求的消耗。链路短，利于定位问题 | 侵入式开发，如果是多种语言，需要分别开发客户端 
服务端| 跨语言，避免多端的重复开发与不一致问题，非浸入式，易统一监控 | 链路长，有性能损耗，难于排查问题

具体介绍如下：
<!--more-->
### 客户端负载均衡
服务端启动时，将本机IP注册到注册中心(如K8s的headless,Etcd,Consul等)，同时开启健康检查，与注册中心保持心跳检测以表明服务的存活状态。如果服务停止，则需要主动注销保证及时性。

客户端需要通过服务发现获取对应服务的注册表，缓存服务的地址列表。客户端通过与这些地址建立grpc子通道，根据负载均衡算法，客户端在请求时会选择启用一个子通道发起请求。同时客户端需要监听注册表的变动以及时做出刷新操作。

- Demo
以下实现以k8s的headless为例：

服务端代码：
```go
package main

import (
	"context"
	"fmt"
	"github.com/ebar-go/ego"
	"github.com/ebar-go/ego/component"
	"github.com/ebar-go/ego/examples/grpc/pb"
	"github.com/ebar-go/ego/utils/runtime"
	"github.com/gin-gonic/gin"
	"google.golang.org/grpc"
	"google.golang.org/grpc/balancer/roundrobin"
	"google.golang.org/grpc/credentials/insecure"
	"os"
	"strconv"
)
// 启动一个grpc服务
func main() {
	aggregator := ego.New()

	aggregator.Aggregate(grpcServer())

	aggregator.Run()
}

type UserService struct {
	pb.UnimplementedUserServiceServer
}

func (srv UserService) Greet(ctx context.Context, in *pb.GreetRequest) (*pb.GreetResponse, error) {
	hostname, _ := os.Hostname()
	return &pb.GreetResponse{Name: fmt.Sprintf("hostname=%s, in=%s, out=%s", hostname, in.Name, "bar")}, nil
}

func grpcServer() runtime.Runnable {
	return ego.NewGRPCServer(":8081").RegisterService(func(s *grpc.Server) {
		pb.RegisterUserServiceServer(s, new(UserService))
	})
}
```

启动一个http服务当作客户端向服务端发送grpc请求：
```go
// 启动http服务
func main() {
	aggregator := ego.New()

	aggregator.Aggregate(httpServer())

	aggregator.Run()
}

const (
    // 如果这里只是grpc-service:8081,将无法触发负载均衡
	target = "dns:///grpc-service:8081" // 表示以dns的方式与k8s的headless service建立连接
)

func httpServer() runtime.Runnable {
	cc, err := grpc.Dial(target,
		grpc.WithTransportCredentials(insecure.NewCredentials()),
		grpc.WithDefaultServiceConfig(fmt.Sprintf(`{"LoadBalancingPolicy": "%s"}`, roundrobin.Name)), // 采用轮询的负载均衡方式
	)
	client := pb.NewUserServiceClient(cc)

	if err != nil {
		panic(err)
	}

	return ego.NewHTTPServer(":8080").
		EnableAvailableHealthCheck().
		EnablePprofHandler().
		WithDefaultRequestLogMiddleware().
		RegisterRouteLoader(func(router *gin.Engine) {
			router.GET("greeter", func(ctx *gin.Context) {

				num, _ := strconv.Atoi(ctx.Query("num"))
				if num == 0 {
					num = 10
				}

				for i := 0; i < num; i++ {
					resp, err := client.Greet(ctx, &pb.GreetRequest{Name: "foo"})
					if err != nil {
						component.Logger().Errorf("Greet: %v", err)
						return
					}
					component.Logger().Info(resp.Name)
				}

				//ctx.String(http.StatusOK, resp.Name)
			})
		})
}
```

当客户端与服务端都准备好之后，通过k8s分别部署http和grpc的deployment，http服务部署一个pod,grpc服务部署三个pod。

部署好之后，访问http服务的接口：`127.0.0.1:8080/greeter`,可以观察http的pod日志，最终结果如下：
```
2022/12/03 14:01:05 info hostname=grpc-demo-5d96c9b775-zk4vm, in=foo, out=bar
2022/12/03 14:01:05 info hostname=grpc-demo-5d96c9b775-nsvvs, in=foo, out=bar
2022/12/03 14:01:05 info hostname=grpc-demo-5d96c9b775-rht7p, in=foo, out=bar
2022/12/03 14:01:05 info hostname=grpc-demo-5d96c9b775-zk4vm, in=foo, out=bar
2022/12/03 14:01:05 info hostname=grpc-demo-5d96c9b775-nsvvs, in=foo, out=bar
2022/12/03 14:01:05 info hostname=grpc-demo-5d96c9b775-rht7p, in=foo, out=bar
2022/12/03 14:01:05 info hostname=grpc-demo-5d96c9b775-zk4vm, in=foo, out=bar
2022/12/03 14:01:05 info hostname=grpc-demo-5d96c9b775-nsvvs, in=foo, out=bar
2022/12/03 14:01:05 info hostname=grpc-demo-5d96c9b775-rht7p, in=foo, out=bar
2022/12/03 14:01:05 info hostname=grpc-demo-5d96c9b775-zk4vm, in=foo, out=bar
```
说明负载均衡算法已生效。

经过测试，`当服务端的pod数量缩减时，客户端会立即做出变更操作;当服务端的pod数量增加时，客户端大概在30秒左右做出变更操作`




### 服务端负载均衡
在服务端与客户端之间提供一个代理中间件，负责与客户端建立连接，同时将请求以负载均衡的策略转发给服务的多个实例。可以利用服务网格实现。参考istio官方文档：[文档地址](https://istio.io/latest/zh/docs/)

后面会细聊如何利用istio对服务做无侵入的负载均衡。