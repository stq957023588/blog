# Docker

## Docker命令

* 查看所有镜像

```
docker images
```

* 查看所有容器

```shell
docker ps
```

* 进入运行中的容器

```shell
docker exec -it [容器ID] /bin/sh
```

* 将容器打包成镜像

```shell
docker commit [OPTIONS] CONTAINER [REPOSITORY[:TAG]]
# OPTIONS参数说明；
#-a :提交的镜像作者；
#-c :使用Dockerfile指令来创建镜像；
#-m :提交时的说明文字；
#-p :在commit时，将容器暂停
```



### docker network

```shell
# 查看所有网络
docker network ls

# 创建网络
docker network create [网络名称]

# 删除网络
docker network rm [网络ID]

#将容器添加进网络中
docker network connect [网络名称] [容器名称]

#将容器从网路中一处
docker network disconnect [网络名称] [容器名称]

#查看对应网络详情
docker network inspect [网络名称]
```

### docker cp

* 复制宿主机的文件到容器

  ```shel
  docker cp [宿主机文件路径] [容器名称]:[容器文件夹]
  ```

* 复制容器文件到宿主机

  ```shel
  docker cp [容器名称]:[容器文件] [宿主机文件路径]
  ```


### docker run 

* -p <对外端口>:<Docker内部端口> 端口映射
* --name <容器名称> 定义容器名称
* --net <网络名称> 指定网络
* -e "<参数名称>=<参数值>" 定义环境变量
* --link <映射名称>:<镜像名称>  内部连接,如Springboot项目连接Docker内部的Mysql,可使用此来进行连接

```shell
#使用后的mysql url:jdbc:mysql://mysql:3306/fool
--link mysql:mysql
```



## 常用镜像命令

### mysql命令

- 普通启动

  ```shell
  docker run -d --name mysql8 -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 mysql:latest --lower_case_table_names=1
  ```

- 导出指定database数据

  ```shell
  docker exec -it  [容器名称] mysqldump -u[用户名] -p[密码] [database] > [保存到本地的SQL文件地址]
  ```

- 导出全部数据以及表结构

  ```shell
  docker exec -it  [容器名称] mysqldump -u[用户名] -p[密码] --all-databases > [保存到本地的SQL文件地址]
  ```

  

## Dockerfile

* ARG, 用于build时添加的参数,使用时带上 --build-arg

```dockerfile
FROM openjdk:8u272
ARG DEPENDENCY
ADD ${DEPENDENCY}livoltek_email_demo-0.0.1-SNAPSHOT.jar email.jar
EXPOSE 9999
ENTRYPOINT ["java", "-jar" ,"email.jar","--spring.profiles.active=${PROFILE}"]
```

使用

```she
docker build -t --build-arg DEPENDENCY=D:\ email:1.0.0 .
```



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

```dockerfile
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