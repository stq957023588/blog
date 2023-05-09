#  Docker

## Docker Desktop安装注意

### 修改镜像源

```json
{
  "registry-mirrors": [
    "https://registry.docker-cn.com"
  ]
}
```



### 修改镜像存放位置

#### WSL虚拟环境

参考

> [Docker Desktop更改镜像存储位置_feir_2011的博客-CSDN博客_docker修改镜像存储位置](https://blog.csdn.net/feir_2011/article/details/124148825)

## Docker命令

查看容器日志

```shell
docker logs [容器名称]
```

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

# 创建网络，并固定IP范围，只有使用了subnet才可以使容器在启动时指定ip
docker network create --subnet=172.11.0.0/20 [网络名称]

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

> 如果需要复制文件夹下所有文件而不是文件夹本身,使用``/.``结尾

### docker run 

* -p <对外端口>:<Docker内部端口> 端口映射
* --name <容器名称> 定义容器名称
* --net <网络名称> 指定网络
* --ip <IP地址> 在网络中指定ip地址，该网络创建时必须带--subnet参数
* -e "<参数名称>=<参数值>" 定义环境变量
* -v [宿主机文件路径]:[容器文件路径] 容器文件路径,必须是在创建镜像时配置好的才可以使用,不是所有路径都可以的
* --link <映射名称>:<镜像名称>  内部连接,如Springboot项目连接Docker内部的Mysql,可使用此来进行连接

```shell
#使用后的mysql url:jdbc:mysql://mysql:3306/fool
--link mysql:mysql
```



## 常用镜像命令

### mysql

容器名称 mysql

宿主机访问使用端口3307

root账号密码123456

忽略大小写

使用fool网络，并固定ip172.18.0.4

```shell
docker run -d --name mysql -p 3307:3306 -e MYSQL_ROOT_PASSWORD=123456 --net fool --ip 172.18.0.4 mysql:latest --lower_case_table_names=1
```

导出指定database数据

```shell
docker exec -it  [容器名称] mysqldump -u[用户名] -p[密码] [database] > [保存到本地的SQL文件地址]
```

导出全部数据以及表结构

```shell
docker exec -it  [容器名称] mysqldump -u[用户名] -p[密码] --all-databases > [保存到本地的SQL文件地址]
```

### redis

带密码启动(密码:123456)

宿主机访问使用端口6380

使用fool网络，需要先创建fool网络

固定fool中的ip为172.18.0.3，使用固定IP，网络创建时必须带--subnet参数指定网段

```shell
docker run -itd --name redis -p 6380:6379 --net fool --ip 172.18.0.3 redis --requirepass 123456
```

### minio

末尾的/data是指定minio文件存储位置（容器内部的位置）

将数据挂载在D盘data文件夹，配置文件挂载在D盘minio/config文件夹下

```shell
docker run -p 8000:9000 -p 8001:9001 --name minio -v D:\data:/data -v D:\minio\config:/root/.minio minio/minio server --console-address ":9000" --address ":9001" /data
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

## Centos安装Docker

### 下载安装包安装

下载安装包

```shell
wget https://download.docker.com/linux/centos/7/x86_64/stable/Packages/docker-ce-20.10.9-3.el7.x86_64.rpm
```

安装

```shell
sudo yum install docker-ce-20.10.9-3.el7.x86_64.rpm
```

可能会报错

```shell
Error: Package: containerd.io-1.6.4-3.1.el7.x86_64 (docker-ce-stable)
           Requires: container-selinux >= 2:2.74
           Available: 2:container-selinux-2.42-1.gitad8f0f7.el7.noarch (extras)
               container-selinux = 2:2.42-1.gitad8f0f7.el7
           Available: 2:container-selinux-2.55-1.el7.noarch (extras)
               container-selinux = 2:2.55-1.el7
           Available: 2:container-selinux-2.66-1.el7.noarch (extras)
               container-selinux = 2:2.66-1.el7
Error: Package: 3:docker-ce-20.10.9-3.el7.x86_64 (/docker-ce-20.10.9-3.el7.x86_64)
           Requires: container-selinux >= 2:2.74
           Available: 2:container-selinux-2.42-1.gitad8f0f7.el7.noarch (extras)
               container-selinux = 2:2.42-1.gitad8f0f7.el7
           Available: 2:container-selinux-2.55-1.el7.noarch (extras)
               container-selinux = 2:2.55-1.el7
           Available: 2:container-selinux-2.66-1.el7.noarch (extras)
               container-selinux = 2:2.66-1.el7
Error: Package: docker-ce-rootless-extras-20.10.15-3.el7.x86_64 (docker-ce-stable)
           Requires: fuse-overlayfs >= 0.7
Error: Package: docker-ce-rootless-extras-20.10.15-3.el7.x86_64 (docker-ce-stable)
           Requires: slirp4netns >= 0.4
 You could try using --skip-broken to work around the problem
 You could try running: rpm -Va --nofiles --nodigest
```

复制第一个报错后面的依赖名称(这里是containerd.io-1.6.4-3.1.el7.x86_64),复制,依赖查找地址 [Download](https://download.docker.com/linux/centos/7/x86_64/stable/Packages/)

下载安装依赖

```shell
wget https://download.docker.com/linux/centos/7/x86_64/stable/Packages/containerd.io-1.6.4-3.1.el7.x86_64.rpm
```

安装依赖

```shell
yum -y install containerd.io-1.6.4-3.1.el7.x86_64.rpm
```

如果依赖安装也报错,那么安装一下依赖

```shell
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
yum install epel-release -y
yum install container-selinux -y
```

然后继续安装containerd.io-1.6.4-3.1.el7.x86_64

```shell
yum install containerd.io-1.6.4-3.1.el7.x86_64.rpm
```

最后安装Docker

```shell
yum install docker-ce-20.10.9-3.el7.x86_64.rpm
```

参考

> [官方文档](https://docs.docker.com/engine/install/centos/#install-using-the-repository)
>
> [containerd.io安装异常解决办法](https://blog.csdn.net/weixin_56160310/article/details/120970518)

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

## 制作带项目的Tomcat镜像

Dockerfile

```dockerfile
FROM tomcat:9.0.44-jdk8
# 创建tomcat用户
ADD tomcat-users.xml conf/tomcat-users.xml
# Linux下tomcat不放manager.xml 可能无法访问项目管理
ADD manager.xml conf/Catalina/localhost/manager.xml
# 在版本9以上时,docker镜像下webapps下面是没有东西的,东西都在webapps.dist下,所以需要删除源webapps,重命名webapps.dist
RUN rm -rf webapps && mv webapps.dist webapps
ADD email.war webapps/email.war
EXPOSE 8080
CMD ["bin/catalina.sh", "run"]

```

tomcat-users.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<tomcat-users xmlns="http://tomcat.apache.org/xml"
              xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
              xsi:schemaLocation="http://tomcat.apache.org/xml tomcat-users.xsd"
              version="1.0">
    <role rolename="manager-gui"/>
    <user username="admin" password="1234" roles="manager-gui"/>
</tomcat-users>

```

manager.xml

```xml
<Context privileged="true" antiResourceLocking="false"
         docBase="${catalina.home}/webapps/manager">
    <Valve className="org.apache.catalina.valves.RemoteAddrValve" allow="^.*$" />
</Context>
```

## ShardingSphere-Proxy镜像

1. 下载并解压[ShardingSphere-Proxy](https://www.apache.org/dyn/closer.lua/shardingsphere/5.3.1/apache-shardingsphere-5.3.1-shardingsphere-proxy-bin.tar.gz)
2. 准备Dockerfile

```dockerfile
FROM openjdk:8
COPY apache-shardingsphere-5.3.1-shardingsphere-proxy-bin /usr/local/shardingsphere-proxy
COPY mysql-connector-java-8.0.29.jar /usr/local/shardingsphere-proxy/ext-lib/
WORKDIR /usr/local/shardingsphere-proxy
EXPOSE 3307
ENTRYPOINT bin/start.sh && tail -10f /usr/local/shardingsphere-proxy/logs/stdout.log
```

3. 将Dockerfile和解压后的ShardingSphere-Proxy文件夹放在同一目录下
4. 在Dockerfile目录下运行build命令

```shell
docker build -t proxy:1.0.0 .
```

5. 运行镜像

```shell
docker run -d -p 3309:3307 --net migration --name proxy -v D:\Work\WorkSpace\dockerfile\shardingsphere-proxy\conf:/usr/local/shardingsphere-proxy/conf proxy:1.0.0
```



