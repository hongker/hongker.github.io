---
title: Golang系列(32)-基于reactor模型的tcp并发服务
date: 2022-11-15 22:56:47
tags: golang
---

本文将介绍通过epoll实现的reactor并发模型的tcp服务。

## Reactor
>Reactor模式，是指通过一个或多个输入同时传递给服务处理器的服务请求的事件驱动处理模式。 

- 普通的函数处理机制为：调用某函数-> 函数执行， 主程序等待阻塞-> 函数将结果返回给主程序-> 主程序继续执行。 
- Reactor 事件处理机制为：主程序将事件以及对应事件处理的方法在 Reactor 上进行注册，如果相应的事件发生，Reactor 将会主动调用事件注册的接口，即 回调函数。

交互图如下（来自网络）：
![Reactor模型](https://pic3.zhimg.com/80/v2-30401fced0ce7a24ac6299f785bc16fa_720w.webp)