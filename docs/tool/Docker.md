# Docker

## Docker命令

* -e 参数传递,须在dockerfile中使用${PARAM}   

```shell
-e "PARAM=dev"
```

* --link [映射名称]:[镜像名称] 内部连接,如Springboot项目连接Docker内部的Mysql,可使用此来进行连接

```shell
#使用后的mysql url:jdbc:mysql://mysql:3306/fool
--link mysql:mysql
```

* -p [对外端口]:[内部端口]

## 制作Springboot项目镜像

1. pom.xml修改spring-boot-maven-plugin,configuration下添加layers

```xml

<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <layers>
            <enabled>true</enabled>
        </layers>
    </configuration>
</plugin>
```

2. 制作Dockerfile文件   
   dockerfile必须与demo-0.0.1-SNAPSHOT.jar在同一目录下   
   ${PROFILE}可通过docker命令-e在进行参数传递

```
FROM java:11.0.11
ADD demo-0.0.1-SNAPSHOT.jar demo.jar
EXPOSE 7777
ENTRYPOINT ["java","-jar","demo.jar","--spring.profiles.active=${PROFILE}"]
```

3. 运行镜像制作命令

```shell
#必须在dockerfile目录下运行此命令
#demo 为镜像名称,1.0.0.Beta为镜像版本
docker build -t demo:1.0.0.Beta .
```

4. 运行镜像

```shell
#--link可以让此镜像可以连接其他镜像
#-e 后面可跟随需要传递的参数
#-p 端口映射,[对外端口]:[内部端口]
docker run -d -e "PROFILE=docker" --name demo --link redis:redis --link mysql:mysql -p 6667:6666 demo:1.0.0.Beta
```