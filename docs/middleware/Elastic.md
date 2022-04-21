# Elasticsearch

## 启动

### Docker启动

```shell
docker run --name es01 --net elastic -p 9201:9200 -p 9301:9300 -e "discovery.type=single-node" -e "network.host=es01" -e "discovery.seed_hosts=['es01']" -it docker.elastic.co/elasticsearch/elasticsearch:7.17.0
```

第一次启动会生成证书,账号密码,以及用于Kibana的token,记得保存

## 证书(crt)

elasticsearch8.0.0,自动生成证书,用于Logstash传输数据到Elasticsearch

```
docker cp [容器名称]:/usr/share/elasticsearch/config/certs/http_ca.crt [本机地址(文件夹)]
```



## 问题

1. 8.0.0版本访问elasticsearch必须时https(哪怕是本地)

# Kibana

## 启动

### Docker启动

```shell
docker run --name kib01 --net elastic -p 5602:5601 docker.elastic.co/kibana/kibana:7.17.0
```

PS: 如果不是8.0.0版本,需要加上参数 -e "ELASTICSEARCH_HOSTS=http://es01:9200" ,用于告诉Kibana Elasticsearch的地址,8.0.0版本使用token告诉Kibana,不需要在启动时添加参数

## Discover

1. 在Stack management/Data Views中创建data view
2. 在Discover 中选择创建的data view

# Logstash

## 启动

### window启动

新建logstash-script.conf文件

```
input {
  tcp {
    host => "127.0.0.1"
    port => 9250
    mode => "server"
    tags => ["tags"]
    codec => plain{charset=>"UTF-8"}
    }

}
output {
    elasticsearch {
        hosts => ["https://127.0.0.1:9201"]
        index => "my-index-%{+YYYY.MM.dd}"
        cacert => "E:/crt/http_ca.crt"
        user => "elastic"
        password => "GfzrF9UupfFEcyiZ1uOp"
    }
    stdout{}
}

```

stdout: 输出到控制台

elasticsearch: 输出到elasticsearch, cacert:是elasticsearch的证书

在安装目录/bin下控制台运行

```shell
.\logstash.bat -f ../pipeline/SpringbootLog.conf
```

### Docker启动

准备docker-compose.yml

```yml
services:
  logstash:
    image: docker.elastic.co/logstash/logstash:8.0.0
    container_name: logstash
    volumes:
      - E:\docker\Logstash\pipeline:/usr/share/logstash/pipeline
      - E:\docker\Logstash\config\logstash.yml:/usr/share/logstash/config/logstash.yml
      - E:\docker\Logstash\config\http_ca.crt:/usr/share/logstash/config/http_ca.crt
    ports:
      - 9250:9250
# 自定义已有的网络
networks:
  default:
    external:
      name: elastic

```

pipeline文件夹下存储*.conf文件

http_ca.crt是elasticsearch启动时生成的文件,需要从elasticsearch启动后复制出来

logstash.yml

```yml
# elasticsearch 用户名
xpack.monitoring.elasticsearch.username: elastic
# elasticsearch 密码
xpack.monitoring.elasticsearch.password: GfzrF9UupfFEcyiZ1uOp
# elasticsearch ip地址 此处如果在docker容器中时,不可以使用容器名称代替ip地址(也许可以,但可能需要其他配置)
xpack.monitoring.elasticsearch.hosts: ["https://172.20.0.2:9200"]
# 证书文件
xpack.monitoring.elasticsearch.ssl.certificate_authority: "/usr/share/logstash/config/http_ca.crt"
```

docker-compose.yml同目录下控制台(cmd)运行 docker compose up -d

## 管道配置文件

### input

tcp输入

```
input {
  tcp {
    host => "127.0.0.1"
    port => 9251
    tags => ["tags"]
    }
}
```

控制台输入

```
input {
	stdin{}
}
```





### output

控制台输出

```
output{
	stdout{}
}
```

elasticsearch输出

```
output{
    elasticsearch {
        hosts => ["https://172.20.0.2:9200"]
        index => "my-index-%{+YYYY.MM.dd}"
        cacert => "/usr/share/logstash/config/http_ca.crt"
        user => "elastic"
        password => "GfzrF9UupfFEcyiZ1uOp"
    }
}
```





### filter

# ELK日志采集

1. 启动ELasticsearch

2. 启动Kibana

3. 启动Logstash

4. 集成Springboot

   依赖:

   ```xml
           <dependency>
               <groupId>net.logstash.logback</groupId>
               <artifactId>logstash-logback-encoder</artifactId>
               <version>7.0.1</version>
           </dependency>
   ```

   logback-srping.xml添加appender

   ```xml
       <appender name="LOGSTASH" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
        	<!-- Logstash d -->   
           <destination>127.0.0.1:9250</destination>
           <queueSize>1048576</queueSize>
           <encoder charset="UTF-8" class="net.logstash.logback.encoder.LogstashEncoder" />
       </appender>
   ```

   