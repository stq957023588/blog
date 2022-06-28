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

## 索引相关的文章

[索引是如何实现的](https://baijiahao.baidu.com/s?id=1721102142129449236&wfr=spider&for=pc)

[聚集索引和非聚集索引的区别](https://m.php.cn/article/489392.html)

## 索引失效的情况

1. 当使用 like 时 % 在前面(e: name like '%Rose'),其他情况诸如在中间以及末尾索引都会生效(e:name like 'R%ose%')

2. 当使用or时可能会失效

   1. or 两边都是同一个表的条件时,两边的字段都是索引时,索引生效(e: s.name = 'rose' or s.age = 14,age 和 name都是索引);当其中一个不是索引的时候,两个字段的索引都不会生效(e: s.remark = '123' or s.name = 'rose', remark 不是索引, name是索引)
   2. or 两边不是同一个表的条件,并且时以join进行关联时,不管主表字段是否时索引字段,只要join表的字段是索引,那么join表的字段索引一定会被触发

3. 使用组合索引时,如果查询条件中没有带有索引第一个字段,则索引失效,

   ```sql
   create index test on student(name,age,sex);
   # 索引生效
   select * from student where name = 'rose' and age =11 and sex = 0;
   # 索引生效
   select * from student where name = 'rose' and sex = 0;
   # 索引未生效
   select * from student where name = 'rose' and age =11 and sex = 0;
   ```

4. 当索引字段冲突时,后面创建的索引失效

5. 当查询字段是varchar时,条件为数字时(e: name = 123, name是varchar),发生隐式转换,会导致索引失效,而当查询字段是数字,而条件是字符串时,索引仍旧为生效

   ```sql
   create index student_name on student(name);
   create index student_age on student(age);
   # 索引失效
   select * from student where name = 123;
   # 索引生效
   select * from student where age = '123';
   ```

6. 对字段进行操作,或者使用函数时索引失效(e: age + 1 = 10;left(name, 2) = 'rose')

7. 使用 !=, not 时索引可能失效,原因就是第八条:当全表扫描速度比索引速度快时，mysql会使用全表扫描，此时索引失效

8. 当全表扫描速度比索引速度快时，mysql会使用全表扫描，此时索引失效。

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