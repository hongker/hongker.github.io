---
title: Golang代码优化08-RPC通信
date: 2020-04-01 14:47:59
tags:
---
本文介绍go里的rpc开发的知识点与相关使用场景。

## 什么是RPC
>RPC（Remote Procedure Call）远程过程调用，简单的理解是一个节点请求另一个节点提供的服务。

## RPC vs REST
- RPC主要是基于TCP/IP协议的，而REST服务主要是基于HTTP协议的
- REST通常以业务为导向，将业务对象上执行的操作映射到HTTP动词，格式非常简单。
- REST也存在一些弊端，比如只支持请求/响应这种单一的通信方式，对象和字符串之间的序列化操作也会影响消息传递速度。在单个请求获取多个资源时存在着挑战，而且有时候很难将所有的动作都映射到HTTP动词。比如注册、授权等等。
- 由于HTTP在应用层中完成，整个通信的代价较高，远程过程调用中直接基于TCP进行远程调用，数据传输在传输层TCP层完成，更适合对效率要求比较高的场景，RPC主要依赖于客户端和服务端之间建立Socket链接进行，底层实现比REST更复杂。

## 主流的RPC框架
主要用的就是GRPC,Thrift,Dubbo这三个，下面分别介绍下使用方式。

### GRPC
GRPC是Google最近公布的开源软件，基于最新的HTTP2.0协议，并支持常见的众多编程语言(Golang,C++,Java,Php,Python等等)。
- 安装
建议先配置go module代理。否则不开VPN下载速度可能会很慢，也许直接下载失败。
```
// 安装protobuf插件
go get -u github.com/golang/protobuf/protoc-gen-go
// 再去下载protoc,地址:https://github.com/golang/protobuf/releases
unzip protoc-x.x.x-linux-x86_64.zip -d /usr/local/
// 查看版本
protoc --version
```

- 示例1 校验用户
目录结构如下：
```
.
├── client      # 客户端
│   └── main.go
├── proto
│   └── user_auth.proto
└── server      # 服务端
    └── main.go

```

<!--more-->

首先需要定义user_auth.proto文件：
```proto
syntax="proto3";

package proto;

// User 定义User服务
service User {
  // 定义用户授权方法
  rpc Auth(AuthRequest) returns (AuthResponse) {}
}

// AuthRequest 授权请求
message AuthRequest {
  // username
  string username = 1;
  // password
  string password = 2;
}
// AuthResponse 授权响应
message AuthResponse {
  // message 提示
  string message = 1;
  // token 授权成功后返回的令牌
  string token = 2;
}
```

通过protoc生成go的编译文件
```
cd proto
protoc user_auth.proto --go_out=plugins=grpc:.
// 执行成功后就能看到新生成的user_auth.pb.go,如果是其他语言，可以基于proto文件生成对应语言的代码。
```

服务端server/main.go:
```go

import (
	"context"
	"fmt"
	"google.golang.org/grpc"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"
	"log"
	"net"
	pb "test/grpc/proto"
)
// UserServer
type UserServer struct {
	pb.UnimplementedUserServer
}

func (u *UserServer) Auth(ctx context.Context, request *pb.AuthRequest) (*pb.AuthResponse, error) {
    resp := new(pb.AuthResponse)
    // 模拟从数据库里获取到数据后的校验
	if request.GetUsername() != "test" {
        // 用户不存在时返回NotFound
		return nil, status.Errorf(codes.NotFound, "the user:%s not found", request.Username)
	}

	if request.GetPassword() != "123456" {
        // 密码错误时返回Unauthenticated
		return nil, status.Error(codes.Unauthenticated, "password is incorrect")
	}
    resp.Message = "success"
    // 成功返回令牌
	resp.Token = "userTokenFromDB"
	return resp, nil
}
func main() {
	listen, err := net.Listen("tcp",":8090")
	if err != nil {
		log.Fatalln(err.Error())
	}

	server := grpc.NewServer()
	pb.RegisterUserServer(server, &UserServer{})
	fmt.Println("grpc server is running..")
	if err := server.Serve(listen); err != nil {
		log.Fatalln(err.Error())
	}

}
```

客户端client/main.go：
```go
package main

import (
	"context"
	"fmt"
	"google.golang.org/grpc"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"
	"log"
	pb "test/grpc/proto"
)

func main() {
	conn , err := grpc.Dial("127.0.0.1:8090", grpc.WithInsecure())
	if err != nil {
		log.Fatalln(err)
	}
    defer conn.Close()
    // new grpc client
    c := pb.NewUserClient(conn)
    
    // 使用超时Context控制请求，如果超过1s，则自动断开
	ctx, cancel := context.WithTimeout(context.Background(), 1e9)
	defer cancel()

    // 调用Auth方法
	resp, err := c.Auth(ctx, &pb.AuthRequest{
		Username:             "test", // 可自行更改触发不同的错误
		Password:             "123456", // 可自行更改触发不同的错误
	})
	if err != nil {
        // 根据不同的错误码，执行相应的逻辑
		if e, ok := status.FromError(err); ok {
			switch e.Code() {
			case codes.NotFound:
				fmt.Println(e.Message())
			case codes.Unauthenticated:
				fmt.Println(e.Message())
			default:
				fmt.Println(e.Message())

			}
		}
		log.Fatalf("login failed: %v", err)
	}
	log.Printf("login success: %s", resp.GetToken())
}
```

运行：

```
// terminal1
go run server/main.go

// terminal2
go run client/main.go

// 正常，输出:
2020/04/01 18:20:13 login success: userTokenFromDB
// 改变password: "123",输出：
login failed: rpc error: code = Unauthenticated desc = password is incorrect
```