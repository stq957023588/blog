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

# Spring Boot 常见配置

```yaml
spring:
  servlet:
    multipart:
      # 最大上传文件大小,默认是1MB
      max-file-size: 1MB
      # 最大请求大小
      max-request-size: 10MB
```

# 统一返回结果

配置类

```java

@ControllerAdvice
public class ResponseConfig implements ResponseBodyAdvice<Object> {
    @Override
    public boolean supports(MethodParameter returnType, Class<? extends HttpMessageConverter<?>> converterType) {
        HttpServletRequest request = SpringContextUtils.getRequest();
        // 从Http请求中获取请求头
        String headerValue = request.getHeader(HttpHeaderConst.UNIFORM_RESPONSE);
        return !RequestHeaderEnum.NON_UNIFORM_RESPONSE.getValue().equals(headerValue);
    }

    @SneakyThrows
    @Override
    public Object beforeBodyWrite(Object body, MethodParameter returnType, MediaType selectedContentType, Class<? extends HttpMessageConverter<?>> selectedConverterType, ServerHttpRequest request, ServerHttpResponse response) {
        // 从http请求中获取在拦截器中预处理时添加的UniformResponse注解(这里是在ResponseInterceptor中添加的)
        // UniformResponse uniformResponseAnn = (UniformResponse) SpringContextUtils.getRequest().getAttribute(ResponseInterceptor.RESPONSE_RESULT);

        // Class<? extends Result> resultClass = uniformResponseAnn.templateClass();
        if (body instanceof Result) {
            return body;
        }

        Result<Object> result = new DefaultResult<>();
        result.setCode(Result.SUCCESS);
        result.setData(body);

        if (body instanceof String) {
            return JSONObject.toJSONString(result);
        }


        return result;
    }
}

```

# 自定义application.yml中的参数

## maven设置

依赖

```xml

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-configuration-processor</artifactId>
    <optional>true</optional>
</dependency>
```

plugin

```xml

<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <excludes>
            <exclude>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-configuration-processor</artifactId>
            </exclude>
        </excludes>
    </configuration>
</plugin>
```

# SpringSecurity

## Maven依赖

```xml

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

## 自定义配置项

YML中的配置

```yaml
jwt:
  issuer: Fool
  tokenHeader: Authorization
  secret: my-springsecurity-plus
  expiration: 604800
```

Java代码

```java

@Data
@Configuration
@ConfigurationProperties(prefix = "jwt")
public class JwtProperty {
    /**
     * 发布者,用于Token生成以及校验
     */
    private String issuer;
    /**
     * request请求中token的请求头名称
     */
    private String tokenHeader;
    /**
     * 生成token所需要的密钥
     */
    private String secret;
    /**
     * token失效时间,单位毫秒
     */
    private Long expiration;
}
```

pom.xml,添加一下依赖可以在写好Java类后更好的配置yml文件(可不添加,无影响),需Maven重新编译打包

```xml

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-configuration-processor</artifactId>
    <optional>true</optional>
</dependency>
```

## UserDetailService

自定义类实现UserDetailService用于获取用户

```java
public class UserService implements UserDetailsService {
    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        // 使用username从数据库中查询用户
        // 如果密码在存储时没有加密,那么则在取出时进行加密
        String password = "1234";
        // 查询用户的权限
        String[] permissions = new String[]{"ADMIN"};
        return new User(username, password, Stream.of(permissions).map(e -> (GrantedAuthority) () -> e).collect(Collectors.toList()));
    }
}
```

## FilterInvocationSecurityMetadataSource

自定义类实现FilterInvocationSecurityMetadataSource,用于获取当前路径哪些权限可以访问,动态权限控制

```java

@Component
public class CustomizeFilterInvocationSecurityMetadataSource implements FilterInvocationSecurityMetadataSource {
    @Override
    public Collection<ConfigAttribute> getAttributes(Object o) throws IllegalArgumentException {
        //获取请求地址
        String requestUrl = ((FilterInvocation) o).getRequestUrl();
        log.debug("Request url:{}", requestUrl);
        // 查询请求路径所需要的权限
        // 如果想要不登陆也可访问,那么返回空即可.SecurityConfig.createList();
        String[] attributes = {"ADMIN"};
        return SecurityConfig.createList(attributes);
    }

    @Override
    public Collection<ConfigAttribute> getAllConfigAttributes() {
        return null;
    }

    @Override
    public boolean supports(Class<?> aClass) {
        return true;
    }
}
```

## AccessDecisionManager

自定义类实现AccessDecisionManager,用于判断当前用户是否拥有访问当前路径的权限

```java

@Component
public class CustomizeAccessDecisionManager implements AccessDecisionManager {
    @Override
    public void decide(Authentication authentication, Object o, Collection<ConfigAttribute> collection) throws AccessDeniedException, InsufficientAuthenticationException {
        for (ConfigAttribute ca : collection) {
            //当前请求需要的权限,此处路径权限是调用CustomizeFilterInvocationSecurityMetadataSource的getAttributes获取的
            String needRole = ca.getAttribute();
            //当前用户所具有的权限
            Collection<? extends GrantedAuthority> authorities = authentication.getAuthorities();
            for (GrantedAuthority authority : authorities) {
                if (authority.getAuthority().equals(needRole)) {
                    return;
                }
            }
        }
        // 权限不足,抛出访问被拒异常,会被上一级捕获,最终调用自定义AccessDeniedHandler中的handle方法
        throw new AccessDeniedException("权限不足!");
    }

    @Override
    public boolean supports(ConfigAttribute configAttribute) {
        return true;
    }

    @Override
    public boolean supports(Class<?> aClass) {
        return true;
    }

}
```

## OncePerRequestFilter

自定义类继承OncePerRequestFilter,实现Token处理   
用于登陆后返回Token而不是产生一个Session的情况,需要将此过滤器放到UsernamePasswordAuthenticationFilter之前执行

```java

@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    private JwtProperty jwtProperty;

    @Autowired
    public void setJwtProperty(JwtProperty jwtProperty) {
        this.jwtProperty = jwtProperty;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        // 获取Token
        String token = request.getHeader(jwtProperty.getTokenHeader());

        UsernamePasswordAuthenticationToken authentication = null;

        try {
            Algorithm algorithm = Algorithm.HMAC256(jwtProperty.getSecret());
            JWTVerifier verifier = JWT.require(algorithm).withIssuer(jwtProperty.getIssuer()).build();
            // 验证Token是否有效,无效会报错
            DecodedJWT verify = verifier.verify(token);
            // 从token中获取用户名称,和权限
            String username = verify.getClaim("username").asString();
            String[] roles = verify.getClaim("roles").asArray(String.class);

            List<GrantedAuthority> permissions = Stream.of(roles).map(e -> (GrantedAuthority) () -> e).collect(Collectors.toList());
            User user = new User(username, "PROTECTED", permissions);
            // 生成一个authentication,
            authentication = new UsernamePasswordAuthenticationToken(user, null, permissions);
            authentication.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
            logger.info(String.format("Authenticated user %s, setting security context", username));
        } catch (JWTVerificationException e) {
            logger.warn(String.format("Verify token failed:%s", token), e);
        } catch (Exception e) {
            logger.error(String.format("Parsing token failed:%s", token), e);
        }
        /*
            将authentication放到上下文中,
            因为在WebSecurityConfig中取消了session,所以无论是否调用过登录接口,上下文中都不会存在authentication
            需要在判断是否登录前从Token中获取用户信息,生成authentication,手动放入上下文
         */
        SecurityContextHolder.getContext().setAuthentication(authentication);
        // 执行下一个过滤器
        filterChain.doFilter(request, response);
    }
}
```

## 配置SpringSecurity

```java

@Configuration
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    private UserService userService;

    private JwtProperty jwtProperty;

    private JwtAuthenticationFilter jwtAuthenticationFilter;

    private CustomizeAccessDecisionManager accessDecisionManager;

    private CustomizeFilterInvocationSecurityMetadataSource securityMetadataSource;

    @Autowired
    public void setJwtAuthenticationFilter(JwtAuthenticationFilter jwtAuthenticationFilter) {
        this.jwtAuthenticationFilter = jwtAuthenticationFilter;
    }

    @Autowired
    public void setJwtProperty(JwtProperty jwtProperty) {
        this.jwtProperty = jwtProperty;
    }

    @Autowired
    public void setAccessDecisionManager(CustomizeAccessDecisionManager accessDecisionManager) {
        this.accessDecisionManager = accessDecisionManager;
    }

    @Autowired
    public void setSecurityMetadataSource(CustomizeFilterInvocationSecurityMetadataSource securityMetadataSource) {
        this.securityMetadataSource = securityMetadataSource;
    }


    @Autowired
    public void setUserService(UserService userService) {
        this.userService = userService;
    }

    /**
     * 将security中加密方式注入到spring容器中
     */
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    /**
     * 将账号密码设置在数据库当中
     */
    @Override
    public void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth
                //将UserDetailsService放到容器中
                .userDetailsService(userService)
                //加密方式放入
                .passwordEncoder(passwordEncoder());
    }


    @Override
    protected void configure(HttpSecurity http) throws Exception {
        // super.configure(http);

        http.cors().and().csrf().disable();
        http.authorizeRequests()
                .withObjectPostProcessor(new ObjectPostProcessor<FilterSecurityInterceptor>() {
                    @Override
                    public <O extends FilterSecurityInterceptor> O postProcess(O o) {
                        //决策管理器
                        o.setAccessDecisionManager(accessDecisionManager);
                        //安全元数据源
                        o.setSecurityMetadataSource(securityMetadataSource);
                        return o;
                    }
                })
                .and()
                //登出
                .logout()
                //允许所有用户
                .permitAll()
                //登出成功处理逻辑
                .logoutSuccessHandler(logoutSuccessHandler())
                //登出之后删除cookie
                .deleteCookies("JSESSIONID")
                //登入
                .and().formLogin()
                //允许所有用户
                .permitAll()
                //登录成功处理逻辑
                .successHandler(authenticationSuccessHandler())
                //登录失败处理逻辑
                .failureHandler(authenticationFailureHandler())
                //异常处理(权限拒绝、登录失效等)
                .and().exceptionHandling()
                //权限拒绝处理逻辑
                .accessDeniedHandler(accessDeniedHandler())
                //匿名用户访问无权限资源时的异常处理
                .authenticationEntryPoint(authenticationEntryPoint())
                //会话管理
                .and()
                // 取消Session,使用JWT
                .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS);
        // .sessionManagement()
        //同一账号同时登录最大用户数
        // .maximumSessions(1)
        //会话失效(账号被挤下线)处理逻辑
        // .expiredSessionStrategy(sessionInformationExpiredStrategy());
        // 将自定义的Token处理过滤器添加到UsernamePasswordAuthenticationFilter之前
        http.addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class);
    }


    /**
     * Session失效调用
     */
    @Bean
    public SessionInformationExpiredStrategy sessionInformationExpiredStrategy() {
        return sessionInformationExpiredEvent -> {
            DefaultResult<Object> result = DefaultResult.beOffline();
            HttpServletResponse httpServletResponse = sessionInformationExpiredEvent.getResponse();
            httpServletResponse.setContentType("text/json;charset=utf-8");
            httpServletResponse.getWriter().write(JSON.toJSONString(result));
        };
    }

    /**
     * 访问被拒
     */
    @Bean
    public AccessDeniedHandler accessDeniedHandler() {
        return (httpServletRequest, httpServletResponse, e) -> {
            DefaultResult<?> result = DefaultResult.accessDenied();
            httpServletResponse.setContentType("text/json;charset=utf-8");
            httpServletResponse.getWriter().write(JSON.toJSONString(result));

        };
    }

    /**
     * 未登录
     */
    @Bean
    public AuthenticationEntryPoint authenticationEntryPoint() {
        return (httpServletRequest, httpServletResponse, e) -> {
            DefaultResult<?> result = DefaultResult.notLoggedIn();
            httpServletResponse.setContentType("text/json;charset=utf-8");
            httpServletResponse.getWriter().write(JSON.toJSONString(result));
        };
    }


    /**
     * 登录成功
     */
    @Bean
    public AuthenticationSuccessHandler authenticationSuccessHandler() {
        return (httpServletRequest, httpServletResponse, authentication) -> {
            UserDetails userDetails = (UserDetails) authentication.getPrincipal();

            long current = System.currentTimeMillis();

            String token = JWT.create().withIssuer(jwtProperty.getIssuer())
                    .withIssuedAt(new Date(current))
                    .withExpiresAt(new Date(current + jwtProperty.getExpiration()))
                    .withClaim("username", userDetails.getUsername())
                    .withArrayClaim("roles", userDetails.getAuthorities().stream().map(GrantedAuthority::getAuthority).collect(Collectors.toList()).toArray(new String[]{}))
                    .sign(Algorithm.HMAC256(jwtProperty.getSecret()));
            DefaultResult<Object> result = DefaultResult.success("登陆成功", token);
            httpServletResponse.setContentType("text/json;charset=utf-8");
            httpServletResponse.getWriter().write(JSON.toJSONString(result));
        };
    }

    /**
     * 登录失败
     */
    @Bean
    public AuthenticationFailureHandler authenticationFailureHandler() {
        return (httpServletRequest, httpServletResponse, e) -> {
            DefaultResult<Object> result = DefaultResult.authFailed(e.getMessage());
            httpServletResponse.setContentType("text/json;charset=utf-8");
            httpServletResponse.getWriter().write(JSON.toJSONString(result));
        };
    }

    /**
     * 登出
     */
    @Bean
    public LogoutSuccessHandler logoutSuccessHandler() {
        return (httpServletRequest, httpServletResponse, authentication) -> {
            DefaultResult<Object> result = DefaultResult.success("登出成功");
            httpServletResponse.setContentType("text/json;charset=utf-8");
            httpServletResponse.getWriter().write(JSON.toJSONString(result));
        };
    }

}
```

# 国际化I8n

## 获取所有国际化信息

```java


```

# 集成FlyWay

## maven pom.xml

```xml
<!--....-->
<!--依赖-->
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
</dependency>
        <!--....-->
        <!--Maven插件(可不添加)-->
<plugin>
<groupId>org.flywaydb</groupId>
<artifactId>flyway-maven-plugin</artifactId>
<configuration>
    <user>root</user>
    <password>123456</password>
    <driver>com.mysql.cj.jdbc.Driver</driver>
    <url>jdbc:mysql://127.0.0.1:3306/fool</url>
    <baselineOnMigrate>true</baselineOnMigrate>
    <!-- //sql脚本位置，flyway会自动去找到这个目录并且执行里面的sql脚本 -->
    <locations>classpath:db/migration/</locations>
</configuration>
</plugin>
```

## sql文件命名格式

V[version]__[name].sql

# 集成Swagger

## 配置

```java

@Configuration
@ConfigurationProperties(prefix = "swagger")
public class SwaggerConfig {

    private boolean enable;

    // swagger2的配置文件，这里可以配置swagger2的一些基本的内容，比如扫描的包等等
    @Bean
    public Docket createRestApi() {
        return new Docket(DocumentationType.SWAGGER_2)
                // 设置不同的组
                .groupName("Group one")
                .apiInfo(apiInfo())
                // 是否启动Swagger,可在application.yml中配置,prod环境建议为false
                .enable(enable)
                .select()
                // 需要扫描的包的路径
                .apis(RequestHandlerSelectors.basePackage("com.fool.demo.controller"))
                .paths(PathSelectors.any())
                .build();
    }

    // 构建 api文档的详细信息函数,注意这里的注解引用的是哪个
    private ApiInfo apiInfo() {
        return new ApiInfoBuilder()
                // 页面标题
                .title("Fool's demo")
                // 创建人信息
                .contact(new Contact("Fool", "主页地址", "邮箱"))
                // 版本号
                .version("1.0")
                // 描述
                .description("This is a demo application")
                .build();
    }
}
```

## 集成Swagger中的坑

* 在2.9.2 版本中使用@ApiParam(defaultValue = "1") 会报错,这是版本Bug

# 集成Solr

依赖

```xml

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-solr</artifactId>
</dependency>
```

# 集成WebSocket

## 坑

* Junit单元测试时,需在@SpringbootTest注解中添加webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT参数
  > @SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)

# 整合Redis

## RedisTemplate

### 创建文件夹

文件下以数组形式存储,双冒号分割

```java
// 多级文件夹用多个::分割
redisTemplate.opsForValue().set("dir::key","value");
```

文件夹下以key/value形式存储,单冒号分割

```java
// 多级文件夹用多个:分割
redisTemplate.opsForValue().set("dir:key","value");
```

## Redis广播

> 可用于微服务下的websocket

配置

```java

@Configuration
public class RedisConfig {
    // ...

    /**
     * redis 消息监听器
     */
    @Bean
    RedisMessageListenerContainer container(RedisConnectionFactory connectionFactory) {
        RedisMessageListenerContainer container = new RedisMessageListenerContainer();
        container.setConnectionFactory(connectionFactory);
        // 定义广播频道
        PatternTopic channel = new PatternTopic("channel");
        // 添加广播监听器
        container.addMessageListener(new RedisRadioMessageListener(), channel);
        return container;
    }
}
```

redis广播监听接收类:

```java

public class RedisRadioMessageListener implements MessageListener {
    @Override
    public void onMessage(Message message, byte[] bytes) {

    }
}
```

发送广播

```java
  public class RedisRadio {
    private StringRedisTemplate template;

    @Autowired
    public void setTemplate(StringRedisTemplate template) {
        this.template = template;
    }

    public void radio(String message) {
        template.convertAndSend("channel", message);
    }
}
```

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

# 测试

在使用Junit测试时,如果项目中包含WebSocket,@SpringBootTest需要添加参数webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT

# 注解

## @Autowired

* 参数   
  | 参数名称 | 参数类型 | 说明 |   
  | ---- | ---- | ---- |   
  | required | boolean | 决定当前所需注入的Bean是否必须存在实例化对象 |

## @ConditionalOnProperty

* 参数 | 参数名称 | 参数类型 | 说明 |   
  | ---- | ---- | ---- |   
  | name | String | application.yml中的参数名称 | | havingValue | String |
  只有application.yml中的值与havingValue中的值匹配时才会被Springboot管理 |