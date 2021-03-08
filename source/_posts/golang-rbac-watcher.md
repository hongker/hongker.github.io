---
title: Golang代码优化22-RBAC分布式同步
date: 2021-02-19 21:42:50
tags: golang
---
前面介绍了rbac组件在单点服务上的基本应用场景,今天来介绍一下，在微服务架构下如何使用rbac来控制路由权限。

## 服务架构
![架构图](http://processon.com/chart_image/6045ee8d07912947636afd81.png?_=1615196757407)

这是一个很普通的微服务架构，用户通过请求服务器的路由资源，发起访问，各服务需要检查用户是否拥有权限。

RBAC组件是将权限规则存入内存里，当管理员通过权限管理后台更新的权限信息后，如何让各服务节点即时加载最新的权限规则呢？


我们发现，管理后台产生新的权限规则，多个服务节点读取最新的权限规则，这是一个典型的生产者/消费者模型。


[casbin](https://github.com/casbin/casbin)有提供一个`Watcher`接口。官网也提供了如`redis`、`etcd`、`kafka`等解决访问，详细文档请参考：[https://casbin.org/docs/en/watchers](https://casbin.org/docs/en/watchers)。下面就用redis做示例说明


### redis
基于redis的发布/订阅(Publish/Subscribe),管理员更新用户权限后，通过redis发布一条信息，其余服务节点，因为订阅了该信息，故每个客户端都会接受到一条通知，此刻更新内存的权限规则，即可实现分布式的权限更新。将此组件称为:`redis-watcher`

- Demo
```go
package main

import (
	"fmt"
	"github.com/casbin/casbin/v2/persist"
	uuid "github.com/satori/go.uuid"
	"reflect"
	"sync"

	"github.com/go-redis/redis"
)
// Watcher
type Watcher struct {
	options options // 选项
	pubConn  redis.UniversalClient //生产消息
	subConn  redis.UniversalClient //订阅消息
	callback func(string) // 回调方法
	closed   chan struct{}
	once     sync.Once
}
// 选项
type options struct {
	id string // 当前节点的id
	channel string // redis的管道名称
}
// Option 选项接口
type Option interface {
	Set(options *options)
}
// channel选项
type channelOption string
func (o channelOption) Set(opts *options)  {
	opts.channel = string(o)
}
func WithChannel(channel string) Option {
	return channelOption(channel)
}
// id选项
type idOption string
func (o idOption) Set(opts *options)  {
	opts.id = string(o)
}
func WithID(id string) Option {
	return idOption(id)
}
// NewWatcher 实例化
func NewWatcher(conn redis.UniversalClient, opts ...Option) (persist.Watcher, error) {
	w := &Watcher{
		closed: make(chan struct{}),
		pubConn: conn,
		subConn: conn,
	}

	w.options = options{
		channel:  "/casbin",
		id : uuid.NewV4().String(),
	}

	for _, opt := range opts {
		opt.Set(&w.options)
	}

	go func() {
		for {
			select {
			case <-w.closed:
				return
			default:
				err := w.subscribe()
				if err != nil {
					fmt.Printf("Failure from Redis subscription: %v", err)
				}
			}
		}
	}()

	return w, nil
}

// SetUpdateCallBack sets the update callback function invoked by the watcher
// when the policy is updated. Defaults to Enforcer.LoadPolicy()
func (w *Watcher) SetUpdateCallback(callback func(string)) error {
	w.callback = callback
	return nil
}

// Update publishes a message to all other casbin instances telling them to
// invoke their update callback
func (w *Watcher) Update() error {
	// 当有权限更新时，发送当前节点的ID,通知其他节点有更新
	if _, err := w.pubConn.Publish(w.options.channel, w.options.id).Result(); err != nil {
		return err
	}
	return nil
}

// Close disconnects the watcher from redis
func (w *Watcher) Close() {
	w.once.Do(func() {
		close(w.closed)
		_ = w.subConn.Close()
		_ = w.pubConn.Close()
	})
}
// subscribe 订阅redis
func (w *Watcher) subscribe() error {
	psc := w.subConn.Subscribe(w.options.channel)
	defer psc.Unsubscribe()
	for {
		receive, err := psc.Receive()
		fmt.Println(reflect.TypeOf(receive))
		if err != nil {
			fmt.Printf("try subscribe channel[%s] error[%s]\n", w.options.channel, err.Error())
			return nil
		}
		switch n := receive.(type) {
		case error:
			return n
		case *redis.Message:
			// 需要跳过当前生产者
			if n.Payload !=  w.options.id && w.callback != nil {
				w.callback("success") // 执行回调
			}

		case *redis.Subscription:
			if n.Count == 0 {
				return nil
			}
		}
	}
}
```

- 使用方式：
```go
func main() {
    authEnforcer, err := casbin.NewEnforcerSafe("rbac/conf/rbac_model.conf")
	if err != nil {
		log.Fatal(err)
	}

	redisConn := redis.NewClient(
		&redis.Options{
			Addr:     "127.0.0.1",
			Password: "",
			DB:       0,
		})

	if _, err := redisConn.Ping().Result(); err != nil {
		log.Fatal(err)
	}

	watcher, err := NewWatcher(redisConn)
	if err != nil {
		log.Fatal(err)
	}
	watcher.SetUpdateCallback(func(s string) {
		// 当有权限更新时，会执行这个回调，具体更新内存的权限逻辑可以自定义
		enforcer.LoadPolicy()
	})
	authEnforcer.SetWatcher(watcher)
}
```