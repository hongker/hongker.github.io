---
title: Golang代码优化15-AOP(面向切面编程)
date: 2020-04-12 10:45:50
tags: golang
---
因为某些原因，触发了我要深入了解下AOP。

<!-- toc -->
## 什么是AOP
>AOP为Aspect Oriented Programming的缩写，意为：面向切面编程，通过预编译方式和运行期间动态代理实现程序功能的统一维护的一种技术。扩展功能不修改源代码实现。

主要应用场景：日志记录，性能统计，安全控制，事务处理，异常处理等等。

## 核心概念
- JoinPoint:连接点。是程序执行中的一个精确执行点，例如类中的一个方法。
- PointCut：切入点。指定哪些组件的哪些方法使用切面组件。
- Advice：通知，用于指定具体作用的位置，是方法之前或之后等等，分为前置通知，后置通知，异常通知，返回通知，环绕通知。
- Aspect: 切面。封装通用业务逻辑的组件，即我们想要插入的代码内容。

其内在设计模式为代理模式。

## Go实现AOP
<!--more-->
```go
// User 
type User struct {
	Name string
	Pass string
}

// Auth 验证
func (u *User) Auth() {
	// 实际业务逻辑
	fmt.Printf("register user:%s, use pass:%s\n", u.Name, u.Pass)
}


// UserAdvice 
type UserAdvice interface {
    // Before 前置通知
    Before(user *User) error
    
    // After 后置通知
	After(user *User)
}

// ValidatePasswordAdvice 用户名验证
type ValidateNameAdvice struct {
}

// ValidatePasswordAdvice 密码验证
type ValidatePasswordAdvice struct {
	MinLength int
	MaxLength int
}

func (ValidateNameAdvice) Before(user *User) error {
	fmt.Println("ValidateNameAdvice before")
	if user.Name == "admin" {
		return errors.New("admin can't be used")
	}

	return nil
}

func (ValidateNameAdvice) After(user *User) {
	fmt.Println("ValidateNameAdvice after")
	fmt.Printf("username:%s validate sucess\n", user.Name)
}

// Before 前置校验
func (advice ValidatePasswordAdvice) Before(user *User) error {
	fmt.Println("ValidatePasswordAdvice before")
	if user.Pass == "123456" {
		return errors.New("pass isn't strong")
	}

	if len(user.Pass) > advice.MaxLength {
		return fmt.Errorf("len of pass must less than:%d", advice.MaxLength)
	}

	if len(user.Pass) < advice.MinLength {
		return fmt.Errorf("len of pass must greater than:%d", advice.MinLength)
	}

	return nil
}

func (ValidatePasswordAdvice) After(user *User) {
	fmt.Println("ValidatePasswordAdvice after")
	fmt.Printf("password:%s validate sucess\n", user.Pass)
}

// UserAdviceGroup,通知管理组
type UserAdviceGroup struct {
	items []UserAdvice
}

// Add 注入可选通知
func (g *UserAdviceGroup) Add(advice UserAdvice) {
	g.items = append(g.items, advice)
}

func (g *UserAdviceGroup) Before(user *User) error {
	for _, item := range g.items {
		if err := item.Before(user); err != nil {
			return err
		}
	}

	return nil
}

// After
func (g *UserAdviceGroup) After(user *User) {
	for _, item := range g.items {
		item.After(user)
	}
}

// UserProxy 代理，也是切面
type UserProxy struct {
	user *User
}

// NewUser return UserProxy
func NewUser(name, pass string) UserProxy {
	return UserProxy{user:&User{Name:name, Pass:pass}}
}

// Auth 校验，切入点
func (p UserProxy) Auth() {
	group := UserAdviceGroup{}
	group.Add(&ValidatePasswordAdvice{MaxLength:10, MinLength:6})
    group.Add(&ValidateNameAdvice{})
    
    // 前置通知
	if err := group.Before(p.user); err != nil {
		panic(err)
	}

    // 实际逻辑
	p.user.Auth()

    // 后置通知
	group.After(p.user)

}
```

使用AOP模式进行解耦，分离主业务与副业务。其实也就那样。