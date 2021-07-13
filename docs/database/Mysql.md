# 基础操作

* 查看是否有安装Mysql
  > mysql -V
* 查询my.cnf所在位置
  > find / -name my.cnf

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
  create user 'repl@123.123.123.123' identity by 'fool.123';
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