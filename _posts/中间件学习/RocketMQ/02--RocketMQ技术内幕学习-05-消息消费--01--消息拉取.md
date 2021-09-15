131 -> 200



## 知识点

* 消费逻辑图

  <img src="../../../../照片/typora/13.png" alt="消费逻辑图"  />

* 消费逻辑精简的顺序图

  <img src="../../../../照片/typora/04.png" alt="Consumer&Broker消费精简图.png" style="zoom:67%;" />



## 重要结构

* ConsumeQueue结构

  ![ConsumeQueue、MappedFileQueue、MappedFile的关系](../../../../照片/typora/03-20210827105523684.png)

  * `ConsumeQueue` : `MappedFileQueue` : `MappedFile` = 1 : 1 : N

* 反应到系统中

  ```java
  Yunai-MacdeMacBook-Pro-2:consumequeue yunai$ pwd
  /Users/yunai/store/consumequeue
  Yunai-MacdeMacBook-Pro-2:consumequeue yunai$ cd TopicRead3/
  Yunai-MacdeMacBook-Pro-2:TopicRead3 yunai$ ls -ls
  total 0
  0 drwxr-xr-x  3 yunai  staff  102  4 27 21:52 0
  0 drwxr-xr-x  3 yunai  staff  102  4 27 21:55 1
  0 drwxr-xr-x  3 yunai  staff  102  4 27 21:55 2
  0 drwxr-xr-x  3 yunai  staff  102  4 27 21:55 3
  Yunai-MacdeMacBook-Pro-2:TopicRead3 yunai$ cd 0/
  Yunai-MacdeMacBook-Pro-2:0 yunai$ ls -ls
  total 11720
  11720 -rw-r--r--  1 yunai  staff  6000000  4 27 21:55 00000000000000000000
  ```

  * `MappedFile` ：00000000000000000000等文件
  * `MappedFileQueue` ：`MappedFile` 所在的文件夹，对 `MappedFile` 进行封装成文件队列，对上层提供可无限使用的文件容量
    * 每个 `MappedFile` 统一文件大小
    * 文件命名方式：fileName[n] = fileName[n - 1] + mappedFileSize。在 `ConsumeQueue` 里默认为 6000000B，啥意思，就是代理地址信息
  * `ConsumeQueue` ：针对 `MappedFileQueue` 的封装使用









## 消息消费者

* 接口 MQConsumer -> MQPushConsumer

* 类图

  ![image-20210813131921371](../../../../照片/typora/image-20210813131921371.png)

  * sendMessageBack：发送消息ACK确认
  * fetchSubscribeMessageQueues：获取消息者对主题topic分配了哪些消息队列
  * registerMessageListener(final MessageListenerConcurrently messageListener)：注册并发消息事件监听器
  * registerMessageListener(final MessageListenerOrderly messageListener)：注册顺序消息事件监听器
  * subscribe：订阅消息
  * unsubscribe：取消订阅消息





### DefaultMQPushConsumer

* 类图

  <img src="../../../../照片/typora/image-20210813133342012.png" alt="image-20210813133342012" style="zoom:50%;" />

* 重要属性
  1. consumerGroup：消费者所属组
  2. messageModel：消息消费模式，分为集群模式，广播模式，默认为集群模式
  3. consumeFromWhere：根据消息进度从消息服务器拉取不到消息时重新计算消息策略。注意：如果从消息进度服务OffsetStore读取到MessageQueue中的偏移量不小于0，则使用读取到的偏移量，只有在读到的偏移量小于0时，下面的策略才会生效
     1. CONSUME_FROM_LAST_OFFSE ：从队列当前最大偏移量开始消费
     2. CONSUME_FROM_FIRST_OFFSET ：从队列当前 小偏移量开始消费
     3. CONSUME_FROM_TIMESTAMP ：从消费者启动时间戳开始消费
  4. allocateMessageQueueStrategy ：集群模式下消息队列负载策略
  5. Map<String /* topic */， String /* sub expression */> subscription ：订阅信息
  6. MessageListener messageListener ：消息业务监听器
  7. Private OffsetStore offsetStore：消息消费进度存储器
  8. int consumeThreadMin = 20，消费者最新线程数
  9. int consumeThreadMax = 64，消费者最大线程数，由于消费者线程池使用无界队列，消费者线程个数其实最多只有 consumeThreadMin个
  10.  consumeConcurrentlyMaxSp ，并发消息消费时处理队列最大跨度，默认2000，表示如果消息处理队列中偏移量最大的消息与偏移量最小的消息的跨度超过2000 延迟50毫秒后再拉取消息
  11. int pul!ThresholdForQueue：默认值 1000，每1000次流控后打印流控日志，
      1. **如何实现的呢？？？semphore???**
  12. long pulllnterval = 0,推模式 拉取任务间隔时间，默认一次拉取任务完成继续拉取
  13. int pullBatchSize： 每次消息拉取所拉取的条数，默认 32条
  14. int consumeMessageBatchMaxSize ：消息并发消费时一次消费消息条数，通俗点说就是每次传入 MessageListten#consum#Message 的消息条数
  15. postSubscriptionWhenPull ：是否每次拉取消息都更新订阅信息，默认为 false。
  16. maxReconsumeTimes 最大消费重试次数 如果消息消费次数超过 maxReconsumeTimes 还未成功，则将该消息转移到一个失败队列 ，等待被删除。
      1. **如何进行重试的，是和Producer一样，使用for循环吗？？？**
  17.  suspendCurrentQueueTimeMillis ：延迟将该队列的消息提交到消费者线程的等待时间， 默认延迟 1s
  18. long consumeTimeout，消息消费超时时间，默认为15，单位为分钟



## 消费者启动里路程

* 入口

  * DefaultMQPushConsumerImpl#start()方法

* 启动的过程中，依次干了以下几件事情

  1. 构建主题订阅信息 SubscriptionData 并加入到 Rebalancelmpl 的订阅消息中，用于后续的负载均衡执行
  2. 初始化 MQC!ientlnstance Rebalancelmple （消息重新负载实现类）等
  3. 初始化消息进度 
     1. 如果消息消费是集群模式，那么消息进度保存在 Broker 上；
     2. 如果是广播模式，那么消息消费进度存储在消费端 
  4. 根据是否是顺序消费，创建消费端消费线程服务 ConsumeMessageService主要负责消息消费，内部维护一个线程池
  5. 向 MQClientlnstance 注册消费者，并启动 MQClientlnstance 在一JVM 中的所有消费者、生产者持有同一个MQClientlnstance, MQClientlnstance 只会启动一次

* 源码

  ```java
  public synchronized void start() throws MQClientException {
    switch (this.serviceState) {
      case CREATE_JUST:
        log.info("the consumer [{}] start beginning. messageModel={}, isUnitMode={}", this.defaultMQPushConsumer.getConsumerGroup(),
                 this.defaultMQPushConsumer.getMessageModel(), this.defaultMQPushConsumer.isUnitMode());
        this.serviceState = ServiceState.START_FAILED;
  
        this.checkConfig();
  
        //1. 构建主题订阅信息SubscriptionData并加入到RebalanceImpl的订阅关系中
        this.copySubscription();
  
  
        //2. 初始化MQClientInstance、RebalanceImpl(消息重新负载)
        if (this.defaultMQPushConsumer.getMessageModel() == MessageModel.CLUSTERING) {
          this.defaultMQPushConsumer.changeInstanceNameToPID();
        }
  
        //2.1 初始化  MQClientInstance  mQClientFactory
        this.mQClientFactory = MQClientManager.getInstance().getOrCreateMQClientInstance(this.defaultMQPushConsumer, this.rpcHook);
  
        //2.2 初始化  RebalanceImpl
        this.rebalanceImpl.setConsumerGroup(this.defaultMQPushConsumer.getConsumerGroup());
        this.rebalanceImpl.setMessageModel(this.defaultMQPushConsumer.getMessageModel());
        this.rebalanceImpl.setAllocateMessageQueueStrategy(this.defaultMQPushConsumer.getAllocateMessageQueueStrategy());
        this.rebalanceImpl.setmQClientFactory(this.mQClientFactory);
  
        this.pullAPIWrapper = new PullAPIWrapper(
          mQClientFactory,
          this.defaultMQPushConsumer.getConsumerGroup(), isUnitMode());
        this.pullAPIWrapper.registerFilterMessageHook(filterMessageHookList);
  
  
  
        //3. 初始化消息进度。
        //   如果消息消费是集群模式，那么消息进度保存在Broker上 RemoteBrokerOffsetStore
        //   如果消息消费是广播模式，那么消息进度保存在消费端上，LocalFileOffsetStore
        if (this.defaultMQPushConsumer.getOffsetStore() != null) {
          this.offsetStore = this.defaultMQPushConsumer.getOffsetStore();
        } else {
          switch (this.defaultMQPushConsumer.getMessageModel()) {
  
              //广播模式
            case BROADCASTING:
              this.offsetStore = new LocalFileOffsetStore(this.mQClientFactory, this.defaultMQPushConsumer.getConsumerGroup());
              break;
  
              //集群模式
            case CLUSTERING:
              this.offsetStore = new RemoteBrokerOffsetStore(this.mQClientFactory, this.defaultMQPushConsumer.getConsumerGroup());
              break;
            default:
              break;
          }
          this.defaultMQPushConsumer.setOffsetStore(this.offsetStore);
        }
        this.offsetStore.load();
  
  
        //4. 根据是否是顺序消费，创建消费端线程服务
        if (this.getMessageListenerInner() instanceof MessageListenerOrderly) {
          this.consumeOrderly = true;
  
          
          //ConsumeMessageService 主要负责消息消费，内部维护一个线程池
          this.consumeMessageService =
            new ConsumeMessageOrderlyService(this, (MessageListenerOrderly) this.getMessageListenerInner());
        } else if (this.getMessageListenerInner() instanceof MessageListenerConcurrently) {
          this.consumeOrderly = false;
          this.consumeMessageService =
            new ConsumeMessageConcurrentlyService(this, (MessageListenerConcurrently) this.getMessageListenerInner());
        }
  
        this.consumeMessageService.start();
  
  
        //5. 想MQClientInstance注册消费者，并且启动MQClientInstance。
        //   在一个JVM中的所有消费者，生成者持有同一个MQClientInstance，MQClientInstance只会启动一次
        boolean registerOK = mQClientFactory.registerConsumer(this.defaultMQPushConsumer.getConsumerGroup(), this);
        if (!registerOK) {
          this.serviceState = ServiceState.CREATE_JUST;
          this.consumeMessageService.shutdown();
          throw new MQClientException("The consumer group[" + this.defaultMQPushConsumer.getConsumerGroup()
                                      + "] has been created before, specify another name please." + FAQUrl.suggestTodo(FAQUrl.GROUP_NAME_DUPLICATE_URL),
                                      null);
        }
  
        
        //此方法启动了，用于拉取消息的服务，也是ServiceThread
        mQClientFactory.start();
        log.info("the consumer [{}] start OK.", this.defaultMQPushConsumer.getConsumerGroup());
        this.serviceState = ServiceState.RUNNING;
        break;
      case RUNNING:
      case START_FAILED:
      case SHUTDOWN_ALREADY:
        throw new MQClientException("The PushConsumer service state not OK, maybe started once, "
                                    + this.serviceState
                                    + FAQUrl.suggestTodo(FAQUrl.CLIENT_SERVICE_NOT_OK),
                                    null);
      default:
        break;
    }
  
    this.updateTopicSubscribeInfoWhenSubscriptionChanged();
    this.mQClientFactory.checkClientInBroker();
    this.mQClientFactory.sendHeartbeatToAllBrokerWithLock();
    this.mQClientFactory.rebalanceImmediately();
  }
  ```

  ```java
  private void copySubscription() throws MQClientException {
    try {
  
      //========1. 构建主题订阅消息SubscriptionData并加入到 RebalanceImpl的订阅消息中
      /*
                  订阅关系来源有两个
                      1. 通过调用DefaultMQPushConsumerlmpl#subscrib巳（ String topic, String subExpression) 方法
                      2. 订阅重试主题消息。从这里可以 出， RocketMQ 消息重试是以消费组为单位，而不是主题，消息重试主题名为 %RETRY%＋消费组名 消费者在启动的时候会自动订阅该
                         主题，参与该主题的消息队列负载
               */
      Map<String, String> sub = this.defaultMQPushConsumer.getSubscription();
      if (sub != null) {
        for (final Map.Entry<String, String> entry : sub.entrySet()) {
          final String topic = entry.getKey();
          final String subString = entry.getValue();
          SubscriptionData subscriptionData = FilterAPI.buildSubscriptionData(this.defaultMQPushConsumer.getConsumerGroup(),
                                                                              topic, subString);
          this.rebalanceImpl.getSubscriptionInner().put(topic, subscriptionData);
        }
      }
  
      if (null == this.messageListenerInner) {
        this.messageListenerInner = this.defaultMQPushConsumer.getMessageListener();
      }
  
      switch (this.defaultMQPushConsumer.getMessageModel()) {
        case BROADCASTING:
          break;
        case CLUSTERING:
  
          //重试主题
          final String retryTopic = MixAll.getRetryTopic(this.defaultMQPushConsumer.getConsumerGroup());
          SubscriptionData subscriptionData = FilterAPI.buildSubscriptionData(this.defaultMQPushConsumer.getConsumerGroup(),
                                                                              retryTopic, SubscriptionData.SUB_ALL);
          this.rebalanceImpl.getSubscriptionInner().put(retryTopic, subscriptionData);
          break;
        default:
          break;
      }
    } catch (Exception e) {
      throw new MQClientException("subscription exception", e);
    }
  }
  ```

  MQClientInstance#start()

  ```java
  public void start() throws MQClientException {
  
    synchronized (this) {
      switch (this.serviceState) {
        case CREATE_JUST:
          this.serviceState = ServiceState.START_FAILED;
          // If not specified,looking address from name server
          if (null == this.clientConfig.getNamesrvAddr()) {
            this.mQClientAPIImpl.fetchNameServerAddr();
          }
          // Start request-response channel
          this.mQClientAPIImpl.start();
          // Start various schedule tasks
          this.startScheduledTask();
          
          //拉取消息的服务，线程
          // Start pull service
          this.pullMessageService.start();
          
          //负载均衡的服务，线程
          // Start rebalance service
          this.rebalanceService.start();
          // Start push service
          this.defaultMQProducer.getDefaultMQProducerImpl().start(false);
          log.info("the client factory [{}] start OK", this.clientId);
          this.serviceState = ServiceState.RUNNING;
          break;
        case START_FAILED:
          throw new MQClientException("The Factory object[" + this.getClientId() + "] has been created before, and failed.", null);
        default:
          break;
      }
    }
  }
  ```

  

  * 总结
    * 一个消费者组拥有一个consumeMessageService，内部包含一个线程池，用于进行消息消费







## 消息拉取

* 问题
  * 集群模式下，一个消费者组有多个消息消费者，同一个主题存在多个消费队列，那么消费者如何进行消息负载均衡？
  * 每个消费组，拥有一个consumeMessageService，consumeMessageService内维护一个线程池来消费消息，那么这些线程又是如何分工合作的呢？





### PullMessageService实现机制

#### PullMessageService拉取消息

* PullMessageService主要是用来负责消息的拉取

* 类图

  <img src="../../../../照片/typora/image-20210813150130825.png" alt="image-20210813150130825" style="zoom:50%;" />

  * 继承ServiceThread的类，虽然从继承关系上来看，是runnable，但是在ServiceThread中，为每个子类都提供了内部的线程Thread。所以，只要执行了start()方法，那么就是启动了一个线程来执行此任务

  * ServiceThread#start()

    ```java
    public void start() {
      log.info("Try to start service thread:{} started:{} lastThread:{}", getServiceName(), started.get(), thread);
      if (!started.compareAndSet(false, true)) {
        return;
      }
      stopped = false;
      this.thread = new Thread(this, getServiceName());
      this.thread.setDaemon(isDaemon);
      this.thread.start();
    }
    ```

* run方法

  ```java
  @Override
  public void run() {
    log.info(this.getServiceName() + " service started");
  
    //stoped声明为volatile，每执行一次业务逻辑检测一下其运行状态，可以通过其他线程将stopped设置为true，从而停止该线程
    while (!this.isStopped()) {
      try {
  
        //从 pullRequestQueue（LinkedBlockingQueue） 中获取一个PullRequest消息拉取任务，如果 pullRequestQueue为空，
        // 则线程将阻塞，直到有拉取任务被放入
        PullRequest pullRequest = this.pullRequestQueue.take();
  
        //调用pullMessage方法进行消息拉取
        this.pullMessage(pullRequest);
      } catch (InterruptedException ignored) {
      } catch (Exception e) {
        log.error("Pull Message Service Run Method exception", e);
      }
    }
  
    log.info(this.getServiceName() + " service end");
  }
  ```

  * 疑问点

    * PullRequest是什么时间加入到阻塞队列中的呢？

      * 查看方法，只有一个executePullRequestImmediately增加了请求，**其中有个延迟加入队列的方法，后面需要关注**

        ```java
        //延迟加入队列，结果不就是延迟消费吗
        public void executePullRequestLater(final PullRequest pullRequest, final long timeDelay) {
          if (!isStopped()) {
            this.scheduledExecutorService.schedule(new Runnable() {
              @Override
              public void run() {
                PullMessageService.this.executePullRequestImmediately(pullRequest);
              }
            }, timeDelay, TimeUnit.MILLISECONDS);
          } else {
            log.warn("PullMessageServiceScheduledThread has shutdown");
          }
        }
        
        
        //立刻加入队列，立刻消费
        public void executePullRequestImmediately(final PullRequest pullRequest) {
          try {
            this.pullRequestQueue.put(pullRequest);
          } catch (InterruptedException e) {
            log.error("executePullRequestImmediately pullRequestQueue.put", e);
          }
        }
        ```

      * 调用地方

        ![image-20210813152622022](../../../../照片/typora/image-20210813152622022.png)

  * PullReqeust对象

    * 类图

    <img src="../../../../照片/typora/image-20210813152904240.png" alt="image-20210813152904240" style="zoom:50%;" />

    * 源码

    ```java
    public class PullRequest {
    
        //消费者组
        private String consumerGroup;
    
        //待拉取消费队列
        private MessageQueue messageQueue;
    
        //消息处理队列，从Broker拉取到的消息先放入ProcessQueue，然后再提交到消费者消费线程池消费
        private ProcessQueue processQueue;
    
        //待拉取的MessageQueue偏移量
        private long nextOffset;
    
        //是否被锁定
        private boolean lockedFirst = false;
     
      	
      	//省略其他方法
      
    }  
    ```

* PullMessageService#pullMessage

  * 源码

    ```java
        private void pullMessage(final PullRequest pullRequest) {
            //1. 根据消费组名从MQClientInstance中获取消费者内部实现类 MQConsumerInner
            final MQConsumerInner consumer = this.mQClientFactory.selectConsumer(pullRequest.getConsumerGroup());
            if (consumer != null) {
    
                //2. 此处强转换成 DefaultMQPushConsumerImpl
                //   说明：PullMessageService，该线程只为PUSH模式服务，而PULL模式，RocketMQ提供了拉取消息的API就可以了
                DefaultMQPushConsumerImpl impl = (DefaultMQPushConsumerImpl) consumer;
              
                //此处进行拉取任务
                impl.pullMessage(pullRequest);
            } else {
                log.warn("No matched consumer for the PullRequest {}, drop it", pullRequest);
            }
        }
    ```

### ProcessQueue实现机制

* ProcessQueue是MessageQueue在消费端的重现、快照

* PullMessageService从消息服务器默认每次拉取32条消息，按消息的**队列偏移量**顺序存放在ProcessQueue中，PullMessageService然后将消息提交到消费者消费线程池，消息消费成功后从ProcessQueue中移除。

* 重要属性和重要方法

* 源码

  ```java
  public class ProcessQueue {
      public final static long REBALANCE_LOCK_MAX_LIVE_TIME =
          Long.parseLong(System.getProperty("rocketmq.client.rebalance.lockMaxLiveTime", "30000"));
      public final static long REBALANCE_LOCK_INTERVAL = Long.parseLong(System.getProperty("rocketmq.client.rebalance.lockInterval", "20000"));
      private final static long PULL_MAX_IDLE_TIME = Long.parseLong(System.getProperty("rocketmq.client.pull.pullMaxIdleTime", "120000"));
      private final InternalLogger log = ClientLogger.getLog();
  
  
      //读写锁，控制多线程并发修改  msgTreeMap
      private final ReadWriteLock lockTreeMap = new ReentrantReadWriteLock();
  
  
      //--------顺序消息存储容器---------
      //消息存储容器，key为消息在ConsumeQueue中的偏移量，MessageExt为消息实体
  
      private final TreeMap<Long, MessageExt> msgTreeMap = new TreeMap<Long, MessageExt>();
  
      //ProcessQueue中的总消息数
      private final AtomicLong msgCount = new AtomicLong();
  
      //ProcessQueue中的总消息大小
      private final AtomicLong msgSize = new AtomicLong();
      private final Lock lockConsume = new ReentrantLock();
  
  
      /**
       * A subset of msgTreeMap, will only be used when orderly consume
       */
      //msgTreeMap的副本，仅仅用来存储顺序消息
      //该结构用于处理顺序消息，消息消费线程从ProcessQueue的msgTreeMap中取出消息前，先将消息临时存储在consumingMsgOrderlyTreeMap中
      private final TreeMap<Long, MessageExt> consumingMsgOrderlyTreeMap = new TreeMap<Long, MessageExt>();
      private final AtomicLong tryUnlockTimes = new AtomicLong(0);
      private volatile long queueOffsetMax = 0L;
  
      //当前ProcessQueue是否被丢弃
      private volatile boolean dropped = false;
  
      //上一次开始消息拉取时间戳
      private volatile long lastPullTimestamp = System.currentTimeMillis();
  
      //上一次消息消费时间戳
      private volatile long lastConsumeTimestamp = System.currentTimeMillis();
  
  
      private volatile boolean locked = false;
      private volatile long lastLockTimestamp = System.currentTimeMillis();
      private volatile boolean consuming = false;
      private volatile long msgAccCnt = 0;
  
      /**
       * 判断锁是否过期
       *   锁超时时间默认为30s
       *   可以通过系统参数 rocketmq.client.rebalance.lockMaxLiveTime来设置
       * @return
       */
      public boolean isLockExpired() {
          return (System.currentTimeMillis() - this.lastLockTimestamp) > REBALANCE_LOCK_MAX_LIVE_TIME;
      }
  
  
      /**
       * 判断PullMessageService是否空闲
       *    默认120s,2min
       *    可以通过系统参数 rocketmq.client.pull.pullMaxIdleTime来设置
       * @return
       */
      public boolean isPullExpired() {
          return (System.currentTimeMillis() - this.lastPullTimestamp) > PULL_MAX_IDLE_TIME;
      }
  
  
  
      //------------------存在自动延迟的机制--------------------
      //TODO 如何实现呢
      /**
       * 移除消费超时的消息
       *  默认超过15分钟未消费的消息将延迟3个延迟级别再消费
       * @param pushConsumer
       */
      public void cleanExpiredMsg(DefaultMQPushConsumer pushConsumer) {
          if (pushConsumer.getDefaultMQPushConsumerImpl().isConsumeOrderly()) {
              return;
          }
  
          int loop = msgTreeMap.size() < 16 ? msgTreeMap.size() : 16;
          for (int i = 0; i < loop; i++) {
              MessageExt msg = null;
              try {
                  this.lockTreeMap.readLock().lockInterruptibly();
                  try {
                      if (!msgTreeMap.isEmpty() && System.currentTimeMillis() - Long.parseLong(MessageAccessor.getConsumeStartTimeStamp(msgTreeMap.firstEntry().getValue())) > pushConsumer.getConsumeTimeout() * 60 * 1000) {
                          msg = msgTreeMap.firstEntry().getValue();
                      } else {
  
                          break;
                      }
                  } finally {
                      this.lockTreeMap.readLock().unlock();
                  }
              } catch (InterruptedException e) {
                  log.error("getExpiredMsg exception", e);
              }
  
              try {
  
                  pushConsumer.sendMessageBack(msg, 3);
                  log.info("send expire msg back. topic={}, msgId={}, storeHost={}, queueId={}, queueOffset={}", msg.getTopic(), msg.getMsgId(), msg.getStoreHost(), msg.getQueueId(), msg.getQueueOffset());
                  try {
                      this.lockTreeMap.writeLock().lockInterruptibly();
                      try {
                          if (!msgTreeMap.isEmpty() && msg.getQueueOffset() == msgTreeMap.firstKey()) {
                              try {
                                  removeMessage(Collections.singletonList(msg));
                              } catch (Exception e) {
                                  log.error("send expired msg exception", e);
                              }
                          }
                      } finally {
                          this.lockTreeMap.writeLock().unlock();
                      }
                  } catch (InterruptedException e) {
                      log.error("getExpiredMsg exception", e);
                  }
              } catch (Exception e) {
                  log.error("send expired msg exception", e);
              }
          }
      }
  
  
      /**
       * 添加消息
       *    PullMessageService拉取消息后，先调用该方法将消息添加到ProcessQueue
       * @param msgs
       * @return
       */
      public boolean putMessage(final List<MessageExt> msgs) {
          boolean dispatchToConsume = false;
          try {
              this.lockTreeMap.writeLock().lockInterruptibly();
              try {
                  int validMsgCnt = 0;
                  for (MessageExt msg : msgs) {
                      MessageExt old = msgTreeMap.put(msg.getQueueOffset(), msg);
                      if (null == old) {
                          validMsgCnt++;
                          this.queueOffsetMax = msg.getQueueOffset();
                          msgSize.addAndGet(msg.getBody().length);
                      }
                  }
                  msgCount.addAndGet(validMsgCnt);
  
                  if (!msgTreeMap.isEmpty() && !this.consuming) {
                      dispatchToConsume = true;
                      this.consuming = true;
                  }
  
                  if (!msgs.isEmpty()) {
                      MessageExt messageExt = msgs.get(msgs.size() - 1);
                      String property = messageExt.getProperty(MessageConst.PROPERTY_MAX_OFFSET);
                      if (property != null) {
                          long accTotal = Long.parseLong(property) - messageExt.getQueueOffset();
                          if (accTotal > 0) {
                              this.msgAccCnt = accTotal;
                          }
                      }
                  }
              } finally {
                  this.lockTreeMap.writeLock().unlock();
              }
          } catch (InterruptedException e) {
              log.error("putMessage exception", e);
          }
  
          return dispatchToConsume;
      }
  
  
      /**
       * 获取当前消息最大间隔
       *      getMaxSpan()/20
       *          并不能说明ProcessQueue中包含的消息个数，
       *          但是能说明当前处理队列中第一条消息与最后一条消息的偏移量已经超过的消息个数
       * @return
       */
      public long getMaxSpan() {
          try {
              this.lockTreeMap.readLock().lockInterruptibly();
              try {
                  if (!this.msgTreeMap.isEmpty()) {
                      return this.msgTreeMap.lastKey() - this.msgTreeMap.firstKey();
                  }
              } finally {
                  this.lockTreeMap.readLock().unlock();
              }
          } catch (InterruptedException e) {
              log.error("getMaxSpan exception", e);
          }
  
          return 0;
      }
  
  
      /**
       * 移除消息
       * @param msgs
       * @return
       */
      public long removeMessage(final List<MessageExt> msgs) {
          long result = -1;
          final long now = System.currentTimeMillis();
          try {
              this.lockTreeMap.writeLock().lockInterruptibly();
              this.lastConsumeTimestamp = now;
              try {
                  if (!msgTreeMap.isEmpty()) {
                      result = this.queueOffsetMax + 1;
                      int removedCnt = 0;
                      for (MessageExt msg : msgs) {
                          MessageExt prev = msgTreeMap.remove(msg.getQueueOffset());
                          if (prev != null) {
                              removedCnt--;
                              msgSize.addAndGet(0 - msg.getBody().length);
                          }
                      }
                      msgCount.addAndGet(removedCnt);
  
                      if (!msgTreeMap.isEmpty()) {
                          result = msgTreeMap.firstKey();
                      }
                  }
              } finally {
                  this.lockTreeMap.writeLock().unlock();
              }
          } catch (Throwable t) {
              log.error("removeMessage exception", t);
          }
  
          return result;
      }
  
      public TreeMap<Long, MessageExt> getMsgTreeMap() {
          return msgTreeMap;
      }
  
      public AtomicLong getMsgCount() {
          return msgCount;
      }
  
      public AtomicLong getMsgSize() {
          return msgSize;
      }
  
      public boolean isDropped() {
          return dropped;
      }
  
      public void setDropped(boolean dropped) {
          this.dropped = dropped;
      }
  
      public boolean isLocked() {
          return locked;
      }
  
      public void setLocked(boolean locked) {
          this.locked = locked;
      }
  
  
      /**
       * 将 consumingMsgOrderlyTreeMap 中所有消息重新放入到 msgTreeMap 并清除 consumingMsgOrderlyTreeMap
       */
      public void rollback() {
          try {
              this.lockTreeMap.writeLock().lockInterruptibly();
              try {
                  this.msgTreeMap.putAll(this.consumingMsgOrderlyTreeMap);
                  this.consumingMsgOrderlyTreeMap.clear();
              } finally {
                  this.lockTreeMap.writeLock().unlock();
              }
          } catch (InterruptedException e) {
              log.error("rollback exception", e);
          }
      }
  
  
      /**
       * 将consumingMsgOrderlyTreeMap 中的消息清除，表示成功处理该批消息
       * @return
       */
      public long commit() {
          try {
              this.lockTreeMap.writeLock().lockInterruptibly();
              try {
                  Long offset = this.consumingMsgOrderlyTreeMap.lastKey();
                  msgCount.addAndGet(0 - this.consumingMsgOrderlyTreeMap.size());
                  for (MessageExt msg : this.consumingMsgOrderlyTreeMap.values()) {
                      msgSize.addAndGet(0 - msg.getBody().length);
                  }
                  this.consumingMsgOrderlyTreeMap.clear();
                  if (offset != null) {
                      return offset + 1;
                  }
              } finally {
                  this.lockTreeMap.writeLock().unlock();
              }
          } catch (InterruptedException e) {
              log.error("commit exception", e);
          }
  
          return -1;
      }
  
  
      /**
       * 重新消费该批消息
       * @param msgs
       */
      public void makeMessageToCosumeAgain(List<MessageExt> msgs) {
          try {
              this.lockTreeMap.writeLock().lockInterruptibly();
              try {
                  for (MessageExt msg : msgs) {
                      this.consumingMsgOrderlyTreeMap.remove(msg.getQueueOffset());
                      this.msgTreeMap.put(msg.getQueueOffset(), msg);
                  }
              } finally {
                  this.lockTreeMap.writeLock().unlock();
              }
          } catch (InterruptedException e) {
              log.error("makeMessageToCosumeAgain exception", e);
          }
      }
  
  
      /**
       * 从ProcessQueue中取出batchSize条消息
       * @param batchSize
       * @return
       */
      public List<MessageExt> takeMessags(final int batchSize) {
          List<MessageExt> result = new ArrayList<MessageExt>(batchSize);
          final long now = System.currentTimeMillis();
          try {
              this.lockTreeMap.writeLock().lockInterruptibly();
              this.lastConsumeTimestamp = now;
              try {
                  if (!this.msgTreeMap.isEmpty()) {
                      for (int i = 0; i < batchSize; i++) {
                          Map.Entry<Long, MessageExt> entry = this.msgTreeMap.pollFirstEntry();
                          if (entry != null) {
                              result.add(entry.getValue());
                              consumingMsgOrderlyTreeMap.put(entry.getKey(), entry.getValue());
                          } else {
                              break;
                          }
                      }
                  }
  
                  if (result.isEmpty()) {
                      consuming = false;
                  }
              } finally {
                  this.lockTreeMap.writeLock().unlock();
              }
          } catch (InterruptedException e) {
              log.error("take Messages exception", e);
          }
  
          return result;
      }
  
      public boolean hasTempMessage() {
          try {
              this.lockTreeMap.readLock().lockInterruptibly();
              try {
                  return !this.msgTreeMap.isEmpty();
              } finally {
                  this.lockTreeMap.readLock().unlock();
              }
          } catch (InterruptedException e) {
          }
  
          return true;
      }
  
      public void clear() {
          try {
              this.lockTreeMap.writeLock().lockInterruptibly();
              try {
                  this.msgTreeMap.clear();
                  this.consumingMsgOrderlyTreeMap.clear();
                  this.msgCount.set(0);
                  this.msgSize.set(0);
                  this.queueOffsetMax = 0L;
              } finally {
                  this.lockTreeMap.writeLock().unlock();
              }
          } catch (InterruptedException e) {
              log.error("rollback exception", e);
          }
      }
  
      public long getLastLockTimestamp() {
          return lastLockTimestamp;
      }
  
      public void setLastLockTimestamp(long lastLockTimestamp) {
          this.lastLockTimestamp = lastLockTimestamp;
      }
  
      public Lock getLockConsume() {
          return lockConsume;
      }
  
      public long getLastPullTimestamp() {
          return lastPullTimestamp;
      }
  
      public void setLastPullTimestamp(long lastPullTimestamp) {
          this.lastPullTimestamp = lastPullTimestamp;
      }
  
      public long getMsgAccCnt() {
          return msgAccCnt;
      }
  
      public void setMsgAccCnt(long msgAccCnt) {
          this.msgAccCnt = msgAccCnt;
      }
  
      public long getTryUnlockTimes() {
          return this.tryUnlockTimes.get();
      }
  
      public void incTryUnlockTimes() {
          this.tryUnlockTimes.incrementAndGet();
      }
  
      public void fillProcessQueueInfo(final ProcessQueueInfo info) {
          try {
              this.lockTreeMap.readLock().lockInterruptibly();
  
              if (!this.msgTreeMap.isEmpty()) {
                  info.setCachedMsgMinOffset(this.msgTreeMap.firstKey());
                  info.setCachedMsgMaxOffset(this.msgTreeMap.lastKey());
                  info.setCachedMsgCount(this.msgTreeMap.size());
                  info.setCachedMsgSizeInMiB((int) (this.msgSize.get() / (1024 * 1024)));
              }
  
              if (!this.consumingMsgOrderlyTreeMap.isEmpty()) {
                  info.setTransactionMsgMinOffset(this.consumingMsgOrderlyTreeMap.firstKey());
                  info.setTransactionMsgMaxOffset(this.consumingMsgOrderlyTreeMap.lastKey());
                  info.setTransactionMsgCount(this.consumingMsgOrderlyTreeMap.size());
              }
  
              info.setLocked(this.locked);
              info.setTryUnlockTimes(this.tryUnlockTimes.get());
              info.setLastLockTimestamp(this.lastLockTimestamp);
  
              info.setDroped(this.dropped);
              info.setLastPullTimestamp(this.lastPullTimestamp);
              info.setLastConsumeTimestamp(this.lastConsumeTimestamp);
          } catch (Exception e) {
          } finally {
              this.lockTreeMap.readLock().unlock();
          }
      }
  
      public long getLastConsumeTimestamp() {
          return lastConsumeTimestamp;
      }
  
      public void setLastConsumeTimestamp(long lastConsumeTimestamp) {
          this.lastConsumeTimestamp = lastConsumeTimestamp;
      }
  
  }
  ```





### 消息拉取基本流程--并发消息

* 消息拉取分为3个阶段
  * 消息拉取客户端  消息拉取请求 封装
  * 消息服务器查找并返回消息
  * 消息拉取客户端 处理 返回的消息

* 消息拉取入口

  * DefaultMQPushConsumerImpl#pullMessage

* 流程图

  <img src="../../../../照片/typora/image-20210816163448951.png" alt="image-20210816163448951" style="zoom:50%;" />

#### 客户端封装消息拉取请求

* 涉及的亮点

  * 流控
  * 延迟处理

* 重要对象

  * 拉取Broker的信息--》FindBrokerResult

    ```java
    public class FindBrokerResult {
        //Broker地址
        private final String brokerAddr;
        //是否是从节点
        private final boolean slave;
        //Broker版本
        private final int brokerVersion;
      
        //省略普通的方法
      
    }
    ```

##### DefaultMQPushConsumerImpl

* pullMessage

  ```java
  public void pullMessage(final PullRequest pullRequest) {
  
    //1. 从PullRequest中获取ProcessQueue
    final ProcessQueue processQueue = pullRequest.getProcessQueue();
    //1.1 如果处理队列当前状态未被丢弃，则更新ProcessQueue的 lastPullTimestamp
    if (processQueue.isDropped()) {
      log.info("the pull request[{}] is dropped.", pullRequest.toString());
      return;
    }
  
    pullRequest.getProcessQueue().setLastPullTimestamp(System.currentTimeMillis());
  
    try {
      this.makeSureStateOK();
    } catch (MQClientException e) {
      log.warn("pullMessage exception, consumer state not ok", e);
      this.executePullRequestLater(pullRequest, pullTimeDelayMillsWhenException);
      return;
    }
  
    //2. 如果当前消费者被挂起，则将拉取任务延迟1s再次放入到PullMessageService的拉取任务队列中，结束本地消息拉取
    if (this.isPause()) {
      log.warn("consumer was paused, execute pull request later. instanceName={}, group={}", this.defaultMQPushConsumer.getInstanceName(), this.defaultMQPushConsumer.getConsumerGroup());
  
      //延迟消息，底层使用线程池执行  ScheduledExecutorService
      this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_SUSPEND);
      return;
    }
  
  
    //消息缓存个数
    long cachedMessageCount = processQueue.getMsgCount().get();
    //消息缓存的总大小
    long cachedMessageSizeInMiB = processQueue.getMsgSize().get() / (1024 * 1024);
  
  
    //3. 进行消息拉取流控，从消息消费数量和消息间隔两个维度进行控制
    //3.1 消息数量流控
    //   解释
    //       如果ProcessQueue 当前处理的消息条数超过了 pu!IThresholdForQueue=lOOO 将触发流控，放弃本次拉取任务，并且该队列的下 次拉取任务将在 50 毫秒后
    //       才加入到拉取任务队列中， 触发 1000 次流控后输出提示语： the consumer message buffer is full, so do flow control,
    //       minOffset= ｛队列最小偏移 ｝， maxOffset= ｛队列最大偏移 ｝， size＝｛消息总条数｝, pullRequest= ｛拉取任务｝, flowControlTimes= ｛流控触发次数｝
    if (cachedMessageCount > this.defaultMQPushConsumer.getPullThresholdForQueue()) {
      this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_FLOW_CONTROL);
      if ((queueFlowControlTimes++ % 1000) == 0) {
        log.warn(
          "the cached message count exceeds the threshold {}, so do flow control, minOffset={}, maxOffset={}, count={}, size={} MiB, pullRequest={}, flowControlTimes={}",
          this.defaultMQPushConsumer.getPullThresholdForQueue(), processQueue.getMsgTreeMap().firstKey(), processQueue.getMsgTreeMap().lastKey(), cachedMessageCount, cachedMessageSizeInMiB, pullRequest, queueFlowControlTimes);
      }
      return;
    }
  
    //3.2 消息间隔流控
    //    解释
    //       ProcessQueue 中队列最大偏移量与最小偏离量的间距， 不能超 consumeConcurrentlyMaxSpan ，否则触发流控，
    //       每触发 1000 次输出提示语： the queue's messages, span too long, so do
    //       flow control, minOffset= ｛队列 小偏移量｝， maxOffs t= ｛队列最大偏移量｝， maxSpan＝｛间隔｝pullRequest= ｛拉取任务信息｝， flowControlTimes＝｛流控触发次数｝
    //       这里主要的考量是担心条消息堵塞，消息进度无法向前推进，可能造成大量消息重复消费
    if (cachedMessageSizeInMiB > this.defaultMQPushConsumer.getPullThresholdSizeForQueue()) {
      this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_FLOW_CONTROL);
      if ((queueFlowControlTimes++ % 1000) == 0) {
        log.warn(
          "the cached message size exceeds the threshold {} MiB, so do flow control, minOffset={}, maxOffset={}, count={}, size={} MiB, pullRequest={}, flowControlTimes={}",
          this.defaultMQPushConsumer.getPullThresholdSizeForQueue(), processQueue.getMsgTreeMap().firstKey(), processQueue.getMsgTreeMap().lastKey(), cachedMessageCount, cachedMessageSizeInMiB, pullRequest, queueFlowControlTimes);
      }
      return;
    }
  
    if (!this.consumeOrderly) {
      if (processQueue.getMaxSpan() > this.defaultMQPushConsumer.getConsumeConcurrentlyMaxSpan()) {
        this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_FLOW_CONTROL);
        if ((queueMaxSpanFlowControlTimes++ % 1000) == 0) {
          log.warn(
            "the queue's messages, span too long, so do flow control, minOffset={}, maxOffset={}, maxSpan={}, pullRequest={}, flowControlTimes={}",
            processQueue.getMsgTreeMap().firstKey(), processQueue.getMsgTreeMap().lastKey(), processQueue.getMaxSpan(),
            pullRequest, queueMaxSpanFlowControlTimes);
        }
        return;
      }
    } else {
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
      } else {
        this.executePullRequestLater(pullRequest, pullTimeDelayMillsWhenException);
        log.info("pull message later because not locked in broker, {}", pullRequest);
        return;
      }
    }
  
  
    //4. 拉取该主题订阅消息
    final SubscriptionData subscriptionData = this.rebalanceImpl.getSubscriptionInner().get(pullRequest.getMessageQueue().getTopic());
    //4.1 如果为空，结束本次消息拉取，
    if (null == subscriptionData) {
      // 延迟3s再加入拉取队列中
      this.executePullRequestLater(pullRequest, pullTimeDelayMillsWhenException);
      log.warn("find the consumer's subscription failed, {}", pullRequest);
      return;
    }
  
    final long beginTimestamp = System.currentTimeMillis();
  
  
    //构建拉取之后的回调函数
    PullCallback pullCallback = new PullCallback() {
      @Override
      public void onSuccess(PullResult pullResult) {
        if (pullResult != null) {
          pullResult = DefaultMQPushConsumerImpl.this.pullAPIWrapper.processPullResult(pullRequest.getMessageQueue(), pullResult,
                                                                                       subscriptionData);
  
          switch (pullResult.getPullStatus()) {
            case FOUND:
              long prevRequestOffset = pullRequest.getNextOffset();
              pullRequest.setNextOffset(pullResult.getNextBeginOffset());
              long pullRT = System.currentTimeMillis() - beginTimestamp;
              DefaultMQPushConsumerImpl.this.getConsumerStatsManager().incPullRT(pullRequest.getConsumerGroup(),
                                                                                 pullRequest.getMessageQueue().getTopic(), pullRT);
  
              long firstMsgOffset = Long.MAX_VALUE;
              if (pullResult.getMsgFoundList() == null || pullResult.getMsgFoundList().isEmpty()) {
                DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
              } else {
                firstMsgOffset = pullResult.getMsgFoundList().get(0).getQueueOffset();
  
                DefaultMQPushConsumerImpl.this.getConsumerStatsManager().incPullTPS(pullRequest.getConsumerGroup(),
                                                                                    pullRequest.getMessageQueue().getTopic(), pullResult.getMsgFoundList().size());
  
                boolean dispatchToConsume = processQueue.putMessage(pullResult.getMsgFoundList());
                DefaultMQPushConsumerImpl.this.consumeMessageService.submitConsumeRequest(
                  pullResult.getMsgFoundList(),
                  processQueue,
                  pullRequest.getMessageQueue(),
                  dispatchToConsume);
  
                if (DefaultMQPushConsumerImpl.this.defaultMQPushConsumer.getPullInterval() > 0) {
                  DefaultMQPushConsumerImpl.this.executePullRequestLater(pullRequest,
                                                                         DefaultMQPushConsumerImpl.this.defaultMQPushConsumer.getPullInterval());
                } else {
                  DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
                }
              }
  
              if (pullResult.getNextBeginOffset() < prevRequestOffset
                  || firstMsgOffset < prevRequestOffset) {
                log.warn(
                  "[BUG] pull message result maybe data wrong, nextBeginOffset: {} firstMsgOffset: {} prevRequestOffset: {}",
                  pullResult.getNextBeginOffset(),
                  firstMsgOffset,
                  prevRequestOffset);
              }
  
              break;
            case NO_NEW_MSG:
              pullRequest.setNextOffset(pullResult.getNextBeginOffset());
  
              DefaultMQPushConsumerImpl.this.correctTagsOffset(pullRequest);
  
              DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
              break;
            case NO_MATCHED_MSG:
              pullRequest.setNextOffset(pullResult.getNextBeginOffset());
  
              DefaultMQPushConsumerImpl.this.correctTagsOffset(pullRequest);
  
              DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
              break;
            case OFFSET_ILLEGAL:
              log.warn("the pull request offset illegal, {} {}",
                       pullRequest.toString(), pullResult.toString());
              pullRequest.setNextOffset(pullResult.getNextBeginOffset());
  
              pullRequest.getProcessQueue().setDropped(true);
              DefaultMQPushConsumerImpl.this.executeTaskLater(new Runnable() {
  
                @Override
                public void run() {
                  try {
                    DefaultMQPushConsumerImpl.this.offsetStore.updateOffset(pullRequest.getMessageQueue(),
                                                                            pullRequest.getNextOffset(), false);
  
                    DefaultMQPushConsumerImpl.this.offsetStore.persist(pullRequest.getMessageQueue());
  
                    DefaultMQPushConsumerImpl.this.rebalanceImpl.removeProcessQueue(pullRequest.getMessageQueue());
  
                    log.warn("fix the pull request offset, {}", pullRequest);
                  } catch (Throwable e) {
                    log.error("executeTaskLater Exception", e);
                  }
                }
              }, 10000);
              break;
            default:
              break;
          }
        }
      }
  
      @Override
      public void onException(Throwable e) {
        if (!pullRequest.getMessageQueue().getTopic().startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
          log.warn("execute the pull request exception", e);
        }
  
        DefaultMQPushConsumerImpl.this.executePullRequestLater(pullRequest, pullTimeDelayMillsWhenException);
      }
    };
  
  
    //5. 构建消息拉取系统标记
    boolean commitOffsetEnable = false;
    long commitOffsetValue = 0L;
    if (MessageModel.CLUSTERING == this.defaultMQPushConsumer.getMessageModel()) {
      commitOffsetValue = this.offsetStore.readOffset(pullRequest.getMessageQueue(), ReadOffsetType.READ_FROM_MEMORY);
      if (commitOffsetValue > 0) {
        commitOffsetEnable = true;
      }
    }
  
    String subExpression = null;
    boolean classFilter = false;
    SubscriptionData sd = this.rebalanceImpl.getSubscriptionInner().get(pullRequest.getMessageQueue().getTopic());
    if (sd != null) {
      if (this.defaultMQPushConsumer.isPostSubscriptionWhenPull() && !sd.isClassFilterMode()) {
        subExpression = sd.getSubString();
      }
  
      classFilter = sd.isClassFilterMode();
    }
  
    //拉取系统标记
    int sysFlag = PullSysFlag.buildSysFlag(
      commitOffsetEnable, // commitOffset
      true, // suspend
      subExpression != null, // subscription
      classFilter // class filter
    );
  
  
    //6. 真正的拉取消息
    try {
      this.pullAPIWrapper.pullKernelImpl(
        pullRequest.getMessageQueue(),
        subExpression,
        subscriptionData.getExpressionType(),
        subscriptionData.getSubVersion(),
        pullRequest.getNextOffset(),
        this.defaultMQPushConsumer.getPullBatchSize(),
        sysFlag,
        commitOffsetValue,
        BROKER_SUSPEND_MAX_TIME_MILLIS,
        CONSUMER_TIMEOUT_MILLIS_WHEN_SUSPEND,
        CommunicationMode.ASYNC,
        pullCallback
      );
    } catch (Exception e) {
      log.error("pullKernelImpl exception", e);
      this.executePullRequestLater(pullRequest, pullTimeDelayMillsWhenException);
    }
  }
  ```

  * 延迟处理

    * 就是简单使用了一个定时的线程去执行，是不是很简单

      ```java
      public class PullMessageService extends ServiceThread {
          private final InternalLogger log = ClientLogger.getLog();
          private final LinkedBlockingQueue<PullRequest> pullRequestQueue = new LinkedBlockingQueue<PullRequest>();
          private final MQClientInstance mQClientFactory;
          private final ScheduledExecutorService scheduledExecutorService = Executors
              .newSingleThreadScheduledExecutor(new ThreadFactory() {
                  @Override
                  public Thread newThread(Runnable r) {
                      return new Thread(r, "PullMessageServiceScheduledThread");
                  }
              });
      
          public PullMessageService(MQClientInstance mQClientFactory) {
              this.mQClientFactory = mQClientFactory;
          }
      
          public void executePullRequestLater(final PullRequest pullRequest, final long timeDelay) {
              if (!isStopped()) {
                  this.scheduledExecutorService.schedule(new Runnable() {
                      @Override
                      public void run() {
                          PullMessageService.this.executePullRequestImmediately(pullRequest);
                      }
                  }, timeDelay, TimeUnit.MILLISECONDS);
              } else {
                  log.warn("PullMessageServiceScheduledThread has shutdown");
              }
          }
       
          //....省去了其他方法....
      }
      ```

  ##### PullAPIWrapper

  ###### pullKernelImpl

  ```java
  public PullResult pullKernelImpl(
    final MessageQueue mq, //从哪个消息队列拉取消息
    final String subExpression,//消息过滤表达式
    final String expressionType,//消息表达式类型，分为 TAG SQL92
    final long subVersion,//
    final long offset,//消息拉取偏移量
    final int maxNums,//本次拉取最大消息条数，默认 32条
    final int sysFlag,//拉取系统标记
    final long commitOffset,//当前MessageQueue 的消费进度（内存中）
    final long brokerSuspendMaxTimeMillis,//消息拉取过程中允许 Broker 挂起时间，默认15s
    final long timeoutMillis,//消息拉取超时时间
    final CommunicationMode communicationMode,//消息拉取模式，默认为异步拉取
    final PullCallback pullCallback//从Broker拉取到消息后的回调方法
  ) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
  
    //1. 根据brokerName、BrokerId从MQClientInstance中获取Broker地址
    //   在整个RocketMQ Broker的部署结构中，相同名称的Broker构成主从结构，其BrokerId会不一样
    //   在每次拉取消息后，会给出一个建议，下次拉取从主节点还是从节点拉取
    FindBrokerResult findBrokerResult =
      this.mQClientFactory.findBrokerAddressInSubscribe(mq.getBrokerName(),
                                                        this.recalculatePullFromWhichNode(mq), false);
    if (null == findBrokerResult) {
      this.mQClientFactory.updateTopicRouteInfoFromNameServer(mq.getTopic());
      findBrokerResult =
        this.mQClientFactory.findBrokerAddressInSubscribe(mq.getBrokerName(),
                                                          this.recalculatePullFromWhichNode(mq), false);
    }
  
    if (findBrokerResult != null) {
      {
        // check version
        if (!ExpressionType.isTagType(expressionType)
            && findBrokerResult.getBrokerVersion() < MQVersion.Version.V4_1_0_SNAPSHOT.ordinal()) {
          throw new MQClientException("The broker[" + mq.getBrokerName() + ", "
                                      + findBrokerResult.getBrokerVersion() + "] does not upgrade to support for filter message by " + expressionType, null);
        }
      }
      int sysFlagInner = sysFlag;
  
      if (findBrokerResult.isSlave()) {
        sysFlagInner = PullSysFlag.clearCommitOffsetFlag(sysFlagInner);
      }
  
      //2. 构建消息请求体
      PullMessageRequestHeader requestHeader = new PullMessageRequestHeader();
      requestHeader.setConsumerGroup(this.consumerGroup);
      requestHeader.setTopic(mq.getTopic());
      requestHeader.setQueueId(mq.getQueueId());
      requestHeader.setQueueOffset(offset);
      requestHeader.setMaxMsgNums(maxNums);
      requestHeader.setSysFlag(sysFlagInner);
      requestHeader.setCommitOffset(commitOffset);
      requestHeader.setSuspendTimeoutMillis(brokerSuspendMaxTimeMillis);
      requestHeader.setSubscription(subExpression);
      requestHeader.setSubVersion(subVersion);
      requestHeader.setExpressionType(expressionType);
  
      //如果消息过滤模式为类过滤， 需要根据主题名称、 broker 地址找到注册在
      //Broke 上的 FilterServer 地址，从 FilterServer 上拉取消息，否则从 Broker 上拉取消息
      String brokerAddr = findBrokerResult.getBrokerAddr();
      if (PullSysFlag.hasClassFilterFlag(sysFlagInner)) {
        brokerAddr = computPullFromWhichFilterServer(mq.getTopic(), brokerAddr);
      }
  
      //3. 异步拉取消息。MQClientInstance为通讯的单例对象，底层真正通讯的是MQClientApiImpl
      PullResult pullResult = this.mQClientFactory.getMQClientAPIImpl().pullMessage(
        brokerAddr,
        requestHeader,
        timeoutMillis,
        communicationMode,
        pullCallback);
  
      return pullResult;
    }
  
    throw new MQClientException("The broker[" + mq.getBrokerName() + "] not exist", null);
  }
  ```

  

##### MQClientAPIImpl

###### pullMessage

```java
public PullResult pullMessage(
  final String addr,
  final PullMessageRequestHeader requestHeader,
  final long timeoutMillis,
  final CommunicationMode communicationMode,
  final PullCallback pullCallback
) throws RemotingException, MQBrokerException, InterruptedException {
  RemotingCommand request = RemotingCommand.createRequestCommand(RequestCode.PULL_MESSAGE, requestHeader);

  switch (communicationMode) {
    case ONEWAY:
      assert false;
      return null;
    case ASYNC:
      this.pullMessageAsync(addr, request, timeoutMillis, pullCallback);
      return null;
    case SYNC:
      return this.pullMessageSync(addr, request, timeoutMillis);
    default:
      assert false;
      break;
  }

  return null;
}
```



###### pullMessageAsync

```java
private void pullMessageAsync(
  final String addr,
  final RemotingCommand request,
  final long timeoutMillis,
  final PullCallback pullCallback
) throws RemotingException, InterruptedException {
  
  //底层使用的是Netty
  this.remotingClient.invokeAsync(addr, request, timeoutMillis, new InvokeCallback() {
    @Override
    public void operationComplete(ResponseFuture responseFuture) {
      RemotingCommand response = responseFuture.getResponseCommand();
      if (response != null) {
        try {
          PullResult pullResult = MQClientAPIImpl.this.processPullResponse(response);
          assert pullResult != null;
          pullCallback.onSuccess(pullResult);
        } catch (Exception e) {
          pullCallback.onException(e);
        }
      } else {
        if (!responseFuture.isSendRequestOK()) {
          pullCallback.onException(new MQClientException("send request failed to " + addr + ". Request: " + request, responseFuture.getCause()));
        } else if (responseFuture.isTimeout()) {
          pullCallback.onException(new MQClientException("wait response from " + addr + " timeout :" + responseFuture.getTimeoutMillis() + "ms" + ". Request: " + request,
                                                         responseFuture.getCause()));
        } else {
          pullCallback.onException(new MQClientException("unknown reason. addr: " + addr + ", timeoutMillis: " + timeoutMillis + ". Request: " + request, responseFuture.getCause()));
        }
      }
    }
  });
}
```

* 最底层代码，netty

  ```java
  public void invokeAsyncImpl(final Channel channel, final RemotingCommand request, final long timeoutMillis,
                              final InvokeCallback invokeCallback)
    throws InterruptedException, RemotingTooMuchRequestException, RemotingTimeoutException, RemotingSendRequestException {
    long beginStartTime = System.currentTimeMillis();
    final int opaque = request.getOpaque();
  
    //控制最大的并发数，不能无限制的占有内存
    boolean acquired = this.semaphoreAsync.tryAcquire(timeoutMillis, TimeUnit.MILLISECONDS);
  
    if (acquired) {
      final SemaphoreReleaseOnlyOnce once = new SemaphoreReleaseOnlyOnce(this.semaphoreAsync);
      long costTime = System.currentTimeMillis() - beginStartTime;
      if (timeoutMillis < costTime) {
        once.release();
        throw new RemotingTimeoutException("invokeAsyncImpl call timeout");
      }
  
      final ResponseFuture responseFuture = new ResponseFuture(channel, opaque, timeoutMillis - costTime, invokeCallback, once);
      this.responseTable.put(opaque, responseFuture);
      try {
        //是不是很熟悉
        channel.writeAndFlush(request).addListener(new ChannelFutureListener() {
          @Override
          public void operationComplete(ChannelFuture f) throws Exception {
            if (f.isSuccess()) {
              responseFuture.setSendRequestOK(true);
              return;
            }
            requestFail(opaque);
            log.warn("send a request command to channel <{}> failed.", RemotingHelper.parseChannelRemoteAddr(channel));
          }
        });
      } catch (Exception e) {
        responseFuture.release();
        log.warn("send a request command to channel <" + RemotingHelper.parseChannelRemoteAddr(channel) + "> Exception", e);
        throw new RemotingSendRequestException(RemotingHelper.parseChannelRemoteAddr(channel), e);
      }
    } else {
      if (timeoutMillis <= 0) {
        throw new RemotingTooMuchRequestException("invokeAsyncImpl invoke too fast");
      } else {
        String info =
          String.format("invokeAsyncImpl tryAcquire semaphore timeout, %dms, waiting thread nums: %d semaphoreAsyncValue: %d",
                        timeoutMillis,
                        this.semaphoreAsync.getQueueLength(),
                        this.semaphoreAsync.availablePermits()
                       );
        log.warn(info);
        throw new RemotingTimeoutException(info);
      }
    }
  }
  ```

  

#### 消息服务端Broker处理拉取请求，组装消息

* 客户端发送拉取消息请求后，Broker组装消息，返回给客户端
  * 记录消息偏移量等信息
* 入口
  * PullMessageProcessor#processRequest

##### PullMessageProcessor

###### processRequest-->只包含重点部分

```java
private RemotingCommand processRequest(final Channel channel, RemotingCommand request, boolean brokerAllowSuspend)
  throws RemotingCommandException {
  RemotingCommand response = RemotingCommand.createResponseCommand(PullMessageResponseHeader.class);
  final PullMessageResponseHeader responseHeader = (PullMessageResponseHeader) response.readCustomHeader();
  final PullMessageRequestHeader requestHeader =
    (PullMessageRequestHeader) request.decodeCommandCustomHeader(PullMessageRequestHeader.class);

  response.setOpaque(request.getOpaque());

  log.debug("receive PullMessage request command, {}", request);



  SubscriptionGroupConfig subscriptionGroupConfig =
    this.brokerController.getSubscriptionGroupManager().findSubscriptionGroupConfig(requestHeader.getConsumerGroup());




  //1. 根据订阅消息，构建消息过滤器
  MessageFilter messageFilter;
  if (this.brokerController.getBrokerConfig().isFilterSupportRetry()) {
    messageFilter = new ExpressionForRetryMessageFilter(subscriptionData, consumerFilterData,
                                                        this.brokerController.getConsumerFilterManager());
  } else {
    messageFilter = new ExpressionMessageFilter(subscriptionData, consumerFilterData,
                                                this.brokerController.getConsumerFilterManager());
  }

  //2. 调用MessageStore.getMessage查找消息
  final GetMessageResult getMessageResult =
    this.brokerController.getMessageStore().getMessage(
    requestHeader.getConsumerGroup(),  //消费组名称
    requestHeader.getTopic(),//主题名称
    requestHeader.getQueueId(),//队列ID
    requestHeader.getQueueOffset(), //待拉取偏移量
    requestHeader.getMaxMsgNums(),//最大拉取消息条数
    messageFilter);//消息过滤器


  //3. 根据PullRequest填充responseHeader的 nextBeginOffset、minOffset、maxOffset
  if (getMessageResult != null) {
    response.setRemark(getMessageResult.getStatus().name());
    responseHeader.setNextBeginOffset(getMessageResult.getNextBeginOffset());
    responseHeader.setMinOffset(getMessageResult.getMinOffset());
    responseHeader.setMaxOffset(getMessageResult.getMaxOffset());


    //4. 如果从节点包含了下一次的拉取的偏移量，设置下次拉取任务的brokerId，优先从从节点上拉取数据
    if (getMessageResult.isSuggestPullingFromSlave()) {
      responseHeader.setSuggestWhichBrokerId(subscriptionGroupConfig.getWhichBrokerWhenConsumeSlowly());
    } else {
      responseHeader.setSuggestWhichBrokerId(MixAll.MASTER_ID);
    }


    //5. 设置转换关系，根据GetMessageResult编码转换成关系
    switch (getMessageResult.getStatus()) {
      case FOUND:
        response.setCode(ResponseCode.SUCCESS);
        break;
      case MESSAGE_WAS_REMOVING:
        response.setCode(ResponseCode.PULL_RETRY_IMMEDIATELY);
        break;
      case NO_MATCHED_LOGIC_QUEUE:
      case NO_MESSAGE_IN_QUEUE:
        if (0 != requestHeader.getQueueOffset()) {
          response.setCode(ResponseCode.PULL_OFFSET_MOVED);

          // XXX: warn and notify me
          log.info("the broker store no queue data, fix the request offset {} to {}, Topic: {} QueueId: {} Consumer Group: {}",
                   requestHeader.getQueueOffset(),
                   getMessageResult.getNextBeginOffset(),
                   requestHeader.getTopic(),
                   requestHeader.getQueueId(),
                   requestHeader.getConsumerGroup()
                  );
        } else {
          response.setCode(ResponseCode.PULL_NOT_FOUND);
        }
        break;
      case NO_MATCHED_MESSAGE:
        response.setCode(ResponseCode.PULL_RETRY_IMMEDIATELY);
        break;
      case OFFSET_FOUND_NULL:
        response.setCode(ResponseCode.PULL_NOT_FOUND);
        break;
      case OFFSET_OVERFLOW_BADLY:
        response.setCode(ResponseCode.PULL_OFFSET_MOVED);
        // XXX: warn and notify me
        log.info("the request offset: {} over flow badly, broker max offset: {}, consumer: {}",
                 requestHeader.getQueueOffset(), getMessageResult.getMaxOffset(), channel.remoteAddress());
        break;
      case OFFSET_OVERFLOW_ONE:
        response.setCode(ResponseCode.PULL_NOT_FOUND);
        break;
      case OFFSET_TOO_SMALL:
        response.setCode(ResponseCode.PULL_OFFSET_MOVED);
        log.info("the request offset too small. group={}, topic={}, requestOffset={}, brokerMinOffset={}, clientIp={}",
                 requestHeader.getConsumerGroup(), requestHeader.getTopic(), requestHeader.getQueueOffset(),
                 getMessageResult.getMinOffset(), channel.remoteAddress());
        break;
      default:
        assert false;
        break;
    }

  //.....省略一些方法

  boolean storeOffsetEnable = brokerAllowSuspend;
  storeOffsetEnable = storeOffsetEnable && hasCommitOffsetFlag;

  //6. 如果commitlog标记可用 ，并且 当前节点为主节点，则更新消息消费进度
  storeOffsetEnable = storeOffsetEnable
    && this.brokerController.getMessageStoreConfig().getBrokerRole() != BrokerRole.SLAVE;
  //6.1 更新消息消费进度
  if (storeOffsetEnable) {
    this.brokerController.getConsumerOffsetManager().commitOffset(RemotingHelper.parseChannelRemoteAddr(channel),
                                                                  requestHeader.getConsumerGroup(), requestHeader.getTopic(), requestHeader.getQueueId(), requestHeader.getCommitOffset());
  }
  return response;
}
```





##### DefaultMessageStore

###### getMessage

```java
public GetMessageResult getMessage(final String group, final String topic, final int queueId, final long offset,
                                   final int maxMsgNums,
                                   final MessageFilter messageFilter) {
  if (this.shutdown) {
    log.warn("message store has shutdown, so getMessage is forbidden");
    return null;
  }

  if (!this.runningFlags.isReadable()) {
    log.warn("message store is not readable, so getMessage is forbidden " + this.runningFlags.getFlagBits());
    return null;
  }

  long beginTime = this.getSystemClock().now();

  //1. 根据主题名称与队列编号获取消息消费队列
  GetMessageStatus status = GetMessageStatus.NO_MESSAGE_IN_QUEUE;
  //待查找的队列偏移量
  long nextBeginOffset = offset;
  //当前消息队列最小偏移量
  long minOffset = 0;
  //当前消息队列 大偏移量
  long maxOffset = 0;

  GetMessageResult getResult = new GetMessageResult();

  //当前commitlog文件最大偏移量
  final long maxOffsetPy = this.commitLog.getMaxOffset();

  //1.1 查找到消息队列
  ConsumeQueue consumeQueue = findConsumeQueue(topic, queueId);
  if (consumeQueue != null) {
    minOffset = consumeQueue.getMinOffsetInQueue();
    maxOffset = consumeQueue.getMaxOffsetInQueue();

    //2. 根据消息偏移量异常情况，校对下一次拉取偏移量
    //2.1 maxOffset = 表示当前消费队列中没有消息，拉取结果：NO_MESSAGE_ IN_ QUEUE
    //    如果当前 Broker 为主节点 或 offsetChecklnS!ave 为false 下次拉取偏移量依然为 offset
    //    如果当前 Broker 为从节点， offsetCheckinS!ave为true，设置下次拉取偏移为0
    if (maxOffset == 0) {
      status = GetMessageStatus.NO_MESSAGE_IN_QUEUE;
      nextBeginOffset = nextOffsetCorrection(offset, 0);

      //2.2 offset < minOffset ，表示待拉取消息偏移量小于队列的起始偏移量，拉取结果为：OFFSET TOO SMALL
      //    如果当前 Broker 为主节点或 offsetChecklnS!ave为false ，下次拉取偏移量依然为offset
      //    如果当前 Broker 为从节点 并且 offsetChecklnSlave为true ，下次拉取偏移量设置为 minOffset
    } else if (offset < minOffset) {
      status = GetMessageStatus.OFFSET_TOO_SMALL;
      nextBeginOffset = nextOffsetCorrection(offset, minOffset);

      //2.3 offset == maxOffset ，如果待拉取偏移量等于队列最大偏移量，拉取结果：OFFSET_OVERFLOW_ONE
      //   下次拉取偏移量依然为 offset
    } else if (offset == maxOffset) {
      status = GetMessageStatus.OFFSET_OVERFLOW_ONE;
      nextBeginOffset = nextOffsetCorrection(offset, offset);

      //2.4 Offset > maxOffset 表示偏移量越界，拉取结果： OFFSET_OVERFLOW_BADLY
      //    根据是否是主节点 从节点，同样校对下次拉取偏移
    } else if (offset > maxOffset) {
      status = GetMessageStatus.OFFSET_OVERFLOW_BADLY;
      if (0 == minOffset) {
        nextBeginOffset = nextOffsetCorrection(offset, minOffset);
      } else {
        nextBeginOffset = nextOffsetCorrection(offset, maxOffset);
      }
    } else {
      SelectMappedBufferResult bufferConsumeQueue = consumeQueue.getIndexBuffer(offset);
      if (bufferConsumeQueue != null) {
        try {
          status = GetMessageStatus.NO_MATCHED_MESSAGE;

          long nextPhyFileStartOffset = Long.MIN_VALUE;
          long maxPhyOffsetPulling = 0;

          int i = 0;
          final int maxFilterMessageCount = Math.max(16000, maxMsgNums * ConsumeQueue.CQ_STORE_UNIT_SIZE);
          final boolean diskFallRecorded = this.messageStoreConfig.isDiskFallRecorded();
          ConsumeQueueExt.CqExtUnit cqExtUnit = new ConsumeQueueExt.CqExtUnit();
          for (; i < bufferConsumeQueue.getSize() && i < maxFilterMessageCount; i += ConsumeQueue.CQ_STORE_UNIT_SIZE) {
            //物理偏移量
            long offsetPy = bufferConsumeQueue.getByteBuffer().getLong();
            //物理大小
            int sizePy = bufferConsumeQueue.getByteBuffer().getInt();
            long tagsCode = bufferConsumeQueue.getByteBuffer().getLong();

            maxPhyOffsetPulling = offsetPy;

            if (nextPhyFileStartOffset != Long.MIN_VALUE) {
              if (offsetPy < nextPhyFileStartOffset)
                continue;
            }

            boolean isInDisk = checkInDiskByCommitOffset(offsetPy, maxOffsetPy);

            if (this.isTheBatchFull(sizePy, maxMsgNums, getResult.getBufferTotalSize(), getResult.getMessageCount(),
                                    isInDisk)) {
              break;
            }

            boolean extRet = false, isTagsCodeLegal = true;
            if (consumeQueue.isExtAddr(tagsCode)) {
              extRet = consumeQueue.getExt(tagsCode, cqExtUnit);
              if (extRet) {
                tagsCode = cqExtUnit.getTagsCode();
              } else {
                // can't find ext content.Client will filter messages by tag also.
                log.error("[BUG] can't find consume queue extend file content!addr={}, offsetPy={}, sizePy={}, topic={}, group={}",
                          tagsCode, offsetPy, sizePy, topic, group);
                isTagsCodeLegal = false;
              }
            }

            if (messageFilter != null
                && !messageFilter.isMatchedByConsumeQueue(isTagsCodeLegal ? tagsCode : null, extRet ? cqExtUnit : null)) {
              if (getResult.getBufferTotalSize() == 0) {
                status = GetMessageStatus.NO_MATCHED_MESSAGE;
              }

              continue;
            }

            //3. 根据消息偏移量获取消息
            SelectMappedBufferResult selectResult = this.commitLog.getMessage(offsetPy, sizePy);
            if (null == selectResult) {
              if (getResult.getBufferTotalSize() == 0) {
                status = GetMessageStatus.MESSAGE_WAS_REMOVING;
              }

              nextPhyFileStartOffset = this.commitLog.rollNextFile(offsetPy);
              continue;
            }

            if (messageFilter != null
                && !messageFilter.isMatchedByCommitLog(selectResult.getByteBuffer().slice(), null)) {
              if (getResult.getBufferTotalSize() == 0) {
                status = GetMessageStatus.NO_MATCHED_MESSAGE;
              }
              // release...
              selectResult.release();
              continue;
            }

            this.storeStatsService.getGetMessageTransferedMsgCount().incrementAndGet();
            getResult.addMessage(selectResult);
            status = GetMessageStatus.FOUND;
            nextPhyFileStartOffset = Long.MIN_VALUE;
          }

          if (diskFallRecorded) {
            long fallBehind = maxOffsetPy - maxPhyOffsetPulling;
            brokerStatsManager.recordDiskFallBehindSize(group, topic, queueId, fallBehind);
          }

          nextBeginOffset = offset + (i / ConsumeQueue.CQ_STORE_UNIT_SIZE);

          long diff = maxOffsetPy - maxPhyOffsetPulling;
          long memory = (long) (StoreUtil.TOTAL_PHYSICAL_MEMORY_SIZE
                                * (this.messageStoreConfig.getAccessMessageInMemoryMaxRatio() / 100.0));
          getResult.setSuggestPullingFromSlave(diff > memory);
        } finally {

          bufferConsumeQueue.release();
        }
      } else {
        status = GetMessageStatus.OFFSET_FOUND_NULL;
        nextBeginOffset = nextOffsetCorrection(offset, consumeQueue.rollNextFile(offset));
        log.warn("consumer request topic: " + topic + "offset: " + offset + " minOffset: " + minOffset + " maxOffset: "
                 + maxOffset + ", but access logic queue failed.");
      }
    }
  } else {
    status = GetMessageStatus.NO_MATCHED_LOGIC_QUEUE;
    nextBeginOffset = nextOffsetCorrection(offset, 0);
  }

  if (GetMessageStatus.FOUND == status) {
    this.storeStatsService.getGetMessageTimesTotalFound().incrementAndGet();
  } else {
    this.storeStatsService.getGetMessageTimesTotalMiss().incrementAndGet();
  }
  long elapsedTime = this.getSystemClock().now() - beginTime;
  this.storeStatsService.setGetMessageEntireTimeMax(elapsedTime);

  //4. 设置响应回去的信息，包括下一次的偏移量信息
  getResult.setStatus(status);
  getResult.setNextBeginOffset(nextBeginOffset);
  getResult.setMaxOffset(maxOffset);
  getResult.setMinOffset(minOffset);
  return getResult;
}
```





#### 消息拉取客户端处理消息

* 在Netty请求的地方，增加了返回的回调函数，PullCallBack，那么服务器响应之后，会回调PullCallback的on Success或onException

* 执行顺序，应该是

  * MQClientAPIImpl#pullMessageAsync：发送请求的地方--》PullCallBack的回调方法

    ```java
    private void pullMessageAsync(
      final String addr,
      final RemotingCommand request,
      final long timeoutMillis,
      final PullCallback pullCallback
    ) throws RemotingException, InterruptedException {
      this.remotingClient.invokeAsync(addr, request, timeoutMillis, new InvokeCallback() {
        @Override
        public void operationComplete(ResponseFuture responseFuture) {
          RemotingCommand response = responseFuture.getResponseCommand();
          if (response != null) {
            try {
              //1. code码转换成状态码，返回的是PullResultExt
              PullResult pullResult = MQClientAPIImpl.this.processPullResponse(response);
              assert pullResult != null;
    
              //2. 执行PullCallBack的方法
              pullCallback.onSuccess(pullResult);
            } catch (Exception e) {
              pullCallback.onException(e);
            }
          } else {
            if (!responseFuture.isSendRequestOK()) {
              pullCallback.onException(new MQClientException("send request failed to " + addr + ". Request: " + request, responseFuture.getCause()));
            } else if (responseFuture.isTimeout()) {
              pullCallback.onException(new MQClientException("wait response from " + addr + " timeout :" + responseFuture.getTimeoutMillis() + "ms" + ". Request: " + request,
                                                             responseFuture.getCause()));
            } else {
              pullCallback.onException(new MQClientException("unknown reason. addr: " + addr + ", timeoutMillis: " + timeoutMillis + ". Request: " + request, responseFuture.getCause()));
            }
          }
        }
      });
    }
    ```

  * PullCallBack的方法

    ```java
    //构建拉取之后的回调函数
    PullCallback pullCallback = new PullCallback() {
      @Override
      public void onSuccess(PullResult pullResult) {
        if (pullResult != null) {
          pullResult = DefaultMQPushConsumerImpl.this.pullAPIWrapper.processPullResult(pullRequest.getMessageQueue(), pullResult,
                                                                                       subscriptionData);
    
          switch (pullResult.getPullStatus()) {
            case FOUND:
              long prevRequestOffset = pullRequest.getNextOffset();
              pullRequest.setNextOffset(pullResult.getNextBeginOffset());
              long pullRT = System.currentTimeMillis() - beginTimestamp;
              DefaultMQPushConsumerImpl.this.getConsumerStatsManager().incPullRT(pullRequest.getConsumerGroup(),
                                                                                 pullRequest.getMessageQueue().getTopic(), pullRT);
    
              long firstMsgOffset = Long.MAX_VALUE;
              if (pullResult.getMsgFoundList() == null || pullResult.getMsgFoundList().isEmpty()) {
                DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
              } else {
                firstMsgOffset = pullResult.getMsgFoundList().get(0).getQueueOffset();
    
                DefaultMQPushConsumerImpl.this.getConsumerStatsManager().incPullTPS(pullRequest.getConsumerGroup(),
                                                                                    pullRequest.getMessageQueue().getTopic(), pullResult.getMsgFoundList().size());
    
                boolean dispatchToConsume = processQueue.putMessage(pullResult.getMsgFoundList());
                DefaultMQPushConsumerImpl.this.consumeMessageService.submitConsumeRequest(
                  pullResult.getMsgFoundList(),
                  processQueue,
                  pullRequest.getMessageQueue(),
                  dispatchToConsume);
    
                if (DefaultMQPushConsumerImpl.this.defaultMQPushConsumer.getPullInterval() > 0) {
                  DefaultMQPushConsumerImpl.this.executePullRequestLater(pullRequest,
                                                                         DefaultMQPushConsumerImpl.this.defaultMQPushConsumer.getPullInterval());
                } else {
                  DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
                }
              }
    
              if (pullResult.getNextBeginOffset() < prevRequestOffset
                  || firstMsgOffset < prevRequestOffset) {
                log.warn(
                  "[BUG] pull message result maybe data wrong, nextBeginOffset: {} firstMsgOffset: {} prevRequestOffset: {}",
                  pullResult.getNextBeginOffset(),
                  firstMsgOffset,
                  prevRequestOffset);
              }
    
              break;
            case NO_NEW_MSG:
              pullRequest.setNextOffset(pullResult.getNextBeginOffset());
    
              DefaultMQPushConsumerImpl.this.correctTagsOffset(pullRequest);
    
              DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
              break;
            case NO_MATCHED_MSG:
              pullRequest.setNextOffset(pullResult.getNextBeginOffset());
    
              DefaultMQPushConsumerImpl.this.correctTagsOffset(pullRequest);
    
              DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
              break;
            case OFFSET_ILLEGAL:
              log.warn("the pull request offset illegal, {} {}",
                       pullRequest.toString(), pullResult.toString());
              pullRequest.setNextOffset(pullResult.getNextBeginOffset());
    
              pullRequest.getProcessQueue().setDropped(true);
              DefaultMQPushConsumerImpl.this.executeTaskLater(new Runnable() {
    
                @Override
                public void run() {
                  try {
                    DefaultMQPushConsumerImpl.this.offsetStore.updateOffset(pullRequest.getMessageQueue(),
                                                                            pullRequest.getNextOffset(), false);
    
                    DefaultMQPushConsumerImpl.this.offsetStore.persist(pullRequest.getMessageQueue());
    
                    DefaultMQPushConsumerImpl.this.rebalanceImpl.removeProcessQueue(pullRequest.getMessageQueue());
    
                    log.warn("fix the pull request offset, {}", pullRequest);
                  } catch (Throwable e) {
                    log.error("executeTaskLater Exception", e);
                  }
                }
              }, 10000);
              break;
            default:
              break;
          }
        }
      }
    
      @Override
      public void onException(Throwable e) {
        if (!pullRequest.getMessageQueue().getTopic().startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
          log.warn("execute the pull request exception", e);
        }
    
        DefaultMQPushConsumerImpl.this.executePullRequestLater(pullRequest, pullTimeDelayMillsWhenException);
      }
    };
    ```

    * 看看MQ的响应结构PullResult是什么样的，如何抽象的？？？

      ```java
      public class PullResult {
      
          //拉取结果
          private final PullStatus pullStatus;
      
          //下次拉取偏移量
          private final long nextBeginOffset;
      
          //消息队列最小偏移量
          private final long minOffset;
      
          //消息队列最大偏移量
          private final long maxOffset;
      
          //具体拉取消的消息列表
          private List<MessageExt> msgFoundList;
        
      }
      ```

##### DefaultMQPushConsumerImpl

###### pullMessage#PullCallback

```java
//构建拉取之后的回调函数
PullCallback pullCallback = new PullCallback() {
  @Override
  public void onSuccess(PullResult pullResult) {
    if (pullResult != null) {
      pullResult = DefaultMQPushConsumerImpl.this.pullAPIWrapper.processPullResult(pullRequest.getMessageQueue(), pullResult,
                                                                                   subscriptionData);

      switch (pullResult.getPullStatus()) {
          //-----------------找到消息-------------------
        case FOUND:
          long prevRequestOffset = pullRequest.getNextOffset();
          //1. 设置下次拉取的偏移量
          pullRequest.setNextOffset(pullResult.getNextBeginOffset());
          long pullRT = System.currentTimeMillis() - beginTimestamp;
          DefaultMQPushConsumerImpl.this.getConsumerStatsManager().incPullRT(pullRequest.getConsumerGroup(),
                                                                             pullRequest.getMessageQueue().getTopic(), pullRT);

          long firstMsgOffset = Long.MAX_VALUE;

          //1.1 如果获取的消息列表W为空，则立即将PullRequest放入到 PullMessageService 的 pullRequestQueue
          //    以便 PullMessageService 能及时唤醒并再次执行消息拉取
          //    为何会空？？因为 consumer 也会进行过滤
          if (pullResult.getMsgFoundList() == null || pullResult.getMsgFoundList().isEmpty()) {
            DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
          } else {
            firstMsgOffset = pullResult.getMsgFoundList().get(0).getQueueOffset();

            DefaultMQPushConsumerImpl.this.getConsumerStatsManager().incPullTPS(pullRequest.getConsumerGroup(),
                                                                                pullRequest.getMessageQueue().getTopic(), pullResult.getMsgFoundList().size());


            //2. 将消息存储到 ProcessQueue 中，然后将拉取到的消息提交到 ConsumeMessageService 中供消费者消费
            //   注意：该方法是一个异步方法，也就是 PullCallBack 将消息提交到ConsumeMessageService 中就会立即返回，
            //        至于这些消息如何消费， PullCallBack 不关注
            boolean dispatchToConsume = processQueue.putMessage(pullResult.getMsgFoundList());
            DefaultMQPushConsumerImpl.this.consumeMessageService.submitConsumeRequest(
              pullResult.getMsgFoundList(),
              processQueue,
              pullRequest.getMessageQueue(),
              dispatchToConsume);


            //3. 根据拉取间隔参数，如果 pullInterval > 0，则等待pullInterval毫秒后将PullRequest对象
            //   放入到 PullMessageService 的 pullRequestQueue中，该消息队列的下次拉取即将被激活，
            //   达到持续消息拉取，实现准实时拉取消息的效果
            //   总结：拉取完上一条数据，然后再去拉取下一条消息，如果有间隔，则间隔后拉取
            if (DefaultMQPushConsumerImpl.this.defaultMQPushConsumer.getPullInterval() > 0) {
              DefaultMQPushConsumerImpl.this.executePullRequestLater(pullRequest,
                                                                     DefaultMQPushConsumerImpl.this.defaultMQPushConsumer.getPullInterval());
            } else {
              DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
            }
          }

          if (pullResult.getNextBeginOffset() < prevRequestOffset
              || firstMsgOffset < prevRequestOffset) {
            log.warn(
              "[BUG] pull message result maybe data wrong, nextBeginOffset: {} firstMsgOffset: {} prevRequestOffset: {}",
              pullResult.getNextBeginOffset(),
              firstMsgOffset,
              prevRequestOffset);
          }

          break;


          //-----------------没有新消息-------------------
          //直接使用服务器端校正的偏移 进行下一次消息的拉取
        case NO_NEW_MSG:
          pullRequest.setNextOffset(pullResult.getNextBeginOffset());

          DefaultMQPushConsumerImpl.this.correctTagsOffset(pullRequest);

          DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
          break;


          //-----------------没有匹配消息-------------------
          //直接使用服务器端校正的偏移 进行下一次消息的拉取
        case NO_MATCHED_MSG:
          pullRequest.setNextOffset(pullResult.getNextBeginOffset());

          DefaultMQPushConsumerImpl.this.correctTagsOffset(pullRequest);

          DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
          break;

          //-----------------偏移量非法-------------------
        case OFFSET_ILLEGAL:
          log.warn("the pull request offset illegal, {} {}",
                   pullRequest.toString(), pullResult.toString());
          pullRequest.setNextOffset(pullResult.getNextBeginOffset());

          //1. 先将 ProcessQueue设置dropped为true，表示丢弃该消费队列，意味着 ProcessQueue 中拉取的消息将停止消费
          pullRequest.getProcessQueue().setDropped(true);
          DefaultMQPushConsumerImpl.this.executeTaskLater(new Runnable() {

            @Override
            public void run() {
              try {
                //根据服务端下一次校对的偏移量尝试更新消息消费进度（ 内存中 ），然后尝试持久化消息消费进度，
                DefaultMQPushConsumerImpl.this.offsetStore.updateOffset(pullRequest.getMessageQueue(),
                                                                        pullRequest.getNextOffset(), false);
                DefaultMQPushConsumerImpl.this.offsetStore.persist(pullRequest.getMessageQueue());

                //将该消息队列从 Rebalancelmpl 的处理队列中移除，意味着暂停该消息队列的消息拉取，
                //等待下一次消息队列 重新负载
                DefaultMQPushConsumerImpl.this.rebalanceImpl.removeProcessQueue(pullRequest.getMessageQueue());

                log.warn("fix the pull request offset, {}", pullRequest);
              } catch (Throwable e) {
                log.error("executeTaskLater Exception", e);
              }
            }
          }, 10000);
          break;
        default:
          break;
      }
    }
  }

  @Override
  public void onException(Throwable e) {
    if (!pullRequest.getMessageQueue().getTopic().startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
      log.warn("execute the pull request exception", e);
    }

    DefaultMQPushConsumerImpl.this.executePullRequestLater(pullRequest, pullTimeDelayMillsWhenException);
  }
};
```



#### 消息拉取长轮询机制

* RocketMQ 并没有真正实现推模式，而是消费者主动向消息服务器拉取消息， RocketMQ推模式是循环向消息服务端发送消息拉取请求，如果消息消费者向 RocketMQ 发送消息拉取时，消息并未到达消费队列
  * 如果不启用长轮询机制，则会在服务端等待 shortPollingTimeMills 时间后（挂起）再去判断消息是否已到达消息队列，如果消息未到达则提示消息拉取客户端 PULL_NOT_FOUND （消息不存在）
  * 如果开启长轮询模式， RocketMQ 一方面会每 5s 轮询检查一次消息是否可达，同时一有新消息到达后立马通知挂起线程再次验证新消息是否是自己感兴趣的消息，
    * 如果是则从 commitlog 文件提取消息返回给消息拉取客户端
    * 否则直到挂起超时，超时时间由消息拉取方在消息拉取时封装在请求参数中 
      * PUSH 模式默认为15s
      * PULL 模式通过 DefaultMQPullConsum r#setBrokerSuspendMaxTim Millis 设置 
* RocketMQ 通过在 Broker 端配置 longPollingEnable 为仕回来开启长轮询模式
* RocketMQ轮询机制由两个线程共同来完成
  * PullRequestHoldService：每隔5s重试一次
  * DefaultMessageStore#ReputMessageService，每处理一次重新拉取，Thread.sleep ( 1 )，继续下一次检查
    * 当先消息到达CommitLog时，ReputMessageService线程负责将消息转发给ConsumeQueue、IndexFile。如果Broker端开启了长轮询模式并且角色是主节点，则最终调用PullRequestHoldService线程的notifyMessageArriving方法唤醒挂起线程，判断当前消费队列最大偏移量是否大于待拉取偏移量，如果大于则拉取消息。长轮询模式使得消息拉取能实现准实时。



##### PullMessageProcessor

* 长轮询的入口，应该是消息拉取时服务端从commitlog未找到消息时开始

###### processRequest--部分代码

```java
//未拉取到消息
case ResponseCode.PULL_NOT_FOUND:

/*
                          Channel channel ：网络通道，通过该通道向消息拉取客户端发送响应结果
                          RemotingCommand request: 消息、拉取请求
                          boolean brokerAllowSuspend : Broker 端是否支持挂起，
                                                       处理消息拉取时默认传入 true, 表示支持如果未找到消息则挂起，
                                                       如果该参数为 false ，未找到消息时直接返回客户端消息未找到
                       */


//1. 如果 brokerAllowSuspend 为true，表示支持挂起，则将响应对象response设置为null,将不会立即向客户端写入响应
//   hasSuspendFlag参数在拉取消息时构建的拉取标记，默认为true
//   默认支持挂起，则根据是否开启长轮询来决定挂起方式，
//       如果支持长轮询模式，挂起超时时间来源于请求参数，
//              PUSH模式默认为15s
//              PULL模式通过DefaultMQPullConsumer#brokerSuspenMaxTimeMillis设置，默认20s
//                  然后创建拉取任务PullRequest并提交到PullRequestHoldService线程中
if (brokerAllowSuspend && hasSuspendFlag) {
  long pollingTimeMills = suspendTimeoutMillisLong;
  if (!this.brokerController.getBrokerConfig().isLongPollingEnable()) {
    pollingTimeMills = this.brokerController.getBrokerConfig().getShortPollingTimeMills();
  }

  String topic = requestHeader.getTopic();
  long offset = requestHeader.getQueueOffset();
  int queueId = requestHeader.getQueueId();

  PullRequest pullRequest = new PullRequest(request, channel, pollingTimeMills,
                                            this.brokerController.getMessageStore().now(), offset, subscriptionData, messageFilter);

  //暂停式拉取消息
  this.brokerController.getPullRequestHoldService().suspendPullRequest(topic, queueId, pullRequest);
  response = null;
  break;
}
```



##### PullRequestHoldService

* 类图：集成ServiceThread，可想而知是一个线程

  <img src="../../../../照片/typora/image-20210816185016320.png" alt="image-20210816185016320" style="zoom:50%;" />

###### suspendPullRequest

```java
public class PullRequestHoldService extends ServiceThread {
    private static final InternalLogger log = InternalLoggerFactory.getLogger(LoggerName.BROKER_LOGGER_NAME);
    private static final String TOPIC_QUEUEID_SEPARATOR = "@";
    private final BrokerController brokerController;
    private final SystemClock systemClock = new SystemClock();

    //维护拉取消息 key: topic@queueId   value: 拉取消息的请求集合
    private ConcurrentMap<String/* topic@queueId */, ManyPullRequest> pullRequestTable =
        new ConcurrentHashMap<String, ManyPullRequest>(1024);

    public PullRequestHoldService(final BrokerController brokerController) {
        this.brokerController = brokerController;
    }

    public void suspendPullRequest(final String topic, final int queueId, final PullRequest pullRequest) {
        //1. 根据消息主题与消息队列构建key
        String key = this.buildKey(topic, queueId);
        //底层使用synchronized来锁定队列List
        ManyPullRequest mpr = this.pullRequestTable.get(key);
        if (null == mpr) {
            mpr = new ManyPullRequest();
            ManyPullRequest prev = this.pullRequestTable.putIfAbsent(key, mpr);
            if (prev != null) {
                mpr = prev;
            }
        }

        mpr.addPullRequest(pullRequest);
    }
    //....省略部分代码....
}
```

* 由于此对象是一个线程，异步执行任务，那么必须看看run方法是如何执行的，因为在启动的时候，就意味着存在一个线程来执行此对象的run方法

###### run

```java
@Override
public void run() {
  log.info("{} service started", this.getServiceName());
  //长轮询不断的校验  pullRequestTable 是否存在需要进行拉取消息的请求
  while (!this.isStopped()) {
    try {

      //如果开启长轮询，则每5s尝试一次，判断新消息是否到达
      if (this.brokerController.getBrokerConfig().isLongPollingEnable()) {
        this.waitForRunning(5 * 1000);
      } else {
        //如果未开启长轮询，则默认等待1s再次尝试
        this.waitForRunning(this.brokerController.getBrokerConfig().getShortPollingTimeMills());
      }

      long beginLockTimestamp = this.systemClock.now();

      //核心逻辑
      this.checkHoldRequest();

      long costTime = this.systemClock.now() - beginLockTimestamp;
      if (costTime > 5 * 1000) {
        log.info("[NOTIFYME] check hold request cost {} ms.", costTime);
      }
    } catch (Throwable e) {
      log.warn(this.getServiceName() + " service has exception. ", e);
    }
  }

  log.info("{} service end", this.getServiceName());
}
```



###### checkHoldRequest

```java
private void checkHoldRequest() {
  //遍历拉取任务表
  //根据主题与队列获取消息消费队列最大偏移量，如果该偏移量大于待拉取偏移量， 说明有新的消息到达，调用 notifyMessageArriving 发消息拉取
  for (String key : this.pullRequestTable.keySet()) {
    String[] kArray = key.split(TOPIC_QUEUEID_SEPARATOR);
    if (2 == kArray.length) {
      String topic = kArray[0];
      int queueId = Integer.parseInt(kArray[1]);
      final long offset = this.brokerController.getMessageStore().getMaxOffsetInQueue(topic, queueId);
      try {
        //存在新的消息，调用此方法触发消息拉取
        this.notifyMessageArriving(topic, queueId, offset);
      } catch (Throwable e) {
        log.error("check hold request failed. topic={}, queueId={}", topic, queueId, e);
      }
    }
  }
}
```



###### notifyMessageArriving

```java
public void notifyMessageArriving(final String topic, final int queueId, final long maxOffset, final Long tagsCode,
                                  long msgStoreTime, byte[] filterBitMap, Map<String, String> properties) {
  String key = this.buildKey(topic, queueId);

  //1. 首先从 ManyPullRequest 获取当前该主题、队列所有的挂起拉取任务
  ManyPullRequest mpr = this.pullRequestTable.get(key);
  if (mpr != null) {
    List<PullRequest> requestList = mpr.cloneListAndClear();
    if (requestList != null) {
      List<PullRequest> replayList = new ArrayList<PullRequest>();

      for (PullRequest request : requestList) {
        long newestOffset = maxOffset;


        if (newestOffset <= request.getPullFromThisOffset()) {
          newestOffset = this.brokerController.getMessageStore().getMaxOffsetInQueue(topic, queueId);
        }


        //2. 如果消息队列的最大偏移量大于待拉取偏移量
        //   如果消息匹配则调用 executeRequestWhenWakeup 将消息返回给消息拉取客户端，否则等待下一次尝试
        if (newestOffset > request.getPullFromThisOffset()) {
          boolean match = request.getMessageFilter().isMatchedByConsumeQueue(tagsCode,
                                                                             new ConsumeQueueExt.CqExtUnit(tagsCode, msgStoreTime, filterBitMap));
          // match by bit map, need eval again when properties is not null.
          if (match && properties != null) {
            match = request.getMessageFilter().isMatchedByCommitLog(null, properties);
          }

          if (match) {
            try {
              this.brokerController.getPullMessageProcessor().executeRequestWhenWakeup(request.getClientChannel(),
                                                                                       request.getRequestCommand());
            } catch (Throwable e) {
              log.error("execute request when wakeup failed.", e);
            }
            continue;
          }
        }

        //3. 如果挂起超时时间超时，则不继续等待将直接返回客户消息未找到
        if (System.currentTimeMillis() >= (request.getSuspendTimestamp() + request.getTimeoutMillis())) {
          try {
            this.brokerController.getPullMessageProcessor().executeRequestWhenWakeup(request.getClientChannel(),
                                                                                     request.getRequestCommand());
          } catch (Throwable e) {
            log.error("execute request when wakeup failed.", e);
          }
          continue;
        }

        replayList.add(request);
      }

      if (!replayList.isEmpty()) {
        mpr.addPullRequest(replayList);
      }
    }
  }
}
```



######  PullMessageProcessor # executeRequestWhenWakeup

```java
public void executeRequestWhenWakeup(final Channel channel,
                                     final RemotingCommand request) throws RemotingCommandException {
  Runnable run = new Runnable() {
    @Override
    public void run() {
      try {

        //长轮询的入口代码，关键在于brokerAllowSuspend=false，表示，不支持拉取线程挂起
        //   即，当根据偏移量无法获取消息时将不挂起线程等待新消息到来
        //       而是直接返回客户端本次消息拉取未找到消息
        final RemotingCommand response = PullMessageProcessor.this.processRequest(channel, request, false);

        if (response != null) {
          response.setOpaque(request.getOpaque());
          response.markResponseType();
          try {
            channel.writeAndFlush(response).addListener(new ChannelFutureListener() {
              @Override
              public void operationComplete(ChannelFuture future) throws Exception {
                if (!future.isSuccess()) {
                  log.error("processRequestWrapper response to {} failed",
                            future.channel().remoteAddress(), future.cause());
                  log.error(request.toString());
                  log.error(response.toString());
                }
              }
            });
          } catch (Throwable e) {
            log.error("processRequestWrapper process request over, but response failed", e);
            log.error(request.toString());
            log.error(response.toString());
          }
        }
      } catch (RemotingCommandException e1) {
        log.error("excuteRequestWhenWakeup run", e1);
      }
    }
  };
  this.brokerController.getPullMessageExecutor().submit(new RequestTask(run, channel, request));
}
```

