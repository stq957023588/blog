# Java

一个面向对象的编程语言

# JAVA自带工具

## jps

获取JAVA pid的工具

命令行执行

```shell
jps
```

-l : 输出主类的全名，如果进程执行的是jar包，输出jar路径
-m : 输出虚拟机进程启动时传递给主类main()函数的参数
-v : 输出虚拟机进程启动时JVM参数
-m : 输出虚拟机进程启动时传递给主类main()函数的参数

## jamp

用于打印指定Java进程的共享对象内存映射或堆内存细节（dump文件）

pid 进程ID 可以通过jps命令获取

```shell
jmap -heap:format=b,file=java_dump.dump <pid>
```

## jvisualvm

jdk自带的用于分析dump文件的工具,位于jdk/bin目录下

# JVM

file.encoding 修改字符编码

## JVM参数

# 远程调试

服务器添加JVM参数`-agentlib:jdwp=transport=dt_socket,address=5005,server=y,suspend=n`

```shell
java -jar -agentlib:jdwp=transport=dt_socket,address=5005,server=y,suspend=n TestApplication-0.0.1.jar
```



# CPU占用过高问题排查

1. top 查看cpu占用情况
2. top -Hp {PID} 查看进程中占用CPU过高的线程
3. jstack {线程} > thread_stack.log 查看堆栈信息，找到对应占用CPU的代码



# KeyTool

用于生成证书

```shell
keytool -genkey -alias pacs -keyalg RSA -keypass 123456 -storepass stor -keystore "D:\Wowjoy\123.jks"
```



# 基础类型占用空间

byte,1字节,8位,范围 -2^7^~2^7^-1

short,2字节,16位,范围 -2^16^~2^16^-1

int,4字节,32位,范围 -2^32^~2^32^-1

long,8字节,64位,范围 -2^64^~2^64^-1

char,2字节

float,4字节

double,8字节

boolean,true 和 false

# JVM

## 类加载机制

1. 加载,将class文件以二进制流的形式加载到内存中,并生成一个class对象
2. 连接
   1. 验证
      1. 文件格式的验证:严重文件的魔数,JDK版本等
      2. 元数据验证:判断类的继承问题等
      3. 字节码验证
   2. 准备:给类属性分配内存,以及初始化
   3. 解析:将符号引用转为直接引用
3. 初始化:执行编译时生成的clinit方法

## 获取自定义JVM参数

```java
// 自定义JVM参数 -Dfool.string=fool -Dfool.boolean=true
String foolString = "fool.string";
// 获取字符串参数
System.getProperty(foolString);
String foolBoolean = "fool.boolean";
// 获取布尔值参数,只有在JVM参数是true时才返回true,是其他值的时候都返回false,包括不存在的值的
Boolean.getBoolean(foolBoolean);
```



## 内存模型

# API使用

## CompletableFuture

异步方法调用

### Complete

```java
public CompletableFuture<T> whenComplete(BiConsumer<? super T,? super Throwable> action)
public CompletableFuture<T> whenCompleteAsync(BiConsumer<? super T,? super Throwable> action)
public CompletableFuture<T> whenCompleteAsync(BiConsumer<? super T,? super Throwable> action, Executor executor)
```

使用

```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    try {
        TimeUnit.SECONDS.sleep(2);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }

    return "SUCCESS";
});


future.whenComplete((result, error) -> {
    System.out.println("error: " + error);
    System.out.println("val: " + result);
}).whenComplete((result,error) -> {
    System.out.println("error: " + error);
    System.out.println("val: " + result);
});
```

输出:

```
error: null
val: SUCCESS
error: null
val: SUCCESS
```

complete返回的仍旧是第一个CompletableFuture,无法自定义返回值

### Handle

```java
public <U> CompletableFuture<U> handle(BiFunction<? super T,Throwable,? extends U> fn)
public <U> CompletableFuture<U> handleAsync(BiFunction<? super T,Throwable,? extends U> fn)
public <U> CompletableFuture<U> handleAsync(BiFunction<? super T,Throwable,? extends U> fn, Executor executor)
```

使用

```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    try {
        TimeUnit.SECONDS.sleep(2);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }

    return "SUCCESS";
});

future.handle((result, error) -> {
    Optional.ofNullable(error).ifPresent(Throwable::printStackTrace);
    return result.length();
}).handle((result, error) -> {
    Optional.ofNullable(error).ifPresent(Throwable::printStackTrace);
    return result == 9;
}).thenAccept(result -> {
    System.out.println("final result:" + result);
});
```

Handle方法可自定义返回值,用于下一个CompletableFuture

### Apply

```java
public <U> CompletableFuture<U> thenApply(Function<? super T,? extends U> fn)
public <U> CompletableFuture<U> thenApplyAsync(Function<? super T,? extends U> fn)
public <U> CompletableFuture<U>  thenApplyAsync(Function<? super T,? extends U> fn, Executor executor)
```

使用

```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    try {
        TimeUnit.SECONDS.sleep(2);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }

    return "SUCCESS";
});


future.thenApply(result -> {
    String s = result.toLowerCase(Locale.ROOT);
    return s.length();
}).thenApply(result -> {
    result += 10;
    return result;
}).thenAccept(System.out::println);
```

apply方法类似handle,但不会再每一层对error进行处理

### Exception

异常捕获

```java
public CompletableFuture<T> exceptionally(Function<Throwable, ? extends T> fn)
```

使用

```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    try {
        TimeUnit.SECONDS.sleep(2);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }

    return "SUCCESS";
});

future.thenApply(result -> {
    if ("T".equals(result)){
        return "T-001";
    }
    return null;
}).thenApply(result -> {
    int l = result.length();
    return l + 1;
}).exceptionally(error -> {
    error.printStackTrace();
    return 0;
}).thenAccept(System.out::println);
```

最终结果输出0

# 常用类解析

## List

### ArrayList,LinkedList区别

ArrayList

* 使用的是动态数组结构
* 对于随机访问的set和get相对速度较快
* 对于插入和删除操作,因为需要通过System.arraycopy进行数据移动,会比较耗时

LinkedList

* 使用的是链表结构
* 对于get和set速度较慢,因为需要从头移动指针
* 对于插入和删除操作,只需要移动指针即可,相对较快

[参考](https://hollischuang.gitee.io/tobetopjavaer/#/basics/java-basic/arraylist-vs-linkedlist-vs-vector)

## Map

### HashMap和HashTable区别

HashMap

* 是非同步的(线程不安全)
* 继承了AbstractMap
* key可以出现一次null,value值可以出现多次
* 初始容量的为16,且扩容后的容量大小一定为 2的倍数
* 使用自定义的散列方法(hash)对key进行散列

HashTable

* 是同步的(线程安全)
* 继承了Dictionary
* key和value均不允许出现null值
* hashtable中的数据初始容量为11,且扩容为old*2 + 1
* 使用的key自带的散列方法

## JUC

### ConcurrentHashMap

通过循环的方式来实现，本次循环内修改的数据，将影响下一次循环内的执行内容

**属性**

```java
/**
 * The largest possible table capacity.  This value must be
 * exactly 1<<30 to stay within Java array allocation and indexing
 * bounds for power of two table sizes, and is further required
 * because the top two bits of 32bit hash fields are used for
 * control purposes.
 */
private static final int MAXIMUM_CAPACITY = 1 << 30;

/**
 * The default initial table capacity.  Must be a power of 2
 * (i.e., at least 1) and at most MAXIMUM_CAPACITY.
 */
private static final int DEFAULT_CAPACITY = 16;

/**
 * The largest possible (non-power of two) array size.
 * Needed by toArray and related methods.
 */
static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

/**
 * The default concurrency level for this table. Unused but
 * defined for compatibility with previous versions of this class.
 */
private static final int DEFAULT_CONCURRENCY_LEVEL = 16;

/**
 * The load factor for this table. Overrides of this value in
 * constructors affect only the initial table capacity.  The
 * actual floating point value isn't normally used -- it is
 * simpler to use expressions such as {@code n - (n >>> 2)} for
 * the associated resizing threshold.
 */
private static final float LOAD_FACTOR = 0.75f;

/**
 * The bin count threshold for using a tree rather than list for a
 * bin.  Bins are converted to trees when adding an element to a
 * bin with at least this many nodes. The value must be greater
 * than 2, and should be at least 8 to mesh with assumptions in
 * tree removal about conversion back to plain bins upon
 * shrinkage.
 */
static final int TREEIFY_THRESHOLD = 8;

/**
 * The bin count threshold for untreeifying a (split) bin during a
 * resize operation. Should be less than TREEIFY_THRESHOLD, and at
 * most 6 to mesh with shrinkage detection under removal.
 */
static final int UNTREEIFY_THRESHOLD = 6;

/**
 * The smallest table capacity for which bins may be treeified.
 * (Otherwise the table is resized if too many nodes in a bin.)
 * The value should be at least 4 * TREEIFY_THRESHOLD to avoid
 * conflicts between resizing and treeification thresholds.
 * 
 * 所有Node链表转化为红黑树的最小容量
 */
static final int MIN_TREEIFY_CAPACITY = 64;

/**
 * Minimum number of rebinnings per transfer step. Ranges are
 * subdivided to allow multiple resizer threads.  This value
 * serves as a lower bound to avoid resizers encountering
 * excessive memory contention.  The value should be at least
 * DEFAULT_CAPACITY.
 */
private static final int MIN_TRANSFER_STRIDE = 16;

/**
 * The number of bits used for generation stamp in sizeCtl.
 * Must be at least 6 for 32bit arrays.
 */
private static final int RESIZE_STAMP_BITS = 16;

/**
 * The maximum number of threads that can help resize.
 * Must fit in 32 - RESIZE_STAMP_BITS bits.
 */
private static final int MAX_RESIZERS = (1 << (32 - RESIZE_STAMP_BITS)) - 1;

/**
 * The bit shift for recording size stamp in sizeCtl.
 */
private static final int RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS;

/*
 * Encodings for Node hash fields. See above for explanation.
 */
static final int MOVED     = -1; // hash for forwarding nodes
static final int TREEBIN   = -2; // hash for roots of trees
static final int RESERVED  = -3; // hash for transient reservations
static final int HASH_BITS = 0x7fffffff; // usable bits of normal node hash
```

**初始化表格/initTable**

```java
private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
        while ((tab = table) == null || tab.length == 0) {
            // 判断是否初始化成功①
            if ((sc = sizeCtl) < 0)
                Thread.yield(); // lost initialization race; just spin
            else if (U.compareAndSetInt(this, SIZECTL, sc, -1)) { // CAS获取初始化资格
                try {
                    if ((tab = table) == null || tab.length == 0) {
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = tab = nt;
                        sc = n - (n >>> 2);
                    }
                } finally {
                    sizeCtl = sc;
                }
                break;
            }
        }
        return tab;
    }
```

1. 通过自旋CAS竞争初始化资格
2. 获取初始化资格后，判断是否已经初始化完毕
3. 如果未初始化，初始化一个Node数组

**treeifyBin**

将所有节点转换成红黑树

```java
private final void treeifyBin(Node<K,V>[] tab, int index) {
    Node<K,V> b; int n;
    if (tab != null) {
        if ((n = tab.length) < MIN_TREEIFY_CAPACITY)
            tryPresize(n << 1);
        else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
            synchronized (b) {
                if (tabAt(tab, index) == b) {
                    TreeNode<K,V> hd = null, tl = null;
                    for (Node<K,V> e = b; e != null; e = e.next) {
                        TreeNode<K,V> p = new TreeNode<K,V>(e.hash, e.key, e.val, null, null);
                        if ((p.prev = tl) == null)
                            hd = p;
                        else
                            tl.next = p;
                        tl = p;
                    }
                    setTabAt(tab, index, new TreeBin<K,V>(hd));
                }
            }
        }
    }
}
```



**put方法**

```java
public V put(K key, V value) {
    return putVal(key, value, false);
}

/** Implementation for put and putIfAbsent */
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh; K fk; V fv;
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value)))
                break;                   // no lock when adding to empty bin
        }
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else if (onlyIfAbsent // check first node without acquiring lock
                 && fh == hash
                 && ((fk = f.key) == key || (fk != null && key.equals(fk)))
                 && (fv = f.val) != null)
            return fv;
        else {
            V oldVal = null;
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) {
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key, value);
                                break;
                            }
                        }
                    }
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                              value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                    else if (f instanceof ReservationNode)
                        throw new IllegalStateException("Recursive update");
                }
            }
            if (binCount != 0) {
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    addCount(1L, binCount);
    return null;
}
```

1. 判空处理
2. 判断Node数组长度，如果数组为空或长度为0，进行初始化
3. 获取key对应数组下标的元素，并判空，如果为空，CAS设置值，如果设置成功则结束
4. 如果设置失败，进行下一轮循环
5. 判断是否需要进行扩容，如果需要扩容，则扩容后进行下一轮循环，并更新tab为扩容后的Node数组，然后进入下一轮循环
6. 对对应Node元素进行加锁，然后判断当前元素所组成的是红黑树还是链表
   1. 如果是链表，则遍历链表，找到与key值完全相同的节点替换value，或者将新节点放在链表最后
   2. 如果是红黑树，则进行红黑树的数据插入
7. 通过binCount记录数据变动次数，如果数据变动次数大于红黑树化的阈值（默认是8），那么需要将链表转化为红黑树（会判断数组长度是否超过MIN_TREEIFY_CAPACITY（64）

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

# 多线程与并发

## ThreadLocal

ThreadLoca用于实现线程本地变量的存储，变量实际存储在Thread对象中，set时获取当前线程存以及线程储本地变量的ThreadLocalMap对象，以ThreadLocal对象为key，进行键值对存储，获取值（get方法）时也是通过获取当前线程以及当前线程的ThreadLocalMap对象，根据ThradLocal对象获取值

## AQS

[一行一行源码分析清楚 AbstractQueuedSynchronizer](/programming-language/一行一行源码分析清楚AbstractQueuedSynchronizer.md)

[一行一行源码分析清楚 AbstractQueuedSynchronizer2](/programming-language/一行一行源码分析清楚+AbstractQueuedSynchronizer+(二).md)

[一行一行源码分析清楚 AbstractQueuedSynchronizer3](/programming-language/一行一行源码分析清楚+AbstractQueuedSynchronizer+(三).md)

原作者:[Javadoop](https://www.javadoop.com/)

## 线程池

### 创建线程池

```java
// 当线程池中的线程满了(达到了5个),则进入队列(LikedBlockingDeque),如果队列也满了,那么就创建线程,当线程达到最大值(10个),则进入拒绝策略
// 如果当前线程池中的线程数大于核心线程数(5),那么线程如果空闲时间超过keepAliveTime(1,单位为TimeUnit.MINUTES),该线程为被终止
ThreadPoolExecutor threadPoolExecutor=new ThreadPoolExecutor(5,10,1,TimeUnit.MINUTES,new LinkedBlockingDeque<>());
```

### 线程池参数详解

线程池创建参数：核心线程数，最大线程数，等待队列，闲置时长单位，闲置时长，拒绝 策略  如果线程池中线程数小于核心线程数，那么每添加一个任务，就添加一个线程，否则将任 务添加到等待队列中，如果等待队列满了，那么就创建新的线程，直到线程数等于最大线 程数，接下来如果继续接收到任务，那么会触发拒绝策略  当线程池中线程数大于核心线程数时，如果线程闲置时间大于闲置时长，那么就会自动销 毁，知道线程数等于核心线程数 

### 线程池核心线程数设置

线程池线程数通常为 cpu 核心数 N + 1 或者 2N

### 拒绝策略

1. AbortPolicy 直接拒绝策略
2. CallerRunPolicy 调用者线程执行当前任务策略
3. DiscardOledestPolicy 丢弃队列中最老的任务,再次提交策略

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

### 扩容

1. 计算新的阈值和容量
2. 新建一个长度为新容量的Node数组
3. 遍历数组以及链表（或红黑树），重新计算每个数据的HASH值

```java
final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
        else {               // zero initial threshold signifies using defaults
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order
                        // 低位链表，表示数据进行HASH后仍存放在原索引位置
                        Node<K,V> loHead = null, loTail = null;
                        // 高位链表，表示数据进行HASH后不在原索引的位置了，即当前索引+旧容量①
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        // 将链表按照顺序拆分为索引不变和索引改变两条链表
                        do {
                            next = e.next;
                            // 索引不变
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            // suo'yin
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            // 新索引位置一定为空，因为索引位置大小超过了原先容量
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }
```

1. 为什么扩容后新索引位置不是原索引位置就是当前索引位置+旧容量位置

   假设 `HashMap` **扩容前**：

   - `oldCap = 8`（即 8 个桶）。
   - `hash 值 % 8` 决定了元素存在哪个桶。

   扩容后：

   - `newCap = 16`（桶数翻倍）。
   - 计算索引 `index = hash % 16`，但实际上只需要看 `hash & oldCap` 这 1 位（因为 `oldCap = 8`，对应二进制 `00001000`）。

   | hash 值（假设） | 旧索引 (`hash % 8`) | `(hash & 8) == 0`? | 迁移后索引         |
   | --------------- | ------------------- | ------------------ | ------------------ |
   | `00000111 (7)`  | `7`                 | ✅ 是               | **7** (原索引不变) |
   | `00001111 (15)` | `7`                 | ❌ 否               | **7 + 8 = 15**     |
   | `00000010 (2)`  | `2`                 | ✅ 是               | **2** (原索引不变) |
   | `00001010 (10)` | `2`                 | ❌ 否               | **2 + 8 = 10**     |



### put时候发生的事情

首先会判断map是否为空或者长度是否为0,如果是重新计算长度

然后判断是否hash冲突了,如果不冲突就将数据根据计算出来的hash放在数组中

如果冲突了:

1. 如果key是同一个,更新value
2. 如果已经转为红黑树,调用putTreeVal
3. 其他情况,将值挂载在对应下标的数组元素链表中

最后map大小自增1,然后判断是否超过了阈值(负载因子*容量),如果超过了,重新计算大小

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