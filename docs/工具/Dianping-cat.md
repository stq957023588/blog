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
   client.xml用于配置指向的CAT服务器,CAT会将服务端也当作一个客户端,所以此处部署服务端,需要用到client.xml

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
# 修改时区
RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && echo ‘Asia/Shanghai’ >/etc/timezone
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
# tomcat端口
EXPOSE  8080
# cat传输数据的端口
EXPOSE  2280
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
docker run -d --name mysql-5 -p 3307:3306 -e MYSQL_ROOT_PASSWORD=123456 -e TZ=Asia/Shanghai mysql:5.7.36
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

# 点评/CAT埋点监控

1. 添加依赖 依赖是在cat项目打包时生成,可以在idea中运行install放入本地仓库

```xml

<dependency>
    <groupId>com.dianping.cat</groupId>
    <artifactId>cat-client</artifactId>
    <version>3.0.0</version>
</dependency>

<dependency>
<groupId>com.dianping.cat</groupId>
<artifactId>cat-core</artifactId>
<version>3.0.0</version>
</dependency>
```

2. springboot url埋点

```java

@Configuration
public class BeanConfig {
    @Bean
    public FilterRegistrationBean<CatFilter> catFilter() {
        FilterRegistrationBean<CatFilter> registration = new FilterRegistrationBean<>();
        CatFilter filter = new CatFilter();
        registration.setFilter(filter);
        registration.addUrlPatterns("/*");
        registration.setName("cat-filter");
        registration.setOrder(1);
        return registration;
    }
}
```

3. mybatis sql监控 将cat源码中的integration/mybatis/CatMybatisPlugin.java复制到项目中,并在class上添加@Component注解即可

```java
package com.dianping.cat.plugins;

import com.alibaba.druid.pool.DruidDataSource;
import com.dianping.cat.Cat;
import com.dianping.cat.message.Message;
import com.dianping.cat.message.Transaction;
import org.apache.ibatis.datasource.pooled.PooledDataSource;
import org.apache.ibatis.datasource.unpooled.UnpooledDataSource;
import org.apache.ibatis.executor.Executor;
import org.apache.ibatis.mapping.*;
import org.apache.ibatis.plugin.*;
import org.apache.ibatis.reflection.MetaObject;
import org.apache.ibatis.session.Configuration;
import org.apache.ibatis.session.ResultHandler;
import org.apache.ibatis.session.RowBounds;
import org.apache.ibatis.type.TypeHandlerRegistry;

import javax.sql.DataSource;
import java.lang.reflect.Field;
import java.lang.reflect.InvocationTargetException;
import java.text.DateFormat;
import java.util.Date;
import java.util.List;
import java.util.Locale;
import java.util.Properties;
import java.util.regex.Matcher;
import java.util.regex.Pattern;


/**
 *  1.Cat-Mybatis plugin:  Rewrite on the version of Steven;
 *  2.Support DruidDataSource,PooledDataSource(mybatis Self-contained data source);
 * @author zhanzehui(west_20 @ 163.com)
 */

@Intercepts({
        @Signature(method = "query", type = Executor.class, args = {
                MappedStatement.class, Object.class, RowBounds.class,
                ResultHandler.class}),
        @Signature(method = "update", type = Executor.class, args = {MappedStatement.class, Object.class})
})
public class CatMybatisPlugin implements Interceptor {

    private static final Pattern PARAMETER_PATTERN = Pattern.compile("\\?");
    private static final String MYSQL_DEFAULT_URL = "jdbc:mysql://UUUUUKnown:3306/%s?useUnicode=true";
    private Executor target;

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        MappedStatement mappedStatement = this.getStatement(invocation);
        String methodName = this.getMethodName(mappedStatement);
        Transaction t = Cat.newTransaction("SQL", methodName);

        String sql = this.getSql(invocation, mappedStatement);
        SqlCommandType sqlCommandType = mappedStatement.getSqlCommandType();
        Cat.logEvent("SQL.Method", sqlCommandType.name().toLowerCase(), Message.SUCCESS, sql);

        String url = this.getSQLDatabaseUrlByStatement(mappedStatement);
        Cat.logEvent("SQL.Database", url);

        return doFinish(invocation, t);
    }

    private MappedStatement getStatement(Invocation invocation) {
        return (MappedStatement) invocation.getArgs()[0];
    }

    private String getMethodName(MappedStatement mappedStatement) {
        String[] strArr = mappedStatement.getId().split("\\.");
        String methodName = strArr[strArr.length - 2] + "." + strArr[strArr.length - 1];

        return methodName;
    }

    private String getSql(Invocation invocation, MappedStatement mappedStatement) {
        Object parameter = null;
        if (invocation.getArgs().length > 1) {
            parameter = invocation.getArgs()[1];
        }

        BoundSql boundSql = mappedStatement.getBoundSql(parameter);
        Configuration configuration = mappedStatement.getConfiguration();
        String sql = sqlResolve(configuration, boundSql);

        return sql;
    }

    private Object doFinish(Invocation invocation, Transaction t) throws InvocationTargetException, IllegalAccessException {
        Object returnObj = null;
        try {
            returnObj = invocation.proceed();
            t.setStatus(Transaction.SUCCESS);
        } catch (Exception e) {
            Cat.logError(e);
            throw e;
        } finally {
            t.complete();
        }

        return returnObj;
    }


    private String getSQLDatabaseUrlByStatement(MappedStatement mappedStatement) {
        String url = null;
        DataSource dataSource = null;
        try {
            Configuration configuration = mappedStatement.getConfiguration();
            Environment environment = configuration.getEnvironment();
            dataSource = environment.getDataSource();

            url = switchDataSource(dataSource);

            return url;
        } catch (NoSuchFieldException | IllegalAccessException | NullPointerException e) {
            Cat.logError(e);
        }

        Cat.logError(new Exception("UnSupport type of DataSource : " + dataSource.getClass().toString()));
        return MYSQL_DEFAULT_URL;
    }

    private String switchDataSource(DataSource dataSource) throws NoSuchFieldException, IllegalAccessException {
        String url = null;

        if (dataSource instanceof DruidDataSource) {
            url = ((DruidDataSource) dataSource).getUrl();
        } else if (dataSource instanceof PooledDataSource) {
            Field dataSource1 = dataSource.getClass().getDeclaredField("dataSource");
            dataSource1.setAccessible(true);
            UnpooledDataSource dataSource2 = (UnpooledDataSource) dataSource1.get(dataSource);
            url = dataSource2.getUrl();
        } else if (dataSource instanceof HikariDataSource) {
            // 一般Springboot集成mybatis使用的时HikarDataSource
            HikariDataSource hikariDataSource = (HikariDataSource) dataSource;
            url = hikariDataSource.getJdbcUrl();
        } else {
            //other dataSource expand
        }

        return url;
    }

    public String sqlResolve(Configuration configuration, BoundSql boundSql) {
        Object parameterObject = boundSql.getParameterObject();
        List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
        String sql = boundSql.getSql().replaceAll("[\\s]+", " ");
        if (parameterMappings.size() > 0 && parameterObject != null) {
            TypeHandlerRegistry typeHandlerRegistry = configuration.getTypeHandlerRegistry();
            if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
                sql = sql.replaceFirst("\\?", Matcher.quoteReplacement(resolveParameterValue(parameterObject)));

            } else {
                MetaObject metaObject = configuration.newMetaObject(parameterObject);
                Matcher matcher = PARAMETER_PATTERN.matcher(sql);
                StringBuffer sqlBuffer = new StringBuffer();
                for (ParameterMapping parameterMapping : parameterMappings) {
                    String propertyName = parameterMapping.getProperty();
                    Object obj = null;
                    if (metaObject.hasGetter(propertyName)) {
                        obj = metaObject.getValue(propertyName);
                    } else if (boundSql.hasAdditionalParameter(propertyName)) {
                        obj = boundSql.getAdditionalParameter(propertyName);
                    }
                    if (matcher.find()) {
                        matcher.appendReplacement(sqlBuffer, Matcher.quoteReplacement(resolveParameterValue(obj)));
                    }
                }
                matcher.appendTail(sqlBuffer);
                sql = sqlBuffer.toString();
            }
        }
        return sql;
    }

    private String resolveParameterValue(Object obj) {
        String value = null;
        if (obj instanceof String) {
            value = "'" + obj.toString() + "'";
        } else if (obj instanceof Date) {
            DateFormat formatter = DateFormat.getDateTimeInstance(DateFormat.DEFAULT, DateFormat.DEFAULT, Locale.CHINA);
            value = "'" + formatter.format((Date) obj) + "'";
        } else {
            if (obj != null) {
                value = obj.toString();
            } else {
                value = "";
            }

        }
        return value;
    }

    @Override
    public Object plugin(Object target) {
        if (target instanceof Executor) {
            this.target = (Executor) target;
            return Plugin.wrap(target, this);
        }
        return target;
    }

    @Override
    public void setProperties(Properties properties) {
    }

}
```
