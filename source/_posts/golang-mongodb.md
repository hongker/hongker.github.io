---
title: Golang24-mongodb客户端
date: 2021-03-24 22:04:47
tags: golang
---
上一篇文章介绍了如何部署mongodb集群，本文介绍如何用mgo客户端操作mongodb。

## 安装
```
go get github.com/globalsign/mgo
```

## 连接mongo
```go
import (
	"fmt"
	"github.com/globalsign/mgo"
	"log"
)

func main() {
	dialInfo := &mgo.DialInfo{
		Addrs:    []string{"172.17.0.1:27017","172.17.0.1:27018"},
	}

	session ,err := mgo.DialWithInfo(dialInfo)
	if err != nil {
		log.Fatalf("connect: %v\n", err)
	}
	fmt.Println(session.DatabaseNames())
	log.Println("connect success")
}
```

## 查询
```go
type User struct {
	Name string `json:"name"`
}

func Run(session *mgo.Session)  {
	collection := session.Copy().DB("demo").C("user")
	// 查看总数
	count, err := collection.Count()
	if err != nil {
		log.Fatalf("count: %v\n", err)
	}
	log.Printf("collection count:%d\n", count)

	// 插入数据
	if err := collection.Insert(&User{Name: "bob"}); err != nil {
		log.Fatalf("insert: %v\n", err)
	}
	log.Println("insert success")

	// 查询数据
	entity := new(User)
	if err := collection.Find(bson.M{"name":"bob"}).One(entity); err != nil {
		log.Fatalf("Find: %v\n", err)
	}
	log.Printf("find user: %s\n", entity.Name)
}
```