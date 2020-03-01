[TOC]

## 一、Consumer的poll模型  

### poll执行流程

一般我们通过`KafkaConsumer#poll()`方法拉取最新的日志。这个方法执行的时候大概经过了以下流程：

![](http://assets.processon.com/chart_image/5bdd0c53e4b01ac4965fb8f0.png?_=1541215751783)

### 相关源码解析   

```java
// KafkaConsumer.java 
//timeout如果为0，表示不等待立即返回
public ConsumerRecords<K, V> poll(long timeout) {
        acquire();
        try {
            //timeout不能小于0.
            if (timeout < 0)
                throw new IllegalArgumentException("Timeout must not be negative");
			//subscribed和assigned两种订阅模式必须选择一个
            if (this.subscriptions.hasNoSubscriptionOrUserAssignment())
                throw new IllegalStateException("Consumer is not subscribed to any topics or assigned any partitions");

            // poll for new data until the timeout expires
            long start = time.milliseconds();
            long remaining = timeout;
            do {
                //拉取数据
                Map<TopicPartition, List<ConsumerRecord<K, V>>> records = pollOnce(remaining);
                if (!records.isEmpty()) {
                    //如果拉取到的数据不为空，则可以提前把下次的拉取请求也发送出去，这样下次执行poll时就不会因为等待数据而阻塞太久
                    if (fetcher.sendFetches() > 0 || client.pendingRequestCount() > 0)
                        client.pollNoWakeup();

                    if (this.interceptors == null)
                        return new ConsumerRecords<>(records);
                    else
                        return this.interceptors.onConsume(new ConsumerRecords<>(records));
                }

                long elapsed = time.milliseconds() - start;
                remaining = timeout - elapsed;
            } while (remaining > 0);

            return ConsumerRecords.empty();
        } finally {
            release();
        }
    }

private Map<TopicPartition, List<ConsumerRecord<K, V>>> pollOnce(long timeout) {
   		//加入到指定Group组中，只有在subscribed模式下才需要加入到Group
    	//这个过程包括了获取要消费的partition
        coordinator.poll(time.milliseconds());

        //判断要消费的partition的对应offset是否已经知道了，如果不知道就要去获取起始的offset
        if (!subscriptions.hasAllFetchPositions())
            updateFetchPositions(this.subscriptions.missingFetchPositions());

        //可能FETCH请求之前提前发送过，我们把之前获取到的数据直接拿回来就好了
        Map<TopicPartition, List<ConsumerRecord<K, V>>> records = fetcher.fetchedRecords();
        if (!records.isEmpty())
            return records;

        //发送FETCH请求
        fetcher.sendFetches();

        long now = time.milliseconds();
        long pollTimeout = Math.min(coordinator.timeToNextPoll(now), timeout);

        client.poll(pollTimeout, now, new PollCondition() {
            @Override
            public boolean shouldBlock() {
                return !fetcher.hasCompletedFetches();
            }
        });

        //在拉取数据的过程中，可能Group的状态发生了改变，所以需要检查一下Group的状态是否发生了改变
    	//如果Group状态发生了改变，就需要重新rejoin，以尽快让Group状态处于稳定
        if (coordinator.needRejoin())
            return Collections.emptyMap();

        return fetcher.fetchedRecords();
    }
```

下面我们详细的介绍一下各个流程

## 二、获取要消费的partition  

### 三种订阅模式  

在使用KafkaConsumer时，有三种模式获取要消费的partiton。这些模式定义在SubscriptionType枚举类中

- **AUTO_TOPICS**模式：通过订阅相关topic，加入到指定Group后，由GroupCoordinator来分配要消费的partition。**AUTO_TOPICS模式是topic粒度级别的订阅**
- **AUTO_PATTERN**模式：用户可以指定一个parttern，consumer需要去获取所有topics，然后去匹配parttern，匹配上的那些topic就是要消费的那些topic，之后和**AUTO_TOPICS**模式加入Group获取要消费的partition。**AUTO_PATTERN模式是topic粒度级别的订阅**
- **USER_ASSIGNED**模式：直接执行KafkaConsumer#assign()方法来指定要消费的topic-partition。**USER_ASSIGNED模式是parition粒度级别的订阅**

### AUTO_TOPICS 和 AUTO_PATTERN模式  

subscribe有3个重载方法：

```java
    public void subscribe(Collection<String> topics);
    public void subscribe(Collection<String> topics, ConsumerRebalanceListener callback);
    public void subscribe(Pattern pattern, ConsumerRebalanceListener callback);
```

我们可以指定一个topic列表，或者直接指定要消费topic的pattern。

如果是AUTO_TOPICS模式，Consumer会去broker拉取指定topics的元数据。如果是AUTO_PATTERN，Consumer就会将所有topics的元数据拉取下来，然后去匹配获取真正要消费的topics是哪些。

获取到topics的元数据后，在执行poll方法拉取数据的时候，consumer就会自动帮我们加入Group，然后获取要消费的partition。

Consumer如何加入和离开Group的，可以看我的这篇文章：

[Consumer 加入&离开 Group详解](https://blog.csdn.net/u013332124/article/details/83548706)

### USER_ASSIGNED模式

通过assign方法，我们可以直接指定要消费那些topic-partition。

```java
public void assign(Collection<TopicPartition> partitions) {}
```

由于该模式是用户自己指定要消费哪些topic-partition，因此，当topic的partition数量发生改变时，程序不会做通知，用户需要自行去处理。

该模式下Consumer就不会执行加入Group的那些操作。

## 三、获取partition当前commitedOffset

获取到要消费哪些partition后，就要知道要从partiton的哪个offset开始消费。也就是要确定commitedOffset

![](http://assets.processon.com/chart_image/5bdd5f58e4b01e422a3b7501.png?_=1541234851264)

Consumer在消费时，会在内存中保存各个partition的commitedOffset。

但是在刚刚获取到要消费的parititon是哪些的时候，还并没有partition的commitedOffset数据。这时候就需要向GroupCoordinator发送**OFFSET_FETCH请求**来获取对应partition的commitedOffset。

不过，只有AUTO_TOPICS 和 AUTO_PATTERN模式才可能可以从GroupCoordinator获取到commitedOffset记录（如果指定的Group是新建的，则也获取不到）。USER_ASSIGNED模式由于不归GroupCoordinator管理，所以GroupCoordinator无法知道其commitedOffset。

如果无法从GroupCoordinator获取消费过的offset，Consumer就会通过`auto.offset.reset`配置定义的策略设置一个新的offset作为commitedOffset。`auto.offset.reset`有三种策略：

1. latest:**默认策略**。获取partition中最新的一条offset。也就是LEO
2. earliest：获取partition的最早的一条offset。
3. none：配置了该策略的话，如果没有从GroupCoordinator获取到commitedOffset，就会抛异常。

Consumer根据策略的获取offset的方式是向broker发送**LIST_OFFSETS请求**。broker就会根据策略返回指定partition的offset

## 四、Consumer 消费数据  

知道了消费的parition和对应的commitedOffset后，Consumer就开始消费数据了。Consumer会遍历要消费的partition，找出要往哪些broker发送**FETCH请求**(根据partition的leader所在的broker)。之后遍历这些broker，逐个发送FETCH请求，leader在同一个broker的partition的FETCH请求会一起发送。

### broker 处理 FETCH 请求

broker收到fetch请求后会先过滤掉一些没有权限和不合法的partition。之后调用broker上replicaManager组件的fetchMessages方法。

ReplicaManager组件的介绍可以看我的另一篇博客：

[ReplicaManager 详解（八）](https://blog.csdn.net/u013332124/article/details/82860786)

下面介绍一下replicaManager#fetchMessages()方法

```scala
def fetchMessages(timeout: Long,
                    replicaId: Int,
                    fetchMinBytes: Int,
                    fetchMaxBytes: Int,
                    hardMaxBytesLimit: Boolean,
                    fetchInfos: Seq[(TopicPartition, PartitionData)],
                    quota: ReplicaQuota = UnboundedQuota,
                    responseCallback: Seq[(TopicPartition, FetchPartitionData)] => Unit) {
    //判断是否来自follower
    val isFromFollower = replicaId >= 0
    //如果replicaId!=-2，就说明是正常的fetch请求，则只能拉取leader partition的数据
    val fetchOnlyFromLeader: Boolean = replicaId != Request.DebuggingConsumerId
    //判断是否是从broker发出的fetch请求，不是的话就只能读取HW线以下的数据
    val fetchOnlyCommitted: Boolean = ! Request.isValidBrokerId(replicaId)

    //从本地文件读取log
    val logReadResults = readFromLocalLog(
      replicaId = replicaId,
      fetchOnlyFromLeader = fetchOnlyFromLeader,
      readOnlyCommitted = fetchOnlyCommitted,
      fetchMaxBytes = fetchMaxBytes,
      hardMaxBytesLimit = hardMaxBytesLimit,
      readPartitionInfo = fetchInfos,
      quota = quota)

    //如果是从其他的broker发来的fetch请求，在读取完数据后还要更新一些数据
    if(Request.isValidBrokerId(replicaId))
      updateFollowerLogReadResults(replicaId, logReadResults)

    val logReadResultValues = logReadResults.map { case (_, v) => v }
    //获取拉取到的数据总大小是多少
    val bytesReadable = logReadResultValues.map(_.info.records.sizeInBytes).sum
    val errorReadingData = logReadResultValues.foldLeft(false) ((errorIncurred, readResult) =>
      errorIncurred || (readResult.error != Errors.NONE))
	//如果满足下面这些条件，则可以立刻返回数据给请求者
    if (timeout <= 0 || fetchInfos.isEmpty || bytesReadable >= fetchMinBytes || errorReadingData) {
      val fetchPartitionData = logReadResults.map { case (tp, result) =>
        tp -> FetchPartitionData(result.error, result.hw, result.info.records)
      }
      responseCallback(fetchPartitionData)
    } else {
      //否则就构建一个延迟操作等待条件满足
      val fetchPartitionStatus = logReadResults.map { case (topicPartition, result) =>
        val fetchInfo = fetchInfos.collectFirst {
          case (tp, v) if tp == topicPartition => v
        }.getOrElse(sys.error(s"Partition $topicPartition not found in fetchInfos"))
        (topicPartition, FetchPartitionStatus(result.info.fetchOffsetMetadata, fetchInfo))
      }
      val fetchMetadata = FetchMetadata(fetchMinBytes, fetchMaxBytes, hardMaxBytesLimit, fetchOnlyFromLeader,
        fetchOnlyCommitted, isFromFollower, replicaId, fetchPartitionStatus)
      val delayedFetch = new DelayedFetch(timeout, fetchMetadata, this, quota, responseCallback)

      val delayedFetchKeys = fetchPartitionStatus.map { case (tp, _) => new TopicPartitionOperationKey(tp) }

      delayedFetchPurgatory.tryCompleteElseWatch(delayedFetch, delayedFetchKeys)
    }
  }
```

1. 从本地文件中读取数据
2. 如果是其他broker发来的fetch请求，还会更新一些数据，比如partition HW的变动，以及ISR的变动等。
3. 根据一些条件判断是否要将读取到的数据立即返回。有一些情况需要等待一定时间后再返回，kafka会将构造一个延迟操作。比如fetch请求配置了fetchMinBytes，但是之前拉取到的数据又没达到指定的最小大小，则就需要等待更多的数据来消费。

readFromLocalLog():

```Scala
  def readFromLocalLog(replicaId: Int,
                       fetchOnlyFromLeader: Boolean,
                       readOnlyCommitted: Boolean,
                       fetchMaxBytes: Int,
                       hardMaxBytesLimit: Boolean,
                       readPartitionInfo: Seq[(TopicPartition, PartitionData)],
                       quota: ReplicaQuota): Seq[(TopicPartition, LogReadResult)] = {
		def read(tp: TopicPartition, fetchInfo: PartitionData, limitBytes: Int, minOneMessage: Boolean): LogReadResult = {
        	...                                                                                               		}
    
	//计算limitBytes、minOneMessage，然后逐个遍历topicPartition读取数据
    var limitBytes = fetchMaxBytes
    val result = new mutable.ArrayBuffer[(TopicPartition, LogReadResult)]
    //是否至少要获取一条数据。有时可能因为每一条消息都比较大，都大于fetchMaxBytes，导致数据永远无法被消费的问题,这时候就需要设置minOneMessage为true保证一定可以获取到一条数据
    var minOneMessage = !hardMaxBytesLimit
    readPartitionInfo.foreach { case (tp, fetchInfo) =>
        //根据相关限制条件从partition中读取数据
      val readResult = read(tp, fetchInfo, limitBytes, minOneMessage)
        //从该partition中读取的数据大小
      val messageSetSize = readResult.info.records.sizeInBytes
      //
      if (messageSetSize > 0)
        minOneMessage = false
          //计算剩下的limitBytes
      limitBytes = math.max(0, limitBytes - messageSetSize)
      result += (tp -> readResult)
    }
    result
  }
//readFromLocalLog中定义的read方法
def read(tp: TopicPartition, fetchInfo: PartitionData, limitBytes: Int, minOneMessage: Boolean): LogReadResult = {
      val offset = fetchInfo.offset
      //每个partition也有自己拉取的maxBytes
      val partitionFetchSize = fetchInfo.maxBytes

      BrokerTopicStats.getBrokerTopicStats(tp.topic).totalFetchRequestRate.mark()
      BrokerTopicStats.getBrokerAllTopicsStats().totalFetchRequestRate.mark()

      try {
        trace(s"Fetching log segment for partition $tp, offset $offset, partition fetch size $partitionFetchSize, " +
          s"remaining response limit $limitBytes" +
          (if (minOneMessage) s", ignoring response/partition size limits" else ""))

        //是否只能从leader读数据
        val localReplica = if (fetchOnlyFromLeader)
          getLeaderReplicaIfLocal(tp)
        else
          getReplicaOrException(tp)

        //是否只能读取已提交的offset(也就是HW水位线以下的offset)
        val maxOffsetOpt = if (readOnlyCommitted)
          Some(localReplica.highWatermark.messageOffset)
        else
          None

        val initialLogEndOffset = localReplica.logEndOffset.messageOffset
        val initialHighWatermark = localReplica.highWatermark.messageOffset
        val fetchTimeMs = time.milliseconds
        val logReadInfo = localReplica.log match {
          case Some(log) =>
            val adjustedFetchSize = math.min(partitionFetchSize, limitBytes)
            val fetch = log.read(offset, adjustedFetchSize, maxOffsetOpt, minOneMessage)

            if (shouldLeaderThrottle(quota, tp, replicaId))
              FetchDataInfo(fetch.fetchOffsetMetadata, MemoryRecords.EMPTY)
            else if (!hardMaxBytesLimit && fetch.firstEntryIncomplete)
              FetchDataInfo(fetch.fetchOffsetMetadata, MemoryRecords.EMPTY)
            else fetch

          case None =>
            error(s"Leader for partition $tp does not have a local log")
            FetchDataInfo(LogOffsetMetadata.UnknownOffsetMetadata, MemoryRecords.EMPTY)
        }

        LogReadResult(info = logReadInfo,
                      hw = initialHighWatermark,
                      leaderLogEndOffset = initialLogEndOffset,
                      fetchTimeMs = fetchTimeMs,
                      readSize = partitionFetchSize,
                      exception = None)
      } catch {
        case e@ (_: UnknownTopicOrPartitionException |
                 _: NotLeaderForPartitionException |
                 _: ReplicaNotAvailableException |
                 _: OffsetOutOfRangeException) =>
          LogReadResult(info = FetchDataInfo(LogOffsetMetadata.UnknownOffsetMetadata, MemoryRecords.EMPTY),
                        hw = -1L,
                        leaderLogEndOffset = -1L,
                        fetchTimeMs = -1L,
                        readSize = partitionFetchSize,
                        exception = Some(e))
        case e: Throwable =>
          BrokerTopicStats.getBrokerTopicStats(tp.topic).failedFetchRequestRate.mark()
          BrokerTopicStats.getBrokerAllTopicsStats().failedFetchRequestRate.mark()
          error(s"Error processing fetch operation on partition $tp, offset $offset", e)
          LogReadResult(info = FetchDataInfo(LogOffsetMetadata.UnknownOffsetMetadata, MemoryRecords.EMPTY),
                        hw = -1L,
                        leaderLogEndOffset = -1L,
                        fetchTimeMs = -1L,
                        readSize = partitionFetchSize,
                        exception = Some(e))
      }
    }
```

该方法主要是遍历要消费的partition，然后获取Log对象根据限定条件去读取数据。有两点值得注意：

1. 读取数据的时候有一个布尔值的参数**minOneMessage**，表示本次是否至少要读取一条数据。在0.10.2之前的版本，如果有一条消息的大小超过了Consumer的fetchMaxBytes所设置的值，就会导致这条消息以及后面的消息都无法消费的情况，程序就一直卡在那了。因此，0.10.2之后新增了这个参数，用来保证这种情况下还能顺利的消费数据。
2. 从上面的代码可以看出，partition是逐个消费的，有个limitBytes的变量表示当前还能消费多少。**如果第一个消费的partition拉取的数据大小就达到了limitBytes，就会导致后面的partition都无法消费的问题**。因此每个partition也有自己的消费上限值maxBytes，这个值是由Consumer的配置` max.partition.fetch.bytes`决定的，初始值是1M。这个配置最好要远小于`fetch.max.bytes`比较好，就不会导致排在后面的partition无法消费的问题。(partition的排序顺序是根据TopicPartition对象的hash值决定的)

**Log.read()方法:**

```Scala
  def read(startOffset: Long, maxLength: Int, maxOffset: Option[Long] = None, minOneMessage: Boolean = false): FetchDataInfo = {
    trace("Reading %d bytes from offset %d in log %s of length %d bytes".format(maxLength, startOffset, name, size))

    val currentNextOffsetMetadata = nextOffsetMetadata
    val next = currentNextOffsetMetadata.messageOffset
    if(startOffset == next)
      return FetchDataInfo(currentNextOffsetMetadata, MemoryRecords.EMPTY)
	//根据startOffset获取对应的segment
    var entry = segments.floorEntry(startOffset)

    // attempt to read beyond the log end offset is an error
    if(startOffset > next || entry == null)
      throw new OffsetOutOfRangeException("Request for offset %d but we only have log segments in the range %d to %d.".format(startOffset, segments.firstKey, next))

    while(entry != null) {
        //获取该segment中最大的那个offset所在的位置
      val maxPosition = {
        if (entry == segments.lastEntry) {
          val exposedPos = nextOffsetMetadata.relativePositionInSegment.toLong
              //为了防止在这个过程中active segment变动的情况
          if (entry != segments.lastEntry)
            entry.getValue.size
          else
            exposedPos
        } else {
          entry.getValue.size
        }
      }
       // 通过segment的方法读取数据
      val fetchInfo = entry.getValue.read(startOffset, maxOffset, maxLength, maxPosition, minOneMessage)
      if(fetchInfo == null) {
        entry = segments.higherEntry(entry.getKey)
      } else {
        return fetchInfo
      }
    }
      
    FetchDataInfo(nextOffsetMetadata, MemoryRecords.EMPTY)
  }
//LogSegment.scala
def read(startOffset: Long, maxOffset: Option[Long], maxSize: Int, maxPosition: Long = size,
           minOneMessage: Boolean = false): FetchDataInfo = {
    if (maxSize < 0)
      throw new IllegalArgumentException("Invalid max size for log read (%d)".format(maxSize))

    val logSize = log.sizeInBytes // this may change, need to save a consistent copy
    val startOffsetAndSize = translateOffset(startOffset)

    // if the start position is already off the end of the log, return null
    if (startOffsetAndSize == null)
      return null

    val startPosition = startOffsetAndSize.position.toInt
    val offsetMetadata = new LogOffsetMetadata(startOffset, this.baseOffset, startPosition)

    val adjustedMaxSize =
      if (minOneMessage) math.max(maxSize, startOffsetAndSize.size)
      else maxSize

    // return a log segment but with zero size in the case below
    if (adjustedMaxSize == 0)
      return FetchDataInfo(offsetMetadata, MemoryRecords.EMPTY)

    // calculate the length of the message set to read based on whether or not they gave us a maxOffset
    val length = maxOffset match {
      case None =>
        min((maxPosition - startPosition).toInt, adjustedMaxSize)
      case Some(offset) =>
        if (offset < startOffset)
          return FetchDataInfo(offsetMetadata, MemoryRecords.EMPTY, firstEntryIncomplete = false)
        val mapping = translateOffset(offset, startPosition)
        val endPosition =
          if (mapping == null)
            logSize // the max offset is off the end of the log, use the end of the file
          else
            mapping.position
        min(min(maxPosition, endPosition) - startPosition, adjustedMaxSize).toInt
    }

    FetchDataInfo(offsetMetadata, log.read(startPosition, length),
      firstEntryIncomplete = adjustedMaxSize < startOffsetAndSize.size)
  }
```

1. Log.read()方法会先获取到startOffset对应的那个segment，同时计算出该segment可以消费的最大的那个位置。这个过程并没有使用任何同步原语，因此如果刚好找到的segment是active segment的话，可能导致在计算时active segment刚好出现变动的情况，因此上面的代码中出现了`if (entry == segments.lastEntry)`的判断。用这种巧妙的方法替代同步原语，也能提升一定的性能。

2. 在LogSegment.read()方法中，主要就是计算出要读的起始位置以及结束的位置，然后执行FileRecords.read()读取信息。FileRecords.read()其实就只是对LogSegment的FileRecords根据startPosition和endPosition做了一个切片。之后FETCH请求在真正返回的时候只会读取startPosition到endPosition之间的数据。

   kafka在读取数据的时候，底层其实使用了javaNio的`FileChannel#transferTo()`方法实现了**零拷贝**。正常情况下，我们读数据都是将数据从输入流读取到内存，然后通过输出流传输过去。这种方式直接在日志文件输入流和socket输出流之间打通了一条通道，性能会提升很多。

**延迟fetch操作的相关代码：**

```scala
//DelayedFetch.scala
  /**
   * The operation can be completed if:
   *
   * Case A: This broker is no longer the leader for some partitions it tries to fetch
   * Case B: This broker does not know of some partitions it tries to fetch
   * Case C: The fetch offset locates not on the last segment of the log
   * Case D: The accumulated bytes from all the fetching partitions exceeds the minimum bytes
   *
   * Upon completion, should return whatever data is available for each valid partition
   */
  override def tryComplete() : Boolean = {
    var accumulatedSize = 0
    var accumulatedThrottledSize = 0
    fetchMetadata.fetchPartitionStatus.foreach {
      case (topicPartition, fetchStatus) =>
        val fetchOffset = fetchStatus.startOffsetMetadata
        try {
          if (fetchOffset != LogOffsetMetadata.UnknownOffsetMetadata) {
            val replica = replicaManager.getLeaderReplicaIfLocal(topicPartition)
            val endOffset =
              if (fetchMetadata.fetchOnlyCommitted)
                replica.highWatermark
              else
                replica.logEndOffset

            // Go directly to the check for Case D if the message offsets are the same. If the log segment
            // has just rolled, then the high watermark offset will remain the same but be on the old segment,
            // which would incorrectly be seen as an instance of Case C.
            if (endOffset.messageOffset != fetchOffset.messageOffset) {
              if (endOffset.onOlderSegment(fetchOffset)) {
                // Case C, this can happen when the new fetch operation is on a truncated leader
                debug("Satisfying fetch %s since it is fetching later segments of partition %s.".format(fetchMetadata, topicPartition))
                return forceComplete()
              } else if (fetchOffset.onOlderSegment(endOffset)) {
                // Case C, this can happen when the fetch operation is falling behind the current segment
                // or the partition has just rolled a new segment
                debug("Satisfying fetch %s immediately since it is fetching older segments.".format(fetchMetadata))
                // We will not force complete the fetch request if a replica should be throttled.
                if (!replicaManager.shouldLeaderThrottle(quota, topicPartition, fetchMetadata.replicaId))
                  return forceComplete()
              } else if (fetchOffset.messageOffset < endOffset.messageOffset) {
                // we take the partition fetch size as upper bound when accumulating the bytes (skip if a throttled partition)
                val bytesAvailable = math.min(endOffset.positionDiff(fetchOffset), fetchStatus.fetchInfo.maxBytes)
                if (quota.isThrottled(topicPartition))
                  accumulatedThrottledSize += bytesAvailable
                else
                  accumulatedSize += bytesAvailable
              }
            }
          }
        } catch {
          case _: UnknownTopicOrPartitionException => // Case B
            debug("Broker no longer know of %s, satisfy %s immediately".format(topicPartition, fetchMetadata))
            return forceComplete()
          case _: NotLeaderForPartitionException =>  // Case A
            debug("Broker is no longer the leader of %s, satisfy %s immediately".format(topicPartition, fetchMetadata))
            return forceComplete()
        }
    }

    // Case D
    if (accumulatedSize >= fetchMetadata.fetchMinBytes
      || ((accumulatedSize + accumulatedThrottledSize) >= fetchMetadata.fetchMinBytes && !quota.isQuotaExceeded()))
      forceComplete()
    else
      false
  }
//在tryComplete中执行forceComplete()最终会触发onComplete()方法的执行
  override def onComplete() {
    val logReadResults = replicaManager.readFromLocalLog(
      replicaId = fetchMetadata.replicaId,
      fetchOnlyFromLeader = fetchMetadata.fetchOnlyLeader,
      readOnlyCommitted = fetchMetadata.fetchOnlyCommitted,
      fetchMaxBytes = fetchMetadata.fetchMaxBytes,
      hardMaxBytesLimit = fetchMetadata.hardMaxBytesLimit,
      readPartitionInfo = fetchMetadata.fetchPartitionStatus.map { case (tp, status) => tp -> status.fetchInfo },
      quota = quota
    )

    val fetchPartitionData = logReadResults.map { case (tp, result) =>
      tp -> FetchPartitionData(result.error, result.hw, result.info.records)
    }

    responseCallback(fetchPartitionData)
  }
```

在前面我们有讲到，如果拉取到的数据还没达到了minFetchBytes，kafka会建立一个延迟操作，以尽量保证可以拉取到更多的数据。对于这个延迟操作，有下面4种情况可能提前让该操作完成：

1. 要拉取的partition的leader不在本broker上了
2. broker获取不到要拉取的partiton的相关信息
3. 要拉取的offset不属于最新的那个segment了
4. 可以拉取的数据已经达到minFetchBytes了

关于kafka延迟操作的相关介绍可以看我的另外一篇博客：

[TimingWheel 时间轮详解](https://blog.csdn.net/u013332124/article/details/82119144)

## 五、offset提交  

消费完数据之后，offset的提交方式非常重要。**offset提交早了可能导致消息丢失，offset提交晚了可能导致消息重复消费。**在Consumer类中，有5个方法可以提交offset。

```java
//同步提交已经拉取回来的partition的offset
public void commitSync();
//同步提交指定topic-partition的指定offset
public void commitSync(Map<TopicPartition, OffsetAndMetadata> offsets);
//异步提交已经拉取回来的partition的offset
public void commitAsync();
//异步提交已经拉取回来的partition的offset，带上一个回调函数
public void commitAsync(OffsetCommitCallback callback);
//异步提交指定topic-partition的指定offset，带上一个回调函数
public void commitAsync(Map<TopicPartition, OffsetAndMetadata> offsets, OffsetCommitCallback callback);
```

这里总体分为两种形式，同步提交和异步提交。但是如果保证**消息至少消费一次、至多消费一次、刚好只消费一次**只能由用户自己来控制。后面我会再写一篇博客专门介绍这个。

### 同步和异步提交

同步提交和异步提交其实底层都是向broker发送OFFSET_COMMIT请求。不同的是同步的会等待请求返回，异步的不会等待请求返回。

**同步提交commit代码：**

```java
    public boolean commitOffsetsSync(Map<TopicPartition, OffsetAndMetadata> offsets, long timeoutMs) {
        invokeCompletedOffsetCommitCallbacks();

        if (offsets.isEmpty())
            return true;

        long now = time.milliseconds();
        long startMs = now;
        long remainingMs = timeoutMs;
        do {
            if (coordinatorUnknown()) {
                if (!ensureCoordinatorReady(now, remainingMs))
                    return false;

                remainingMs = timeoutMs - (time.milliseconds() - startMs);
            }
			//发送OFFSET_COMMIT请求
            RequestFuture<Void> future = sendOffsetCommitRequest(offsets);
            client.poll(future, remainingMs);

            if (future.succeeded()) {
                if (interceptors != null)
                    interceptors.onCommit(offsets);
                return true;
            }

            if (!future.isRetriable())
                throw future.exception();

            time.sleep(retryBackoffMs);

            now = time.milliseconds();
            remainingMs = timeoutMs - (now - startMs);
        } while (remainingMs > 0);

        return false;
    }
```

在同步提交的方法中，发送完OFFSET_COMMIT请求后会等待请求返回了方法才返回。

**异步提交commit代码：**

```java
private void doCommitOffsetsAsync(final Map<TopicPartition, OffsetAndMetadata> offsets, final OffsetCommitCallback callback) {
        this.subscriptions.needRefreshCommits();
        RequestFuture<Void> future = sendOffsetCommitRequest(offsets);
        final OffsetCommitCallback cb = callback == null ? defaultOffsetCommitCallback : callback;
        future.addListener(new RequestFutureListener<Void>() {
            @Override
            public void onSuccess(Void value) {
                if (interceptors != null)
                    interceptors.onCommit(offsets);

                completedOffsetCommits.add(new OffsetCommitCompletion(cb, offsets, null));
            }

            @Override
            public void onFailure(RuntimeException e) {
                Exception commitException = e;

                if (e instanceof RetriableException)
                    commitException = new RetriableCommitFailedException(e);

                completedOffsetCommits.add(new OffsetCommitCompletion(cb, offsets, commitException));
            }
        });
}
```

异步请求只是发送一个OFFSET_COMMIT请求，并不会等待请求返回。

### 自动提交offset

如果partition是由GroupCoordinator分配并管理的，并且Consumer配置`enable.auto.commit`为true，则在每次poll拉取数据之前，kafka都会自动异步提交已经拉取过的offset。该配置默认是开启的，自动提交的频率和`auto.commit.interval.ms`有关，该配置的默认值是5000。



