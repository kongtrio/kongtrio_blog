[TOC]

## 一、producer 相关命令  

### 1.  kafka-console-producer 生产消息

使用kafka-console-producer我们可以快速往某个topic推送消息。kafka-console-producer使用的也是KafkaProducer类进行消息的推送，因此KafkaProducer支持的参数kafka-console-producer都可以配置。

有关KafkaProducer的相关原理可以看我的这篇博客：

<https://blog.csdn.net/u013332124/article/details/81321942>

```shell
# 执行下面这条命令后会进入producer的交互界面，输入字符串就会将消息推送到kafka集群
kafka-console-producer --broker-list 127.0.0.1:9092 --topic test

# 推送10条消息 分别是1、2、3、...、10
seq 10 | kafka-console-producer --broker-list 127.0.0.1:9092 --topic yangjb

# 推送hello world 到kafka集群
echo "nihao world" | kafka-console-producer --broker-list 127.0.0.1:9092 --topic yangjb
```

### 2. 使用 kafka-producer-perf-test 进行producer的基准测试

我们要修改某个配置时，经常想知道这个配置的修改对kafka的性能会有哪些影响，这时候就可以来个基准测试来衡量配置修改对producer性能的影响。kafka官方就提供了这样一个工具，让我们很方面的对produer的性能进行测试。

下面是kafka-producer-perf-test支持的一些参数 

```shell
  --topic TOPIC          指定topic
  --num-records NUM-RECORDS 要推送多少条数据
  --payload-delimiter PAYLOAD-DELIMITER 当使用payload文件生成数据时，指定每条消息的之间的分割符，默认是换行符
  --throughput THROUGHPUT 推送消息时的吞吐量，单位是 messages/sec。必须指定
  --producer-props PROP-NAME=PROP-VALUE [PROP-NAME=PROP-VALUE ...] 指定producer的一些配置
  --producer.config CONFIG-FILE 直接指定配置文件
  --print-metrics       是否要在最后输出度量指标，默认是false
# 生成数据的方式有两种，一种是我们指定一个消息大小，该工具会随机生成一个指定大小的字符串推送，一个是我们指定一个文件，工具会从该文件中随机选取一条消息推送
# 下面两种方式只能选择一种
  --record-size RECORD-SIZE 指定每条消息的大小,大小是bytes
  --payload-file PAYLOAD-FILE 指定文件存放目录
```

下面我们测试使用producer一次推送100条数据

```shell
# 通过 --producer-props指定要连接的broker地址
# --num-records  指定一共要推送100条
#  --throughput 表示吞吐量，限制每秒20
# --record-size 表示每条消息的大小是20B
kafka-producer-perf-test --producer-props bootstrap.servers=127.0.0.1:9092 client.id=perftest --num-records 100 --throughput 10 --topic test --record-size 20
```

最后输出报告：

```shell
52 records sent, 10.4 records/sec (0.00 MB/sec), 5.2 ms avg latency, 137.0 max latency.
100 records sent, 9.993005 records/sec (0.00 MB/sec), 3.78 ms avg latency, 137.00 ms max latency, 2 ms 50th, 4 ms 95th, 137 ms 99th, 137 ms 99.9th.
```

我们可以编辑一个payload.txt，输入

```shell
hello
world
producer
perf
test
```

接着使用该payload.txt进行测试

```shell
# --payload-file 指定文件地址
kafka-producer-perf-test --producer-props bootstrap.servers=127.0.0.1:9092 client.id=perftest --payload-file payload.txt --num-records 100 --throughput 100 --topic test
```

该工具在执行时，会读取payload.txt的内容，然后根据`--payload-delimiter`将文本分成一条条消息，接着测试的时候会随机发送这些消息。

### 3. 使用 kafka-verifiable-producer 批量推送消息

kafka提供了kafka-verifiable-producer工具用于快速的推送一批消息到producer，并且可以打印出各条推送消息的元信息。推送的消息是从0开始不断往上递增。

支持参数

```shell
  --topic TOPIC          指定topic
  --broker-list HOST1:PORT1[,HOST2:PORT2[...]] 指定kafka broker地址
  --max-messages MAX-MESSAGES 一共要推送多少条，默认为-1，-1表示一直推送到进程关闭位置
  --throughput THROUGHPUT 推送消息时的吞吐量，单位messages/sec。默认为-1，表示没有限制
  --acks ACKS            每次推送消息的ack值，默认是-1
  --producer.config CONFIG_FILE 指定producer的配置文件
  --value-prefix VALUE-PREFIX 推送的消息默认是递增的数字，我们可以在这些消息前面加上指定的前缀。这个前缀好像也必须是数字
```

demo:

```shell
# --max-messages 10 总共推送10条
# 每秒推送2条
kafka-verifiable-producer --broker-list 127.0.0.1:9092 --topic test --max-messages 10 --throughput 2
```

输出:

```shell
{"timestamp":1544327879247,"name":"startup_complete"}
{"timestamp":1544327879413,"name":"producer_send_success","key":null,"value":"0","offset":91029,"partition":0,"topic":"test"}
{"timestamp":1544327879415,"name":"producer_send_success","key":null,"value":"1","offset":91030,"partition":0,"topic":"test"}
{"timestamp":1544327879904,"name":"producer_send_success","key":null,"value":"2","offset":91031,"partition":0,"topic":"test"}
{"timestamp":1544327880406,"name":"producer_send_success","key":null,"value":"3","offset":91032,"partition":0,"topic":"test"}
{"timestamp":1544327880913,"name":"producer_send_success","key":null,"value":"4","offset":91033,"partition":0,"topic":"test"}
{"timestamp":1544327881414,"name":"producer_send_success","key":null,"value":"5","offset":91034,"partition":0,"topic":"test"}
{"timestamp":1544327881918,"name":"producer_send_success","key":null,"value":"6","offset":91035,"partition":0,"topic":"test"}
{"timestamp":1544327882422,"name":"producer_send_success","key":null,"value":"7","offset":91036,"partition":0,"topic":"test"}
{"timestamp":1544327882924,"name":"producer_send_success","key":null,"value":"8","offset":91037,"partition":0,"topic":"test"}
{"timestamp":1544327883430,"name":"producer_send_success","key":null,"value":"9","offset":91038,"partition":0,"topic":"test"}
{"timestamp":1544327883942,"name":"shutdown_complete"}
{"timestamp":1544327883943,"name":"tool_data","sent":10,"acked":10,"target_throughput":2,"avg_throughput":2.1294718909710393}
```

### 4. 使用kafka-replay-log-producer进行topic之间的消息复制

使用kafka-replay-log-producer可以将一个topic的消息复制到另外一个topic上。它的流程是先从topic拉取消息，然后推送到另一个topic。

支持的参数：

```shell
--broker-list <String: hostname:port>  指定broker的地址
--inputtopic <String: input-topic>     要读取的topic名称
--messages <Integer: count>            要复制的消息数量，默认是-1，也就是全部
--outputtopic <String: output-topic>   要复制到哪个topic
--property <String: producer properties>     可以指定producer的一些参数                       
--reporting-interval <Integer: size>   汇报进度的频率，默认是5000汇报一次
--sync                                 是否开启同步模式
--threads <Integer: threads>           复制消息的线程数
--zookeeper <String: zookeeper url>    zk地址
```

demo：

```shell
# 把topic-test的消息复制给topic-aaaa
# --messages 表示只复制前50条
kafka-replay-log-producer --broker-list 127.0.0.1:9092 --zookeeper 127.0.0.1:2181/kafka --inputtopic test --outputtopic aaaa --messages 50
```

## 二、consumer相关命令

### 1. kafka-console-consumer 消费消息

使用kafka-console-consumer可以消费指定topic的消息。底层也是使用KafkaConsumer进行消费的。相关消费原理可以看我的这两篇博客：

[Consumer 加入&离开 Group详解（九）](https://blog.csdn.net/u013332124/article/details/83548706)

[Consumer 拉取日志流程详解（十）](https://blog.csdn.net/u013332124/article/details/83692537)

```shell
# 指定消费topic-test的消息
# --from-beginning 表示如果之前没有过消费记录，就从第一条开始消费
kafka-console-consumer --bootstrap-server 127.0.0.1:9092 --group hahaeh --topic test --from-beginning
```

kafka-console-consumer 还支持配置一些其他的参数，用户可以自行通过 —help 参数查看。

### 2. 使用 kafka-consumer-perf-test 进行consumer的基准测试

和producer一样，kafka也会consumer提供了一个命令来进行基准测试。

```shell
# --fetch-size 表示一次请求拉取多少条数据
# --messages 表示总共要拉取多少条数据
kafka-consumer-perf-test --broker-list 127.0.0.1:9092 --fetch-size 200 --group oka --topic test --messages 200
```

输出：

```shell
start.time, end.time, data.consumed.in.MB, MB.sec, data.consumed.in.nMsg, nMsg.sec, rebalance.time.ms, fetch.time.ms, fetch.MB.sec, fetch.nMsg.sec
2018-12-09 13:37:19:118, 2018-12-09 13:37:20:393, 0.0003, 0.0002, 162, 127.0588, 21, 1254, 0.0002, 129.1866
```

kafka-consumer-perf-test还支持其他参数：

```shell
--consumer.config <String: config file>   指定Consumer使用的配置文件
--date-format 定义输出的日志格式
--from-latest 如果之前没有消费记录，是否从之前消费过的地方开始消费
--num-fetch-threads 拉取消息的线程数
--threads 处理消息的线程数
--reporting-interval  多久输出一次执行过程信息
```

### 3. 使用kafka-verifiable-consumer批量拉取消息 

kafka-verifiable-consumer可以批量的拉取消息，其实和kafka-console-consumer命令差不多。不过使用kafka-verifiable-consumer消费消息输出的内容更丰富，还包括offset等信息，并且可以设置只读取几条消息等。kafka-console-consumer是有多少读多少。

```shell
# --max-messages 5 表示只拉取5条
# --verbose 表示输出每一条消息的内容
kafka-verifiable-consumer --broker-list 127.0.0.1:9092 --max-messages 5 --group-id hello --topic test --verbose
```

输出：

```shell
{"timestamp":1544335112709,"name":"startup_complete"}
{"timestamp":1544335112862,"name":"partitions_revoked","partitions":[]}
{"timestamp":1544335112883,"name":"partitions_assigned","partitions":[{"topic":"test","partition":0}]}
{"timestamp":1544335112919,"name":"record_data","key":null,"value":"90218","topic":"test","partition":0,"offset":90877}
{"timestamp":1544335112920,"name":"record_data","key":null,"value":"90219","topic":"test","partition":0,"offset":90878}
{"timestamp":1544335112920,"name":"record_data","key":null,"value":"0","topic":"test","partition":0,"offset":90879}
{"timestamp":1544335112921,"name":"record_data","key":null,"value":"1","topic":"test","partition":0,"offset":90880}
{"timestamp":1544335112921,"name":"record_data","key":null,"value":"2","topic":"test","partition":0,"offset":90881}
{"timestamp":1544335112921,"name":"records_consumed","count":162,"partitions":[{"topic":"test","partition":0,"count":5,"minOffset":90877,"maxOffset":90881}]}
{"timestamp":1544335112928,"name":"offsets_committed","offsets":[{"topic":"test","partition":0,"offset":90882}],"success":true}
{"timestamp":1544335112943,"name":"shutdown_complete"}
```

kafka-verifiable-consumer命令还支持以下参数：

```shell
--session-timeout consumer的超时时间
--enable-autocommit 是否开启自动offset提交，默认是false
--reset-policy  当以前没有消费记录时，选择要拉取offset的策略，可以是'earliest', 'latest','none'。默认是earliest
--assignment-strategy  consumer分配分区策略，默认是RoundRobinAssignor
--consumer.config 指定consumer的配置
```

### 4. 使用kafka-consumer-groups命令管理ConsumerGroup

#### ➀、列出所有的ConsumerGroup(新旧版本api区别) 

由于kafka consumer api有新版本和旧版本的区别，因此使用kafka-consumer-groups进行group的管理时，内部使用的机制也不一样。如果我们使用`—zookeeper`来连接集群，则使用的是旧版本的consumer group管理规则，也就是ConsumerGroup的一些元数据是存储在zk上的。如果使用`--bootstrap-server`来连接，则是面向新版本的consumer group规则。

列出使用旧版本的所有consumer group

```shell
kafka-consumer-groups --zookeeper 127.0.0.1:2181/kafka --list
```

列出新版本的所有consumer group

```shell
kafka-consumer-groups --bootstrap-server 127.0.0.1:9092  --list
```

#### ➁、删除ConsumerGroup

删除指定的group

```shell
# 删除 helo和hahah 这两个group
kafka-consumer-groups --bootstrap-server 127.0.0.1:9092  --delete --group helo --group hahah
```

删除指定group的指定topic的消费记录(**topic级别的删除仅在旧版本api中支持**)

```shell
# 旧版本api 必须指定zk地址
kafka-consumer-groups --zookeeper 127.0.0.1:2181/kafka  --delete --group helo --topic test
```

删除指定topic在所有group中的消费记录(**topic级别的删除仅在旧版本api中支持**)

```shell
# 旧版本api 必须指定zk地址
kafka-consumer-groups --zookeeper 127.0.0.1:2181/kafka  --delete --topic test
```

#### ➂、列出ConsumerGroup详情

通过—describe可以从不同维度观察group的信息。

##### 查看group的offset消费记录

```shell
# --offset是--describe的默认选项，可以不传
kafka-consumer-groups --bootstrap-server 127.0.0.1:9092 --describe --group mytest --offset
```

输出

```shell
TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID                                     HOST            CLIENT-ID
test            0          796670          796676          6               consumer-1-00ab2315-e3f3-4261-8392-f9fae4668f87 /172.20.16.13   consumer-1
```

##### 查看group的member信息

```shell
# --members 表示输出member信息
kafka-consumer-groups --bootstrap-server 127.0.0.1:9092 --describe --group mytest --members
```

输出

```shell
CONSUMER-ID                                     HOST            CLIENT-ID       #PARTITIONS
consumer-1-00ab2315-e3f3-4261-8392-f9fae4668f87 /172.20.16.13   consumer-1      1
```

##### 查看group的状态

```shell
kafka-consumer-groups --bootstrap-server 127.0.0.1:9092 --describe --group mytest --state
```

输出：

```shell
COORDINATOR (ID)          ASSIGNMENT-STRATEGY       STATE                #MEMBERS
172.20.16.13:9092 (0)     range                     Stable               1
```

#### ➃、重置group的消费记录  

**当选择重置消费记录操作时，目标Group的状态一定不能是活跃的**。也就是该group中不能有consumer在消费。

通过 `--reset-offsets` 可以重置指定group的消费记录。和`--reset-offsets`搭配的有两个选项，`--dry-run`和`--execute`，默认是`--dry-run`。

##### dry-run 模式 

当运行在`--dry-run`模式下，重置操作不会真正的执行，只会预演重置offset的结果。该模式也是为了让用户谨慎的操作，否则直接重置消费记录会造成各个consumer消息读取的异常。

```shell
# --shift-by -1 表示将消费的offset重置成当前消费的offset-1
kafka-consumer-groups --bootstrap-server 127.0.0.1:9092 --reset-offsets --shift-by -1 --topic test --group mytest --dry-run
```

输出

```shell
TOPIC                          PARTITION  NEW-OFFSET
test                           0          797054
```

此时如果去查询该group的消费offset，会发现该group的消费offset其实还是797055，并没有发生改变。

##### —execute 模式  

通过`--execute`参数可以直接执行重置操作。

```shell
kafka-consumer-groups --bootstrap-server 127.0.0.1:9092 --reset-offsets --shift-by -1 --topic test --group mytest --execute
```

##### 重置offset的几种策略  

该命令提供了多种offset重置策略给我们选择如何重置offset

```shell
--to-current 直接重置offset到当前的offset，也就是LOE
--to-datetime <String: datetime>  重置offset到指定时间的offset处
--to-earliest  重置offset到最开始的那条offset
--to-offset <Long: offset> 重置offset到目标的offset
--shift-by <Long:n> 根据当前的offset进行重置，n可以是正负数
--from-file <String: path to CSV file> 通过外部的csv文件描述来进行重置
```

Demo:

```shell
# 将group mytest的test 这个topic的消费offset重置到666
# 注意如果topic分区中的最小offset比666还大的话，就会直接使用该最小offset作为消费offset
kafka-consumer-groups --bootstrap-server 127.0.0.1:9092 --reset-offsets --topic test --group mytest --execute --to-offset 666
# 重置到最早的那条offset
kafka-consumer-groups --bootstrap-server 127.0.0.1:9092 --reset-offsets --topic test --group mytest --execute --to-earliest
```

我们再看下如何使用`--from-file`来重置offset。首先先编辑一个文件 reset.csv

```shell
test,0,796000
```

3列分别是topicName,partition,offset。最后输入重置命令

```shell
kafka-consumer-groups --bootstrap-server 127.0.0.1:9092 --reset-offsets --group mytest --execute --from-file reset.csv
```

## 三、replica数据一致性校验

通过kafka-replica-verification命令可以检查指定topic的各个partition的replic的数据是否一致。

```shell
kafka-replica-verification --broker-list 127.0.0.1:9092
```

默认是检查全部topic，可以通过指定`topic-white-list`来指定只检查一些topic。

replica一致性检查主要是根据partition的HW来检查的，大概原理是在所有的broker都开启一个fetcher，然后拉取数据做检查各个replica的数据是否一致。因此，进行该检查时要保证所有的broker都在线，否则该工具会一直阻塞直到broker全部启动。