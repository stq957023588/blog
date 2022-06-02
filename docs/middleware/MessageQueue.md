# 消息队列

## 如何保证消息的不重复消费

保证消息的不重复消费即需要实现消费消息的接口的幂等性

首先需要消息带有一个全局唯一的标识,在消费者端对全局唯一标识进行校验

在涉及到金钱等场景需要使用强校验,使用全局唯一标识去数据库表中查询是否存在记录,如果有就return,如果没有就执行业务逻辑,

在不重要的场景使用弱校验即可,将全局唯一标识存入redis缓存中,并设置失效时间,在一定时间内通过判断redis中是否存在来判断

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





