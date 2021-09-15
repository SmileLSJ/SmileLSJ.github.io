## 解决问题

5. 顺序消息原理
6. 顺序消息和并发消息的区别
   1. 顺序消息在创建消息队列拉取任务时需要在Broker服务器锁定该消息队列



## 顺序消息

* RocketMQ支持局部消息顺序消费，可以确保同 个消息消 队列中的消息被顺序消

  费

  * 如果需要做到全局顺序则可以将主题配置成一个队列

* 一条消息消费的包含四个步骤

  * 消息队列负载
  * 消息拉取
  * 消息消费
  * 消息消费进度存储



### 消息队列负载

* RocketMQ 首先需要通过 RebalanceService 线程实现消息队列的负载， 集群模式下同一个消费组 内的消费者共同承担其订阅主题下消息队列的消费，同一个消息消费队列在同一时刻只会被消费组内一个消费者消费， 一个消费者同一时刻可以分配多个消费队列
* 顺序消息和并发消息的第一个关键区别
  * 顺序消息在创建消息队列拉取任务时需要在Broker服务器锁定该消息队列，为何，因为要保证加入队列的顺序性，不能被其他线程先加入消息

* RebalanceImpl#updateProcessQueueTableInRebalance

  * 关键代码

    ```java
    //3. 遍历本次负载分配到的队列集合
    List<PullRequest> pullRequestList = new ArrayList<PullRequest>();
    for (MessageQueue mq : mqSet) {
      //3.1 消息队列时新增加的消息队列：ProcessQueueTable中，没有包含该消息队列
      if (!this.processQueueTable.containsKey(mq)) {
    
        //-----------顺序消息逻辑---------
        //如果是顺序消息，首先需要尝试向Broker发起锁定该消息队列的请求
        //如果返回加锁成功，则创建该消息队列的拉取任务，也就是下面的逻辑
        //否则跳过，等待其他消费者释放该消息队列的锁，然后再下一次队列重新负载时再尝试加锁
        if (isOrder && !this.lock(mq)) {
          log.warn("doRebalance, {}, add a new mq failed, {}, because lock failed", consumerGroup, mq);
          continue;
        }
    
        //3.2 从内存中移除该消息队列的消费进度
        this.removeDirtyOffset(mq);
        ProcessQueue pq = new ProcessQueue();
    
        //3.3 从磁盘中读取该消息队列的消费进度
        long nextOffset = this.computePullFromWhere(mq);
    
        //如果是-1，则pullRequestList仍为空List，不会去拉取消息
        if (nextOffset >= 0) {
          ProcessQueue pre = this.processQueueTable.putIfAbsent(mq, pq);
          if (pre != null) {
            log.info("doRebalance, {}, mq already exists, {}", consumerGroup, mq);
          } else {
    
            //4 创建 拉取消息请求 pullRequest
            log.info("doRebalance, {}, add a new mq, {}", consumerGroup, mq);
            PullRequest pullRequest = new PullRequest();
            pullRequest.setConsumerGroup(consumerGroup);
            pullRequest.setNextOffset(nextOffset);
            pullRequest.setMessageQueue(mq);
            pullRequest.setProcessQueue(pq);
            pullRequestList.add(pullRequest);
            changed = true;
          }
        } else {
          log.warn("doRebalance, {}, add new mq failed, {}", consumerGroup, mq);
        }
      }
    }
    ```

    

### 消息拉取

* RocketMQ消息拉取由PullMessageService线程负责，根据消息拉取任务循环拉取消息

* 关键代码：DefaultMQPushConsumerImpl#pullMessage

  ```java
  if (processQueue.isLocked()) {
    if (!pullRequest.isLockedFirst()) {
      final long offset = this.rebalanceImpl.computePullFromWhere(pullRequest.getMessageQueue());
      boolean brokerBusy = offset < pullRequest.getNextOffset();
      log.info("the first time to pull message, so fix offset from broker. pullRequest: {} NewOffset: {} brokerBusy: {}",
               pullRequest, offset, brokerBusy);
      if (brokerBusy) {
        log.info("[NOTIFYME]the first time to pull message, but pull request offset larger than broker consume offset. pullRequest: {} NewOffset: {}",
                 pullRequest, offset);
      }
  
      pullRequest.setLockedFirst(true);
      pullRequest.setNextOffset(offset);
    }
    //顺序消息的消息处理队列 processQueue 未被锁定，则延迟3s后再将 PullRequest对象放入到拉取任务中
  } else {
    this.executePullRequestLater(pullRequest, pullTimeDelayMillsWhenException);
    log.info("pull message later because not locked in broker, {}", pullRequest);
    return;
  }
  ```

  

### 消息消费

* 顺序消息消费的实现类：

  * org.apache.rocketmq.client.impl.consumer.ConsumeMessageOrderlyService

  

#### ConsumeMessageOrderlyService构造方法

```java
public class ConsumeMessageOrderlyService implements ConsumeMessageService {
    private static final InternalLogger log = ClientLogger.getLog();

    //每次消费任务最大持续时间，默认为60s，可以通过 -Drocketmq.client.maxTimeConsumeContinuously 改变默认值
    private final static long MAX_TIME_CONSUME_CONTINUOUSLY =
        Long.parseLong(System.getProperty("rocketmq.client.maxTimeConsumeContinuously", "60000"));

    //消息消费者实现类
    private final DefaultMQPushConsumerImpl defaultMQPushConsumerImpl;

    //消息消费者
    private final DefaultMQPushConsumer defaultMQPushConsumer;

    //顺序消息消费监听器
    private final MessageListenerOrderly messageListener;

    //消息消费任务队列
    private final BlockingQueue<Runnable> consumeRequestQueue;

    //消息消费线程池
    private final ThreadPoolExecutor consumeExecutor;

    //消费组名
    private final String consumerGroup;

    //消息消费端消息消费队列锁容器，内部持有
    //   ConcurrentMap<MessageQueue, Object> mqLockTable =new ConcurrentHashMap<MessageQueue,Object> （）
    private final MessageQueueLock messageQueueLock = new MessageQueueLock();

    //调度任务线程池
    private final ScheduledExecutorService scheduledExecutorService
    private volatile boolean stopped = false;

    public ConsumeMessageOrderlyService(DefaultMQPushConsumerImpl defaultMQPushConsumerImpl,
        MessageListenerOrderly messageListener) {
        this.defaultMQPushConsumerImpl = defaultMQPushConsumerImpl;
        this.messageListener = messageListener;

        this.defaultMQPushConsumer = this.defaultMQPushConsumerImpl.getDefaultMQPushConsumer();
        this.consumerGroup = this.defaultMQPushConsumer.getConsumerGroup();
        this.consumeRequestQueue = new LinkedBlockingQueue<Runnable>();

        //初始化实例参数
        //   这里的关键是消息任务队列为 LinkedBlockingQueue，消息消费线程池最大运行时线程个数 cnsumeThreadMin consumeThreadMax参数将失效
       this.consumeExecutor = new ThreadPoolExecutor(
            this.defaultMQPushConsumer.getConsumeThreadMin(),
            this.defaultMQPushConsumer.getConsumeThreadMax(),
            1000 * 60,
            TimeUnit.MILLISECONDS,
            this.consumeRequestQueue,
            new ThreadFactoryImpl("ConsumeMessageThread_"));

        this.scheduledExecutorService = Executors.newSingleThreadScheduledExecutor(new ThreadFactoryImpl("ConsumeMessageScheduledThread_"));
    }

  
   //...省略其他方法... 
}
```



#### ConsumeMessageOrderlyService启动方法

```java
public void start() {
  //如果消费模式为集群模式，启动定时任务，默认每隔 20s 执行一次锁定分配给自己的消息消费队列
  //通过 -Drocketmq.client.rebalance. locklnterval =20000 设置间隔，该值建议与一次消息负载频率设置相同
  if (MessageModel.CLUSTERING.equals(ConsumeMessageOrderlyService.this.defaultMQPushConsumerImpl.messageModel())) {
    this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
      @Override
      public void run() {
        ConsumeMessageOrderlyService.this.lockMQPeriodically();
      }
    }, 1000 * 1, ProcessQueue.REBALANCE_LOCK_INTERVAL, TimeUnit.MILLISECONDS);
  }
}
```

* 从上面的消息拉取中可知，集群模式下顺序消息消费在创建拉取任务时并未将 ProcessQueue的locked状态设置为 true，在未锁定消息队列之前无法执行消息拉取任务， ConsumeMessageOrderlyService以每20s 频率对分配给自己的消息队列进行自动加锁操作，从而消费加锁成功的消息消费队列。**也就是只有加锁成功的消费队列，才能在上面的消息拉取中构建出拉取请求，进行进行消息的获取。**

* 消费队列加锁流程

  RebalanceImpl#lockAll

  ```java
  public void lockAll() {
    //1. 转换broker和消息队列的结构
    HashMap<String, Set<MessageQueue>> brokerMqs = this.buildProcessQueueTableByBrokerName();
  
    Iterator<Entry<String, Set<MessageQueue>>> it = brokerMqs.entrySet().iterator();
    while (it.hasNext()) {
      Entry<String, Set<MessageQueue>> entry = it.next();
      final String brokerName = entry.getKey();
      final Set<MessageQueue> mqs = entry.getValue();
  
      if (mqs.isEmpty())
        continue;
  
      FindBrokerResult findBrokerResult = this.mQClientFactory.findBrokerAddressInSubscribe(brokerName, MixAll.MASTER_ID, true);
      if (findBrokerResult != null) {
        LockBatchRequestBody requestBody = new LockBatchRequestBody();
        requestBody.setConsumerGroup(this.consumerGroup);
        requestBody.setClientId(this.mQClientFactory.getClientId());
        requestBody.setMqSet(mqs);
  
        try {
  
          //2. 向Broker（Master主节点）发送锁定消息队列，该方法返回成功被当前消费者锁定的消息消费队列
          Set<MessageQueue> lockOKMQSet =
            this.mQClientFactory.getMQClientAPIImpl().lockBatchMQ(findBrokerResult.getBrokerAddr(), requestBody, 1000);
  
  
          //3. 将成功锁定的消息消费队列相对应的处理队列设置为锁定状态，同时更新加锁时间
          for (MessageQueue mq : lockOKMQSet) {
            ProcessQueue processQueue = this.processQueueTable.get(mq);
            if (processQueue != null) {
              if (!processQueue.isLocked()) {
                log.info("the message queue locked OK, Group: {} {}", this.consumerGroup, mq);
              }
  
              processQueue.setLocked(true);//设置锁定状态
              processQueue.setLastLockTimestamp(System.currentTimeMillis());//更新加锁时间
            }
          }
  
          //4. 遍历当前处理队列中的消息消费队列
          //   如果当前消费者不持有该消息队列的锁，将处理队列锁状态设置为false
          //       暂停该消息消费队列的消息拉取与消息消费
          for (MessageQueue mq : mqs) {
            if (!lockOKMQSet.contains(mq)) {
              ProcessQueue processQueue = this.processQueueTable.get(mq);
              if (processQueue != null) {
                processQueue.setLocked(false);
                log.warn("the message queue locked Failed, Group: {} {}", this.consumerGroup, mq);
              }
            }
          }
        } catch (Exception e) {
          log.error("lockBatchMQ exception, " + mqs, e);
        }
      }
    }
  }
  
  ```

  ```java
  /**
  * 将消息队列按照Broker组织成  HashMap<String, Set<MessageQueue>>
  *     方便下一步向Broker发送锁定消息队列请求
  *
  */
  private HashMap<String/* brokerName */, Set<MessageQueue>> buildProcessQueueTableByBrokerName() {
    HashMap<String, Set<MessageQueue>> result = new HashMap<String, Set<MessageQueue>>();
    for (MessageQueue mq : this.processQueueTable.keySet()) {
      Set<MessageQueue> mqs = result.get(mq.getBrokerName());
      if (null == mqs) {
        mqs = new HashSet<MessageQueue>();
        result.put(mq.getBrokerName(), mqs);
      }
  
      mqs.add(mq);
    }
  
    return result;
  }
  ```



#### ConsumeMessageOrderlyService 提交消费任务

```java
@Override
public void submitConsumeRequest(
  final List<MessageExt> msgs,
  final ProcessQueue processQueue,
  final MessageQueue messageQueue,
  final boolean dispathToConsume) {
  if (dispathToConsume) {
    //构建消费任务 ConsumeRequest，并提交到消费线程池中
    ConsumeRequest consumeRequest = new ConsumeRequest(processQueue, messageQueue);
    this.consumeExecutor.submit(consumeRequest);
  }
}
```



##### ConsumeRequest

```java
class ConsumeRequest implements Runnable {
  //消息处理队列
  private final ProcessQueue processQueue;

  //消息队列
  private final MessageQueue messageQueue;


  public ConsumeRequest(ProcessQueue processQueue, MessageQueue messageQueue) {
    this.processQueue = processQueue;
    this.messageQueue = messageQueue;
  }

  public ProcessQueue getProcessQueue() {
    return processQueue;
  }

  public MessageQueue getMessageQueue() {
    return messageQueue;
  }

  @Override
  public void run() {

    //1. 如果消息处理队列为丢弃，则停止本次消费任务
    if (this.processQueue.isDropped()) {
      log.warn("run, the message queue not be able to consume, because it's dropped. {}", this.messageQueue);
      return;
    }

    //2. 根据消息队列获取一个对象 然后消息消费时先申请独占 objLock。顺序消息消费的并发度为消息队列。
    //   也就是一个消息消费队列同一时刻只会被一个消费线程池中一个线程消费
    final Object objLock = messageQueueLock.fetchLockObject(this.messageQueue);
    synchronized (objLock) {

      //3. 如果是广播模式，直接进入消费，无需锁定处理队列
      if (MessageModel.BROADCASTING.equals(ConsumeMessageOrderlyService.this.defaultMQPushConsumerImpl.messageModel())

          //如果是集群模式，消息消费的前提条件是 processQueue被锁定并且锁未超时
          || (this.processQueue.isLocked() && !this.processQueue.isLockExpired())) {
        final long beginTime = System.currentTimeMillis();
        for (boolean continueConsume = true; continueConsume; ) {

          if (this.processQueue.isDropped()) {
            log.warn("the message queue not be able to consume, because it's dropped. {}", this.messageQueue);
            break;
          }

          if (MessageModel.CLUSTERING.equals(ConsumeMessageOrderlyService.this.defaultMQPushConsumerImpl.messageModel())
              && !this.processQueue.isLocked()) {
            log.warn("the message queue not locked, so consume later, {}", this.messageQueue);
            ConsumeMessageOrderlyService.this.tryLockLaterAndReconsume(this.messageQueue, this.processQueue, 10);
            break;
          }

          if (MessageModel.CLUSTERING.equals(ConsumeMessageOrderlyService.this.defaultMQPushConsumerImpl.messageModel())
              && this.processQueue.isLockExpired()) {
            log.warn("the message queue lock expired, so consume later, {}", this.messageQueue);
            ConsumeMessageOrderlyService.this.tryLockLaterAndReconsume(this.messageQueue, this.processQueue, 10);
            break;
          }

          //4. 顺序消息消费处理逻辑，每一个 ConsumeRequest 消费任务不是以消费消息条数来计算的，
          //   而是根据消费时间，默认当消费时长大于 MAX_TIME_CONSUME_CONTINUOUSLY ，默认
          //   60s 后，本次消费任务结束，由消费组内其他线程继续消费
          long interval = System.currentTimeMillis() - beginTime;
          if (interval > MAX_TIME_CONSUME_CONTINUOUSLY) {
            ConsumeMessageOrderlyService.this.submitConsumeRequestLater(processQueue, messageQueue, 10);
            break;
          }


          //5.每次从处理队列中按顺序取出 consumeBatchSize ，如果未取到消息，
          //  continueConsume为false ，本次消费任务结束
          //  从ProcessQueue中取出的消息，会临时存储在 ProcessQueue的consumingMsgOrderlyTreeMap 属性中
          final int consumeBatchSize =
            ConsumeMessageOrderlyService.this.defaultMQPushConsumer.getConsumeMessageBatchMaxSize();

          List<MessageExt> msgs = this.processQueue.takeMessags(consumeBatchSize);
          defaultMQPushConsumerImpl.resetRetryAndNamespace(msgs, defaultMQPushConsumer.getConsumerGroup());
          if (!msgs.isEmpty()) {
            final ConsumeOrderlyContext context = new ConsumeOrderlyContext(this.messageQueue);

            ConsumeOrderlyStatus status = null;

            ConsumeMessageContext consumeMessageContext = null;

            //6. 执行消息消费钩子函数（消息消费之前before方法）
            //   通过 DefaultMQPushConsumerimpl#registerConsumeMessageHook ( ConsumeMessageHook consumeMessagehook ）
            //   注册消息消费钩子函数并可以注册多个
            if (ConsumeMessageOrderlyService.this.defaultMQPushConsumerImpl.hasHook()) {
              consumeMessageContext = new ConsumeMessageContext();
              consumeMessageContext
                .setConsumerGroup(ConsumeMessageOrderlyService.this.defaultMQPushConsumer.getConsumerGroup());
              consumeMessageContext.setNamespace(defaultMQPushConsumer.getNamespace());
              consumeMessageContext.setMq(messageQueue);
              consumeMessageContext.setMsgList(msgs);
              consumeMessageContext.setSuccess(false);
              // init the consume context type
              consumeMessageContext.setProps(new HashMap<String, String>());
              ConsumeMessageOrderlyService.this.defaultMQPushConsumerImpl.executeHookBefore(consumeMessageContext);
            }

            long beginTimestamp = System.currentTimeMillis();
            ConsumeReturnType returnType = ConsumeReturnType.SUCCESS;
            boolean hasException = false;
            try {

              //7. 申请消息消费锁，
              //   如果消息队列被丢弃，放弃该消息消费队列的消费
              //   如果未被丢弃，执行消息消费监听器，调用业务方具体消息监听器执行真正的消息消费处理逻辑，并通知RocketMQ消息消费结果
              this.processQueue.getLockConsume().lock();
              if (this.processQueue.isDropped()) {
                log.warn("consumeMessage, the message queue not be able to consume, because it's dropped. {}",
                         this.messageQueue);
                break;
              }

              status = messageListener.consumeMessage(Collections.unmodifiableList(msgs), context);
            } catch (Throwable e) {
              log.warn("consumeMessage exception: {} Group: {} Msgs: {} MQ: {}",
                       RemotingHelper.exceptionSimpleDesc(e),
                       ConsumeMessageOrderlyService.this.consumerGroup,
                       msgs,
                       messageQueue);
              hasException = true;
            } finally {
              this.processQueue.getLockConsume().unlock();
            }

            if (null == status
                || ConsumeOrderlyStatus.ROLLBACK == status
                || ConsumeOrderlyStatus.SUSPEND_CURRENT_QUEUE_A_MOMENT == status) {
              log.warn("consumeMessage Orderly return not OK, Group: {} Msgs: {} MQ: {}",
                       ConsumeMessageOrderlyService.this.consumerGroup,
                       msgs,
                       messageQueue);
            }

            long consumeRT = System.currentTimeMillis() - beginTimestamp;
            if (null == status) {
              if (hasException) {
                returnType = ConsumeReturnType.EXCEPTION;
              } else {
                returnType = ConsumeReturnType.RETURNNULL;
              }
            } else if (consumeRT >= defaultMQPushConsumer.getConsumeTimeout() * 60 * 1000) {
              returnType = ConsumeReturnType.TIME_OUT;
            } else if (ConsumeOrderlyStatus.SUSPEND_CURRENT_QUEUE_A_MOMENT == status) {
              returnType = ConsumeReturnType.FAILED;
            } else if (ConsumeOrderlyStatus.SUCCESS == status) {
              returnType = ConsumeReturnType.SUCCESS;
            }

            if (ConsumeMessageOrderlyService.this.defaultMQPushConsumerImpl.hasHook()) {
              consumeMessageContext.getProps().put(MixAll.CONSUME_CONTEXT_TYPE, returnType.name());
            }

            if (null == status) {
              status = ConsumeOrderlyStatus.SUSPEND_CURRENT_QUEUE_A_MOMENT;
            }

            //8. 执行消息消费钩子函数
            if (ConsumeMessageOrderlyService.this.defaultMQPushConsumerImpl.hasHook()) {
              consumeMessageContext.setStatus(status.toString());
              consumeMessageContext
                .setSuccess(ConsumeOrderlyStatus.SUCCESS == status || ConsumeOrderlyStatus.COMMIT == status);
              ConsumeMessageOrderlyService.this.defaultMQPushConsumerImpl.executeHookAfter(consumeMessageContext);
            }

            ConsumeMessageOrderlyService.this.getConsumerStatsManager()
              .incConsumeRT(ConsumeMessageOrderlyService.this.consumerGroup, messageQueue.getTopic(), consumeRT);

            //9. 如果消息消费结果为 ConsumeOrderlyStatus.SUCCESS ， 执行 ProcessQueue的 commit 方法，并返回待跟新的消息消费进度
            continueConsume = ConsumeMessageOrderlyService.this.processConsumeResult(msgs, status, context, this);
          } else {
            continueConsume = false;
          }
        }
      } else {
        if (this.processQueue.isDropped()) {
          log.warn("the message queue not be able to consume, because it's dropped. {}", this.messageQueue);
          return;
        }

        ConsumeMessageOrderlyService.this.tryLockLaterAndReconsume(this.messageQueue, this.processQueue, 100);
      }
    }
  }
```



#### 消息队列锁实现

* 顺序消息消费的各个环节基本都是围绕消息消费队列（ MessageQueue ）与消息处理队列（ProceeQueue） 展开的

* 消息消费进度拉取，消息进度消费都 判断 ProceeQueue的locked 是否为 true ，设置 ProceeQueue为 true 的前提条件是消息消费者（ cid ）向 Broker端发送锁定消息队列的请求并返回加锁成功
* 服务端关于MessageQueue加锁处理类
  * org.apache.rocketmq.broker.client.rebalance.RebalanceLockManager

* 关键属性

  ```java
  
  //锁最大存活时间，默认60s
  private final static long REBALANCE_LOCK_MAX_LIVE_TIME = Long.parseLong(System.getProperty(
    "rocketmq.broker.rebalance.lockMaxLiveTime", "60000"));
  private final Lock lock = new ReentrantLock();
  
  //锁容器，以消息消费组分组，每个消息队列对应一个锁对象，表示当前该消息队列被消费组中哪个消费者所持有
  private final ConcurrentMap<String/* group */, ConcurrentHashMap<MessageQueue, LockEntry>> mqLockTable =
    new ConcurrentHashMap<String, ConcurrentHashMap<MessageQueue, LockEntry>>(1024);
  ```

* 重要方法

  ```java 
      /**
       * 申请对 mqs 消息消费队列集合加锁
       * @param group 消息消费组名
       * @param mqs 待加锁的消息消费队列集合
       * @param clientId 消息消费者(cid)
       * @return 返回成功加锁的消息队列集合
       */
      public Set<MessageQueue> tryLockBatch(final String group, final Set<MessageQueue> mqs,
          final String clientId) {...}
  
  
      /**
       * 申请对 mqs 消息消费队列集合解锁
       * @param group 消息消费组名
       * @param mqs 待解锁的消息队列集合
       * @param clientId 持有锁的消息消费者
       */
      public void unlockBatch(final String group, final Set<MessageQueue> mqs, final String clientId) {...}
  ```

  

# 整体总结

* RocketMQ 消息消费方式分别为 群模式与广播模式 集群模式

* 消息队列负载由 Rebalanceservice 线程默认每隔 20s 进行一次消息队列负载，根据当前消费组内消费者个数与主题队列数 按照某一种负载算法进行队列分配，分配原则为

  * 同一个消费者可以分配多个消息消费队列，
  * 同一个消息消费队列同一时间只会分配给一个消费者

* 消息拉取由 PullMessageService 线程根据 RebalanceService 线程创建的拉取任务进行拉取，默认一批拉取 32 条消息，提交给消费者消费线程池后继续下一次的消息拉取。

  * 如果消息消费过慢产生消息堆积会触发消息消费拉取流控 
  * PullMessageServicve与RebalanceService 线程的交互图如图所示

    <img src="../../../../照片/typora/image-20210822155914468.png" alt="image-20210822155914468" style="zoom:50%;" />

* 并发消息消费指消费线程池中的线程可以并发地对同一个消息消费队列的消息进行消费，消费成功后，取出消息处理队列中最小的消息偏移量作为消息消费进度偏移量，存在于消息消费进度存储文件中，

  * 集群模式消息进度存储在 Broker （消息服务器）
  * 广播模式消息进度存储在消费者端 

* 如果业务方返回 RECONSUME_LATER ，则 RocketMQ 启用消息消费重试机制，

  * 将原消息的主题与队列存储在消息属性中，
  * 将消息存储在主题名为SCHEDULE_TOPIC_XXXX 的消息消费队列中，等待指定时间后， RocketMQ 将自动将该消息重新拉取并再次将消息存储在 commitlog ，进而转发到原主要的消息消费队列供消费者消费，消息消费重试主题为 %RETRY% 消费者组名

* RocketMQ 不支持任意精度的定时调度消息，只支持自定义的消息延迟级别，例如 ls、2s、 5s 等，可通过在 broker 配置文件中设置 messageDeLayLeve。

  *  其实现原理是 RocketMQ为这些延迟级别定义对应的消息消费队列，
  * 其主题为 SCHEDULE_TOPIC_XXXX 
  * 然后创建对应延迟级别的定时任务从消息消费队列中将消息拉取，并恢复消息的原主题与原消息消费队列再次存入 commitlog 文件并转发到相应的消息消费队列，以便消息消费者拉取消息并消费

* 顺序消息消费一般使用集群模式，是指消息消费者内的线程池中的线程对消息消费队列只能串行消费。与**并发消息消费最本质的区别是消费消息时必须成功锁定消息消费队列，Broke 端会存储消息消费队列的锁占用情况**

