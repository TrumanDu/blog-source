---
title: Kafka cluster 在产线部署最佳实践
tags: []
categories: []
toc: false
date: 2020-09-29 19:20:35
---

## 概要

记录线上Zookeeper集群和Kafka集群部署过程，操作系统配置，以及一些参数的设置。给大家部署提供一些宝贵意见和参考。部署一个集群，按照官方社区的文档，很容易就搭建一个集群，但是为了更好的发挥集群的性能，有很多设置是可以避免产生不必要的问题，都是在惨痛的教训中产生的经验。

本文内容来自**Newegg**及 **Confluent** 产线上Kafka cluster运维经验，仅供参考。

Newegg 产线Kafka版本选择Confluent发行版本,版本对照表：

| Confluent Platform | Apache Kafka |
| :----------------- | :----------- |
| 5.5.x              | 2.5.x        |

Docker镜像列表：

| Service | Info |
| :----------------- | :----------- |
|Zookeeper|confluentinc/cp-zookeeper:5.5.1|
|Kafka| confluentinc/cp-kafka:5.5.1|

## Zookeeper Cluster

### 硬件

内存至少4GB，Zookeeper对swap敏感，应当避免swap。

### 集群配置

zookeeper节点数据应该为2n+1,n大于0，保持集群中超过一半节点存活。

Confluent docker 镜像参数设置是在ENV(环境变量)中设置，以ZOOKEEPER开头，以`_`替换`.`

#### 参数清单

```

- ZOOKEEPER_SERVER_ID=1
- ZOOKEEPER_TICK_TIME=2000
- ZOOKEEPER_CLIENT_PORT=2181
- ZOOKEEPER_INIT_LIMIT=10
- ZOOKEEPER_SYNC_LIMIT=5
- ZOOKEEPER_SERVERS=192.168.0.1:2888:3888;192.168.0.2:2888:3888;192.168.0.3:2888:3888
- ZOOKEEPER_AUTOPURGE_SNAP_RETAIN_COUNT=5
- ZOOKEEPER_AUTOPURGE_PURGE_INTERVAL=10
- KAFKA_HEAP_OPTS=-Xmx4G -Xms4G
```

ZOOKEEPER_AUTOPURGE_SNAP_RETAIN_COUNT控制快照的数量，ZOOKEEPER_AUTOPURGE_PURGE_INTERVAL控制清除快照的时间间隔，默认不清除，这个很重要。

### 文件映射

| Host volume                  | Container volume    |
| ---------------------------- | ------------------- |
| /opt/app/cp-zookeeper-5.5.1/data |/var/lib/zookeeper/data |
| /opt/app/cp-zookeeper-5.5.1/log  | /var/lib/zookeeper/log      |
| /etc/localtime               | /etc/localtime      |

## Kafka Cluster

### 硬件

CPU推荐24核+，内存推荐64GB,Kafka对堆内内存使用很谨慎，不需要将它堆内存设置超过6GB，剩余更多的内存留给系统和Page Cache，有助于提交读写效率。

### 系统参数

#### File Descriptors
众所周知，Kafka严重依赖磁盘，越多的partition需要创建越多的文件。文件描述符限制了系统打开文件句柄数，避免系统在今后的运行中出现`Too many open files`
```
ulimit -n 1000000
```

临时性修改，系统重启后会失效

为了重启后同样生效，可以修改`/etc/security/limits.conf`,重新登录session

```
* soft nofile 1000000
* hard nofile 1000000
```

#### mmap
`vm.max_map_count`会限制一个进程可以拥有的VMA(虚拟内存区域)的数量，这个值应该远远大于index文件的数量（broker segment growth）。关于index文件的统计可以使用：`find . -name '*index' | wc -l`。

临时性生效，重启后失效：

```
sysctl -w vm.max_map_count=262144
```

设置机器重启后仍然生效
```
echo 'vm.max_map_count=262144' >> /etc/sysctl.conf
sysctl -p
```

### JVM

推荐的GC（在JDK 1.8 u5测试）配置如下：

```
-Xms6g -Xmx6g -XX:MetaspaceSize=96m -XX:+UseG1GC -XX:MaxGCPauseMillis=20
       -XX:InitiatingHeapOccupancyPercent=35 -XX:G1HeapRegionSize=16M
       -XX:MinMetaspaceFreeRatio=50 -XX:MaxMetaspaceFreeRatio=80
```

按以上配置，如下是领英内部最繁忙集群的状态 (at peak):

- 60 brokers
- 50k partitions (replication factor 2)
- 800k messages/sec in
- 300 MBps inbound, 1 GBps + outbound

所有broker 90%的GC暂停时间大约21ms，每秒执行不到一次young GC

### 集群配置

Confluent docker 镜像参数设置是在ENV(环境变量)中设置，以KAFKA开头，以`_`替换`.`

#### 参数清单

```
- KAFKA_BROKER_ID=0
- KAFKA_ZOOKEEPER_CONNECT=192.168.0.1:2181,192.168.0.2:2181,192.168.0.3:2181
- KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://192.168.0.1:9092
- KAFKA_AUTO_CREATE_TOPICS_ENABLE=false
- KAFKA_DELETE_TOPIC_ENABLE=true
- KAFKA_SOCKET_RECEIVE_BUFFER_BYTES=102400
- KAFKA_JMX_PORT=8888
- KAFKA_JMX_HOSTNAME=192.168.0.1
- KAFKA_HEAP_OPTS=-Xms6g -Xmx6g -XX:MetaspaceSize=96m -XX:+UseG1GC -XX:MaxGCPauseMillis=20 -XX:InitiatingHeapOccupancyPercent=35 -XX:G1HeapRegionSize=16M -XX:MinMetaspaceFreeRatio=50 -XX:MaxMetaspaceFreeRatio=80
-KAFKA_LOG4J_LOGGERS=kafka.controller=INFO,state.change.logger=INFO,kafka.producer.async.DefaultEventHandler=INFO
- KAFKA_MESSAGE_MAX_BYTES=4000000
- KAFKA_REPLICA_FETCH_MAX_BYTES=4100000
- KAFKA_CONFLUENT_SUPPORT_METRICS_ENABLE=false
```

KAFKA_SOCKET_RECEIVE_BUFFER_BYTES和KAFKA_MESSAGE_MAX_BYTES，KAFKA_REPLICA_FETCH_MAX_BYTES非必须，需要根据真实消息的限制进行修改。但是需要谨记的是修改MESSAGE_MAX_BYTES切记修改REPLICA_FETCH_MAX_BYTES。

KAFKA_CONFLUENT_SUPPORT_METRICS_ENABLE只有在使用Confluent  docker images的时候需要设置。避免产生一些收集指标的错误log。

### 文件映射

| Host volume                  | Container volume    |
| ---------------------------- | ------------------- |
| /opt/app/cp-kafka-5.5.1/data | /var/lib/kafka/data |
| /opt/app/cp-kafka-5.5.1/log  | /var/log/kafka      |
| /etc/localtime               | /etc/localtime      |



## 参考

1. [Running ZooKeeper in Production](https://docs.confluent.io/current/zookeeper/deployment.html#running-zk-in-production)
2. [Running Kafka in Production](https://docs.confluent.io/current/kafka/deployment.html#running-ak-in-production)