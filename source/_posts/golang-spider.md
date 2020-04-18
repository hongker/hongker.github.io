---
title: Golang代码优化17-爬虫
date: 2020-04-17 16:53:29
tags: golang
---
本文介绍golang里也可以像python那样强大的使用爬虫。

## 什么是爬虫
> 网络爬虫（又称为网页蜘蛛，网络机器人，在FOAF社区中间，更经常的称为网页追逐者），是一种按照一定的规则，自动地抓取万维网信息的程序或者脚本。另外一些不常使用的名字还有蚂蚁、自动索引、模拟程序或者蠕虫。

## 说明
爬虫在19年国家颁布《中华人民共和国网络安全法》里，明确指出了非法爬取数据的规定，此文章仅为学习讨论，合法学习并使用爬虫。

## 爬虫框架
- chromedb
无外部依赖，直接驱动支持chrome的DevTool协议的浏览器发起http请求

安装：
```
go get -u github.com/chromedb/chromedb
```
当然你的电脑上还需要安装chrome

## 示例Demo
```go
package main

import (
	"context"
	"github.com/chromedp/chromedp"
	"log"
	"strings"
)

func main() {
	// 如果是windows,出现以下错误时：
	// exec: "google-chrome": executable file not found in %PATH%
	// 需要通过下面方式指定运行环境
	// linux只要安装了chrome,一般不会出现这些问题
	allocCtx, allocCancel := chromedp.NewExecAllocator(context.Background(), chromedp.ExecPath("C:\\Users\\cx\\AppData\\Local\\Google\\Chrome\\Application\\chrome.exe"))
	defer allocCancel()

	// 如果是linux,可以直接使用chrome headless模式:
	// chromedp.NewContext(context.Background())
	ctx, cancel := chromedp.NewContext(allocCtx)
	defer cancel()


	var res string

	// 运行
	err := chromedp.Run(ctx,
		// 指定url
		chromedp.Navigate("https://segmentfault.com/questions"),

		// 通过指定标签页，获取html内容，并将内容写入res变量里
		chromedp.Text(`div[class="copyright"]`, &res, chromedp.NodeVisible, chromedp.BySearch),
	)

	if err != nil {
		log.Fatal(err)
	}

	log.Println(strings.TrimSpace(res))

}
```

