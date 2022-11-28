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
- Thread: 多线程事件处理模块，利用协程池并发处理客户端请求，包括读取、解包、处理逻辑、打包、发送数据等操作。
- Connection: 客户端连接抽象对象，同时支持TCP/WebSocket协议的连接。
- Context: 请求上下文对象，负责携带客户端请求数据。
- Engine: 请求处理引擎，负责执行Context。采用责任链的设计模式，提供注入中间件的使用方式。
- Router：路由模块，细化请求处理回调事件，允许按照Action来注入处理客户端请求的Handler。

<!--more-->

## 细节优化
除了整体拥有良好的设计之外，还需要再细节上有充分的优化。
- 利用多核特性，提高与客户端建立连接的性能。
```go
type Options struct {
    Core int
    // ...
}
func DefaultOptions() Options {
	return Options{
		Core: runtime.NumCPU(),
    }
}
type TCPAcceptor struct{
    options Options
}

func (acceptor *TCPAcceptor) Listen(onAccept func(conn net.Conn)) (err error) {
	// ...

	// use multiple cpus to improve performance
	for i := 0; i < acceptor.options.Core; i++ {
		go func() {
			defer runtime.HandleCrash()
			acceptor.accept(lis, onAccept)
		}()
	}

	return
}
// accept connection
func (acceptor *TCPAcceptor) accept(lis *net.TCPListener, onAccept func(conn net.Conn)) {
	for {
		select {
		case <-acceptor.done:
			return
		default:
			conn, err := lis.AcceptTCP()
			if err != nil {
				// if listener close then return
				log.Printf("listener.Accept(\"%s\") error(%v)", lis.Addr().String(), err)
				continue
			}
			// ...

			onAccept(codec.NewLengthFieldBasedFromDecoder(conn, acceptor.options.LengthOffset))
		}
	}

}
```
- 利用对象池的设计，提供Context对象的初始化与回收，降低GC压力。
```go
// Engine provide context/handler management
type Engine struct {
    // ...
	contextProvider pool.Provider[*Context] // is a pool for Context
}

func NewEngine() *Engine {
	e := &Engine{}
	e.contextProvider = pool.NewSyncPoolProvider[*Context](func() interface{} {
		return &Context{engine: e}
	})
	return e
}

// compute run invoke function with context
func (e *Engine) compute(conn *Connection, packet *codec.Packet) {
	// acquire context from provider
	ctx := e.contextProvider.Acquire()
	ctx.reset(conn, packet)
	defer e.contextProvider.Release(ctx)

	e.invoke(ctx, 0)
}

```
- 利用对象池的设计，提供读取客户端请求数据的字节数组，避免重复分配空间。
```go

// HandleRequest handle new request for connection
func (thread *Thread) HandleRequest(conn *Connection) {
	// read message from connection
	var (
		n      = 0
		bytes  = pool.GetByte(thread.options.MaxReadBufferSize)
		packet = codec.NewPacket(thread.codec)
	)

	err := runtime.Call(func() (lastErr error) {
		n, lastErr = conn.Read(bytes)
		return
	}, func() error {
		return packet.Unpack(bytes[:n])
	})

	if err != nil {
		log.Printf("[%s] read failed: %v\n", conn.ID(), err)
		// put back immediately when decode failed
		pool.PutByte(bytes)
		conn.Close()
		return
	}

	// compute
	thread.worker.Schedule(func() {
		defer runtime.HandleCrash()
		defer pool.PutByte(bytes)

		thread.engine.compute(conn, packet)
	})
}
```
- 利用分片结构的设计，减少锁的竞争，提高Connection获取效率。
```go
type ShardSubReactor struct {
	container structure.Sharding[*SingleSubReactor]
}

func (shard *ShardSubReactor) RegisterConnection(conn *Connection) {
	shard.container.GetShard(conn.fd).RegisterConnection(conn)
}

func (shard *ShardSubReactor) GetConnection(fd int) *Connection {
	return shard.container.GetShard(fd).GetConnection(fd)
}

```

- 利用自动伸缩的协程池设计，提高系统的并发处理能力。允许空闲时自动缩小协程数量，高并发时自动增加协程数量。