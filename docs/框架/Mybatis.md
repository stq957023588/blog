# Mybatis

## 流式查询



## 缓存机制

[缓存机制-美团](https://tech.meituan.com/2018/01/19/mybatis-cache.html)

### 一级缓存

mybatis的一级缓存通过localCacheScope进行开启(默认开启)

```xml
<setting name="localCacheScope" value="SESSION"/>		
```

在同一个SqlSession中,如果使用相同条件查询(没有插入修改语句),mybatis只有在第一次回去查询数据库

```java
SqlSession sqlSession = sqlSessionFactory.openSession(true);
StudentMapper mapper = sqlSession.getMapper(StudentMapper.class);
System.out.println("first query");
mapper.selectByPrimaryKey(4L);
System.out.println("second query");
mapper.selectByPrimaryKey(4L);
System.out.println("third query");
mapper.selectByPrimaryKey(4L);
```

如果控制台打印了Sql语句,可以发现只有在first query后面打印了Sql语句

```
first query
2022-03-11 14:24:35.186  INFO 3796 --- [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Starting...
2022-03-11 14:24:36.145  INFO 3796 --- [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Start completed.
JDBC Connection [HikariProxyConnection@1841440668 wrapping com.mysql.cj.jdbc.ConnectionImpl@7d7b4e04] will not be managed by Spring
==>  Preparing: select id,name,age from student where id = ?
==> Parameters: 4(Long)
<==    Columns: id, name, age
<==        Row: 4, Rose, 15
<==      Total: 1
second query
third query

```

不过当遇到DML语句时,一级缓存就会失效

```java
SqlSession sqlSession = sqlSessionFactory.openSession(true);
StudentMapper mapper = sqlSession.getMapper(StudentMapper.class);

Student student = mapper.selectByPrimaryKey(4L);
System.out.println(student);
int i = mapper.insertStudent(buildStudent());
System.out.println("insert count:" + i);
mapper.selectByPrimaryKey(4L);
```

查询主键为4的学生后,向表中插入一条学生记录,再次查询主键为4的学生,这时mybatis会去查询数据库,而非使用缓存

```
==>  Preparing: select id,name,age from student where id = ?
==> Parameters: 4(Long)
<==    Columns: id, name, age
<==        Row: 4, Rose, 15
<==      Total: 1
Student(id=4, name=Rose, age=15)
==>  Preparing: insert into student(name, age) values (?, ?)
==> Parameters: df08c4aa-649c-4337-920f-ec68a89962fd(String), 23(Integer)
<==    Updates: 1
insert count:1
==>  Preparing: select id,name,age from student where id = ?
==> Parameters: 4(Long)
<==    Columns: id, name, age
<==        Row: 4, Rose, 15
<==      Total: 1
```

第二次查询会有SQL语句打印

mybatis的一级缓存的范围仅限于当前SqlSession,而且另一个SqlSession的DDL并不会使当前的SqlSession的缓存失效

```java
SqlSession sqlSession1 = sqlSessionFactory.openSession(true);
SqlSession sqlSession2 = sqlSessionFactory.openSession(true);

StudentMapper mapper1 = sqlSession1.getMapper(StudentMapper.class);
StudentMapper mapper2 = sqlSession2.getMapper(StudentMapper.class);

System.out.println(mapper1.selectByPrimaryKey(5L));
System.out.println(mapper2.selectByPrimaryKey(5L));

int jack = mapper1.updateStudentName("Jack", 5L);
System.out.println("mapper1 update count:" + jack);

System.out.println(mapper1.selectByPrimaryKey(5L));
System.out.println(mapper2.selectByPrimaryKey(5L));
```

2个SqlSession同时查询,其中一个修改了表数据,,第二次查询其中一个返回的时Jack另一个返回的应该时原数据,

> 注:Springboot使用autowired的mapper需要在方法上添加@Transactional,不然每次查询都会新创建一个SqlSession

```
==>  Preparing: select id,name,age from student where id = ?
==> Parameters: 5(Long)
<==    Columns: id, name, age
<==        Row: 5, Jack, 19
<==      Total: 1
Student(id=5, name=Jack, age=19)
JDBC Connection [HikariProxyConnection@436338687 wrapping com.mysql.cj.jdbc.ConnectionImpl@a99c42c] will not be managed by Spring
==>  Preparing: select id,name,age from student where id = ?
==> Parameters: 5(Long)
<==    Columns: id, name, age
<==        Row: 5, Jack, 19
<==      Total: 1
Student(id=5, name=Jack, age=19)
==>  Preparing: update student set name = ? where id =?
==> Parameters: Jhon(String), 5(Long)
<==    Updates: 1
mapper1 update count:1
==>  Preparing: select id,name,age from student where id = ?
==> Parameters: 5(Long)
<==    Columns: id, name, age
<==        Row: 5, Jhon, 19
<==      Total: 1
Student(id=5, name=Jhon, age=19)
Student(id=5, name=Jack, age=19)
```

### 二级缓存

二级缓存由多个SqlSessio共享,

启用二级缓存需要在配置文件中启用

```xml
<setting name="cacheEnabled" value="true"/>
```

然后再对应的映射xml中使用cache标签声明这个namespace使用二级缓存

```xml
<cache/>
```

想要二级缓存生效,SqlSession必须提交,

```java
SqlSession sqlSession1 = sqlSessionFactory.openSession(true);
SqlSession sqlSession2 = sqlSessionFactory.openSession(true);

StudentMapper mapper1 = sqlSession1.getMapper(StudentMapper.class);
StudentMapper mapper2 = sqlSession2.getMapper(StudentMapper.class);

Student student = mapper1.selectByPrimaryKey(4L);
System.out.println(student);
// 如果sqlSession1没有提交,mapper2的查询还是会走数据库
sqlSession1.commit();
Student student1 = mapper2.selectByPrimaryKey(4L);
System.out.println(student1);
```

二级缓存仍然会因为DDL语句而失效

```java
SqlSession sqlSession1 = sqlSessionFactory.openSession(true);
SqlSession sqlSession2 = sqlSessionFactory.openSession(true);
SqlSession sqlSession3 = sqlSessionFactory.openSession(true);

StudentMapper mapper1 = sqlSession1.getMapper(StudentMapper.class);
StudentMapper mapper2 = sqlSession2.getMapper(StudentMapper.class);
StudentMapper mapper3 = sqlSession3.getMapper(StudentMapper.class);

Student student = mapper1.selectByPrimaryKey(4L);
System.out.println(student);
sqlSession1.commit();

Student student1 = mapper2.selectByPrimaryKey(4L);
System.out.println(student1);

mapper3.updateStudentName("小明",4L);
sqlSession3.commit();

Student student2 = mapper2.selectByPrimaryKey(4L);
System.out.println(student2);

```

第二次查询走的缓存,第三次查询因为前面由DDL语句,则走的是数据库

```
JDBC Connection [@59320794 wrapping com.mysql.cj.jdbc.ConnectionImpl@69d667a5] will not be managed by Spring
==>  Preparing: select id,name,age from student where id = ?
==> Parameters: 4(Long)
<==    Columns: id, name, age
<==        Row: 4, Rose, 15
<==      Total: 1
Student(id=4, name=Rose, age=15)
As you are using functionality that deserializes object streams, it is recommended to define the JEP-290 serial filter. Please refer to https://docs.oracle.com/pls/topic/lookup?ctx=javase15&id=GUID-8296D8E8-2B93-4B9A-856E-0A65AF9B8C66
Cache Hit Ratio [com.fool.mybatiscache.mapper.StudentMapper]: 0.5
Student(id=4, name=Rose, age=15)
JDBC Connection [HikariProxyConnection@1111497601 wrapping com.mysql.cj.jdbc.ConnectionImpl@f1d1463] will not be managed by Spring
==>  Preparing: update student set name = ? where id =?
==> Parameters: 小明(String), 4(Long)
<==    Updates: 1
Cache Hit Ratio [com.fool.mybatiscache.mapper.StudentMapper]: 0.3333333333333333
JDBC Connection [HikariProxyConnection@1780974980 wrapping com.mysql.cj.jdbc.ConnectionImpl@28f90752] will not be managed by Spring
==>  Preparing: select id,name,age from student where id = ?
==> Parameters: 4(Long)
<==    Columns: id, name, age
<==        Row: 4, 小明, 15
<==      Total: 1
Student(id=4, name=小明, age=15)
```

因为二级缓存是根据namespace存储的,不同的namespace(相当于一个mapper.xml)之间不会因为DDL而互相影响,所以可能会导致一个namespace修改了数据,而另一个namespace的缓存没有更新,读取到了缓存中的脏数据

解决二级缓存可能读取到缓存中的脏数据问题,可以使用cache-ref标签引用其他namespace的缓存

```xml
<cache-ref namespace="mapper.StudentMapper"/>
```

引用其他namespace的缓存可能导致粒度变粗,多个namespace下的DDL操作都会对缓存造成影响