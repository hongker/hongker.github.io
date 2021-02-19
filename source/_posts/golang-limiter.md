---
title: Golang代码优化21-限流器(Limiter)
date: 2021-02-18 17:39:59
tags: golang
---
在实际的场景中，正常的请求不会造成系统异常。但我们也需要防止异常高并发场景导致整个系统崩溃的情况出现。下面介绍限流器的使用。

## 为什么需要限流器
>用户增长过快、热门业务或者爬虫等恶意攻击行为致使请求量突然增大，比如学校的教务系统，到了查分之日，请求量涨到之前的 100 倍都不止，没多久该接口几乎不可使用，并引发连锁反应导致整个系统崩溃。

通过限流器，给接口设置最高访问量，保证在系统的可承受范围内，提供服务。一旦超过该阈值，不允许用户访问，保证系统的其他服务能继续正常运行。

## 实现
```go
package limiter

import (
	"time"

	"golang.org/x/time/rate"
)

// Limiter 限流器
type Limiter struct {
	*rate.Limiter
}

// New 实例化
func New(maxRate int) *Limiter {
	instance := new(Limiter)
	instance.Limiter = rate.NewLimiter(rate.Every(time.Second), maxRate)

	return instance
}
```

## 示例
```go
package main
// LimitMiddleware 限流器中间件
func LimitMiddleware(maxRate int) gin.HandlerFunc {
	limiter := New(maxRate) // 实例化一个限流器
	return func(ctx *gin.Context) {
		if !limiter.Allow() { // 这里用于控制当限流器达到阈值的判断
			ctx.JSON(403, gin.H{"message": "Busy"})
			ctx.Abort()
		}
		ctx.Next()
	}
}

func main() {
	r := gin.Default()
	r.Use(LimitMiddleware(100)) // 每秒最高允许100个请求
	r.GET("/test", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "pong",
		})
	})
	r.Run()
}

```
当并发达到100以上时，后面的请求都会返回403。这样可以保证不会给系统更大的压力。

## 原理
请参考：[https://zhuanlan.zhihu.com/p/140440501](https://zhuanlan.zhihu.com/p/140440501)