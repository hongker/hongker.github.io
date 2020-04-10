---
title: Golang代码优化13-枚举
date: 2020-04-10 13:20:31
tags: golang
---
本文介绍golang中枚举的使用。

## 什么是枚举类型
>是一组命名的常数，常量值可以是连续的，也可以是断续的。

## 说明
Golang中并没有真正的枚举类型，往往都是自己定义常量，当作枚举类型来使用。

## 简单用法
下面以订单状态来举例说明go中枚举类型的使用方式。

- 定义枚举常量
```go
type OrderState uint
const(
	// 待确认
	OrderConfirmed OrderState = iota + 1
	// 待支付
	OrderPaying
	// 待发货
	OrderShipping
	// 待收货
	OrderReceiving
	// 已收货
	OrderReceived
	// 已完成
	OrderCompleted
)
func main() {
	var state =6 // 模拟从数据库中获取到的状态
	if OrderState(state) == OrderCompleted {
		fmt.Println("订单已完成")
	}
}
```

## 扩展

- 对于枚举常量，我们时常也需要展示类型的名称，如展示订单状态名称
```go
var (
	OrderStateItems = map[OrderState]string{
		OrderConfirmed: "待确认",
		OrderPaying:    "待支付",
		OrderShipping:  "待发货",
		OrderReceiving: "待收货",
		OrderReceived:  "已收货",
		OrderCompleted: "已完成",
	}
)
// OrderStateEnum 枚举类
type OrderStateEnum struct {
	// v 值
	v OrderState
}
// Name 获取订单状态名称
func (e OrderStateEnum) Name() string  {
	return OrderStateItems[e.v]
}

func main() {
	var e = OrderStateEnum{OrderCompleted}
	fmt.Println(e.Name())
}
```

- 检查枚举是否合法
```go
// IsValid 检查枚举是否合法
func (e OrderStateEnum) IsValid() bool {
	_, ok := OrderStateItems[e.v]
	return ok
}
func main() {
	var e = OrderStateEnum{OrderCompleted}
	fmt.Println(e.IsValid())
}
```

