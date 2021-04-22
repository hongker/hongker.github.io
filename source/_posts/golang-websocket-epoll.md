---
title: Golang(27)-websocket进阶
date: 2021-04-22 21:14:05
tags: golang
---
之前有简单的介绍过`websocket`的基本用法，本文将介绍在golang里如何处理大量websocket连接。

## 初阶
现在有一个即时通讯的需求，我们实现一个简单的websocket服务如下所示：
```go
package main
import (
	"fmt"
	"github.com/gorilla/websocket"
	"log"
	"net/http"
)
func main() {
	http.ListenAndServe(":8085", http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		// 升级为websocket连接
		conn, err := NewWebsocketConnection(w, r)
		if err != nil {
			http.NotFound(w, r)
			return
		}
		go func() { // 开启一个协程去处理连接
			defer conn.Close() // 连接结束时需要关闭
			for {
				_, msg, err := conn.ReadMessage() // 接受消息
				if err != nil {
					log.Println("unable to read message:", err.Error())
					return
				}
				// 向客户端回话,或者其他业务逻辑
				reply := fmt.Sprintf("received:%s", string(msg))
				if err := conn.WriteMessage(websocket.TextMessage, []byte(reply)); err != nil {
					log.Printf("unable to send message")
				}
			}
		}()
	}))
}
var u = websocket.Upgrader{CheckOrigin: func(r *http.Request) bool { return true }} // use default options
// WebsocketConn return web socket connection
func NewWebsocketConnection(w http.ResponseWriter, r *http.Request) (*websocket.Conn, error) {
	return u.Upgrade(w, r, nil)

}
```

通过`wscat -c localhost:8085`连接到websocket,并发送`hello`和`123`,得到测试结果如下：
```

> hello
< received:hello
> 123
< received:123
```

## 思考
>如果我们的服务需要面向100w个用户时，会发生什么情况？
```
1.每创建一个websocket连接，按照以上的实现方式，我们就需要创建一个goroutine来接收客户端的信息。一个goroutine大概需要2~8kb的内存
2.如果是同时有100万个连接，假设每个goroutine占用4kb内存，那么内存消耗大概在：4kb*1000000=4G。
```

光是保持连接，不做任何处理就已经消耗了4G的内存，还是挺恐怖的，所以下面开始介绍用epoll模型来解决这个问题。
<!--more-->

## Epoll
>epoll是Linux内核为处理大批量文件描述符而作了改进的poll，是Linux下多路复用IO接口select/poll的增强版本，它能显著提高程序在大量并发连接中只有少量活跃的情况下的系统CPU利用率。

Epoll提供了3个方法：Create、Ctl、Wait
- Create: 创建epoll句柄，返回文件标识符(fd)。
- Ctl: 根据epoll的fd，完成注册事件、删除事件、更新事件。
- Wait: 返回就绪事件。

我们可以通过epoll模型，来管理websocket连接，用来替代通过goroutine去监听的方案。

实现如下：