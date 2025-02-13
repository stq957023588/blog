# Elasticsearch

> 该文档适用于8.0.0版本

## 倒排索引

参考：

[Elasticsearch-入门反向索引 - 知乎](https://zhuanlan.zhihu.com/p/341723054)

## 安装

### window下安装

下载压缩包，并解压

进入bin目录，运行``elasticsearch.bat``

注册为服务

```shell
elasticsearch-service.bat install
```

卸载服务

```shell
elasticsearch-service.bat remove
```



### Docker下安装

```shell
docker run --name es01 --net elastic -p 9201:9200 -p 9301:9300 -e "discovery.type=single-node" -e "network.host=es01" -e "discovery.seed_hosts=['es01']" -it docker.elastic.co/elasticsearch/elasticsearch:7.17.0
```

第一次启动会生成证书,账号密码,以及用于Kibana的token,记得保存

此处启动命令如果带上-it 则会打印账号密码以及token,

### 集群

在第一个节点中运行``elasticsearch-create-enrollment-token``生成token

启动第二个节点

```sh
docker run -e ENROLLMENT_TOKEN="<token>" --name es02 --net elastic -it docker.elastic.co/elasticsearch/elasticsearch:8.0.0
```

如果在启动第二个节点时,第一个节点容器自动关闭,则在需要修改所有节点的JVM heap大小

启动设置

```sh
docker run -e ES_JAVA_OPTS="-Xms1g -Xmx1g" -e ENROLLMENT_TOKEN="<token>" --name es02 -p 9201:9200 --net elastic -it docker.elastic.co/elasticsearch/elasticsearch:8.0.1
```

## bin目录下的命令

重置密码

```shell
elasticsearch-reset-password -u elastic --url https://localhost:9200
```

获取kibana使用的token

```shell
elasticsearch-create-enrollment-token -s kibana --url "https://127.0.0.1:9200"
```

获取用于集群的token

```shell
elasticsearch-create-enrollment-token -s node --url "https://127.0.0.1:9200"
```



## 证书(crt)

elasticsearch8.0.0,自动生成证书,用于Logstash传输数据到Elasticsearch

```
docker cp [容器名称]:/usr/share/elasticsearch/config/certs/http_ca.crt [本机地址(文件夹)]
```

## 中文分词

使用IK,[下载地址](https://github.com/medcl/elasticsearch-analysis-ik)

## RESTful API

查看所有索引

```
GET /_cat/indices
```



添加文档

```
POST /<索引名称>/_doc
{

}
```

## 创建ApiKey

name：名称

expiration：过期时间

```
POST /_security/api_key
{
  "name": "my-api-key",
  "expiration": "1d",   
  "role_descriptors": { 
    "role-a": {
      "cluster": ["all"],
      "indices": [
        {
          "names": ["index-a*"],
          "privileges": ["read"]
        }
      ]
    },
    "role-b": {
      "cluster": ["all"],
      "indices": [
        {
          "names": ["index-b*"],
          "privileges": ["all"]
        }
      ]
    }
  },
  "metadata": {
    "application": "my-application",
    "environment": {
       "level": 1,
       "trusted": true,
       "tags": ["dev", "staging"]
    }
  }
}
```



## 对PDF等文件进行索引

添加中文分词器(IK)

安装插件

```shell
bin/elasticsearch-plugin install ingest-attachment
```

创建管道

```shell
curl -XPUT 'ES_HOST:ES_PORT/_ingest/pipeline/attachment?pretty' -H 'Content-Type: application/json' -d '{
 "description" : "Extract attachment information encoded in Base64 with UTF-8 charset",
 "processors" : [
   {
     "attachment" : {
       "field" : "data"
     }
   }
 ]
}'
```

创建索引

```shell
curl -XPUT 'ES_HOST:ES_PORT/{索引名称}' -H 'Content-Type: application/json' -d '{
  "mappings": {
    "properties": {
            "attachment.content": {
                "type": "text",
                "analyzer": "ik_max_word",
                "search_analyzer": "ik_smart"
            }
        }
  }
}'
```

添加文档

```shell
curl -XPUT 'ES_HOST:ES_PORT/test_index/_doc?pipeline=attachment&pretty' -H 'Content-Type: application/json' -d '{
 "data": "UWJveCBlbmFibGVzIGxhdW5jaGluZyBzdXBwb3J0ZWQsIGZ1bGx5LW1hbmFnZWQsIFJFU1RmdWwgRWxhc3RpY3NlYXJjaCBTZXJ2aWNlIGluc3RhbnRseS4g"
}'
```

查询文档

```
GET /test_index/_search
{
  "_source": false,
  "fields": [
    "attachment.content"
  ], 
  "query": {
    "match": {
      "attachment.content": {
        "query": "第一次"
      }
    }
  },
  "highlight": {
    "fields": {
      "attachment.content": {
        "pre_tags": "<em>",
        "post_tags": "</em>"
      }
    }
  }
}
```

## Elasticsearch-Java

依赖

```xml
<dependency>
    <groupId>co.elastic.clients</groupId>
    <artifactId>elasticsearch-java</artifactId>
    <version>8.0.0</version>
    <exclusions>
        <exclusion>
            <groupId>jakarta.json</groupId>
            <artifactId>jakarta.json-api</artifactId>
        </exclusion>
    </exclusions>
</dependency>

<dependency>
    <groupId>jakarta.json</groupId>
    <artifactId>jakarta.json-api</artifactId>
    <version>2.0.1</version>
</dependency>
```

> 在springboot使用必须指定jakarta.json-api的版本,因为springboot本身也是用了jakarta.json-api,但版本是1.x,会导致冲突

客户端创建

```java
public ElasticsearchClient elasticsearchClient() throws Exception {
    // 读取TSL证书
    Path caCertificatePath = Paths.get("http_ca.crt");
    CertificateFactory factory = CertificateFactory.getInstance("X.509");
    Certificate trustedCa;
    try (InputStream is = Files.newInputStream(caCertificatePath)) {
        trustedCa = factory.generateCertificate(is);
    }
    KeyStore trustStore = KeyStore.getInstance("pkcs12");
    trustStore.load(null, null);
    trustStore.setCertificateEntry("ca", trustedCa);
    SSLContextBuilder sslContextBuilder = SSLContexts.custom().loadTrustMaterial(trustStore, null);
    final SSLContext sslContext = sslContextBuilder.build();

    // 设置账号密码
    final CredentialsProvider credentialsProvider = new BasicCredentialsProvider();
    credentialsProvider.setCredentials(AuthScope.ANY, new UsernamePasswordCredentials("elastic", "gJrWUCkuY=Mk3z_jH7rE"));

    // 设置请求客户端
    RestClient restClient = RestClient.builder(new HttpHost("localhost", 9201, "https"))
            .setHttpClientConfigCallback(httpClientBuilder -> httpClientBuilder.setSSLContext(sslContext).setDefaultCredentialsProvider(credentialsProvider))
            .build();

    // 使用Jackson映射器创建传输层
    ElasticsearchTransport transport = new RestClientTransport(restClient, new JacksonJsonpMapper());
    // 创建API客户端
    return new ElasticsearchClient(transport);
}
```

创建一个名称为``springboot``的索引

```java
public void testCreateIndex() {
    try {
        CreateIndexResponse createIndexResponse = elasticsearchClient.indices().create(c -> c.index("springboot"));
        Boolean acknowledged = createIndexResponse.acknowledged();
        System.out.println("create index result:" + acknowledged);
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

添加普通文档

```java
public void testCreateDocument() {
    try {
        Student student = new Student();
        elasticsearchClient.create(new CreateRequest.Builder<EncodeFile>().index("test_index").id("1").document(student).build());

    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

索引文件

```java
public void testCreateDocument() {
    try {
        File dir = new File("E:\\temporary\\elasticsearch-document");
        File[] files = dir.listFiles();
        Base64.Encoder encoder = Base64.getEncoder();
        assert files != null;
        int i = 0;
        for (File f : files) {
            byte[] bytes = FileUtils.readFileToByteArray(f);
            String encode = encoder.encodeToString(bytes);
            EncodeFile encodeFile = new EncodeFile();
            encodeFile.setFilename(f.getName());
            encodeFile.setData(encode);

            elasticsearchClient.create(new CreateRequest.Builder<EncodeFile>().pipeline("attachment").index("test_index").id(i + "").document(encodeFile).build());
            i++;
        }

    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

查询文档

```java
public void testSearch() {
    try {
        SearchResponse<EncodeFile> search = elasticsearchClient.search(
                e -> e.index("test_index")
                        .source(new SourceConfig.Builder().fetch(false).build())
                        .fields(Stream.of(new FieldAndFormat.Builder().field("attachment.content").build()).collect(Collectors.toList()))
                        .query(q -> q.intervals(
                                i -> i.field("attachment.content").allOf(a -> a.intervals(i2 -> i2.match(m2 -> m2.query("第一次").analyzer("ik_smarter"))))
                        ))
                , EncodeFile.class);
        List<Hit<EncodeFile>> hits = search.hits().hits();
        hits.forEach(hit -> System.out.println(hit.fields()));
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

以上查询等价于一下查询

```json
GET test_index/_search
{
  "_source": false,
  "fields": [
    "attachment.content"
  ], 
  "query": {
    "match": {
      "attachment.content": {
        "query": "第一次",
        "analyzer": "ik_smart"
      }
    }
  }
}
```



## 问题

1. 8.0.0版本访问elasticsearch必须时https(哪怕是本地)

# Kibana

## 启动

### Docker启动

```shell
docker run -d --name kiba --net elastic -p 5602:5601 docker.elastic.co/kibana/kibana:8.0.0
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

   