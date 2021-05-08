---
title: Go数据结构与算法(04)-分治算法
date: 2021-05-09 05:44:13
tags: algorithm
---
算法思想主要有分治算法、回溯法、贪心算法、动态规划法等，本文介绍分治算法。

## 定义
>将一个复杂的问题分成两个或更多个相同或相似的子问题，求得子问题的解，再合并就得到原问题的解。其核心思想是“分而治之”。

## 基本思路
- 分：将一个问题分解成多个相似且足够小的子问题。
- 治：通常通过递归的方式，对子问题进行求解。
- 合并：将子问题的解合并得到原问题的解。

## 典型算法
### 快速排序
- 算法描述
```
1.选择数组的第一个数作为基数，将后面的数与之比较，较小的放右边，较大的放左边，分成2个数组。
2.通过递归的方式对上述两个数组再次拆分。
3.直到无法拆分为止，整个数组就变成有序的数组。
```
快速排序的时间复杂度是 `O(nlogn)`。
- 实现
```go
// QuickSort 快速排序,left为数组的第一个下标，right为数组的最后一个下标
func QuickSort(items []int, left, right int) {
    low := left // 临时变量,低位下标
    high := right // 临时变量，高位下标
    if low >  high { // 无法再分，作为截止条件
        return
    }
    flag := items[low] // 设置基数，用于对比
    for low < high { // 循环条件
        for low < high && items[high] >= flag { // 从右往左遍历，直到发现比基数小的数
            high--
        }
        items[low] = items[high] // 将较小的数放在低位地址
        for low < high && items[low] <= flag { // 从左往右遍历，直到发现比基数大的数
            low++
        }
        items[high] = items[low] // 将较大的树放在高位地址
    }
    items[low] = flag // 经过一轮遍历后，将基数赋值给低位地址，形成左边为比基数小的数组，右边为比基数大的数组
    QuickSort(items, left, low-1) // 递归，分解
    QuickSort(items, low+1, right) // 递归，分解
}
```

- 使用
```go
func main() {
    items := []int{5,1,3,9,8,2,6,4,7}
    QuickSort(items, 0, len(items)-1)
    fmt.Println(items)
}
```

### 归并排序
- 算法描述
```
1.分：将n个元素的序列分成n/2个子序列，每个子序列含两个元素。
2.治：比较子序列，形成有序数组。
3.合并：通过递归的方式，对两个数组进行排序并合并成一个数组。直到全部完成合并。
```

- 实现
```go
// 自顶向下归并排序，排序范围在[left,right)的数组
func MergeSort(items []int, left, right int)  {
	// 元素数量大于1时才进入递归
	if right - left > 1 {
		// 将数组一分为二
		middle := (left + right) / 2

		// 递归对左边的数组进行排序
		MergeSort(items, left, middle)
		// 递归对右边的数组进行排序
		MergeSort(items, middle, right)
		// 合并
		merge(items, left, middle, right)
	}

}
// 归并
func merge(arr []int, low, middle, high int)  {
	leftLen := middle-low // 左数组的长
	rightLen := high-middle // 右数组的长
	// 使用临时数组来存储将要排序的元素
	temp := make([]int, 0, leftLen+rightLen)

	// 遍历对比两个数组的每个元素
	l, r := 0, 0
	for l < leftLen && r < rightLen{
		lValue := arr[low+l] // 左边数组的元素
        rValue := arr[middle+r] // 右边数组的元素
        // 比较找到较小的，先放入临时数组
		if lValue < rValue {
			temp = append(temp, lValue)
			l++
		}else {
			temp = append(temp, rValue)
			r++
		}
	}

	// 将剩下的元素追加到临时数组里
	temp = append(temp, arr[low + l:middle]...)
	temp = append(temp, arr[middle+r:high]...)

	// 此刻临时数组已变成有序的，将其元素复制到原数组里
	for idx, val := range temp {
		arr[low+idx] = val
	}
}
```

## 思考
该如何在实际环境中使用到这种分治思想呢？
