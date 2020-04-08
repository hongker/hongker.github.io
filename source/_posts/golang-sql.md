---
title: Golang代码优化12-gorm
date: 2020-04-07 13:00:00
tags: golang
---
本文介绍再golang中通过gorm对mysql的操作。

## 什么是Gorm
>Golang写的，开发人员友好的ORM库。官方地址：`http://gorm.book.jasperxu.com/`.

## 安装
```
go get -u github.com/jinzhu/gorm
```

## 准备工作
```sql
// 创建数据库
create database test charset=utf8;

use test;

// 创建Users表
drop table if exists users;
create table users (
    id int unsigned not null primary key auto_increment comment '主键',
    username varchar(50) not null default '' comment '姓名',
    password varchar(32) not null default '' comment '密码',
    is_deleted tinyint unsigned not null default  0 comment  '是否删除',
    created_at  timestamp    default CURRENT_TIMESTAMP not null comment '创建时间',
    updated_at  timestamp    default CURRENT_TIMESTAMP not null on update CURRENT_TIMESTAMP comment '更新时间',
    key `idx_del`(`is_deleted`)
)engine=Innodb charset=utf8mb4 comment='示例';
```

## 创建Connection
```go
// Connection 创建数据库连接
func Connection() *gorm.DB {
    // dsn的格式为 用户名:密码/tcp(主机地址)/数据库名称?charset=字符格式
	dsn := "root:123456@tcp(10.0.75.2)/test?charset=utf8&parseTime=True&loc=Local"
	db, err := gorm.Open("mysql", dsn)
	if err != nil {
		panic(err)
	}

	return db
}
```

<!--more-->

## 定义模型
- 针对gorm对`timestamp`类型的格式化
```go
// Timestamp 自定义gorm的时间戳格式
type Timestamp struct {
	time.Time
}

// MarshalJSON 解析
func (t Timestamp) MarshalJSON() ([]byte, error) {
	formatted := fmt.Sprintf("\"%s\"", t.Format(date.TimeFormat))
	return []byte(formatted), nil
}

// Value
func (t Timestamp) Value() (driver.Value, error) {
	var zeroTime time.Time
	if t.Time.UnixNano() == zeroTime.UnixNano() {
		return nil, nil
	}
	return t.Time, nil
}

// Scan 转换时间戳
func (t *Timestamp) Scan(v interface{}) error {
	value, ok := v.(time.Time)
	if ok {
		*t = Timestamp{Time: value}
		return nil
	}
	return fmt.Errorf("can not convert %v to timestamp", v)
}
```

- 定义模型
```go
const (
	TableUser = "users"
)
// User 用户模型
type User struct {
	ID       int    `gorm:"primary_key;AUTO_INCREMENT;column:id" json:"id"`
	Username string `json:"username" gorm:"column:username"`
	Password string `json:"password" gorm:"column:password"`
	IsDeleted int `json:"is_deleted" gorm:"column:is_deleted"`
	// 对time进行格式化
	CreatedAt Timestamp `gorm:"column:created_at" json:"created_at"`
	UpdatedAt Timestamp `gorm:"column:created_at" json:"updated_at"`
}

// TableName 指定模型的表名称
func (User) TableName() string {
	return TableUser
}
```

- 定义Dao
设计一个Dao来进行数据表的操作
```go

// UserDao
type UserDao struct {
	db *gorm.DB
}

// GetUserDao
func GetUserDao(db *gorm.DB) *UserDao {
	return &UserDao{db:db}
}

// Create 创建记录
func (dao *UserDao) Create(user *User) error  {
	return dao.db.Omit("created_at", "updated_at").
		Create(user).Error
}

// GetByUsername 根据用户名获取记录
func (dao *UserDao) GetByUsername(username string) (*User, error) {
	query := dao.db.Table(TableUser).Where("username = ?", username)

	user := new(User)
	if err := query.First(user).Error; err != nil {
		return nil, err
	}

	return user, nil
}

// Update 更新字段
func (dao *UserDao) Update(id int, columns map[string]interface{}) error {
	return dao.db.Where("id = ?", id).Updates(columns).Error
}
```

- 创建记录
```go
//Md5 return the encrypt string by md5 algorithm
func Md5(s string) string {
	h := md5.New()
	h.Write([]byte(s))
	return hex.EncodeToString(h.Sum(nil))
}


import (
	"crypto/md5"
	"encoding/hex"
	"fmt"
	_ "github.com/go-sql-driver/mysql"
	"github.com/jinzhu/gorm"
)

func main() {
	conn := Connection()
	defer conn.Close()

	// 创建
	user := &User{
		Username: "user1",
		Password: Md5("123456"), // 使用md5加密
	}

	dao := GetUserDao(conn)
	if err := dao.Create(user); err != nil {
		fmt.Println("failed to create user:", err.Error())
	}else {
		fmt.Println("created user success:", user.ID)
	}
}
```

创建成功输出:
```
created user success: 1
```

- 登录校验
```go
func main() {
	conn := Connection()
	defer conn.Close()

	router := gin.Default()
	router.POST("/user/auth", func(ctx *gin.Context) {
		type AuthRequest struct {
			Username string `json:"username"` // 这里要求用json请求
			Password string `json:"password"`
		}

		var request AuthRequest
		if err := ctx.ShouldBindJSON(&request); err != nil {
			ctx.JSON(200, gin.H{
				"message" : fmt.Sprintf("参数错误:%s", err.Error()),
			})
			return
		}

		dao := GetUserDao(conn)
		user, err := dao.GetByUsername(request.Username)
		if err != nil {
			ctx.JSON(200, gin.H{
				"message" : fmt.Sprintf("用户不存在:%s", request.Username),
			})
			return
		}

		if Md5(request.Password) != user.Password {
			ctx.JSON(200, gin.H{
				"message" : "密码错误",
			})
			return
		}

		ctx.JSON(200, gin.H{
			"message" : "success",
			"token": "token", // 返回用户登录令牌
		})
	})

	router.Run(":8080")
}
```
通过curl验证：
```
curl -X POST -H "Content-Type:application/json" -d '{"username":"user1",password:"123456"}' localhost:8080/user/auth

// 接口返回：
{
    "message": "success",
    "token": "token"
}
```

- 连接池
```go
// 设置mysql的空闲连接数
conn.DB().SetMaxIdleConns(options.MaxIdleConnections)

// 设置mysql的最大打开的连接数
conn.DB().SetMaxOpenConns(options.MaxOpenConnections)
```

- 设置连接过期时间
```go
conn.DB().SetConnMaxLifetime(time.Second * 10)
```

- 开启log
```
conn.LogMode(true)
```

以上介绍的是gorm的常用的操作，如有其他疑问请查看官方文档或者再下方提问，随缘回复。。