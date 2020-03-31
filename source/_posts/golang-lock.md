---
title: Golang代码优化04-锁
date: 2020-03-31 13:52:03
tags: golang
---
介绍在项目开发中，经常用到到保证数据安全的锁的使用。

## sync.Mutex 互斥锁
- demo
```go
type Item struct {
	mu sync.Mutex
	number int
}
// NewItem
func NewItem() *Item {
	return &Item{
		mu:     sync.Mutex{},
		number: 0,
	}
}
// SetNumber
func (item *Item) SetNumber(num int) {
	item.mu.Lock()
	defer item.mu.Unlock()
	item.number = num
}
// Number
func (item *Item) Number() int {
	item.mu.Lock()
	defer item.mu.Unlock()
	return item.number
}
```
通过mu互斥锁保证写入数据和读取数据都是线程安全的。如果不用锁，在多个协程里执行SetNumber()容易导致获取的数据和预期不一致

## RWMutex 读写锁
在读多写少的场景中，可以优先使用读写锁RWMutex。
```go
package main

import (
    "fmt"
    "sync"
    "time"
)

func main() {
    var wg sync.WaitGroup

    u := NewUser()
    // 通过协程并发读取，这里使用读锁，比互斥锁效率更高
    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            fmt.Println("Get Age:", u.GetAge(),time.Now().Second())
        }()
    }

    // 激活一个申请写锁的goroutine
    go func() {
        wg.Add(1)
        defer wg.Done()
        u.ChangeAge(10)
    }()
    // 阻塞，直到所有wg.Done
    wg.Wait()
}

// NewUser return user instance
func NewUser() *U {
    return &U{
        rwm: sync.RWMutex{},
        Age: 0,
    }
}

// U user
type U struct {
    rwm sync.RWMutex
    Age int
}
// GetAge
func (u *U) GetAge() int {
    u.rwm.RLock()
    defer u.rwm.RUnlock()
    return u.Age
}
// ChangeAge
func (u *U) ChangeAge(age int) {
    u.rwm.Lock()
    defer u.rwm.Unlock()
    u.Age = age
}
```

<!--more-->

## sync.WaitGroup
WaitGroup相当于协程任务的管理者，监控受管理的协程是否已执行完成。
```go
func main() {
    wg := sync.WaitGroup{}
    wg.Add(100)
    for i := 0; i < 100; i++ {
        go func(i int) {
            fmt.Println(i)
            wg.Done()
        }(i)
    }
    wg.Wait()
}
```