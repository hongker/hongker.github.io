---
title: Go数据结构与算法(06)-Bitmap
date: 2022-06-29 20:43:36
tags: algorithm
---
本文将介绍Bitmap的使用场景与算法实现。

## 什么是Bitmap
>所谓的Bit-map就是用一个bit位来标记某个元素对应的Value， 而Key即是该元素。比如将第4位字节设为1，就代表某个集合已存在ID为4的记录。

## 适用场景
由于存储数据的是非常小的字节，所以bitmap能节省超大空间，可以用来处理大量数据的排序、查询以及去重。

## 实现
### Bitmap

- 实现
通过一个uint64的变量可以存储64位数据。
```go
package bit
// Bitmap 支持存储64个字节
type Bitmap uint64

func (b Bitmap) Set(pos uint64) Bitmap {
	// 1.根据位移运算，得到一个指定位置为1的数字，比如pos为4,则得到的是 0001 0000
	// 2.通过按位异或运算将值设置上去
	return b | (1 << pos)
}

func (b Bitmap) Clear(pos uint64) Bitmap {
	// 1.位移运算得到指定位置为1的数字，如：0001 0000
	// 2.取反,如：1110 1111
	// 3.按位与,如b为 0011 0000,最终结果为: 0010 0000
	return b & ^(1 << pos)
}

func (b Bitmap) Has(pos uint64) bool {
	// 1.位移运算得到指定位置为1的数字，如：0001 0000
	// 2.按位与,如b为 0011 0000,得到结果为：0001 0000
	// 3.只要结果不为0,则证明该位上的值为1
	return (b & (1 << pos)) != 0
}
// toNums 读取数据
func (b Bitmap) toNums(offset uint64, nums *[]uint64) {
	for i := uint64(0); i < 64; i++ {
		if b.Has(i) {
			*nums = append(*nums, i+offset)
		}
	}
}

func (b Bitmap) Count() int {
	val := b
	// 将b的二进制分为32等份，计算每个部分含1的个数。
	// 示例：10 11 10 01 00 11 10 11 00 01 10 01 10 00 01 00 10 11 10 01 00 11 10 11 00 01 10 01 10 00 01 00
	// 经过运算：01 10 01 01 00 10 01 10 00 01 01 01 01 00 01 00 01 10 01 01 00 10 01 10 00 01 01 01 01 00 01 00
	val = (val & 0x5555555555555555) + ((val >> 1) & 0x5555555555555555)
	// 后面的计算就是一步步地将每部分的值加起来
	// 第1部分+第2部分，第3部分+第4部分，...第31部分+第32部分
	val = (val & 0x3333333333333333) + ((val >> 2) & 0x3333333333333333)
	// 第1部分+第2部分，第3部分+第4部分，...第15部分+第16部分
	val = (val & 0x0F0F0F0F0F0F0F0F) + ((val >> 4) & 0x0F0F0F0F0F0F0F0F)
	// 第1部分+第2部分，第3部分+第4部分，...第7部分+第8部分
	val = (val & 0x00FF00FF00FF00FF) + ((val >> 8) & 0x00FF00FF00FF00FF)
	// 第1部分+第2部分，第3部分+第4部分
	val = (val & 0x0000FFFF0000FFFF) + ((val >> 16) & 0x0000FFFF0000FFFF)
	// 第1部分+第2部分
	val = (val & 0x00000000FFFFFFFF) + ((val >> 32) & 0x00000000FFFFFFFF)

	// 更为精简的操作
	//val -= (val >> 1) & 0x5555555555555555
	//val = (val>>2)&0x3333333333333333 + val&0x3333333333333333
	//val += val >> 4
	//val &= 0x0f0f0f0f0f0f0f0f
	//val *= 0x0101010101010101

	return int(val)
}
```

<!--more-->

- 使用
```go
package bit

import (
	"github.com/stretchr/testify/assert"
	"testing"
)

func TestBitmap(t *testing.T) {
	b := Bitmap(0)
	b = b.Set(0)
	assert.True(t, b.Has(0))

	b = b.Set(4)
	assert.True(t, b.Has(4))
	assert.Equal(t, 2, b.Count())
	b = b.Clear(0).Clear(4)
	assert.False(t, b.Has(0))
	assert.False(t, b.Has(4))
	assert.Equal(t, 0, b.Count())

	b = b.Set(63)
	assert.True(t, b.Has(63))
	assert.Equal(t, 1, b.Count())
}
```

## BitmapArray
一个Bitmap只能存储64个数据，那么要想存储更多的数据，就需要用到多个Bitmap组成的BitmapArray。
- 实现
```go
package bit

import "errors"

// BitmapArray bitmap数组，支持存储n*64位bit
type BitmapArray struct {
	blocks  []Bitmap // 分块
	lowest  uint64   // 最低位
	highest uint64   // 最高位
	empty   bool     // 是否为空
}

const (
	mask = 64 // 每一个分块的大小
)

// getIndexAndRemainder 计算分块位置与bit位置
func getIndexAndRemainder(k uint64) (uint64, uint64) {
	return k / mask, k % mask
}
// Capacity 计算容量
func (ba *BitmapArray) Capacity() uint64 {
	// 也就是分块数量*64
	return uint64(len(ba.blocks)) * mask
}

// Set 设置位置为1
func (ba *BitmapArray) Set(pos uint64) error {
	if pos >= ba.Capacity() {
		return errors.New("out of range")
	}

	if ba.empty { // 判断是否为空
		ba.lowest = pos
		ba.highest = pos
		ba.empty = false
	} else {
		if pos < ba.lowest {
			ba.lowest = pos
		} else if pos > ba.highest {
			ba.highest = pos
		}
	}

	idx, bit := getIndexAndRemainder(pos)
	ba.blocks[idx] = ba.blocks[idx].Set(bit)

	return nil
}

// Empty 判断是否为空
func (ba *BitmapArray) Empty() bool {
	return ba.empty
}

// Has 判断是否有值
func (ba *BitmapArray) Has(pos uint64) bool {
	if pos >= ba.Capacity() {
		return false
	}

	idx, bit := getIndexAndRemainder(pos)
	return ba.blocks[idx].Has(bit)
}

// Clear 清除
func (ba *BitmapArray) Clear(pos uint64) {
	if pos >= ba.Capacity() {
		return
	}
	idx, bit := getIndexAndRemainder(pos)
	ba.blocks[idx].Clear(bit)
}

// Count 统计总数
func (ba *BitmapArray) Count() uint64 {
	if ba.empty {
		return 0
	}

	var count uint64
	for _, block := range ba.blocks {
		count += uint64(block.Count())
	}
	return count
}

// ToNums 将bitmap里存储的数据转成数字数组
func (ba *BitmapArray) ToNums() []uint64 {
	nums := make([]uint64, 0, ba.highest-ba.lowest/4)
	for i, block := range ba.blocks {
		block.toNums(uint64(i)*s, &nums)
	}

	return nums
}

// NewBitmapArray 根据指定最大长度，初始化bitmap数组
func NewBitmapArray(size uint64) *BitmapArray {
	idx, bit := getIndexAndRemainder(size)
	if bit > 0 { // 当余数不为0时，需要多实例化一个bitmap
		idx++
	}

	ba := &BitmapArray{
		blocks:  make([]Bitmap, idx),
		lowest:  0,
		highest: 0,
		empty:   true,
	}

	return ba
}

```

- 示例
利用BitmapArray来对大量数据进行排序
```go

func TestBitmapArray(t *testing.T) {
	// 模拟随机生成10万个数字
	n := 100000
	ba := NewBitmapArray(uint64(n))
	for i := 0; i < n; i++ {
		ba.Set(uint64(rand.Int63n(1000000)))
	}

	// 得到的结果就是升序排列后的数组
	nums := ba.ToNums()
	fmt.Println(nums)
}
```