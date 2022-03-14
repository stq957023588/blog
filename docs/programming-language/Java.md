# Java

一个面向对象的编程语言

# JVM



# java-agent

## 给运行中的Java程序添加代理

1. 代理方法,代理方法名称为agentmain

```java
public class TestAgent {

    public static void agentmain(String args, Instrumentation instrumentation) {
        System.out.println("load agent after main run.args=" + args);
        Class<?>[] classes = instrumentation.getAllLoadedClasses();

        for (Class<?> cls : classes) {
            System.out.println(cls.getName());
        }
        System.out.println("agent run completely");
    }
}
```

2. 使用maven进行打包,需要添加maven插件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.fool</groupId>
    <artifactId>AgentDemo</artifactId>
    <version>0.0.1</version>

    <properties>
        <maven.compiler.source>11</maven.compiler.source>
        <maven.compiler.target>11</maven.compiler.target>
    </properties>
    <!--...-->
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-jar-plugin</artifactId>
                <version>3.2.0</version>
                <configuration>
                    <archive>
                        <manifest>
                            <addClasspath>true</addClasspath>
                        </manifest>
                        <manifestEntries>
                            <Agent-Class>
                                <!--代理类全名-->
                                com.fool.TestAgent
                            </Agent-Class>
                        </manifestEntries>
                    </archive>
                </configuration>
            </plugin>
        </plugins>
    </build>

</project>
```

3. 任意启动一个java程序叫做程序A,例:一个空的tomcat,或者新建springboot程序执行

4. 使用jps命令获取启动的java程序的pid

```shell
jps
#23136 Jps
#23112 RemoteMavenServer36
#19564
#23308 Launcher
#23772 Application
```

5. 向程序A中注入代理

```java

public class Application {

    public static void main(String[] args) {
        try {
            // 23772是java程序的PID
            VirtualMachine vm = VirtualMachine.attach("23772");
            // loadAgent方法参数第一个是代理jar的地址,第二个参数是JVM参数,由agentmain中的第一个参数接收
            vm.loadAgent("E:\\workspace\\AgentDemo\\target\\AgentDemo-0.0.1.jar");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

}
```

6. 打开程序A的输出控制台,可以发现已经执行了TestAgent.agentmain()方法

# 线程

## 线程池

### 创建线程池

```java
// 当线程池中的线程满了(达到了5个),则进入队列(LikedBlockingDeque),如果队列也满了,那么就创建线程,当线程达到最大值(10个),则进入拒绝策略
// 如果当前线程池中的线程数大于核心线程数(5),那么线程如果空闲时间超过keepAliveTime(1,单位为TimeUnit.MINUTES),该线程为被终止
ThreadPoolExecutor threadPoolExecutor=new ThreadPoolExecutor(5,10,1,TimeUnit.MINUTES,new LinkedBlockingDeque<>());
```

### 线程工厂

* 自定义命名线程工厂

```java
public class NamedThreadFactory implements ThreadFactory {

    private final AtomicInteger poolNumber = new AtomicInteger(1);

    private final ThreadGroup threadGroup;

    private final AtomicInteger threadNumber = new AtomicInteger(1);

    public final String namePrefix;

    NamedThreadFactory(String name) {
        SecurityManager s = System.getSecurityManager();
        threadGroup = (s != null) ? s.getThreadGroup() :
                Thread.currentThread().getThreadGroup();
        if (null == name || "".equals(name.trim())) {
            name = "pool";
        }
        namePrefix = name + "-" +
                poolNumber.getAndIncrement() +
                "-thread-";
    }

    @Override
    public Thread newThread(Runnable r) {
        Thread t = new Thread(threadGroup, r,
                namePrefix + threadNumber.getAndIncrement(),
                0);
        if (t.isDaemon())
            t.setDaemon(false);
        if (t.getPriority() != Thread.NORM_PRIORITY)
            t.setPriority(Thread.NORM_PRIORITY);
        return t;
    }
}
```

* 使用自定义命名线程工厂

```java
ThreadPoolExecutor threadPoolExecutor=new ThreadPoolExecutor(5,5,1,TimeUnit.MINUTES,new LinkedBlockingDeque<>(),new NamedThreadFactory("测试"));
```

# 关键字

## native

当被native修饰时,表明这是一个本地方法,本地方法是用非Java语言编写的,在Java程序外实现,由JVM去调用.[详细说明](https://blog.csdn.net/wike163/article/details/6635321)

# 源码分析

## HashMap

### HashMap容量为什么为2的指数

HashMap中将hash生成的整型转换成链表数组中的下标的方法使用的位运算return h & (length-1) 表示的是 h 除以 length - 1 取余,而要使位运算成立,length必须为2的指数



# 第三方类库

## MapStruct

类转换

## So-Token

轻量级 Java 权限认证框架 [官网](https://sa-token.dev33.cn/doc/)

## Forest

将Http映射成接口 [官网](https://forest.dtflyx.com/docs/)

## Faker

假数据制造,[GitHub](https://github.com/DiUS/java-faker)

## Wiremock

测试框架