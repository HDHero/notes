# kafka文档 

## 配置server.properties，

> broker节点全局唯一
>
> log.dirs = /tmp/kafka-logs

1.启动命令

```shell
/bin/kafka-server-start.sh -daemon config/server.properties
```

2.查看所有主题配置

```shell
/bin/kafka-topics.sh --list --zookeeper xxxx:2181
```

3.创建主题

分区、副本数

```shell
/bin/kafka-topics.sh --create --zookeeper xxxx:2181 --topic first --partitions 2 --replication-factor 2
```

4.删除主题

```shell
/bin/kafka-topics.sh --delete --zookeeper xxxx:2181 --topic first
```

5.查看主题详情

```shell
/bin/kafka-topics.sh --describe --topic first --zookeeper xxxx:2181
```



## 命令行控制生产者和消费者

1.生产者启动

```shell

bin/kafka-console-producer.sh --topic first --broker-list xxx:9092
```

2.消费者启动

```shell
2.2 版本以前
bin/kafka-console-consumer.sh --topic first --zookeeper xxx:2181

bin/kafka-console-consumer.sh --topic first --zookeeper xxx:2181 --from-beginning

2.2 版本后
./bin/kafka-console-consumer.sh --bootstrap-server xxx:9092 --from-beginning --topic second
```

## 日志文件

/home/xx/tmp/xxx

consumer_offersets-32 32分区

