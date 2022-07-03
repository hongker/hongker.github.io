---
title: Go数据结构与算法(07)-BTree
date: 2022-07-03 11:24:33
tags: algorithm
---
本文将介绍btree的原理与golang的实现。

## 什么是Btree
B树是一种平衡的多路查找树。树，可广泛用于磁盘访问。 M阶树顺序的B树最多可以有m-1个键和M个子树。 使用B树的主要原因之一是它能够在单个节点中存储大量键，并且通过保持树的高度相对较小来存储大键值。


- 概念
```
根节点：最顶层节点
叶子节点：没有子树的节点就是叶子节点
度数：在树中，每个节点的子节点（子树）的个数就称为该节点的度（degree）。
```

- 特点
```
1.每个叶结点具有相同的深度，也就是树的高度。
2.节点的元素以非降序排序，即x.k <= x.k1 <= ... <= x.kn。且 t-1 <=n <= 2t-1
```



## 使用场景
Btree主要用于提高访问数据的效率，比如数据库的索引文件。

## 实现

- 插入
```go
package btree

import "sort"

// Item是一个接口类型，含有一个Less方法，通过这个接口可以实现类似泛型的功能。
type Item interface {
	Less(than Item) bool
}

// IntItem 定义元素类型为int型
type IntItem int
// Less returns true if int(a) < int(b).
func (a IntItem) Less(b Item) bool {
	return a < b.(IntItem)
}

// Items 元素数组
type Items []Item
// find 查询元素对应的位置
// 比如数组为 [1,3,5], 查找3时，返回1, true;查找4时，返回2, false
func (items Items) find(t Item) (index int, found bool) {
	// search方法使用二分搜索去返回元素中最小的满足[0,n)中最小的满足f(i)函数为true的索引i，如果f(i)为true，则f(i+1)也为true
	i := sort.Search(len(items), func(i int) bool {
		return t.Less(items[i])
	})
	if i > 0 && !items[i-1].Less(t) {
		// 当上一个元素刚好和t相等时，返回下标和true
		return i - 1, true
	}

	// 返回第一个大于目标元素的数组索引
	return i, false
}
// insertAt 按指定位置插入一个Item
func (items *Items) insertAt(index int, item Item) {
	*items = append(*items, nil) // 扩展一个位置出来
	if index < len(*items) {     // 可能index==len，需要插入的位置就在最后一个
		copy((*items)[index+1:], (*items)[index:]) // 位置后面的往后挪一下
	}
	(*items)[index] = item // 覆盖指定位置的数据
}

// node 定义树的节点
type node struct {
	items    Items    // 元素数组
	children children // 子节点的指针数组
}
// split 将一个节点分裂成包含左右子节点的一个父节点，使得node的items和children减半，且父节点只保留中间一个元素
func (n *node) split(mid int) (Item, *node) {
	if len(n.items)-1 < mid {
		panic("error index")
	}

	upItem := n.items[mid]
	// 新增加一个节点
	newNode := &node{}
	// 将原节点的元素分一半给新节点
	newNode.items = append(newNode.items, n.items[mid+1:]...)
	// 原节点只保存一半
	n.items = n.items[:mid]

	// 子节点也要分半
	if n.children != nil {
		newNode.children = append(newNode.children, n.children[mid+1:]...)
		n.children = n.children[:mid+1]
	}

	return upItem, newNode
}

// maybeSplitChild 对children进行判断
// 1.node的items和children元素个数分别+1,保证不破坏btree的属性
// 2.保证了后续插入查询地柜下沉到node的某一个子树的时候，子树items未满
func (n *node) maybeSplitChild(childIndex, maxCap int) bool {
	l := len(n.children[childIndex].items)

	// 如果items已满，则进行一次分裂
	if l >= maxCap {
		upItem, newNode := n.children[childIndex].split(l / 2)

		// 将中间的那个数据项插入到childIndex的位置
		n.items.insertAt(childIndex, upItem)
		// 将新节点插入到children里，位置为childIndex+1
		n.children.insertAt(childIndex+1, newNode)

		return true
	} else {
		return false
	}
}

// insert 插入元素
func (n *node) insert(item Item, maxCap int) Item {
	// 查找元素合适的插入位置
	index, found := n.items.find(item)
	if found {
		return item
	}
	// 是叶子节点，直接将数据插入items
	if len(n.children) == 0 {
		n.items.insertAt(index, item)
		return nil
	}

	// 不是叶子节点，看看i处的子Node是否需要分裂， 这里的操作是为了保证后面进行插入下沉到子节点时，子节点n.children[index]一定未满
	if n.maybeSplitChild(index, maxCap) {
		// 分裂了，导致当前node的变化，需要重新定位
		// 保证插入查询能下沉到items符合范围的子树中
		inTree := n.items[index] // 获取新升级的item
		switch {
		case item.Less(inTree):
			// 要插入的item比分裂产生的item小，i没改变
		case inTree.Less(item):
			index++ // 要插入的item比分裂产生的item大，i++
		default:
			// 分裂升level的item和插入的item一致，替换
			out := n.items[index]
			n.items[index] = item
			return out
		}
	}

	// 递归插入到items符合插入范围的子树
	return n.children[index].insert(item, maxCap)
}

// children 子节点
type children []*node
// insertAt 按指定位置插入一个Item
func (c *children) insertAt(index int, n *node) {
	*c = append(*c, nil) // 扩展一个位置出来
	if index < len(*c) {     // 可能index==len，需要插入的位置就在最后一个
		copy((*c)[index+1:], (*c)[index:]) // 位置后面的往后挪一下
	}
	(*c)[index] = n // 覆盖指定位置的数据
}

type btree struct {
	degree uint  // 树的度
	root   *node // 根节点
}

// MinCap 返回节点元素个数的下界
func (tree *btree) MinCap() int {
	return int(tree.degree - 1)
}

// MaxCap 返回节点元素的上届
func (tree *btree) MaxCap() int {
	return int(2*tree.degree - 1)
}

// Insert 插入元素
func (tree *btree) Insert(item Item) Item {
	//如果是空树，则创建根后插入
	if tree.root == nil {
		tree.root = new(node)
		tree.root.items = append(tree.root.items, item)
		return nil
	}

	// 判断根节点的元素个数是否已达上限
	// 如果根结点已满，先对根节点进行分裂处理
	if len(tree.root.items) >= tree.MaxCap() {
		midIndex := len(tree.root.items) / 2
		upItem, newNode := tree.root.split(midIndex)
		newRoot := new(node)
		newRoot.items = append(newRoot.items, upItem)
		newRoot.children = append(newRoot.children, tree.root, newNode)
		tree.root = newRoot
	}

	return tree.root.insert(item, tree.MaxCap())
}

func New(degree uint) *btree  {
	return &btree{degree: degree}
}
```
- 删除

- 查找


