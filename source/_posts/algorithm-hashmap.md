---
title: Go数据结构与算法(10)-实现hashmap
date: 2022-07-14 21:11:25
tags: algorithm
---

面试官经常问到的一个问题，hashmap是如何实现的？今天我们就来说下原理

## 什么是hashmap?
>map就是用于存储键值对（<key,value>）的集合类，也可以说是一组键值对的映射。而hashmap就是将key进行hash运算，得到目标元素在哈希表中的位置，然后再进行少量比较即可得到元素，这使得 HashMap 的查找效率极高

### hash函数
根据输入的关键字得到固定长度的输出，通常是返回一个较大的整形。

### hash映射
将通过hash运算得到的整数平均分配到指定位置，一般可以采用位运算或者模运算来实现。在性能方面，位运算的效率高于模运算。

### 寻址算法
地址是通过hash值映射得到的，那么hash映射后会出现不同的key对应到相同的地址，也就是所谓的hash冲突，为此解决冲突主要通过以下两种方法去寻址。
- 链表法：每个桶对应的是一个链表，通过遍历链表，找到对应的key。
- 开放地址法：根据初次映射的地址，遍历相邻的位置，找到对应的key。

<!--more-->

## 算法实现
```go
package hashmap

const loadFactor = 0.65 // 负载因子，控制扩容的触发，参考golang的原生map

type HashMap struct {
	count   uint64  //总数量
	buckets buckets //桶个数
}

func New(hint uint64) *HashMap {
	if hint == 0 {
		hint = 16
	}
	return &HashMap{
		count:   0,
		buckets: make(buckets, roundUp(hint)),
	}
}

// Get 查询，返回值以及是否存在
func (hm *HashMap) Get(key uint64) (interface{}, bool) {
	// 根据关键字查找对应的位置
	idx := hm.buckets.find(key)
	if hm.buckets[idx] == nil {
		return nil, false
	}
	return hm.buckets[idx].val, true
}

// Set 赋值
func (hm *HashMap) Set(key uint64, val interface{}) {
	// 判断到负载因子大于指定阈值时，就需要扩容并重新分配
	if float64(hm.count+1)/float64(len(hm.buckets)) > loadFactor {
		hm.rebuild()
	}
	hm.buckets.set(&item{key: key, val: val})
	hm.count++
}

// rebuild 扩容并重新分配
func (hm *HashMap) rebuild() {
	// 利用一个扩容的临时buckets，重新赋值
	temp := make(buckets, roundUp(uint64(len(hm.buckets)+1)))
	for _, item := range hm.buckets {
		if item == nil {
			continue
		}
		temp.set(item)
	}
	hm.buckets = temp
}

type buckets []*item

// find 查询key所在位置
func (buckets buckets) find(key uint64) uint64 {
	// 进行hash运算后，找到hash映射后的位置
	idx := buckets.hashFor(hashcode(key))
	// 利用开放地址法处理hash冲突后的寻址
	for buckets[idx] != nil && buckets[idx].key != key {
		// 当冲突后，通过线性探测，依次遍历找到空闲位置
		idx = (idx + 1) & (uint64(len(buckets)) - 1)
	}

	return idx
}

func (buckets buckets) set(item *item) {
	idx := buckets.find(item.key)
	if buckets[idx] == nil { // 如果为空闲位置，则直接赋值
		buckets[idx] = item
		return
	}

	// 如果key已存在，则覆盖val
	buckets[idx].val = item.val
}

// hashFor 通过位运算确定hashcode对应的位置
func (buckets buckets) hashFor(hashcode uint64) uint64 {
	return hashcode & (uint64(len(buckets)) - 1)
}

// item 数据项
type item struct {
	key uint64 // 关键字
	val interface{} // 值
}



// hashcode hash函数
func hashcode(key uint64) uint64 {
	key ^= key >> 33
	key *= 0xff51afd7ed558ccd
	key ^= key >> 33
	key *= 0xc4ceb9fe1a85ec53
	key ^= key >> 33
	return key
}



// roundUp 返回邻近的2的N次方的数
func roundUp(v uint64) uint64 {
	v--
	v |= v >> 1
	v |= v >> 2
	v |= v >> 4
	v |= v >> 8
	v |= v >> 16
	v |= v >> 32
	v++
	return v
}
```