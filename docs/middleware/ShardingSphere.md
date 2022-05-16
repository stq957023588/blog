# ShardingSphere

## Sharding-JDBC



### Springboot中使用

基于ShardingSphere 5.1.1 版本

#### 单机模式使用

引入依赖(只需要引入一个就行)

```xml
<dependency>
    <groupId>org.apache.shardingsphere</groupId>
    <artifactId>shardingsphere-jdbc-core-spring-boot-starter</artifactId>
    <version>${shardingsphere.version}</version>
</dependency>
```

在application.yml中配置	

使用内存模式

```yaml
spring:
  shardingsphere:
    # 内存模式,(其他模式: 单机模式,集群模式)
    mode: Memory
```

配置数据源

```yaml
spring:
  shardingsphere:
    datasource:
      # 定义所有的数据源名称,多个数据源使用逗号分隔
      names: sharding00,sharding01
      # 这里使用names中定义的数据源名称
      sharding00:
        # 数据库连接池,通常可以使用spring-boot-starter-jdbc中带的HikariCP
        type: com.zaxxer.hikari.HikariDataSource
        # 常用数据库连接参数
        driver-class-name: com.mysql.cj.jdbc.Driver
        jdbc-url: jdbc:mysql://127.0.0.1:3306/sharding_00
        username: root
        password: 123456
      sharding01:
        # 数据库连接池,通常可以使用spring-boot-starter-jdbc中带的HikariCP
        type: com.zaxxer.hikari.HikariDataSource
        # 常用数据库连接参数
        driver-class-name: com.mysql.cj.jdbc.Driver
        jdbc-url: jdbc:mysql://127.0.0.1:3306/sharding_01
        username: root
        password: 123456
```

配置分片规则

使用ShardingSphere自带的分片算法模板定义的具体的分片算法

```yaml
spring:
  shardingsphere:
    rules:
      sharding:
        sharding-algorithms:
          # 分片算法名称(自定义)
          mod2:
            # 使用的分片算法
            type: MOD
            # 分片算法参数
            props:
              sharding-count: 2
          # 分片算法名称(自定义)
          mod4:
            # 使用的分片算法
            type: MOD
            # 分片算法参数
            props:
              sharding-count: 4
```

配置分布式key生成器

```yaml
spring:
  shardingsphere:
    rules:
      sharding:
        key-generators:
          #自定义分布式key
          redis:
            type: REDIS
            props:
              redis-key: FOOL_REDIS_KEY   
          cos-snow:
            # 使用的分布式key生成算法
            type: COSID_SNOWFLAKE
            # 分布式key生成算法id
            props:
              as-string: false
          cos-id:
            type: COSID
            props:
              as-string: false
              id-name: __fool__
          snow1:
            type: SNOWFLAKE
            props:
              max-vibration-offset: 3
```



给具体的表定义分片规则,以及唯一键生成器定义

```yaml
spring:
  shardingsphere:
    rules:
      sharding:
        tables:
          # 虚拟表名
          order:
            # 实际表名(数据库中存在的表名)
            actual-data-nodes: sharding00.orders_$->{[0,2]},sharding01.orders_$->{[1,3]}
		   # 数据源分片规则(分库规则)
		   database-strategy:
		     # 标准分片
		     standard: 
		       # 用于分片的字段
		       sharding-column: value
		       # 分片算法名称(在上面已经定义好了)
                sharding-algorithm-name: mod2
            table-strategy:
              # 数据表分片规则(分表规则),和分库规则一样
              standard:
                sharding-column: value
                sharding-algorithm-name: mod4
            # 分布式key生成算法
            key-generate-strategy:
              # 分布式key字段名称
              column: id
              # 分布式key生成器名称
              key-generator-name: snow1
              
```

配置绑定表,存在多表关联查询时添加绑定表设置,以防止笛卡尔积

```yaml
spring:
  shardingsphere:
    rules:
      sharding:
        binding-tables:
          - order,order_detail
```

读写分离

> 读写分离使用的数据源是在spring.shardingsphere.rules.readwrite-splitting.data-sources定义的数据源名称

```yaml
spring:
  shardingsphere:
    datasource:
      names: sharding00,sharding01,read00,read01
      # 定义数据源
      read00:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        jdbc-url: jdbc:mysql://127.0.0.1:3305/sharding_00?allowPublicKeyRetrieval=true
        username: root
        password: 123456
      read01:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        jdbc-url: jdbc:mysql://127.0.0.1:3305/sharding_01?allowPublicKeyRetrieval=true
        username: root
        password: 123456
      sharding00:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        jdbc-url: jdbc:mysql://127.0.0.1:3306/sharding_00?allowPublicKeyRetrieval=true
        username: root
        password: 123456
      sharding01:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        jdbc-url: jdbc:mysql://127.0.0.1:3306/sharding_01?allowPublicKeyRetrieval=true
        username: root
        password: 123456
    rules:
      readwrite-splitting:
      	# 定义读写分离数据源
        data-sources:
          # 读写分离数据源名称,如需读写分离,在定义实际节点时使用此数据源名称
          rws01:
            load-balancer-name: normal
            type: Static
            props:
           	  # 写数据源,使用spring.shardingsphere.datasource下定义的数据源名称
              write-data-source-name: sharding01
              # 读数据源,使用spring.shardingsphere.datasource下定义的数据源名称
              read-data-source-names: read01
          rws00:
            load-balancer-name: normal
            type: Static
            props:
              write-data-source-name: sharding00
              read-data-source-names: read00
        load-balancers:
          normal:
            type: RANDOM
            props:
              aaa: 0
      sharding:
        binding-tables:
          - order,order_detail
        tables:
          order_detail:
            # 读写分离,数据源名称 使用 spring.shardingsphere.rules.readwrite-spliting.data-sources下定义的数据源
            actual-data-nodes: rws00.order_detail_0$->{[0,2]},rws01.order_detail_0$->{[1,3]}
            database_strategy:
              standard:
                sharding-column: order_id
                sharding-algorithm-name: mod2
            table-strategy:
              standard:
                sharding-column: order_id
                sharding-algorithm-name: mod4
          order:
            actual-data-nodes: rws00.order_0$->{[0,2]},rws01.order_0$->{[1,3]}
            database-strategy:
              standard:
                sharding-column: id
                sharding-algorithm-name: mod2
            table-strategy:
              standard:
                sharding-column: id
                sharding-algorithm-name: mod4

```

读写分离测试结果

查询使用是read00数据源

```
2022-05-10 16:09:34.469  INFO 19848 --- [           main] ShardingSphere-SQL                       : Logic SQL: select * from `order` where id = ?
2022-05-10 16:09:34.469  INFO 19848 --- [           main] ShardingSphere-SQL                       : SQLStatement: MySQLSelectStatement(table=Optional.empty, limit=Optional.empty, lock=Optional.empty, window=Optional.empty)
2022-05-10 16:09:34.469  INFO 19848 --- [           main] ShardingSphere-SQL                       : Actual SQL: read00 ::: select * from `order_00` where id = ? ::: [204]
```

插入使用的sharding01数据源

```
2022-05-10 16:27:07.138  INFO 16084 --- [           main] ShardingSphere-SQL                       : Logic SQL: insert into `order`(id,`number`, total_price) values (?,?, ?)
2022-05-10 16:27:07.138  INFO 16084 --- [           main] ShardingSphere-SQL                       : SQLStatement: MySQLInsertStatement(setAssignment=Optional.empty, onDuplicateKeyColumns=Optional.empty)
2022-05-10 16:27:07.138  INFO 16084 --- [           main] ShardingSphere-SQL                       : Actual SQL: sharding01 ::: insert into `order_01`(id,`number`, total_price) values (?, ?, ?) ::: [209, 817f391bec83467b8f53f2c06e6a2de8, 41.13544372549201]
```

数据加密,

```yaml
spring:
  shardingsphere:
    rules:
      # 数据加密
      encrypt:
      	# 定义需要数据加密的表
        tables:
          # 数据加密表名,此处的表需要在spring.shardingsphere.rules.sharding.tables下d
          user:
            # 是否通过加密列查询, 选择false需要定义明文列(plain-column),大多用于已存在明文的数据表,但需要进行迁移的情况
            query-with-cipher-column: true
            # 定义加密列
            columns:
              # 数据加密逻辑列名,非实际数据表中的列
              password:
                # 加密列,实际数据表中的列
                cipher-column: password
                # 明文列,实际数据表中的列,非必填,可不使用 用于存储实际数据,加密后即可获得对应加密列的值,
                plain-column: password_plain
                # 加密算法名
                encryptor-name: aes_encryptor
        # 定义加密算法
        encryptors:
          # 加密算法名称(自定义)
          aes_encryptor:
            # 加密算法类型
            type: AES
            # 加密算法参数
            props:
              aes-key-value: 123456abc

```





#### 自定义分布式key

新建一个类实现KeyGenerateAlgorithm接口

```java
public class RedisKeyGenerateAlgorithm implements KeyGenerateAlgorithm {
    // 定义分布式key生成算法时 type 的值,
    public static final String TYPE = "REDIS";
    // 定义分布式key生成算法时 props 的值
    private Properties props = new Properties();

    private RedisTemplate redisTemplate;
    private String redisKey;

    /**
     * 生成分布式key
     * @return 分布式key
     */
    @Override
    public Comparable<?> generateKey() {
        return this.redisTemplate.opsForValue().increment(this.redisKey);
    }
  
    /**
     * 初始化
     */
    @Override
    public void init() {
        this.redisTemplate = SpringContextUtils.getBean(RedisTemplate.class,"stringLongRedisTemplate");
        this.redisKey = getProps().getProperty("redis-key", "redis-distributed-key");
    }
    /**
     * 获取type
     */
    @Override
    public String getType() {
        return TYPE;
    }


    @Override
    public Properties getProps() {
        return this.props;
    }

    @Override
    public void setProps(Properties props) {
        this.props = props;
    }

}
```

创建好类之后,需要在resources下创建META-INF/services文件夹,

并在这个文件夹下创建org.apache.shardingsphere.sharding.spi.KeyGenerateAlgorithm文件(这个是固定名称)

org.apache.shardingsphere.sharding.spi.KeyGenerateAlgorithm这个文件以text打开,输入自定义实现类的全名称

> PS. shardingsphere加载分片算法以及分布式key生成算法的实现类是使用了ServiceLoader,所以按照ServiceLoader规范添加



#### 自定义分片算法

新建一个类实现StandardShardingAlgorithm

后续操作同[自定义分布式key](#自定义分布式key)
