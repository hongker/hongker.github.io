---
title: Golang代码优化02-全局变量
date: 2020-03-30 15:29:21
tags: golang
---
在项目中，全局变量使用的较为普遍，如全局DB连接池、Redis连接池等。

## 初始化
Go的全局变量，在main函数执行前初始化。
```go
var a = 1
func main()  {
	fmt.Println(a)
}
```

- 数据库连接
database初始化数据库连接
```go
package database
var Instance *gorm.DB
// init
func init() {
    db, err := gorm.Open("mysql", "user:password@/dbname?charset=utf8&parseTime=True&loc=Local")
    if err != nil {
        log.Fatalf("failed to connect database:%s", err.Error())
    }
    Instance = db
}
```

使用
```go
package main
import "database" // import,触发init函数
func main() {
    // ping mysql
    database.Instance.DB().Ping()
}
```
<!--more-->
## 使用uber的dig库管理全局变量
- 示例1
安装
```
go get go.uber.org/dig
```

```go
package app
import "go.uber.org/dig"
var container  = dig.New()
func init() {
    db, err := gorm.Open("mysql", "user:password@/dbname?charset=utf8&parseTime=True&loc=Local")
    if err != nil {
        log.Fatalf("failed to connect database:%s", err.Error())
    }
    // register into container
    container.Provide(func() (*gorm.DB) {
		return db
	})
}

// DBConnection return *gorm.DB
func DBConnection() (connection *gorm.DB) {
    _ = container.Invoke(func(db *gorm.DB) {
        connection = db
    })
    return
}
```

使用
```go
package main
import "app" // import,触发init函数
func main() {
    // ping mysql
    app.DBConnection.DB().Ping()
}
```

- 示例2
延迟加载
```go
package app
// container instance
var container  = dig.New()

// EventManager 
type EventManager struct {
    events []Event
}
// Event
type Event struct {
    Name string
}
// Register add events 
func (m *EventManager) Register(e Event) {
    m.events = append(m.events, e)
}
// GetEventManager
func GetEventManager() (manager *EventManager) {
    if err := container.Invoke(func(m *EventManager) {
		manager = m
	}); err != nil {
        manager = &EventManager{events: make([]Event)}
        // register into container
		_ = Container.Provide(func() *EventManager{
			return manager
		})
	}
	return
}
// usage
// app.GetEventManager().Register(ev)
```
