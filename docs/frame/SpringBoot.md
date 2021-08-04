# SpringBoot

> 后台框架

# 配置启动参数

* 在application.yml 中配置
* Jar包运行,在命令后面添加
  > java -jar springboot.jar --spring.profiles.active=dev
* 添加JVM参数
  > java -jar springboot.jar -Dspring.profiles.active=dev
* Tomcat war包启动,修改/bin/catalina.bat(Linux下为catalina.sh)
  > @echo off
  >
  > setlocal
  >
  > set "JAVA_OPTS=%JAVA_OPTS% -Dspring.profiles.active=dev"
  >
  > if not ""%1"" == ""run"" goto mainEntry

# 整合Redis

## 监听缓存失效

redis配置文件将notify-keyspace-events "" 改为 notify-keyspace-events Ex
```
notify-keyspace-events Ex
#
#  By default all notifications are disabled because most users don't need
#  this feature and the feature has some overhead. Note that if you don't
#  specify at least one of K or E, no events will be delivered.
#  notify-keyspace-events ""
```

监听类

```java
public class RedisExpireMessageListener implements MessageListener {
    @Override
    public void onMessage(Message message, byte[] bytes) {
        // 失效缓存的Key
        String expireKey = new String(message.getBody());
        // ...
    }
}
```

配置

```java

@Configuration
public class RedisConfig extends CachingConfigurerSupport {
    //...
    @Bean
    RedisMessageListenerContainer container(RedisConnectionFactory connectionFactory) {
        RedisMessageListenerContainer container = new RedisMessageListenerContainer();
        container.setConnectionFactory(connectionFactory);
        // 监听DB0中失效的缓存
        container.addMessageListener(new RedisExpireMessageListener(), new PatternTopic("__keyevent@0__:expired"));
        return container;
    }
    //...
}
```

## 添加自定义注解实现注解定义失效时间

> 代码:[GitHub](https://github.com/stq957023588/RedisPlus)

Pom:

```xml
<!--    pom-->
<dependencies>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-context</artifactId>
        <version>5.3.8</version>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
        <version>2.3.4.RELEASE</version>
    </dependency>
</dependencies>
```

失效时间注解:

```java

@Target(ElementType.METHOD)
@Documented
@Retention(RetentionPolicy.RUNTIME)
public @interface CacheExpire {
    long expire() default 1000 * 60 * 50 * 24;
}
```

自定义CacheManager

```java
public class TtlRedisCacheManager extends RedisCacheManager implements ApplicationContextAware, InitializingBean {

    private ApplicationContext applicationContext;

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }

    private final Map<String, RedisCacheConfiguration> initialCacheConfiguration = new LinkedHashMap<>();

    /**
     * key serializer pair
     */
    final RedisSerializationContext.SerializationPair<String> keySerializationPair;
    /**
     * value serializer pair
     */
    final RedisSerializationContext.SerializationPair<Object> valueSerializationPair;


    public TtlRedisCacheManager(RedisCacheWriter cacheWriter, RedisCacheConfiguration defaultCacheConfiguration, RedisSerializationContext.SerializationPair<String> keySerializationPair, RedisSerializationContext.SerializationPair<Object> valueSerializationPair) {
        super(cacheWriter, defaultCacheConfiguration);
        this.keySerializationPair = keySerializationPair;
        this.valueSerializationPair = valueSerializationPair;
    }

    @Override
    protected Collection<RedisCache> loadCaches() {
        return initialCacheConfiguration.entrySet().stream()
                .map(entry -> super.createRedisCache(entry.getKey(), entry.getValue()))
                .collect(Collectors.toList());
    }

    @Override
    public void afterPropertiesSet() {
        Stream.of(applicationContext.getBeanNamesForType(Object.class))
                .forEach(beanName -> add(applicationContext.getType(beanName)));
        super.afterPropertiesSet();
    }

    /**
     * 查询所有@CacheExpire的方法 获取过期时间
     *
     * @param clazz 类
     */
    private void add(final Class<?> clazz) {
        ReflectionUtils.doWithMethods(clazz, method -> {
            ReflectionUtils.makeAccessible(method);

            Cacheable cacheable = AnnotationUtils.findAnnotation(method, Cacheable.class);
            if (cacheable == null) {
                return;
            }
            CacheExpire cacheExpire = AnnotationUtils.findAnnotation(method, CacheExpire.class);
            add(cacheable.cacheNames(), cacheExpire);

        }, method -> null != AnnotationUtils.findAnnotation(method, CacheExpire.class));
    }

    private void add(String[] cacheNames, CacheExpire cacheExpire) {
        for (String cacheName : cacheNames) {
            if (cacheName == null || cacheName.isEmpty()) {
                continue;
            }

            if (cacheExpire == null || cacheExpire.expire() <= 0) {
                continue;
            }

            RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
                    .entryTtl(Duration.ofMillis(cacheExpire.expire()))
                    .disableCachingNullValues()
                    // .prefixKeysWith(cacheName)
                    .serializeKeysWith(keySerializationPair)
                    .serializeValuesWith(valueSerializationPair);

            initialCacheConfiguration.put(cacheName, config);
        }
    }
}
```

RedisConfig

```java

@Configuration
public class RedisConfig extends CachingConfigurerSupport {
    // ...
    @Bean
    public CacheManager cacheManager(RedisConnectionFactory factory) {
        RedisCacheWriter writer = RedisCacheWriter.nonLockingRedisCacheWriter(factory);


        StringRedisSerializer keySerializer = new StringRedisSerializer();
        GenericJackson2JsonRedisSerializer valueSerializer = new GenericJackson2JsonRedisSerializer();
        RedisSerializationContext.SerializationPair<String> keySerializationPair = RedisSerializationContext.SerializationPair.fromSerializer(keySerializer);
        RedisSerializationContext.SerializationPair<Object> valueSerializationPair = RedisSerializationContext.SerializationPair.fromSerializer(valueSerializer);
        // 默认参数
        RedisCacheConfiguration redisCacheConfiguration = RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofDays(30))
                .disableCachingNullValues()
                .serializeKeysWith(keySerializationPair)
                .serializeValuesWith(valueSerializationPair);

        return new TtlRedisCacheManager(writer, redisCacheConfiguration, keySerializationPair, valueSerializationPair);
    }
    // ....
}
```

# 为不同包下的Mapper.java设置不同的数据源

Java代码:

```java

@Configuration
@MapperScan(basePackages = "com.demo.mapper.fool", sqlSessionFactoryRef = "foolSqlSessionFactory")
public class FoolDatabaseConfig {
    /**
     * 将数据库信息转成一个DataSource
     * @return dataSource
     */
    @Bean("foolDataSource")
    @ConfigurationProperties(prefix = "spring.datasource.fool")
    public DataSource foolDataSource() {
        return DataSourceBuilder.create().build();
    }

    /**
     * 此处定义了mapper.xml文件所处的位置
     * mapperPath可以定义在application.yml中,可以用于切换不同数据库
     * @param dataSource foolDataSource() 的返回值
     * @return SqlSessionFactory
     */
    @Bean("foolSqlSessionFactory")
    public SqlSessionFactory foolSqlSessionFactory(@Qualifier("foolDataSource") DataSource dataSource) throws Exception {
        String mapperPath = "classpath*:mapper.*.xml";
        SqlSessionFactoryBean bean = new SqlSessionFactoryBean();
        bean.setDataSource(dataSource);
        bean.setMapperLocations(new PathMatchingResourcePatternResolver().getResource(mapperPath));
        org.apache.ibatis.session.Configuration configuration = new org.apache.ibatis.session.Configuration();
        configuration.setMapUnderscoreToCamelCase(true);
        bean.setConfiguration(configuration);
        return bean.getObject();
    }

    @Bean("foolSqlSessionTemplate")
    public SqlSessionTemplate foolSqlSessionTemplate(@Qualifier("foolSqlSessionFactory") SqlSessionFactory sqlSessionFactory) {
        return new SqlSessionTemplate(sqlSessionFactory);
    }

    /**
     * 此处@Bean中的参数建议使用已经定义好的常量
     * @param dataSource foolDataSource() 的返回值
     * @return foolTransactionManager
     */
    @Bean("foolTransactionManager")
    public PlatformTransactionManager foolTransactionManager(@Qualifier("foolDataSource") DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }
}

/**
 * 多数据源事务管理
 */
@Aspect
@Component
public class MultiTransactionManagerAop {

    private final ComboTransaction comboTransaction;

    @Autowired
    public MultiTransactionManagerAop(ComboTransaction comboTransaction) {
        this.comboTransaction = comboTransaction;
    }

    @PointCut("@annotation(com.demo.annotation.MultiTransactional)")
    public void pointCut() {

    }

    public Object inMultiTransactions(ProceedingJoinPoint point, MultiTransactional multiTransactional) {
        return comboTransaction.inCombinedTx(() -> {
            try {
                return point.proceed();
            } catch (Throwable throwable) {
                if (throwable instanceof RuntimeException) {
                    throw (RuntimeException) throwable;
                }
                throw new RuntimeException(throwable);
            }
        }, multiTransactional.value());
    }

}

/**
 * 这里的value 值限定为FoolDatabaseConfig.foolTransactionManager方法上的@Bean注解中的值
 * 多个DatabaseConfig,就存在多个
 */
@Target(ElementType.METHOD)
@Retention(RetentionPloicy.RUNTIME)
@Documented
public @interface MultiTransactional {
    String[] value() default {};
}

@Component
public class ComboTransaction {

    private Map<String, TxBroker> txBrokerMap;

    @Autowired
    public void setTxBrokerMap(List<TxBroker> txBrokers) {
        txBrokerMap = new HashMap<>(txBrokers.size());
        txBrokers.foreach(broker -> {
            Method method = broker.getClass().getMethod("inTransaction", Callable.class);
            Transactional transactional = AnnotationUtils.findAnnotation(method, Transactional.class);
            if (txBrokerName == null) {
                return;
            }
            txBrokerMap.put(transactional.value(), broker);
        })
    }

    public <V> V inCombinedTx(Callable<V> callable, String[] transactions) {
        if (callable == null) {
            return;
        }
        Callable<V> combined = Stream.of(transactions)
                .filter(e -> !StringUtils.isEmpty(e))
                .distinct()
                .reduce(callable, (r, tx) -> txBrokerMap.get(tx), (r1, r2) -> r2);
        try {
            return combined.call();
        } catch (RuntimeException e) {
            throw e;
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }
}

public interface TxBroker {
    <V> V inTransaction(Callable<V> callable);
}


@Component
public class FoolTxBroker implements TxBroker {

    /**
     * 这里的@Transactional中的值是FoolDatabaseConfig.foolTransactionManager方法上的@Bean注解中的值
     * @param callable
     * @param <V>
     * @return
     */
    @java.lang.Override
    @Transactional("foolTransactionManager")
    public <V> V inTransaction(Callable<V> callable) {
        try {
            return callable.call();
        } catch (RuntimeException e) {
            throw e;
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }
}

```

application.yml 文件:

```yaml
spring:
  datasource:
    fool:
      driver-class-name:
      jdbc-url:
      username:
      password:
```

# 注解

## @Autowired

* 参数   
  | 参数名称 | 参数类型 | 说明 |   
  | ---- | ---- | ---- |   
  | required | boolean | 决定当前所需注入的Bean是否必须存在实例化对象 |   