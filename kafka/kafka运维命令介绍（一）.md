[TOC]

## 一、连接zk

由于kafka的各种元数据都存储在zk，要连接kafka集群也要通过zk获取各个broker的ip端口然后连接broker。因此，大多数kafka自带的运维命令都要指定zk的地址，比如用kafka-topics列出所有topics：

```shell
kafka-topics --zookeeper localhost:2181/kafka --list
```

`--zookeeper`参数是必须指定的。另外，如果kafka集群启动的时候在配置文件中指定了namespace，记得要在zk的地址后面也要加上kafka所属的namespace。否则kafka就找不到kafka集群的相关元数据了。

由于kafka的元数据都存储在zk，因此掌握好如何查看zk的数据也是运维kafka集群的一个关键。

### zkCli 命令  

zookeeper安装包一般都会提供zkCli命令来让用户连接zookeeper集群。

```shell
./zkCli.sh -timeout 5000 -r -server ip:port
```

之后进入zk的交互界面，就可以输出相关命令查看zk的数据了。

```shell
# 查看kafka集群下所有的brokers id列表
ls /brokers/ids
# 查看 1003 broker的信息
get /brokers/ids/1003
```

其他zk交互界面的命令这里不多做介绍，zkCli的帮助文档已经写的很清楚了。

## 二、topic 相关  

kafka-topics可以进行和topics相关的一些操作。下面介绍一下如何运用该命令来操作kafka topics。

该命令最终是调用kafka源码中的TopicCommand类来实现的。

### 列出所有的topic & 获取命令帮助

```shell
# 列出帮助文档，英文好的同学基本看帮助文档就可以指定大概怎么使用该命令了
kafka-topics --help
# 列出kafka集群下的所有topics，这里需要指定kafka机器元数据存储所在的zk机器地址，记得如果有namespace，要也加上，否则将连不上kafka集群
 kafka-topics --zookeeper localhost:2181/kafka --list
```

### 创建topic

```shell
# 创建一个topic为test的topic，并指定分区数为5，副本数为1。这里的副本数不能超过broker的数量，否则会报错
kafka-topics --topic test --zookeeper localhost:2181/kafka --create --replication-factor 1 --partitions 5
# 创建时指定副本在哪个broker上,多个partition之间用逗号分隔，副本之间用":"分割，第一个副本默认是leader
kafka-topics.sh --zookeeper 172.19.0.5:2181 --topic lyt2 --create --replica-assignment 1001:1002,1001:1002,1001:1002
```

### 列出所有topic的详情

通过 ` --describe` 参数可以列出我们指定的topics详情，包括 partitions、leader、replicas、isr等。

```shell
kafka-topics --zookeeper localhost:2181/kafka --describe test test_yangjb
# 输出
Topic:test	PartitionCount:5	ReplicationFactor:3	Configs:
	Topic: test	Partition: 0	Leader: 1001	Replicas: 1001,1002,1003	Isr: 1002,1001,1003
	Topic: test	Partition: 1	Leader: 1002	Replicas: 1002,1003,1001	Isr: 1002,1003,1001
	Topic: test	Partition: 2	Leader: 1003	Replicas: 1003,1001,1002	Isr: 1002,1001,1003
	Topic: test	Partition: 3	Leader: 1001	Replicas: 1001,1003,1002	Isr: 1002,1001,1003
	Topic: test	Partition: 4	Leader: 1002	Replicas: 1002,1001,1003	Isr: 1002,1001,1003
```

下面是一些 使用`—describe`时可以使用的其他参数

```shell
# 只列出修改了默认配置的那些topic。并可以查看修改了哪些topic配置
--topics-with-overrides
# 列出那些目前没有leader的topic
--under-replicated-partitions
# 列出那些正在同步的topic或者同步出现异常的topic
--under-replicated-partitions
```

### 删除topic

注意，kafka删除topic是异步的，因此并不是命令返回了topic就已经被成功删除。而是等待后台的删除任务执行成功才真正删除该topic。

```shell
kafka-topics --zookeeper localhost:2181/kafka --delete --topic yangjb_test
```

### 修改topic相关信息

通过 `--alter` 参数可以修改topic的信息，能修改的信息包括 partition数量、replica分配情况、topic配置。如果要修改 partition数量时，修改的后的数量一定要比当前的数量大，否则会报错。

```shell
# 将partition数量修改成7个
kafka-topics --zookeeper localhost:2181/kafka --topic test --alter --partitions 7
# 通过 --replica-assignment 参数指定新增partition的副本分布情况
# 如果原先的partition数量是3，那么新增的一个分区的副本分布应该在1002和1003
kafka-topics --zookeeper localhost:2181/kafka --topic test -alter --partitions 4 --replica-assignment 1001:1002,1001:1002,1001:1002,1002:1003
# 修改topic test的配置 flush.ms =30000 。
kafka-topics --zookeeper localhost:2181/kafka  --topic test --alter --config flush.ms=30000
# 删除topic test的 flush.ms 配置
kafka-topics --zookeeper localhost:2181/kafka  --topic test --alter --delete-config flush.ms
```

**注意，在后续的kafka版本中，关于topic的配置的修改删除可能会被移到kafka-configs.sh中**。官方建议使用kafka-configs来修改topic的配置。

## 三、分区副本重分配  

在数据量大的情况下，各个broker上的数据量经常会不一致，有的broker上数据非常大，有的则很小，为了让数据更均匀的分布在各个broker，我们就要学会对topic的partion进行分区副本重分配。

首先建立一个json文件，用来描述如何分配分区副本。

`assign.json`：

```shell
{
  "partitions": [
    {
      "topic": "test",
      "partition": 1,
      "replicas": [
        1002,
        1003
      ]
    },
    {
      "topic": "test",
      "partition": 2,
      "replicas": [
        1003,
        1002
      ]
    }
  ],
  "version": 1
}
```

文件中只要指定要重新分配副本的分区号就可以，不需要列出所有分区。

### 提交分区副本重分配任务：

```shell
# --execute 参数表示执行
kafka-reassign-partitions --zookeeper localhost:2181/kafka --reassignment-json-file assign.json --execute
# --verify 参数表示查看分区副本重分配任务的执行状态
kafka-reassign-partitions --zookeeper localhost:2181/kafka --reassignment-json-file assign.json --verify
```

### 让系统自动帮我们生成重分配json文件：

执行命令之前需要建立一个json文件，告诉系统要重分配哪些分区:

`gen.json`:

```shell
{
  "topics": [
    {
      "topic": "foo"
    }
  ],
  "version": 1
}
```

接着执行命令

```shell
# --generate 表示生成重分配的json文件
# --topics-to-move-json-file 指定要重分配哪些topic
# --broker-list 表示要分配到哪些broker上去
kafka-reassign-partitions --zookeeper localhost:2181/kafka --generate --topics-to-move-json-file gen.json --broker-list 1001,1002,1003
```

### 其他参数

**以下参数低版本的kafka并不支持，要使用之前请先确定你使用的版本支持这些特性**

```shell
# 指定重分配时，在一个broker上，各个日志目录之间复制数据的阈值，最低要求 1 KB/s
# 如果重分配任务正在进行，第二次执行会修改原来设置的阈值
--replica-alter-log-dirs
# 指定重分配时，在不同broker之间传输数据的阈值，最低要求 1 KB/s
# 如果重分配任务正在进行，第二次执行会修改原来设置的阈值
--throttle
# 等待重分配任务开始的超时时间
--timeout
```

### 分区副本重分配过程

详情可以看kafka源码的`KafkaController#onPartitionReassignment()`的方法注解。

RAR = Reassigned replicas，目标要分配的副本情况
OAR = Original list of replicas for partition，原先的副本分配情况
AR = current assigned replicas，当前的副本分配情况

1. 更新zk处的partition副本配置：AR=RAR+OAR
2. 向所有RAR+OAR的副本发送元数据更新请求
3. 将新增的那部分的副本状态设置为NewReplica。也就是 RAR-OAR 那部分副本
4. 等待所有的副本和leader保持同步。也就是抱着RAR+OAR的副本都在isr中了
5. 将所有在RAR中的副本状态都设置为OnlineReplica
6. 在内存中先将AR=RAR
7. 如果leader不在RAR中，就需要重新竞选leader。采用ReassignedPartitionLeaderSelector选举
8. 将所有准备移除的副本状态设置为OfflineReplica。也就是OAR-RAR的那部分副本。这时partition的isr会收缩
9. 将所有准备移除的副本状态设置为NonExistentReplica。这时所在的分区副本数据会被删除。
10. 将内存中的AR更新到zk
11. 更新zk的/admin/reassign_partitions路径，移除这个partition
12. 发送新的元数据到各个broker上

假设当前有OAR = {1, 2, 3}， RAR = {4,5,6}，在进行partition reaassigned的过程中会发生如下变化

| AR            | leader/isr      | 步骤     |
| ------------- | --------------- | -------- |
| {1,2,3}       | 1/{1,2,3}       | 初始状态 |
| {1,2,3,4,5,6} | 1/{1,2,3,4,5,6} | 步骤2    |
| {1,2,3,4,5,6} | 1/{1,2,3,4,5,6} | 步骤4    |
| {1,2,3,4,5,6} | 4/{1,2,3,4,5,6} | 步骤7    |
| {1,2,3,4,5,6} | 4/{1,2,3,4,5,6} | 步骤8    |
| {4,5,6}       | 4/{4,5,6}       | 步骤10   |

## 四、删除某个partition的数据  

使用kafka-delete-records命令可以删除指定topic-partition在指定offset之前的所有数据。

该命令是kafka在0.11版本之后才支持的。

首先需要编写删除offset描述json文件：

delete.json

```shell
{
  "partitions": [
    {
      "topic": "test",
      "partition": 0,
      "offset": 24
    }
  ],
  "version": 1
}
```

上面的json文件表示删除topic是test的0号parition的24之前的所有offset，也就是1-23这些offset的数据都会被删除掉。

```shell
kafka-delete-records --bootstrap-server 127.0.0.1:9092 --offset-json-file delete.json
```

## 五、全局&topic配置修改  

通过`kafka-configs`命令，我们可以修改broker的配置，以及topic的配置、client和user的配置。

### 配置更新原理  

kafka-configs命令修改配置后会被写到对应的zookeeper的节点上持久化，之后kafka集群重启后还会加载这些配置，并覆盖配置文件的那些配置。也就是说，如果在此处设置了某个配置项，之后在配置文件中对这个配置项的改动都不会起作用，因为被覆盖了。

用该命令修改了配置后，可以在zk的节点下看到对应的配置内容。

节点目录一般是 `/config/entityType/entityName`,entityType可以是brokers、topics、users、clients。entityName表示具体的名称，比如broker的id，topic的名称等。

比如要看0号broker修改过的配置项，可以在zk交互界面中输入

```shell
# /kafka 是命名空间
get /kafka/config/brokers/0
```

### 修改broker配置  

```shell
# 将0号broker的配置 log.cleaner.backoff.ms修改成1000,flush.ms 也修改成1000
# --alter 表示要修改配置项
# --add-config 后面跟着要修改的配置项
kafka-configs --bootstrap-server 127.0.0.1:9092 --entity-type brokers --entity-name 0 --add-config log.cleaner.backoff.ms=1000,flush.ms=1000 --alter
# 删除0号broker 对 log.cleaner.backoff.ms的配置
kafka-configs --bootstrap-server 127.0.0.1:9092 --entity-type brokers --entity-name 0 --delete-config log.cleaner.backoff.ms --alter
# 列出0号broker修改过的配置项
kafka-configs --bootstrap-server 127.0.0.1:9092 --entity-type brokers --entity-name 0 --describe
```

### 修改topic的配置  

```shell
# 将test这个topic的 delete.retention.ms修改成1000,flush.ms 也修改成1000
kafka-configs --zookeeper 127.0.0.1:2181/kafka --entity-type topics --entity-name test --add-config delete.retention.ms=1000,flush.ms=1000 --alter
# 删除test这个topic的 delete.retention.ms和flush.ms配置项
kafka-configs --zookeeper 127.0.0.1:2181/kafka --entity-type topics --entity-name test --delete-config delete.retention.ms,flush.ms --alter
# 列出 test这个topic修改过的配置项
kafka-configs --zookeeper 127.0.0.1:2181/kafka --entity-type topics --entity-name test --describe
```

### 修改client的配置  

这里的client是指客户端，也就是produer或者consumer。客户端支持修改的配置有

```shell
# 请求限制
request_percentage
# 推送消息时的流量控制
producer_byte_rate
# 消费时的流量控制
consumer_byte_rate
```

通过指定clientId我们可以控制指定客户端的配置，从而控制他们的流量不会超过我们设定的值

```shell
# 设置 客户端id 为test的 producer_byte_rate和consumer_byte_rate为1024
kafka-configs --zookeeper 127.0.0.1:2181/kafka --alter --add-config 'producer_byte_rate=1024,consumer_byte_rate=1024' --entity-type clients --entity-name test
```

## 六、查看broker上磁盘的使用情况

**在0.11版本中**，新增一个命令`kafka-log-dirs`可以查看broker的磁盘使用情况。

该命令可以从两个维度观察磁盘的使用情况，一个是指定broker id，查看该broker的数据目录的各个topic parition的占用大小。还可以直接指定topic，查看这些topic的partition在各个broker上的使用情况。甚至可以两个过滤条件一起用，同时指定brokerId和topic。

```shell
# 查看0、1号broker上各个topic partition的磁盘使用情况
kafka-log-dirs --bootstrap-server 127.0.0.1:9092 --broker-list 0,1 --describe
# 查看topic:test 在各个broker上的磁盘使用情况
kafka-log-dirs --bootstrap-server 127.0.0.1:9092 --topic-list test --describe
# 查看topic test 在0号broker上的磁盘使用情况
kafka-log-dirs --bootstrap-server 127.0.0.1:9092 --topic-list test --broker-list 0 --describe
```

输出示例：

```shell
{
  "version": 1,
  "brokers": [
    {
      "broker": 1001,
      "logDirs": [
        {
          "logDir": "/kafka/kafka-logs-7da01186c90a",
          "error": null,
          "partitions": [
            {
              "partition": "test-4",
              "size": 0,
              "offsetLag": 0,
              "isFuture": false
            },
            {
              "partition": "test-0",
              "size": 0,
              "offsetLag": 0,
              "isFuture": false
            }
          ]
        }
      ]
    }
  ]
}
```

## 七、使用kafka-preferred-replica-election进行leader选举  

当我们查看某个topic partition时，会输出该partiton replica的列表，其中replica列表的第一个replica被kafka称为preferred replica。

```shell
Topic: test	Partition: 0	Leader: 1002	Replicas: 1001,1002,1003	Isr: 1002,1001,1003
```

上面的test partition-0中，1001就是那个preferred replica。在大多情况下，preferred replica一般就是leader，但是有些情况可能不是。因此，kafka提供了kafka-preferred-replica-election来将preferred replica选举成leader。

首先我们需要编辑prefered.json 文件：

```shell
{
  "partitions": [
    {
      "topic": "test",
      "partition": 0
    }
  ]
}
```

该文件告诉工具我们要对topic test的partition-0进行preferred选举:

```shell
kafka-preferred-replica-election --zookeeper 127.0.0.1:2181 --path-to-json-file prefered.json
```

