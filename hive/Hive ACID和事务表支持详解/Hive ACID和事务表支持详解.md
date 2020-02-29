[TOC]

## 一、ACID介绍 

ACID就是常见数据库事务的四大特性：Atomicity(原子性)、Consistency（一致性）、Isolation（隔离性）、Durability（持久性）。

在Hive 0.13之前，Hive支持**分区级别**上原子性、一致性、持久性，隔离性可以通过hive提供的锁机制来实现（通过zookeeper锁或者内存锁来**锁住一个分区的数据**）。**从Hive 0.13开始，Hive可以支持行级别上面的ACID语义了**。因此我们可以在有其他程序读取一个分区数据时往这个分区插入新的数据。

## 二、使用限制 

1. 不支持 BEGIN、COMMIT、ROLLBACK 等语句，所有的语句都是自动提交
2. 仅支持ORC格式
3. 事务的支持默认是关闭的，需要配置相关参数打开
4. 表需要配置分桶，外部表不能设置成事务表，因为外部表的文件存储格式不在hive的管理之中。（因为Hive事务的实现主要依赖于表分桶的存储格式，如果表没分桶，那么表底下的文件就会很散乱，hive的事务机制无法有效的读取）
5. 非 ACID 的会话不能读写ACID表，也就是说，需要在会话中手动set参数开启hive事务管理支持后才可以操作ACID表
6. 目前仅支持快照隔离级别，不支持脏读、读已提交、可重复读、串行等隔离级别
7. 现有的zk和内存锁和事务不兼容
8. 使用oracle作为metastore数据库，以及设置了"datanucleus.connectionPoolingType=BONECP"的话，会导致一些间断性的"No such lock.." 和 "No such transaction..."错误，这种情况建议将配置改成"datanucleus.connectionPoolingType=DBCP"
9. 事务表不支持 [LOAD DATA...](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+DML#LanguageManualDML-Loadingfilesintotables)  语句（在2.4.0 之前并没有被禁止，语句还是可以执行的）,主要还是由于LOAD DATA的方式加载数据会导致表中的数据文件格式乱掉，其实分桶表理论上也是不允许load data语句加载数据的。

## 三、支持的一些新的语法

1. *INSERT...VALUES 语句*
2. *UPDATE 语句*
3. *DELETE 语句*
4. *SHOW TRANSACTIONS 语句，用于展示目前正在运行的所有事务*
5. *SHOW COMPACTIONS 语句，用于展示目前正在运行的所有压缩任务*

## 四、主要设计和实现 

HDFS本身是不支持直接修改文件的,也不能保证有人追加内容时的读一致性。因此，为了支持ACID的特性，Hive只能使用其他数据仓库常用的方法，也就是增量的形式记录更新和删除（也称做读时更新）。

存储在事务表中的数据会被分成两种类型的文件：

1. base文件，用来存放平常的数据
2. delta文件，用来存储新增、更新、删除的数据。**每一个事务处理数据的结果都会单独新建一个delta文件夹用来存储数据**。

（会有定时任务定期的将delta文件合并成base文件，后面会详细介绍）

在有用户要读取这个表的数据时，就会将base文件和delta文件都读取到内存，然后进行合并（就是判断哪些记录有被修改，哪些记录被删除等）。

### base和delta文件夹的基本结构

假设有一张表名为t，分桶数量只有2的表，那它表的数据结构可能如下：

```shell
hive> dfs -ls -R /user/hive/warehouse/t;
drwxr-xr-x   - ekoifman staff          0 2016-06-09 17:03 /user/hive/warehouse/t/base_0000022
-rw-r--r--   1 ekoifman staff        602 2016-06-09 17:03 /user/hive/warehouse/t/base_0000022/bucket_00000
-rw-r--r--   1 ekoifman staff        602 2016-06-09 17:03 /user/hive/warehouse/t/base_0000022/bucket_00001
drwxr-xr-x   - ekoifman staff          0 2016-06-09 17:06 /user/hive/warehouse/t/delta_0000023_0000023_0000
-rw-r--r--   1 ekoifman staff        611 2016-06-09 17:06 /user/hive/warehouse/t/delta_0000023_0000023_0000/bucket_00000
-rw-r--r--   1 ekoifman staff        611 2016-06-09 17:06 /user/hive/warehouse/t/delta_0000023_0000023_0000/bucket_00001
drwxr-xr-x   - ekoifman staff          0 2016-06-09 17:07 /user/hive/warehouse/t/delta_0000024_0000024_0000
-rw-r--r--   1 ekoifman staff        610 2016-06-09 17:07 /user/hive/warehouse/t/delta_0000024_0000024_0000/bucket_00000
-rw-r--r--   1 ekoifman staff        610 2016-06-09 17:07 /user/hive/warehouse/t/delta_0000024_0000024_0000/bucket_00001
```

其中delta_0000023_0000023_0000中，0000023表示对应事务的ID，0000表示序号。

从上面的表中我们可以看到有两个事务的数据还未合并成base。

### 事务表的读取

假设我们要读取上面表”t“的数据，由于它的分桶数量是2，因此正常情况下，它的并行度应该也是2。

hive会启动两个task，一个task读取base_0000022/bucket_00000、delta_0000023_0000023_0000/bucket_00000、delta_0000024_0000024_0000/bucket_00000然后进行合并，另一个task则读取base_0000022/bucket_00001、delta_0000023_0000023_0000/bucket_00001、delta_0000024_0000024_0000/bucket_00001然后进行合并。**所以和正常的表相比，事务表在读取分桶数据时需要再读取delta文件夹下面对应分桶数据**。因此我们也要保证delta文件的数量不会太大太多，这就需要delta文件的压缩机制了。

### delta文件的压缩

Compactor是一个在Hive Metastore上运行的一系列后台线程，主要包括Initiator, Worker, Cleaner, AcidHouseKeeperService 以及一些其他的组件。

#### 1、 压缩类型

- minor 压缩：将多个delta文件合并成一个delta文件 （维度是分桶级别）
- major 压缩：将多个delta文件和base文件合并成一个新的base文件 （维度是分桶级别）

所有的压缩任务都在后台运行，并且不会影响到数据的读写。当压缩完，压缩任务会等待旧文件的读取完毕后才删除该旧文件。（由于读取的时候需要指定一系列的事务id然后进行读取，因此因合并而生成的base文件或者delta文件并不会被读者看到并误读）

#### 2、Initiator 组件

这个组件主要是用于发现哪些表或者分区需要进行压缩。这个组件需要修改Metastore中的配置**hive.compactor.initiator.on**来开启。同时Hive新提供了几个" *.threshold"的参数用于判断是否要对表/分区进行压缩以及进行哪种类型的压缩。一个压缩任务只会压缩一个分区(如果表没有分区那就是直接压缩表的数据)，如果对一个分区压缩失败次数达到了 hive.compactor.initiator.failed.compacts.threshold 的次数，那么后面将不会再对该分区进行压缩。

#### 3、 Worker

每个Worker对应一个压缩任务。这个压缩任务其实就是一个MapReduce任务，任务名称格式为 <hostname>-compactor-<db>.<table>.<partition>。Worker会提交MapReduce任务到集群，并等待该任务完成（可以通过hive.compactor.job.queue指定提交的队列）。

hive.compactor.worker.threads 决定了有多少个任务运行在每个Metastore实例中。整个Hive集群的Worker数量决定了整个压缩任务的并行度。

#### 4、Cleaner

整个组件是用来删除压缩完的delta文件的，另外，如果一个delta文件被认为不再需要了，也会被这个组件删除。

#### 5、 AcidHouseKeeperService

这个组件主要用来监听事务开启后客户端的心跳，如果客户端在开启一个事务后，有 hive.txn.timeout 时间没有发送心跳过来，这个组件就会关闭这个事务并释放相关的锁。

#### 6、 SHOW COMPACTIONS

这个命令可以列出正在运行的压缩任务的信息以及近期的一些历史任务的信息。

### 事务表的隐藏字段

如果我们直接用orc api读取事务表的数据文件，会发现hive在事务表中添加了很多隐藏字段。假设我们创建一个表，有两个id和name两个字段，这时候我们读取该表的某个数据文件，会发现各字段列如下：

```shell
operation
originalTransaction
bucket
rowId
currentTransaction
row
```

其中row字段是一个struct类型，包含的就是我们实际的字段。另外，operation=0表示是新增数据，operation=1表示更新，operation=2表示删除的数据。

我们再把他的schema直接输出，可以看到：

```java
struct<
    operation:int,
    originalTransaction:bigint,
    bucket:int,
    rowId:bigint,
    currentTransaction:bigint,
    row:struct<
        _col1:int,
        _col2:string
    >
>
```

所以事务表的每一条数据都会存储它的事务id，行号、以及分桶id。

## 五、相关配置

要开启对事务表的支持，我们至少需要修改以下的配置：

### 客户端方面的修改

(会话中设置或者修改HiveServer2的配置文件)：

- hive.support.concurrency – true
- hive.enforce.bucketing – true (Hive 2.0之后就不用专门设置了)
- hive.exec.dynamic.partition.mode – nonstrict
- hive.txn.manager – org.apache.hadoop.hive.ql.lockmgr.DbTxnManager

### 服务端方面

主要修改MetaStore上的配置

- hive.compactor.initiator.on – true
- hive.compactor.worker.threads – 压缩任务的数量

### 为事务新增的相关配置

<https://cwiki.apache.org/confluence/display/Hive/Hive+Transactions#HiveTransactions-NewConfigurationParametersforTransactions>

### 一些旧的配置修改

| Configuration key                | Must be set to                                    |
| -------------------------------- | ------------------------------------------------- |
| hive.enforce.bucketing           | true (default is false) (Hive 2.0 开始就不需要了) |
| hive.exec.dynamic.partition.mode | nonstrict (default is strict)                     |
| hive.support.concurrency         | true (default is false)                           |

## 六、事务表的创建

注意，事务表创建后就不能再改成非事务表了。并且需要在会话中设置 ”hive.txn.manager=org.apache.hadoop.hive.ql.lockmgr.DbTxnManager “后，才可以对事务表进行增删改查。

如果不想事务表自动继续内压缩，可以在创建事务表时添加配置"`NO_AUTO_COMPACTION`"。

下面是创建一个事务表的demo：

```mysql
CREATE TABLE table_name (
  id                int,
  name              string
)
CLUSTERED BY (id) INTO 2 BUCKETS STORED AS ORC
TBLPROPERTIES ("transactional"="true",
  "compactor.mapreduce.map.memory.mb"="2048",     -- specify compaction map job properties
  "compactorthreshold.hive.compactor.delta.num.threshold"="4",  -- trigger minor compaction if there are more than 4 delta directories
  "compactorthreshold.hive.compactor.delta.pct.threshold"="0.5" -- trigger major compaction if the ratio of size of delta files to
                                                                   -- size of base files is greater than 50%
);
```

针对事务表的压缩类型进行一些修改

```mysql
ALTER TABLE table_name COMPACT 'minor'
   WITH OVERWRITE TBLPROPERTIES ("compactor.mapreduce.map.memory.mb"="3072");  -- specify compaction map job properties
ALTER TABLE table_name COMPACT 'major'
   WITH OVERWRITE TBLPROPERTIES ("tblprops.orc.compress.size"="8192");         -- change any other Hive table properties
```

## 七、一些问题的解答

### 1. 执行update、delete会生成job提交到集群吗？性能如何，能否hold住大量的更新操作？

可以把每次的UPDATE和DELETE操作理解为一次查询然后写入一个新的文件的过程。因此如果涉及大量的修改删除操作，性能可能会很差。  

### 2. 隐藏字段会导致数据文件变大，增量是多少？

因为隐藏字段大多都是int类型，在orc文件中压缩比会很好，因此实际并不会占用太大空间。

做了个测试，100M的数据文件大概会因为隐藏字段而膨胀到120M，增量大概是20%。  

### 3. 什么场景下适合用这个特性？

如果对于行级更新删除需求比较频繁的，可以考虑使用事务表，**但平常的hive表并不建议使用事务表**。因为事务表的限制很多，加上由于hive表的特性，也很难满足高并发的场景。

另外，如果事务表太多，并且存在大量的更新操作，metastore后台启动的合并线程会定期的提交MapReduce Job，也会一定程度上增重集群的负担。

**所以，结论是，除非有非常迫切的行级更新需求，又只能用hive表来做，才需要去考虑事务表。**

### 4. 目前Hive ACID的活跃度如何？

社区不是很活跃，虽然hive从0.13就开始支持ACID了。但是现在Hive版本已经到3.x了，根据在调研过程搜索到的资料来看，真正用的人应该不会太多。

## 八、参考资料

- <https://cwiki.apache.org/confluence/display/Hive/Hive+Transactions>
- <https://hortonworks.com/tutorial/using-hive-acid-transactions-to-insert-update-and-delete-data/#operational-tools-for-acid>
- <https://blog.csdn.net/wzq6578702/article/details/72802151>