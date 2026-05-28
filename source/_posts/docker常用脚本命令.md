---
title: docker常用脚本命令
date: 2024-11-14 11:48:11
categories:
 - docker
tags: 
 - docker
---

#### docker常用脚本
###### 运行redis
```
docker run -d \
  -p 6380:6379 \
  --name myredis \
  -v /path/to/your/redis.conf:/usr/local/etc/redis/redis.conf \
  -e REDIS_PASSWORD=yourpassword \
  redis redis-server /usr/local/etc/redis/redis.conf
```
###### 运行rocketmq
```
docker run -d \
  --name myrabbitmq \
  -p 5672:5672 \
  -p 15672:15672 \
  -v /path/to/your/rabbitmq.conf:/etc/rabbitmq/rabbitmq.conf \
  -e RABBITMQ_DEFAULT_USER=guest \
  -e RABBITMQ_DEFAULT_PASS=yourpassword \
  -e RABBITMQ_DEFAULT_VHOST=/ \
  rabbitmq:3-management
```
###### 运行mysql
```
docker run -d \
  --name mymysql \
  -e MYSQL_ROOT_PASSWORD=mysecretpassword \
  -e MYSQL_DATABASE=mydatabase \
  -e MYSQL_USER=myuser \
  -e MYSQL_PASSWORD=mypassword \
  -p 3306:3306 \
  -v /path/to/your/my.cnf:/etc/mysql/my.cnf \
  -v /path/to/your/initdb.sql:/docker-entrypoint-initdb.d/initdb.sql \
  mysql:latest
```
###### 运行kafka
```
version: '3'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    ports:
      - "2181:2181"
    networks:
      - kafka-network

  kafka:
    image: confluentinc/cp-kafka:latest
    depends_on:
      - zookeeper
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    ports:
      - "9092:9092"
    networks:
      - kafka-network

networks:
  kafka-network:
    driver: bridge
```

##### mysql导出数据
```
mysqldump -h rm-wz9gtw0898hi535otyo.mysql.rds.aliyuncs.com -P 3306 -u user -p -t --skip-lock-tables --no-tablespaces --set-gtid-purged=OFF datanase > D:\databaseg.sql
```
