# 消息队列

## 如何保证消息的不丢失

消息的丢失可能存在于三个地方:

1. 消息生成发送到消息队列
2. 消息队列本身存储丢失
3. 消息队列将消息传递给消费者

为了确保消息成功发送到消息队列,在发送消息的时候会有个异步的回调函数,来获取由消息队列返回的ack,来表示消息队列接受到了消息

消息队列中的存储不丢失由消息中间件来保证

最后一步消息的消费,在消费者完成具体的业务逻辑后,返回消息确认的ack给消息队列

通过以上方法来保证消息的不丢失

## 如何保证消息的不重复消费

保证消息的不重复消费即需要实现消费消息的接口的幂等性

首先需要消息带有一个全局唯一的标识,在消费者端对全局唯一标识进行校验

在涉及到金钱等场景需要使用强校验,使用全局唯一标识去数据库表中查询是否存在记录,如果有就return,如果没有就执行业务逻辑,

在不重要的场景使用弱校验即可,将全局唯一标识存入redis缓存中,并设置失效时间,在一定时间内通过判断redis中是否存在来判断

## 消息积压情况如何处理

消息积压,说明消息产生速度远远大于消费的速度,所以问题是出现在消费者端

首先应急可以临时增加消费者端的数量,然后可以通过日志等判断等判断是需要进行代码优化提高消费速度,又或者进行消费者扩容

# RabbitMQ

## docker安装

拉取镜像

```shell
docker pull rabbitmq			
```

启动rabbitmq

```shell
docker run -d --hostname myrabbit --name some-rabbit -p 15671:15672 -p 5671:5672 -e RABBITMQ_DEFAULT_USER=admin -e RABBITMQ_DEFAULT_PASS=123456 rabbitmq:latest
```

启动管理页面

首先进入容器

```shell
docker exec -it some-rabbit /bin/sh
```

安装rabbitmq插件

```shelll
rabbitmq-plugins enable rabbitmq_management
```

浏览器访问127.0.0.1:15671,输入账号/密码 admin/123456 就可以进入rabbitmq的管理页面

# 一些不错的消息队列文章

[敢在简历上写消息队列，这几个问题必须拿下！](https://mp.weixin.qq.com/s/fEeuSdDFzOjjPWu4x1zd8Q)

> 主要提到了消息丢失问题,重复消息问题以及消息积压情况如何处理,最后是一些rabbitmq的进阶知识方向



