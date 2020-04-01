---
title: Goland代码优化06-上下文Context
date: 2020-03-31 19:50:27
tags: golang
---
本文介绍关于Context的一些要点。

## 什么是Context
>Context，上下文，golang协程的相关环境快照，其中包含函数调用以及涉及的相关的变量值。
通过Context在协程之间进行数据传递，相对于维护全局变量要简单。但也有人说它想`virus`,到处传播，褒贬不一。所以不同的场景下，还得程序猿自己决定是否适用。

## 结构
Context是一个树形结构。首先需要创建的是根节点。
- 顶层Context: Background
```go
func main() {
    // 声明一个空的Context，它将作为所有由此继承Context的根节点
    ctx := context.Background()
}
```

- 子孙节点 WithCancel/WithDeadline/WithTimeout/WithValue
```
// 带cancel返回值的Context，一旦cancel被调用，即取消该创建的context
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) 

// 带有效期cancel返回值的Context，即必须到达指定时间点调用的cancel方法才会被执行
func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc) 

// 带超时时间cancel返回值的Context，类似Deadline，前者是时间点，后者为时间间隔
// 相当于WithDeadline(parent, time.Now().Add(timeout)).
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)

// 带key-val的Context
func WithValue(parent Context, key string, val interface) Context
```

<!--more-->
## 示例
- 控制协程同步退出
```go
func main() {
	ctx, cancel := context.WithCancel(context.Background())
	go func() {
		for {
			time.Sleep(1 * time.Second)
			select {
			case <-ctx.Done():
				fmt.Println("done")
				return
			default:
				fmt.Println("working...")
			}
		}
	}()
	time.Sleep(1e9 * 3) // 模拟业务逻辑执行
	cancel()
	time.Sleep(1e9 * 2) // wait goroutine finished
}
```

- 全局TraceID
```go
import "github.com/satori/go.uuid"
func main() {
    // set traceId in context
    ctx := context.WithValue(context.Background(), "traceId", uuid.NewV4().String())
    
    // current login user
	ctx = context.WithValue(ctx, "username", "test")

	var wg sync.WaitGroup
	wg.Add(1)
	go func() {
		defer wg.Done()
		traceId := ctx.Value("traceId").(string)
		username := ctx.Value("username").(string)
		fmt.Printf("traceId: %s, current user: %s\n", traceId, username)
	}()
    // wait goroutine execute finished
	wg.Wait()
}
```

- 超时处理
```go

func main() {
	// 定义一个3秒自动超时的context
	ctx, cancel := context.WithTimeout(context.Background(), 1e9 *3)
	// 使用defer的特性，达到如果提前执行完成，自动调用cancel方法释放资源
	defer cancel()
	go func() {
		for {
			select {
			case <-ctx.Done():
				fmt.Printf("cancel:%s\n", ctx.Err())
				return
			default:
				{
					fmt.Println("do something")
					time.Sleep(1e9 * 1) // 模拟业务代码执行时间
				}
			}
		}
	}()
	time.Sleep(1e9 * 5)
}
```
