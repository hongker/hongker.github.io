---
title: 架构师之路-队列
date: 2020-07-28 14:41:59
tags: 架构与算法
---
本文介绍在golang里如何实现队列得数据结构

## 概念
队列是一种特殊的线性表，特殊之处在于它只允许在表的前端（front）进行删除操作，而在表的后端（rear）进行插入操作，和栈一样，队列是一种操作受限制的线性表。进行插入操作的端称为队尾，进行删除操作的端称为队头。队列中没有元素时，称为空队列。

## 特性
- 入列
从尾部将元素加入到队列里。
- 出列
从队列头部获取元素。
- 查看长度
获取队列存储的元素个数。
- 清空队列
将队列里的数据清空。

## Demo
- 定义Queue接口
```go
// Queue 队列
type Queue interface {
	// Push 入列
	Push(items ...interface{})
	// Pop 出列
	Pop() (interface{})
	// MPop 批量出列
	MPop(n int) ([]interface{})
	// Length 长度
	Length() int
	// Clear 清空
	Clear()
}
```

<!--more-->


- 实现Queue接口
```go
package queue

import "container/list"


type queue struct {
	// 使用链表结构实现
	list *list.List
}

func New() Queue {
	return &queue{list: list.New()}
}

func (q queue) Push(items ...interface{}) {
	for _, item := range items {
		q.list.PushBack(item)
	}
}

func (q queue) Pop() (interface{}) {
	item := q.list.Front()
	if item == nil {
		return nil
	}
	q.list.Remove(item)
	return item.Value
}

func (q queue) MPop(n int) ([]interface{}) {
	if n < 1 { // 当n小于1，返回nil
		return nil
	}
	var result []interface{}
	var items []*list.Element
	item := q.list.Front()

	for i := 0; i <n ; i++ {
		if item == nil { // 判断元素是否为空
			break
		}

		items = append(items, item)
		item = item.Next()
	}
	// 获取元素并删除队列里对应的数据
	for _, item := range items {
		q.list.Remove(item)
		result = append(result, item.Value)
	}
	return result
}

func (q queue) Length() int {
	return q.list.Len()
}

func (q queue) Clear() {
	q.list.Init()
}
```

- 使用
```go
package queue

import (
	"fmt"
	"github.com/stretchr/testify/assert"
	"testing"
)

func TestQueue(t *testing.T) {
	queue := New()
	// test Push
	queue.Push(1,2,3,4)

	// test Length
	assert.Equal(t, queue.Length(), 4)

	// test Pop
	item := queue.Pop()
	assert.Equal(t,  item, 1)
	assert.Equal(t,  queue.Length(), 3)

	// test MPop
	multiItems := queue.MPop(2)
	assert.Equal(t, []interface{}{2,3}, multiItems)
	assert.Equal(t,  queue.Length(), 1)

	// test Clear
	queue.Clear()
	assert.Equal(t, queue.Length(), 0)

}

func BenchmarkQueue_Push(b *testing.B) {
	queue := New()
	for i := 0; i < b.N; i++ {
		queue.Push(i)
	}
	fmt.Println(queue.Length())
}
```

更多请查看代码库:[https://github.com/ebar-go/gostructure](https://github.com/ebar-go/gostructure)