# GO RabbitMQ

## 简述

RabbitMQ是采用Erlang编程语言实现了高级消息队列协议AMQP (Advanced Message Queuing Protocol)的开源消息代理软件（消息队列中间件）

### 名词

* connection:
* channel:
* consumer:消息的消费者
* producter:消息的生产者,投递方
* exchange:交换机，生产者将消息发送给交换器(交换机),再由交换器将消息路由导对应的队列中
* routingkey:路由件，消息投递给交换器,通常会指定一个 RoutingKey ,通过这个路由键来明确消息的路由规则
* queue:消息队列

### 工作流程

#### 消息生产流程

1. 消息生产者连与RabbitMQ Broker 建立一个连接,建立好了连接之后,开启一个信道Channel
2. 声明一个交换机,并设置其相关的属性(交换机类型,持久化等)
3. 声明一个队列并设置其相关属性(排他性,持久化自动删除等)
4. 通过路由键将交换机和队列绑定起来
5. 消息生产者发送消息给 RabbitMQ Broker , 消息中包含了路由键,交换机等信息,交换机根据接收的路由键查找匹配对应的队列
6. 查找匹配成功,则将消息存储到队列中
7. 查找匹配失败,根据生产者配置的属性选择丢弃或者回退给生产者
8. 关闭信道Channel , 关闭连接

#### 消息消费流程

1. 消息消费者连与RabbitMQ Broker 建立一个连接,建立好了连接之后,开启一个信道Channel
2. 消费者向RabbitMQ Broker 请求消费者相应队列中的消息
3. 等待RabbitMQ Broker 回应并投递相应队列中的消息,消费者接收消息
4. 消费者确认(ack) 接收消息, RabbitMQ Broker 消除已经确认的消息
5. 关闭信道Channel ,关闭连接



## Golnag操作MQ

## 依赖库

go get github.com/streadway/amqp

## 代码

### 生产者

```go
package rabbitmq

import (
	"errors"
	"sync"
	"time"

	"github.com/streadway/amqp"
)

const chanMaxCount = 10000 // 并发数和channMax数量

var once sync.Once // 第一次使用时创建连接
var conn *amqp.Connection // connection TCP连接
var chanQueue = make(chan struct{}, chanMaxCount) // 控制并发的channel


type Client struct {
	Data         string
	ExchangeName string
	QueueName    string
	RoutingKey   string
}

func initRabbitmq() {
  url: = "amqp://login:password@ip:port"
	vhost := "" // 根据自己要求写入vhost
	heartbeat := 30 // 根据自己要求写入心跳
	var err error
	conn, err = amqp.DialConfig(
    url,
		amqp.Config{
			Vhost:     vhost,
			Heartbeat: time.Duration(heartbeat) * time.Second,
			ChannelMax: chanMaxCount,
		},
	)
	if err != nil {
		panic("rabbitmq", "连接Rabbit服务器失败|"+err.Error())
	}
}

func (c *Client) Publish() (err error) {
	once.Do(initRabbitmq)
	if conn == nil || conn.IsClosed() {
		initRabbitmq()
	}
	chanQueue <- struct{}{}
	defer func() {
		<-chanQueue
	}()
	channel, err := conn.Channel()
	if err != nil {
		return errors.New("channel建立失败|" + err.Error())
	}
	defer channel.Close()
	if err := channel.ExchangeDeclare(c.ExchangeName, amqp.ExchangeDirect, true, false, false, false, nil); err != nil {
		return errors.New("Exchange声明失败|" + err.Error())
	}
	if _, err := channel.QueueDeclare(c.QueueName, true, false, false, false, nil); err != nil {
		return errors.New("Queue声明失败|" + err.Error())
	}
	if err := channel.QueueBind(c.QueueName, c.RoutingKey, c.ExchangeName, false, nil); err != nil {
		return errors.New("queue绑定exchange失败|" + err.Error())
	}
	err = channel.Publish(c.ExchangeName, c.RoutingKey, false, false, amqp.Publishing{
		ContentType: "text/plain",
		Body:        []byte(c.Data),
	})
	if err != nil {
		return errors.New("发送MQ消息失败|" + err.Error())
	}
	return err
}
```

> 根据rabbit[官网](https://www.rabbitmq.com/channels.html)对channel的说明：channel是建立在connection TCP上面的更轻量级的连接，每个channel都有一个唯一的channel ID，对于使用多个线程/进程进行处理的应用程序，通常会为每个线程/进程打开一个新通道，而不在它们之间共享通道。
>
> 所以在使用时避免channel复用，可能会出现channel异常导致消息发送失败，channel异常关闭 ``channel/connection is not open``等问题。
>
> 所以我在每次使用时会单独获取一个channel，用完close掉。并发创建的channel最大数量不超过配置中ChannelMax数量，如果超过则会提示``channel id space exhausted`` channel id 不够的提示。

### 消费者
