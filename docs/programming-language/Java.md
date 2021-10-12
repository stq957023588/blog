# Java

一个面向对象的编程语言

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
ThreadPoolExecutor threadPoolExecutor=new ThreadPoolExecutor(5,5,1, TimeUnit.MINUTES,new LinkedBlockingDeque<>(),new NamedThreadFactory("测试"));
```

# 关键字

## native

当被native修饰时,表明这是一个本地方法,本地方法是用非Java语言编写的,在Java程序外实现,由JVM去调用.[详细说明](https://blog.csdn.net/wike163/article/details/6635321)