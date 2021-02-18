---
title: Golang代码优化20-通过SingleFlight防止缓存击穿
date: 2021-02-18 10:57:09
tags: golang
---
在高并发场景中，我们最普遍的方案是采用高性能的缓存。在缓存使用过程中，需要注意`缓存穿透`、`缓存雪崩`、`缓存击穿`。以下介绍如何通过`singleflight`防止`缓存击穿`。

## 什么是缓存击穿
>缓存击穿是指缓存中没有但数据库中有的数据（一般是缓存时间到期），这时由于并发用户特别多，同时读缓存没读到数据，又同时去数据库去取数据，引起数据库压力瞬间增大，造成过大压力。

## SingleFlight
在多个并发请求对一个失效的key进行源数据获取时，只让其中一个得到执行，其余阻塞等待到执行的那个请求完成后，将结果传递给阻塞的其他请求达到防止击穿的效果。

## 安装
```
go get golang.org/x/sync/singleflight
```

## 示例
```go
package main

import (
	"fmt"
	"log"
	"time"

	"golang.org/x/sync/singleflight"
)

var singleHandle singleflight.Group

func main() {
	for i := 0; i < 10; i++ { // 模拟10个并发
		go func() {
			fmt.Println(GetName("username"))
		}()
	}
	// 等待协程执行结束
	time.Sleep(3 * time.Second)
}

// GetName 获取名称
func GetName(cacheKey string) string {
	result, _, _ := singleHandle.Do(cacheKey, func() (ret interface{}, err error) {
		log.Printf("getting %s from database\n", cacheKey)
		log.Printf("setting %s in cache\n", cacheKey)
		return "hongker", nil
	})
	return result.(string)
}
```

- 结果
```
2021/02/18 14:26:12 getting username from database
2021/02/18 14:26:12 setting username in cache
hongker
hongker
hongker
hongker
hongker
hongker
hongker
hongker
hongker
hongker
```
只有一个协程执行从数据库中获取并设置缓存的操作，其他协程则是在等待获取缓存。避免缓存击穿。