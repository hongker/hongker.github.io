---
title: Golang代码优化09-日志
date: 2020-04-01 21:48:16
tags:
---
本文介绍在golang中的日志使用方式。

## 官方自带Log包
- demo
```go
package main

import "log"

func init() {
	// 设置log前缀
	log.SetPrefix("TRACE: ")

	// 设置日志内容：日期、时间、文件
	log.SetFlags(log.Ldate | log.Ltime |log.Llongfile)
}

func main()  {
	// 打印标准日志
	log.Println("message")

	// 使用格式化输出日志
	log.Printf("the result is :%s", "log")

	// 致命错误,直接退出程序
	log.Fatal("fatal message")
}
```

## 自定义Logger
指定目录存储日志
```go
package main

import (
	"io"
	"log"
	"os"
)

var (
	LogComponent *Logger
)
// Logger
type Logger struct {
	// logPath 日志存储目录
	logPath string
	// info 记录信息
	info *log.Logger
	// error 记录错误
	error *log.Logger
	// 其他如debug,trace,warning如上
}

func (l *Logger) Info(message string) {
	l.info.Println(message)
}

func (l *Logger) Error(message string) {
	l.error.Println(message)
}

func getFile(filePath string) *os.File {
	file, err := os.OpenFile(filePath, os.O_CREATE|os.O_WRONLY|os.O_APPEND, 0666)
	if err != nil {
		log.Fatalf("Failed to open %s:%v\n", filePath, err)
	}

	return file
}
// NewLogger 
func NewLogger(logPath string) *Logger {
	logger := &Logger{logPath:logPath}

	logger.info = log.New(io.MultiWriter(getFile(logPath + "info.log"), os.Stderr),
		"INFO: ",
		log.Ldate|log.Ltime|log.Llongfile)

	logger.error = log.New(io.MultiWriter(getFile(logPath+"error.log"), os.Stderr),
		"ERROR: ",
		log.Ldate|log.Ltime|log.Llongfile)

	return logger
}

func init() {
	// 将日志放在log目录下，用绝对路径会更好点
	LogComponent = NewLogger("./log/")
}

func main() {
	LogComponent.Info("Special Information")
	LogComponent.Error("Something has failed")
}
```
<!--more-->

## 使用logrus
- 安装
```
go get github.com/sirupsen/logrus
```
示例：
```go
func main() {
    // 实例化,实际项目中一般用全局变量来初始化一个日志管理器
	logger := logrus.New()

	// 设置日志内容为json格式
	logger.SetFormatter(&logrus.JSONFormatter{
		TimestampFormat:   "2006-01-02 15:04:05", // 时间格式
		DisableTimestamp:  false, //是否禁用日期
		DisableHTMLEscape: false, // 是否禁用html转义
		DataKey:          "" ,
		FieldMap:          logrus.FieldMap{
			logrus.FieldKeyMsg : "content", // 修改"msg"字段名称为"content"
		},
		CallerPrettyfier:  nil,
		PrettyPrint:       false, // 是否需要格式化
	})

	// 设置记录日志的最高等级，比如这里设置的info等级，那么只有比info低级的 Warn(), Error(), Fatal()以及自身Info()能够打印
    logger.Level = logrus.InfoLevel
    
    // 指定日志的输出为文件，默认是os.Stdout
	f, err := os.OpenFile("logrus.log", os.O_CREATE|os.O_WRONLY|os.O_APPEND, 0666)
	if err != nil {
		log.Fatal("Failed to open:", err.Error())
	}
	logger.Out = f

	logger.Info("this is info level")
	logger.Debug("this is debug level")
	logger.Warning("this is waring level")
    logger.Error("this is error level")
    // 日志+致命错误，程序直接退出
	//logger.Fatal("this is fatal level")

    // 在日志内容里增加一个字段title,它的值为user
    // 输出：{"content":"this is from user service log","level":"info","time":"2020-04-02 14:01:29","title":"user"}
	logger.WithField("title","user").Info("this is from user service log")
}
```

## uber的zap
号称超高性能的日志库
- 安装
```
go get -u go.uber.org/zap
```

- demo
```go
func main() {
    cfg := zap.Config{
		// Level 最小启用的日志等级,可以通过cfg.Level.SetLevel()动态调整
		Level:       zap.NewAtomicLevelAt(zap.DebugLevel),
		// Development 当开启时，使用DPanic()，会直接panic,调试时极其方便
		Development: false,
		// 格式设为json
		Encoding:    "json",
		EncoderConfig: zapcore.EncoderConfig{
			TimeKey:        "time", //时间的key
			LevelKey:       "level", // 日志等级的key
			NameKey:        "logger",
			CallerKey:      "caller",
			MessageKey:     "msg",
			StacktraceKey:  "trace",
			LineEnding:     zapcore.DefaultLineEnding,
			EncodeLevel:    zapcore.LowercaseLevelEncoder,
			EncodeDuration: zapcore.SecondsDurationEncoder,
            EncodeCaller:   zapcore.ShortCallerEncoder,
            // 设定时间格式
			EncodeTime: func(t time.Time, encoder zapcore.PrimitiveArrayEncoder) {
				encoder.AppendString(fmt.Sprintf("%d%02d%02d_%02d%02d%02d", t.Year(), t.Month(), t.Day(), t.Hour(), t.Minute(), t.Second()))
			},
		},
		// 日志输出目录
		OutputPaths:      []string{"/tmp/zap.log"},
		// 将系统内的error记录到文件的地址
		ErrorOutputPaths: []string{"/tmp/zap.log"},

		// 加入一些初始的字段数据，比如项目名
		InitialFields: map[string]interface{}{
			"system_name": "user",
		},
	}

	logger, err := cfg.Build()
	if err != nil {
		log.Fatal(err.Error())
	}

	defer logger.Sync()
	logger.Info("info", zap.Field{
		Key:       "hello",
		String:    "world",
		Type: zapcore.StringType,
	})
	logger.Error("error", zap.Field{
		Key:       "hello",
		String:    "world",
		Type: zapcore.StringType,
	})
	logger.DPanic("panic", zap.Field{
		Key:       "hello",
		String:    "world",
		Type: zapcore.StringType,
	})
}
```