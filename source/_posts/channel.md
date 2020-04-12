---
title: Golang代码优化03-协程与管道
date: 2020-03-30 16:38:50
tags: golang
---
介绍一些常规的用法
## channel的状态
- channel:未初始化时为nil
- active:正常的channel,可读可写
- closed:调用了close()后的状态

## channel的操作
- 读 <-channel
- 写 ->channel
- 关闭 close

## 常用的channel使用方法
- 使用for..range 读取channel
当需要不断从channel中读取数据时，使用for..range,当channel关闭时，for..range会自动退出
```go
func main() {
    ch := make(chan string)
	go sendData(ch)
	for out := range ch {
		fmt.Println(out)
	}
}

func sendData(ch chan string) {
    ch <- "Washington"
	ch <- "Tripoli"
	ch <- "London"
	ch <- "Beijing"
	ch <- "Tokyo"
	defer close(ch)
}
```

- 使用value,ok判断channel是否关闭
当直接读取channel时，如果channel已关闭，会触发panic,故需要判断channel是否关闭
```go
func main() {
    ch := make(chan string)
	go sendData(ch)
	for {
		if out, open := <- ch; open {
			fmt.Println(out)
		}else {
			break
		}
	}
}
```

- 使用select读取多个channel
select可以同时监控多个通道的情况，只处理未阻塞的case。当通道为nil时，对应的case永远为阻塞，无论读写。特殊关注：普通情况下，对nil的通道写操作是要panic的。

- 使用channel的声明控制读写权限
协程对某个channel只读或只写

- 使用缓冲channel增强并发和异步
有缓冲通道是异步的，无缓冲通道是同步的。有缓冲通道可供多个协程同时处理，在一定程度可提高并发性。

<!--more-->

- 为操作加上超时
在实际开发中，无法预估时间的程序，为了保证程序安全，我们需要超时控制的操作。
```go
func main() {
	fmt.Println(doWithTimeout(1e9 * 2))
}

func doWithTimeout(timeout time.Duration) (int, error) {
	select {
	case res := <- do():
		return res, nil
	case <-time.After(timeout):
		return 0, errors.New("timeout")
	}
}

func do() <- chan int {
	outCh := make(chan int)
	go func() {
		time.Sleep(1e9 * 3) // 模拟如复杂的sql查询
		outCh <- 1
	}()
	return outCh
}
```

- 使用chan struct{}作为信号channel
使用channel传递信号，而不是传递数据时
```go
func main() {
	done := make(chan struct{})
	go func() {
		time.Sleep(1e9 * 2)
		close(done) // 发送关闭信息
	}()
	for {
		select {
		case <-done:
			fmt.Println("done")
			return
		}
	}

}
```

## 读取Excel
通过channel实现异步读取excel的demo
```go
package main

import (
	"fmt"
	"github.com/tealeg/xlsx"
)

func main()  {
	// data.xls:
	// title
	// 1
	// 2
	// 3
	reader := NewExcelReader("static/data.xlsx", 10)
	// read
	go func() {
		fmt.Println(reader.Read(0))
	}()

	// output
	for out := range reader.OutPut() {
		// other process
		fmt.Println(out)
	}

}

// ExcelReader reader of excel
type ExcelReader struct {
	// excel file name
	FileName string

	// data
	items chan map[string]string
}

// NewExcelReader return ExcelReader with filename and chan length
func NewExcelReader(filename string, chanLen int) *ExcelReader {
	return &ExcelReader{
		FileName: filename,
		items:    make(chan map[string]string, chanLen),
	}
}

// Read read excel sheet
func (r *ExcelReader) Read(sheetNo int /* sheet number*/) error {
	// open file
	xlFile, err := xlsx.OpenFile(r.FileName)
	if err != nil {
		return err
	}

	sheets := xlFile.Sheets[sheetNo]

	// set first row as map filed
	var fieldArr []string

	for idx, row := range sheets.Rows {
		var arr []string
		// read data
		for _, cell := range row.Cells {
			arr = append(arr, cell.String())
		}

		if arr == nil { // filter empty row
			continue
		}

		// read filed
		if idx == 0 {
			fieldArr = arr
			continue
		}

		item := make(map[string]string)
		for key, field := range fieldArr {
			item[field] = arr[key]
		}

		r.items <- item
	}

	// close channel when read finished
	close(r.items)
	return nil
}

// OutPut return read data
func (r *ExcelReader) OutPut() <- chan map[string]string {
	return r.items
}
```

## Goroutine
Goroutines 是在 Golang 中执行并发任务的方式。Go线程实现模型MPG
- Machine,表示内核线程。
- Processor,处理器，执行G的上下文环境，每个P会维护一个本地的go routine队列。
- Goroutine,并发的最小逻辑单元，轻量级的线程。

一个线程使用一个处理器，一个处理器运行多个协程。

- Runtime里与Goroutine相关函数
```go
Goexit:退出当前执行的goroutine
Gosched:让出当前goroutine的执行权限，调度器安排其他等待的任务运行，并在下次某个时候从该位置回复执行。
NumCPU:返回CPU核数量。
NumGoroutine：返回正在执行和排队的任务总数。
GOMAXPROCS：用来设置可以并行计算的CPU核数的最大值，并返回之前的值。
```

- 调度原理
1.Processor的数量由GOMAXPROCS设置
2.用户需要做的就是添加goroutine,即通过`go`开启协程。
3.Goroutine的数量超过了Machine的处理能力，且有空余的Processor的话，runtime会自动创建Machine
4.Machine拿到Processor后开始工作，取Goroutine的顺序：本地队列>全局队列>其他P的队列。如果没有Goroutine，Machine归还Processor后休眠。