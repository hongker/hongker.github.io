---
title: Go数据结构与算法系列(02)-集合
date: 2020-08-04 22:11:46
tags: algorithm
---

本篇文章主要介绍golang里集合的实现思路。

## 什么是集合？
>集合通常是由一组无序的, 不能重复的元素构成

## 功能
- 添加元素
- 删除元素
- 获取集合的元素数量
- 检查集合是否包含某元素
- 清空集合
- 判断集合是否为空
- 集合转数组
- 复制

接口设计如下：
```go
type Set interface {
	// 添加元素
	Add(items ...interface{})
	// 包含
	Contain(item interface{}) bool
	// 删除
	Remove(item interface{})
	// 集合大小
	Size() int
	// 清空
	Clear()
	// 判断是否为空
	Empty() bool
	// 创建副本
	Duplicate() Set
	// 数组
	ToSlice() []interface{}
}
```

## 具体实现
<!--more-->
### 线程不安全
我们使用golang的`map`来实现集合
```go
import (
	"fmt"
	"strings"
)

// threadUnsafeSet 非线程安全的集合，采用元素值作为key，空的struct作为值
type threadUnsafeSet map[interface{}]struct{}

func newThreadUnsafeSet() threadUnsafeSet {
	return make(threadUnsafeSet)
}

func (set *threadUnsafeSet) Add(items ...interface{}) {
	for _, item := range items {
		(*set)[item] = struct{}{}
	}
}

func (set *threadUnsafeSet) Contain(item interface{}) bool {
	_, ok := (*set)[item]
	return ok
}

func (set *threadUnsafeSet) Remove(item interface{}) {
	delete((*set), item)
}

func (set *threadUnsafeSet) Size() int {
	return len((*set))
}

func (set *threadUnsafeSet) Clear() {
	*set = newThreadUnsafeSet()
}

func (set *threadUnsafeSet) Empty() bool {
	return set.Size() == 0
}

func (set *threadUnsafeSet) Duplicate() Set {
	duplicateSet := newThreadUnsafeSet()
	for item, _ := range *set {
		duplicateSet.Add(item)
	}
	return &duplicateSet
}

func (set *threadUnsafeSet) String() string {
	items := make([]string, 0, len(*set))

	for elem := range *set {
		items = append(items, fmt.Sprintf("%v", elem))
	}
	return fmt.Sprintf("{%s}", strings.Join(items, ", "))
}

func (set *threadUnsafeSet) ToSlice() []interface{} {
	keys := make([]interface{}, 0, set.Size())
	for elem := range *set {
		keys = append(keys, elem)
	}

	return keys
}
```

## 线程安全的集合
在非线程安全集合的基础上，通过读写锁实现线程安全。
```go
import "sync"

type threadSafeSet struct {
	s threadUnsafeSet
	sync.RWMutex
}

func newThreadSafeSet() *threadSafeSet  {
	return &threadSafeSet{s: newThreadUnsafeSet()}
}

func (set *threadSafeSet) Add(items ...interface{}) {
	set.Lock() //数据新增采用互斥锁
	set.s.Add(items...)
	set.Unlock()
}

func (set *threadSafeSet) Contain(item interface{}) bool {
	set.RLock() // 采用读写锁
	defer set.RUnlock()
	return set.s.Contain(item)
}

func (set *threadSafeSet) Remove(item interface{}) {
	set.Lock()
	set.s.Remove(item)
	set.Unlock()
}

func (set *threadSafeSet) Size() int {
	set.RLock()
	defer set.RUnlock()
	return set.s.Size()
}

func (set *threadSafeSet) Clear() {
	set.Lock()
	set.s.Clear()
	set.Unlock()
}

func (set *threadSafeSet) Empty() bool {
	return set.Size() == 0
}

func (set *threadSafeSet) Duplicate() Set {
	set.RLock()
	defer set.RUnlock()
	s := set.s.Duplicate()
	return &threadSafeSet{s: *(s.(*threadUnsafeSet))}
}

func (set *threadSafeSet) ToSlice() []interface{} {
	set.RLock()
	defer set.RUnlock()
	return set.s.ToSlice()
}
```

### 初始化函数
```go
func ThreadSafe(items ...interface{}) Set {
	s := newThreadSafeSet()
	s.Add(items...)
	return s
}

func ThreadUnsafe(items ...interface{}) Set {
	s := newThreadUnsafeSet()
	s.Add(items...)
	return &s
}
```

### 单元测试
```go
import (
	"fmt"
	"github.com/stretchr/testify/assert"
	"testing"
)

func TestThreadUnsafeSet_Add(t *testing.T) {
	set := ThreadSafe()
	//set := ThreadUnsafe()

	set.Add(1,2,3,4)

	assert.True(t, set.Contain(1))
	set.Remove(2)
	assert.Equal(t, 3, set.Size())

	set.Clear()
	assert.Equal(t, 0, set.Size())
	assert.True(t, set.Empty())

	assert.False(t, set.Contain(5))
	set.Add(5)
	assert.True(t, set.Contain(5))
	fmt.Println(set.ToSlice())

	set.Add("hello", "world")
	fmt.Println(set.ToSlice())

	copySet := set.Duplicate()
	fmt.Println(copySet.ToSlice())
}
```