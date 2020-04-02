---
title: Golang代码优化01-错误信息
date: 2020-03-30 15:04:12
tags: golang
---
记录项目开发中Go写法的代码优化。

## 定制化error
在项目开发中，我们不想直接暴露系统底层的错误，同时常规的error无法满足项目精细化的处理，我们经常会对error通过code进行分类。故定义如下:
```go
// Error 
type Error struct {
	Code    int    `json:"code"`
	Message string `json:"message"`
}

// Error string
func (e *Error) Error() string {
	b, _ := json.Marshal(e)
	return string(b)
}

// New return error with code
func New( code int, message string) *Error {
	return &Error{
		Code:    code,
		Message: message,
	}
}

// Parse tries to parse a JSON string into an error. If that
// fails, it will set the given string as the error detail.
func Parse(errStr string) *Error {
	e := new(Error)

	if err := json.Unmarshal([]byte(errStr), e); err != nil {
		e.Code = http.StatusInternalServerError
		e.Message = err.Error()
	}
	return e
}
```

## 使用
```go
// new
err := New(200105001, "something wrong" )
fmt.Println("err:",err.Error())

// parse
errParse := Parse(err.Error())
fmt.Println("err code": errParse.Code)
```

<!--more-->

基于Error扩展出部分常用的方法：
- 401 Unauthorized
```go
// Unauthorized generates a 401 error.
func Unauthorized(format string, v ...interface{}) *Error {
	return New(http.StatusUnauthorized, fmt.Sprintf(format, v...))
}
```

- 403 Forbidden
```go
// Forbidden generates a 403 error.
func Forbidden(format string, v ...interface{}) *Error {
	return New(http.StatusForbidden, fmt.Sprintf(format, v...))
}
```

- 404 NotFound
```go
// NotFound generates a 404 error.
func NotFound(format string, v ...interface{}) *Error {
	return New(http.StatusNotFound, fmt.Sprintf(format, v...))
}
```

- 405 MethodNotAllowed
```go
// MethodNotAllowed generates a 405 error.
func MethodNotAllowed(format string, v ...interface{}) *Error {
	return New(http.StatusMethodNotAllowed, fmt.Sprintf(format, v...))
}
```

- 408 Timeout

```go
// Timeout generates a 408 error.
func Timeout(format string, v ...interface{}) *Error {
	return New(http.StatusRequestTimeout, fmt.Sprintf(format, v...))
}
```

- 500 InternalServer
```go
// InternalServerError generates a 500 error.
func InternalServer(format string, v ...interface{}) *Error {
	return New(http.StatusInternalServerError, fmt.Sprintf(format, v...))
}
```