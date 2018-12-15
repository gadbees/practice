> 前段时间写了[MySql实时数据变更事件捕获kafka confluent之debezium](https://github.com/m65536/practice/blob/master/kafka/MySql%E5%AE%9E%E6%97%B6%E6%95%B0%E6%8D%AE%E5%8F%98%E6%9B%B4%E4%BA%8B%E4%BB%B6%E6%8D%95%E8%8E%B7kafka%20confluent%E4%B9%8Bdebezium.md),使用的是confluent整套的，接下来这篇将会介绍笔者当前公司的实际实战。

> 首先明确需求，公司订单数据越来越大，商户端和E端各种业务需求也越来越多查询越发复杂，我们想引进elasticsearch来实现查询和搜索。那么问题来了，实时更新的订单数据如何同步到es中，业务代码中insert或者update es中的index这肯定是不可取的，我们选择使用kafka和debezium结合使用，读取MySQLbinlog及时写入es.

![](https://github.com/m65536/resource/blob/master/image/kafka/MySQL-kafka-connect-debezium-es-0.png?raw=true)

 本文将会完整大家一套上图环境.

## 安装
#### MySQL
  MySQL的安装比较简单，同时需要MySQL开启binlog，为了简单我这里使用docker启动一个MySQL并且里面已创建有数据。
  
* docker安装
```aidl
docker run -it --rm --name mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=debezium -e MYSQL_USER=mysqluser -e MYSQL_PASSWORD=mysqlpw debezium/example-mysql:0.8

```

* local安装
  如果本地没有docker环境或者MySQL环境可以参考[mysql 5.7 主从同步配置（windows）](https://blog.csdn.net/natahew/article/details/71122569)和[MySQL 5.7.18 数据库主从（Master/Slave）同步安装与配置详解](https://www.jishux.com/plus/view-641331-1.html)配置。
  
  这里给出同上docker的数据库和表结构，点击[msyql table inventory ddl](https://github.com/m65536/resource/blob/master/script/debezium/sql_ddl_0/inventory_ddl.sql)下载。

#### zookeeper
  kafka的启动依赖于zookeeper,所以这里先安装并且启动zookeeper.
* [download](https://mirrors.tuna.tsinghua.edu.cn/apache/zookeeper/zookeeper-3.4.10/)

* [install and start](https://zookeeper.apache.org/doc/r3.4.13/zookeeperStarted.html)
  
  首先需要配置`conf/zoo.cfg`,可以直接复制一份`conf/zoo_sample.cfg`使用.切换到安装目录下`bin/zkServer.sh start`启动zookeeper.

#### kafka
* [kafka download](https://kafka.apache.org/downloads)
* [MySQL Connector plugin archive download](https://repo1.maven.org/maven2/io/debezium/debezium-connector-mysql/0.8.3.Final/debezium-connector-mysql-0.8.3.Final-plugin.tar.gz)

  下载好了的kafka文件目录里面其实默认已经包含了几个connect,这里我们需要使用的是`debezium`这个插件,所以需要把下载后的debezium安装到connect中,安装方法也比较简单,把解压后的`MySQL Connector plugin archive`里面的内容全部copy到kafka安装目录`libs`目录下即可.

* 启动kafka [Quickstart](https://kafka.apache.org/quickstart)

  在安装目录下执行`bin/kafka-server-start.sh config/server.properties`

* 启动kafka connect [Running Kafka Connect
](https://kafka.apache.org/documentation/#connect)

  在安装目录下执行`./bin/connect-distributed.sh config/connect-distributed.properties`

#### elasticsearch