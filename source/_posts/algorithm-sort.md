---
title: Go实现算法系列01-排序
date: 2020-04-09 16:15:58
tags: 算法
---
<!-- toc -->
本文介绍通过go实现常见的排序算法。

## 说明
虽然go的内置sort包实现了相关排序算法，但是我们要了解其原理，通过自己实现一次，加深印象。

## 冒泡排序
- 原理：比较两个相邻的元素，将值大的元素交换到右边。
时间复杂度为O(n^2)
- Demo
```go
// BubbleSort 冒泡排序
// 比较两个元素，将最大的元素放在右边
func BubbleSort(a [] int)  {
	// 从最后一个元素进行排序
	for first := len(a) - 1;first>=1;first-- {
		// 遍历剩余元素
		for second := first - 1;second>=0;second-- {
			// 找到大的
			if a[first] < a[second] {
				// 交换
				a[first], a[second] = a[second], a[first]
			}
		}
	}
}
```
- Test
```go
import (
	"fmt"
	"testing"
)

var a = []int{10,1,35,61,89,36,55}
func TestBubbleSort(t *testing.T) {
	BubbleSort(a)
	fmt.Println(a)
}
```
- 输出:
```
[1 10 35 36 55 61 89]
```
<!--more-->

## 选择排序
- 原理：首先在未排序序列中找到最小（大）元素，存放到排序序列的起始位置，然后，再从剩余未排序元素中继续寻找最小（大）元素，然后放到已排序序列的末尾。以此类推，直到所有元素均排序完毕
时间复杂度为O(n^2)
- Demo
```go
// SelectionSort 选择排序
// 找到最小的值，分别放到数组前面。
func SelectionSort(a []int)  {
	// 遍历数据
	for i := 0; i<len(a);i++ {
		// 定义最小值的下标
		min := i
		// 遍历剩下的元素，找到最小值的下表
		for j :=i+1; j< len(a);j++ {
			if a[j] < a[min] {
				min = j
			}
		}

		// 交换数据
		if min != i {
			a[min], a[i] = a[i], a[min]
		}
	}
}
```
- Test
```go
import (
	"fmt"
	"testing"
)

var a = []int{10,1,35,61,89,36,55}
func TestSelectionSort(t *testing.T) {
	SelectionSort(a)
	fmt.Println(a)
}
```
- 输出
```
[1 10 35 36 55 61 89]
```
## 快速排序
- 原理：通过一趟排序将待排记录分隔成独立的两部分，其中一部分记录的关键字均比另一部分的关键字小，则可分别对这两部分记录继续进行递归排序，以达到整个序列有序。

- Demo
```go
// QuickSort 快速排序
// low : 数组的左侧下标, high: 数组的右侧下标
func QuickSort(a []int, low int, high int)  {
	if low >= high { // 已排序完成
		return
	}

	pivot := a[low] // 指定基准值
	right := high
	left := low
	for {
		// 在不超过起点(low)的情况下,如果右边的值比基准值大，则继续遍历，为了找到比基准值小的数据
		for a[right] >= pivot && right > low {
			right--
		}
		// 在不超过终点(high)的情况下，如果左边的值比基准值小，则继续遍历，为了找到比基准值大的数据
		for a[left]<= pivot && left < high {
			left++
		}

		if left < right { // 此条件表示找到对应的数据，满足可交换的条件，比基准值大的放右边，比基准值小的放左边
			a[left], a[right] = a[right], a[left]
		}else {
			// 表示已遍历完成，退出循环
			break
		}
	}
	// 这里很重要，右侧值与基准值交换，将基准值归位，此刻数组已形成右侧为比基准值小的数据，右侧为比基准值大的数据
	a[low], a[right] = a[right], a[low]
	QuickSort(a, low, right)
	QuickSort(a, right+1, high)
}
```

## 插入排序
- 原理：通过构建有序序列，对于未排序数据，在已排序序列中从后向前扫描，找到相应位置并插入。

- Demo
```go
// InsertSort 插入排序
func InsertSort(a []int)  {
	// 前节点、当前值
	var preIndex, current int
	for i:=1; i<len(a);i++  {
		preIndex = i-1
		current = a[i]
		// 遍历，当前一个值大于当前值时
		for preIndex>=0 && a[preIndex] > current {
			// 将前节点值向后挪一位
			a[preIndex+1] = a[preIndex]
			// 继续向前遍历
			preIndex--
		}
		// 将当前值插入
		a[preIndex+1] = current
	}
}
```
