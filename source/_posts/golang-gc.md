---
title: Golang代码优化14-垃圾回收
date: 2020-04-11 10:33:47
tags: golang
---
垃圾回收是golang开发者需要关注的重点知识。

## 什么是垃圾回收
>垃圾回收（英语：Garbage Collection，缩写为GC）是一种自动内存管理机制. 垃圾回收器(Garbage Collector)尝试回收不再被程序所需要的对象所占用的内存. 

## 经典算法
### 引用计算
将新引用对象的计数器加一，将不引用的对象的计数器减一。

### 标记-清除
找到不可达的对象，然后做上标记。回收标记对象。因为扫描时会冻结所有的线程，执行时程序会暂停、卡顿，即Stop The World。(Dio咋瓦鲁多)

## Golang的GC 
golang基于标记-清除算法进行优化，采用三色标记法。
- 原理
首先将程序创建的对象都标记为白色。
从root根出发扫描所有根对象，扫描所有可达的对象，标记为灰色。（root区域主要是程序运行到当前时刻的栈和全局数据区域。）
从灰色对象中找到其引用对象，把灰色对象本身标记为黑色。
监视对象中的内存修改，并重复上一步骤，知道灰色标记的对象不存在。
此时，GC回收白色对象。 
最后将所有黑色标记的对象重置为白色。

- 三色标记法如何避免Stop the world的？
因为三色标记法结束后仅剩黑色与白色对象。如果不碰触黑色对象，只清除白色对象，则不会影响程序的执行。故清除操作可以和用户程序并发执行。

针对新生成的对象，go采用了`写屏障`保证该对象不会被立刻清除。简单理解就是新生成的对象，一律被标记为灰色。

说实话我对原理的理解还是很生硬，仅记住三色标记法的原理。

## 如何进行GC优化
<!--more-->
- 减少对象的分配，合理重复利用。示例：
使用`sync.Pool`初始化buffer池
```go
// Adapter buffer pool
type Adapter struct {
	pool sync.Pool
}

// NewAdapter
func NewAdapter() *Adapter {
	return &Adapter{pool:sync.Pool{New: func() interface{} {
		return bytes.NewBuffer(make([]byte, 4096))
	}}}
}

func main() {
    adapter := NewAdapter()
    buffer := adapter.pool.Get().(*bytes.Buffer)
	buffer.Reset()
	defer func() {
		if buffer != nil {
			adapter.pool.Put(buffer)
			buffer = nil
		}
    }()
    
    // 使用buffer读取信息..
}
```
- 避免string与[]byte转换
两者发生转换的时候，底层数据结结构会进行复制，因此导致 gc 效率会变低。优化如下：
```go
// Str2Byte return bytes of s
func Str2Byte(s string) []byte {
	x := (*[2]uintptr)(unsafe.Pointer(&s))
	h := [3]uintptr{x[0], x[1], x[1]}
	return *(*[]byte)(unsafe.Pointer(&h))
}

// Byte2Str return string of b
func Byte2Str(b []byte) string {
	return *(*string)(unsafe.Pointer(&b))
}
```

- 少量使用`+`连接string
Go里面string是最基础的类型，是一个只读类型，针对他的每一个操作都会创建一个新的string。 如果是少量小文本拼接，用 “+” 就好；如果是大量小文本拼接，用 strings.Join；如果是大量大文本拼接，用 bytes.Buffer。
```go
func main() {
    strings.Join([]string{"hello,","world"},"")
}
```