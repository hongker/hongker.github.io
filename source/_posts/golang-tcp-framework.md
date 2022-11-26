---
title: Golang系列(33)-高性能Socket框架的设计
date: 2022-11-26 14:46:08
tags: golang
---
前面介绍了Epoll,Reactor等等，本文主要是介绍高性能Socket框架的设计思路以及各个模块的调用时序。

更多请参考：我的自研网络框架 [znet](https://github.com/ebar-go/znet)，欢迎Star与提Issue。

## 整体设计
想要达到高性能的目标，首先必须在整体方面有良好的设计。

### 模块设计
![Framework](http://assets.processon.com/chart_image/62b3d00e637689074ac74fb3.png?1)
- Network：Socket服务的总控，负责初始化和管理各个子模块。
- Acceptor: 连接接收模块，负责与客户端建立连接。
- Reactor: 事件调度主模块，负责监听活跃连接以及注册回调事件(OnOpen/OnClose/OnMessage/OnError)。
- SubReactor: 事件调度子模块，负责管理连接，以及执行新消息回调事件。
- Thread: 多线程事件处理模块，负责处理客户端请求，包括读取、解包、处理逻辑、打包、发送数据等操作。
- Connection: 客户端连接抽象对象，同时支持TCP/WebSocket协议的连接。
- Context: 请求上下文对象，负责携带客户端请求数据。
- Engine: 请求处理引擎，负责执行Context。采用责任链的设计模式，提供注入中间件的使用方式。
- Router：路由模块，细化请求处理回调事件，允许按照Action来注入处理客户端请求的Handler。


## 细节优化
除了整体拥有良好的设计之外，还需要再细节上有充分的优化。
- 利用多核特性，提高与客户端建立连接的性能。
- 利用对象池的设计，提供Context对象的初始化与回收，降低GC压力。
- 利用对象池的设计，提供读取客户端请求数据的字节数组，避免重复分配空间。
- 利用分片结构的设计，减少锁的竞争，提高Connection获取效率。