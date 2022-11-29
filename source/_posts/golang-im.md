---
title: Golang系列(34)- 实现百万连接的IM服务
date: 2022-11-28 20:42:18
tags: golang
---
前面我们介绍了如何设计一个高性能tcp框架，现在我们基于该框架，利用较少的硬件资源，实现一个支持百万连接的实时聊天(IM)服务

## 即时通讯
>即时通信（ IM ）是指能够即时发送和接收 互联网 消息等的业务。

## 功能设计
本文将围绕以下三个基础功能来实现一个简易但性能强悍的IM

- 私信聊天：用户A可以向用户B发送消息，系统将该消息写入用户B的信箱。
- 群组聊天：用户A加入某个群组后发送消息，系统将该消息直接广播给其他群组成员。
- 查询历史消息：用户可以指定某个聊天会话查询历史消息。

## 扩展
以下功能不做实现，但如果一个更完善的IM服务，需要具备以下特性，这里只是简单介绍下方案：
- 如何保证消息的顺序性?
> 消息ID采用可比较性的规则，比如Mongo的ObjectID，Snowflake算法，或者时间戳+随机字符串。客户端根据消息ID来排序显示。

- 如何保证消息的可达率？
> 私信消息先写入信箱，然后给客户端发送新消息通知，让客户端来读取信箱，读取后删除已读消息。
> 群聊消息采用广播的方式，直接给在线的客户端推送消息，不需要做消息回执。离线用户上线后通过拉取历史数据来读取离线消息。

- 如何解决带宽占用太高？
>IM服务里群聊消息广播是带宽占用太高的大头，可以通过以下策略对进行省流：
1.设置默认不接收群聊推送，减少群聊消息的推送量
2.将一个时间片内的消息合并后再推送，同时设置推送的消息条数上限。丢弃多余消息条数，以提供获取历史消息的方式，按照用户触发去查询完整的消息列表。
3.利用压缩算法(如gzip)对消息内容进行压缩

- 如何优化超大群的消息体验？
对于千人/万人/十万人的大群，提高聊天体验，可以从延时性，每秒消息条数等方面入手。
> 1.降低延时：可以按照分片的设计思想，将大量的连接数平均分配到多个分片里，通过多线程的方式同时对分片内的连接推送消息，比如一个十万人的大群，分成1024个分片，每个分片负责100人的推送任务。
2.每秒接收消息条数: 首先限制客户端的发言频率，其次是限制每秒推送的消息总条数。当每秒产生的消息超过上限时，可以采取丢弃或补全策略。
a)丢弃策略：通过限流算法，按照每个会话设置每秒请求数，超过上限的直接不处理。如腾讯采用的策略是每秒40条上限，超过则丢弃。
b)补全策略：全部接收并处理，只是推送时，推送最新N条消息而非全部，设计消息Sequence字段，第n条消息的Sequence字段值为n。客户端根据该字段进行判断是否有消息断层，如果有，通过拉取历史消息来补全，保证会话的完整性。

(PS：我参加的某个游戏聊天综合项目采用补全策略，必须吐槽下。需求方面，切换会话可以通过拉取历史来查看完整的消息列表，出现消息断层的情况只会出现在消息量太大且用户正聚焦在此会话，当消息量太大还补全个毛线。技术方面，客户端实力不行，设计好的补全策略的逻辑都搞不清，加上涉及本地缓存等复杂逻辑，脑子都是懵的，两个字：辣鸡)

- 如何选择聊天数据的存储方案和过期策略？
>可以根据服务的架构来确定存储方案，如下：
1.单点服务：可以采用内存+持久性存储(MongoDB/ElasticSearch等)
2.分布式服务：可以采用分布式缓存(Redis)+持久性存储(MongoDB/ElasticSearch等)
可以根据聊天类型设置过期策略，以分布式服务为例，策略如下：
1.私信为高级，可以采用Redis+Mongo的混合式存储，Redis设置较长的过期时间，Mongo永久存储。缓存过期后可以从Mongo中读取后再次加载到Redis中。且根据会话设置较大的消息条数上限，超过则删除更早的数据。
2.小群为中级，只采用Redis存储，Redis设置较长的过期时间，且根据会话设置较大的消息条数上限，超过则删除更早的数据
3.大群为低级，只采用Redis存储，Redis设置较短的过期时间，且根据会话设置较小的消息条数上限，超过则删除历史数据。


<!--more-->


## 实现
以下是简要IM服务的demo：
- main.go
```go
package main
import (
	"fmt"
	"github.com/ebar-go/ego/utils/runtime/signal"
	"github.com/ebar-go/znet"
	"log"
)

func main() {
	instance := znet.New()

    // 监听tcp
	instance.ListenTCP(":8081")
    // 同时监听websocket
	instance.ListenWebsocket(":8082")

	New().Install(instance.Router())

	if err := instance.Run(signal.SetupSignalHandler()); err != nil {
		log.Fatal(err)
	}
}
```

- handler.go
```go

const (
	ActionLogin               = 1 // 登录
	ActionSendUserMessage     = 2 // 发送私信
	ActionSubscribeChannel    = 3 // 订阅群组
	ActionSendChannelMessage  = 4 // 发送群聊
	ActionQueryHistoryMessage = 5 // 查询历史消息

    ActionNewUserMessageNotify    = 101 // 新私信通知
	ActionNewChannelMessageNotify = 102 // 新群聊通知
)

type Handler struct {
	codec    codec.Codec
	users    *structure.ConcurrentMap[string, *znet.Connection] // 用户
	channels *structure.ConcurrentMap[string, *Channel] // 群组
}

func New() *Handler {
	return &Handler{
		codec:    codec.NewJsonCodec(),
		users:    structure.NewConcurrentMap[string, *znet.Connection](),
		channels: structure.NewConcurrentMap[string, *Channel](),
	}
}

// Install 初始化路由
func (chat *Handler) Install(router *znet.Router) {
	router.Route(ActionLogin, znet.StandardHandler(chat.login))
	router.Route(ActionSendUserMessage, znet.StandardHandler(chat.sendUserMessage))
	router.Route(ActionSubscribeChannel, znet.StandardHandler(chat.subscribeChannel))
	router.Route(ActionSendChannelMessage, znet.StandardHandler(chat.sendChannelMessage))
	router.Route(ActionQueryHistoryMessage, znet.StandardHandler(chat.queryHistoryMessage))
}

// login 登录
func (handler *Handler) login(ctx *znet.Context, req *LoginRequest) (resp *LoginResponse, err error) {
	uid := uuid.NewV4().String()
	ctx.Conn().Property().Set("uid", uid)
	ctx.Conn().Property().Set("name", req.Name)
	handler.users.Set(uid, ctx.Conn())

	resp = &LoginResponse{ID: uid}
	return
}

// sendUserMessage 发送私信
func (handler *Handler) sendUserMessage(ctx *znet.Context, req *SendUserMessageRequest) (resp *SendUserMessageResponse, err error) {
	receiver, err := handler.users.Find(req.ReceiverID)
	if err != nil {
		return nil, errors.WithMessage(err, "find receiver")
	}

	packet := codec.NewPacket(handler.codec)

	message := Message{
		ID:      "msg" + uuid.NewV4().String(),
		Content: req.Content,
		Sender: User{
			ID:   ctx.Conn().GetStringFromProperty("uid"),
			Name: ctx.Conn().GetStringFromProperty("name"),
		},
		CreatedAt: time.Now().UnixMilli(),
	}
	p, err := packet.EncodeWith(ActionNewUserMessageNotify, 1, message)

	if err != nil {
		return nil, errors.WithMessage(err, "encode packet")
	}
	if _, err = receiver.Write(p); err != nil {
		return nil, errors.WithMessage(err, "write message")
	}

	resp = &SendUserMessageResponse{ID: message.ID}
	return
}

type Channel struct {
	Name    string `json:"name"`
	Members []string
}

// subscribeChannel 订阅群组
func (handler *Handler) subscribeChannel(ctx *znet.Context, req *SubscribeChannelRequest) (resp *SubscribeChannelResponse, err error) {
	channel, exist := handler.channels.Get(req.Name)
	if !exist {
		channel = &Channel{Name: req.Name, Members: make([]string, 0, 100)}
		channel.Members = append(channel.Members, ctx.Conn().ID())
		handler.channels.Set(req.Name, channel)
		return
	}

	uid := ctx.Conn().GetStringFromProperty("uid")
	for _, member := range channel.Members {
		if member == uid {
			return
		}
	}

	channel.Members = append(channel.Members, uid)

	return
}

// sendChannelMessage 发送群聊
func (handler *Handler) sendChannelMessage(ctx *znet.Context, req *SendChannelMessageRequest) (resp *SendChannelMessageResponse, err error) {
	channel, err := handler.channels.Find(req.Channel)
	if err != nil {
		return nil, errors.WithMessage(err, "get channel")
	}

	packet := codec.NewPacket(handler.codec)

	message := ChannelMessage{
		Message: Message{
			ID:      "msg" + uuid.NewV4().String(),
			Content: req.Content,
			Sender: User{
				ID:   ctx.Conn().GetStringFromProperty("uid"),
				Name: ctx.Conn().GetStringFromProperty("name"),
			},
			CreatedAt: time.Now().UnixMilli(),
		},
		Channel: channel.Name,
	}
	p, err := packet.EncodeWith(ActionNewChannelMessageNotify, 1, message)

	if err != nil {
		return nil, errors.WithMessage(err, "encode packet")
	}

    // TODO 实现异步通知
	for _, member := range channel.Members {
		receiver, err := handler.users.Find(member)
		if err != nil {
			continue
		}
		if _, err = receiver.Write(p); err != nil {
			continue
		}
	}

	resp = &SendChannelMessageResponse{ID: message.ID}
	return
}
```

- dto.go
```go

type LoginRequest struct {
	Name string `json:"name"`
}
type LoginResponse struct {
	ID string `json:"id"`
}

type SendUserMessageRequest struct {
	ReceiverID string `json:"receiverId"`
	Content    string `json:"content"`
}
type SendUserMessageResponse struct {
	ID string `json:"id"`
}

type SubscribeChannelRequest struct {
	Name string `json:"name"`
}
type SubscribeChannelResponse struct{}

type SendChannelMessageRequest struct {
	Channel string `json:"channel"`
	Content string `json:"content"`
}
type SendChannelMessageResponse struct {
	ID string `json:"id"`
}

type QueryHistoryMessageRequest struct{}
type QueryHistoryMessageResponse struct{}

type User struct {
	ID   string `json:"id"`
	Name string `json:"name"`
}
type Message struct {
	ID        string `json:"id"`
	Content   string `json:"content"`
	Sender    User   `json:"sender"`
	CreatedAt int64  `json:"createdAt"`
}

type ChannelMessage struct {
	Message
	Channel string `json:"channel"`
}

```
