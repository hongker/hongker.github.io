---
title: Golang系列(32)-通过reactor模型监听连接
date: 2022-11-16 18:38:10
tags: golang
---

上一篇文章介绍了reactor模型，本文介绍如何通过reactor模型监听TCP连接


更多请参考：我的自研网络框架 [znet](https://github.com/ebar-go/znet)，欢迎Star与提Issue。

## 模块设计
- Acceptor: 负责与客户端建立链接
- Reactor: 通过Epoll负责注册连接与监听活跃连接
- SubReactor: 负责管理连接,可支持分片特性，提高吞吐
- Thread: 负责处理数据包的接收、解包、打包、发送,可支持多线程并发处理

## 流程设计
参考网图：
![flow](https://picx.zhimg.com/80/v2-4da008d8b7f55a0c18bef0e87c5c5bb1_720w.webp?source=1940ef5c)

<!--more-->

## 接口设计
```go
type Acceptor interface{
    Schema() Schema
	// Listen 启动监听函数
	Listen(onAccept func(conn net.Conn)) error

	// Shutdown 关闭
	Shutdown()
}

type Reactor interface {
    // Run 启动epoll监听活跃连接
    Run(stopCh <-chan struct{}, onRequest ConnectionHandler)

    // InitConnection 注册连接,也是acceptor的Listen函数的回调参数
    InitConnection(conn net.Conn) 
}

type SubReactor interface {
    // 注册连接
    RegisterConnection(conn *Connection)
    // 注销连接
	UnregisterConnection(conn *Connection)
    // 根据文件描述符获取连接
	GetConnection(fd int) *Connection
    // 提供活跃连接的句柄
	Offer(fds ...int)
    // 循环执行处理活跃连接的回调函数
	Polling(stopCh <-chan struct{}, callback func(int))
}

type Thread interface {
    // 处理请求：接收数据->解包->处理逻辑->打包->发送数据
    HandleRequest(conn *Connection)
}
```