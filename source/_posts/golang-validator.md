---
title: Golang优化代码11-数据校验
date: 2020-04-01 21:50:43
tags: golang
---
本文介绍`github.com/go-playground/validator/v10`的用法。

## 验证器
>在项目中经常对客户端发起的接口请求参数进行校验，通常开发者必须自己编写一套验证的规则，但是使用统一的数据验证器能减轻开发者的负担，以及简化代码。
程序员并不是偷懒，而是提高自己的效率。

## 安装
```
go get github.com/go-playground/validator/v10
```

## 使用

- required 必填项
对于请求参数来说，使用的最多的就是必填项。数字0,空string、slices,map,pointer,interface,channel,function都会校验不通过。
```go
import "github.com/go-playground/validator/v10"

// 初始化验证器
var validate = validator.New()

func main() {
    var err error // 空interface
	var m map[string]int // 空map
	var ch chan int // 空channel
	var s []int // 空slice
	var p *log.Logger // 空pointer
	var f func() // 空func
	items := map[string]interface{}{
		"zero": 0, // 改成非0会输出nil，说明通过校验，没有错误
		"string":"",
		"slice": s,
		"map": m,
		"pointer" : p,
		"interface": err,
		"channel" : ch,
		"function": f,
	}
	for idx, val := range items {
		fmt.Printf("%s:%v\n", idx, validate.Var(val, "required"))
	}
}
```
输出错误信息，说明校验都未通过:
```
function:Key: '' Error:Field validation for '' failed on the 'required' tag
zero:Key: '' Error:Field validation for '' failed on the 'required' tag
string:Key: '' Error:Field validation for '' failed on the 'required' tag
slice:Key: '' Error:Field validation for '' failed on the 'required' tag
map:Key: '' Error:Field validation for '' failed on the 'required' tag
pointer:Key: '' Error:Field validation for '' failed on the 'required' tag
interface:Key: '' Error:Field validation for '' failed on the 'required' tag
channel:Key: '' Error:Field validation for '' failed on the 'required' tag
```

- 其他校验
```go
func main() {
    // 验证邮箱格式
	fmt.Println("ValidateEmail:",validate.Var("123456.qq.com", "required,email")) // failed
	fmt.Println("ValidateEmail:",validate.Var("123456@qq.com", "required,email")) // success

	// 校验数字范围,gt:大于,gte:大于等于,lt:小于,lte:小于等于
	fmt.Println("ValidateNumber:", validate.Var(100, "required,gt=100,lt=120")) // failed
	fmt.Println("ValidateNumber:", validate.Var(110, "required,gte=100,lte=120")) // success
}
```

<!--more-->

- 校验结构体
通过tag标签定义结构体属性的校验规则，一次性校验所有属性，提高效率。
```go
func main() {
    type User struct {
		Name string `json:"name" validate:"required"`
		Age uint8 `json:"age" validate:"gte=0,lte=130"`
	}

	user := &User{
		Name: "",
		Age:  0,
    }
    fmt.Println("ValidateUser:", validate.Struct(user)) // failed

    user1 := &User{
		Name: "test",
		Age:  10,
    }
    fmt.Println("ValidateUser:", validate.Struct(user1)) // success
}
```

- 配合gin校验请求参数
因为gin已对接了验证器，所以使用`Bind`,`ShouldBind`等函数即可完成参数校验
```go
func main() {
	router := gin.Default()

	router.POST("user/auth", func(ctx *gin.Context) {
		type AuthRequest struct {
			// 如果是表单提交，使用form,否则获取不到数据
			Email string `json:"email" binding:"required,email"` // 验证邮箱格式
			Pass string `json:"pass" binding:"required,min=6,max=10"` // 验证密码，长度为6~10
		}

		var request AuthRequest
		// 使用bind
		if err := ctx.ShouldBindJSON(&request); err != nil {
			ctx.JSON(200, gin.H{
				"message" : err.Error(),
			})
			return
		}

		// 模拟判断业务
		if request.Email != "123456@qq.com" {
			ctx.JSON(200, gin.H{
				"message" : fmt.Sprintf("邮箱不存在:%s", request.Email),
			})
			return
		}
		ctx.JSON(200, gin.H{
			"message" : "success",
		})
	})

	router.Run(":8080")
}
```



- 自定义验证器
支持错误信息汉化、支持自定义字段名称。一切都在注释中

```go
import (
	"errors"
	"github.com/go-playground/locales/zh"
	ut "github.com/go-playground/universal-translator"
	"github.com/go-playground/validator/v10"
	zh_translations "github.com/go-playground/validator/v10/translations/zh"
	"reflect"
	"sync"
	"github.com/gin-gonic/gin/binding"
)

// Validator
type Validator struct {
	once     sync.Once
	validate *validator.Validate
}

// 定义一个中文翻译器
var translation ut.Translator

func init() {
	// 初始化
	translation, _ = ut.New(zh.New()).GetTranslator("zh")
}

// ValidateStruct validate struct
func (v *Validator) ValidateStruct(obj interface{}) error {
	value := reflect.ValueOf(obj)
	valueType := value.Kind()

	if valueType == reflect.Ptr {
		valueType = value.Elem().Kind()
	}
	if valueType == reflect.Struct {
		v.lazyInit()

		if err := v.validate.Struct(obj); err != nil {
			//验证器
			for _, err := range err.(validator.ValidationErrors) {
				return errors.New(err.Translate(translation))
			}
		}
	}

	return nil
}

// Engine
func (v *Validator) Engine() interface{} {
	v.lazyInit()
	return v.validate
}

// lazyInit 该方法会依次验证 Struct 的属性，遇到第一个错误的时候就会停止验证，并将错误信息返回
func (v *Validator) lazyInit() {
	v.once.Do(func() {
		v.validate = validator.New()
		// 指定验证的tag名称,gin默认是binding,也可以自己改成"validate"或者其他的tag
		v.validate.SetTagName("binding")

		// 定义一个交叫comment的tag为字段的名称,如 Name `comment:"姓名"`
		v.validate.RegisterTagNameFunc(func(fld reflect.StructField) string {
			return fld.Tag.Get("comment")
		})

		// use zh-CN
		_ = zh_translations.RegisterDefaultTranslations(v.validate, translation)
	})
}

// 使用
func main() {
	// 设置自定义验证器
	binding.Validator = new(Validator)

	router := gin.Default()

	router.POST("user/auth", func(ctx *gin.Context) {
		type AuthRequest struct {
			// 如果是表单提交，使用form,否则获取不到数据
			Email string `json:"email" validate:"required,email" comment:"邮箱"` // 验证邮箱格式
			Pass string `json:"pass" binding:"required,min=6,max=10"` // 验证密码，长度为6~10
		}

		var request AuthRequest
		// 使用bind
		if err := ctx.ShouldBindJSON(&request); err != nil {
			ctx.JSON(200, gin.H{
				"message" : err.Error(),
			})
			return
		}

		// 判断业务
		if request.Email != "123456@qq.com" {
			ctx.JSON(200, gin.H{
				"message" : fmt.Sprintf("邮箱不存在:%s", request.Email),
			})
			return
		}
		ctx.JSON(200, gin.H{
			"message" : "success",
		})
	})

	router.Run(":8080")
}
```

可以自己通过如postman测试。。

