---
title: Golang代码优化(29)-TCP编程
date: 2022-06-26 14:53:35
tags: golang
---
本文介绍在golang项目中，基于tcp协议的面向网络编程。

## 通信协议
> 一般来说，基于C/S架构实现的项目，都是基于长链接来实现的。协议有http、tcp、udp,重点介绍tcp。

## 流程说明
- 服务端
```
1.监听端口
2.接收客户端的tcp链接
3.创建goroutine,接收该链接的请求数据，并将相应数据发送给客户端。
```

- 客户端
```
1.建立与服务端的链接
2.发送请求数据，接收服务端返回的响应数据
3.关闭链接
```

- 时序图
![时序图](http://processon.com/chart_image/62b805d2f346fb6dc581ecd2.png)

<!--more-->

## 实现
- 服务端
```go
package network

import (
	"bufio"
	"log"
	"net"
	"runtime"
)

type Server struct {
	opts Options
}

// Options 选项
type Options struct {
	Accept int 		  // 接收tcp的线程数
	SendBuffer    int // 发送数据的缓冲区长度
	ReadBuffer    int // 接收数据的缓冲区长度
}


func NewServer() *Server {
	return &Server{opts: Options{
		Accept:     runtime.NumCPU(),
		SendBuffer: 1024,
		ReadBuffer: 1024,
	}}
}

// Start 启动服务
func (s *Server) Start(bind string)  {
	var (
		listener *net.TCPListener
		addr     *net.TCPAddr
		err error
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
	for i := 0; i < s.opts.Accept; i++ {
		go s.listen(listener)
	}
}

// listen 监听
func (s *Server) listen(lis *net.TCPListener) {
	var (
		conn *net.TCPConn
		err  error
	)

	for {
		// 监听客户端的链接,完成三次握手，得到一个链接对象
		if conn, err = lis.AcceptTCP(); err != nil {
			// if listener close then return
			log.Printf("listener.Accept(%s) error(%v)", lis.Addr().String(), err)
			return
		}
		if err = conn.SetReadBuffer(s.opts.ReadBuffer); err != nil {
			log.Printf("conn.SetReadBuffer() error(%v)", err)
			return
		}
		if err = conn.SetWriteBuffer(s.opts.ReadBuffer); err != nil {
			log.Printf("conn.SetWriteBuffer() error(%v)", err)
			return
		}

		log.Printf("client new request,ip: %v", conn.RemoteAddr())

		// 一个goroutine处理一个连接
		go s.handle(conn)

	}
}


func  (s *Server) handle(conn *net.TCPConn)  {
	log.Printf("start handle client:%s", conn.RemoteAddr().String())
	defer conn.Close()  // 关闭链接

	reader := bufio.NewReader(conn)
	for {
		// 用一个4k的数组来接收数据
		var buf [4096]byte
		n, err := reader.Read(buf[:])  // 读取数据
		if err != nil {
			break
		}
		msg := buf[:n]

		// 将接收的数据作为响应返回给客户端
		conn.Write(msg)
	}
}

```
- 客户端
```go
package network

import (
	"log"
	"net"
)

type Client struct {
	conn net.Conn
}

func NewClient() *Client {
	return &Client{}
}

func (client *Client) Connect(host string) (err error) {
	// 建立与服务器的链接
	client.conn, err = net.Dial("tcp", host)
	return
}

func (client *Client) Send(msg []byte) (err error) {
	_, err = client.conn.Write(msg)
	return
}

func (client *Client) Receive() (err error){
	buf := [4096]byte{}
	n, err := client.conn.Read(buf[:])
	if err != nil {
		return
	}
	log.Println("receive from server:", string(buf[:n]))
	return
}

```

- 运行
```go
package network

import "time"

func RunServer()  {
	server := NewServer()
	server.Start(":9000")
	select{}
}

func RunClient()  {
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
		client.Send([]byte("hello"))
		time.Sleep(time.Second)
	}
}
```

## 关键问题
在网络编程中会遇到以下几个问题，简单介绍如下。
- 粘包
tcp数据传递模式是流模式，在保持长连接的时候可以进行多次的收和发。
>发送端：当我们提交一段数据给TCP发送时，TCP并不立刻发送此段数据，而是等待一小段时间看看在等待期间是否还有要发送的数据，若有则会一次把这两段数据发送出去。
>接收端：TCP会把接收到的数据存在自己的缓冲区中，然后通知应用层取数据。当应用层由于某些原因不能及时的把TCP的数据取出来，就会造成TCP缓冲区中存放了几段数据。

引起粘包的问题就是接收方不确定要传输的数据包的大小，所以解决办法就是通过标识位来存储一段数据包的大小。协议设计如下：

```
--------------
| 包头 | 包体 |
--------------
| 4 位 |  n位 |
--------------

包头存储包的总长度，也就是4+n
接收端读取数据先读取4位包头，得到包体总长度后，再读取n位数据作为包体，这样就是一个完整的包
```

- 心跳
所有的客户端都使用手机移动网络并且网络总是不稳定。经常丢失连接却没有通过FIN或者RST包通知服务端。服务端保持着这个虚连接并且认为这个客户端仍然在线，而事实上却不是。
所以我们需要心跳检测来确认客户端的链接是否正常。

- 带宽
当存在超高链接数且数据交互量非常大时，带宽就成了我们必须考虑的问题。减少带宽的占用一方面能节省服务器成本，还可以降低服务器的IO与负载，进而提高服务端的吞吐率与低延时性。

后续会针对每个问题给出详细的实现方案。
