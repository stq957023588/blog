# SpringCloud

## 使用Nacos作为注册和配置中心

1. 添加依赖

```xml
<!--配置中心依赖-->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
</dependency>
        <!--注册中心依赖-->
<dependency>
<groupId>com.alibaba.cloud</groupId>
<artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
        </
```

2. 新建bootstrap.yml   
   有关nacos的配置必须在bootstrap.yml中,SpringCloud启动会首先读取bootstrap.yml中的数据

```yaml
# 必须配置,作为nacos配置中的配置的DataId前缀
spring:
  application:
    name: consumer
  cloud:
    nacos:
      # 注册中心地址
      discovery:
        server-addr: 127.0.0.1:8849
      # 配置中心地址
      config:
        server-addr: 127.0.0.1:8849
        # 配置中心的配置文件格式,默认properties
        file-extension: yaml
```

3. 在启动类上添加注解

```java
// 动态刷新配置文件数据
@RefreshScope
// 服务发现
@EnableDiscoveryClient
@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

## 熔断器Hystrix

当调用其他微服务的接口时,接口发生报错则进行熔断,调用自定义的接口(此处使用Feign)   
使用FallbackFactory可以根据报错定义调用哪个熔断处理类,而fallback不管报什么错都会调用同一个处理方法

1. 添加Feign依赖

```xml

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

2. 开启熔断器,feign.hystrix.enabled=true
3. 微服务接口

```java
public interface ProviderService {

    String getValue();

}
```

4. 熔断处理类

fallback

```java

@Component
public class ProviderServiceFallback implements ProviderService {
    @Override
    public String getValue() {
        log.warn("fallback");
        return "fallback";
    }
}
```

fallback factory

```java

@Component
public class ProviderServiceFallbackFactory implements FallbackFactory<ProviderService> {
    @Override
    public ProviderService create(Throwable throwable) {
        log.error("Call provider service error", throwable);
        return new ProviderService() {
            @Override
            public String getValue() {
                return "fallback factory ";
            }
        };
    }
}
```

5. 在微服务接口上添加熔断处理类

```java
// name是被调用的微服务名称,fallback是调用的微服务发生报错是调用的方法
@FeignClient(name = "provider", fallback = ProviderServiceFallback.class)
// fallback 和fallbackFactory 不可以同时使用
// @FeignClient(name = "provider", fallbackFactory = ProviderServiceFallbackFactory.class)
public interface ProviderService {
    //被调用的微服务接口地址
    @RequestMapping("/getvalue")
    String getValue();

}
```

## 集成OAuth 2.0

https://www.cnblogs.com/ifme/p/12188982.html