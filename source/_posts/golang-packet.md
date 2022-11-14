---
title: Golang系列(30)-TCP粘包/拆包问题分析与解决
date: 2022-11-14 14:12:31
tags: golang
---
本文介绍在golang的socket编程中，对粘包问题的处理方式。

## 粘包
>因为TCP是面向流，没有边界，而操作系统在发送TCP数据时，会通过缓冲区来进行优化，例如缓冲区为1024个字节大小。
- 如果一次请求发送的数据量比较小，没达到缓冲区大小，TCP则会将多个请求合并为同一个请求进行发送，这就形成了粘包问题。
- 如果一次请求发送的数据量比较大，超过了缓冲区大小，TCP就会将其拆分为多次发送，这就是拆包。

```
PS: websocket通过底层协议帧解决的粘包问题。
```

## 常见的解决方案
- 将消息分为头部和消息体，头部中保存整个消息的长度，只有读取到足够长度的消息之后才算是读到了一个完整的消息(常用)
- 通过自定义协议进行粘包和拆包的处理
- 发送端在每个包的末尾使用固定的分隔符，例如\r\n。
- 发送端将每个包都封装成固定的长度，比如100字节大小。

<!--more-->

## LengthFieldBasedFrameDecode
实现一个基于消息头包含消息长度的协议的解码器来处理粘包问题
- 协议设计:设置header是长度为4的字节数组，存储的是整个包体的长度length，比如body长度为n，那么length=n+4
```
------------|----------------------
|   头字节   |         Body        |
------------|----------------------
|     4     |          n          |
------------|----------------------
```
- Demo
```go

// LengthFieldBasedFrameDecode implements Decoder interface by decode length field
type LengthFieldBasedFrameDecode struct {
	offset int
	endian binary.Endian
}

func NewDecoder(offset int) Decoder {
	return &LengthFieldBasedFrameDecode{
		offset: offset,
		endian: defaultEndian,
	}
}

func (decoder *LengthFieldBasedFrameDecode) Decode(reader io.Reader) (buf []byte, err error) {
	// read length field of packet
	p := pool.GetByte(decoder.offset)
	defer pool.PutByte(p)
	_, err = io.ReadFull(reader, p)
	if err != nil {
		return
	}

	// read other part of packet
	length := int(decoder.endian.Int32(p)) - decoder.offset
	if length <= 0 {
		// when connection is closed, first read packet length may be successfully, but connection has closed
		err = errors.New("packet exceeded, connection may be closed")
		return
	}
	buf = make([]byte, length)
	_, err = io.ReadFull(reader, buf)
	return
}
```