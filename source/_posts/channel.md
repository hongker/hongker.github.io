---
title: 协程与管道
date: 2020-03-30 16:38:50
tags:
---
介绍一些常规的用法
## channel的状态
- channel:未初始化时为nil
- active:正常的channel,可读可写
- closed:调用了close()后的状态

## channel的操作
- 读 <-channel
- 写 ->channel
- 关闭 close

## 常用的channel使用方法
- 使用for..range 读取channel
当需要不断从channel中读取数据时，使用for..range,当channel关闭时，for..range会自动退出
```go
func main() {
    ch := make(chan string)
	go sendData(ch)
	for out := range ch {
		fmt.Println(out)
	}
}

func send(ch chan string) {
    ch <- "Washington"
	ch <- "Tripoli"
	ch <- "London"
	ch <- "Beijing"
	ch <- "Tokyo"
	defer close(ch)
}
```

- 使用value,ok判断channel是否关闭
当直接读取channel时，如果channel已关闭，会触发panic,故需要判断channel是否关闭
```go
func main() {
    ch := make(chan string)
	go sendData(ch)
	for {
		if out, open := <- ch; open {
			fmt.Println(out)
		}else {
			break
		}
	}
}
```

- 使用select读取多个channel
select可以同时监控多个通道的情况，只处理未阻塞的case。当通道为nil时，对应的case永远为阻塞，无论读写。特殊关注：普通情况下，对nil的通道写操作是要panic的。

- 使用channel的声明控制读写权限
协程对某个channel只读或只写

- 使用缓冲channel增强并发和异步
有缓冲通道是异步的，无缓冲通道是同步的。有缓冲通道可供多个协程同时处理，在一定程度可提高并发性。

<!--more-->

- 为操作加上超时
在实际开发中，无法预估时间的程序，为了保证程序安全，我们需要超时控制的操作。
```go
func main() {
	fmt.Println(doWithTimeout(1e9 * 2))
}

func doWithTimeout(timeout time.Duration) (int, error) {
	select {
	case res := <- do():
		return res, nil
	case <-time.After(timeout):
		return 0, errors.New("timeout")
	}
}

func do() <- chan int {
	outCh := make(chan int)
	go func() {
		time.Sleep(1e9 * 3) // 模拟如复杂的sql查询
		outCh <- 1
	}()
	return outCh
}
```

- 使用chan struct{}作为信号channel
使用channel传递信号，而不是传递数据时
```go
func main() {
	done := make(chan struct{})
	go func() {
		time.Sleep(1e9 * 2)
		close(done) // 发送关闭信息
	}()
	for {
		select {
		case <-done:
			fmt.Println("done")
			return
		}
	}

}
```