# MyCat

## 在Linux安装部署

### 下载安装

1. 从[官网](http://www.mycat.org.cn)下载
2. 上传到服务器后解压
   > 这里时上传到了/data文件夹下
   > tar -zxvf Mycat-server-1.6.7.6-release-20201112144313-linux.tar.gz

### 配置

#### 配置用户信息

```xml
   <!-- 用户名 -->
<user name="admin">
    <!-- 密码 -->
    <property name="password">123456</property>
    <!-- 库名,多个库名使用逗号隔开.schema.xml中schema标签的name值 -->
    <property name="schemas">ami</property>
    <!-- 默认库 -->
    <property name="defaultSchema">ami</property>
</user>
```

#### 配置数据库主机

```xml

<dataHost name="dh@127.0.0.1" maxCon="1000" minCon="10" balance="0" writeType="0" dbType="mysql" dbDriver="jdbc"
          switchType="1" slaveThreshold="100">
    <heartbeat>select user()</heartbeat>
    <!-- 真实数据库信息,可配置多个 -->
    <writeHost host="hostM1" url="jdbc:mysql://127.0.0.1:3306" user="root" password="root"/>
</dataHost>
```

#### 配置节点

```xml
<!-- name:节点名称,自定义;dataHost:数据库主机名称;database:真实数据库中存在的database -->
<dataNode name="mycat" dataHost="dh@127.0.0.1" database="mycat"/>
```

#### 配置库

```xml
<!-- name:库名,用户配置中的schema sqlMaxLimit:一次性最大查询量 -->
<schema name="ami" checkSQLschema="true" sqlMaxLimit="2000" randomDataNode="mycat">
</schema>
```

##### 配置表

```xml

<schema name="ami" checkSQLschema="true" sqlMaxLimit="100" randomDataNode="mycat">
    <!-- name:数据库表明,需真实存在 -->
    <!-- primaryKey: 主键,对应真实表的主键 -->
    <!-- dataNode: 对应节点,多个节点使用逗号隔开, -->
    <!-- rule: 分片规则,具体配置在rule.xml中 -->
    <!-- autoIncrement: 是否自增,自增具体配置在下面 -->
    <!--fetchStoreNodeByJdbc 启用ER表使用JDBC方式获取DataNode-->
    <table name="users" primaryKey="id" dataNode="mycat,fool" rule="mod-long" autoIncrement="true"
           fetchStoreNodeByJdbc="true"/>
</schema>
```

## 添加序列

1. 在对应的dataNode下面的一个database中执行sql语句,

```sql
DROP TABLE IF EXISTS MYCAT_SEQUENCE;
CREATE TABLE MYCAT_SEQUENCE
(
    NAME          VARCHAR(50) NOT NULL,
    current_value INT         NOT NULL,
    increment     INT         NOT NULL DEFAULT 100,
    PRIMARY KEY (NAME)
) ENGINE = INNODB;


INSERT INTO MYCAT_SEQUENCE(NAME, current_value, increment)
VALUES ('GLOBAL', 100000, 100);

DROP FUNCTION IF EXISTS `mycat_seq_currval`;
DELIMITER ;;
CREATE FUNCTION `mycat_seq_currval`(seq_name VARCHAR(50))
    RETURNS VARCHAR(64) CHARSET utf8
    DETERMINISTIC
BEGIN
    DECLARE retval VARCHAR(64);
    SET retval = "-999999999,null";
    SELECT CONCAT(CAST(current_value AS CHAR), ",", CAST(increment AS CHAR))
    INTO retval
    FROM MYCAT_SEQUENCE
    WHERE NAME = seq_name;
    RETURN retval;
END
;;
DELIMITER ;

DROP FUNCTION IF EXISTS `mycat_seq_nextval`;
DELIMITER ;;
CREATE FUNCTION `mycat_seq_nextval`(seq_name VARCHAR(50))
    RETURNS VARCHAR(64)
        CHARSET utf8
    DETERMINISTIC
BEGIN
    UPDATE MYCAT_SEQUENCE
    SET current_value = current_value + increment
    WHERE NAME = seq_name;
    RETURN mycat_seq_currval(seq_name);
END
;;
DELIMITER ;


DROP FUNCTION IF EXISTS `mycat_seq_setval`;
DELIMITER ;;
CREATE FUNCTION `mycat_seq_setval`(seq_name VARCHAR(50), VALUE INTEGER)
    RETURNS VARCHAR(64) CHARSET utf8
    DETERMINISTIC
BEGIN
    UPDATE MYCAT_SEQUENCE
    SET current_value = VALUE
    WHERE NAME = seq_name;
    RETURN mycat_seq_currval(seq_name);
END
;;
DELIMITER ;
```

2. 在/conf/sequence_db_conf.properties下添加对应的参数
   > STUDENT=mycat
   > STUDENT是表明对应的大写,mycat是对应的dataNode

## Windows上安装部署

### 启动

1. 