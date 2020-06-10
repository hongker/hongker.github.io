---
title: Golang代码优化19-自动生成API文档之Swagger
date: 2020-06-10 22:17:19
tags: golang
---
本文介绍如何巧妙的使用go-swagger自动生成API接口文档。

<!-- toc -->
## 什么是swagger
>Swagger 是一个规范和完整的框架，用于生成、描述、调用和可视化 RESTful 风格的 Web 服务。

## 准备工作
- 下载swag工具
```
// 通过go get下载并编译，在$GOPATH/bin目录下会生成swag命令，用于扫描注解。
go get -u github.com/swaggo/swag/cmd/swag
```
- 引入ego框架,已集成好的gin-swagger组件
```
go get github.com/ebar-go/ego
```

## API文档初始化
- 先写项目说明
```go

// @title Swagger Example API
// @version 1.0
// @description This is a sample server Petstore server.
// @termsOfService http://swagger.io/terms/

// @contact.name hongker
// @contact.url http://hongker.github.io
// @contact.email xiaok2013@live.com

// @license.name Apache 2.0
// @license.url http://www.apache.org/licenses/LICENSE-2.0.html

// @host localhost:8080
// @BasePath /v1
func main()  {
	s := http.NewServer()

	// 加载路由
	route.Load(s.Router)

	// 启动
	secure.Panic(s.Start())
}
```

<!--more-->
- 通过命令自动生成swagger.json
```
// 生成的文件默认放在当前目录的docs下
swag init
```

- 引入swagger的web
```go
// 我一般都是在router模块下加载
import(
    _ "ego-demo/docs" // 这一行必须
    "github.com/ebar-go/ego/http/middleware"
)
// 通过 {host}/swagger/index.html访问swagger web
router.GET("/swagger/*any", middleware.Swagger())
```
启动web服务，访问 `/swagger/index.html`就能看到效果了

### 生成API
- demo
```go
// UserAuthHandler 用户登录
// @Summary 用户登录
// @Description 通过邮箱和密码登录，换取token
// @Accept  json
// @Produce json
// @Param email body string true "邮箱"
// @Param pass body string true "密码"
// @Success 0 "success"
// @Failure 500 "error"
// @Router /user/auth [post]
func UserAuthHandler(ctx *gin.Context) {
    // demo
}
```

- 说明
```
Summary : 副标题
Description : 接口描述
Accept : 接收类型
Produce: 响应类型
Param: 参数
Success: 成功的响应
Failure: 失败的响应
Router : 路由(包括uri, method)
```

- 使用结构体
使用定义好的结构体，避免每个参数都得写一行注解。非常喜欢这种方式。
```go
// UserRegisterHandler 用户注册
// @Summary 用户注册
// @Description 通过邮箱和密码注册账户
// @Accept  json
// @Produce json
// @Param req body request.UserRegisterRequest true "请求参数"
// @Success 0 "success"
// @Failure 500 "error"
// @Router /user/register [post]
func UserRegisterHandler(ctx *gin.Context) {
	var req request.UserRegisterRequest

	// 校验参数
	if err := ctx.ShouldBindJSON(&req); err != nil {
		// 使用抛出异常的方式，截断代码逻辑，让recover输出响应内容，减少return
		secure.Panic(errors.New(statusCode.InvalidParam, err.Error()))
    }
}

// UserRegisterRequest 注册请求
type UserRegisterRequest struct {
	// 邮箱
	Email string `json:"email" binding:"required,email" comment:"邮箱"`       // 验证邮箱格式
	// 密码
	Pass  string `json:"pass" binding:"required,min=6,max=10" comment:"密码"` // 验证密码，长度为6~10
}
```
注：   
1.参数名称读的是json的tag，如Email的"email".       
2.参数说明读的是属性上一行的注释，如Email上面的"邮箱"。   