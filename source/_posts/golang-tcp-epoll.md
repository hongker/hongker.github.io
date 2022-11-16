---
title: Golang系列(32)-Reactor模型(Epoll)
date: 2022-11-15 22:56:47
tags: golang
---

本文将Reactor并发模型与Epoll的实现

更多请参考：我的自研网络框架 [znet](https://github.com/ebar-go/znet)，欢迎Star与提Issue。

## Reactor
>Reactor模式，是指通过一个或多个输入同时传递给服务处理器的服务请求的事件驱动处理模式。 

- 普通的函数处理机制为：调用某函数-> 函数执行， 主程序等待阻塞-> 函数将结果返回给主程序-> 主程序继续执行。 
- Reactor 事件处理机制为：主程序将事件以及对应事件处理的方法在 Reactor 上进行注册，如果相应的事件发生，Reactor 将会主动调用事件注册的接口，即 回调函数。

交互图如下（来自网络）：
![Reactor模型](https://pic3.zhimg.com/80/v2-30401fced0ce7a24ac6299f785bc16fa_720w.webp)

## 为什么要用Reactor模型
相比常规模式的为每个连接开启一个线程来读取和写入数据，Reactor模型只需要一个主线程就能管理所有的连接。这样可以极大的节省内存占用。

## Epoll
> epoll 全称 eventpoll，是 linux 内核实现IO多路复用（IO multiplexing）的一个实现。IO多路复用的意思是在一个操作里同时监听多个输入输出源，在其中一个或多个输入输出源可用的时候返回，然后对其的进行读写操作。

linux下主要是通过epoll实现的reactor模型。
<!--more-->

### 关键函数
- epoll_create1: 创建一个epoll实例，文件描述符
- epoll_ctl: 将监听的文件描述符添加到epoll实例中，实例代码为将标准输入文件描述符添加到epoll中
- epoll_wait: 等待epoll事件从epoll实例中发生， 并返回事件以及对应文件描述符l

### 水平触发(level-triggered:LT)和边缘触发(edge-triggered:ET)
- 水平触发: socket接收缓冲区不为空 有数据可读 读事件一直触发；socket发送缓冲区不满 可以继续写入数据 写事件一直触发。
- 边沿触发：socket的接收缓冲区状态变化时触发读事件，即空的接收缓冲区刚接收到数据时触发读事件。

简单的说，边沿触发仅触发一次，水平触发会一直触发。不指定选项时系统默认是水平触发。

### 事件宏
- EPOLLIN ： 表示对应的文件描述符可以读（包括对端SOCKET正常关闭）；
- EPOLLOUT： 表示对应的文件描述符可以写；
- EPOLLPRI： 表示对应的文件描述符有紧急的数据可读（这里应该表示有带外数据到来）；
- EPOLLERR： 表示对应的文件描述符发生错误；
- EPOLLHUP： 表示对应的文件描述符被挂断；
- EPOLLET： 将 EPOLL设为边缘触发(Edge Triggered)模式（默认为水平触发），这是相对于水平触发(Level Triggered)来说的。
- EPOLLONESHOT： 只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个socket的话，需要再次把这个socket加入到EPOLL队列里


### 实现

- Epoll
```go
//go:build linux

package poller

import (
	"golang.org/x/sys/unix"
	"net"
	"reflect"
	"sync"
	"syscall"
)

// Epoll implements of Poller for linux
type Epoll struct {
	lock sync.RWMutex
	// 注册的事件的文件描述符
	fd int
	// max event size, default: 100
	maxEventSize int

	connBuffers []int
	events      []unix.EpollEvent
}

func (e *Epoll) Add(fd int) error {
	e.lock.Lock()
	defer e.lock.Unlock()
	// 向 epoll 实例注册文件描述符对应的事件
	// POLLIN(0x1) 表示对应的文件描述字可以读
	// POLLHUP(0x10) 表示对应的文件描述字被挂起
	// EPOLLET(0x80000000) 将EPOLL设为边缘触发(Edge Triggered)模式，这是相对于水平触发(Level Triggered)来说的。缺省是水平触发(Level Triggered)。

	// 只有当链接有数据可以读或者连接被关闭时，wait才会唤醒
	err := unix.EpollCtl(e.fd,
		unix.EPOLL_CTL_ADD,
		fd,
		&unix.EpollEvent{Events: unix.POLLIN | unix.POLLHUP | unix.EPOLLET, Fd: int32(fd)})

	if err != nil {
		return err
	}
	return nil

}

func (e *Epoll) Remove(fd int) error {
	e.lock.Lock()
	defer e.lock.Unlock()
	// 向 epoll 实例删除文件描述符对应的事件
	err := unix.EpollCtl(e.fd, syscall.EPOLL_CTL_DEL, fd, nil)
	if err != nil {
		return err
	}
	return nil
}

func (e *Epoll) Wait() ([]int, error) {
	events := e.events
	var (
		n   int
		err error
	)
	for {
		n, err = unix.EpollWait(e.fd, events, 100)
		if err == nil {
			break
		}
		if err == unix.EINTR {
			continue
		}
		return nil, err
	}
	e.lock.RLock()

	connections := e.connBuffers[:0]
	for i := 0; i < n; i++ {
		if events[i].Events == 0 || events[i].Fd == 0 {
			continue
		}

		connections = append(connections, int(e.events[i].Fd))
	}

	e.lock.RUnlock()
	return connections, nil
}

func (e *Epoll) Close() error {
	e.lock.Lock()
	defer e.lock.Unlock()
	return unix.Close(e.fd)
}

func NewPollerWithBuffer(size int) (Poller, error) {
	fd, err := unix.EpollCreate1(0)
	if err != nil {
		return nil, err
	}

	return &Epoll{
		fd:           fd,
		maxEventSize: size,
		events:       make([]unix.EpollEvent, size, size),
		connBuffers:  make([]int, size, size),
	}, nil
}

// SocketFD get socket connection fd
func (e *Epoll) SocketFD(conn net.Conn) int {
	tcpConn := reflect.Indirect(reflect.ValueOf(conn)).FieldByName("conn")
	fdVal := tcpConn.FieldByName("fd")
	pfdVal := reflect.Indirect(fdVal).FieldByName("pfd")
	return int(pfdVal.FieldByName("Sysfd").Int())
}

```