---
title: Golang代码优化18-权限RBAC
date: 2020-04-25 18:08:07
tags: golang
---
本文介绍golang中RBAC(基于角色的权限控制)的使用方式

## 什么是RBAC
>RBAC  是基于角色的访问控制（Role-Based Access Control ）在 RBAC  中，权限与角色相关联，用户通过成为适当角色的成员而得到这些角色的权限。这就极大地简化了权限的管理。这样管理都是层级相互依赖的，权限赋予给角色，而把角色又赋予用户，这样的权限设计很清楚，管理起来很方便。

## 实现
我选用的是`github.com/casbin/casbin`,选择原因:社区的认同(Star很高)。

### 安装
```
go get github.com/casbin/casbin
```

### 定义模型
- conf/rbac_model.conf
```
[request_definition]
r = sub, obj, act

[policy_definition]
p = sub, obj, act

[role_definition]
g = _, _

[policy_effect]
e = some(where (p.eft == allow))

[matchers]
m = r.sub == p.sub && keyMatch(r.obj, p.obj) && (r.act == p.act || p.act == "*")
```

- conf/policy.csv
权限细则，可以读csv文件，可以读数据库。。
```
p, admin, /*, *
p, anonymous, /article, GET
p, anonymous, /article/*, GET
p, member, /permission, GET
p, member, /article, GET
p, member, /article/*, *
g, test01, anonymous
g, user01, member
```
说明：
- line 1: 定义admin角色能访问所有资源
- line 2,3: 定义anonymous角色能查看文章列表、文章详情
- line 4: 定义member角色能获取权限
- line 5,6: 定义member角色能访问文章所有权限
- line 7: 定义test01的角色为anonymous
- line 8: 定义user01的角色为member

### 定义中间件
```go
// enforcer.go

package main

import (
	"github.com/casbin/casbin"
	"github.com/gin-gonic/gin"
	"log"
	"net/http"
)

var enforcer *casbin.Enforcer

// init 初始化
func init()  {
	authEnforcer, err := casbin.NewEnforcerSafe("conf/rbac_model.conf", "conf/policy.csv")
	if err != nil {
		log.Fatal(err)
	}
	enforcer = authEnforcer
}

// GetPermissionsByUser 获取用户权限，用于前端展示
func GetPermissionsByUser(username string) [][]string  {
	return enforcer.GetImplicitPermissionsForUser(username)
}

// CheckPermission 权限校验中间件
func CheckPermission(ctx *gin.Context)  {
    // 获取角色，一般不会通过参数传递角色，而是根据用户当前的token解析得到用户的角色
    // 这里为了演示而简单处理
	role := ctx.Query("role")
	if role == "" {
		role = "anonymous"
	}

    // 根据角色,路由，请求方法校验权限
	hasPermission, err := enforcer.EnforceSafe(role, ctx.Request.URL.Path, ctx.Request.Method)

    // 程序异常
	if err != nil {
		ctx.String(http.StatusInternalServerError, err.Error())
		ctx.Abort()
	}

    // 没有权限
	if !hasPermission {
		ctx.String(http.StatusUnauthorized, "StatusUnauthorized")
		ctx.Abort()
	}

	ctx.Next()
}
```

### web服务

```go
package main

import (
	"github.com/gin-gonic/gin"
	"net/http"
)

func main() {
	router := gin.Default()
    // 使用中间件
    router.Use(CheckPermission)
    
    // 根据用户
	router.GET("permission", func(context *gin.Context) {
		context.JSON(http.StatusOK, gin.H{
			"data": GetPermissionsByUser(context.Query("user")),
		})
	})

    // 定义一组文章的接口
	article := router.Group("article")
	{
        // 列表查询
		article.GET("", func(context *gin.Context) {
			context.JSON(http.StatusOK, gin.H{
				"data": "paginate",
			})
		})

        // 查看详情
		article.GET("/:id", func(context *gin.Context) {
			context.JSON(http.StatusOK, gin.H{
				"data": "info:" + context.Param("id"),
			})
		})

        // 更新
		article.PUT("/:id", func(context *gin.Context) {
			context.JSON(http.StatusOK, gin.H{
				"data": "update"+ context.Param("id"),
			})
		})

        // 创建
		article.POST("", func(context *gin.Context) {
			context.JSON(http.StatusOK, gin.H{
				"data": "create",
			})
		})

        // 删除
		article.DELETE("/:id", func(context *gin.Context) {
			context.JSON(http.StatusOK, gin.H{
				"data": "delete"+ context.Param("id"),
			})
		})
	}


	router.Run(":8888")

}

```

## 测试
- 启动
```
go run mian.go
```

- 测试
```
curl localhost:8888/article
// output: {"data":"paginate"}
curl localhost:8888/article/1
// output: {"data":"info:1"}
curl localhost:8888/permission
// output: StatusUnauthorized
curl localhost:8888/permission?role=member&user=user01
// output: {"data":[["member","/permission","GET"],["member","/article","GET"],["member","/article/*","*"]]}
```