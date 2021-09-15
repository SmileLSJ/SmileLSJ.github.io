## 解决问题

* PullMe sageService 在启动 由于 LinkedBlockingQueue PullRequest> pullRequestQueue

  没有 PullRequest 对象 ，故 PullMessageService 线程将阻塞，简单回顾一波

  ```java
  //PullMessageService#run
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

* 问题
  * PullRequest对象什么时候创建并加入到pullRequestQueue中，以便唤醒PullMessageService线程？？？
    * Rebalanceservice线程每隔20s对消费者订阅的主题进行一次队列重新分配，每一次分配都会获取主题的所有队列、从 Broker务器实时查询当前该主题 该消费组内消费者列表，对新分配的消息队列会创建对应的PullRequest对象。在一个JVM进程中，同一个消费组同一个队列只会存在一个PullReqeust对象
  * 集群内多个消费者是如何负载主题下的多个消费队列，并且如果有新的消费者加入 ，消息队列又会如何重新分步？？？
    * 由于每次进行队列重新负载时会从 Broker **实时查询**出当前消费组内所有消费者，并且对消息队列、消费者列表进行排序，这样新加入的消费者就会在队列重新分布时分配到消费队列从而消费消息
  * 负载均衡算法有哪些？



## 知识点

1. 负载均衡算法？

   1. AllocateMessageQueueAveragely ：平均分配

      1. 举例来说，如果现在有8个消息消费队列 q1 q2 q3 q4 q5 q6 q7 q8，有3个消费者c1 c2 c3 ，那么根据该负载算法 ，消息队列分配如下：

         c1: q1  q2  q3 

         c2: q4  q5  q6 

         c3: q7  q8 

   2. AllocateMessageQueueAveragelyByCircle ：平均轮询分配

      1. 举例来说，如果现在有8个消息消费队列 q1 q2 q3 q4 q5 q6 q7 q8，有3个消费者 c1 c2 

         c3 ，那么根据该负载算法，消息队列分配如下：

         c1: q1  q4  q7 

         c2: q2  q5  q8 

         c3: q3  q6 

   3. AllocateMessageQueueConsistentHash：一致性 hash。不推荐使用，因为消息队列负载信息不容易跟踪

   4. AllocateMessageQueueByConfig ：根据配置，为每一个消费者配置固定的消息队列

   5. AllocateMessageQueueByMachineRoom ：根据 Broker部署机房名，对每个消费者负责不同的 Broker 上的队列

2. RocketMQ消息拉取由PullMessageService与RebalanceService共同协作完成

   <img src="../../../../照片/typora/image-20210817141722346.png" alt="image-20210817141722346" style="zoom:50%;" />



## 消息队列负载与重新分布机制

* RocketMQ消息队列重新分布是由RebalanceService线程来实现。

  

* 一个MQClientInstance持有一个RebalanceService实现，并随着MQClientInstance的启动而启动，既然是线程，那么可以先去看看run方法

  ```java
  @Override
  public void run() {
    log.info(this.getServiceName() + " service started");
  
    //RebalanceService线程默认每个20s执行一次 mqClientFactory.doRebalance() 方法
    while (!this.isStopped()) {
      this.waitForRunning(waitInterval);
      this.mqClientFactory.doRebalance();
    }
  
    log.info(this.getServiceName() + " service end");
  }
  ```

  

  MQClientInstance#doRebalance

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
          // Start pull service
          this.pullMessageService.start();
          // 开启负载均衡的线程
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
  
  public void doRebalance() {
    //MQClientInstance遍历已注册的消费者，对消费者执行doRebalance方法
    //TODO 如何获取到所有的注册的消费者和呢？？？
    for (Map.Entry<String, MQConsumerInner> entry : this.consumerTable.entrySet()) {
      MQConsumerInner impl = entry.getValue();
      if (impl != null) {
        try {
          impl.doRebalance();
        } catch (Throwable e) {
          log.error("doRebalance exception", e);
        }
      }
    }
  }
  ```

  当前链路到达 DefaultMQPushConsumerImpl#doRebalance()方法

* DefaultMQPushConsumerImpl

  ```java
  public class DefaultMQPushConsumerImpl implements MQConsumerInner {
  	  //....省略其他代码...
      private final RebalanceImpl rebalanceImpl = new RebalancePushImpl(this);
      
      @Override
      public void doRebalance() {
          if (!this.pause) {
              this.rebalanceImpl.doRebalance(this.isConsumeOrderly());
          }
      }
    
      //....省略其他代码...
  }
  ```

  * 每个DefaultMQPushConsumerImpl都持有一个单独的RebalanceImpl对象



### RebalanceImpl

#### **RebalanceImpl#doRebalance()**

```java
public abstract class RebalanceImpl {
    protected static final InternalLogger log = ClientLogger.getLog();

    //消息队列，消费进度映射表，key为消息队列，value为此消息队列的进度
    protected final ConcurrentMap<MessageQueue, ProcessQueue> processQueueTable = new ConcurrentHashMap<MessageQueue, ProcessQueue>(64);

    //主题的队列信息表，key为主题，value为主题的队列信息
    protected final ConcurrentMap<String/* topic */, Set<MessageQueue>> topicSubscribeInfoTable =
        new ConcurrentHashMap<String, Set<MessageQueue>>();

    //主题的订购信息表，key为主题，value为对于主题的订购条件等等
    protected final ConcurrentMap<String /* topic */, SubscriptionData> subscriptionInner =
        new ConcurrentHashMap<String, SubscriptionData>();

  
}
```

```java
public void doRebalance(final boolean isOrder) {
  //1. 获取主题，以及针对主题的订购信息，包含了订购哪个主题，什么tag
  //Rebalancelmpl的Map<String,SubscriptionData> subTable 在调用消费者 DefaultMQPushConsumerlmpl#subscribe 方法时填充
  Map<String, SubscriptionData> subTable = this.getSubscriptionInner();

  if (subTable != null) {
    for (final Map.Entry<String, SubscriptionData> entry : subTable.entrySet()) {
      final String topic = entry.getKey();
      try {

        //2. RocketMQ针对单个主题进行消息队列重新负载（以集群模式）
        this.rebalanceByTopic(topic, isOrder);
      } catch (Throwable e) {
        if (!topic.startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
          log.warn("rebalanceByTopic Exception", e);
        }
      }
    }
  }

  this.truncateMessageQueueNotMyTopic();
}
```



#### **RebalanceImpl#rebalanceByTopic()**

```java
private void rebalanceByTopic(final String topic, final boolean isOrder) {
  switch (messageModel) {
    case BROADCASTING: {
      Set<MessageQueue> mqSet = this.topicSubscribeInfoTable.get(topic);
      if (mqSet != null) {
        boolean changed = this.updateProcessQueueTableInRebalance(topic, mqSet, isOrder);
        if (changed) {
          this.messageQueueChanged(topic, mqSet, mqSet);
          log.info("messageQueueChanged {} {} {} {}",
                   consumerGroup,
                   topic,
                   mqSet,
                   mqSet);
        }
      } else {
        log.warn("doRebalance, {}, but the topic[{}] not exist.", consumerGroup, topic);
      }
      break;
    }
    case CLUSTERING: {//集群模式

      //1. 从主题订阅信息缓存表中获取主题的队列信息
      //TODO: 此消息什么时候跟新？？？
      Set<MessageQueue> mqSet = this.topicSubscribeInfoTable.get(topic);

      //根据主题，以及消费者组，向Broker获取所有consumerId信息
      //问题：Broker为何会存在所有Consumer的信息？consumer启动的时候，会向Broker发送心跳包，心跳包包含这些信息
      // 类似于级联网关的节点互通情况
      List<String> cidAll = this.mQClientFactory.findConsumerIdList(topic, consumerGroup);
      if (null == mqSet) {
        if (!topic.startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
          log.warn("doRebalance, {}, but the topic[{}] not exist.", consumerGroup, topic);
        }
      }

      if (null == cidAll) {
        log.warn("doRebalance, {} {}, get consumer id list failed", consumerGroup, topic);
      }

      if (mqSet != null && cidAll != null) {//如果两者为空，则说明没有订阅过任何的消息
        List<MessageQueue> mqAll = new ArrayList<MessageQueue>();
        mqAll.addAll(mqSet);

        //2. 对 cidAll,mqAll 排序，
        //   这个很重要，同一个消费组内看到的视图保持一致，确保同一个消费队列不会被多个消费者分配
        Collections.sort(mqAll);
        Collections.sort(cidAll);

        AllocateMessageQueueStrategy strategy = this.allocateMessageQueueStrategy;

        List<MessageQueue> allocateResult = null;
        try {
          //3. 根据当前的消息队列信息情况，负载出可能属于此消费者的队列信息
          //   为何是可能？因为有可能发生了变化，但是本地还没有来得及更新
          allocateResult = strategy.allocate(
            this.consumerGroup,
            this.mQClientFactory.getClientId(),//当前消费者的consumerID
            mqAll,
            cidAll);//所有的consumerId
        } catch (Throwable e) {
          log.error("AllocateMessageQueueStrategy.allocate Exception. allocateMessageQueueStrategyName={}", strategy.getName(),
                    e);
          return;
        }

        Set<MessageQueue> allocateResultSet = new HashSet<MessageQueue>();
        if (allocateResult != null) {
          allocateResultSet.addAll(allocateResult);
        }

        boolean changed = this.updateProcessQueueTableInRebalance(topic, allocateResultSet, isOrder);
        if (changed) {
          log.info(
            "rebalanced result changed. allocateMessageQueueStrategyName={}, group={}, topic={}, clientId={}, mqAllSize={}, cidAllSize={}, rebalanceResultSize={}, rebalanceResultSet={}",
            strategy.getName(), consumerGroup, topic, this.mQClientFactory.getClientId(), mqSet.size(), cidAll.size(),
            allocateResultSet.size(), allocateResultSet);
          this.messageQueueChanged(topic, mqSet, allocateResultSet);
        }
      }
      break;
    }
    default:
      break;
  }
}
```







#### **RebalanceImpl#updateProcessQueueTableInRebalance()**

```java
/**
     *
     * @param topic 主题
     * @param mqSet 经过负载均衡之后选择的消息队列
     * @param isOrder 是否是有序
     * @return
     */
private boolean updateProcessQueueTableInRebalance(final String topic, final Set<MessageQueue> mqSet,
                                                   final boolean isOrder) {
  boolean changed = false;

  //1. 获取当前消费者，负载的消息队列
  Iterator<Entry<MessageQueue, ProcessQueue>> it = this.processQueueTable.entrySet().iterator();
  while (it.hasNext()) {
    Entry<MessageQueue, ProcessQueue> next = it.next();
    MessageQueue mq = next.getKey();
    ProcessQueue pq = next.getValue();

    if (mq.getTopic().equals(topic)) {

      //2. 之前属于当前消费者消费的消息队列，现在不属于了。
      //     说明经过本次消息队列负载后，该mq已经被分配给其他消费者了
      //     故需要暂停该消息队列消息的消费
      if (!mqSet.contains(mq)) {
        //暂停此进行的使用
        pq.setDropped(true);
        //判断是否将MessageQueue，ProcessQueue缓存表中移除当前 MessageQueue信息
        //removeUnnecessaryMessageQueue 方法主要持久化待移除 MessageQueue 消息消费进度
        if (this.removeUnnecessaryMessageQueue(mq, pq)) {
          it.remove();
          changed = true;
          log.info("doRebalance, {}, remove unnecessary mq, {}", consumerGroup, mq);
        }

        //如果上一次是超时的，那么就需要通过consumeType来判断是否删除此MessageQueue
      } else if (pq.isPullExpired()) {
        switch (this.consumeType()) {
          case CONSUME_ACTIVELY:
            break;
          case CONSUME_PASSIVELY:
            pq.setDropped(true);
            if (this.removeUnnecessaryMessageQueue(mq, pq)) {
              it.remove();
              changed = true;
              log.error("[BUG]doRebalance, {}, remove unnecessary mq, {}, because pull is pause, so try to fixed it",
                        consumerGroup, mq);
            }
            break;
          default:
            break;
        }
      }
    }
  }

  //3. 遍历本次负载分配到的队列集合
  List<PullRequest> pullRequestList = new ArrayList<PullRequest>();
  for (MessageQueue mq : mqSet) {
    //3.1 消息队列时新增加的消息队列：ProcessQueueTable中，没有包含该消息队列
    if (!this.processQueueTable.containsKey(mq)) {
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

  //4. 将 PullRequest 放入到 PullMessageService 中， 以便唤 PullMessageService 线程
  this.dispatchPullRequest(pullRequestList);

  return changed;
}
```



### RebalanceLitePullImpl

#### RebalanceLitePullImpl#computePullFromWhere		

```java
/*

        ConsumeFrom Where 相关消费 度校 策略只有 从磁盘 获取消费 度返回-1
        时才会生效，如果从消息进度存储文件中返回的消费进度小于-1，表示偏移量非
        法，则使用偏移量-1去拉取消息，那么会发生什么呢？首先第一次去消息服务器
        拉取消息时无法取到消息，但是会用-1去更新消费进度，然后将消息消费队列丢
        弃， 在下一 次消息队列负载时会再次消费

*/
@Override
public long computePullFromWhere(MessageQueue mq) {
  ConsumeFromWhere consumeFromWhere = litePullConsumerImpl.getDefaultLitePullConsumer().getConsumeFromWhere();
  long result = -1;
  switch (consumeFromWhere) {
    case CONSUME_FROM_LAST_OFFSET: {//从队列最新偏移量开始消费

      //先从内存中获取，如果没有从磁盘中获取最新的偏移量
      long lastOffset = litePullConsumerImpl.getOffsetStore().readOffset(mq, ReadOffsetType.MEMORY_FIRST_THEN_STORE);
      if (lastOffset >= 0) {
        result = lastOffset;
      } else if (-1 == lastOffset) {//CONSUME_FROM_LAST_OFFSET 模式下获取该消息队列当前最大的偏移量
        if (mq.getTopic().startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) { // First start, no offset
          result = 0L;
        } else {
          try {
            result = this.mQClientFactory.getMQAdminImpl().maxOffset(mq);
          } catch (MQClientException e) {
            result = -1;
          }
        }
      } else {// 表示该消息进度文件中存储了错误的偏移量
        result = -1;
      }
      break;
    }
    case CONSUME_FROM_FIRST_OFFSET: {//从头开始消费
      long lastOffset = litePullConsumerImpl.getOffsetStore().readOffset(mq, ReadOffsetType.MEMORY_FIRST_THEN_STORE);
      if (lastOffset >= 0) {
        result = lastOffset;
      } else if (-1 == lastOffset) {
        result = 0L;
      } else {// 表示该消息进度文件中存储了错误的偏移量
        result = -1;
      }
      break;
    }
    case CONSUME_FROM_TIMESTAMP: {//从消费者启动的时间戳对应的消费进度开始消费
      long lastOffset = litePullConsumerImpl.getOffsetStore().readOffset(mq, ReadOffsetType.MEMORY_FIRST_THEN_STORE);
      if (lastOffset >= 0) {
        result = lastOffset;
      } else if (-1 == lastOffset) {//尝试去操作消息存储时间戳为消费启动的时间戳，如果能找到则返回找到的偏移量，否则返回0
        if (mq.getTopic().startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
          try {
            result = this.mQClientFactory.getMQAdminImpl().maxOffset(mq);
          } catch (MQClientException e) {
            result = -1;
          }
        } else {
          try {
            long timestamp = UtilAll.parseDate(this.litePullConsumerImpl.getDefaultLitePullConsumer().getConsumeTimestamp(),
                                               UtilAll.YYYYMMDDHHMMSS).getTime();
            result = this.mQClientFactory.getMQAdminImpl().searchOffset(mq, timestamp);
          } catch (MQClientException e) {
            result = -1;
          }
        }
      } else {
        result = -1;
      }
      break;
    }
  }
  return result;
}
```



### RebalancePushImpl

#### RebalancePushImpl#dispatchPullRequest

```java
//又到了获取消息的地方
@Override
public void dispatchPullRequest(List<PullRequest> pullRequestList) {
  for (PullRequest pullRequest : pullRequestList) {
    this.defaultMQPushConsumerImpl.executePullRequestImmediately(pullRequest);
    log.info("doRebalance, {}, add a new pull request {}", consumerGroup, pullRequest);
  }
}
```



