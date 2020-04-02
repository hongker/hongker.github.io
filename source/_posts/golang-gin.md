---
title: Golang代码优化10-gin
date: 2020-04-01 21:50:13
tags: golang
---
本文介绍gin的一些知识点,如自定义Response,中间件等。

## gin
>Gin 是一个 go 写的 web 框架，具有高性能的优点。

初级的使用方式不介绍了，具体请查阅官方文档。官方地址：`https://github.com/gin-gonic/gin`

以下介绍基于gin开发项目的一些常用模块。

## 自定义Response
每个公司都会自定义接口的数据结构。故我们需要基于`Json()`自定义一个更方便好用的response
```go
// Response 数据结构体
type Response struct {
    // StatusCode 业务状态码
	StatusCode int `json:"status_code"`

    // Message 提示信息
	Message    string      `json:"message"`

    // Data 数据，用interface{}的目的是可以用任意数据
	Data       interface{} `json:"data"`

    // Meta 源数据,存储如请求ID,分页等信息
	Meta       Meta        `json:"meta"`

    // Errors 错误提示，如 xx字段不能为空等
	Errors     []ErrorItem `json:"errors"`
}

// Meta 元数据
type Meta struct {
	RequestId      string                 `json:"request_id"`
	// 还可以集成分页信息等
}


// ErrorItem 错误项
type ErrorItem struct {
	Key   string `json:"key"`
	Value string `json:"error"`
}

// New return response instance
func New() *Response {
	return &Response{
		StatusCode: 200,
		Message:    "",
		Data:       nil,
		Meta: Meta{
			RequestId: uuid.NewV4().String(),
		},
		Errors: []ErrorItem{},
	}
}

```

封装gin.Context以自定义一些方便的方法
```go
// Wrapper include context
type Wrapper struct {
	ctx *gin.Context
}

// WrapContext
func WrapContext(ctx *gin.Context) *Wrapper {
	return &Wrapper{ctx:ctx}
}

// Json 输出json,支持自定义response结构体
func (wrapper *Wrapper) Json(response *Response) {
	wrapper.ctx.JSON(200, response)
}

// Success 成功的输出
func (wrapper *Wrapper) Success( data interface{}) {
	response := New()
	response.Data = data
	wrapper.Json(response)
}

// Error 错误输出
func (wrapper *Wrapper) Error( statusCode int, message string) {
	response := New()
	response.StatusCode = statusCode
	response.Message = message
	wrapper.Json(response)
}
```

使用：
```go
package main

import (
	"github.com/gin-gonic/gin"
	uuid "github.com/satori/go.uuid"
)

func main()  {
	router := gin.Default()
	router.GET("/", func(ctx *gin.Context) {
		WrapContext(ctx).Success("hello,world")
	})

	router.Run(":8088")
}
```
通过`go run main.go`运行后，浏览器访问`localhost:8088`

## 中间件
介绍一些常用的中间件，如跨域、Jwt校验、请求日志等。
<!--more-->
- 备注
引入中间件比如在注册路由之前,谨记!

- 跨域中间件
```go
package middleware
import (
	"github.com/gin-gonic/gin"
)
// CORS 跨域中间件
func CORS(ctx *gin.Context) {
	method := ctx.Request.Method

	// set response header
	ctx.Header("Access-Control-Allow-Origin", ctx.Request.Header.Get("Origin"))
	ctx.Header("Access-Control-Allow-Credentials", "true")
	ctx.Header("Access-Control-Allow-Headers", "Content-Type, Access-Control-Allow-Headers, Authorization, X-Requested-With")
	ctx.Header("Access-Control-Allow-Methods", "GET,POST,PUT,PATCH,DELETE,OPTIONS")

    // 默认过滤这两个请求,使用204(No Content)这个特殊的http status code
	if method == "OPTIONS" || method == "HEAD" { 
		ctx.AbortWithStatus(204)
		return
	}

	ctx.Next()
}
```
使用如下：
```go
func main() {
    router := gin.Default()
    router.Use(CORS)
	router.GET("/", func(ctx *gin.Context) {
		WrapContext(ctx).Success("hello,world")
	})

	router.Run(":8088")
}
```

- Jwt校验
```go
package main

import (
	"errors"
	"github.com/dgrijalva/jwt-go"
	"github.com/gin-gonic/gin"
	"strings"
	"time"
)

var (
	TokenNotExist       = errors.New("token not exist")
	TokenValidateFailed = errors.New("token validate failed")
	ClaimsKey = "uniqueClaimsKey"
	SignKey = "test"
)

// JwtAuth jwt
type JwtAuth struct {
	SignKey []byte
}

// ParseToken parse token
func (jwtAuth JwtAuth) ParseToken(token string) (jwt.Claims, error) {
	tokenClaims, err := jwt.Parse(token, func(token *jwt.Token) (interface{}, error) {
		return jwtAuth.SignKey, nil
	})

	if err != nil {
		return nil, err
	}

	if tokenClaims.Claims == nil || !tokenClaims.Valid {
		return nil, TokenValidateFailed
	}

	return tokenClaims.Claims, nil
}

// GenerateToken
func (jwtAuth JwtAuth) GenerateToken(tokenExpireTime int64 /* 过期时间 */, iss string /* key*/) (string, error) {
	now := time.Now().Unix()
	exp := now + tokenExpireTime
	claim := jwt.MapClaims{
		"iss": iss,
		"iat": now,
		"exp": exp,
	}
	token := jwt.NewWithClaims(jwt.SigningMethodHS256, claim)
	tokenStr, err := token.SignedString(jwtAuth.SignKey)
	return tokenStr, err
}

// JWT gin的jwt中间件
func JWT(ctx *gin.Context) {
	// 解析token
	if err := validateToken(ctx); err != nil {
		WrapContext(ctx).Error(401, err.Error())
		ctx.Abort()
		return
	}

	ctx.Next()
}

// validateToken 验证token
func validateToken(ctx *gin.Context) error {
	// 获取token
	tokenStr := ctx.GetHeader("Authorization")
	kv := strings.Split(tokenStr, " ")
	if len(kv) != 2 || kv[0] != "Bearer" {
		return TokenNotExist
	}

	jwtAuth := &JwtAuth{SignKey: []byte(SignKey)}
	claims, err := jwtAuth.ParseToken(kv[1])
	if err != nil {
		return err
	}

	// token存入context
	ctx.Set(ClaimsKey, claims)
	return nil
}
```

使用如下：
```go
func main()  {
	router := gin.Default()
	router.GET("/", func(ctx *gin.Context) {
		WrapContext(ctx).Success("hello,world")
	})

    // 指定user这组路由都需要校验jwt
	user := router.Group("/user").Use(JWT)
	{
		user.GET("/info", func(ctx *gin.Context) {
			claims, exist := ctx.Get(ClaimsKey)
			if !exist {
				WrapContext(ctx).Error(1001, "获取用户信息失败")
			}
			WrapContext(ctx).Success(claims)
		})
	}


	router.Run(":8088")
}
```

请求测试：
```
curl  "localhost:8088/user/info"
// 输出:
// {"status_code":401,"message":"token not exist","data":null,"meta":{"request_id":"e69361cf-1fd4-42e4-8af8-d18fac1e70fb"},"errors":[]}

// 通过GenerateToken()生成一个token
curl -H "Authorization:Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE1ODU4MjQ2NzgsImlhdCI6MTU4NTgyMTA3OCwiaXNzIjoiYWEifQ.Eyo8KptVUgGfnRG8zsjDilAJOBmaXMtjqJxw__a32HY"  localhost:8088/user/info
// 输出：
{"status_code":200,"message":"","data":{"exp":1585824678,"iat":1585821078,"iss":"aa"},"meta":{"request_id":"464743de-1033-4656-96f8-36c1529f13e0"},"errors":[]}
```

- 请求日志
记录每个请求的重要信息
```go
import (
	"bytes"
	"fmt"
	"github.com/gin-gonic/gin"
	"io/ioutil"
	"log"
	"net/http"
	"time"
)

// bodyLogWriter 定义一个存储响应内容的结构体
type bodyLogWriter struct {
	gin.ResponseWriter
	body *bytes.Buffer
}

// Write 读取响应数据
func (w bodyLogWriter) Write(b []byte) (int, error) {
	w.body.Write(b)
	return w.ResponseWriter.Write(b)
}

// RequestLog gin的请求日志中间件
func RequestLog(c *gin.Context) {
	// 记录请求开始时间
	t := time.Now()
	blw := &bodyLogWriter{body: bytes.NewBufferString(""), ResponseWriter: c.Writer}
	// 必须!
	c.Writer = blw

	// 获取请求信息
	requestBody := getRequestBody(c)

	c.Next()

	// 记录请求所用时间
	latency := time.Since(t)

	// 获取响应内容
	responseBody := blw.body.String()

	logContext := make(map[string]interface{})
	// 日志格式
	logContext["request_uri"] = c.Request.RequestURI
	logContext["request_method"] = c.Request.Method
	logContext["refer_service_name"] = c.Request.Referer()
	logContext["refer_request_host"] = c.ClientIP()
	logContext["request_body"] = requestBody
	logContext["request_time"] = t.String()
	logContext["response_body"] = responseBody
	logContext["time_used"] = fmt.Sprintf("%v", latency)
	logContext["header"] = c.Request.Header

	log.Println(logContext)
}

// getRequestBody 获取请求参数
func getRequestBody(c *gin.Context) interface{} {
	switch c.Request.Method {
	case http.MethodGet:
		return c.Request.URL.Query()

	case http.MethodPost:
		fallthrough
	case http.MethodPut:
		fallthrough
	case http.MethodPatch:
		var bodyBytes []byte // 我们需要的body内容
        // 可以用buffer代替ioutil.ReadAll提高性能
		bodyBytes, err := ioutil.ReadAll(c.Request.Body)
		if err != nil {
			return nil
        }
        // 将数据还回去
		c.Request.Body = ioutil.NopCloser(bytes.NewBuffer(bodyBytes))

		return string(bodyBytes)

	}

	return nil
}
```

使用
```
router.Use(ReqeustLog)
```

今天就到这儿吧，还有一些比如全局ID中间件，后面来写。