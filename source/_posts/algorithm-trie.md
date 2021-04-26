---
title: Go数据结构与算法(03)-字典树
date: 2021-04-26 07:36:09
tags: algorithm
---
本文将介绍字典树的相关知识与Golang的实现方式。

## Trie
在计算机科学中，trie，又称前缀树或字典树，是一种有序树，用于保存关联数组，其中的键通常是字符串。与二叉查找树不同，键不是直接保存在节点中，而是由节点在树中的位置决定。一个节点的所有子孙都有相同的前缀，也就是这个节点对应的字符串，而根节点对应空字符串。

Trie的典型应用是用于统计，排序和保存大量的字符串（但不仅限于字符串），所以经常被搜索引擎系统用于文本词频统计、敏感词检测、文本提示等。它的优点是：利用字符串的公共前缀来减少查询时间，最大限度地减少无谓的字符串比较，查询效率比哈希树高。


## 结构
如果插入数据为`sex`,`son`,`sleep`, `some`,则树结构如下所示：
- Trie:
```
s
├── e ── x
├── o ── n
│   └─ m ── e
└── l ── e ── e ── p
```


<!--more-->

## 实现：
```go
const(
	maxCap = 26 // 只存26个英文字母，可根据实际需求更改大小
)

// Trie 字典树
type Trie struct {
	// 子节点
	children map[rune]*Trie
	// 节点类型
	isWord bool
}

// Insert 插入元素
func (trie *Trie) Insert(word string) {
	for _, w := range word {
		if trie.children[w] == nil { // 不存在子节点
			// 初始化新的子节点
			node := new(Trie)
			node.children = make(map[rune]*Trie, maxCap)
			trie.children[w] = node
		}
		// 存在子节点,继续遍历
		trie = trie.children[w]
	}
	// 遍历结束,设置结束标识
	trie.isWord = true
}
// Search 查询
func (trie *Trie) Search(word string) bool{
	// 遍历字符串
	for _, w := range word {
		// 如果发现有字符不存在，则说明不匹配
		if trie.children[w] == nil {
			return false
		}
		// 继续遍历
		trie = trie.children[w]
	}
	// 遍历完整个字符后，且同时满足字符串已结束，说明匹配成功
	// 如果存在sleep，而搜索的sle,则为false
	return trie.isWord
}

// Delete 删除
func (trie *Trie) Delete(word string)  {
	// 遍历字符串
	for _, w := range word {
		// 如果发现有字符不存在，则说明不匹配
		if trie.children[w] == nil {
			return
		}
		// 继续遍历
		trie = trie.children[w]
	}

	trie.isWord = false
}

// New 实例化
func New() *Trie {
	return &Trie{
		children:  make(map[rune]*Trie, maxCap),
		isWord: false,
	}
}
```

使用如下：
```go
func main() {
    trie := New()
	words := []string{"sex","sleep","son", "some"}
	for _, word := range words {
		trie.Insert(word)
    }
    fmt.Println(trie.Search("son"))
}
```

## 备注
因为trie内部存储是用的原生map,所以当如果在并发场景下可能出现map的并发读写错误。解决方案可以是通过读写锁控制。