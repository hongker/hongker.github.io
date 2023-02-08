---
title: Go数据结构与算法(14)-栈(Stack)
date: 2023-02-08 22:06:57
tags: algorithm
---

本文主要介绍栈的概念与实现，以及应用场景

## 概念
`栈 Stack`是一种遵循`先入后出`数据操作规则的线性数据结构。

- 栈顶：栈的最顶部元素
- 栈底：栈的最底部元素
- 入栈：将元素添加到栈顶
- 出栈：删除栈顶元素

## 应用场景
- 浏览器前进与后退
- 操作的撤销与反撤销


## 实现
常用操作

| 方法      | 说明 | 时间复杂度 |
| ----------- | ----------- | -----------|
| Push      | 元素入栈       | O(1) |
| Pop   | 元素出栈        |O(1) |
| Size   | 获取栈的长度        |O(1) |
| Empty   | 判断栈是否为空        |O(1) |
| Peek   | 读取栈顶元素        |O(1) |

<!--more-->

```go
package main

import "fmt"

func main() {
	stack := NewStack(10)
	stack.Push(1, 2, 3)
	fmt.Printf("size=%d\n", stack.Size())
	fmt.Printf("top=%v\n", stack.Peek())

	for {
		if stack.Empty() {
			break
		}

		fmt.Printf("pop=%v\n", stack.Pop())
	}

	fmt.Printf("stack empty: %v\n", stack.Empty())

}

type Stack struct {
	items []any
}

// Push pushes many items into the stack
func (stack *Stack) Push(elem ...any) {
	stack.items = append(stack.items, elem...)
}

// Pop return top item of the stack and remove it
func (stack *Stack) Pop() (elem any) {
	if stack.Empty() {
		// return nil when stack is empty
		return
	}
	elem = stack.items[len(stack.items)-1]
	stack.items = stack.items[:len(stack.items)-1]
	return
}

// Size returns the size of the stack
func (stack *Stack) Size() int {
	return len(stack.items)
}

// Empty returns true if stack is empty
func (stack *Stack) Empty() bool {
	return 0 == stack.Size()
}

// Peek returns top item of the stack
func (stack *Stack) Peek() (elem any) {
	return stack.items[len(stack.items)-1]
}

func NewStack(cap int) *Stack {
	return &Stack{
		items: make([]any, 0, cap),
	}
}

```
