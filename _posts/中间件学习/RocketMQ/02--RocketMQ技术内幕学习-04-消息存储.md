## 探究点

1. RocketMQ 存储概要设计
2. 消息发送存储流
3. 存储文件组织与内存映射机制
4. RocketMQ 存储文件
5. 消息消费队列、 索引文件构建机和制
6. RocketMQ 文件恢复机制
7. RocketMQ 刷盘机制
8. RocketMQ 文件删除机制







## 知识点

![RocketMQ技术内幕-04—消息存储](../../../../照片/typora/RocketMQ技术内幕-04—消息存储.png)





## 重要对象

### DefaultMessageStore

* 作用：其他模块对消息实体的操作都是通过此类进行操作的

* 源码

  ```java
  public class DefaultMessageStore implements MessageStore {
      private static final InternalLogger log = InternalLoggerFactory.getLogger(LoggerName.STORE_LOGGER_NAME);
  
      //消息存储配置属性
      private final MessageStoreConfig messageStoreConfig;
  
  
      // CommitLog
      // CommitLog文件的存储实现类
      private final CommitLog commitLog;
  
  
      //消息队列存储缓存表，按消息主题分组
      private final ConcurrentMap<String/* topic */, ConcurrentMap<Integer/* queueId */, ConsumeQueue>> consumeQueueTable;
  
  
      //消息队列文件ConsumeQueue刷盘线程
      private final FlushConsumeQueueService flushConsumeQueueService;
  
  
      //清除CommitLog文件服务
      private final CleanCommitLogService cleanCommitLogService;
  
  
      //清除ConsumeQueue文件服务
      private final CleanConsumeQueueService cleanConsumeQueueService;
  
  
      //索引文件实习类
      private final IndexService indexService;
  
  
      //MappedFile分配服务
      private final AllocateMappedFileService allocateMappedFileService;
  
  
      //CommitLog消息分发，根据CommitLog文件构建CosnumeQueue IndexFile文件
      private final ReputMessageService reputMessageService;
  
  
      //存储HA机制，HA是不是高可用???
      private final HAService haService;
  
      private final ScheduleMessageService scheduleMessageService;
  
      private final StoreStatsService storeStatsService;
  
  
      //消息堆内存缓存
      private final TransientStorePool transientStorePool;
  
      private final RunningFlags runningFlags = new RunningFlags();
      private final SystemClock systemClock = new SystemClock();
  
      private final ScheduledExecutorService scheduledExecutorService =
          Executors.newSingleThreadScheduledExecutor(new ThreadFactoryImpl("StoreScheduledThread"));
      private final BrokerStatsManager brokerStatsManager;
  
      //消息拉取长轮询模式消息达到监听器
      //TODO 此对象干嘛用的
      private final MessageArrivingListener messageArrivingListener;
  
      //Broker配置属性
      private final BrokerConfig brokerConfig;
  
      private volatile boolean shutdown = true;
  
      //文件刷盘检测点
      private StoreCheckpoint storeCheckpoint;
  
      private AtomicLong printTimes = new AtomicLong(0);
  
  
      //CommitLog文件转发请求
      private final LinkedList<CommitLogDispatcher> dispatcherList;
    
      //省略方法
      ....
  }      
  ```

### AppendMessageResult

<img src="../../../../照片/typora/image-20210804134631596.png" alt="image-20210804134631596" style="zoom:50%;" />

```java
/**
 * 追加结果
 * When write a message to the commit log, returns results
 */
public class AppendMessageResult {
  // 消息追加结果：Return code
  private AppendMessageStatus status;

  // 消息的物理偏移量：Where to start writing
  private long wroteOffset;

  // Write Bytes
  private int wroteBytes;

  // 消息ID，包含了消息偏移量
  private String msgId;

  // 消息存储时间戳
  private long storeTimestamp;

  //消息消费队列逻辑偏移量
  private long logicsOffset;
  private long pagecacheRT = 0;

  //消息条数，批量消息发送时消息条数
  private int msgNum = 1;
}
```





## 重要流程

### 消息发送存储流程

![RocketMQ--消息存储--消息发送存储过程](../../../../照片/typora/RocketMQ--消息存储--消息发送存储过程.jpg)



#### DefaultMessageStore

##### putMessage

* 以下几步
  * 先校验消息，校验将要写入的Broker
  * 保存消息
    * 延迟消息，
      * 将消息的原主题名称与原消息ID存入消息属性中，原因是因为会将此消息发送到延迟的消息主题中，需要保留之前的主题信息
      * 用延迟消息主题SCHEDULE_TOPIC、消息队列ID更新原先消息的主题与队列
    * 获取可以写入的CommitLog，将消息写入到CommitLog中

```java
@Override
public PutMessageResult putMessage(MessageExtBrokerInner msg) {

  //-----------------1. 存储前校验------------------
  //1. 校验当前Broker是否可以写入数据，下面的任何一个情况都不允许写入，（发现此处也是统一处理然后判断，如果不符合，则返回，很像自己写的代码）
  //   1. Broker停止工作
  //   2. Broker为SLAVE角色，或当前Rocket不支持写入
  //   3. 消息主题长度超 256 个字符、消息属性长度超过 65536 个字符将拒绝该消息写人
  PutMessageStatus checkStoreStatus = this.checkStoreStatus();
  if (checkStoreStatus != PutMessageStatus.PUT_OK) {
    return new PutMessageResult(checkStoreStatus, null);
  }

  PutMessageStatus msgCheckStatus = this.checkMessage(msg);
  if (msgCheckStatus == PutMessageStatus.MESSAGE_ILLEGAL) {
    return new PutMessageResult(msgCheckStatus, null);
  }


  //-----------------2. 存储消息------------------
  long beginTime = this.getSystemClock().now();

  //内部包含对延迟消息的存储操作
  PutMessageResult result = this.commitLog.putMessage(msg);
  long elapsedTime = this.getSystemClock().now() - beginTime;
  if (elapsedTime > 500) {
    log.warn("not in lock elapsed time(ms)={}, bodyLength={}", elapsedTime, msg.getBody().length);
  }

  this.storeStatsService.setPutMessageEntireTimeMax(elapsedTime);

  if (null == result || !result.isOk()) {
    this.storeStatsService.getPutMessageFailedTimes().incrementAndGet();
  }

  return result;
}
```

##### checkStoreStatus

```java
private PutMessageStatus checkStoreStatus() {
  if (this.shutdown) {
    log.warn("message store has shutdown, so putMessage is forbidden");
    return PutMessageStatus.SERVICE_NOT_AVAILABLE;
  }

  if (BrokerRole.SLAVE == this.messageStoreConfig.getBrokerRole()) {
    long value = this.printTimes.getAndIncrement();
    if ((value % 50000) == 0) {
      log.warn("message store has shutdown, so putMessage is forbidden");
    }
    return PutMessageStatus.SERVICE_NOT_AVAILABLE;
  }

  if (!this.runningFlags.isWriteable()) {
    long value = this.printTimes.getAndIncrement();
    if ((value % 50000) == 0) {
      log.warn("message store has shutdown, so putMessage is forbidden");
    }
    return PutMessageStatus.SERVICE_NOT_AVAILABLE;
  } else {
    this.printTimes.set(0);
  }

  if (this.isOSPageCacheBusy()) {
    return PutMessageStatus.OS_PAGECACHE_BUSY;
  }
  return PutMessageStatus.PUT_OK;
}
```



#### CommitLog

##### putMessage

```java
public PutMessageResult putMessage(final MessageExtBrokerInner msg) {
  // Set the storage time
  //设置消息存储时间
  msg.setStoreTimestamp(System.currentTimeMillis());
  // Set the message body BODY CRC (consider the most appropriate setting
  // on the client)
  msg.setBodyCRC(UtilAll.crc32(msg.getBody()));
  // Back to Results
  AppendMessageResult result = null;

  StoreStatsService storeStatsService = this.defaultMessageStore.getStoreStatsService();

  String topic = msg.getTopic();
  int queueId = msg.getQueueId();

  final int tranType = MessageSysFlag.getTransactionValue(msg.getSysFlag());


  //------------------延迟消息特殊处理-----------------
  //1. 延迟消息处理
  if (tranType == MessageSysFlag.TRANSACTION_NOT_TYPE
      || tranType == MessageSysFlag.TRANSACTION_COMMIT_TYPE) {
    // Delay Delivery

    //如果消息的延迟级别大于0，将备份原主题名称和原消息队列ID存入消息属性中，
    if (msg.getDelayTimeLevel() > 0) {
      if (msg.getDelayTimeLevel() > this.defaultMessageStore.getScheduleMessageService().getMaxDelayLevel()) {
        msg.setDelayTimeLevel(this.defaultMessageStore.getScheduleMessageService().getMaxDelayLevel());
      }


      //重置延迟消息的主题和queueId
      //延迟消息的主题和queueId值
      topic = ScheduleMessageService.SCHEDULE_TOPIC;
      queueId = ScheduleMessageService.delayLevel2QueueId(msg.getDelayTimeLevel());

      // Backup real topic, queueId
      MessageAccessor.putProperty(msg, MessageConst.PROPERTY_REAL_TOPIC, msg.getTopic());
      MessageAccessor.putProperty(msg, MessageConst.PROPERTY_REAL_QUEUE_ID, String.valueOf(msg.getQueueId()));
      msg.setPropertiesString(MessageDecoder.messageProperties2String(msg.getProperties()));


      //用延迟消息主题 SCHEDULE_TOPIC 和消息队列ID更新原先消息的主题和队列
      //如果不保存，之前的则会丢失
      msg.setTopic(topic);
      msg.setQueueId(queueId);
    }
  }

  InetSocketAddress bornSocketAddress = (InetSocketAddress) msg.getBornHost();
  if (bornSocketAddress.getAddress() instanceof Inet6Address) {
    msg.setBornHostV6Flag();
  }

  InetSocketAddress storeSocketAddress = (InetSocketAddress) msg.getStoreHost();
  if (storeSocketAddress.getAddress() instanceof Inet6Address) {
    msg.setStoreHostAddressV6Flag();
  }

  long elapsedTimeInLock = 0;

  MappedFile unlockMappedFile = null;


  //-----------------准备写入CommitLog文件---------------
  //2. 获取当前可以写入的CommitLog文件
  MappedFile mappedFile = this.mappedFileQueue.getLastMappedFile();
  //2.1 获取锁，表示写入CommitLog是串行的
  putMessageLock.lock(); //spin or ReentrantLock ,depending on store config
  try {
    long beginLockTimestamp = this.defaultMessageStore.getSystemClock().now();
    this.beginTimeInLock = beginLockTimestamp;

    // Here settings are stored timestamp, in order to ensure an orderly
    // global
    msg.setStoreTimestamp(beginLockTimestamp);


    //--------获取CommitLog对应的对象MappedFile--------
    //2.2 如果mapperFile为空，表明 ${ROCKET_HOME}/store/commitlog目录下不存在任何文件
    if (null == mappedFile || mappedFile.isFull()) {
      //用偏移量0创建第一个commit文件
      mappedFile = this.mappedFileQueue.getLastMappedFile(0); // Mark: NewFile may be cause noise
    }

    //2.3 如果文件创建失败抛出异常，很可能是磁盘空间不足，或者权限不够
    if (null == mappedFile) {
      log.error("create mapped file1 error, topic: " + msg.getTopic() + " clientAddr: " + msg.getBornHostString());
      beginTimeInLock = 0;
      return new PutMessageResult(PutMessageStatus.CREATE_MAPEDFILE_FAILED, null);
    }


    result = mappedFile.appendMessage(msg, this.appendMessageCallback);
    switch (result.getStatus()) {
      case PUT_OK:
        break;
      case END_OF_FILE:
        unlockMappedFile = mappedFile;
        // Create a new file, re-write the message
        mappedFile = this.mappedFileQueue.getLastMappedFile(0);
        if (null == mappedFile) {
          // XXX: warn and notify me
          log.error("create mapped file2 error, topic: " + msg.getTopic() + " clientAddr: " + msg.getBornHostString());
          beginTimeInLock = 0;
          return new PutMessageResult(PutMessageStatus.CREATE_MAPEDFILE_FAILED, result);
        }
        result = mappedFile.appendMessage(msg, this.appendMessageCallback);
        break;
      case MESSAGE_SIZE_EXCEEDED:
      case PROPERTIES_SIZE_EXCEEDED:
        beginTimeInLock = 0;
        return new PutMessageResult(PutMessageStatus.MESSAGE_ILLEGAL, result);
      case UNKNOWN_ERROR:
        beginTimeInLock = 0;
        return new PutMessageResult(PutMessageStatus.UNKNOWN_ERROR, result);
      default:
        beginTimeInLock = 0;
        return new PutMessageResult(PutMessageStatus.UNKNOWN_ERROR, result);
    }

    elapsedTimeInLock = this.defaultMessageStore.getSystemClock().now() - beginLockTimestamp;
    beginTimeInLock = 0;
  } finally {
    putMessageLock.unlock();
  }

  if (elapsedTimeInLock > 500) {
    log.warn("[NOTIFYME]putMessage in lock cost time(ms)={}, bodyLength={} AppendMessageResult={}", elapsedTimeInLock, msg.getBody().length, result);
  }

  if (null != unlockMappedFile && this.defaultMessageStore.getMessageStoreConfig().isWarmMapedFileEnable()) {
    this.defaultMessageStore.unlockMappedFile(unlockMappedFile);
  }

  PutMessageResult putMessageResult = new PutMessageResult(PutMessageStatus.PUT_OK, result);

  // Statistics
  //统计主题的增加一条消息
  storeStatsService.getSinglePutMessageTopicTimesTotal(msg.getTopic()).incrementAndGet();

  //主题的大小增加写入消息的大小
  storeStatsService.getSinglePutMessageTopicSizeTotal(topic).addAndGet(result.getWroteBytes());

  //执行刷盘操作
  handleDiskFlush(result, putMessageResult, msg);

  //执行主从同步操作
  handleHA(result, putMessageResult, msg);

  return putMessageResult;
}
```





#### MappedFile

##### appendMessagesInner

```java
/**
     * 写入消息的地方
     * @param messageExt
     * @param cb
     * @return
     */
public AppendMessageResult appendMessagesInner(final MessageExt messageExt, final AppendMessageCallback cb) {
  assert messageExt != null;
  assert cb != null;

  //获取MappedFile当前写指针
  int currentPos = this.wrotePosition.get();

  //-------------文件未满可以写入------------
  if (currentPos < this.fileSize) {

    //通过slice()方法创建一个与MappedFile的共享内存区，并设置position为当前指针
    //这个方法最后的结果就是写入到MappedFile的共享内存中
    ByteBuffer byteBuffer = writeBuffer != null ? writeBuffer.slice() : this.mappedByteBuffer.slice();
    byteBuffer.position(currentPos);


    AppendMessageResult result;
    if (messageExt instanceof MessageExtBrokerInner) {

      //加到了MappedFile对象的ByteBuffer中
      result = cb.doAppend(this.getFileFromOffset(), byteBuffer, this.fileSize - currentPos, (MessageExtBrokerInner) messageExt);
    } else if (messageExt instanceof MessageExtBatch) {
      result = cb.doAppend(this.getFileFromOffset(), byteBuffer, this.fileSize - currentPos, (MessageExtBatch) messageExt);
    } else {
      return new AppendMessageResult(AppendMessageStatus.UNKNOWN_ERROR);
    }
    this.wrotePosition.addAndGet(result.getWroteBytes());
    this.storeTimestamp = result.getStoreTimestamp();
    return result;
  }


  //-------------文件已满，不可写入------------
  //没有足够的空间写入
  log.error("MappedFile.appendMessage return null, wrotePosition: {} fileSize: {}", currentPos, this.fileSize);
  return new AppendMessageResult(AppendMessageStatus.UNKNOWN_ERROR);
}
```



#### AppendMessageCallback

##### doAppend

```java
/**
         * 将消息存储到ByteBuffer中
         * @param fileFromOffset
         * @param byteBuffer
         * @param maxBlank
         * @param msgInner
         * @return
         */
public AppendMessageResult doAppend(final long fileFromOffset, final ByteBuffer byteBuffer, final int maxBlank,
                                    final MessageExtBrokerInner msgInner) {
  // STORETIMESTAMP + STOREHOSTADDRESS + OFFSET <br>


  //---------------------1. 生成消息的唯一ID--------------------
  // PHY OFFSET
  long wroteOffset = fileFromOffset + byteBuffer.position();

  int sysflag = msgInner.getSysFlag();

  int bornHostLength = (sysflag & MessageSysFlag.BORNHOST_V6_FLAG) == 0 ? 4 + 4 : 16 + 4;
  int storeHostLength = (sysflag & MessageSysFlag.STOREHOSTADDRESS_V6_FLAG) == 0 ? 4 + 4 : 16 + 4;
  ByteBuffer bornHostHolder = ByteBuffer.allocate(bornHostLength);
  ByteBuffer storeHostHolder = ByteBuffer.allocate(storeHostLength);

  this.resetByteBuffer(storeHostHolder, storeHostLength);


  //-------------创建消息唯一msgId-------------
  //创建消息的唯一ID，分别由三部分组成：4字节IP  4字节端口号  8字节消息偏移量
  //唯一ID中有偏移量，可以很好的获取到偏移量信息
  String msgId;
  if ((sysflag & MessageSysFlag.STOREHOSTADDRESS_V6_FLAG) == 0) {

    //底层使用工具类UtilAll，可以使用UtilAll.string2bytes获取消息偏移量
    msgId = MessageDecoder.createMessageId(this.msgIdMemory, msgInner.getStoreHostBytes(storeHostHolder), wroteOffset);
  } else {
    msgId = MessageDecoder.createMessageId(this.msgIdV6Memory, msgInner.getStoreHostBytes(storeHostHolder), wroteOffset);
  }


  //---------------------2. 记录该消息在消息队列中的偏移量--------------------
  // Record ConsumeQueue information
  keyBuilder.setLength(0);
  keyBuilder.append(msgInner.getTopic());
  keyBuilder.append('-');
  keyBuilder.append(msgInner.getQueueId());
  String key = keyBuilder.toString();

  //像是一个索引下标，有数据就增加1，从0开始
  Long queueOffset = CommitLog.this.topicQueueTable.get(key);
  if (null == queueOffset) {
    queueOffset = 0L;

    //CommitLog中保存了当前所有消息队列的当前待写入偏移量
    CommitLog.this.topicQueueTable.put(key, queueOffset);
  }




  // Transaction messages that require special handling
  final int tranType = MessageSysFlag.getTransactionValue(msgInner.getSysFlag());
  switch (tranType) {
      // Prepared and Rollback message is not consumed, will not enter the
      // consumer queuec
    case MessageSysFlag.TRANSACTION_PREPARED_TYPE:
    case MessageSysFlag.TRANSACTION_ROLLBACK_TYPE:
      queueOffset = 0L;
      break;
    case MessageSysFlag.TRANSACTION_NOT_TYPE:
    case MessageSysFlag.TRANSACTION_COMMIT_TYPE:
    default:
      break;
  }

  /**
             * Serialize message
             */
  final byte[] propertiesData =
    msgInner.getPropertiesString() == null ? null : msgInner.getPropertiesString().getBytes(MessageDecoder.CHARSET_UTF8);

  final int propertiesLength = propertiesData == null ? 0 : propertiesData.length;

  if (propertiesLength > Short.MAX_VALUE) {
    log.warn("putMessage message properties length too long. length={}", propertiesData.length);
    return new AppendMessageResult(AppendMessageStatus.PROPERTIES_SIZE_EXCEEDED);
  }

  final byte[] topicData = msgInner.getTopic().getBytes(MessageDecoder.CHARSET_UTF8);
  final int topicLength = topicData.length;

  final int bodyLength = msgInner.getBody() == null ? 0 : msgInner.getBody().length;


  //---------------------3. 写入Buffer中--------------------
  //根据消息体的长度、主题的长度、属性的长度 结合消息存储格式计算消息的总长度
  final int msgLen = calMsgLength(msgInner.getSysFlag(), bodyLength, topicLength, propertiesLength);

  //3.1 如果长度大于CommitLog文件的空闲空间，则报错
  // Exceeds the maximum message
  if (msgLen > this.maxMessageSize) {
    CommitLog.log.warn("message size exceeded, msg total size: " + msgLen + ", msg body size: " + bodyLength
                       + ", maxMessageSize: " + this.maxMessageSize);
    return new AppendMessageResult(AppendMessageStatus.MESSAGE_SIZE_EXCEEDED);
  }


  //3.2 没有足够的空间写入
  // Determines whether there is sufficient free space
  if ((msgLen + END_FILE_MIN_BLANK_LENGTH) > maxBlank) {
    this.resetByteBuffer(this.msgStoreItemMemory, maxBlank);
    // 1 TOTALSIZE
    this.msgStoreItemMemory.putInt(maxBlank);
    // 2 MAGICCODE
    this.msgStoreItemMemory.putInt(CommitLog.BLANK_MAGIC_CODE);
    // 3 The remaining space may be any value
    // Here the length of the specially set maxBlank
    final long beginTimeMills = CommitLog.this.defaultMessageStore.now();

    //将消息内容存储到ByteBuffer中，然后创建AppendMessageResult。
    //此处只是将消息存储在MapperFile对应的内存映射Buffer中，并没有刷写到磁盘
    byteBuffer.put(this.msgStoreItemMemory.array(), 0, maxBlank);

    //返回END_OF_FILE退出，Broker会重新创建一个新的CommitLog文件来存储消息
    return new AppendMessageResult(AppendMessageStatus.END_OF_FILE, wroteOffset, maxBlank, msgId, msgInner.getStoreTimestamp(),
                                   queueOffset, CommitLog.this.defaultMessageStore.now() - beginTimeMills);
  }


  //-------------------计算消息的存储空间----------------
  // Initialization of storage space
  this.resetByteBuffer(msgStoreItemMemory, msgLen);
  // 1 TOTALSIZE
  this.msgStoreItemMemory.putInt(msgLen);
  // 2 MAGICCODE
  this.msgStoreItemMemory.putInt(CommitLog.MESSAGE_MAGIC_CODE);
  // 3 BODYCRC
  this.msgStoreItemMemory.putInt(msgInner.getBodyCRC());
  // 4 QUEUEID
  this.msgStoreItemMemory.putInt(msgInner.getQueueId());
  // 5 FLAG
  this.msgStoreItemMemory.putInt(msgInner.getFlag());
  // 6 QUEUEOFFSET
  this.msgStoreItemMemory.putLong(queueOffset);
  // 7 PHYSICALOFFSET
  this.msgStoreItemMemory.putLong(fileFromOffset + byteBuffer.position());
  // 8 SYSFLAG
  this.msgStoreItemMemory.putInt(msgInner.getSysFlag());
  // 9 BORNTIMESTAMP
  this.msgStoreItemMemory.putLong(msgInner.getBornTimestamp());
  // 10 BORNHOST
  this.resetByteBuffer(bornHostHolder, bornHostLength);
  this.msgStoreItemMemory.put(msgInner.getBornHostBytes(bornHostHolder));
  // 11 STORETIMESTAMP
  this.msgStoreItemMemory.putLong(msgInner.getStoreTimestamp());
  // 12 STOREHOSTADDRESS
  this.resetByteBuffer(storeHostHolder, storeHostLength);
  this.msgStoreItemMemory.put(msgInner.getStoreHostBytes(storeHostHolder));
  // 13 RECONSUMETIMES
  this.msgStoreItemMemory.putInt(msgInner.getReconsumeTimes());
  // 14 Prepared Transaction Offset
  this.msgStoreItemMemory.putLong(msgInner.getPreparedTransactionOffset());
  // 15 BODY
  this.msgStoreItemMemory.putInt(bodyLength);
  if (bodyLength > 0)
    this.msgStoreItemMemory.put(msgInner.getBody());
  // 16 TOPIC
  this.msgStoreItemMemory.put((byte) topicLength);
  this.msgStoreItemMemory.put(topicData);
  // 17 PROPERTIES
  this.msgStoreItemMemory.putShort((short) propertiesLength);
  if (propertiesLength > 0)
    this.msgStoreItemMemory.put(propertiesData);

  final long beginTimeMills = CommitLog.this.defaultMessageStore.now();
  // Write messages to the queue buffer
  byteBuffer.put(this.msgStoreItemMemory.array(), 0, msgLen);

  AppendMessageResult result = new AppendMessageResult(AppendMessageStatus.PUT_OK, wroteOffset, msgLen, msgId,
                                                       msgInner.getStoreTimestamp(), queueOffset, CommitLog.this.defaultMessageStore.now() - beginTimeMills);


  //---------------------4. 更新消息队列逻辑偏移量--------------------
  switch (tranType) {
    case MessageSysFlag.TRANSACTION_PREPARED_TYPE:
    case MessageSysFlag.TRANSACTION_ROLLBACK_TYPE:
      break;
    case MessageSysFlag.TRANSACTION_NOT_TYPE:
    case MessageSysFlag.TRANSACTION_COMMIT_TYPE:
      // The next update ConsumeQueue information
      //TODO 消息消息队列逻辑偏移量 有啥用呢？？？？  topic-key
      CommitLog.this.topicQueueTable.put(key, ++queueOffset);
      break;
    default:
      break;
  }
  return result;
}
```

	###### MessageDecoder

​	* 消息ID组成

<img src="../../../../照片/typora/image-20210804135150286.png" alt="image-20210804135150286" style="zoom:50%;" />

```java
public static String createMessageId(final ByteBuffer input, final ByteBuffer addr, final long offset) {
  input.flip();
  int msgIDLength = addr.limit() == 8 ? 16 : 28;
  input.limit(msgIDLength);

  input.put(addr);
  input.putLong(offset);

  return UtilAll.bytes2string(input.array());
}
```





### 存储文件组织与内存映射

* RocketMQ通过使用内存映射文件来提高IO访问性能，无论是CommitLog，ConsumeQueue还是IndexFile，单个文件都被设计为固定长度，如果一个文件写满以后再创建一个新文件，文件名为该文件**第一条消息对应的全局物理偏移量**

* RocketMQ使用MappedFile、MappedFileQueue来封装存储文件

  <img src="../../../../照片/typora/image-20210805094418230.png" alt="image-20210805094418230" style="zoom:50%;" />

#### MappedFileQueue映射文件队列

* 类图

  <img src="../../../../照片/typora/image-20210805095330340.png" alt="image-20210805095330340" style="zoom:50%;" />

* 属性

  ```java
  public class MappedFileQueue {
      private static final InternalLogger log = InternalLoggerFactory.getLogger(LoggerName.STORE_LOGGER_NAME);
      private static final InternalLogger LOG_ERROR = InternalLoggerFactory.getLogger(LoggerName.STORE_ERROR_LOGGER_NAME);
  
      private static final int DELETE_FILES_BATCH_MAX = 10;
  
      //存储目录
      private final String storePath;
  
      //单个文件的存储大小
      private final int mappedFileSize;
  
      //MappedFile文件集合
      private final CopyOnWriteArrayList<MappedFile> mappedFiles = new CopyOnWriteArrayList<MappedFile>();
  
      //创建MappedFile服务类
      private final AllocateMappedFileService allocateMappedFileService;
  
      //当前刷盘指针，表示该指针之前的所有数据全部持久化到磁盘
      private long flushedWhere = 0;
  
      //当前数据提交指针，内存中ByteBuffer当前的写指针，该值大于等于flushedWhere
      private long committedWhere = 0;
  
      private volatile long storeTimestamp = 0;
    
      ...
  }  
  ```

  

##### 不同维度查询的方法

* getMappedFileByTime

  * 根据消息存储时间来查找MappedFile

    ```java
    //原理：从MappedFile列表中第一个文件开始查找
    //找到第一个最后一次更新时间大于待查找时间戳的文件，如果不存在，就返回最后一个MappedFile文件
    public MappedFile getMappedFileByTime(final long timestamp) {
      Object[] mfs = this.copyMappedFiles(0);
    
      if (null == mfs)
        return null;
    
      for (int i = 0; i < mfs.length; i++) {
        MappedFile mappedFile = (MappedFile) mfs[i];
    
    
        if (mappedFile.getLastModifiedTimestamp() >= timestamp) {
          return mappedFile;
        }
      }
    
      return (MappedFile) mfs[mfs.length - 1];
    }
    ```

* findMappedFileByOffset

  * 根据消息偏移量offset查找MappedFile

    * 由于RocketMQ采取定时删除存储文件的策略，意味着文件中第一个文件并不是一开始创建的最初的那个开始文件，而是删除后的文件，所以，查找的应该是当前偏移量相对于第一个文件的偏移量，然后除以文件大小，获取到哪个文件中，既可以获取到文件

    ```java
    public MappedFile findMappedFileByOffset(final long offset, final boolean returnFirstOnNotFound) {
      try {
        MappedFile firstMappedFile = this.getFirstMappedFile();
        MappedFile lastMappedFile = this.getLastMappedFile();
        if (firstMappedFile != null && lastMappedFile != null) {
    
          //告警提示
          if (offset < firstMappedFile.getFileFromOffset() || offset >= lastMappedFile.getFileFromOffset() + this.mappedFileSize) {
            LOG_ERROR.warn("Offset not matched. Request offset: {}, firstOffset: {}, lastOffset: {}, mappedFileSize: {}, mappedFiles count: {}",
                           offset,
                           firstMappedFile.getFileFromOffset(),
                           lastMappedFile.getFileFromOffset() + this.mappedFileSize,
                           this.mappedFileSize,
                           this.mappedFiles.size());
          } else {//真正寻找的地方，index为哪个文件
            int index = (int) ((offset / this.mappedFileSize) - (firstMappedFile.getFileFromOffset() / this.mappedFileSize));
            MappedFile targetFile = null;
            try {
              targetFile = this.mappedFiles.get(index);
            } catch (Exception ignored) {
            }
    
            //   from   offset    fileSize
            if (targetFile != null && offset >= targetFile.getFileFromOffset()
                && offset < targetFile.getFileFromOffset() + this.mappedFileSize) {
              return targetFile;
            }
    
    
            //重新在此获取
            for (MappedFile tmpMappedFile : this.mappedFiles) {
              if (offset >= tmpMappedFile.getFileFromOffset()
                  && offset < tmpMappedFile.getFileFromOffset() + this.mappedFileSize) {
                return tmpMappedFile;
              }
            }
          }
    
          if (returnFirstOnNotFound) {
            return firstMappedFile;
          }
        }
      } catch (Exception e) {
        log.error("findMappedFileByOffset Exception", e);
      }
    
      return null;
    }
    ```

    

#### MappedFile内存映射文件

* 用于提高IO性能
  * 问题：怎么提高呢？？？

* MappedFile是RocketMQ内存映射文件的具体实现

* 类图

  <img src="../../../../照片/typora/image-20210805102221212.png" alt="image-20210805102221212" style="zoom:50%;" />

* 属性值

  ```java
  /**
   * MappedFile是RocketMQ内存映射文件的具体实现
   */
  public class MappedFile extends ReferenceResource {
  
      //操作系统每页大小，默认4K
      public static final int OS_PAGE_SIZE = 1024 * 4;
      protected static final InternalLogger log = InternalLoggerFactory.getLogger(LoggerName.STORE_LOGGER_NAME);
  
      //当前JVM实例中MappedFile虚拟内存
      private static final AtomicLong TOTAL_MAPPED_VIRTUAL_MEMORY = new AtomicLong(0);
  
      //当前JVM实例中MappedFile对象个数
      private static final AtomicInteger TOTAL_MAPPED_FILES = new AtomicInteger(0);
  
      //当前该文件的写位置
      protected final AtomicInteger wrotePosition = new AtomicInteger(0);
  
      //当前该文件的提交指针，如果开启transientStorePoolEnable ，则数据会存储在TransientStorePool中，然后提交到内存映射ByteBuffer中，再刷新到磁盘
      protected final AtomicInteger committedPosition = new AtomicInteger(0);
  
      //刷写到磁盘指针，该指针之前的数据持久化到磁盘中
      private final AtomicInteger flushedPosition = new AtomicInteger(0);
  
      //文件大小
      protected int fileSize;
  
      //文件通道
      protected FileChannel fileChannel;
      /**
       * Message will put to here first, and then reput to FileChannel if writeBuffer is not null.
       * 堆内存ByteBuffer，如果不为空，数据首先将存储到该Buffer中，然后提交到MappedFile对应的内存映射文件Buffer。
       */
      protected ByteBuffer writeBuffer = null;
  
      //堆内存池，transientStorePoolEnable = true是启用，有啥用呢？？？？
      protected TransientStorePool transientStorePool = null;
  
      //文件名称
      private String fileName;
  
      //该文件的初始偏移量
      private long fileFromOffset;
  
      //物理文件
      private File file;
  
      //物理文件对应的内存映射Buffer
      private MappedByteBuffer mappedByteBuffer;
  
      //文件最后一次内容写入时间
      private volatile long storeTimestamp = 0;
  
      //是否是MappedFileQueue队列中第一个文件
      private boolean firstCreateInQueue = false;
  }
  ```

##### MappedFile初始化

* 根据是否开启transientStorePoolEnable存在两种初始化情况

  * transientStorePoolEnable为true表示	

    * 内容先存储在对外内存，然后通过Commit线程将数据提交到内存映射Buffer中，再通过Flush线程将内存映射Buffer中的数据持久化到磁盘中

    ```java
    public void init(final String fileName, final int fileSize,
                     final TransientStorePool transientStorePool) throws IOException {
      init(fileName, fileSize);
      //如果 transientStorePoolEnable 为 true,则初始化MappedFile的writeBuffer，该buffer从transientStorePool申请过来
      this.writeBuffer = transientStorePool.borrowBuffer();
      this.transientStorePool = transientStorePool;
    }
    
    
    
    
    private void init(final String fileName, final int fileSize) throws IOException {
      this.fileName = fileName;
      this.fileSize = fileSize;
      this.file = new File(fileName);
    
      //文件名称代表该文件的起始偏移量
      this.fileFromOffset = Long.parseLong(this.file.getName());
      boolean ok = false;
    
      ensureDirOK(this.file.getParent());
    
      try {
        
        //-----------------将磁盘中的文件传输到内存映射中-----------
        //文件通道，通过RandomAccessFile创建读写文件通道
        this.fileChannel = new RandomAccessFile(this.file, "rw").getChannel();
        //将文件内容使用NIO的内存映射Buffer将文件映射到内存中
        this.mappedByteBuffer = this.fileChannel.map(MapMode.READ_WRITE, 0, fileSize);
        TOTAL_MAPPED_VIRTUAL_MEMORY.addAndGet(fileSize);
        TOTAL_MAPPED_FILES.incrementAndGet();
        ok = true;
      } catch (FileNotFoundException e) {
        log.error("Failed to create file " + this.fileName, e);
        throw e;
      } catch (IOException e) {
        log.error("Failed to map file " + this.fileName, e);
        throw e;
      } finally {
        if (!ok && this.fileChannel != null) {
          this.fileChannel.close();
        }
      }
    }
    ```

##### MappedFile提交

* 内存映射文件的提交动作由MappedFile的commit方法实现

* commit方法

  ```java
  /**
       * 提交，操作主题是writeBuffer
       * 作用：将MappedFild#writeBuffer中的数据提交到文件通道FileChannel中
       * @param commitLeastPages 本次提交最小的页数
       * @return
       */
  public int commit(final int commitLeastPages) {
    //受transientStorePool决定
    if (writeBuffer == null) {
      //no need to commit data to file channel, so just regard wrotePosition as committedPosition.
      return this.wrotePosition.get();
    }
  
    //可以执行提交
    if (this.isAbleToCommit(commitLeastPages)) {
      if (this.hold()) {
        //真正提交的代码
        commit0(commitLeastPages);
        this.release();
      } else {
        log.warn("in commit, hold failed, commit offset = " + this.committedPosition.get());
      }
    }
    //如果不可以提交，则直接返回当前提交的位置，待下次提交
  
    // All dirty data has been committed to FileChannel.
    if (writeBuffer != null && this.transientStorePool != null && this.fileSize == this.committedPosition.get()) {
      this.transientStorePool.returnBuffer(writeBuffer);
      this.writeBuffer = null;
    }
  
  
    //返回最后可以提交的位置
    return this.committedPosition.get();
  }
  ```

* isAbleToCommit

  ```java
  /**
       * 判断是否执行commit操作
       * @param commitLeastPages
       * @return
       */
  protected boolean isAbleToCommit(final int commitLeastPages) {
    int flush = this.committedPosition.get();
    int write = this.wrotePosition.get();
  
    //如果文件满了，真正存储数据的文件
    if (this.isFull()) {
      return true;
    }
  
    //比较当前脏页的数量，如果大于提交的最小页数，就返回true
    if (commitLeastPages > 0) {
      //比较wrotePosition(当前writeBuffer的写指针)与上一次提交的指针(commitedPosition)的差值
      //除以 OS_PAGE_SIZE得到当前脏页的数量
      return ((write / OS_PAGE_SIZE) - (flush / OS_PAGE_SIZE)) >= commitLeastPages;
    }
  
    //commitLeastPages小于0，只要存在脏页就提交
    return write > flush;
  }
  ```

* Commit0()方法

  * commit的作用就是将MappedFile#writeBuffer中的数据提交到文件通道FileChannel中

  ```java
  protected void commit0(final int commitLeastPages) {
    int writePos = this.wrotePosition.get();
    int lastCommittedPosition = this.committedPosition.get();
  
    if (writePos - this.committedPosition.get() > 0) {
      try {
        //创建writeBuffer的共享缓存区
        //slice()方法创建一个共享缓存区，与原先的ByteBuffer共享内存，但是维护一套独立的指针(position,mark,limit)
        ByteBuffer byteBuffer = writeBuffer.slice();
  
        //新创建的position回退到上一次提交的位置
        byteBuffer.position(lastCommittedPosition);
  
        //设置当前最大有效数据指针为写入位置的偏移量
        byteBuffer.limit(writePos);
  
        //把commitedPosition 到 writePosition的数据复制到FileChannel中，然后更新commitedPosition指针为wrotePosition
        this.fileChannel.position(lastCommittedPosition);
        this.fileChannel.write(byteBuffer);
  
        //更新位置
        this.committedPosition.set(writePos);
      } catch (Throwable e) {
        log.error("Error occurred when commit data to FileChannel.", e);
      }
    }
  }
  ```

##### MappedFile刷盘

* 刷盘指的是将内存中的数据刷写到磁盘，永久存储在磁盘中，具体实现由flush实现

* flush

  ```java
  /**
       * 将内存中的数据刷写到磁盘，永久存储在磁盘中
       * @return The current flushed position
       */
  public int flush(final int flushLeastPages) {
    if (this.isAbleToFlush(flushLeastPages)) {
      if (this.hold()) {
  
        //获取MappedFile的最大读指针
        int value = getReadPosition();
  
        try {
          //We only append data to fileChannel or mappedByteBuffer, never both.
          if (writeBuffer != null || this.fileChannel.position() != 0) {
  
            //刷新到磁盘
            this.fileChannel.force(false);
          } else {
  
            //刷新到磁盘
            this.mappedByteBuffer.force();
          }
        } catch (Throwable e) {
          log.error("Error occurred when force data to disk.", e);
        }
  
        this.flushedPosition.set(value);
        this.release();
      } else {
        log.warn("in flush, hold failed, flush offset = " + this.flushedPosition.get());
        this.flushedPosition.set(getReadPosition());
      }
    }
    return this.getFlushedPosition();
  }
  ```

##### 获取MappedFile最大读指针

* RocketMQ文件的一个组织方式是内存映射文件，预先申请一块连续的固定大小的内存，需要一套指针标识当前最大有效数据的位置，获取最大有效数据偏移量的方法由MappedFile的getReadPosition方法实现

* getReadPosition

  * 如果writeBuffer为空，则直接返回当前的写指针
  * 如果writeBuffer不为空，则返回上一次提交的指针。在MappedFile设计中，只有提交了的数据（写入到MappedByteBuffer或FileChannel中的数据）才是安全的数据。

  ```java
  public int getReadPosition() {
    return this.writeBuffer == null ? this.wrotePosition.get() : this.committedPosition.get();
  }
  ```



##### MappedFile销毁

* MappedFile文件销毁的实现方式为destroy(long intervalForcibly)，intervalForcibly表示拒绝被销毁的最大存活时间

* destroy

  ```java
  public boolean destroy(final long intervalForcibly) {
  
    //关闭MappedFile
    this.shutdown(intervalForcibly);
  
  
    //判断是否清理完成
    if (this.isCleanupOver()) {
      try {
  
        //关闭通道
        this.fileChannel.close();
        log.info("close file channel " + this.fileName + " OK");
  
        long beginTime = System.currentTimeMillis();
  
        //删除物理文件
        boolean result = this.file.delete();
        log.info("delete file[REF:" + this.getRefCount() + "] " + this.fileName
                 + (result ? " OK, " : " Failed, ") + "W:" + this.getWrotePosition() + " M:"
                 + this.getFlushedPosition() + ", "
                 + UtilAll.computeElapsedTimeMilliseconds(beginTime));
      } catch (Exception e) {
        log.warn("close file channel " + this.fileName + " Failed. ", e);
      }
  
      return true;
    } else {
      log.warn("destroy mapped file[REF:" + this.getRefCount() + "] " + this.fileName
               + " Failed. cleanupOver: " + this.cleanupOver);
    }
  
    return false;
  }
  
  
  //判断标准是引用次数小于等于0并且cleanupover为true
  //cleanupover为true的条件是release成功将MappedByteBuffer资源释放掉
  public boolean isCleanupOver() {
    return this.refCount.get() <= 0 && this.cleanupOver;
  }
  ```

  * ReferenceResource#shutdown

    ```java
    //关闭MappedFile
    public void shutdown(final long intervalForcibly) {
      //初次调用 available为true
      if (this.available) {
        //设置为false，并设置初次关闭的时间戳(firstShutdownTimestamp)为当前时间戳
        this.available = false;
        this.firstShutdownTimestamp = System.currentTimeMillis();
    
        //调用release释放资源，只有在引用次数小于1的情况下才会释放资源
        this.release();
      } else if (this.getRefCount() > 0) { //如果引用次数大于0，且比当前时间与firstShutdownTimestamp，如果已经超过了其最大拒绝存活期
        //每执行一次，将引用数减少1000，知道应用数小于0通过执行realse方法释放资源
        if ((System.currentTimeMillis() - this.firstShutdownTimestamp) >= intervalForcibly) {
          this.refCount.set(-1000 - this.getRefCount());
          this.release();
        }
      }
    }
    ```

  * ReferenceResource#release

    ```java
    //countDownLatch的使用
    //如果没有应用，那么久<=0，执行cleanUp方法
    public void release() {
      long value = this.refCount.decrementAndGet();
      if (value > 0)
        return;
    
      synchronized (this) {
    
        this.cleanupOver = this.cleanup(value);
      }
    }
    ```

  * ReferenceResource#cleanup

    ```java
    @Override
    public boolean cleanup(final long currentRef) {
    
      //MappedFile当前可用，不进行删除
      if (this.isAvailable()) {
        log.error("this file[REF:" + currentRef + "] " + this.fileName
                  + " have not shutdown, stop unmapping.");
        return false;
      }
    
      //已经清理，则无需重新清理
      if (this.isCleanupOver()) {
        log.error("this file[REF:" + currentRef + "] " + this.fileName
                  + " have cleanup, do not do it again.");
        return true;
      }
    
      //堆外内存，调用堆外内存的cleanup进行处理
      //维护MappedFile类变量
      clean(this.mappedByteBuffer);
      TOTAL_MAPPED_VIRTUAL_MEMORY.addAndGet(this.fileSize * (-1)); //mappedFile有效数据的内存大小
      TOTAL_MAPPED_FILES.decrementAndGet();//mappedFile的有效文件数据
      log.info("unmap file[REF:" + currentRef + "] " + this.fileName + " OK");
      return true;
    }
    ```

    

##### TransientStorePool

* 含义：短暂的存储池

* 作用：

  * 用来临时存储数据，数据先写入该内存映射中，然后由commit线程定时将数据从该内存复制到目的物理文件对应的内存映射中MappedFile

* 核心属性

  ```java
  public class TransientStorePool {
      private static final InternalLogger log = InternalLoggerFactory.getLogger(LoggerName.STORE_LOGGER_NAME);
  
      //availableBuffers 个数
      private final int poolSize;
  
      //每个ByteBuffer大小
      private final int fileSize;
  
      //ByteBuffer容器
      private final Deque<ByteBuffer> availableBuffers;
    
      public void init() {
          //创建poolSize个堆外内存
          for (int i = 0; i < poolSize; i++) {
              ByteBuffer byteBuffer = ByteBuffer.allocateDirect(fileSize);
  
              final long address = ((DirectBuffer) byteBuffer).address();
              Pointer pointer = new Pointer(address);
  
              //锁定内存，避免被置换到交换区
              LibC.INSTANCE.mlock(pointer, new NativeLong(fileSize));
  
              availableBuffers.offer(byteBuffer);
          }
      } 
    
      ....
        
        
  }  
  ```

  

### RocketMQ文件

#### CommitLog文件

* commitlog目录下，主要用于存储消息

* 该目录下的文件主要存储消息，特点

  * 每一条消息长度不相同
  * 每条消息的前4个字节存储该消息的总长度

* 消息组织方式

  <img src="../../../../照片/typora/image-20210806170937690.png" alt="image-20210806170937690" style="zoom:50%;" />



##### 重要方法

###### 获取commitlog文件夹下最小偏移量

* 代码

  ```java
  /**
       * 获取当前CommitLog目录最小偏移量
       * @return
       */
  public long getMinOffset() {
  
    //获取第一个MappedFile文件
    MappedFile mappedFile = this.mappedFileQueue.getFirstMappedFile();
    if (mappedFile != null) {
      if (mappedFile.isAvailable()) {
  
        //获取第一个文件的起始偏移量
        return mappedFile.getFileFromOffset();
      } else {
        return this.rollNextFile(mappedFile.getFileFromOffset());
      }
    }
  
    return -1;
  }
  ```




###### 根据offset返回下一个文件的起始偏移量

* 原理：

  * 当前内容的绝对偏移量 - offset% mappedFileSize（相对于当前MappedFile的相对偏移量），这样就获取了当前内容所在文件的起始绝对偏移量，下一个就加文件大小就好了，因为CommitLog文件大小是固定的

* 源码

  ```java
  public long rollNextFile(final long offset) {
    int mappedFileSize = this.defaultMessageStore.getMessageStoreConfig().getMappedFileSizeCommitLog();
    return offset + mappedFileSize - offset % mappedFileSize;
  }
  ```



###### 根据偏移量和消息长度查找消息

* 原理

  * 根据偏移量找到当前内容在commitLog中的相对偏移量，然后通过索引和消息大小，相对偏移量获取内容

* 源码

  ```java
  /**
       * 根据偏移量和消息长度查找消息
       * @param offset
       * @param size
       * @return
       */
  public SelectMappedBufferResult getMessage(final long offset, final int size) {
    int mappedFileSize = this.defaultMessageStore.getMessageStoreConfig().getMappedFileSizeCommitLog();
  
    //找到偏移量的文件
    MappedFile mappedFile = this.mappedFileQueue.findMappedFileByOffset(offset, offset == 0);
  
    if (mappedFile != null) {
      //相对偏移量
      int pos = (int) (offset % mappedFileSize);
      return mappedFile.selectMappedBuffer(pos, size);
    }
    return null;
  }
  
  
  
  
  public MappedFile findMappedFileByOffset(final long offset, final boolean returnFirstOnNotFound) {
    try {
      MappedFile firstMappedFile = this.getFirstMappedFile();
      MappedFile lastMappedFile = this.getLastMappedFile();
      if (firstMappedFile != null && lastMappedFile != null) {
  
        //告警提示
        if (offset < firstMappedFile.getFileFromOffset() || offset >= lastMappedFile.getFileFromOffset() + this.mappedFileSize) {
          LOG_ERROR.warn("Offset not matched. Request offset: {}, firstOffset: {}, lastOffset: {}, mappedFileSize: {}, mappedFiles count: {}",
                         offset,
                         firstMappedFile.getFileFromOffset(),
                         lastMappedFile.getFileFromOffset() + this.mappedFileSize,
                         this.mappedFileSize,
                         this.mappedFiles.size());
        } else {//真正寻找的地方，index为哪个文件
          //原因，因为第一个文件不一定是index为0的，因为会删除
          int index = (int) ((offset / this.mappedFileSize) - (firstMappedFile.getFileFromOffset() / this.mappedFileSize));
          MappedFile targetFile = null;
          try {
            targetFile = this.mappedFiles.get(index);
          } catch (Exception ignored) {
          }
  
          //   from   offset    fileSize
          if (targetFile != null && offset >= targetFile.getFileFromOffset()
              && offset < targetFile.getFileFromOffset() + this.mappedFileSize) {
            return targetFile;
          }
  
  
          //重新在此获取
          for (MappedFile tmpMappedFile : this.mappedFiles) {
            if (offset >= tmpMappedFile.getFileFromOffset()
                && offset < tmpMappedFile.getFileFromOffset() + this.mappedFileSize) {
              return tmpMappedFile;
            }
          }
        }
  
        if (returnFirstOnNotFound) {
          return firstMappedFile;
        }
      }
    } catch (Exception e) {
      log.error("findMappedFileByOffset Exception", e);
    }
  
    return null;
  }
  ```



#### ConsumeQueue文件

* 原因

  * RocketMQ为了适应消息消费的检索需求，因为同一个主题的消息不连续的存储在commitLog文件中，那么如果消费消息直接从消息存储文件（commitlog）中去遍历查找订阅主题下的消息，效率极其低下。所以，所以设计了消息消费队列文件(ConsumeQueue)

* 该文件可以看成是CommitLog关于消息消费的”索引“文件

  ![image-20210809163424864](../../../../照片/typora/image-20210809163424864.png)

  * consumequeu的第一级目录为消息主题
  * consumequeu的第二级目录为消息队列

* 存储格式：为了加速检索速度和节省磁盘空间，不会包含全量信息

  <img src="../../../../照片/typora/image-20210809163701769.png" alt="image-20210809163701769" style="zoom:50%;" />

* ConsumeQueue特点
  * 默认包含30万个条目
  * 单个文件的长度为30w * 20字节
  * 单个ConsumeQueue文件可以看成是一个ConsumeQueue条目的数组，其下标为ConsumeQueue的逻辑偏移量。重点：消息消费进度存储的偏移量即逻辑偏移量，可以理解为当前consumequeue的索引下标
  * ConsumeQueue即为CommitLog文件的索引文件，其构建机制是当消息到达CommitLog文件后，由专门的线程产生消息转发任务，从而构建消息消费队列文件与index文件



##### 重要方法

###### 根据消息逻辑偏移量（相当于索引下标）获取消息

* 消息的逻辑偏移量，即在consumequeue文件中的索引下标

* 原理

  * 首先获取在当前consumequeue中的物理偏移量，即相对物理偏移量
  * 问题：
    * 如何通过consumequeue来获取

* 代码

  ```java
  //如何从 consumequeue然后获取对应的mappedFile文件
  public SelectMappedBufferResult getIndexBuffer(final long startIndex) {
  
    int mappedFileSize = this.mappedFileSize;
  
    //得到在ConsumeQueue中的物理偏移量,相对偏移量
    long offset = startIndex * CQ_STORE_UNIT_SIZE;
  
    //如果消息偏移量大于文件的最小偏移量，表示没有被清理
    if (offset >= this.getMinLogicOffset()) {
  
      //根据偏移量获取具体的哪个文件,offset获取的是相对于第一个文件的相对偏移量，相对于0000000000000，即使当前可能000000000文件不存在，但是偏移量是相对于此的偏移量
      MappedFile mappedFile = this.mappedFileQueue.findMappedFileByOffset(offset);
      if (mappedFile != null) {
        //通过offset与物理大小取模获取该文件的偏移量
        SelectMappedBufferResult result = mappedFile.selectMappedBuffer((int) (offset % mappedFileSize));
        return result;
      }
    }
  
    //小于最小偏移量，说明消息已经被清理
    return null;
  }
  ```

  * MappedFileQueue#findMappedFileByOffset

  ```java
  //@param offset Offset. 相对于当前文件夹的相对偏移量，怎么理解呢，文件第一个相对偏移量是0000000000000，后面都是在这个基础上进行增加
  public MappedFile findMappedFileByOffset(final long offset, final boolean returnFirstOnNotFound) {
    try {
      MappedFile firstMappedFile = this.getFirstMappedFile();
      MappedFile lastMappedFile = this.getLastMappedFile();
      if (firstMappedFile != null && lastMappedFile != null) {
  
        //告警提示
        if (offset < firstMappedFile.getFileFromOffset() || offset >= lastMappedFile.getFileFromOffset() + this.mappedFileSize) {
          LOG_ERROR.warn("Offset not matched. Request offset: {}, firstOffset: {}, lastOffset: {}, mappedFileSize: {}, mappedFiles count: {}",
                         offset,
                         firstMappedFile.getFileFromOffset(),
                         lastMappedFile.getFileFromOffset() + this.mappedFileSize,
                         this.mappedFileSize,
                         this.mappedFiles.size());
        } else {//真正寻找的地方，index为哪个文件
          int index = (int) ((offset / this.mappedFileSize) - (firstMappedFile.getFileFromOffset() / this.mappedFileSize));
          MappedFile targetFile = null;
          try {
            targetFile = this.mappedFiles.get(index);
          } catch (Exception ignored) {
          }
  
          //   from   offset    fileSize
          if (targetFile != null && offset >= targetFile.getFileFromOffset()
              && offset < targetFile.getFileFromOffset() + this.mappedFileSize) {
            return targetFile;
          }
  
  
          //重新在此获取
          for (MappedFile tmpMappedFile : this.mappedFiles) {
            if (offset >= tmpMappedFile.getFileFromOffset()
                && offset < tmpMappedFile.getFileFromOffset() + this.mappedFileSize) {
              return tmpMappedFile;
            }
          }
        }
  
        if (returnFirstOnNotFound) {
          return firstMappedFile;
        }
      }
    } catch (Exception e) {
      log.error("findMappedFileByOffset Exception", e);
    }
  
    return null;
  }
  ```

###### 根据消息存储时间来查找消息

* getOffsetInQueueByTime(final long timestamp)--未仔细查看，后续如果需要可仔细看看具体流程

  ```java
  public long getOffsetInQueueByTime(final long timestamp) {
  
    //1. 根据时间定为到文件
    MappedFile mappedFile = this.mappedFileQueue.getMappedFileByTime(timestamp);
    if (mappedFile != null) {
      long offset = 0;
  
      //2. 通过二分查找来加速检索
      //2.1 获取最小的偏移量
      int low = minLogicOffset > mappedFile.getFileFromOffset() ? (int) (minLogicOffset - mappedFile.getFileFromOffset()) : 0;
      int high = 0;
      int midOffset = -1, targetOffset = -1, leftOffset = -1, rightOffset = -1;
      long leftIndexValue = -1L, rightIndexValue = -1L;
      long minPhysicOffset = this.defaultMessageStore.getMinPhyOffset();
      SelectMappedBufferResult sbr = mappedFile.selectMappedBuffer(0);
      if (null != sbr) {
        ByteBuffer byteBuffer = sbr.getByteBuffer();
        high = byteBuffer.limit() - CQ_STORE_UNIT_SIZE;
        try {
  
          //二分查找进行查询
          while (high >= low) {
            midOffset = (low + high) / (2 * CQ_STORE_UNIT_SIZE) * CQ_STORE_UNIT_SIZE;
            byteBuffer.position(midOffset);
            long phyOffset = byteBuffer.getLong();
            int size = byteBuffer.getInt();
            if (phyOffset < minPhysicOffset) {
              low = midOffset + CQ_STORE_UNIT_SIZE;
              leftOffset = midOffset;
              continue;
            }
  
            long storeTime =
              this.defaultMessageStore.getCommitLog().pickupStoreTimestamp(phyOffset, size);
            if (storeTime < 0) {
              return 0;
            } else if (storeTime == timestamp) {
              targetOffset = midOffset;
              break;
            } else if (storeTime > timestamp) {
              high = midOffset - CQ_STORE_UNIT_SIZE;
              rightOffset = midOffset;
              rightIndexValue = storeTime;
            } else {
              low = midOffset + CQ_STORE_UNIT_SIZE;
              leftOffset = midOffset;
              leftIndexValue = storeTime;
            }
          }
  
  
          //找到了存储时间等于待查找时间戳的消息
          if (targetOffset != -1) {
  
            offset = targetOffset;
          } else {
  
            //返回当前时间戳大并且最接近待查找的便宜量
            if (leftIndexValue == -1) {
  
              offset = rightOffset;
              //返回当前时间戳小并且最接近待查找的便宜量
            } else if (rightIndexValue == -1) {
  
              offset = leftOffset;
            } else {
              offset =
                Math.abs(timestamp - leftIndexValue) > Math.abs(timestamp
                                                                - rightIndexValue) ? rightOffset : leftOffset;
            }
          }
  
          return (mappedFile.getFileFromOffset() + offset) / CQ_STORE_UNIT_SIZE;
        } finally {
          sbr.release();
        }
      }
    }
    return 0;
  }
  ```

  

#### Index文件（后续如果需要查看源码再来查看）

* RocketMQ专门为消息订阅构建的索引文件，提高根据主题和消息队列检索消息的速度

* RocketMQ引入了Hash索引机制为消息建立索引，HashMap设计包含两点

  * Hash槽
  * Hash冲突的链表结构

* Index索引文件

  <img src="../../../../照片/typora/image-20210809222532796.png" alt="image-20210809222532796" style="zoom:50%;" />





### 文件更新

#### 实时更新消息消费队列和索引文件

* 消息消费队列文件，消息属性索引文件都是基于CommitLog文件构建的，当消息生产者提交的消息存储的CommitLog文件中，ConsumeQueue，IndexFile需要及时更新，否则消息无法及时被消费，根据消息属性查找消息也会出现较大延迟
* Rocket通过开启一个线程ReputMessageService来**准实时**转发CommitLog文件更新事件，相应的任务处理器根据转发的消息及时更新ConsumeQueue,IndexFile文件

* ReputMessageService创建过程

  * DefaultMessageStore的构造方法

  ```java
  public DefaultMessageStore(final MessageStoreConfig messageStoreConfig, final BrokerStatsManager brokerStatsManager,
                             final MessageArrivingListener messageArrivingListener, final BrokerConfig brokerConfig) throws IOException {
    ....
      this.reputMessageService = new ReputMessageService();
    ....
  }
  ```

* ReputMessageService启动过程

  * DefaultMessageStore#start

  ```java
  public void start() throws Exception {
    .......
    {
      /**
               * 1. Make sure the fast-forward messages to be truncated during the recovering according to the max physical offset of the commitlog;
               * 2. DLedger committedPos may be missing, so the maxPhysicalPosInLogicQueue maybe bigger that maxOffset returned by DLedgerCommitLog, just let it go;
               * 3. Calculate the reput offset according to the consume queue;
               * 4. Make sure the fall-behind messages to be dispatched before starting the commitlog, especially when the broker role are automatically changed.
               */
      long maxPhysicalPosInLogicQueue = commitLog.getMinOffset();
  
      //查找ConsumeQueue中最大的消费偏移量
      for (ConcurrentMap<Integer, ConsumeQueue> maps : this.consumeQueueTable.values()) {
        for (ConsumeQueue logic : maps.values()) {
          if (logic.getMaxPhysicOffset() > maxPhysicalPosInLogicQueue) {
            maxPhysicalPosInLogicQueue = logic.getMaxPhysicOffset();
          }
        }
      }
      if (maxPhysicalPosInLogicQueue < 0) {
        maxPhysicalPosInLogicQueue = 0;
      }
      if (maxPhysicalPosInLogicQueue < this.commitLog.getMinOffset()) {
        maxPhysicalPosInLogicQueue = this.commitLog.getMinOffset();
        /**
                   * This happens in following conditions:
                   * 1. If someone removes all the consumequeue files or the disk get damaged.
                   * 2. Launch a new broker, and copy the commitlog from other brokers.
                   *
                   * All the conditions has the same in common that the maxPhysicalPosInLogicQueue should be 0.
                   * If the maxPhysicalPosInLogicQueue is gt 0, there maybe something wrong.
                   */
        log.warn("[TooSmallCqOffset] maxPhysicalPosInLogicQueue={} clMinOffset={}", maxPhysicalPosInLogicQueue, this.commitLog.getMinOffset());
      }
  
  
      //设置ReputMessageService从哪个物理偏移量开始转发消息给ConsumeQueue和IndexFile
      this.reputMessageService.setReputFromOffset(maxPhysicalPosInLogicQueue);
  
      //reputMessageService线程开始
      this.reputMessageService.start();
  
    }
  ```

##### ReputMessageService

* 类图

  <img src="../../../../照片/typora/image-20210810095103115.png" alt="image-20210810095103115" style="zoom:50%;" />

###### 创建&启动

* Start()方法

  ```java
  public void start() {
    log.info("Try to start service thread:{} started:{} lastThread:{}", getServiceName(), started.get(), thread);
    if (!started.compareAndSet(false, true)) {
      return;
    }
    
    //用于响应中断，可见性，使用volatile
    stopped = false;
    
    //启动一个线程来执行此runnable
    this.thread = new Thread(this, getServiceName());
    this.thread.setDaemon(isDaemon);
    this.thread.start();
  }
  ```

* run()方法

  * ReputMessageService线程每执行一次任务推送，休息1毫秒就继续尝试推送消息到消息消费队列和索引文件

  ```java
  @Override
  public void run() {
    DefaultMessageStore.log.info(this.getServiceName() + " service started");
  
    
    //此处正是通过初始化时的stopped字段进行控制
    //好处是，可以在其他方法中控制任务的执行
    while (!this.isStopped()) {
      try {
        
        //消息1毫秒
        Thread.sleep(1);
        
        //关键：推送消息到消息消费队列和索引文件
        this.doReput();
      } catch (Exception e) {
        DefaultMessageStore.log.warn(this.getServiceName() + " service has exception. ", e); 
      }
    }
  
    DefaultMessageStore.log.info(this.getServiceName() + " service end");
  }
  ```

* doReput

  ```java
  private void doReput() {
  
    //重置推送的起始物理偏移量
    if (this.reputFromOffset < DefaultMessageStore.this.commitLog.getMinOffset()) {
      log.warn("The reputFromOffset={} is smaller than minPyOffset={}, this usually indicate that the dispatch behind too much and the commitlog has expired.",
               this.reputFromOffset, DefaultMessageStore.this.commitLog.getMinOffset());
      this.reputFromOffset = DefaultMessageStore.this.commitLog.getMinOffset();
    }
  
  
    //循环所有commitLog文件，只要可用就执行for循环
    //逻辑：推送消息的物理偏移量 < 最大偏移量
    for (boolean doNext = true; this.isCommitLogAvailable() && doNext; ) {
  
      if (DefaultMessageStore.this.getMessageStoreConfig().isDuplicationEnable()
          && this.reputFromOffset >= DefaultMessageStore.this.getConfirmOffset()) {
        break;
      }
  
  
      //1. 返回reputFromOffset偏移量开始的全部有效数据(commitLog文件)，然后循环读取每条消息
      SelectMappedBufferResult result = DefaultMessageStore.this.commitLog.getData(reputFromOffset);
      if (result != null) {
        try {
          this.reputFromOffset = result.getStartOffset();
  
          //2. 从result返回的ByteBuffer中循环读取消息，一次读取一条
          for (int readSize = 0; readSize < result.getSize() && doNext; ) {
  
            //2.1 创建DispatchRequest对象
            DispatchRequest dispatchRequest =
              DefaultMessageStore.this.commitLog.checkMessageAndReturnSize(result.getByteBuffer(), false, false);
            int size = dispatchRequest.getBufferSize() == -1 ? dispatchRequest.getMsgSize() : dispatchRequest.getBufferSize();
  
            if (dispatchRequest.isSuccess()) {
  
              //2.2. 消息长度大于0，存在消息
              if (size > 0) {
  
                //3. 通过构建消息消费队列ConsumeQueue , 索引文件 IndexFile
  
                //体现，更新两文件，ReputMessageService是构建更新消息，然后进行推送，进而更新，而不是在此run方法中执行，虽然也是同一个线程执行，但是体现出了，将所有保存的处理逻辑交给DefaultMessageStore来处理
                DefaultMessageStore.this.doDispatch(dispatchRequest);
  
                if (BrokerRole.SLAVE != DefaultMessageStore.this.getMessageStoreConfig().getBrokerRole()
                    && DefaultMessageStore.this.brokerConfig.isLongPollingEnable()) {
                  DefaultMessageStore.this.messageArrivingListener.arriving(dispatchRequest.getTopic(),
                                                                            dispatchRequest.getQueueId(), dispatchRequest.getConsumeQueueOffset() + 1,
                                                                            dispatchRequest.getTagsCode(), dispatchRequest.getStoreTimestamp(),
                                                                            dispatchRequest.getBitMap(), dispatchRequest.getPropertiesMap());
                }
  
                this.reputFromOffset += size;
                readSize += size;
                if (DefaultMessageStore.this.getMessageStoreConfig().getBrokerRole() == BrokerRole.SLAVE) {
                  DefaultMessageStore.this.storeStatsService
                    .getSinglePutMessageTopicTimesTotal(dispatchRequest.getTopic()).incrementAndGet();
                  DefaultMessageStore.this.storeStatsService
                    .getSinglePutMessageTopicSizeTotal(dispatchRequest.getTopic())
                    .addAndGet(dispatchRequest.getMsgSize());
                }
              } else if (size == 0) {
                this.reputFromOffset = DefaultMessageStore.this.commitLog.rollNextFile(this.reputFromOffset);
                readSize = result.getSize();
              }
            } else if (!dispatchRequest.isSuccess()) {
  
              if (size > 0) {
                log.error("[BUG]read total count not equals msg total size. reputFromOffset={}", reputFromOffset);
                this.reputFromOffset += size;
              } else {
                doNext = false;
                // If user open the dledger pattern or the broker is master node,
                // it will not ignore the exception and fix the reputFromOffset variable
                if (DefaultMessageStore.this.getMessageStoreConfig().isEnableDLegerCommitLog() ||
                    DefaultMessageStore.this.brokerConfig.getBrokerId() == MixAll.MASTER_ID) {
                  log.error("[BUG]dispatch message to consume queue error, COMMITLOG OFFSET: {}",
                            this.reputFromOffset);
                  this.reputFromOffset += result.getSize() - readSize;
                }
              }
            }
          }
        } finally {
          result.release();
        }
      } else {
        doNext = false;
      }
    }
  }
  ```

**SelectMappedBufferResult**

* 说明，只代表了一个MappedFile文件

* 变量

  ```java
  public class SelectMappedBufferResult {
  
      private final long startOffset;
  
      private final ByteBuffer byteBuffer;
  
      private int size;
  
      private MappedFile mappedFile;
  
  }
  ```

  







**DispatchRequest**

* 结构

  <img src="../../../../照片/typora/image-20210810101445486.png" alt="image-20210810101445486" style="zoom:50%;" />

* 源码

  ```java
  public class DispatchRequest {
  
      //消息主题名称
      private final String topic;
  
      //消息队列id
      private final int queueId;
  
      //消息物理偏移量
      private final long commitLogOffset;
  
      //消息长度
      private int msgSize;
  
      //消息过滤tag hashcode
      private final long tagsCode;
  
      //消息存储时间戳
      private final long storeTimestamp;
  
      //消息队列偏移量
      private final long consumeQueueOffset;
  
      //消息索引key，多个索引key用空格隔开
      private final String keys;
  
      //是否成功解析到完整的消息
      private final boolean success;
  
      //消息唯一key
      private final String uniqKey;
  
      //消息提供标记
      private final int sysFlag;
  
      //消息预处理事务偏移量
      private final long preparedTransactionOffset;
  
      //消息属性
      private final Map<String, String> propertiesMap;
  
      //位图
      private byte[] bitMap;
    
    
      //省略构造方法和get set方法
    
  }  
  ```



##### 根据消息跟新ConsumeQueue

* 上面在收到commitLog写入commitLog之后，使用ReputMessageService来转发推送消息DispatchRequest，交由DefaultMessageStore进行处理



###### DefaultMessageStore

* doDispatch

  * 此处没有使用策略，为何，因为所有的CommitLogDispatcher都需要跟新，包括ConsumeQueue，IndexQueue，BitMap

  ```java
  public void doDispatch(DispatchRequest req) {
    for (CommitLogDispatcher dispatcher : this.dispatcherList) {
      dispatcher.dispatch(req);
    }
  }
  ```

  * CommitLogDispatcher

    ```java
    public interface CommitLogDispatcher {
    
        void dispatch(final DispatchRequest request);
    }
    ```

    * 子类:少了一个，但是平时不接触后面如果需要，再更新

      * CommitLogDispatcherBuildConsumeQueue：跟新ConsumeQueue
      * CommitLogDispatcherBuildIndex：更新IndexFile

      ![image-20210810110536142](../../../../照片/typora/image-20210810110536142.png)

* putMessagePositionInfo

  ```java
  public void putMessagePositionInfo(DispatchRequest dispatchRequest) {
    //根据消息主题与队列ID，查询消息消费队列文件ConsumeQueue
    ConsumeQueue cq = this.findConsumeQueue(dispatchRequest.getTopic(), dispatchRequest.getQueueId());
  
    //将内容追加到ConsumeQueue的内存映射文件，只追加不刷盘，ConsumeQueue的刷盘方式固定为异步刷盘模式
    cq.putMessagePositionInfoWrapper(dispatchRequest);
  }
  ```

  



**CommitLogDispatcherBuildConsumeQueue**

* dispatch

  ```java
  class CommitLogDispatcherBuildConsumeQueue implements CommitLogDispatcher {
  
    @Override
    public void dispatch(DispatchRequest request) {
      final int tranType = MessageSysFlag.getTransactionValue(request.getSysFlag());
      switch (tranType) {
        case MessageSysFlag.TRANSACTION_NOT_TYPE:
        case MessageSysFlag.TRANSACTION_COMMIT_TYPE:
          DefaultMessageStore.this.putMessagePositionInfo(request);
          break;
        case MessageSysFlag.TRANSACTION_PREPARED_TYPE:
        case MessageSysFlag.TRANSACTION_ROLLBACK_TYPE:
          break;
      }
    }
  }
  ```

  



**ConsumeQueue**

* putMessagePositionInfoWrapper

  ```java
  public void putMessagePositionInfoWrapper(DispatchRequest request) {
    final int maxRetries = 30;
    boolean canWrite = this.defaultMessageStore.getRunningFlags().isCQWriteable();
  
    //最大重试次数
    for (int i = 0; i < maxRetries && canWrite; i++) {
      long tagsCode = request.getTagsCode();
      if (isExtWriteEnable()) {
        ConsumeQueueExt.CqExtUnit cqExtUnit = new ConsumeQueueExt.CqExtUnit();
        cqExtUnit.setFilterBitMap(request.getBitMap());
        cqExtUnit.setMsgStoreTime(request.getStoreTimestamp());
        cqExtUnit.setTagsCode(request.getTagsCode());
  
        long extAddr = this.consumeQueueExt.put(cqExtUnit);
        if (isExtAddr(extAddr)) {
          tagsCode = extAddr;
        } else {
          log.warn("Save consume queue extend fail, So just save tagsCode! {}, topic:{}, queueId:{}, offset:{}", cqExtUnit,
                   topic, queueId, request.getCommitLogOffset());
        }
      }
  
  
      //---------1. 根据消息更新ConsumeQueue------------
      //将内容追加到ConsumeQueue的内存映射文件中，只追击不刷盘
      boolean result = this.putMessagePositionInfo(request.getCommitLogOffset(),
                                                   request.getMsgSize(), tagsCode, request.getConsumeQueueOffset());
      if (result) {
        if (this.defaultMessageStore.getMessageStoreConfig().getBrokerRole() == BrokerRole.SLAVE ||
            this.defaultMessageStore.getMessageStoreConfig().isEnableDLegerCommitLog()) {
          this.defaultMessageStore.getStoreCheckpoint().setPhysicMsgTimestamp(request.getStoreTimestamp());
        }
        this.defaultMessageStore.getStoreCheckpoint().setLogicsMsgTimestamp(request.getStoreTimestamp());
        return;
      } else {
        // XXX: warn and notify me
        log.warn("[BUG]put commit log position info to " + topic + ":" + queueId + " " + request.getCommitLogOffset()
                 + " failed, retry " + i + " times");
  
        try {
          Thread.sleep(1000);
        } catch (InterruptedException e) {
          log.warn("", e);
        }
      }
    }
  
    // XXX: warn and notify me
    log.error("[BUG]consume queue can not write, {} {}", this.topic, this.queueId);
    this.defaultMessageStore.getRunningFlags().makeLogicsQueueError();
  }
  ```



##### 根据消息更新Index索引文件

###### CommitLogDispatcherBuildIndex

* dispatch

  ```java
  //构建IndexFile索引文件
  class CommitLogDispatcherBuildIndex implements CommitLogDispatcher {
  
    @Override
    public void dispatch(DispatchRequest request) {
      
      //此值默认为true
      if (DefaultMessageStore.this.messageStoreConfig.isMessageIndexEnable()) {
        DefaultMessageStore.this.indexService.buildIndex(request);
      }
    }
  }
  ```

###### IndexService

* buildIndex

  ```java
  //真正的刷盘是异步刷盘，此处只是写入到了
  public void buildIndex(DispatchRequest req) {
    //1. 获取后创建IndexFile文件并获取所有文件最大的物理偏移量
    IndexFile indexFile = retryGetAndCreateIndexFile();
    if (indexFile != null) {
      long endPhyOffset = indexFile.getEndPhyOffset();
      DispatchRequest msg = req;
      String topic = msg.getTopic();
      String keys = msg.getKeys();
  
      //2. 如果消息的物理偏移量小于索引文件中的物理偏移量，说明重复消息，忽略本次索引构建
      if (msg.getCommitLogOffset() < endPhyOffset) {
        return;
      }
  
      final int tranType = MessageSysFlag.getTransactionValue(msg.getSysFlag());
      switch (tranType) {
        case MessageSysFlag.TRANSACTION_NOT_TYPE:
        case MessageSysFlag.TRANSACTION_PREPARED_TYPE:
        case MessageSysFlag.TRANSACTION_COMMIT_TYPE:
          break;
        case MessageSysFlag.TRANSACTION_ROLLBACK_TYPE:
          return;
      }
  
      //如果消息的唯一键不为空，则添加到Hash索引中，以便加速根据唯一键检索消息
      if (req.getUniqKey() != null) {
        
        //写入IndexFile文件
        indexFile = putKey(indexFile, msg, buildKey(topic, req.getUniqKey()));
        if (indexFile == null) {
          log.error("putKey error commitlog {} uniqkey {}", req.getCommitLogOffset(), req.getUniqKey());
          return;
        }
      }
  
      //构建索引键，RocketMQ支持为同一个消息建立多个索引，多个索引键空格分开。
      if (keys != null && keys.length() > 0) {
        String[] keyset = keys.split(MessageConst.KEY_SEPARATOR);
        for (int i = 0; i < keyset.length; i++) {
          String key = keyset[i];
          if (key.length() > 0) {
            indexFile = putKey(indexFile, msg, buildKey(topic, key));
            if (indexFile == null) {
              log.error("putKey error commitlog {} uniqkey {}", req.getCommitLogOffset(), req.getUniqKey());
              return;
            }
          }
        }
      }
    } else {
      log.error("build index error, stop building index");
    }
  }
  ```

###### IndexFile

* putKey

  ```java
  /**
       * 将消息索引键和消息偏移量写入到IndexFile的mappedByteBuffer，并没有落盘
       * @param key 消息索引
       * @param phyOffset 消息物理偏移量
       * @param storeTimestamp 消息存储时间
       * @return
       */
  public boolean putKey(final String key, final long phyOffset, final long storeTimestamp) {
  
    //使用的索引个数，小于最大的索引个数，表示可以写入数据
    if (this.indexHeader.getIndexCount() < this.indexNum) {
      int keyHash = indexKeyHashMethod(key);
  
      //获取hash槽下标
      int slotPos = keyHash % this.hashSlotNum;
  
      //获取相对于IndexFile的相对物理地址
      int absSlotPos = IndexHeader.INDEX_HEADER_SIZE + slotPos * hashSlotSize;
  
      FileLock fileLock = null;
  
      try {
  
        // fileLock = this.fileChannel.lock(absSlotPos, hashSlotSize,
        // false);
        //读取hash槽中存储的数据
        int slotValue = this.mappedByteBuffer.getInt(absSlotPos);
        if (slotValue <= invalidIndex || slotValue > this.indexHeader.getIndexCount()) {
          slotValue = invalidIndex;
        }
  
  
        //计算待存储消息的时间戳与第一条消息时间戳的差值，并转换成秒
        long timeDiff = storeTimestamp - this.indexHeader.getBeginTimestamp();
  
        timeDiff = timeDiff / 1000;
  
        if (this.indexHeader.getBeginTimestamp() <= 0) {
          timeDiff = 0;
        } else if (timeDiff > Integer.MAX_VALUE) {
          timeDiff = Integer.MAX_VALUE;
        } else if (timeDiff < 0) {
          timeDiff = 0;
        }
  
  
        //存储到IndexFile中
        //新增条目的起始物理地址 = 头部字节长度 + hash槽数量 * 单个hash槽大小（4个字节） + 当前Index条目个数 * 单个Index条目大小（20个字节）
        int absIndexPos =
          IndexHeader.INDEX_HEADER_SIZE + this.hashSlotNum * hashSlotSize
          + this.indexHeader.getIndexCount() * indexSize;
  
        //hashcode，此处存储的是key的HashCode，而不是Key，原因是将Index条目设计为定长结构，才能方便的检索与定位条目
        this.mappedByteBuffer.putInt(absIndexPos, keyHash);
        //物理偏移量，此处保存的是最新的物理偏移量，为何呢？如果出现冲突怎么办，很显然，是使用链表的接口，每个Index条目存储上一个的地址
        this.mappedByteBuffer.putLong(absIndexPos + 4, phyOffset);
        //时间差
        this.mappedByteBuffer.putInt(absIndexPos + 4 + 8, (int) timeDiff);
        //当前hash槽的值
        this.mappedByteBuffer.putInt(absIndexPos + 4 + 8 + 4, slotValue);
  
        this.mappedByteBuffer.putInt(absSlotPos, this.indexHeader.getIndexCount());
  
  
        //更新文件索引头信息
        if (this.indexHeader.getIndexCount() <= 1) {
          this.indexHeader.setBeginPhyOffset(phyOffset);
          this.indexHeader.setBeginTimestamp(storeTimestamp);
        }
  
        this.indexHeader.incHashSlotCount();
        this.indexHeader.incIndexCount();
        this.indexHeader.setEndPhyOffset(phyOffset);
        this.indexHeader.setEndTimestamp(storeTimestamp);
  
        return true;
      } catch (Exception e) {
        log.error("putKey exception, Key: " + key + " KeyHashCode: " + key.hashCode(), e);
      } finally {
        if (fileLock != null) {
          try {
            fileLock.release();
          } catch (IOException e) {
            log.error("Failed to release the lock", e);
          }
        }
      }
    } else {
      log.warn("Over index file capacity: index count = " + this.indexHeader.getIndexCount()
               + "; index max num = " + this.indexNum);
    }
  
    return false;
  }
  ```



### 消息队列和索引文件恢复

* 背景
  * RocketMQ存储首先将消息全量存储在CommitLog文件中，然后异步生成转发任务更新ConsumeQueue、Index文件
  * 消息成功存储到CommitLog文件中，转发任务未成功执行，此时消息服务器Broker由于某个原因宕机，导致CommitLog、ConsumeQueue、IndexFile文件数据不一致。由于没有转发到ConsumeQueue，那么这部分消息将永远不会被消费者消费。
* 问题
  * RocketMQ是如何使CommitLog、消息消费队列ConsumeQueue，达到最终一致性的？？？
    * 肯定是在宕机之后恢复的时候，加载文件的时候发现，数据不一致性的问题，想必从加载文件开始

#### 文件加载

* CommitLog、ConsumeQueue以及MappedFileQueue和MappedFile的关系

  ![image-20210810160704490](../../../../照片/typora/image-20210810160704490.png)

* 加载流程图

  * 通过流程图看出，MappedFileQueue的作用
    * 代表一个集合
      * 在CommitLog的时候，代表了目录 ${ROCKET_HOME}/store/commitlog，
      * 在ConsumeQueue的时候，代表了目录 ${ROCKET_HOME}/store/cosumequeue/topic/queueid
  * MappedFile作用
    * 封装commitLog文件
    * 封装consumequeue文件
    * 解释：无论是commitLog文件还是consumequeue文件都是使用mappedFile对象来代替的，内部包含真正的文件，以及操作的通道

  ![RocketMQ--消息存储--消息消费文件和索引文件恢复](../../../../照片/typora/RocketMQ--消息存储--消息消费文件和索引文件恢复.jpg)



#### DefaultMessageStore

##### load()

* 源码

  ```java
  /**
       * 加载CommitLog的方法
       * @throws IOException
       */
  public boolean load() {
    boolean result = true;
  
    try {
  
      //1. 判断上次是不是异常退出。
      boolean lastExitOK = !this.isTempFileExist();
      log.info("last shutdown {}", lastExitOK ? "normally" : "abnormally");
  
      //2. 加载延迟队列
      if (null != scheduleMessageService) {
        result = result && this.scheduleMessageService.load();
      }
  
      //3. 加载CommitLog文件
      // load Commit Log
      result = result && this.commitLog.load();
  
      //4. 加载消息消费队列
      // load Consume Queue
      result = result && this.loadConsumeQueue();
  
      if (result) {
  
        //5. 加载存储检测点，检测点主要记录commitLog文件、ConsumeQueue文件、Index索引文件的刷盘点，将在下文的文件刷盘中再次提交
        this.storeCheckpoint =
          new StoreCheckpoint(StorePathConfigHelper.getStoreCheckpoint(this.messageStoreConfig.getStorePathRootDir()));
  
        //6. 加载索引文件，如果上次异常退出，而且索引文件
        this.indexService.load(lastExitOK);
  
        //7. 根据Broker是否正常停止执行不同的恢复策略
        this.recover(lastExitOK);
  
        log.info("load over, and the max phy offset = {}", this.getMaxPhyOffset());
      }
    } catch (Exception e) {
      log.error("load exception", e);
      result = false;
    }
  
    if (!result) {
      this.allocateMappedFileService.shutdown();
    }
  
    return result;
  }
  ```

##### isTempFileExist

* 源码

  ```java
  /**
       * 一开始创建文件，在正常退出的时候，使用JVM的钩子函数，去删除abort文件
       * 实现机制，Broker在启动时创建${ROCKET_HOME}/store/abort文件，正常退出，会通过JVM钩子函数删除abort文件。如果异常，那么此文件是存在的，不会执行删除
       * @return
       */
  private boolean isTempFileExist() {
    String fileName = StorePathConfigHelper.getAbortFile(this.messageStoreConfig.getStorePathRootDir());
    File file = new File(fileName);
    return file.exists();
  }
  
  ```



#### CommitLog

##### load()

* 源码

  ```java
  
  public boolean load() {
    boolean result = this.mappedFileQueue.load();
    log.info("load commit log " + (result ? "OK" : "Failed"));
    return result;
  }
  
  
  ```

  

#### MappedFileQueue

##### load()

* 源码

  ```java
  public boolean load() {
  
    //加载${ROCKET_HOME}/store/commitLog目录下的所有文件并按照文件名排序
    File dir = new File(this.storePath);
    File[] files = dir.listFiles();
    if (files != null) {
      // ascending order
      Arrays.sort(files);
      for (File file : files) {
  
        //如果文件大小与配置文件的单个文件大小不一致，将忽略该目录下所有文件
        if (file.length() != this.mappedFileSize) {
          log.warn(file + "\t" + file.length()
                   + " length not matched message store config value, please check it manually");
  
          //直接返回false
          return false;
        }
  
        try {
  
          //创建commtiLog消息文件 对应的对象 MappedFile
          MappedFile mappedFile = new MappedFile(file.getPath(), mappedFileSize);
  
          mappedFile.setWrotePosition(this.mappedFileSize);
          mappedFile.setFlushedPosition(this.mappedFileSize);
          mappedFile.setCommittedPosition(this.mappedFileSize);
          this.mappedFiles.add(mappedFile);
          log.info("load " + file.getPath() + " OK");
        } catch (IOException e) {
          log.error("load file " + file + " error", e);
          return false;
        }
      }
    }
  
    return true;
  }
  ```

  * MappedFile的构造方法，最终执行init方法

    ```java
    private void init(final String fileName, final int fileSize) throws IOException {
      this.fileName = fileName;
      this.fileSize = fileSize;
      this.file = new File(fileName);
    
      //文件名称代表该文件的起始偏移量
      this.fileFromOffset = Long.parseLong(this.file.getName());
      boolean ok = false;
    
      ensureDirOK(this.file.getParent());
    
      try {
        //文件通道，通过RandomAccessFile创建读写文件通道
        this.fileChannel = new RandomAccessFile(this.file, "rw").getChannel();
        //将文件内容使用NIO的内存映射Buffer将文件映射到内存中
        this.mappedByteBuffer = this.fileChannel.map(MapMode.READ_WRITE, 0, fileSize);
        TOTAL_MAPPED_VIRTUAL_MEMORY.addAndGet(fileSize);
        TOTAL_MAPPED_FILES.incrementAndGet();
        ok = true;
      } catch (FileNotFoundException e) {
        log.error("Failed to create file " + this.fileName, e);
        throw e;
      } catch (IOException e) {
        log.error("Failed to map file " + this.fileName, e);
        throw e;
      } finally {
        if (!ok && this.fileChannel != null) {
          this.fileChannel.close();
        }
      }
    }
    ```



* 上面加载的流程最后 recover(lastExitOk)正是，根据上一次是否是正常退出，然后进行文件恢复工作

  ```java
  private void recover(final boolean lastExitOK) {
    
    //恢复consumequeue
    long maxPhyOffsetOfConsumeQueue = this.recoverConsumeQueue();
  
    //恢复commitlog
    //正常退出
    if (lastExitOK) {
      this.commitLog.recoverNormally(maxPhyOffsetOfConsumeQueue);
    } else {
      //异常退出
      this.commitLog.recoverAbnormally(maxPhyOffsetOfConsumeQueue);
    }
  
    this.recoverTopicQueueTable();
  }
  ```

  ```java
  /*
   * 恢复消费队列，返回所有消费队列内的最大offset，即该offset就是commitlog中已经转储到消费队列的offset
   */
  //org.apache.rocketmq.store.DefaultMessageStore.recoverConsumeQueue()
  private long recoverConsumeQueue() {
      long maxPhysicOffset = -1;
      for (ConcurrentMap<Integer, ConsumeQueue> maps : this.consumeQueueTable.values()) {//遍历同topic下的所有ConsumeQueue集合
          for (ConsumeQueue logic : maps.values()) {//遍历同queueID下的所有ConsumeQueue
              logic.recover();//恢复消费队列
              if (logic.getMaxPhysicOffset() > maxPhysicOffset) {
                  maxPhysicOffset = logic.getMaxPhysicOffset();
              }
          }
      }
   
      return maxPhysicOffset;//返回所有消息队列文件内消息在commitlog中的最大偏移量
  }
  //org.apache.rocketmq.store.ConsumeQueue.recover()
  public void recover() {
      final List<MappedFile> mappedFiles = this.mappedFileQueue.getMappedFiles();
      if (!mappedFiles.isEmpty()) {
   
          int index = mappedFiles.size() - 3;//最多恢复三个
          if (index < 0)
              index = 0;
   
          int mappedFileSizeLogics = this.mappedFileSize;//20字节
          MappedFile mappedFile = mappedFiles.get(index);
          ByteBuffer byteBuffer = mappedFile.sliceByteBuffer();//共用同一个缓冲区，但是position各自独立
          long processOffset = mappedFile.getFileFromOffset();//队列文件名
          long mappedFileOffset = 0;
          long maxExtAddr = 1;
          while (true) {//消费队列存储单元是一个20字节定长数据，commitlog offset(8) + size(4) + message tag hashcode(8),commitlog offset是指这条消息在commitlog文件实际偏移量，size指消息大小，消息tag的哈希值，用于校验
              for (int i = 0; i < mappedFileSizeLogics; i += CQ_STORE_UNIT_SIZE) {
                  long offset = byteBuffer.getLong();//先读取8字节，即commitlog offset
                  int size = byteBuffer.getInt();//再读取4字节，即msg size
                  long tagsCode = byteBuffer.getLong();//再读取8字节，即message tag hashcode
   
                  if (offset >= 0 && size > 0) {
                      mappedFileOffset = i + CQ_STORE_UNIT_SIZE;//consumequeue上当前消息末尾位置，该值为20*N，其中N是表示当前消息在consumequeue上是第几个消息
                      this.maxPhysicOffset = offset;//队列内消息在commitlog中的偏移量,this.maxPhysicOffset最终为该队列下的consumequeue文件内的消息在commitlog的最大物理偏移量，即在commitlog的位置，该值也就是commitlog转储到consumequeue的位置，该位置后的消息就需要转储到consumequeue
                      if (isExtAddr(tagsCode)) {//用于扩展的consumequeue，忽略，默认是不开启，生产通常也不开启，没有研究过这个
                          maxExtAddr = tagsCode;
                      }
                  } else {
                      log.info("recover current consume queue file over,  " + mappedFile.getFileName() + " "
                          + offset + " " + size + " " + tagsCode);
                      break;
                  }
              }
   
              if (mappedFileOffset == mappedFileSizeLogics) {//达到consumequeue文件末尾
                  index++;
                  if (index >= mappedFiles.size()) {//遍历到该队列下是最后一个consumequeue文件则退出循环
   
                      log.info("recover last consume queue file over, last mapped file "
                          + mappedFile.getFileName());
                      break;
                  } else {
                      mappedFile = mappedFiles.get(index);//获取下一个mappedFile对象
                      byteBuffer = mappedFile.sliceByteBuffer();
                      processOffset = mappedFile.getFileFromOffset();//重置processOffset为mappedFile文件名
                      mappedFileOffset = 0;
                      log.info("recover next consume queue file, " + mappedFile.getFileName());
                  }
              } else {
                  log.info("recover current consume queue queue over " + mappedFile.getFileName() + " "
                      + (processOffset + mappedFileOffset));
                  break;
              }
          }
   
          processOffset += mappedFileOffset;//processOffset
          this.mappedFileQueue.setFlushedWhere(processOffset);//设置刷新位置
          this.mappedFileQueue.setCommittedWhere(processOffset);//设置提交位置
          this.mappedFileQueue.truncateDirtyFiles(processOffset);//清理大于指定offset的脏文件
          
   
          if (isExtReadEnable()) {//忽略，生产也不开启扩展consumequeue
              this.consumeQueueExt.recover();
              log.info("Truncate consume queue extend file by max {}", maxExtAddr);
              this.consumeQueueExt.truncateByMaxAddress(maxExtAddr);
          }
      }
  }
  ```

  



#### 文件恢复

* 存储启动时所谓的文件恢复主要完成 flushedPosition, committedWhere 指针的设置 、消息消费队列最大偏移 加载到内存，并删除 flushedPosition 之后所有的文件
* 无论正常还是异常，针对consumeQueue中正常的数据，那么必然是肯定是同步成功的，所以无论正常还是异常，对于consumequeue的同步都可以认为，先针对consumequeue去掉有问题的数据，然后再恢复commitlog的时候
  * 如果正常，那么就直接恢复commitlog即可
  * 如果异常，把commitlog中可能没有同步成功的数据，重新进行发送请求，更新一次consumequeue就好了

##### Broker正常停止文件恢复

* 因为是正常退出，所以没有针对ConsumeQueue进行恢复处理

###### CommitLog

* recoverNormally

  ```java
  public void recoverNormally(long maxPhyOffsetOfConsumeQueue) {
    boolean checkCRCOnRecover = this.defaultMessageStore.getMessageStoreConfig().isCheckCRCOnRecover();
    final List<MappedFile> mappedFiles = this.mappedFileQueue.getMappedFiles();
    if (!mappedFiles.isEmpty()) {
      // Began to recover from the last third file
      //1. Broker正常停止再重启，从倒数第三个文件开始进行恢复，不足三个，从第一个文件恢复
      int index = mappedFiles.size() - 3;
      if (index < 0)
        index = 0;
  
      //1.1 获取文件
      MappedFile mappedFile = mappedFiles.get(index);
      ByteBuffer byteBuffer = mappedFile.sliceByteBuffer();
  
  
      //2. mappedFileOffSet为当前文件已检验通过的offSet
      //   processOffset是为了存储CommitLog文件已确认的物理偏移量
      //CommitLog文件已确认的物理偏移量 = mappedFile.getFileFromOffset(文件起始的相对偏移量) + mappedFileOffset
      long processOffset = mappedFile.getFileFromOffset();
  
      //为了存储当前文件CommitLog已经校验通过的，相对于上面获取的mappedFile的起始物理偏移量的相对偏移量
      // mappedFileOffset 记录了所有消息的消息长度总和，就是相对于mappedFile的相对偏移量
      long mappedFileOffset = 0;
      while (true) {
        //3. 遍历消息，每次一条消息进行处理，byteBuffer就代表了具体的消息内容
        DispatchRequest dispatchRequest = this.checkMessageAndReturnSize(byteBuffer, checkCRCOnRecover);
  
  
        int size = dispatchRequest.getMsgSize();
        // Normal data：如果查找结果成功，且消息的长度大于0，表示消息正确
        if (dispatchRequest.isSuccess() && size > 0) {
          //指针向前移动本条消息的长度
          mappedFileOffset += size;
        }
  
  
  
        //如果查找结果成功，且消息的长度为0，表示已到该文件的末尾
        else if (dispatchRequest.isSuccess() && size == 0) {
          index++;
          //后面没有文件，直接退出
          if (index >= mappedFiles.size()) {
            // Current branch can not happen
            log.info("recover last 3 physics file over, last mapped file " + mappedFile.getFileName());
            break;
            //还有下一个文件，则获取下一个文件
          } else {
            mappedFile = mappedFiles.get(index);
            byteBuffer = mappedFile.sliceByteBuffer();
            //绝对物理偏移量修改
            processOffset = mappedFile.getFileFromOffset();
            //相对物理偏移量，要修改为0
            mappedFileOffset = 0;
            log.info("recover next physics file, " + mappedFile.getFileName());
          }
        }
        // Intermediate file read error
        //如果查找结构为false,表明该文件未填满所有消息，跳出循环，结束遍历文件，此处的文件会删除
        else if (!dispatchRequest.isSuccess()) {
          log.info("recover physics file end, " + mappedFile.getFileName());
          break;
        }
      }
  
      processOffset += mappedFileOffset;
      //4. 跟新地址信息
      this.mappedFileQueue.setFlushedWhere(processOffset);
      this.mappedFileQueue.setCommittedWhere(processOffset);
  
      //5. 删除offset之后的所有文件
      this.mappedFileQueue.truncateDirtyFiles(processOffset);
  
      // Clear ConsumeQueue redundant data
      if (maxPhyOffsetOfConsumeQueue >= processOffset) {
        log.warn("maxPhyOffsetOfConsumeQueue({}) >= processOffset({}), truncate dirty logic files", maxPhyOffsetOfConsumeQueue, processOffset);
        this.defaultMessageStore.truncateDirtyLogicFiles(processOffset);
      }
    } else {
      // Commitlog case files are deleted
      log.warn("The commitlog files are deleted, and delete the consume queue files");
      this.mappedFileQueue.setFlushedWhere(0);
      this.mappedFileQueue.setCommittedWhere(0);
      this.defaultMessageStore.destroyLogics();
    }
  }
  
  ```

* truncateDirtyFiles

  ```java
  public void truncateDirtyFiles(long offset) {
    List<MappedFile> willRemoveFiles = new ArrayList<MappedFile>();
  
    //遍历所有文件
    for (MappedFile file : this.mappedFiles) {
      long fileTailOffset = file.getFileFromOffset() + this.mappedFileSize;
  
      //如果文件的尾部偏移量大于 获取正常数据时获取的偏移量
      if (fileTailOffset > offset) {
  
        //如果有效偏移量，在当前文件中，表示当前文件包含了有效偏移量，重新设置flushPosition和commitedPosition
        if (offset >= file.getFileFromOffset()) {
          file.setWrotePosition((int) (offset % this.mappedFileSize));
          file.setCommittedPosition((int) (offset % this.mappedFileSize));
          file.setFlushedPosition((int) (offset % this.mappedFileSize));
          //否则，表示，该文件是在有效文件后面创建的，而且还获取不到有效的数据，那么调用MappedFile#destroy释放MappedFile占用的内存资源
          //然后，加入到待删除文件列表中
        } else {
          file.destroy(1000);
          willRemoveFiles.add(file);
        }
      }
    }
  
    this.deleteExpiredFile(willRemoveFiles);
  }
  ```

  



##### Broker异常停止文件恢复

* 如果 Broker异常启动， 在文件恢复过程中 RocketMQ 会将最后一个有效文件中的所有消息重新转发到消息消费队 列与索引文件，确保不丢失消息，但同时会带来消息重复的问题，纵观RocktMQ 的整体设计思想， RocketMQ 保证消息不丢失但不保证消息不会重复消费 故消息消费业务方需要 实现消息消费的幕等设计

* 异常恢复的步骤和正常文件恢复的流程基本相同，差别

  * 正常停止默认从倒数第三个文件开始进行恢复，异常停止则需要从最后一个文件往前走，找到第一个消息存储正常的文件
  * 如果commitlog目录没有消息文件，如果在消息消费队列目录下存在文件，则需要销毁

* 源码

  ```java
  @Deprecated
  public void recoverAbnormally(long maxPhyOffsetOfConsumeQueue) {
    // recover by the minimum time stamp
    boolean checkCRCOnRecover = this.defaultMessageStore.getMessageStoreConfig().isCheckCRCOnRecover();
    final List<MappedFile> mappedFiles = this.mappedFileQueue.getMappedFiles();
    if (!mappedFiles.isEmpty()) {
      // Looking beginning to recover from which file
  
      //区别1：从最后一个文件开始
      int index = mappedFiles.size() - 1;
      MappedFile mappedFile = null;
      for (; index >= 0; index--) {
        mappedFile = mappedFiles.get(index);
  
        //找到第一个消息存储正常的文件
        if (this.isMappedFileMatchedRecover(mappedFile)) {
          log.info("recover from this mapped file " + mappedFile.getFileName());
          break;
        }
      }
  
      if (index < 0) {
        index = 0;
        //正常文件
        mappedFile = mappedFiles.get(index);
      }
  
      ByteBuffer byteBuffer = mappedFile.sliceByteBuffer();
      long processOffset = mappedFile.getFileFromOffset();
      long mappedFileOffset = 0;
      
      //将MappedFile中的消息内容，推送给ConsumeQueue和IndexQueue
      while (true) {
        DispatchRequest dispatchRequest = this.checkMessageAndReturnSize(byteBuffer, checkCRCOnRecover);
        int size = dispatchRequest.getMsgSize();
  
        if (dispatchRequest.isSuccess()) {
          // Normal data
          if (size > 0) {
            mappedFileOffset += size;
  
  
            //分发给defaultMessageStore，重新执行更新
            if (this.defaultMessageStore.getMessageStoreConfig().isDuplicationEnable()) {
              if (dispatchRequest.getCommitLogOffset() < this.defaultMessageStore.getConfirmOffset()) {
                this.defaultMessageStore.doDispatch(dispatchRequest);
              }
            } else {
              this.defaultMessageStore.doDispatch(dispatchRequest);
            }
          }
          // Come the end of the file, switch to the next file
          // Since the return 0 representatives met last hole, this can
          // not be included in truncate offset
          else if (size == 0) {
            index++;
            if (index >= mappedFiles.size()) {
              // The current branch under normal circumstances should
              // not happen
              log.info("recover physics file over, last mapped file " + mappedFile.getFileName());
              break;
            } else {
              mappedFile = mappedFiles.get(index);
              byteBuffer = mappedFile.sliceByteBuffer();
              processOffset = mappedFile.getFileFromOffset();
              mappedFileOffset = 0;
              log.info("recover next physics file, " + mappedFile.getFileName());
            }
          }
        } else {
          log.info("recover physics file end, " + mappedFile.getFileName() + " pos=" + byteBuffer.position());
          break;
        }
      }
  
      processOffset += mappedFileOffset;
      this.mappedFileQueue.setFlushedWhere(processOffset);
      this.mappedFileQueue.setCommittedWhere(processOffset);
      this.mappedFileQueue.truncateDirtyFiles(processOffset);
  
      // Clear ConsumeQueue redundant data
      if (maxPhyOffsetOfConsumeQueue >= processOffset) {
        log.warn("maxPhyOffsetOfConsumeQueue({}) >= processOffset({}), truncate dirty logic files", maxPhyOffsetOfConsumeQueue, processOffset);
        this.defaultMessageStore.truncateDirtyLogicFiles(processOffset);
      }
    }
    // Commitlog case files are deleted
    else {
      log.warn("The commitlog files are deleted, and delete the consume queue files");
      this.mappedFileQueue.setFlushedWhere(0);
      this.mappedFileQueue.setCommittedWhere(0);
      this.defaultMessageStore.destroyLogics();
    }
  }
  ```

  

## 刷盘机制

### Broker同步刷盘

* 同步刷盘：消息追加到内存映射文件的内存中后，立即将数据从内存刷写到磁盘文件

* 原理

  * 消息生产者在消息服务端将消息内容追加到内存映射文件中（内存）后，需要同步将内存的内容立刻刷写到磁盘 通过调用内存映射文件

    ( MappedB yteB uffer force 方法）可将内存中的数据写入磁盘

#### 源码

* DefaultMessageStore#putMessage -> commitLog.putMessage -> commitLog.handleDiskFlush
* 优点
  * 异步转同步的用法
  * 为了读取的时候，防止写入性能问题，采用了两个队列，在每次落盘的时候，切换，这样落盘和写入分割开来，有单类似COW，空间换时间



##### CommitLog

###### handleDiskFlush

```java
public void handleDiskFlush(AppendMessageResult result, PutMessageResult putMessageResult, MessageExt messageExt) {
  // Synchronization flush
  // ---------------同步刷盘-----------------
  if (FlushDiskType.SYNC_FLUSH == this.defaultMessageStore.getMessageStoreConfig().getFlushDiskType()) {
    final GroupCommitService service = (GroupCommitService) this.flushCommitLogService;
    if (messageExt.isWaitStoreMsgOK()) {

      //1. 构建 GroupCommitRequest 同步任务，并提交到 GroupCommitService
      GroupCommitRequest request = new GroupCommitRequest(result.getWroteOffset() + result.getWroteBytes());
      
      //放入落盘去请求，唤醒落盘线程执行落盘操作
      service.putRequest(request);
      
      //Future获取异步结果
      CompletableFuture<PutMessageStatus> flushOkFuture = request.future();
      PutMessageStatus flushStatus = null;
      try {
        //2. 等待同步刷盘任务完成，如果超时则返回刷盘错误，刷盘成功后正常返回给调用方
        flushStatus = flushOkFuture.get(this.defaultMessageStore.getMessageStoreConfig().getSyncFlushTimeout(),
                                        TimeUnit.MILLISECONDS);
      } catch (InterruptedException | ExecutionException | TimeoutException e) {
        //flushOK=false;
      }
      if (flushStatus != PutMessageStatus.PUT_OK) {
        log.error("do groupcommit, wait for flush failed, topic: " + messageExt.getTopic() + " tags: " + messageExt.getTags()
                  + " client address: " + messageExt.getBornHostString());
        putMessageResult.setPutMessageStatus(PutMessageStatus.FLUSH_DISK_TIMEOUT);
      }
    } else {
      service.wakeup();
    }
  }
  // Asynchronous flush
  //---------------异步刷盘---------------
  else {
    if (!this.defaultMessageStore.getMessageStoreConfig().isTransientStorePoolEnable()) {
      flushCommitLogService.wakeup();
    } else {
      commitLogService.wakeup();
    }
  }
}
```

* 此处异步转同步，使用GroupCommitService（内部的一个线程），来执行任务，使用countDownlatch来等待，Future来获取异步的结果



##### GroupCommitService

```java
/**
     * 同步刷盘服务
     * GroupCommit Service
     */
class GroupCommitService extends FlushCommitLogService {

  //同步刷盘任务暂存容器
  private volatile List<GroupCommitRequest> requestsWrite = new ArrayList<GroupCommitRequest>();

  //GroupCommitService 线程每次处理的request容器，设计亮点，避免了任务提交与任务执行的锁冲突
  private volatile List<GroupCommitRequest> requestsRead = new ArrayList<GroupCommitRequest>();


  //异步转同步的机制  使用countDownLatch来实现，好处，可以在异步的时间范围内，执行其他逻辑
  //客户端提交同步刷盘任务到GroupCommitService线程，如果该线程处于等待状态则将其唤醒
  public synchronized void putRequest(final GroupCommitRequest request) {
    synchronized (this.requestsWrite) {
      this.requestsWrite.add(request);
    }

    //如果执行同步任务的线程等待中，则将其唤醒，执行落盘的操作，执行run方法
    //waitPoint是CountDownLatch，起到了调度的任务
    if (hasNotified.compareAndSet(false, true)) {
      waitPoint.countDown(); // notify
    }
  }



  //将写的临时缓存队列，复制给读的缓存队列
  //由于避免同步刷盘消费任务与其他消息生产者提交任务直接的锁竞争
  //GroupCommitService提供读容器与写容器，这两个容器没执行完一次任务后，交换，继续消费任务
  private void swapRequests() {
    List<GroupCommitRequest> tmp = this.requestsWrite;
    this.requestsWrite = this.requestsRead;
    this.requestsRead = tmp;
  }

  private void doCommit() {
    synchronized (this.requestsRead) {
      if (!this.requestsRead.isEmpty()) {

        //遍历同步刷盘任务列表，根据加入顺序逐一执行刷屏盘逻辑
        for (GroupCommitRequest req : this.requestsRead) {
          // There may be a message in the next file, so a maximum of
          // two times the flush
          boolean flushOK = false;
          for (int i = 0; i < 2 && !flushOK; i++) {
            flushOK = CommitLog.this.mappedFileQueue.getFlushedWhere() >= req.getNextOffset();

            if (!flushOK) {
              //2. 执行刷盘操作，最终调用MappedByteBuffer的force方法
              CommitLog.this.mappedFileQueue.flush(0);
            }
          }

          //设置刷盘结果
          req.wakeupCustomer(flushOK);
        }


        //3. 处理完所有同步刷盘任务后 ，更新刷盘检测点 StoreCheckPoint中的physicMsgTiemstamp，但并么有执行检测点的刷盘操作
        // 刷盘检测点的刷盘操作将在刷写消息队列文件时触发
        long storeTimestamp = CommitLog.this.mappedFileQueue.getStoreTimestamp();
        if (storeTimestamp > 0) {
          CommitLog.this.defaultMessageStore.getStoreCheckpoint().setPhysicMsgTimestamp(storeTimestamp);
        }

        //4. 清除已经落盘的数据
        this.requestsRead.clear();
      } else {
        // Because of individual messages is set to not sync flush, it
        // will come to this process
        CommitLog.this.mappedFileQueue.flush(0);
      }
    }
  }


  //异步线程执行同步磁盘的操作
  //执行操作
  public void run() {
    CommitLog.log.info(this.getServiceName() + " service started");

    while (!this.isStopped()) {
      try {
        //每执行一批同步刷盘请求(requestRead容器中请求)后，休息10ms
        //会将两个区域进行交换，然后获取执行数据
        this.waitForRunning(10);
        //同步刷盘请求
        this.doCommit();
      } catch (Exception e) {
        CommitLog.log.warn(this.getServiceName() + " service has exception. ", e);
      }
    }

    // Under normal circumstances shutdown, wait for the arrival of the
    // request, and then flush
    try {
      Thread.sleep(10);
    } catch (InterruptedException e) {
      CommitLog.log.warn("GroupCommitService Exception, ", e);
    }


    //写读切换，加锁
    //类似于，读取数据相当于副本
    synchronized (this) {
      this.swapRequests();
    }

    this.doCommit();

    CommitLog.log.info(this.getServiceName() + " service end");
  }

  @Override
  protected void onWaitEnd() {
    this.swapRequests();
  }

  @Override
  public String getServiceName() {
    return GroupCommitService.class.getSimpleName();
  }

  @Override
  public long getJointime() {
    return 1000 * 60 * 5;
  }
}
```

* 父类方法ServiceThread

  * waitForRunning

    ```java
    protected void waitForRunning(long interval) {
      if (hasNotified.compareAndSet(true, false)) {
        //重要，调用了子类中的交换缓存区
        this.onWaitEnd();
        return;
      }
    
      //entry to wait
      waitPoint.reset();
    
      try {
        waitPoint.await(interval, TimeUnit.MILLISECONDS);
      } catch (InterruptedException e) {
        log.error("Interrupted", e);
      } finally {
        hasNotified.set(false);
        this.onWaitEnd();
      }
    }
    ```

##### GroupCommitRequest

```java
public static class GroupCommitRequest {

  //刷盘点偏移量
  private final long nextOffset;

  private CompletableFuture<PutMessageStatus> flushOKFuture = new CompletableFuture<>();
  private final long startTimestamp = System.currentTimeMillis();
  private long timeoutMillis = Long.MAX_VALUE;

  public GroupCommitRequest(long nextOffset, long timeoutMillis) {
    this.nextOffset = nextOffset;
    this.timeoutMillis = timeoutMillis;
  }

  public GroupCommitRequest(long nextOffset) {
    this.nextOffset = nextOffset;
  }


  public long getNextOffset() {
    return nextOffset;
  }

  public void wakeupCustomer(final boolean flushOK) {
    long endTimestamp = System.currentTimeMillis();
    PutMessageStatus result = (flushOK && ((endTimestamp - this.startTimestamp) <= this.timeoutMillis)) ?
      PutMessageStatus.PUT_OK : PutMessageStatus.FLUSH_SLAVE_TIMEOUT;
    //设置消息发送结果，通过future可以获取到值
    this.flushOKFuture.complete(result);
  }

  public CompletableFuture<PutMessageStatus> future() {
    return flushOKFuture;
  }

}
```



### 重要对象

#### CountDownLatch2

* 与CountDownLatch的区别







### Broker异步刷盘

* 两种机制

  * transientStorePoolEnable=true
    * RocketMQ会单独申请一个与目标物理文件(commitlog)同样大小的堆外内存，该堆外内存将使用**内存锁定，确保不会被置换到虚拟内存中去**
    * 消息首先追加到堆外内存，然后提交到与物理文件的内存映射文件中，再flush到磁盘
  * transientStorePoolEnable=true
    * 消息直接追加到与物理文件直接映射的内存中，然后刷写到磁盘中

* true的流程图

  <img src="../../../../照片/typora/image-20210812140557954.png" alt="image-20210812140557954" style="zoom:50%;" />
  1. 首先将消息直接追加到ByteBuffer(堆外内存DirectByteBuffer)，wrotePosition随着消息的不断追加向后移动
  2.  CommitRealTimeService 线程默认每 200ms将ByteBuffer 新追加的内容（wrotePosition减去commitedPosition ）的数据提交到 MappedByteBuffer中
  3. MappedByteBuffer在内存中追加提交的内容，wrotePostion指针向后移动，然后返回
  4. commit操作成功返回，将commitedPosition向后移动本次提交的内容长度，此时wrotePosition指针仍然不受影响可以向后移动
  5. FlushRealTimeServcie线程默认每500ms将MappedByteBuffer中新追加的内存（wrotePosition减去上一次刷写位置flushedPosition）通过调用MappedByteBuffer#force()方法将数据刷写到磁盘

  * 关系
    * 有数据提交，那么commitPosition向后移动
    * 每次从堆外内存写入MappedByteBuffer，wrotePosition向后移动
    * 每次刷盘完，就将flushPosition向后移动
    * commitPosition>=wrotePosition>=flushPosition
    * 内存映射中的内容 = (commitPosition - wrotePosition)之间的内容
    * 需要刷盘的内容 = (wrotePosition - flushPosition)之间的内容



#### CommitRealTimeService提交检查

* 将堆外内存的数据提交到MappedFile的MappedByteBuffer中

* 源码

  ```java
  class CommitRealTimeService extends FlushCommitLogService {
  
    private long lastCommitTimestamp = 0;
  
    @Override
    public String getServiceName() {
      return CommitRealTimeService.class.getSimpleName();
    }
  
    @Override
    public void run() {
      CommitLog.log.info(this.getServiceName() + " service started");
      while (!this.isStopped()) {
  
        //1. 3个重要参数定义
        //CommitRealTimeService任务间隔时间，默认200ms
        int interval = CommitLog.this.defaultMessageStore.getMessageStoreConfig().getCommitIntervalCommitLog();
  
        //一次提交任务至少包含页数，如果待提交数据不足，小于该参数配置的值，将忽略本次提交任务
        int commitDataLeastPages = CommitLog.this.defaultMessageStore.getMessageStoreConfig().getCommitCommitLogLeastPages();
  
        //两次真实提交最大间隔，默认200ms
        int commitDataThoroughInterval =
          CommitLog.this.defaultMessageStore.getMessageStoreConfig().getCommitCommitLogThoroughInterval();
  
        long begin = System.currentTimeMillis();
  
        //2. 如果距上次提交间隔超过 commitDataThoroughInterval ，则本次提交忽略 commitDataLeastPages 参数，也就是如果待提交数据小于指定页数，也执行提交操作
        if (begin >= (this.lastCommitTimestamp + commitDataThoroughInterval)) {
          this.lastCommitTimestamp = begin;
          commitDataLeastPages = 0;
        }
  
  
        //3. 执行提交操作，将待提交数据提交到物理文件的内存映射内存区
        //如果返回false，并不是代表提交失败，而是只提交了一部分数据，唤醒刷盘线程执行刷盘操作
        try {
          boolean result = CommitLog.this.mappedFileQueue.commit(commitDataLeastPages);
          long end = System.currentTimeMillis();
          if (!result) {
            this.lastCommitTimestamp = end; // result = false means some data committed.
            //now wake up flush thread.
  
            //唤醒真正刷盘的线程
            flushCommitLogService.wakeup();
          }
  
          if (end - begin > 500) {
            log.info("Commit data to file costs {} ms", end - begin);
          }
  
          //每完成一次提交动作，将等待200ms再执行下一次提交任务
          this.waitForRunning(interval);
        } catch (Throwable e) {
          CommitLog.log.error(this.getServiceName() + " service has exception. ", e);
        }
      }
  
      boolean result = false;
      for (int i = 0; i < RETRY_TIMES_OVER && !result; i++) {
        result = CommitLog.this.mappedFileQueue.commit(0);
        CommitLog.log.info(this.getServiceName() + " service shutdown, retry " + (i + 1) + " times " + (result ? "OK" : "Not OK"));
      }
      CommitLog.log.info(this.getServiceName() + " service end");
    }
  }
  ```



#### FlushCommitLogService

* 源码

  ```java
  class FlushRealTimeService extends FlushCommitLogService {
    private long lastFlushTimestamp = 0;
    private long printTimes = 0;
  
    public void run() {
      CommitLog.log.info(this.getServiceName() + " service started");
  
      while (!this.isStopped()) {
  
        //默认为false ，使用await方法等待
        //如果为true  ，使用Thread.sleep方法进行等待
        boolean flushCommitLogTimed = CommitLog.this.defaultMessageStore.getMessageStoreConfig().isFlushCommitLogTimed();
  
        //FlushRealTimeService线程任务执行间隔
        int interval = CommitLog.this.defaultMessageStore.getMessageStoreConfig().getFlushIntervalCommitLog();
  
        //一次刷写任务至少包含页数，如果待刷写数据不足，小于该参数配置的值，将忽略本次刷写任务
        int flushPhysicQueueLeastPages = CommitLog.this.defaultMessageStore.getMessageStoreConfig().getFlushCommitLogLeastPages();
  
        //两次真实刷写任务最大间隔，默认10s
        int flushPhysicQueueThoroughInterval =
          CommitLog.this.defaultMessageStore.getMessageStoreConfig().getFlushCommitLogThoroughInterval();
  
        boolean printFlushProgress = false;
  
        // Print flush progress
        //1. 如果距离上次刷盘时间超了，真实的刷盘时间间隔，则忽略数据不足的情况，进行刷盘操作
        long currentTimeMillis = System.currentTimeMillis();
        if (currentTimeMillis >= (this.lastFlushTimestamp + flushPhysicQueueThoroughInterval)) {
          this.lastFlushTimestamp = currentTimeMillis;
          flushPhysicQueueLeastPages = 0;
          printFlushProgress = (printTimes++ % 10) == 0;
        }
  
        try {
  
          //2. 执行刷盘任务前先等待指定的时间间隔，防止数据过快
          if (flushCommitLogTimed) {
            Thread.sleep(interval);
          } else {
            this.waitForRunning(interval);
          }
  
          if (printFlushProgress) {
            this.printFlushProgress();
          }
  
          long begin = System.currentTimeMillis();
  
          //调用真正的刷盘机制
          CommitLog.this.mappedFileQueue.flush(flushPhysicQueueLeastPages);
          long storeTimestamp = CommitLog.this.mappedFileQueue.getStoreTimestamp();
          if (storeTimestamp > 0) {
            CommitLog.this.defaultMessageStore.getStoreCheckpoint().setPhysicMsgTimestamp(storeTimestamp);
          }
          long past = System.currentTimeMillis() - begin;
          if (past > 500) {
            log.info("Flush data to disk costs {} ms", past);
          }
        } catch (Throwable e) {
          CommitLog.log.warn(this.getServiceName() + " service has exception. ", e);
          this.printFlushProgress();
        }
      }
  
      // Normal shutdown, to ensure that all the flush before exit
      boolean result = false;
      for (int i = 0; i < RETRY_TIMES_OVER && !result; i++) {
        result = CommitLog.this.mappedFileQueue.flush(0);
        CommitLog.log.info(this.getServiceName() + " service shutdown, retry " + (i + 1) + " times " + (result ? "OK" : "Not OK"));
      }
  
      this.printFlushProgress();
  
      CommitLog.log.info(this.getServiceName() + " service end");
    }
  
    @Override
    public String getServiceName() {
      return FlushRealTimeService.class.getSimpleName();
    }
  
    private void printFlushProgress() {
      // CommitLog.log.info("how much disk fall behind memory, "
      // + CommitLog.this.mappedFileQueue.howMuchFallBehind());
    }
  
    @Override
    public long getJointime() {
      return 1000 * 60 * 5;
    }
  }
  ```

  * MappedFileQueue#flush

    ```java
    public boolean flush(final int flushLeastPages) {
      boolean result = true;
      MappedFile mappedFile = this.findMappedFileByOffset(this.flushedWhere, this.flushedWhere == 0);
      if (mappedFile != null) {
        long tmpTimeStamp = mappedFile.getStoreTimestamp();
        //真正刷盘
        int offset = mappedFile.flush(flushLeastPages);
        long where = mappedFile.getFileFromOffset() + offset;
        result = where == this.flushedWhere;
        this.flushedWhere = where;
        if (0 == flushLeastPages) {
          this.storeTimestamp = tmpTimeStamp;
        }
      }
    
      return result;
    }
    ```

    

#### ServiceThread

* 封装了原有的线程
* 具有下面特性
  * 当处于stopped状态的时候，不执行任何任务
  * 当不处于stopped状态的时候，用户调用waitForRunning可以启动一段定时器，并阻塞一段时间，或者
  * 使用wakeup()立即结束跑完当前的定时器，立即退出阻塞
* 具体设计理念，见：https://www.jianshu.com/p/2fb3293ed810





## 过期文件删除

* 原因
  * 由于 RocketMQ 操作 CommitLog、ConsumeQueu巳文件是基于内存映射机制并在启动的时候会加载 commitlog、ConsumeQueue 目录下的所有文件，为了避免内存与磁盘的浪费，不可能将消息永久存储在消息服务器上，所以需要引人一种机制来删除己过期的文件
* 原理
  * RocketMQ 顺序写 Commitlog 文件、ConsumeQueue 文件，所有写操作全部落在最后一个CommitLog 、ConsumeQueue文件上，**之前的文件在下一个文件创建后将不会再被更新，只新增不修改**
  * RocketMQ 清除过期文件的方法是 ：
    * 如果非当前写文件在一定时间间隔内没有再次被更新，则认为是过期文件，可以被删除， RocketMQ 不会关注这个文件上的消息是否全部被消费
    * 默认每个文件的过期时间为72 小时（3天），通过在 Broker 配置文件中设置 fi leReservedTime 来改变过期时间，单位为小时 



#### 启动流程

* DefaultMessageStore#addScheduleTask

  * DefaultMessageStore#start() -> DefaultMessageStore#addScheduleTask

* 源码

  ```java
  private void addScheduleTask() {
  
    this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
      @Override
      public void run() {
        
        //清理文件的方法
        DefaultMessageStore.this.cleanFilesPeriodically();
      }
    }, 1000 * 60, this.messageStoreConfig.getCleanResourceInterval(), TimeUnit.MILLISECONDS);
  
    //省略其他定时任务
  }
  ```



### 清理文件

#### DefaultMessageStore

##### cleanFilesPeriodically

```java
private void cleanFilesPeriodically() {
  //清理commitlog文件，此处只是调动方法，没有开启新的线程
  this.cleanCommitLogService.run();

  //清理consumequeue文件，同上
  this.cleanConsumeQueueService.run();
}
```



#### CleanCommitLogService

###### run

```java
public void run() {
  try {
    this.deleteExpiredFiles();

    this.redeleteHangedFile();
  } catch (Throwable e) {
    DefaultMessageStore.log.warn(this.getServiceName() + " service has exception. ", e);
  }
}
```



###### deleteExpireFiles

```java
private void deleteExpiredFiles() {
  int deleteCount = 0;

  //文件保留时间，从最后一次更新时间到现在，如果超过了该时间，则认为是过期文件，可以删除
  long fileReservedTime = DefaultMessageStore.this.getMessageStoreConfig().getFileReservedTime();

  //删除物理文件的间隔，因为在一次清除过程中，可能需要被删除的文件不止一个，该值指定了两次删除文件的间隔时间
  int deletePhysicFilesInterval = DefaultMessageStore.this.getMessageStoreConfig().getDeleteCommitLogFilesInterval();

  //destroyMapedFileIntervalForcibly表示第一次拒绝删除之后能保留的最大时间，在此时间内，同样可以被拒绝删除，同时会将引用减少1000个，超过该时间间隔后，文件将会被删除

  // 在清除过期文件时，如果该文件被其他线程所占用（引用次数大于0，比如读取消息），此时会阻止此次删除任务
  // 同时在第一个试图删除该文件时记录当前时间戳，destroyMapedFileIntervalForcibly表示第一次拒绝删除之后能保留的最大时间，在此时间内，同样可以被拒绝删除，同时会将引用减少1000个，超过该时间间隔后，文件将会被删除
  int destroyMapedFileIntervalForcibly = DefaultMessageStore.this.getMessageStoreConfig().getDestroyMapedFileIntervalForcibly();


  //RocketMQ在三种情况任何一个满足，就会执行删除文件操作
  //1. 指定删除文件的时间点，RocketMQ通过deleteWhen设置一天的固定时间执行一次删除过期文件操作，默认是凌晨4点
  //   原理：此定时任务检查，只要满足下面的条件就去删除文件，原来如此
  boolean timeup = this.isTimeToDelete();

  //2. 磁盘空间是否充足，如果磁盘空间不充足，则返回true，表示应该触发过期文件删除操作
  boolean spacefull = this.isSpaceToDelete();

  //3. 预留，手工触发，可以通过调用excuteDeleteFilesManualy方法手动触发过期文件删除，目前RocketMQ暂未封装手工触发文件删除的命令
  boolean manualDelete = this.manualDeleteFileSeveralTimes > 0;



  if (timeup || spacefull || manualDelete) {

    if (manualDelete)
      this.manualDeleteFileSeveralTimes--;

    boolean cleanAtOnce = DefaultMessageStore.this.getMessageStoreConfig().isCleanFileForciblyEnable() && this.cleanImmediately;

    log.info("begin to delete before {} hours file. timeup: {} spacefull: {} manualDeleteFileSeveralTimes: {} cleanAtOnce: {}",
             fileReservedTime,
             timeup,
             spacefull,
             manualDeleteFileSeveralTimes,
             cleanAtOnce);

    fileReservedTime *= 60 * 60 * 1000;

    //真正清理文件的地方
    deleteCount = DefaultMessageStore.this.commitLog.deleteExpiredFile(fileReservedTime, deletePhysicFilesInterval,
                                                                       destroyMapedFileIntervalForcibly, cleanAtOnce);
    if (deleteCount > 0) {
    } else if (spacefull) {
      log.warn("disk space will be full soon, but delete file failed.");
    }
  }
}
```

###### isTimeToDelete

```java
private boolean isTimeToDelete() {
  //4点
  String when = DefaultMessageStore.this.getMessageStoreConfig().getDeleteWhen();
  if (UtilAll.isItTimeToDo(when)) {
    DefaultMessageStore.log.info("it's time to reclaim disk space, " + when);
    return true;
  }

  return false;
}
```

###### isSpaceToDelete

```java
private boolean isSpaceToDelete() {
  //占用比例，其中diskMaxUsedSpaceRatio 表示commitlog和consumequeue文件所在磁盘分区的最大使用量，如果超过该值，则需要立即清除过期文件
  double ratio = DefaultMessageStore.this.getMessageStoreConfig().getDiskMaxUsedSpaceRatio() / 100.0;

  //是否需要立即执行清除过期文件的操作
  cleanImmediately = false;

  {
    //commitlog的存储路径
    String storePathPhysic = DefaultMessageStore.this.getMessageStoreConfig().getStorePathCommitLog();

    //当前commitlog目录所在的磁盘分区的磁盘使用率，
    //    通过 File#getTotalSpace()获取文件所在磁盘分区的总容量
    //    通过 File#getFreeSpace()获取文件所在磁盘分区的剩余容量
    double physicRatio = UtilAll.getDiskPartitionSpaceUsedPercent(storePathPhysic);

    //diskSpaceWarningLevelRatio: 通过系统参数 -Drocketmq.broker.diskSpaceWarningLevelRatio设置，默认是0.9
    //    如果磁盘分区使用率超过该阈值，将设置磁盘不可写，此时会拒绝新消息的写入
    if (physicRatio > diskSpaceWarningLevelRatio) {
      boolean diskok = DefaultMessageStore.this.runningFlags.getAndMakeDiskFull();
      if (diskok) {
        DefaultMessageStore.log.error("physic disk maybe full soon " + physicRatio + ", so mark disk full");
      }

      cleanImmediately = true;

      //diskSpaceCleanForciblyRatio：通过参数－Drocketmq. broker.diskSpaceCleanForciblyRatio设置，默认为0.85
      //   如果磁盘分区使用超过该阔值，建议立即执行过期文件清除，但不拒绝新消息的写入
    } else if (physicRatio > diskSpaceCleanForciblyRatio) {
      cleanImmediately = true;
    } else {
      //磁盘使用正常，无需清理
      boolean diskok = DefaultMessageStore.this.runningFlags.getAndMakeDiskOK();
      if (!diskok) {
        DefaultMessageStore.log.info("physic disk space OK " + physicRatio + ", so mark disk ok");
      }
    }

    //磁盘需要清理
    if (physicRatio < 0 || physicRatio > ratio) {
      DefaultMessageStore.log.info("physic disk maybe full soon, so reclaim space, " + physicRatio);
      return true;
    }
  }

  {
    //ConsumeQueue的清理
    String storePathLogics = StorePathConfigHelper
      .getStorePathConsumeQueue(DefaultMessageStore.this.getMessageStoreConfig().getStorePathRootDir());
    double logicsRatio = UtilAll.getDiskPartitionSpaceUsedPercent(storePathLogics);
    if (logicsRatio > diskSpaceWarningLevelRatio) {
      boolean diskok = DefaultMessageStore.this.runningFlags.getAndMakeDiskFull();
      if (diskok) {
        DefaultMessageStore.log.error("logics disk maybe full soon " + logicsRatio + ", so mark disk full");
      }

      cleanImmediately = true;
    } else if (logicsRatio > diskSpaceCleanForciblyRatio) {
      cleanImmediately = true;
    } else {
      boolean diskok = DefaultMessageStore.this.runningFlags.getAndMakeDiskOK();
      if (!diskok) {
        DefaultMessageStore.log.info("logics disk space OK " + logicsRatio + ", so mark disk ok");
      }
    }

    if (logicsRatio < 0 || logicsRatio > ratio) {
      DefaultMessageStore.log.info("logics disk maybe full soon, so reclaim space, " + logicsRatio);
      return true;
    }
  }

  return false;
}
```



##### CommitLog

###### deleteExpiredFile

```java
public int deleteExpiredFile(
  final long expiredTime,
  final int deleteFilesInterval,
  final long intervalForcibly,
  final boolean cleanImmediately
) {
  return this.mappedFileQueue.deleteExpiredFileByTime(expiredTime, deleteFilesInterval, intervalForcibly, cleanImmediately);
}
```



##### MappedFileQueue

###### deleteExpiredFileByTime

```java
public int deleteExpiredFileByTime(final long expiredTime,
                                   final int deleteFilesInterval,
                                   final long intervalForcibly,
                                   final boolean cleanImmediately) {
  Object[] mfs = this.copyMappedFiles(0);

  if (null == mfs)
    return 0;

  //倒数第二个文件
  int mfsLength = mfs.length - 1;
  int deleteCount = 0;
  List<MappedFile> files = new ArrayList<MappedFile>();
  if (null != mfs) {

    //从倒数第二个文件开始遍历
    for (int i = 0; i < mfsLength; i++) {
      MappedFile mappedFile = (MappedFile) mfs[i];

      //文件最大的存活时间 = 上一次更新时间 + 文件存储时间（默认72小时）
      long liveMaxTimestamp = mappedFile.getLastModifiedTimestamp() + expiredTime;

      //文件存活已经超了最大的存活时间，就清理
      //前面的磁盘判断需要立刻清理，就执行清理
      if (System.currentTimeMillis() >= liveMaxTimestamp || cleanImmediately) {
        //mappedFile的destroy,清除MappedFile占有的相关资源
        if (mappedFile.destroy(intervalForcibly)) {

          //加入统一删除的文件列表中欧冠
          files.add(mappedFile);
          deleteCount++;

          //每批删除10个文件
          if (files.size() >= DELETE_FILES_BATCH_MAX) {
            break;
          }

          if (deleteFilesInterval > 0 && (i + 1) < mfsLength) {
            try {
              Thread.sleep(deleteFilesInterval);
            } catch (InterruptedException e) {
            }
          }
        } else {
          break;
        }
      } else {
        //avoid deleting files in the middle
        break;
      }
    }
  }

  //删除集合中的所有的文件
  deleteExpiredFile(files);

  return deleteCount;
}
```







###### deleteExpiredFile

```java
void deleteExpiredFile(List<MappedFile> files) {

  if (!files.isEmpty()) {

    Iterator<MappedFile> iterator = files.iterator();
    while (iterator.hasNext()) {
      MappedFile cur = iterator.next();
      if (!this.mappedFiles.contains(cur)) {
        iterator.remove();
        log.info("This mappedFile {} is not contained by mappedFiles, so skip it.", cur.getFileName());
      }
    }

    try {
      if (!this.mappedFiles.removeAll(files)) {
        log.error("deleteExpiredFile remove failed.");
      }
    } catch (Exception e) {
      log.error("deleteExpiredFile has exception.", e);
    }
  }
}
```





## 优秀文章

* 内存映射细节：https://zhuanlan.zhihu.com/p/360912438
* 经典网址：https://blog.csdn.net/yulewo123/article/details/103019288/
