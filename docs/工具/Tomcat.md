# Tomcat

## Docker启动

> 在tomcat9.0以上,webapps为空文件夹,需要删除webapps文件夹,然后重命名webapps.bak文件夹为webapps

## 自定义JAVA_HOME
在setclasspath.bat文件的添加 set JAVA_HOME=jdk地址

## 部署安装

### 创建用户

修改conf/tomcat-users.xml

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

### Linux下访问manager(项目管理)报错问题

在conf/Catalina/localhost文件夹下创建manager.xml

```xml
<!--m-->
<Context privileged="true" antiResourceLocking="false"
         docBase="${catalina.home}/webapps/manager">
    <Valve className="org.apache.catalina.valves.RemoteAddrValve" allow="^.*$" />
</Context>
```

### 控制台乱码问题

修改conf/logging.properties文件，将java.util.logging.ConsoleHandler.encoding 修改成GBK
