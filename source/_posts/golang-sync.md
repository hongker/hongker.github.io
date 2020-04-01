---
title: Golang代码优化07-sync包
date: 2020-04-01 09:38:43
tags: golang
---
本文介绍golang的sync包里的知识点与使用方式。

## 什么是Sync包
>Golang sync包提供了基础的异步操作方法，包括互斥锁Mutex，执行一次Once和并发等待组WaitGroup
主要的功能如下:
```
sync.Mutex: 互斥锁
sync.RWMutex: 读写锁
WaitGroup: 并发等待管理
Once：执行一次
Cond：信号量
Pool：临时对象池
Map：自带锁的map
```

## sync.Mutex&sync.RWMutex
- sync.Mutex称为互斥锁，常用在并发编程里面。互斥锁需要保证的是同一个时间段内不能有多个并发协程同时访问某一个资源(临界区)
- sync.RWMutex目的是为了能够支持多个并发协程同时读取某一个资源，但只有一个并发协程能够更新资源。也就是说读和写是互斥的，写和写也是互斥的，读和读是不互斥的。
这两个锁的用法在之前已介绍过，故不再重复介绍。需要的请查看：[Golang代码优化04-锁](http://localhost:4000/2020/03/31/golang-lock/)

## sync.WaitGroup
sync.WaitGroup指的是等待组，在Golang并发编程里面非常常见，指的是等待一组工作完成后，再进行下一组工作。

```
func (wg *WaitGroup) Add(delta int)  Add添加n个并发协程
func (wg *WaitGroup) Done()  Done完成一个并发协程
func (wg *WaitGroup) Wait()  Wait等待其它并发协程结束
```
简单的理解就是Boss监督Worker工作，等待所有的Worker完成后才收工！
```go
func main() {
    n := 100
    wg := sync.WaitGroup{}
    // 监控n个协程
    wg.Add(n) 
    for i := 0; i < n; i++ {
        go func(i int) {
            fmt.Println(i)
            // 标识已完成工作
            wg.Done() 
        }(i)
    }
    // 阻塞等待所有goroutine完成工作
    wg.Wait()
    fmt.Println("All done!")
}
```

## sync.Once
<!--more-->
sync.Once指的是只执行一次的对象实现，常用来控制某些函数只能被调用一次。
>插播一条故事，关于sync.Once，在03-31与EasySwoole的作者有过一次讨论
>他：说说sync.Once的作用
>我：常用于单例模式
>他：一看你就是Go的初学者。
>我：恩，大佬说说看，我好学学一波
>他：本质在于、、、、go是多线程协程，而单例模式的时候，如果用锁来解决单例问题，那么效率是非常差的。这个可以参考java多线程下的同步锁问题，为此go提供了一个sync.Once机制，不过实际上，go的本质实现，也是锁。嗯，反正就是，你没回到到问题的本质。
>我：学习了，受益颇多。

emmm...于是乎抱着听了大佬的一波解释，因而有点怀疑的想法，下来仔细研究了sync.Once的源码，代码其实很简单，和大佬说的其实差不多。但是实际不是那么简单，写go的人真的牛逼(实在是666)。下面看看源码：

```go
type Once struct {
    // done 标识符，这里放在第一位也是有讲究的
    // 一个重要的概念 hot path ，即 Do 方法的调用会是高频的，而每次调用访问 done，done位于结构体的第一个字段，可以通过结构体指针直接进行访问
    // 访问其他的字段需要通过偏移量计算就慢了
    done uint32
    // m 互斥锁
	m    Mutex
}
// Do 
func (o *Once) Do(f func()) {
    // 函数atomic.LoadInt32接受一个*int32类型的指针值，并会返回该指针值指向的那个值
    // 在这里读取o.done的值的同时，当前计算机中的任何CPU都不会进行其它的针对此值的读或写操作。这样的约束是受到底层硬件的支持的
    // 刚开始我理解的主要作用是用原子操作可以提高性能，和大佬说的一致。但实际并非如此。普通赋值o.done = 1也可以实现相同的作用。
    // 如果是普通赋值，当协程A刚执行完原子赋值操作，协程B阻塞等待加锁的时候，协程C在这里的判断为true,因为协程间对同一变量不存在同步手段，Go并不能保证协程A的赋值操作能被C读到，所以有可能在协程C里会执行o.doSlow(f),与只执行一次想违背。而使用原子操作读写，刚好可以避免这一个问题。
	if atomic.LoadUint32(&o.done) == 0 {
		// Outlined slow-path to allow inlining of the fast-path.
		o.doSlow(f)
	}
}
// doSlow 
func (o *Once) doSlow(f func()) {
    // 加锁，保证协程安全
	o.m.Lock()
    defer o.m.Unlock()
    // 因为使用了 Mutex 进行了锁操作，o.done == 0 处于锁操作的临界区中，所以可以直接进行比较
	if o.done == 0 {
        // 原子store函数执行次数1
		defer atomic.StoreUint32(&o.done, 1)
		f()
	}
}
```

总的来说，sync.Once的使用场景例如单例模式、系统初始化。
>因为这个插曲，深入研究了sync.Once的原理，受益匪浅，还是感谢大佬的启发。虽然大佬说的有道理，但我依然坚持我的观点`常用于单例模式`。如果有一天谁能说服我不能这么用，我也会听听理由，自己思考一番。

## sync.Cond
sync.Cond指的是同步条件变量，一般需要与互斥锁组合使用，本质上是一些正在等待某个条件的协程的同步机制。主要函数如下：
```
// Wait 等待通知
func (c *Cond) Wait()
// Signal 单播通知
func (c *Cond) Signal()
// Broadcast 广播通知
func (c *Cond) Broadcast()
```
- 示例
```go
func main() {
    cond := sync.NewCond(&sync.Mutex{})

    for i := 0; i < 3; i++ {
        go func(i int) {
            cond.L.Lock() // 获取锁
            fmt.Println("上班，摸鱼...")
            cond.Wait() //等待通知  暂时阻塞
            fmt.Println("等待下班...")
            cond.L.Unlock() //释放锁

            fmt.Println("下班，Worker:", i)
        }(i)
    }

    time.Sleep(1e9)
    cond.Signal()  // 让leader先走

    time.Sleep(1e9)
    cond.Broadcast() // 全部下班

    time.Sleep(2e9) // 等待协程执行完成
}
```

## sync.Pool
通常用golang来构建高并发场景下的应用，但是由于golang内建的GC机制会影响应用的性能，为了减少GC，golang提供了对象重用的机制，也就是sync.Pool对象池
>注：千万不能把它当成内存池使用。
```go
func main() {
    var bufferPool = sync.Pool{
		New: func() interface{} {
			return new(bytes.Buffer)
		},
	}

	for i:=0;i<10;i++ {
        // 获取buffer
		buffer := bufferPool.Get().(*bytes.Buffer)
		buffer.Reset()
		bufferPool.Put(buffer)
		buffer.Write([]byte("hello" + strconv.Itoa(i))
		fmt.Println( buffer.String())
	}
}
```
- 接收并读取http的响应内容
实际代码之前已描述过，请查看:[Golang代码优化03-Http响应处理](https://hongker.github.io/2020/03/31/http-response/)

## sync.Map
貌似不建议使用了。暂不介绍

