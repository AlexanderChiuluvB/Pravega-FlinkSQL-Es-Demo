# Pravega-FlinkSQL-Es-Demo
Demo：基于 Flink SQL &amp; Pravega 构建流式应用

Demo: Streaming application with Flink SQL and Pravega

本demo使用到的文档,数据,DockerFile,部分镜像都参考自云邪大佬的Flink-sql-demo(https://github.com/wuchong/flink-sql-demo)

本demo的英文版本会贡献给Pravega社区,欢迎star与fork(https://github.com/pravega/pravega-samples)

本文将基于 Pravega, MySQL, Elasticsearch, Kibana，使用 Flink SQL 构建一个电商用户行为的实时分析应用。本文所有的操作都将在 Flink SQL CLI 上执行，全程只涉及 SQL 纯文本，无需一行 Java/Scala 代码，无需安装 IDE。

最终效果图:
![result](https://github.com/AlexanderChiuluvB/Pravega-FlinkSQL-Es-Demo/blob/master/pic/Screenshot%20from%202020-04-19%2000-07-37.png)

## Prerequsites

A computer with Linux or MacOS installed with Docker and Java8

### Use Docker compose to start applications

All the applications will be started as docker container with `docker-compose up -d`

```yaml
version: '2.1'
services:
  pravega:
    image: pravega/pravega
    ports:
        - "9090:9090"
        - "12345:12345"
    command: "standalone"
    network_mode: host
  datagen:
    image: 16307110258/datagen:0.1
    command: "java -classpath /opt/datagen/datagen.jar sender --input /opt/datagen/user_behavior.log --speedup 1000"
    depends_on:
      - pravega
    network_mode: host
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.6.0
    environment:
      - cluster.name=docker-cluster
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - discovery.type=single-node
    ports:
      - "9200:9200"
      - "9300:9300"
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
  kibana:
    image: docker.elastic.co/kibana/kibana:7.6.0
    ports:
      - "5601:5601"
  mysql:
    image: jark/mysql-example:0.1
    ports:
      - "3306:3306"
    environment:
      - MYSQL_ROOT_PASSWORD=123456
    network_mode: host

```

The Docker compose include below containers:

* Datagen:

Data generator.When started, it will send data to Pravega cluster automatically. 
By default it will send 1000 data per second, lasting for about 3 hours. You can also
modify the `speedup` parameter in `docker-compose.yml` to adjust the data generatoring speed.

* Pravega:

Used as streaming data storage. Data generatored by Datagen container will be sent to Pravega.
And the data stored in Pravega will be sent to Flink using Pravega-Flink connector.(https://github.com/pravega/flink-connectors)

* MySQL:

MySQL5.7 with pre-defined `category` table

* Elasticsearch:

Storage of Flink SQL result data.

* Kibana:

Visualize Elasticsearch index data for OLAP.

Use the following command to start all the containers.
`docker-compose up -d`

You can use `docker ps` to check whether all the containers are working or visit ` http://localhost:5601/ ` to check
whether kibana is working.

Finally, you can use the following command to stop all the containers.
`docker-compose down`


### Download and install Flink 1.10

1.Download Flink 1.10.0 ：https://www.apache.org/dist/flink/flink-1.10.0/flink-1.10.0-bin-scala_2.12.tgz

2.`cd flink-1.10.0`

3.Download the following jars and copy them to lib/
```

wget -P ./lib/ https://repo1.maven.org/maven2/org/apache/flink/flink-json/1.10.0/flink-json-1.10.0.jar | \
wget -P ./lib/ https://repo1.maven.org/maven2/org/apache/flink/flink-sql-connector-elasticsearch6_2.12/1.10.0/flink-sql-connector-elasticsearch6_2.12-1.10.0.jar | \
wget -P ./lib/ https://repo1.maven.org/maven2/org/apache/flink/flink-jdbc_2.12/1.10.0/flink-jdbc_2.12-1.10.0.jar | \
wget -P ./lib/ https://repo1.maven.org/maven2/mysql/mysql-connector-java/5.1.48/mysql-connector-java-5.1.48.jar

```
**TODO**
Add Flink pravega connector jar URL. 写作本文时候bugfix版本仍未发布。

4.Modify ` taskmanager.numberOfTaskSlots` as 10 in `conf/flink-conf.yaml` since we will launch multiple tasks in Flink.

5.`./bin/start-cluster.sh` start the Flink cluster.
If succeed, you can visited Flink Web UI in http://localhost:8081.

6.`bin/sql-client.sh embedded ` to start the FlinkSQL client.


### Create Pravega Table with DDL

The datagen container will send data to `testSteam` stream in `examples` scope of Pravega.

(You can compare the `stream` concept in Pravega with the `topic` conecpt in Kafka, and scope is namespace in Pravega. For more
information please refer [Pravega-conecpt](http://pravega.io/docs/latest/pravega-concepts/))


```sql
CREATE TABLE user_behavior (
   user_id BIGINT,
    item_id BIGINT,
    category_id BIGINT,
    behavior STRING,
    ts TIMESTAMP(3),
    proctime as PROCTIME(),
    WATERMARK FOR ts as ts - INTERVAL '5' SECOND
) WITH (
  'update-mode' = 'append',
  'connector.type' = 'pravega',
  'connector.version' = '1',
  'connector.metrics' = 'true',
  'connector.connection-config.controller-uri' = 'tcp://localhost:9090',
  'connector.connection-config.default-scope' = 'examples',
  'connector.reader.stream-info.0.stream' = 'testStream',
  'connector.writer.stream' = 'testStream',
  'connector.writer.mode' = 'atleast_once',
  'connector.writer.txn-lease-renewal-interval' = '10000',
  'format.type' = 'json',
  'format.fail-on-missing-field' = 'false'
)
```

We defined 5 fields in the schema. Besides, we use `PROCTIME()` built-in function to create a processing field `proctime`.
We also use WATERMARK grammar to declare watermark policy on `ts` field. (Out-of-order data within 5 seconds will be tolerated.) For more information about time attrubite and DDL grammar Please refer 

* Time attribtue:
https://ci.apache.org/projects/flink/flink-docs-release-1.10/dev/table/streaming/time_attributes.html

* DDL:
https://ci.apache.org/projects/flink/flink-docs-release-1.10/dev/table/sql/create.html#create-table


Use `show tables;` and `describe user_behavior;` to check whether the table is created successfully.

Then we can run `SELECT * FROM user_behavior;` in sql-client to take a glimpse of the data.

![SAMPLE1](https://github.com/AlexanderChiuluvB/Pravega-FlinkSQL-Es-Demo/blob/master/pic/Screenshot%20from%202020-04-21%2016-44-09.png)


## Count the transaction number per hour

### Use DDL to create Elasticsearch table.

We first create a ES sink table to save the `hour_of_day` and `buy_cnt`.

```sql
CREATE TABLE buy_cnt_per_hour ( 
    hour_of_day BIGINT,
    buy_cnt BIGINT
) WITH (
    'connector.type' = 'elasticsearch', -- use elasticsearch connector
    'connector.version' = '6',  -- elasticsearch verion，6 can support es 6+ and 7+ version
    'connector.hosts' = 'http://localhost:9200',  -- elasticsearch address
    'connector.index' = 'buy_cnt_per_hour',  -- elasticsearch index name 
    'connector.document-type' = 'user_behavior', -- elasticsearch type(deprecated in es7+)
    'connector.bulk-flush.max-actions' = '1',  -- refresh with each data
    'format.type' = 'json',  -- data output format json
    'update-mode' = 'append'
);
```

We don't have to create `buy_cnt_per_hour` index beforehand, Flink job will automatically create the index in es.

**Attention**

Please do not use `SELECT * FROM buy_cnt_per_hour`. Doing so means that you regard the es table as source table but not
sink table. And es table only supports as sink table.


### Submit the Query

We need to use `TUMBLE` window function to group the time as windows(For more information about group window function please refer https://ci.apache.org/projects/flink/flink-docs-release-1.10/dev/table/sql/queries.html#group-windows ). Then we count the number of 'buy' behavior within each window and insert into the es sink table `buy_cnt_per_hour`

```
INSERT INTO buy_cnt_per_hour
SELECT HOUR(TUMBLE_START(ts, INTERVAL '1' HOUR)), COUNT(*)
FROM user_behavior
WHERE behavior = 'buy'
GROUP BY TUMBLE(ts, INTERVAL '1' HOUR);
```

When the query is submitted, you can see a new Flink job is submitted in Flink Web UI.

![FLINK WEB UI](https://github.com/AlexanderChiuluvB/Pravega-FlinkSQL-Es-Demo/blob/master/pic/Screenshot%20from%202020-04-21%2017-09-25.png)


### Kibana for visualization

1.Create Index Pattern in kibana with the `buy_cnt_per_hour` es index.

![index pattern](https://github.com/AlexanderChiuluvB/Pravega-FlinkSQL-Es-Demo/blob/master/pic/Screenshot%20from%202020-04-21%2017-18-54.png)

2.Create `Area` type Dashboard called `User behavior log analysis` with the `buy_cnt_per_hour` index.
The specific configuration of dashboard can be seen in the following screenshot.

![visresult1](https://github.com/AlexanderChiuluvB/Pravega-FlinkSQL-Es-Demo/blob/master/pic/Screenshot%20from%202020-04-21%2017-22-44.png)


TBC.
