# Integrating PostgreSQL and ksqlDB

![1_lIe4iMnYB8HPkoCYbL7cwQ](https://user-images.githubusercontent.com/85284506/208126039-3f49ca5c-9167-495c-822c-9e30d45bbfc4.jpg)


## Component
+ Kafka and Zookeeper 
+ ksqlDB Server and CLI
+ Kafka Schema Registry
+ Kafka Connect
+ PostgreSQL

## Setting up Docker
```bash
docker build -t debezium-connect -f Dockerfile .
```

```bash
docker compose up -d
```


## PostgreSQL Data Sources

```bash
docker exec -it postgres psql -U postgres
```

### Create Database
```sql
CREATE DATABASE source;
```

```bash
\connect source;
```

### Create Table and Load Data

```sql
CREATE TABLE bank_marketing 
(id INTEGER, age INTEGER, job TEXT, marital TEXT, education TEXT, housing TEXT, loan TEXT, contact TEXT, month TEXT, day_of_week TEXT, duration INTEGER, subscribe TEXT);
```
```bash
\copy bank_marketing FROM 'bank-marketing.csv' DELIMITER ',' CSV HEADER
```

![postgresql table](https://user-images.githubusercontent.com/85284506/208126368-babf9326-f65f-4ab3-adc4-3a7ac15fcebd.jpg)


## PostgreSQL Source Connector

```bash
curl -X POST -H "Accept:application/json" -H "Content-Type: application/json" \
      --data @postgres-source.json http://localhost:8083/connectors
```

![topics](https://user-images.githubusercontent.com/85284506/208126565-42d2abc1-fea6-4566-b8fb-a10da757af03.jpg)

The connector will look for the PostgreSQL tables matching the table.whitelist in `json.file` and import it as a Kafka topic.

## ksqlDB

```docker
docker exec -it ksqldb-cli ksql http://ksqldb-server:8088
```

![ksql cli](https://user-images.githubusercontent.com/85284506/208126224-f74fda0a-f819-41c7-9ea7-c33fe336e754.jpg)

```bash
set 'commit.interval.ms'='2000';
set 'cache.max.bytes.buffering'='10000000';
set 'auto.offset.reset'='earliest';
```

### Create Stream
```sql
CREATE STREAM stream_table (id INTEGER, age INTEGER, job STRING, marital STRING, education STRING, housing STRING, loan STRING, contact STRING, month STRING, day_of_week STRING, duration INTEGER, subscribe STRING) \
WITH (KAFKA_TOPIC='dbserver1.public.bank_marketing', VALUE_FORMAT='AVRO')
```
![streams](https://user-images.githubusercontent.com/85284506/208126895-52eecf70-9cd4-49e3-ab65-9f1786be81eb.jpg)


## Create Table
```sql
CREATE TABLE FINAL_TABLE AS \
  SELECT id as id, \
         LATEST_BY_OFFSET(subscribe) as latest_subscribe \
  FROM stream_table \
  GROUP BY id;
```

![tables](https://user-images.githubusercontent.com/85284506/208126777-060f9fca-8425-4acb-8403-3af018e03a14.jpg)


## PostgreSQL Sink Connector

```bash
curl -X POST -H "Accept:application/json" -H "Content-Type: application/json" \
      --data @postgres-sink.json http://localhost:8083/connectors
```
