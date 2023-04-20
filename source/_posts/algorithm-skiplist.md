---
title: 数据结构与算法(15)-跳表(SkipList)
date: 2023-04-18 22:33:56
tags: algorithm
---
在redis中，利用SkipList实现了如SortedSet这样的数据结构。本文将介绍SkipList算法实现细节

## 什么是SkipList(跳表)
>跳表是一种基于多级索引的数据结构，用于快速查找链表中的数据。它的查询和插入操作都可以在 O(log n) 的时间复杂度内完成。

- 设计思想
跳表的基本思想是为链表建立多级索引，每层索引包含了下一层索引的部分节点。较高层的索引节点可以直接跨越较低层的多个节点，从而实现了快速查找的功能。

- 适用场景
因为跳表具有简单、易实现、效率高等优点，所以被广泛应用于各类数据存储系统和搜索算法中。比如Redis的有序集合就是采用SkipList实现的

## 实现

<!--more-->
```go
// 定义一个节点结构体
type node struct {
    key int          // 节点键值
    value interface{}  // 节点储存数据值
    next []*node       // 节点所在层数的指针数组
}

// 跳表结构体定义
type SkipList struct {
    head *node            // 头节点
    maxLevels int         // 最大层数
    p float32             // 索引分布概率参数
    level int             // 当前层数
}

// 新建跳表节点的函数
func newSkipListNode(key int, value interface{}, level int) *node {
    return &node{
        key: key,
        value: value,
        next: make([]*node, level),   // 根据level新建next切片
    }
}

// 创建跳表的函数
func NewSkipList(p float32, maxLevels int) *SkipList {
    head := newSkipListNode(0, nil, maxLevels)     //新建头节点
    return &SkipList{head: head, maxLevels: maxLevels, p: p}  //返回跳表结构体指针
}

// 插入数据的函数
func (s *SkipList) Insert(key int, value interface{}) {
    current := s.head    // 设置当前节点为头节点
    update := make([]*node, s.maxLevels)  //创建更新数组用来记录每层需要更新的节点

    // 搜索待插入节点的位置，并记录每一层需要更新的节点
    for i := s.level - 1; i >= 0; i-- {
        // 找到i层上第一个键值小于要插入的key值的节点
        for current.next[i] != nil && current.next[i].key < key {
            current = current.next[i]
        }
        update[i] = current   // 将搜索到的节点记录在更新数组中
    }

    // 如果要插入的节点已经存在，直接更新其值即可
    if current.next[0] != nil && current.next[0].key == key {
        current.next[0].value = value
        return
    }

    // 根据索引分布概率参数随机生成新节点的层数
    newLevel := s.randomLevel()

    // 如果新建节点的层数大于当前跳表的层数，需要为更高的层次更新指针
    if newLevel > s.level {
        for i := s.level; i < newLevel; i++ {
            update[i] = s.head
        }
        s.level = newLevel
    }

    // 创建新的节点并依次与原来的节点相连
    newNode := newSkipListNode(key, value, newLevel)
    for i := 0; i < newLevel; i++ {
        newNode.next[i] = update[i].next[i]
        update[i].next[i] = newNode
    }
}

// 查找数据的函数
func (s *SkipList) Search(key int) interface{} {
    current := s.head

    // 从顶层向底层开始搜索
    for i := s.level - 1; i >= 0; i-- {
        // 在当前层上找到第一个键值大于等于要查询的key值的节点
        for current.next[i] != nil && current.next[i].key < key {
            current = current.next[i]
        }
    }

    // 检查找到的节点是否是要查找的key值节点
    if current.next[0] != nil && current.next[0].key == key {
        return current.next[0].value
    }

    // 若未找到，返回nil值
    return nil
}

// 删除数据的函数
func (s *SkipList) Delete(key int) {
    current := s.head      // 设置当前节点为头节点
    update := make([]*node, s.maxLevels)  // 创建更新数组用来记录每层需要更新的节点

    // 搜索待删除节点的位置，并记录每一层需要更新的节点
    for i := s.level - 1; i >= 0; i-- {
        // 找到i层上第一个键值小于要插入的key值的节点
        for current.next[i] != nil && current.next[i].key < key {
            current = current.next[i]
        }
        update[i] = current   // 将搜索到的节点记录在更新数组中
    }

    // 如果要删除的节点存在，则依次将它与前一个节点相连，直到所有的层数都删除
    if current.next[0] == nil || current.next[0].key != key {
        return
    }
    for i := 0; i < s.level; i++ {
        if update[i].next[i] == current.next[0] {
            update[i].next[i] = current.next[0].next[i]
        }
    }

    // 调整跳表的层数
    for s.level > 1 && s.head.next[s.level-1] == nil {
        s.level--
    }
}

// 随机生成节点层数的函数
func (s *SkipList) randomLevel() int {
    level := 1
    for rand.Float32() < s.p && level < s.maxLevels {
        level++
    }
    return level
}
```