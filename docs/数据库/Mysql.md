# Mysql中如何排查响应慢的SQL

启用慢查询日志

```sql
# 查看是否打开慢查询日志
SHOW VARIABLES LIKE 'slow_query_log';
# 设置开启慢查询日志
SET GLOBAL slow_query_log = 'ON';
# 设置慢查询阈值
SET GLOBAL long_query_time = 1; -- 记录执行时间超过1秒的查询
```

查看慢查询日志

```sql
SHOW VARIABLES LIKE 'slow_query_log_file';
```

导出慢查询日志

```shell
# 显示执行时间最长的前10条查询
mysqldumpslow -s t -t 10 /path/to/slow_query.log
```

# Mysql锁的种类

![aaaaa.awebp](微信图片_20241217112010.png)

来源：

[史上最全MySQL各种锁详解一、前言锁是计算机在执行多线程或线程时用于并发访问同一共享资源时的同步机制，MySQL中的锁 - 掘金](https://juejin.cn/post/6931752749545553933)

# 隔离级别

**读未提交（Read Uncommitted）**：在该隔离级别下，事务可以读取其他未提交事务所做的更改，可能导致脏读、不可重复读和幻读问题。

**读已提交（Read Committed）**：事务只能读取其他已提交事务的更改，避免了脏读问题，但仍可能发生不可重复读和幻读。

**可重复读（Repeatable Read）**：在一个事务内的多次读取操作，结果是一致的，即使其他事务进行了提交的修改。这避免了脏读和不可重复读问题，但可能出现幻读。需要注意的是，InnoDB 存储引擎通过多版本并发控制（MVCC）和 Next-Key Locking 机制解决了幻读问题，因此在 InnoDB 中，可重复读隔离级别也避免了幻读。

**可串行化（Serializable）**：最高的隔离级别，强制事务串行执行，完全避免脏读、不可重复读和幻读问题。然而，这种隔离级别会显著降低并发性能，因此在实际应用中较少使用。

# MVCC

MVCC（Multi-Version Concurrency Control，**多版本并发控制**）是一种用于管理数据库并发访问的机制，旨在提高数据库的并发性能并确保数据的一致性。在 MySQL 的 InnoDB 存储引擎中，MVCC 通过维护数据行的多个版本，使读操作无需加锁即可执行，从而实现高效的并发控制。

**MVCC 的工作原理：**

1. **数据版本管理：**
   - 每行数据在其隐藏字段中包含两个版本号：`DB_TRX_ID`（最近修改该行的事务 ID）和 `DB_ROLL_PTR`（指向该行修改前版本的指针）。
   - 当事务对数据行进行修改时，InnoDB 会创建该行的一个新版本，并更新 `DB_TRX_ID` 和 `DB_ROLL_PTR`，形成一个版本链。
2. **Undo 日志：**
   - 每次数据修改操作都会生成一条 Undo 日志，记录被修改行的旧版本数据。
   - 这些 Undo 日志用于构建数据行的多版本链，支持事务的回滚操作。
3. **Read View（读取视图）：**
   - 当事务启动时，InnoDB 创建一个 Read View，记录当前活跃事务的快照。
   - 在读取数据时，事务根据 Read View 判断数据行的可见性，确保读取到符合隔离级别要求的数据版本。

**MVCC 的优势：**

- **提高并发性能：**
  - 通过维护数据的多个版本，读操作无需加锁即可执行，减少了锁竞争，提高了并发性能。
- **实现一致性读：**
  - 事务在读取数据时，可以根据其隔离级别读取特定版本的数据，确保数据的一致性。

**MVCC 的局限性：**

- **存储开销：**
  - 维护多个数据版本和 Undo 日志会增加存储空间的消耗。
- **版本管理复杂性：**
  - 需要定期清理无用的旧版本数据，以防止版本链过长，影响查询性能。

参考：

[看完这篇还不懂MySQL的MVCC机制算我输-阿里云开发者社区](https://developer.aliyun.com/article/1115763)

# Linux压缩包安装Mysql

```shell
wget https://downloads.mysql.com/archives/get/p/23/file/mysql-8.1.0-linux-glibc2.28-x86_64.tar.xz
xz -d mysql-8.1.0-linux-glibc2.28-x86_64.tar.xz
tar -xvf mysql-8.1.0-linux-glibc2.28-x86_64.tar
mv mysql-8.1.0-linux-glibc2.28-x86_64 mysql
mkdir data
./bin/mysqld --initialize --user=mysql --basedir=/usr/local/mysql --datadir=/usr/local/mysql/data
./support-files/mysql.server start
```

如果有报错：

```shell
./bin/mysqld: error while loading shared libraries: libaio.so.1: cannot open shared object file: No such file or directory
```

安装下libaio

```shell
sudo apt-get install libaio1
```

# Win压缩包安装Mysql

[Win压缩包安装MySql](https://zhuanlan.zhihu.com/p/265148449)

1. 官网[下载](https://dev.mysql.com/downloads/mysql/)Zip压缩包
2. 解压缩后进入文件夹，在文件夹下创建data文件夹和my.ini文件
3. 修改my.ini

```ini
[mysql]
default-character-set = utf8mb4    
[mysqld]
default-storage-engine=INNODB
# MySQL安装根目录的路径。
basedir=C:\Program Files\mysql-8.0.33-winx64
# MySQL服务器数据目录的路径
datadir=C:\Program Files\mysql-8.0.33-winx64\data
#服务端默认编码（数据库级别）
character_set_server = utf8mb4 
```

4. 环境变量下path下添加 bin 路径
5. 管理员打开CMD命令窗口，运行命令

```shell
mysqld --initialize
mysqld --install
```

6. 启动mysql

```shell
net start mysql
```

7. 查看mysql root用户临时密码，打开新建的data文件夹下 ``.err``结尾的文件，找到 A temporary password is generated for root@localhost：[临时密码]
8. 登陆MySQL

```shell
mysql -u root -p [临时密码]
```

> 如果局域网内无法连接，在MySql所在Win上创建入站规则

# 基础操作

命令行登录

```shell
# mysql -u用户名 -p密码 -h主机
mysql -uroot -p123456 -h127.0.0.1
```

查看是否有安装Mysql

```shell
mysql -V
```

查询my.cnf所在位置

```shell
find / -name my.cnf
```

# 数据库锁的种类



参考

> [MySQL 中有哪些锁？表级锁和行级锁有什么区别？](https://mp.weixin.qq.com/s/Inocu3vjMG4ivE19HrxR3g)

# 数据备份以及恢复

## 备份

备份数据库(database)

```shell
# mysqldump -u用户名 -p -h主机 数据库名 > 存放目标文件全路径
mysqldump -uroot -p laboratory > laboratory_backup.sql
```

备份表

```shell
# mysqldump -u用户名 -p -h主机 数据库名 表名 > 存放目标文件全路径
mysqldump -uroot -p laboratory student > laboratory_student_backup.sql
```

备份时报错可以用的参数

```shell
# SSLConnection连接异常
--ssl-mode=disabled
# 用户锁表权限问题（LOCK TABLES)
--skip-lock-tables
# column statistic
--column-statistics=0
```



## 恢复

恢复数据库数据,数据库(database)必须先创建好

```shell
mysql -uroot -p laboratory < laboratory_backup.sql
```

恢复表数据,表可以不事先创建好

```shell
mysql -uroot -p laboratory < laboratory_student_backup.sql
```

使用source恢复数据,需要在客户端进行

```sql
# source 文件名称
source laboratory_backup.sql
```

参考

[mysql 使用sqldump来进行数据库还原](http://t.zoukankan.com/Eaglery-p-5278754.html)

[MySQLmysqldump命令](https://blog.csdn.net/weixin_42347763/article/details/113168026)

# 一些报错

无法插入Emoji表情错误

```
java.sql.SQLException: Incorrect string value: '\xF0\x9F\x90\xBA' for column 'xxx' at row 1
```

解决办法:

1. 修改字段编码

   ```sql
   ALTER TABLE api_log MODIFY COLUMN remark longtext CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
   ```

2. 创建表的时候设置编码

   ```sql
   create table tableName1
   (
   id int not null auto_increment,
   opName varchar(100) not null,
   sysUrl varchar(200) not null,
   createId varchar(36) not null,
   createName varchar(100) not null,
   createTime varchar(19) not null,
   primary key (id)
   ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
   ```


# 存储过程



# 常用方法

## 随机中国人名

```sql
create definer = root@`%` function rand_name() returns varchar(16)
begin
    declare family_str varchar(256) default '赵钱孙李周吴郑王冯陈褚卫蒋沈韩杨朱秦尤许何吕施张孔曹严华金魏陶姜戚谢邹喻柏水窦章云苏潘葛奚范彭郎鲁韦昌马苗凤花方俞任袁柳酆鲍史唐费廉岑薛雷贺倪汤滕殷罗毕郝邬安常乐于时傅皮卞齐康伍余元卜顾孟平黄和穆萧尹姚邵湛汪祁毛禹狄米贝明臧计伏成戴谈宋茅庞熊纪舒屈项祝董梁杜阮蓝闵席季麻强贾路娄危江童颜郭梅盛林刁锺徐邱骆高夏蔡田樊胡凌霍虞万支柯昝管卢莫';
    declare name_str varchar(512) default '谦亨奇固之轮翰朗伯宏先柏镇淇淳一洁铭皑言若鸣朋斌梁栋维启克伦翔旭鹏泽晨辰士以建家致树炎德行时泰盛雄琛钧冠策腾楠榕风航弘瑛玲憧萍雪珍滢筠柔竹霭凝晓欢霄枫芸菲寒伊亚宜可姬舒影荔枝丽秀娟英华慧巧美静淑惠珠莹雪琳晗瑶允元源渊和函妤宜云琪勤珍贞莉兰凤洁琳素云莲真环雪荣爱妹霞亮香月媛艳瑞凡佳嘉叶璧璐娅琦晶妍茹清吉克茜秋珊莎锦黛青倩婷姣婉娴瑾颖露瑶怡婵雁蓓纨仪荷丹蓉眉君琴蕊薇菁梦岚苑婕馨瑗琰韵融园艺咏卿聪澜纯毓悦昭冰爽琬茗羽希宁欣飘育涵琴晴丽美瑶梦茜倩希夕月悦乐彤影珍依沫玉灵瑶嫣倩妍萱漩娅媛怡佩淇雨娜莹娟文芳莉雅芝文晨宇怡全子凡悦思奕依浩泓钊钧铎';
    declare i int default 0;
    declare full_name varchar(32) default '';
    declare rand_num int default 0;
    # 姓氏
    set full_name = concat(full_name,substr(family_str,floor(1+rand()*160),1));
    set full_name = concat(full_name,substr(name_str,floor(1+rand()*256),1));
    set rand_num = rand()*10;
    set full_name= if(rand_num>5,concat(full_name,substr(name_str,floor(1+rand()*256),1)),full_name);
    set rand_num = rand()*100;
    set full_name= if(rand_num<2,concat(full_name,substr(name_str,floor(1+rand()*256),1)),full_name);

    return full_name;
end;
```

## 随机地名

```sql
create definer = root@`%` function rand_place_name(len int) returns varchar(32)
begin
    declare prefixPool varchar(512) default '谦亨奇固之轮翰朗伯宏先柏镇淇淳一洁铭皑言若鸣朋斌梁栋维启克伦翔旭鹏泽晨辰士以建家致树炎德行时泰盛雄琛钧冠策腾楠榕风航弘瑛玲憧萍雪珍滢筠柔竹霭凝晓欢霄枫芸菲寒伊亚宜可姬舒影荔枝丽秀娟英华慧巧美静淑惠珠莹雪琳晗瑶允元源渊和函妤宜云琪勤珍贞莉兰凤洁琳素云莲真环雪荣爱妹霞亮香月媛艳瑞凡佳嘉叶璧璐娅琦晶妍茹清吉克茜秋珊莎锦黛青倩婷姣婉娴瑾颖露瑶怡婵雁蓓纨仪荷丹蓉眉君琴蕊薇菁梦岚苑婕馨瑗琰韵融园艺咏卿聪澜纯毓悦昭冰爽琬茗羽希宁欣飘育涵琴晴丽美瑶梦茜倩希夕月悦乐彤影珍依沫玉灵瑶嫣倩妍萱漩娅媛怡佩淇雨娜莹娟文芳莉雅芝文晨宇怡全子凡悦思奕依浩泓钊钧铎';
    declare suffixPool varchar(256) default '市县镇村城街路巷角州';
    declare place_name varchar(32) default '';

    declare i int unsigned default 0;
    while i < len
        do
            set place_name = concat(place_name, substr(prefixPool, floor(rand() * 269), 1));
            set i = i + 1;
        end while;
    set place_name = concat(place_name, substr(suffixPool, floor(rand() * 11), 1));

    return place_name;
end;
```

## 随机字母数字字符串

```sql
create definer = root@`%` function rand_str(length int) returns varchar(255)
BEGIN
    DECLARE i INT UNSIGNED DEFAULT 0; DECLARE v_result VARCHAR(200) DEFAULT ''; DECLARE v_dict VARCHAR(200) DEFAULT '';
    SET v_dict = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789';
    SET v_dict = LPAD(v_dict, 200, v_dict);
    WHILE i < length
        DO
            SET v_result = CONCAT(v_result, SUBSTR(v_dict, CEIL(RAND() * 200), 1)); SET i = i + 1;
        END WHILE;
    RETURN v_result;
END;
```

# 数据库表设计三范式

[参考](https://zhuanlan.zhihu.com/p/216537638)

## 1NF

第一范式需要保证每一列的是原子的,不可分割的

> 地址字段可以被分为省份,市,县等几部分

## 2Nf

一是表必须有一个主键；二是没有包含在主键中的列必须完全依赖于主键，而不能只依赖于主键的一部分。

> 存在成绩表(学生ID,课程ID,课程名称,成绩),其中课程名称只依赖于课程ID,而成绩依赖学生ID,和课程ID,课程名称只依赖了主键的一部分

## 3NF

首先是 2NF，另外非主键列必须直接依赖于主键，不能存在传递依赖。即不能存在：非主键列 A 依赖于非主键列 B，非主键列 B 依赖于主键的情况。

> 学生表中存在班级ID和班级名称,存在传递依赖,班级名称依赖班级ID,班级ID依赖学生表主键

# 索引

mysql中InnoDB存储引擎的索引结构是B+树结构，他的叶子节点是含有指向前后叶子节点的一个双向链表；叶子节点在逻辑上应当是顺序分布的，但是在磁盘上的物理存储位置不是顺序分布的

## 索引相关的文章

[索引是如何实现的](https://baijiahao.baidu.com/s?id=1721102142129449236&wfr=spider&for=pc)

[聚集索引和非聚集索引的区别](https://m.php.cn/article/489392.html)

# 创建用户

```sql

```

# 主从复制

## 一主一从

* 在主数据库的my.cnf的[mysqld]下新增2行配置
  > [mysqld]
  > ...   
  > \#开启二进制日志   
  > log-bin=mysql-bin   
  > \#设置server-id   
  > server-id=1

* 重启主数据库
  
  > service mysqld restart
  
* 创建用于复制数据的账号
  ```sql
  # 创建账号 123.123.123.123是从数据库所在服务器的ip地址
  create user 'repl'@123.123.123.123' identified by 'fool.123';
  # 分配权限
  grant replication slave on *.* to 'repl@123.123.123.123';
  # 刷新权限
  flush privileges;
  # 查看master数据库(主数据库)状态,并记录file 和 position的值
  show master status;
  ```
* 修改从数据库配置my.cnf
  > [mysqld]   
  > \#设置server-id 必须唯一   
  > server-id=2
* 执行同步sql语句(需登录mysql)
  ```sql
  # master_log_file是主数据库运行show master status 中的file值
  # master_log_pos master status 中的position值
  change master to 
  master_host='111.111.111.111',
  master_user='repl'
  master_password='fool.123'
  master_log_file='msql-bin.000001'
  master_log_pos=48;
  # 启动主从复制
  start slave;
  # 关闭主从复制
  stop slave;
  # 重置主从复制
  reset slave;
  # 查看从数据库状态 Slave_IO_Running 和 Slave_SQL_Running都是yes表明运行正常
  # Last_error表示最近主从复制遇到的问题
  show slave status;
  ```
* 修改主数据库my.cnf设置只同步或不同步主数据库中的一些database
  > [mysqld]
  > ...
  > \#不同步数据库
  > binlog-ignore-db=test
  > \#之同步数据库
  > binlog-do-db=fool
* 注意
  ```sql
  # 创建表时最好设置engine和default charset
  create table student(
    id int auto_increment primary key ,
    name varchar(16)
  )engine=InnoDb default charset=utf8;
  ```

## 多主一从

* 配置主数据
  
  > 主数据库配置与一主一从一致,但是每个主数据库的server-id不能相同
  
* 配置从数据库
  > 在一主一从的配置基础上添加2行配置
  > [mysqld]
  > master_info_repository=table
  > relay_log_info_repository=table
* 执行同步sql语句(需登录mysql)
  ```sql
  # master_log_file是主数据库运行show master status 中的file值
  # master_log_pos master status 中的position值
  # for channel '100' 的100是对应主数据库的server-id
  # 多个主数据库就执行多次,server-id不同即可
  change master to 
  master_host='111.111.111.111',
  master_user='repl'
  master_password='fool.123'
  master_log_file='msql-bin.000001'
  master_log_pos=48
  for channel '100';
  # 启动主从复制
  start slave;
  # 关闭主从复制
  stop slave;
  # 重置主从复制
  reset slave;
  # 查看从数据库状态 Slave_IO_Running 和 Slave_SQL_Running都是yes表明运行正常
  # Last_error表示最近主从复制遇到的问题
  show slave status;
  ```

# 一些问题

## group_cancat

1. 在无数据的情况下使用group_cancat会产生一条全是null的数据，需要增加一个group by语句