# Canal

一个用于增量数据同步的中间件

# 前置条件

MySql开启binlog

```ini
[mysqld]
# 打开binlog
log-bin=mysql-bin
# 选择ROW(行)模式
binlog-format=ROW
# 配置MySQL replaction需要定义，不要和canal的slaveId重复
server_id=1
```

创建用于Canal的Mysql账户

```sql
-- 使用命令登录：mysql -u root -p
-- 创建用户 用户名：canal 密码：Canal@123456
create user 'canal'@'%' identified by 'Canal@123456';
-- 授权 *.*表示所有库
grant SELECT, REPLICATION SLAVE, REPLICATION CLIENT on *.* to 'canal'@'%' ;
```

查看binlog日志文件列表

```sql
show binary logs;
```

查看正在使用的binlog

```sql
show master status;
```

# 安装

[下载](https://github.com/alibaba/canal/releases)

修改配置文件（conf/example/instance.properties）

```properties
## mysql serverId , v1.0.26+ will autoGen
## v1.0.26版本后会自动生成slaveId，所以可以不用配置
# canal.instance.mysql.slaveId=0

# 数据库地址
canal.instance.master.address=127.0.0.1:3306
# binlog日志名称,show binary logs语句可查看
canal.instance.master.journal.name=mysql-bin.000001
# mysql主库链接时起始的binlog偏移量，show binary logs语句可查看
canal.instance.master.position=154
# mysql主库链接时起始的binlog的时间戳
canal.instance.master.timestamp=
canal.instance.master.gtid=

# username/password
# 在MySQL服务器授权的账号密码
canal.instance.dbUsername=canal
canal.instance.dbPassword=Canal@123456
# 字符集
canal.instance.connectionCharset = UTF-8
# enable druid Decrypt database password
canal.instance.enableDruid=false

# table regex .*\\..*表示监听所有表 也可以写具体的表名，用，隔开
canal.instance.filter.regex=.*\\..*
# mysql 数据解析表的黑名单，多个表用，隔开
canal.instance.filter.black.regex=
```

打开`startup.bat`文件启动

# 客户端

## Java

添加依赖

Maven

```xml
<dependency>
    <groupId>com.alibaba.otter</groupId>
    <artifactId>canal.client</artifactId>
    <version>1.1.4</version>
</dependency>
```

Gradle

```groovy
implementation('com.alibaba.otter:canal.client:1.1.4')
```

示例代码

```java
@Component
public class CannalClient implements InitializingBean {

    private final static int BATCH_SIZE = 1000;

    @Override
    public void afterPropertiesSet() throws Exception {
        // 创建链接
        CanalConnector connector = CanalConnectors.newSingleConnector(new InetSocketAddress("127.0.0.1", 11111), "example", "", "");
        try {
            //打开连接
            connector.connect();
            //订阅数据库表,全部表
            connector.subscribe(".*\\..*");
            //回滚到未进行ack的地方，下次fetch的时候，可以从最后一个没有ack的地方开始拿
            connector.rollback();
            while (true) {
                // 获取指定数量的数据
                Message message = connector.getWithoutAck(BATCH_SIZE);
                //获取批量ID
                long batchId = message.getId();
                //获取批量的数量
                int size = message.getEntries().size();
                //如果没有数据
                if (batchId == -1 || size == 0) {
                    try {
                        //线程休眠2秒
                        Thread.sleep(2000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                } else {
                    //如果有数据,处理数据
                    printEntry(message.getEntries());
                }
                //进行 batch id 的确认。确认之后，小于等于此 batchId 的 Message 都会被确认。
                connector.ack(batchId);
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            connector.disconnect();
        }
    }

    /**
     * 打印canal server解析binlog获得的实体类信息
     */
    private static void printEntry(List<Entry> entrys) {
        for (Entry entry : entrys) {
            if (entry.getEntryType() == EntryType.TRANSACTIONBEGIN || entry.getEntryType() == EntryType.TRANSACTIONEND) {
                //开启/关闭事务的实体类型，跳过
                continue;
            }
            //RowChange对象，包含了一行数据变化的所有特征
            //比如isDdl 是否是ddl变更操作 sql 具体的ddl sql beforeColumns afterColumns 变更前后的数据字段等等
            RowChange rowChage;
            try {
                rowChage = RowChange.parseFrom(entry.getStoreValue());
            } catch (Exception e) {
                throw new RuntimeException("ERROR ## parser of eromanga-event has an error , data:" + entry.toString(), e);
            }
            //获取操作类型：insert/update/delete类型
            EventType eventType = rowChage.getEventType();
            //打印Header信息
            System.out.println(String.format("================》; binlog[%s:%s] , name[%s,%s] , eventType : %s",
                    entry.getHeader().getLogfileName(), entry.getHeader().getLogfileOffset(),
                    entry.getHeader().getSchemaName(), entry.getHeader().getTableName(),
                    eventType));
            //判断是否是DDL语句
            if (rowChage.getIsDdl()) {
                System.out.println("================》;isDdl: true,sql:" + rowChage.getSql());
            }
            //获取RowChange对象里的每一行数据，打印出来
            for (RowData rowData : rowChage.getRowDatasList()) {
                //如果是删除语句
                if (eventType == EventType.DELETE) {
                    printColumn(rowData.getBeforeColumnsList());
                    //如果是新增语句
                } else if (eventType == EventType.INSERT) {
                    printColumn(rowData.getAfterColumnsList());
                    //如果是更新的语句
                } else {
                    //变更前的数据
                    System.out.println("------->; before");
                    printColumn(rowData.getBeforeColumnsList());
                    //变更后的数据
                    System.out.println("------->; after");
                    printColumn(rowData.getAfterColumnsList());
                }
            }
        }
    }

    private static void printColumn(List<Column> columns) {
        for (Column column : columns) {
            System.out.println(column.getName() + " : " + column.getValue() + "    update=" + column.getUpdated());
        }
    }
}
```

参考：

[超详细的canal入门，看这篇就够了-阿里云开发者社区](https://developer.aliyun.com/article/770496)

## RabbitMQ

创建RabbitMQ交换机、队列，并进行互相绑定

配置conf\canal.properties

```properties
# tcp, kafka, rocketMQ, rabbitMQ, pulsarMQ
# 配置服务器模式为RabbitMQ
canal.serverMode = rabbitMQ
# 配置主机
rabbitmq.host = localhost
# 配置虚拟主机
rabbitmq.virtual.host = /
# 配置交换机名称
rabbitmq.exchange = mysql.exchange
# 配置RabbitMQ用户名
rabbitmq.username = guest
# 配置RabbitMQ密码
rabbitmq.password = guest
# 配置路由Key
rabbitmq.routingKey = canal.routing.key	
# 配置交换机类型
rabbitmq.deliveryMode = fanout
```

>windows环境下，需要将plugin文件夹下的jar包复制到lib包下

