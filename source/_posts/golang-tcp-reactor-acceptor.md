---
title: Golang系列(32)-通过reactor模型监听连接
date: 2022-11-16 18:38:10
tags: golang
---

上一篇文章介绍了reactor模型，本文介绍如何通过reactor模型监听TCP连接


更多请参考：我的自研网络框架 [znet](https://github.com/ebar-go/znet)，欢迎Star与提Issue。

## Reactor实现方案
- 方案一：单进程/线程，启动单个Reactor，单个Acceptor。不用考虑进程间通信以及数据同步的问题,无法充分利用多核CPU。处理业务逻辑的时间不能太长，否则会延迟响应，所以不适用于计算机密集型的场景。
- 方案二：单进程/多线程，启动单个Reactor,多个Acceptor。解决方案一的问题无法利用多核CPU的问题。但是它依然只有一个主线程处理业务，无法解决瞬时高并发带来的性能问题。
- 方案三：多进程/多线程，启动多个Reactor(一个MainReactor+多个SubReactor)，多个Acceptor。主 Reactor 只负责监听事件，响应事件的工作交给了从 Reactor。

## 模块设计
- Acceptor: 负责与客户端建立链接，并将连接发送给Reactor
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