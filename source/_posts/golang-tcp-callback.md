---
title: Golang系列(31)-TCP回调函数
date: 2022-11-14 19:42:27
tags: golang
---
本文介绍如何给tcp连接增加回调函数，和websocket一样，提供三个回调函数:OnOpen,OnClose,OnMessage。


更多请参考：我的自研网络框架 [znet](https://github.com/ebar-go/znet)

## 回调函数
- OnOpen: 连接建立成功后触发的回调函数。
- OnClose: 连接断开后触发的回调函数。
- OnMessage: 收到新消息时的回调函数。
