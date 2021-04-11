---
title: Golang26-内存模型
date: 2021-04-11 23:04:46
tags: golang
---
本文介绍Golang的内存模型相关知识。
## 什么是go的内存模型
>指定了一系列条件，在这些条件下，可以保证在协程中对变量的读取操作可以观察到其他协程对同一变量写操作的结果，这就是go内存模型。

## 为什么需要这些条件？
因为编译器无法保证指令执行顺序与程序书写顺序一致。   

如下示例：
```go
package main
func main() {
    i := 5
    go func() {
        i = 10
    }()
    for j := 0;j<100;j++ {
        fmt.Println(i)
    }
}
```
多运行几次，会发现结果可能不同，但可以发现，部分`Println`打印的`i`变量依然是`5`，说明未能立观察到协程对`i`变量的写操作。

所以我们为了保证协程间的变量读取的可观察，就需要建立`Happen Before`关系的同步事件。也就是内存模型设定的`条件`。

## Happen Before
以下几种方式，可以建立`Happen Before`关系的同步事件:
```
1.init函数
2.创建/销毁goroutine
3.channel
4.锁
5.Once
```
<!--more-->

### init函数
包a引入包b，那么包b的init就会happen before 包a的init函数。如下：



```go
// a.go
package a
import "b"
func init() {
    fmt.Println("a")
    b.DoSomething()
}

```

```go
// b.go
package b
func init() {
    fmt.Println("b")
}
func DoSomething() {
    fmt.Println("test")
}
```
输出如下:
```
b
a
test
```

### 创建/销毁Goroutine
- 创建goroutine happen before goroutine执行
- goroutine执行 happen before goroutine销毁

### channel
- 对于无缓冲的channel，`recv`操作happen before `send`
```go
package main
import "fmt"
import "time"
func main() {
    i := make(chan int)
    go func() {
        fmt.Println("send")
        i <- 10
    }()
    go func() {
         fmt.Println("recv")
        fmt.Println(<-i)
    }()
   time.Sleep(time.Second)
}
```
输出结果：
```
send
recv
10
```

- 关闭channel的操作 happen before 接受到0值。
- 对于容量为m的channel,第n次`recv`是happen before 第n+m次`send`
```go
package main

import "fmt"
import "time"
func main() {
    ch := make(chan int, 5)
    go func() {
        for i:=0;i<10;i++ {
            time.Sleep(100)
            fmt.Println("send", i)
            ch <- i
        }
        close(ch)
        
    }()
    go func() {
        for i := range ch {
             time.Sleep(100)
             fmt.Println("receive", i)
        }
    }()
   time.Sleep(time.Second * 2)
}
```
输出结果发现，对于长度为5的channel,第1次接收比6次发送要先执行。

### 锁
- 对于任意的`sync.Mutex`或者`sync.RWMutex`,n<m时,n次`Unlock`的调用happen before`Lock`的调用。
- 对于sync.RWMutex,第n次`Unlock`happen before第n次`RLock`，而第n次`RUnlock` 又是happen before第n+1次`Lock`

### sync.Once
- 对于`fn()`的单个调用会happen before所有的`once.Do(fn)`返回之前发生
```go
package main
import "fmt"
import "sync"
func main() {
	var o sync.Once
    i := 5
    for j := 0;j<3;j++ {
		o.Do(func() {
			i += 1
		})
		fmt.Println(i)
    }
}
```
执行结果：
```
6
6
6
```

## 备注
虽然这些都是go开发中一般都知道的`常识`，但是我们还是需要了解，为什么会这样，为什么需要这样？