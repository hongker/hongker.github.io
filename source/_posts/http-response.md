---
title: Golang代码优化04-Http响应处理
date: 2020-03-31 12:28:14
tags: golang
---
介绍读取http的response内容。

## 读取方式
- 1.使用`ioutil.ReadAll()`读取response
```go
func main () {
    // send get request
    resp, err := http.Get("https://api.ipify.org/?format=json")
	if err != nil {
		fmt.Println(err)
		return
    }
    
    // defer must execute after check error,otherwise will panic when resp is nil
    defer resp.Body.Close()
    
    // read body
	body, err := ioutil.ReadAll(resp.Body)
	if err != nil {
		fmt.Println(err)
		return
	}
	fmt.Println(string(body))
}
```
但是`ioutil.ReadAll()`方法有弊端,有时候却会导致一些性能问题。比如http大量请求时轻则导致内存浪费严重，重则导致内存泄漏影响业务。

- 2.使用buffer读取
```go
func main () {
	resp, err := http.Get("https://api.ipify.org/?format=json")
	if err != nil {
		fmt.Println(err)
		return
	}
    defer resp.Body.Close()
    
    // new buffer
	buffer := bytes.NewBuffer(make([]byte, 4056))

	if _, err := io.Copy(buffer, resp.Body);err != nil {
		fmt.Println(err)
		return
	}
	fmt.Println(buffer.String())
}
```

<!--more-->

- 3.BufferPool
当需要处理较多的httpResponse时，可以选择先初始化一个Buffer池，提升效率。
```go
// BufferPool use sync.pool to build a buffer pool
type Adapter struct {
	pool sync.Pool
}

// NewBufferPool return BufferPool instance by buffer length
func NewBufferPool(bufLen int) *BufferPool {
	return &BufferPool{pool:sync.Pool{New: func() interface{} {
		return bytes.NewBuffer(make([]byte, bufLen))
	}}}
}

// StringifyResponse return response body as string
func (bp *BufferPool) StringifyResponse(response *http.Response) (string, error) {
	if response == nil {
		return "", fmt.Errorf("response is empty")
	}

	// close response
	defer func() {
		_ = response.Body.Close()
	}()

	if response.StatusCode != http.StatusOK {
		return "", fmt.Errorf("response status code is:%d", response.StatusCode)
	}
    
	buffer := bp.pool.Get().(*bytes.Buffer) // get buffer from pool
	buffer.Reset() // reset buffer
	defer func() {
		if buffer != nil {
			bp.pool.Put(buffer) // return back
			buffer = nil // close 
		}
	}()
	_, err := io.Copy(buffer, response.Body)

	if err != nil {
		return "", fmt.Errorf("failed to read respone:%s", err.Error())
	}

	return buffer.String(), nil
}
```
