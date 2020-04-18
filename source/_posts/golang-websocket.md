---
title: Golang代码优化16-Websocket
date: 2020-04-17 13:49:39
tags: golang
---

本文介绍golang里使用websocket实现即时通讯。

## 什么是websocket
>WebSocket使得客户端和服务器之间的数据交换变得更加简单，允许服务端主动向客户端推送数据。在WebSocket API中，浏览器和服务器只需要完成一次握手，两者之间就直接可以创建持久性的连接，并进行双向数据传输。

## 安装包
这是我封装好的包
```go
go get github.com/ebar-go/ws
```

## 简单示例
基于gin启动websocket服务
```go
package main
import (
    "github.com/ebar-go/ws"
    "github.com/gin-gonic/gin"
    "net/http"
)

var m ws.Manager
func init() {
    m = ws.New()
}

func main() {
	// use gin router
	router := gin.Default()
	router.GET("/ws", func(ctx *gin.Context) {
		// get websocket conn
		conn, err := ws.GetUpgradeConnection(ctx.Writer, ctx.Request)
		if err != nil {
			http.NotFound(ctx.Writer, ctx.Request)
			return
		}
        
        // 指定客户端的消息处理方法
		client := ws.NewConnection(conn, func(ctx *ws.Context) string {
            // 返回响应内容
			return ctx.GetMessage()
        })
        
        // listen
        go client.Listen()
        
        // register connection
		m.Register(client)
        
	})
    // start websocket service
	go m.Start()

	_ = router.Run()    
}
```

## 测试
使用wscat测试websocket
```
// 安装
npm install -g wscat

// 建立连接
wscat -c ws://localhost:8080

// 发送请求
> hello

// 收到的响应
< hello
```
