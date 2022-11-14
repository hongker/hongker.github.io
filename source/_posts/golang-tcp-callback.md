---
title: Golang系列(31)-TCP回调函数
date: 2022-11-14 19:42:27
tags: golang
---
本文介绍如何给tcp连接增加回调函数，和websocket一样，提供三个回调函数:OnOpen,OnClose,OnMessage。


更多请参考：我的自研网络框架 [znet](https://github.com/ebar-go/znet)，欢迎Star与提Issue。

## 回调函数
- OnOpen: 连接建立成功后触发的回调函数。
- OnClose: 连接断开后触发的回调函数。
- OnMessage: 收到新消息时的回调函数。

## 实现
前面已经提到如何启动一个TCP服务，具体请点击：[TCP编程](http://127.0.0.1:4000/2022/06/26/golang-tcp/)。我们在此基础上去添加回调函数。

- Callback
```go
package network

import "net"

type ConnectionHandler func(conn *net.TCPConn)

// Callback manage connection callback handlers.
type Callback struct {
	open    ConnectionHandler
	close   ConnectionHandler
	request func(conn *net.TCPConn, msg []byte)
}

// triggerOpenEvent is called when the connection is established
func (callback *Callback) triggerOpenEvent(conn *net.TCPConn) {
	if callback.open != nil {
		callback.open(conn)
	}
}

// triggerCloseEvent is called when the connection is closed
func (callback *Callback) triggerCloseEvent(conn *net.TCPConn) {
	if callback.close != nil {
		callback.close(conn)
	}
}

// triggerRequestEvent is called when receive new message
func (callback *Callback) triggerRequestEvent(conn *net.TCPConn, msg []byte) {
	if callback.request != nil {
		callback.request(conn, msg)
	}
}

```

<!--more-->

- Server
```go
package network

import (
	"bufio"
	"log"
	"net"
	"runtime"
)

type Server struct {
	options  Options
	callback *Callback
}

// Options 选项
type Options struct {
	Accept     int // 接收tcp的线程数
	SendBuffer int // 发送数据的缓冲区长度
	ReadBuffer int // 接收数据的缓冲区长度
	OnOpen     ConnectionHandler
	OnClose    ConnectionHandler
	OnMessage  func(conn *net.TCPConn, msg []byte)
}

type Option func(options *Options)

func NewServer(opts ...Option) *Server {
	options := Options{
		Accept:     runtime.NumCPU(),
		SendBuffer: 1024,
		ReadBuffer: 1024,
	}

	for _, setter := range opts {
		setter(&options)
	}

	callback := &Callback{
		open:    options.OnOpen,
		close:   options.OnClose,
		request: options.OnMessage,
	}

	return &Server{options: options, callback: callback}
}

// Start 启动服务
func (s *Server) Start(bind string) {
	var (
		listener *net.TCPListener
		addr     *net.TCPAddr
		err      error
	)
	// 解析绑定地址
	if addr, err = net.ResolveTCPAddr("tcp", bind); err != nil {
		log.Printf("net.ResolveTCPAddr(tcp, %s) error(%v)", bind, err)
		return
	}
	// 绑定服务
	if listener, err = net.ListenTCP("tcp", addr); err != nil {
		log.Printf("net.ListenTCP(tcp, %s) error(%v)", bind, err)
		return
	}

	log.Printf("start tcp listen: %s", bind)

	// 利用多核的优势去处理链接的创建
	for i := 0; i < s.options.Accept; i++ {
		go s.listen(listener)
	}
}

// listen 监听
func (s *Server) listen(lis *net.TCPListener) {
	var (
		err error
	)

	for {
		var conn *net.TCPConn

		// 监听客户端的链接,完成三次握手，得到一个链接对象
		if conn, err = lis.AcceptTCP(); err != nil {
			// if listener close then return
			log.Printf("listener.Accept(%s) error(%v)", lis.Addr().String(), err)
			return
		}
		if err = conn.SetReadBuffer(s.options.ReadBuffer); err != nil {
			log.Printf("conn.SetReadBuffer() error(%v)", err)
			return
		}
		if err = conn.SetWriteBuffer(s.options.ReadBuffer); err != nil {
			log.Printf("conn.SetWriteBuffer() error(%v)", err)
			return
		}

		log.Printf("client new request,ip: %v", conn.RemoteAddr())

		// 一个goroutine处理一个连接
		go s.handle(conn)

	}
}

func (s *Server) handle(conn *net.TCPConn) {
	log.Printf("start handle client:%s", conn.RemoteAddr().String())
	defer conn.Close() // 关闭链接

	// 触发open回调
	s.callback.triggerOpenEvent(conn)

	reader := bufio.NewReader(conn)
	for {
		// 用一个4k的数组来接收数据
		var buf [4096]byte
		n, err := reader.Read(buf[:]) // 读取数据
		if err != nil {
			break
		}
		msg := buf[:n]

		// 触发消息回调
		s.callback.triggerRequestEvent(conn, msg)
	}

	// 触发close回调
	s.callback.triggerCloseEvent(conn)
}

```

- 示例
```go
// RunServer 启动服务端
func RunServer() {
	server := NewServer(func(options *Options) {
		options.OnOpen = func(conn *net.TCPConn) {
			log.Println("welcome")
		}
		options.OnClose = func(conn *net.TCPConn) {
			log.Println("goodbye")
		}
		options.OnMessage = func(conn *net.TCPConn, msg []byte) {
			log.Println("received message: ", string(msg))
			conn.Write([]byte("bar"))
		}
	})
	server.Start(":9000")
	select {}
}

// RunClient 启动客户端
func RunClient() {
	client := NewClient()
	if err := client.Connect("127.0.0.1:9000"); err != nil {
		panic(err)
	}

	go func() {
		for {
			client.Receive()
		}
	}()
	for {
		client.Send([]byte("foo"))
		time.Sleep(time.Second)
	}
}
```