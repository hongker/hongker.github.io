---
title: Go数据结构与算法(08)-B+Tree
date: 2022-07-09 09:47:36
tags: algorithm
---

前面介绍了BTree算法的实现，实际在数据库索引里用的是B+Tree。

## 概念
>B+树是B树的一个变种算法，两者都是平衡多路查找树。

- 与BTree的区别
```
1.非叶子节点只存储键值信息，数据记录都存放在叶子节点中。
2.所有叶子节点之间都有一个链指针。
3.非叶子节点的关键字的个数与其子树的个数相同。
```


- 优点
```
1.IO效率更高：内部结点只有关键字，没有数据，一个结点可以容纳更多的关键字。查询时一次性读入内存中的关键字也就越多
2.查询效率稳定：因为数据全在叶子节点上，所以查询关键字的次数=树的深度，导致每一个数据的查询效率相当
3.方便范围查询：因为叶子节点之间是有链表的方式关联，所以可以直接依次查询指定范围的数据。不必重复索引。
```

## 实现

- 插入数据
```
1.插入数据时首先定位到数据所在的叶子节点，然后将数据插入到该节点，插入数据后的数组依然成正序排列。
2.若插入元素后该节点的关键字数量<=阶数，则直接完成插入操作即可。
3.若插入的关键字为该节点的最大值，则需要修改父节点的索引值。
4.若插入元素后，该节点的关键字数量>阶数，则需要想该节点分裂成两个节点，关键字的个数分别位floor((M+1)/2)和ceil((M+1)/2)
5.若分裂几点后导致父节点的关键字数目>阶数，则父节点也要进行相应的分裂操作。
```

- 查找数据
```
1.根据关键字，找到对应的叶子节点。
2.对叶子节点进行遍历，找到关键字对应的数据。
```

- 删除数据
```
1.当删除某节点中最大或者最小的关键字，就会涉及到更改其双亲节点一直到根节点中所有索引值的更改。
2.删除关键字后，如果若当前节点中的关键字>=阶数/2,则直接完成删除操作
3.在删除关键字后，如果导致节点中的关键字个数<阶数/2,若其兄弟节点中含有多余的关键字，可以从兄弟节点借关键字
4.在删除关键字后，如果导致其节点中的关键字个数<阶数/2,并且其兄弟节点没有多余的关键字，则需要同其兄弟节点进行合并
5.节点合并后，需要修改父节点的关键字个数，若父节点的关键字个数<阶数/2,则递归处理。
```

<!--more-->
- 实现
```go
package tree

// Item 数据项
type Item struct {
	key int64 // 关键字
	value interface{} // 值
}

type  Items []Item

// BPlusTree 实现b+tree
type BPlusTree struct {
	root Node // 根节点
	width int // 阶数(节点的关键字最大总量)
	size int // 数据个数
}


func NewBPlusTree(width int) *BPlusTree {
	// 初始化的根节点是一个空的叶子节点
	return &BPlusTree{
		root:    &leafNode{
			items: make(Items, 0, width),
			next: nil,
		},
		width: width,
		size:    0,
	}
}


func (tree *BPlusTree) Insert(key int64, value interface{}) {
	// 插入数据
	success := tree.root.insert(tree.width, key, value)
	if success {
		tree.size++
	}

	// 检查是否满足分裂条件
	if tree.root.needSplit(tree.width) {
		tree.root = split(tree.width, nil, tree.root)
	}
}


// Find 根据关键字查询数据
func (tree *BPlusTree) Find(key int64) interface{}{
	return tree.root.find(key)
}

func (tree *BPlusTree) Size() int {
	return tree.size
}


// Node 定义树的节点(包括叶子节点和非叶子节点)
type Node interface {
	// search 通过二分查找法，查找关键字的位置
	search(key int64) int
	// insert 插入元素
	insert(width int, key int64, value interface{}) bool
	// needSplit 判断节点是否满足分裂条件
	needSplit(width int) bool
	// split 节点分裂得到中间值，左子树以及右子树
	split() (int64, Node, Node)
	// find 查询数据
	find(key int64) interface{}
}

// leafNode 叶子节点
type leafNode struct {
	items Items // 数据项
	next Node // 临近节点
}


func (node* leafNode) search(key int64) int {
	low, high := 0, len(node.items) - 1
	var mid int
	for low <= high {
		mid = (low + high) / 2
		if key == node.items[mid].key {
			return mid
		}else if key < node.items[mid].key {
			high = mid - 1
		}else {
			low = mid + 1
		}
	}

	return low
}

func (node * leafNode) find(key int64) interface{}  {
	idx := node.search(key)
	target := node
	if idx == len(node.items) {
		if node.next == nil {
			return nil
		}
		target = node.next.(*leafNode)
	}
	for _, item := range target.items {
		if item.key == key {
			return item.value
		}
	}
	return nil

}

func (node* leafNode) insert(width int, key int64, value interface{}) bool {
	idx := node.search(key)
	item := Item{key: key, value: value}
	if idx == len(node.items) {
		// 当关键字大于节点中所有的关键字时，直接插入
		node.items = append(node.items, item)
		return true
	}

	// 当关键字已存在，直接替换
	if node.items[idx].key == key {
		node.items[idx] = item
		return false
	}

	// 按制定位置插入
	node.items = append(node.items, Item{})
	copy(node.items[idx:], node.items[idx+1:])
	node.items[idx] = item
	return true


}

func (node* leafNode) needSplit(width int) bool {
	return len(node.items) > width
}

func (node * leafNode) split() (int64, Node, Node)  {
	// 当关键字个数小于2时，不分裂
	if len(node.items) < 2 {
		return 0, nil, nil
	}

	// 取中间的关键字
	mid := len(node.items) / 2
	item := node.items[mid]

	// 构造左右子树
	leftItems := make(Items,  mid, cap(node.items))
	rightItems := make(Items,	 len(node.items) - mid, cap(node.items))

	// 给左右子树的关键字数组赋值
	copy(leftItems, node.items[:mid])
	copy(rightItems, node.items[mid:])
	leftNode := &leafNode{
		items: leftItems,
		next: node,
	}
	node.items = rightItems
	return item.key, leftNode, node

}

// internalNode 非叶子节点
type internalNode struct {
	keys Keys // 关键字数组
	children Children // 子节点数组
}

func (node* internalNode) search(key int64) int {
	return node.keys.search(key)
}

func (node* internalNode) insert(width int, key int64, value interface{}) bool {
	idx := node.keys.search(key)
	var child Node
	if idx == len(node.keys) {
		// 当目标关键字大于所有节点里的关键字，直接在最后一个节点里插入即可
		child = node.children[len(node.children)-1]
	}else {
		// 节点里存的关键字比对应子节点的关键字要大
		match := node.keys[idx]
		if  key >= match {
			child = node.children[idx+1]
		}else {
			child = node.children[idx]
		}
	}

	success := child.insert(width, key, value)
	if !success {
		return false
	}

	// 插入数据后，检查子节点是否需要分裂
	if child.needSplit(width) {
		split(width, node, child)
	}
	return true

}

func (node* internalNode) needSplit(width int) bool {
	return len(node.children) > width
}

func (node *internalNode) find(key int64) interface{}  {
	idx := node.keys.search(key)
	var child Node
	if idx == len(node.keys) {
		// 当目标关键字大于所有节点里的关键字，直接在最后一个节点里插入即可
		child = node.children[len(node.children)-1]
	}else {
		// 节点里存的关键字比对应子节点的关键字要大
		match := node.keys[idx]
		if  key >= match {
			child = node.children[idx+1]
		}else {
			child = node.children[idx]
		}
	}
	return child.find(key)
}

func (node* internalNode) split() (int64, Node, Node) {
	if len(node.keys) < 3 {
		return 0, nil, nil
	}

	mid := len(node.keys) / 2
	k := node.keys[mid]

	// 构造左右子树
	leftKeys := make(Keys,  mid, cap(node.keys))
	rightKeys := make(Keys,	 len(node.keys) - mid - 1, cap(node.keys))

	// 分裂关键字数组
	copy(leftKeys, node.keys[:mid])
	copy(rightKeys, node.keys[mid+1:])

	// 分裂子节点数组
	left, right := node.children.splitAt(mid+1)
	leftNode := &internalNode{
		keys:     leftKeys,
		children: left,
	}

	node.keys = rightKeys
	node.children = right

	return k, leftNode, node
}


type  Keys []int64

func (keys *Keys) insertAt(pos int, key int64)   {
	if pos == len(*keys) {
		(*keys) = append((*keys), key)
		return
	}
	(*keys) = append((*keys), 0)
	copy((*keys)[pos:], (*keys)[pos+1:])
	(*keys)[pos] = key
}

func (keys Keys) search(key int64) int {
	low, high := 0, len(keys) - 1
	var mid int
	for low <= high {
		mid = (low + high) / 2
		if key == keys[mid] {
			return mid
		}else if key < keys[mid] {
			high = mid - 1
		}else {
			low = mid + 1
		}
	}

	return low
}

type Children []Node
func (children *Children) insertAt(pos int, node Node)   {
	if pos == len(*children) {
		(*children) = append((*children), node)
		return
	}
	(*children) = append((*children), nil)
	copy((*children)[pos:], (*children)[pos+1:])
	(*children)[pos] = node
}
```