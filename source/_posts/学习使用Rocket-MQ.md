---
title: 学习使用Rocket MQ
date: 2021-08-03 09:28:10
tags:
---

## 前言

- 当我们学习完 Rocket MQ 的单机部署时，我们可以进行一个简单的收发消息
- 需要编写的函数 ：生产者，消费者
- [架构地址](https://github.com/apache/rocketmq/blob/master/docs/cn/architecture.md)
- 以下所有案例均来自[官方example](https://github.com/apache/rocketmq-client-go/tree/native/examples)
<!--more-->
## 1.生产者

> [参考地址](https://github.com/apache/rocketmq-client-go/blob/native/examples/producer/simple/main.go)

```
COPY
package main

import (
    "context"
    "fmt"
    "github.com/apache/rocketmq-client-go/v2"
    "github.com/apache/rocketmq-client-go/v2/primitive"
    "github.com/apache/rocketmq-client-go/v2/producer"
    "os"
    "strconv"
)

func main() {
    p, _ := rocketmq.NewProducer(
        // 设置  nameSrvAddr
        // nameSrvAddr 是 Topic 路由注册中心
        producer.WithNameServer([]string{"118.89.121.211:9876"}),
        // 设置 Retry 重连次数
        producer.WithRetry(2),
		
    )
    // 开始连接
    err := p.Start()
    if err != nil {
        fmt.Printf("start producer error: %s", err.Error())
        os.Exit(1)
    }

    // 设置节点名称
    topic := "test"
    // 循坏发送信息
    for i := 0; i < 10; i++ {
        msg := &primitive.Message{
            Topic: topic,
            Body:  []byte("Hello RocketMQ Go Client" + strconv.Itoa(i)),
        }
        // 发送信息
        res, err := p.SendSync(context.Background(),msg)
        if err != nil {
            fmt.Printf("send message error:%s\n",err)
        }else {
            fmt.Printf("send message success: result=%s\n",res.String())
        }
    }

    // 关闭生产者
    err = p.Shutdown()
    if err != nil {
        fmt.Printf("shutdown producer error:%s",err.Error())
    }
}
```

## 2.消费者

> [参考地址](https://github.com/apache/rocketmq-client-go/blob/native/examples/consumer/simple/main.go)

```
COPY
func main() {
	// 设置推送消费者
	c, _ := rocketmq.NewPushConsumer(
		consumer.WithGroupName("testGroup"),
		consumer.WithNameServer([]string{"118.89.121.211:9876"}),
		)
	// topic节点是 - "test" ext 为获取到的数据参数
	err := c.Subscribe("test", consumer.MessageSelector{}, func(ctx context.Context, ext ...*primitive.MessageExt) (consumer.ConsumeResult, error) {
		for i := range ext {
			fmt.Printf("subscribe callback:%v \n", ext[i])
		}
		return consumer.ConsumeSuccess, nil
	})
	if err != nil {
		fmt.Println(err.Error())
	}
	// 开始
	err = c.Start()
	if err != nil {
		fmt.Println(err.Error())
		os.Exit(-1)
	}
	time.Sleep(time.Hour)
	err = c.Shutdown()
	if err != nil {
		fmt.Printf("shutdown Consumer error:%s",err.Error())
	}
}
```


