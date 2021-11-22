# 点评/cat

## Docker部署点评/cat

1. 从[GitHub](https://github.com/dianping/cat.git) 上克隆源码

```shell
git clone https://github.com/dianping/cat.git
```

2. 下载依赖分支 mvn-repo

```shell
git clone -b mvn-repo https://github.com/dianping/cat.git
```

3. 将克隆下来的依赖分支,中的org文件夹拖放到本地的maven仓库

4. 修改源码的pom.xml(因为有些依赖已经不存在远程仓库中)

```xml

<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.dianping.cat</groupId>
    <artifactId>parent</artifactId>
    <version>3.0.0</version>
    <name>parent</name>
    <description>Central Application Tracking</description>
    <packaging>pom</packaging>
    <dependencies>
        <dependency>
            <groupId>org.unidal.framework</groupId>
            <artifactId>test-framework</artifactId>
            <!--    从2.4.0修改为2.5.0-->
            <version>2.5.0</version>
        </dependency>
    </dependencies>

    <plugins>
        <plugin>
            <groupId>org.unidal.maven.plugins</groupId>
            <artifactId>codegen-maven-plugin</artifactId>
            <!--    从2.5.8修改为3.0.1-->
            <version>3.0.1</version>
        </plugin>
        <plugin>
            <groupId>org.unidal.maven.plugins</groupId>
            <artifactId>plexus-maven-plugin</artifactId>
            <!--    从2.5.8修改为3.0.1-->
            <version>3.0.1</version>
        </plugin>
    </plugins>
</project>
```

5. 修改pom.xml后在idea中执行package,复制cat-home/target下的war,并重命名为cat.war

6. 从各自官网下载jdk8和tomcat8的tar.gz文件

7. datasources.xml

```xml
<?xml version="1.0" encoding="utf-8"?>

<data-sources>
    <data-source id="cat">
        <maximum-pool-size>3</maximum-pool-size>
        <connection-timeout>1s</connection-timeout>
        <idle-timeout>10m</idle-timeout>
        <statement-cache-size>1000</statement-cache-size>
        <properties>
            <driver>com.mysql.jdbc.Driver</driver>
            <url><![CDATA[jdbc:mysql://172.19.0.3:3306/cat]]></url>
            <user>root</user>
            <password>123456</password>
            <connectionProperties>
                <![CDATA[useUnicode=true&characterEncoding=UTF-8&autoReconnect=true&socketTimeout=120000]]></connectionProperties>
        </properties>
    </data-source>
</data-sources>
```

8. client.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<config mode="client">
    <servers>
        <server ip="127.0.0.1" port="2280" http-port="8080"/>
    </servers>
</config>
```

9. dockerfile

```dockerfile
FROM         centos
MAINTAINER    fool<fool@126.com>
#把宿主机当前上下文的c.txt拷贝到容器/usr/local/路径下
COPY c.txt /usr/local/cincontainer.txt
#把java与tomcat添加到容器中
ADD jdk-8u301-linux-x64.tar.gz /usr/local/
ADD apache-tomcat-8.0.20.tar.gz /usr/local/
#安装vim编辑器
#RUN yum -y install vim
RUN mv /usr/local/apache-tomcat-8.0.20 /usr/local/tomcat
COPY datasources.xml /data/appdatas/cat/
COPY client.xml /data/appdatas/cat/
COPY tomcat-users.xml /usr/local/tomcat/conf/
COPY cat.war /usr/local/tomcat/webapps/
#设置工作访问时候的WORKDIR路径，登录落脚点
ENV MYPATH /usr/local
WORKDIR $MYPATH
#配置java与tomcat环境变量
ENV JAVA_HOME /usr/local/jdk1.8.0_301
ENV CLASSPATH $JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
ENV CATALINA_HOME /usr/local/tomcat
ENV CATALINA_BASE /usr/local/tomcat
ENV PATH $PATH:$JAVA_HOME/bin:$CATALINA_HOME/lib:$CATALINA_HOME/bin
#容器运行时监听的端口
EXPOSE  8080
#启动时运行tomcat
# ENTRYPOINT ["/usr/local/apache-tomcat-8.0.20/bin/startup.sh" ]
# CMD ["/usr/local/apache-tomcat-8.0.20/bin/catalina.sh","run"]
CMD /usr/local/tomcat/bin/startup.sh && tail -F /usr/local/tomcat/logs/catalina.out


```

10. 将apache-tomcat-8.0.20.tar.gz,cat.war,client.xml,datasources.xml,Dockerfile,jdk-8u301-linux-x64.tar.gz放在同一个文件夹下

11. 制作包含cat.war的tomcat镜像   
    进入Dockerfile文件所在文件夹,运行命令

```shell
# 运行docker build 命令制作镜像
docker build -t tomcat:cat .
```

12. docker部署运行mysql

```shell
# 从dockerhub拉取镜像
docker pull mysql:5.7.36
# 运行mysql镜像
docker run -d --name mysql-5 -e MYSQL_ROOT_PASSWORD=123456 mysql:5.7.36
```

13. 运行Tomcat

```shell
docker run -d --name tomcat-cat tomcat:cat
```

14. 建立网络,并将mysql和tomcat添加进网络

```shell
# 创建网络
docker network create cat-net

# 将容器添加进网络中
docker network connect cat-net mysql-5
docker network connect cat-net tomcat-cat

```