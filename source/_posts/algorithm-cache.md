---
title: Go数据结构与算法(09)-基于LRU算法的缓存
date: 2022-07-14 20:15:04
tags: algorithm
---

在项目中，使用缓存是常见的提升性能的一个方法。为防止内存被无限制的消耗，使用内存缓存必须注意到一个淘汰策略，本文就介绍其中的一种算法：LRU。

## 什么时LRU？
>LRU (Least recently used：最近最少使用)算法在缓存写满的时候，会根据所有数据的访问记录，淘汰掉未来被访问几率最低的数据。也就是说该算法认为，最近被访问过的数据，在将来被访问的几率最大。

简单的理解，就是当缓存需要淘汰时，先淘汰最早写入且最近没有被查询过的缓存元素。

<!--more-->

## 示例图
![lru](lru.png)


## Demo
- 算法实现
>利用链表的特性，将Put或Get调用的关键字，都放在队列的头部。每次触发容量上限，就先删除队列尾部的关键字。



```go
package cache

import (
	"container/list"
	"sync"
)

type LRUCache struct {
	sync.Mutex                    // 锁，保证并发安全
	cap        uint64             // 最大容量
	size       uint64             // 缓存size
	items      map[string]*cached // 缓存数据
	keyList    *list.List         // 链表，用于存储最近使用的key
}

func NewLRUCache(capacity uint64) *LRUCache {
	c := &LRUCache{
		cap:     capacity,
		keyList: list.New(),
		items:   map[string]*cached{},
	}
	return c
}

// Get 获取单个缓存元素，如果不存在，返回nil
func (cache *LRUCache) Get(key string) Item {
	cache.Lock()
	defer cache.Unlock()
	return cache.get(key)
}

func (cache *LRUCache) get(key string) Item {
	// 判断数据是否存在
	cached, exist := cache.items[key]
	if !exist {
		return nil
	}

	// 触发最近使用策略
	cache.record(key)
	return cached.item
}

// BatchGet 批量获取
func (cache *LRUCache) BatchGet(keys ...string) []Item {
	cache.Lock()
	defer cache.Unlock()
	items := make([]Item, len(keys))
	for i, key := range keys {
		items[i] = cache.get(key)
	}
	return items
}

// ensureCapacity 根据新增的数据的长度，判断总容量是否溢出，如果满足，则需要删除一部分数据，直到容量正常
func (cache *LRUCache) ensureCapacity(toAdd uint64) {
	mustRemove := int64(cache.size+toAdd) - int64(cache.cap)
	for mustRemove > 0 {
		// 从队列尾部先删除，也就是先插入先删除
		key := cache.keyList.Back().Value.(string)
		mustRemove -= int64(cache.items[key].item.Size())
		cache.remove(key)
	}
}

// Put 存储元素
func (cache *LRUCache) Put(key string, item Item) {
	cache.Lock()
	defer cache.Unlock()
	cache.remove(key)

	cache.ensureCapacity(item.Size())
	cached := &cached{item: item}
	cached.setElementIfNotNil(cache.record(key))
	cache.items[key] = cached
	cache.size += item.Size()
}

// Remove 删除已缓存的元素
func (cache *LRUCache) Remove(keys ...string) {
	cache.Lock()
	defer cache.Unlock()
	for _, key := range keys {
		cache.remove(key)
	}
}

func (cache *LRUCache) remove(key string) {
	if cached, ok := cache.items[key]; ok {
		// 删除数组里的元素
		delete(cache.items, key)
		// 减去大小
		cache.size -= cached.item.Size()
		// 移除链表中的元素
		cache.keyList.Remove(cached.element)
	}
}

func (cache *LRUCache) Size() uint64 {
	return cache.size
}

// record 将关键字移动到队列首部，表明近期有使用过
func (cache *LRUCache) record(key string) *list.Element {
	// 如果数据已存在，将元素key移动到链表头部
	if item, ok := cache.items[key]; ok {
		cache.keyList.MoveToFront(item.element)
		return item.element
	}
	// 数据不存在，直接插入到头部
	return cache.keyList.PushFront(key)
}


// Item is an item in a cache
type Item interface {
	// Size returns the item's size, in bytes
	Size() uint64
}

// A tuple tracking a cached item and a reference to its node in the eviction list
type cached struct {
	item    Item
	element *list.Element
}

// Sets the provided list element on the cached item if it is not nil
func (c *cached) setElementIfNotNil(element *list.Element) {
	if element != nil {
		c.element = element
	}
}
```

- 使用
```go
package cache

import (
	"github.com/stretchr/testify/assert"
	"testing"
)

type mockItem string
func (item mockItem) Size() uint64 {
	return uint64(len(item))
}

func TestLRUCache(t *testing.T) {
	// 实例化一个容量为10的缓存
	cache := NewLRUCache(10)
	// 插入一个key为a，值为hello的元素,占用5个长度的容量
	cache.Put("a", mockItem("hello"))
	// 测试能正常获取
	assert.Equal(t, mockItem("hello"), cache.Get("a"))
	// 插入一个world,占用6个长度的容量，此刻cache的总容量已被占满
	cache.Put("b", mockItem("world"))
	// 再插入一个go,空间不够，就会淘汰先插入的hello
	cache.Put("c", mockItem("go"))
	assert.Equal(t, mockItem("go"), cache.Get("c"))
	// 此刻再获取，就返回nil,因为hello已被删除
	assert.Nil(t, cache.Get("a"))
}

```