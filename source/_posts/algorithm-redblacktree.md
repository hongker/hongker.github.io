---
title: 数据结构与算法(16)-红黑树(Red-Black Tree)
date: 2023-04-25 19:33:34
tags: algorithm
---
我们今天来介绍下红黑树，根据此文快速掌握红黑树这种数据结构的原理与实现

## 什么是红黑树
>红黑树是一种常用的自平衡二叉搜索树，与 AVL 树、Treap 等数据结构相比，其插入和删除操作的时间复杂度更优秀，而且难度较低。它能够在任何情况下保持平衡，因此在高效的查找、插入、删除等操作中得到广泛应用。

## 特性
- 每个节点要么是红色，要么是黑色；
- 根节点是黑色的；
- 所有叶子节点（即空节点）都是黑色的；
- 如果一个节点是红色的，则它的两个子节点都是黑色的；也就是说不能出现连续的红节点；
- 从任意一个节点到其每个叶子的所有路径都包含相同数目的黑色节点。

## 实现
以下是golang版的红黑树实现代码：
<!--more-->

```go
package rbtree

// 定义节点结构体
type node struct {
	key, value  int
	left, right *node
	color       bool // 节点颜色，true 为红色，false 为黑色
}

// 定义红黑树结构体
type RBTree struct {
	root *node
}

// 初始化红黑树
func NewRBTree() *RBTree {
	return &RBTree{nil}
}

// 判断节点颜色是否为红色
func isRed(n *node) bool {
	if n == nil {
		return false // 叶子节点一定是黑色，即 nil 为黑色
	}
	return n.color
}

// 左旋操作
func rotateLeft(n *node) *node {
	x := n.right
	n.right = x.left
	x.left = n
	x.color = n.color
	n.color = true
	return x
}

// 右旋操作
func rotateRight(n *node) *node {
	x := n.left
	n.left = x.right
	x.right = n
	x.color = n.color
	n.color = true
	return x
}

// 颜色翻转
func flipColors(n *node) {
	n.color = !n.color
	n.left.color = !n.left.color
	n.right.color = !n.right.color
}

// 插入节点操作
func (t *RBTree) Put(key, value int) {
	t.root = put(t.root, key, value)
	t.root.color = false // 根节点必须是黑色
}

func put(n *node, key, value int) *node {
	// 如果节点为空，则插入新节点并返回
	if n == nil {
		return &node{key, value, nil, nil, true} // 新节点一定是红色的
	}

	// 遍历树进行插入操作
	if key < n.key { // 插入左子节点
		n.left = put(n.left, key, value)
	} else if key > n.key { // 插入右子节点
		n.right = put(n.right, key, value)
	} else { // 更新已有节点的值
		n.value = value
	}

	// 节点旋转或颜色翻转等操作，满足红黑树的特性
	if isRed(n.right) && !isRed(n.left) {
		n = rotateLeft(n)
	}
	if isRed(n.left) && isRed(n.left.left) {
		n = rotateRight(n)
	}
	if isRed(n.left) && isRed(n.right) {
		flipColors(n)
	}

	return n
}

// 查找节点操作
func (t *RBTree) Get(key int) int {
	n := t.root
	for n != nil {
		if key < n.key {
			n = n.left
		} else if key > n.key {
			n = n.right
		} else {
			return n.value
		}
	}
	return -1 // 未找到则返回值为-1
}

```