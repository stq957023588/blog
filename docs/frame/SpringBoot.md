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