## 解决问题

1. 并发消息的消费过程是如何的？？？





## 知识点

* PullMessageService 负责对消息队列进行消息拉取，从远端服务器拉取消息后将消息存入 **ProcessQueue** 消息队列处理队列中，然后调用 **ConsumeMessageService#submitConsumeRequest** 方法进行消息消费，使用线程池来消费消息，确保了消息拉取与消息消费的解耦

* 拉取消息的地方：DefaultMQPushConsuemrImpl#pullMessage()#PullCallback

  ```java 
  PullCallback pullCallback = new PullCallback() {
              @Override
              public void onSuccess(PullResult pullResult) {
                  if (pullResult != null) {
                      pullResult = DefaultMQPushConsumerImpl.this.pullAPIWrapper.processPullResult(pullRequest.getMessageQueue(), pullResult,
                          subscriptionData);
  
                      switch (pullResult.getPullStatus()) {
                          //-----------------找到消息-------------------
                          case FOUND:
                          
                           
                                 //省略其他代码
                                  
  														//消费消息的地方
                              DefaultMQPushConsumerImpl.this.consumeMessageService.submitConsumeRequest(
                                      pullResult.getMsgFoundList(),
                                      processQueue,
                                      pullRequest.getMessageQueue(),
                                      dispatchToConsume);
  
  
                         //....省略其他方法...
          };
  ```

  * 可以看出关键在于ConsumeMessageService

    

### ConsumeMessageService

* 类图

  <img src="../../../../照片/typora/image-20210817151513249.png" alt="image-20210817151513249" style="zoom:50%;" />

* 核心方法

  * consumeMessageDirectly

    ```java
    /**
    * 直接消费消息，主要用于通过管理命令收到消费消息
    * @param msg 消息
    * @param brokerName Broker名称
    * @return
    */
    @Override
    public ConsumeMessageDirectlyResult consumeMessageDirectly(MessageExt msg, String brokerName) {...}
    ```

  * submitConsumeRequest

    ```java
    /**
    * 提交消息消费
    * @param msgs 消息列表，默认一次从服务器最多拉取32条
    * @param processQueue 消息处理队列
    * @param messageQueue 消息所属消费队列
    * @param dispatchToConsume 是否转发到消费线程池，并发消费时忽略该参数
    */
    @Override
    public void submitConsumeRequest(
      final List<MessageExt> msgs,
      final ProcessQueue processQueue,
      final MessageQueue messageQueue,
      final boolean dispatchToConsume) {...}
    ```

### ConsumeMessageConcurrentlyService

* 核心属性

  ```java
  public class ConsumeMessageConcurrentlyService implements ConsumeMessageService {
      private static final InternalLogger log = ClientLogger.getLog();
  
      //消息推模式实现类
      private final DefaultMQPushConsumerImpl defaultMQPushConsumerImpl;
  
      //消费者对象
      private final DefaultMQPushConsumer defaultMQPushConsumer;
  
      //并发消息业务事件类
      private final MessageListenerConcurrently messageListener;
  
      //消息消费任务队列
      private final BlockingQueue<Runnable> consumeRequestQueue;
  
      //消息消费线程池
      private final ThreadPoolExecutor consumeExecutor;
  
      //消费组
      private final String consumerGroup;
  
      //添加消费任务到 consumeExecutor 延迟调度器
      private final ScheduledExecutorService scheduledExecutorService;
  
      //定时删除过期消息线程池
      private final ScheduledExecutorService cleanExpireMsgExecutors;
  
      public ConsumeMessageConcurrentlyService(DefaultMQPushConsumerImpl defaultMQPushConsumerImpl,
          MessageListenerConcurrently messageListener) {
          this.defaultMQPushConsumerImpl = defaultMQPushConsumerImpl;
          this.messageListener = messageListener;
  
          this.defaultMQPushConsumer = this.defaultMQPushConsumerImpl.getDefaultMQPushConsumer();
          this.consumerGroup = this.defaultMQPushConsumer.getConsumerGroup();
          this.consumeRequestQueue = new LinkedBlockingQueue<Runnable>();
  
          this.consumeExecutor = new ThreadPoolExecutor(
              this.defaultMQPushConsumer.getConsumeThreadMin(),
              this.defaultMQPushConsumer.getConsumeThreadMax(),
              1000 * 60,
              TimeUnit.MILLISECONDS,
              this.consumeRequestQueue,
              new ThreadFactoryImpl("ConsumeMessageThread_"));
  
          //核心线程为1，最大线程为Integer.MAX_VALUE，队列使用的是数组，那么必然存在一个问题，如果对象很多会发现内存溢出
          this.scheduledExecutorService = Executors.newSingleThreadScheduledExecutor(new ThreadFactoryImpl("ConsumeMessageScheduledThread_"));
          this.cleanExpireMsgExecutors = Executors.newSingleThreadScheduledExecutor(new ThreadFactoryImpl("CleanExpireMsgScheduledThread_"));
      }
  
      public void start() {
          this.cleanExpireMsgExecutors.scheduleAtFixedRate(new Runnable() {
  
              @Override
              public void run() {
                  cleanExpireMsg();
              }
  
          }, this.defaultMQPushConsumer.getConsumeTimeout(), this.defaultMQPushConsumer.getConsumeTimeout(), TimeUnit.MINUTES);
      }
  }
  ```

  

## 消息消费

* 入口
  * ConsumeMessageConcurrentlyService#submitConsumeRequest

### ConsumeMessageConcurrentlyService

#### submitConsumeRequest

```java
/**
* 提交消息消费
* @param msgs 消息列表，默认一次从服务器最多拉取32条
* @param processQueue 消息处理队列
* @param messageQueue 消息所属消费队列
* @param dispatchToConsume 是否转发到消费线程池，并发消费时忽略该参数
*/
@Override
public void submitConsumeRequest(
  final List<MessageExt> msgs,
  final ProcessQueue processQueue,
  final MessageQueue messageQueue,
  final boolean dispatchToConsume) {

  //1. ，消息批次，在这里看来也就是一次消息消费任务 ConsumeRequest 中包含的消息条数，默认为1
  final int consumeBatchSize = this.defaultMQPushConsumer.getConsumeMessageBatchMaxSize();

  //1.1 直接将拉取到的消息放入到 ConsumeRequest 中，然后将 cosumeRequest 提交到消息消费者线程池中，
  if (msgs.size() <= consumeBatchSize) {
    ConsumeRequest consumeRequest = new ConsumeRequest(msgs, processQueue, messageQueue);
    try {
      //线程池异步执行，无界队列
      this.consumeExecutor.submit(consumeRequest);
    } catch (RejectedExecutionException e) {
      //提交过程中出现拒绝提交异常则延迟 5s 再提交
      //   这里其实是给出一种标准的拒绝提交实现方式，
      //   实际过程中由于消费者线程池使用的任务队列为LinkedBlockingQueue无界队列，故不会出现拒绝提交异常
      this.submitConsumeRequestLater(consumeRequest);
    }
  } else {
    //1.2 如果拉取的消息条数大于 consumeMessageBatchMaxSize 则对拉取消息进行分页，
    //   每页 consumeMessageBatchMaxSize 条消息，创建多个 ConsumeRequest 任务并提交到消费线程池
    for (int total = 0; total < msgs.size(); ) {
      List<MessageExt> msgThis = new ArrayList<MessageExt>(consumeBatchSize);
      for (int i = 0; i < consumeBatchSize; i++, total++) {
        if (total < msgs.size()) {
          msgThis.add(msgs.get(total));
        } else {
          break;
        }
      }

      ConsumeRequest consumeRequest = new ConsumeRequest(msgThis, processQueue, messageQueue);
      try {
        
        
        this.consumeExecutor.submit(consumeRequest);
      } catch (RejectedExecutionException e) {
        for (; total < msgs.size(); total++) {
          msgThis.add(msgs.get(total));
        }

        this.submitConsumeRequestLater(consumeRequest);
      }
    }
  }
}
```

* 上面可知都是进行了线程池的异步执行，那么去看看ConsumeRequest是什么



#### ConsumeRequest

* 通过源码可知实现Runnable，这样正好说明了，之前的线程池去异步执行任务，进行消息消费

* 源码

  ```java
  class ConsumeRequest implements Runnable {
    private final List<MessageExt> msgs;
    private final ProcessQueue processQueue;
    private final MessageQueue messageQueue;
  
    public ConsumeRequest(List<MessageExt> msgs, ProcessQueue processQueue, MessageQueue messageQueue) {
      this.msgs = msgs;
      this.processQueue = processQueue;
      this.messageQueue = messageQueue;
    }
  
    public List<MessageExt> getMsgs() {
      return msgs;
    }
  
    public ProcessQueue getProcessQueue() {
      return processQueue;
    }
  
    //具体的消息消费
    @Override
    public void run() {
  
      //1. 先检查 processQueue 的dropped ，如果设置为true，则停止该队列的消费，
      //   在进行消息重新负载时如果该消息队列被分配给消费组内其他消费者后，需要 droped 设置为 true，阻止消费者继续消费不属于自己的消息队列
      if (this.processQueue.isDropped()) {
        log.info("the message queue not be able to consume, because it's dropped. group={} {}", ConsumeMessageConcurrentlyService.this.consumerGroup, this.messageQueue);
        return;
      }
  
      MessageListenerConcurrently listener = ConsumeMessageConcurrentlyService.this.messageListener;
      ConsumeConcurrentlyContext context = new ConsumeConcurrentlyContext(messageQueue);
      ConsumeConcurrentlyStatus status = null;
  
      //TODO : 后续查看具体的内容
      defaultMQPushConsumerImpl.resetRetryAndNamespace(msgs, defaultMQPushConsumer.getConsumerGroup());
  
      //2. 执行消息消费钩子函数 ConsumeMessageHook#consumeMessageBefore 函数
      //      通过 consumer.getDefaultMQPushConsumerlmpl().registerConsumeMessageHook (hook）方法消息消费执行钩子函数
      ConsumeMessageContext consumeMessageContext = null;
      if (ConsumeMessageConcurrentlyService.this.defaultMQPushConsumerImpl.hasHook()) {
        consumeMessageContext = new ConsumeMessageContext();
        consumeMessageContext.setNamespace(defaultMQPushConsumer.getNamespace());
        consumeMessageContext.setConsumerGroup(defaultMQPushConsumer.getConsumerGroup());
        consumeMessageContext.setProps(new HashMap<String, String>());
        consumeMessageContext.setMq(messageQueue);
        consumeMessageContext.setMsgList(msgs);
        consumeMessageContext.setSuccess(false);
        ConsumeMessageConcurrentlyService.this.defaultMQPushConsumerImpl.executeHookBefore(consumeMessageContext);
      }
  
      long beginTimestamp = System.currentTimeMillis();
      boolean hasException = false;
      ConsumeReturnType returnType = ConsumeReturnType.SUCCESS;
      try {
        if (msgs != null && !msgs.isEmpty()) {
          for (MessageExt msg : msgs) {
            MessageAccessor.setConsumeStartTimeStamp(msg, String.valueOf(System.currentTimeMillis()));
          }
        }
  
        //3. 执行具体的消息消费，调用应用程序消息监昕器的 consumeMessage 方法，
        //   进入到具体的消息消费业务逻辑，返回该批消息的消费结果
        status = listener.consumeMessage(Collections.unmodifiableList(msgs), context);
      } catch (Throwable e) {
        log.warn("consumeMessage exception: {} Group: {} Msgs: {} MQ: {}",
                 RemotingHelper.exceptionSimpleDesc(e),
                 ConsumeMessageConcurrentlyService.this.consumerGroup,
                 msgs,
                 messageQueue);
        hasException = true;
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
      } else if (ConsumeConcurrentlyStatus.RECONSUME_LATER == status) {
        returnType = ConsumeReturnType.FAILED;
      } else if (ConsumeConcurrentlyStatus.CONSUME_SUCCESS == status) {
        returnType = ConsumeReturnType.SUCCESS;
      }
  
      if (ConsumeMessageConcurrentlyService.this.defaultMQPushConsumerImpl.hasHook()) {
        consumeMessageContext.getProps().put(MixAll.CONSUME_CONTEXT_TYPE, returnType.name());
      }
  
      if (null == status) {
        log.warn("consumeMessage return null, Group: {} Msgs: {} MQ: {}",
                 ConsumeMessageConcurrentlyService.this.consumerGroup,
                 msgs,
                 messageQueue);
        status = ConsumeConcurrentlyStatus.RECONSUME_LATER;
      }
  
  
      //4. 执行消息消费钩子函数 ConsumeMessageHook#consumeMessageAfter 函数
      if (ConsumeMessageConcurrentlyService.this.defaultMQPushConsumerImpl.hasHook()) {
        consumeMessageContext.setStatus(status.toString());
        consumeMessageContext.setSuccess(ConsumeConcurrentlyStatus.CONSUME_SUCCESS == status);
        ConsumeMessageConcurrentlyService.this.defaultMQPushConsumerImpl.executeHookAfter(consumeMessageContext);
      }
  
      //增加消费次数
      ConsumeMessageConcurrentlyService.this.getConsumerStatsManager()
        .incConsumeRT(ConsumeMessageConcurrentlyService.this.consumerGroup, messageQueue.getTopic(), consumeRT);
  
  
      //5. 在处理结果前再次验证一下 ProcessQueue 的 isDroped状态值，
      //   如果设置为 true ，将不对结果进行处理，也就是说如果在消息消费过程中进入到【2】时，
      //   如果由于由新的消费者加入或原先的消费者出现若机导致原先分给消费者的队列
      //   在负载之后分配给别的消费者，那么在应用程序的角度来看的话，消息会被重复消费
      if (!processQueue.isDropped()) {
        ConsumeMessageConcurrentlyService.this.processConsumeResult(status, context, this);
      } else {
        log.warn("processQueue is dropped without process consume result. messageQueue={}, msgs={}", messageQueue, msgs);
      }
    }
  
    public MessageQueue getMessageQueue() {
      return messageQueue;
    }
  
  }
  ```

  * 进而执行 ConsumeMessageConcurrentlyService#processConsumeResult方法

    ```java
    public void processConsumeResult(
      final ConsumeConcurrentlyStatus status,
      final ConsumeConcurrentlyContext context,
      final ConsumeRequest consumeRequest
    ) {
      int ackIndex = context.getAckIndex();
    
      if (consumeRequest.getMsgs().isEmpty())
        return;
    
      //根据消息监昕器返回的结果,计算 ackIndex ，
      //  如果返回 CONSUME_SUCCESS,acklndex 设置为 msgs.size()-1 ，
      //  如果返回 RECONSUME_LATER, acklndex -1
      //  这是为下文发送 msg back(ACK)消息做准备的
      switch (status) {
        case CONSUME_SUCCESS:
          if (ackIndex >= consumeRequest.getMsgs().size()) {
            ackIndex = consumeRequest.getMsgs().size() - 1;
          }
          int ok = ackIndex + 1;
          int failed = consumeRequest.getMsgs().size() - ok;
          //增加成功次数
          this.getConsumerStatsManager().incConsumeOKTPS(consumerGroup, consumeRequest.getMessageQueue().getTopic(), ok);
          //增加失败次数
          this.getConsumerStatsManager().incConsumeFailedTPS(consumerGroup, consumeRequest.getMessageQueue().getTopic(), failed);
          break;
        case RECONSUME_LATER://重新消费
          ackIndex = -1;
          this.getConsumerStatsManager().incConsumeFailedTPS(consumerGroup, consumeRequest.getMessageQueue().getTopic(),
                                                             consumeRequest.getMsgs().size());
          break;
        default:
          break;
      }
    
      switch (this.defaultMQPushConsumer.getMessageModel()) {
          //如果是集群模式， 业务方返回 RECONSUME_LATER ，消息并不会重新被消费，只是以警告级别输出到日志文件
        case BROADCASTING:
          for (int i = ackIndex + 1; i < consumeRequest.getMsgs().size(); i++) {
            MessageExt msg = consumeRequest.getMsgs().get(i);
            log.warn("BROADCASTING, the message consume failed, drop it, {}", msg.toString());
          }
          break;
    
          //如果是集群模式， 消息消费成功，由于 ackIndex=consumeRequest.getMsgs().size()-1,
          // 故 i=ackIndex+1  等于 consumeRequest.getMsgs().size()
          //并不会执行 sendMessageBack
        case CLUSTERING:
          List<MessageExt> msgBackFailed = new ArrayList<MessageExt>(consumeRequest.getMsgs().size());
          for (int i = ackIndex + 1; i < consumeRequest.getMsgs().size(); i++) {
            //只有当 业务方 返回 RECONSUME_LATER ackIndex = -1，
            MessageExt msg = consumeRequest.getMsgs().get(i);
    
            //----------------发送ACK消息-----------------
            //该批消息都需要发 ACK 消息
            boolean result = this.sendMessageBack(msg, context);
    
            //如果消息发送 ACK 失败，则直接将本批 ACK 消费发送失败的消息再次封装为 ConsumeRequest ，
            //然后延迟 Ss 重新消费。如果 ACK 消息发送成功，则该消息会延迟消费
            if (!result) {
              msg.setReconsumeTimes(msg.getReconsumeTimes() + 1);
              msgBackFailed.add(msg);
            }
          }
    
    
          //重新消费
          if (!msgBackFailed.isEmpty()) {
            consumeRequest.getMsgs().removeAll(msgBackFailed);
            //重新发送消费请求
            this.submitConsumeRequestLater(msgBackFailed, consumeRequest.getProcessQueue(), consumeRequest.getMessageQueue());
          }
          break;
        default:
          break;
      }
    
    
      //从 ProcessQueue 移除这批消息， 这里返回的偏移量是移除该批消息后最小的偏移量，
      // 然后用该偏移量更新消息消费进度 ，以便在消费者重启后能从上一次的消费进度开始消费，避免消息重复消费
      long offset = consumeRequest.getProcessQueue().removeMessage(consumeRequest.getMsgs());
      if (offset >= 0 && !consumeRequest.getProcessQueue().isDropped()) {
    
        
        //--------------跟新消息队列的消费进度-------------
        //跟新消息队列的消费进度
        this.defaultMQPushConsumerImpl.getOffsetStore().updateOffset(consumeRequest.getMessageQueue(), offset, true);
      }
    
      /*
                跟新进度的时候，发现返回RECONSUME_LATER也会进行更新
    
                    原因：重新消费的消息，虽然和之前的消息一样，但是对于Broker而言，会创建一条与原先消息属性相同的消息
    
                         拥有一个唯一的新 msgld ，并存储原消息ID ，该消息会存入到 commitlog 文件中，
                         与原先的消息没有任何关联，那该消息当然也会进入到ConsuemeQueue 队列中，将拥有
                         一个全新的队列偏移量
             */
    }
    ```

* 上面说明了消息的消费的整体流程，根据业务执行的返回状态，判断是否消息消费成功，更新本地存储的消息消费进度，消费完成之后会发送给Broker ACK来告知消息消费情况，那么必然涉及两个大点

  * 消息确认（ACK）
  * 消息消费进度存储问题



### 消息确认

* 如果消息监听器返回的消费结果为 RECONSUME_LATER ，则需要将这些消息发送给Broker 延迟消息。如果发送 ACK 消息失败，将延迟 5s 后提交线程池进行消费。
* ACK消息发送的网络客户端人口： MQClientAPIImpl#consumerSendMessageBack ，命令编码：RequestCode. CONSUMER_SEND_MSG_BACK



#### 消息确认发送端

##### MQClientImpl

###### consumerSendMessageBack

```java
public void consumerSendMessageBack(
  final String addr,
  final MessageExt msg,
  final String consumerGroup,
  final int delayLevel,
  final long timeoutMillis,
  final int maxConsumeRetryTimes
) throws RemotingException, MQBrokerException, InterruptedException {

  //协议头部信息：延迟消费的请求头信息
  ConsumerSendMsgBackRequestHeader requestHeader = new ConsumerSendMsgBackRequestHeader();
  RemotingCommand request = RemotingCommand.createRequestCommand(RequestCode.CONSUMER_SEND_MSG_BACK, requestHeader);

  requestHeader.setGroup(consumerGroup);//消费组名
  requestHeader.setOriginTopic(msg.getTopic());//消息主题
  requestHeader.setOffset(msg.getCommitLogOffset());//偏移量
  requestHeader.setDelayLevel(delayLevel);//延迟等级
  requestHeader.setOriginMsgId(msg.getMsgId());//消息id
  requestHeader.setMaxReconsumeTimes(maxConsumeRetryTimes);//最大重试次数

  //发送给Broker
  RemotingCommand response = this.remotingClient.invokeSync(MixAll.brokerVIPChannel(this.clientConfig.isVipChannelEnabled(), addr),
                                                            request, timeoutMillis);
  assert response != null;
  switch (response.getCode()) {
    case ResponseCode.SUCCESS: {
      return;
    }
    default:
      break;
  }

  throw new MQBrokerException(response.getCode(), response.getRemark());
}
```





#### 消息确认接收端

* SendMessageProcessor是NettyRequestProcessor，用来处理Netty请求，查看processRequest方法

  ```java
  @Override
  public RemotingCommand processRequest(ChannelHandlerContext ctx,
                                        RemotingCommand request) throws RemotingCommandException {
    RemotingCommand response = null;
    try {
      //异步处理请求
      response = asyncProcessRequest(ctx, request).get();
    } catch (InterruptedException | ExecutionException e) {
      log.error("process SendMessage error, request : " + request.toString(), e);
    }
    return response;
  }
  ```

  ```java
  public CompletableFuture<RemotingCommand> asyncProcessRequest(ChannelHandlerContext ctx,
                                                                RemotingCommand request) throws RemotingCommandException {
    final SendMessageContext mqtraceContext;
    switch (request.getCode()) {
      //ACK  
      case RequestCode.CONSUMER_SEND_MSG_BACK:
        return this.asyncConsumerSendMsgBack(ctx, request);
      default:
        SendMessageRequestHeader requestHeader = parseRequestHeader(request);
        if (requestHeader == null) {
          return CompletableFuture.completedFuture(null);
        }
        mqtraceContext = buildMsgContext(ctx, requestHeader);
        this.executeSendMessageHookBefore(ctx, request, mqtraceContext);
        if (requestHeader.isBatch()) {
          return this.asyncSendBatchMessage(ctx, request, mqtraceContext, requestHeader);
        } else {
          return this.asyncSendMessage(ctx, request, mqtraceContext, requestHeader);
        }
    }
  }
  ```

* 关键方法：asyncConsumerSendMsgBack

  ```java
  private CompletableFuture<RemotingCommand> asyncConsumerSendMsgBack(ChannelHandlerContext ctx,
                                                                      RemotingCommand request) throws RemotingCommandException {
    final RemotingCommand response = RemotingCommand.createResponseCommand(null);
    final ConsumerSendMsgBackRequestHeader requestHeader =
      (ConsumerSendMsgBackRequestHeader)request.decodeCommandCustomHeader(ConsumerSendMsgBackRequestHeader.class);
    String namespace = NamespaceUtil.getNamespaceFromResource(requestHeader.getGroup());
    if (this.hasConsumeMessageHook() && !UtilAll.isBlank(requestHeader.getOriginMsgId())) {
      ConsumeMessageContext context = buildConsumeMessageContext(namespace, requestHeader, request);
      this.executeConsumeMessageHookAfter(context);
    }
  
    //1. 获取消费组的订阅配置信息
    //消费组订阅信息配置信息存储在 Broker 的 ${ROCKET_HOME}/store/config/subscriptionGroup.json
    SubscriptionGroupConfig subscriptionGroupConfig =
      this.brokerController.getSubscriptionGroupManager().findSubscriptionGroupConfig(requestHeader.getGroup());
    if (null == subscriptionGroupConfig) {
      response.setCode(ResponseCode.SUBSCRIPTION_GROUP_NOT_EXIST);
      response.setRemark("subscription group not exist, " + requestHeader.getGroup() + " "
                         + FAQUrl.suggestTodo(FAQUrl.SUBSCRIPTION_GROUP_NOT_EXIST));
      return CompletableFuture.completedFuture(response);
    }
    if (!PermName.isWriteable(this.brokerController.getBrokerConfig().getBrokerPermission())) {
      response.setCode(ResponseCode.NO_PERMISSION);
      response.setRemark("the broker[" + this.brokerController.getBrokerConfig().getBrokerIP1() + "] sending message is forbidden");
      return CompletableFuture.completedFuture(response);
    }
  
    if (subscriptionGroupConfig.getRetryQueueNums() <= 0) {
      response.setCode(ResponseCode.SUCCESS);
      response.setRemark(null);
      return CompletableFuture.completedFuture(response);
    }
  
    //2. 创建重试主题，重试主题名称 %RETRY%＋消费组名称，并从重试队列中随机选择一个队列 ，并构建 TopicConfig 主题配置信息
    String newTopic = MixAll.getRetryTopic(requestHeader.getGroup());
    int queueIdInt = Math.abs(this.random.nextInt() % 99999999) % subscriptionGroupConfig.getRetryQueueNums();
    int topicSysFlag = 0;
    if (requestHeader.isUnitMode()) {
      topicSysFlag = TopicSysFlag.buildSysFlag(false, true);
    }
  
    TopicConfig topicConfig = this.brokerController.getTopicConfigManager().createTopicInSendMessageBackMethod(
      newTopic,
      subscriptionGroupConfig.getRetryQueueNums(),
      PermName.PERM_WRITE | PermName.PERM_READ, topicSysFlag);
    if (null == topicConfig) {
      response.setCode(ResponseCode.SYSTEM_ERROR);
      response.setRemark("topic[" + newTopic + "] not exist");
      return CompletableFuture.completedFuture(response);
    }
  
    if (!PermName.isWriteable(topicConfig.getPerm())) {
      response.setCode(ResponseCode.NO_PERMISSION);
      response.setRemark(String.format("the topic[%s] sending message is forbidden", newTopic));
      return CompletableFuture.completedFuture(response);
    }
  
  
    //3. 根据消息物理偏移量 commitlog 文件中获取消息， 同时将消息的主题存入属性中
    MessageExt msgExt = this.brokerController.getMessageStore().lookMessageByOffset(requestHeader.getOffset());
    if (null == msgExt) {
      response.setCode(ResponseCode.SYSTEM_ERROR);
      response.setRemark("look message by offset failed, " + requestHeader.getOffset());
      return CompletableFuture.completedFuture(response);
    }
  
    final String retryTopic = msgExt.getProperty(MessageConst.PROPERTY_RETRY_TOPIC);
    if (null == retryTopic) {
      MessageAccessor.putProperty(msgExt, MessageConst.PROPERTY_RETRY_TOPIC, msgExt.getTopic());
    }
    msgExt.setWaitStoreMsgOK(false);
  
    int delayLevel = requestHeader.getDelayLevel();
  
  
  
    int maxReconsumeTimes = subscriptionGroupConfig.getRetryMaxTimes();
    if (request.getVersion() >= MQVersion.Version.V3_4_9.ordinal()) {
      maxReconsumeTimes = requestHeader.getMaxReconsumeTimes();
    }
  
    //4. ：设置消息重试次数， 如果消息已重试次数超过 maxReconsumeTimes
    if (msgExt.getReconsumeTimes() >= maxReconsumeTimes 
        || delayLevel < 0) {
  
      // 再次改变newTopic 主题为 DLQ （” %DLQ%”），死信队列，该主题的权限为只写，说明消息一旦进入到 DLQ 列中，
      // RocketMQ 将不负责再次调度进行消费了， 需要人工干预
      newTopic = MixAll.getDLQTopic(requestHeader.getGroup());
      queueIdInt = Math.abs(this.random.nextInt() % 99999999) % DLQ_NUMS_PER_GROUP;
  
      topicConfig = this.brokerController.getTopicConfigManager().createTopicInSendMessageBackMethod(newTopic,
                                                                                                     DLQ_NUMS_PER_GROUP,
                                                                                                     PermName.PERM_WRITE, 0);
      if (null == topicConfig) {
        response.setCode(ResponseCode.SYSTEM_ERROR);
        response.setRemark("topic[" + newTopic + "] not exist");
        return CompletableFuture.completedFuture(response);
      }
    } else {
      if (0 == delayLevel) {
        delayLevel = 3 + msgExt.getReconsumeTimes();
      }
      msgExt.setDelayTimeLevel(delayLevel);
    }
  
  
    //5. 根据原先的消息创建一个新的消息对象
    MessageExtBrokerInner msgInner = new MessageExtBrokerInner();
    msgInner.setTopic(newTopic);
    msgInner.setBody(msgExt.getBody());
    msgInner.setFlag(msgExt.getFlag());
    MessageAccessor.setProperties(msgInner, msgExt.getProperties());
    msgInner.setPropertiesString(MessageDecoder.messageProperties2String(msgExt.getProperties()));
    msgInner.setTagsCode(MessageExtBrokerInner.tagsString2tagsCode(null, msgExt.getTags()));
  
    msgInner.setQueueId(queueIdInt);
    msgInner.setSysFlag(msgExt.getSysFlag());
    msgInner.setBornTimestamp(msgExt.getBornTimestamp());
    msgInner.setBornHost(msgExt.getBornHost());
    msgInner.setStoreHost(msgExt.getStoreHost());
    msgInner.setReconsumeTimes(msgExt.getReconsumeTimes() + 1);
  
    //5.1 重试消息会拥有自 的唯一消息 ID
    //( ms ld ）并存人到 comm tlo 中，并不会去更新原先消 是会将原先的主题、 消息
    //ID 存入消息的属 主题名 为重试 性与原先消息保持相同
    String originMsgId = MessageAccessor.getOriginMessageId(msgExt);
    MessageAccessor.setOriginMessageId(msgInner, UtilAll.isBlank(originMsgId) ? msgExt.getMsgId() : originMsgId);
  
    //6. 将消息存入到 CommitLog 件中
    CompletableFuture<PutMessageResult> putMessageResult = this.brokerController.getMessageStore().asyncPutMessage(msgInner);
    return putMessageResult.thenApply((r) -> {
      if (r != null) {
        switch (r.getPutMessageStatus()) {
          case PUT_OK:
            String backTopic = msgExt.getTopic();
            String correctTopic = msgExt.getProperty(MessageConst.PROPERTY_RETRY_TOPIC);
            if (correctTopic != null) {
              backTopic = correctTopic;
            }
            this.brokerController.getBrokerStatsManager().incSendBackNums(requestHeader.getGroup(), backTopic);
            response.setCode(ResponseCode.SUCCESS);
            response.setRemark(null);
            return response;
          default:
            break;
        }
        response.setCode(ResponseCode.SYSTEM_ERROR);
        response.setRemark(r.getPutMessageStatus().name());
        return response;
      }
      response.setCode(ResponseCode.SYSTEM_ERROR);
      response.setRemark("putMessageResult is null");
      return response;
    });
  }
  ```

  * asyncPutMessage中最后调用的是commitlog的putMessage，此方法中
    * 如果消息的延迟级别 delayTimeLevel 大于0，替换消息的主题与队列为定时任务主题“ SCHEDULE_TOPIC_XXXX ”，队列 ID 为延迟级别减1。
    * 再次将消息主题、队列存入消息的属性中，键分别为 PROPERTY REAL TOPIC，PROPERTY _REAL QUEUE ID
  * 后续的执行
    * ACK 消息存入 CommitLog 文件后 ，将依 RocketMQ 定时消息机制在延迟时间到期后再次将消息拉取，提交消费线程池。
  * 总结
    * 延迟任务是和定时任务机制有关系的。下一部分专门针对学习





#### 消息进度管理

* 消息消费者在消费一批消息后，需要记录该批消息已经消费完毕，否则当消费者重新启动时又得从消息消费队列的开始消费，这显然是不能接受的

* 消息消费流程可知

  * 一次消息消费后会从 ProcessQueue 处理队列中移除该批消息，返回 ProcessQueue 最小偏移量，并存入消息进度表中 那消息进度文件存储在哪合适呢？

* 消息进度文件存储

  * 广播模式：同一个消费组的所有消息消费者都需要消费主题下的所有消息，也就是同组内的消费者的消息消费行为是对立的，互相不影响，故消息进度需要独立存储，最理想的存储地方应该是与消费者绑定
  * 集群模式：同一个消费组内的所有消息消费者共享消息主题下的所有消息，同一条消息（同一个消息消费队列）在同一时间只会被消费组内的一个消费者消费，并且随着消费队列的动态变化重新负载，所以消费进度需要保存在一个每个消费者都能访问到的地方

* RocketMQ消息消费进度接口

  <img src="../../../../照片/typora/image-20210818160012772.png" alt="image-20210818160012772" style="zoom:50%;" />

  ```java
  public interface OffsetStore {
      /**
       * 从消息进度存储文件加载消息进度到内存
       */
      void load() throws MQClientException;
  
      /**
       * 更新内存中的消息消费进度\
       * @param mq 消息消费队列
       * @param offset 消息消费偏移量
       * @param increaseOnly true 表示 offset 必须大于内存中当前的消费偏移量才更新
       */
      void updateOffset(final MessageQueue mq, final long offset, final boolean increaseOnly);
  
      /**
       * 读取消息消费进度
       * @param mq 消息消费队列
       * @param type  读取方式，可选值
       *              READ_FROM_MEMORY ：从内 中；
       *              READ_FROM_STORE 从磁盘中
       *              MEMORY_FIRST_THEN_STORE 先从内存读取，再从磁盘读取
       */
      long readOffset(final MessageQueue mq, final ReadOffsetType type);
  
      /**
       * 持久化指定消息队列进度到磁盘
       * @param mqs 消息队列集合
       */
      void persistAll(final Set<MessageQueue> mqs);
  
      /**
       * Persist the offset,may be in local storage or remote name server
       */
      void persist(final MessageQueue mq);
  
      /**
       * Remove offset
       * 将消息队列的消息消费进度从内存中移除
       */
      void removeOffset(MessageQueue mq);
  
      /**
       * @return The cloned offset table of given topic
       * 克隆该主题下所有消息队列的消息消费进度
       */
      Map<MessageQueue, Long> cloneOffsetTable(String topic);
  
      /**
       * 更新存储在 Brokder 端的消息消费进度，使用集群模式
       * @param mq
       * @param offset
       * @param isOneway
       */
      void updateConsumeOffsetToBroker(MessageQueue mq, long offset, boolean isOneway) throws RemotingException,
          MQBrokerException, InterruptedException, MQClientException;
  }
  ```



##### 广播模式消费进度存储：LocalFileOffsetStore

* 比较简单，直接上类，重要方法分析

  ```java
  public class LocalFileOffsetStore implements OffsetStore {
  
      //消息进度存储目录， 可以通过－Drocketmq.client.localOffsetStoreDir
      //如果未指定 ，则默认为用户主目录 /.rocketmq_offsets
      public final static String LOCAL_OFFSET_STORE_DIR = System.getProperty(
          "rocketmq.client.localOffsetStoreDir",
          System.getProperty("user.home") + File.separator + ".rocketmq_offsets");
      private final static InternalLogger log = ClientLogger.getLog();
  
      //消息客户端
      private final MQClientInstance mQClientFactory;
  
      //消息消费组
      private final String groupName;
  
      //消息进度存储文件， LOCAL_OFFSET_STORE_DIR/.rocketmq_ offsets/{ mQClientFactory.getClientDd()}/groupName/offsets.json
      private final String storePath;
  
      //消息队列消费进度(内存)
      private ConcurrentMap<MessageQueue, AtomicLong> offsetTable =
          new ConcurrentHashMap<MessageQueue, AtomicLong>();
  
      public LocalFileOffsetStore(MQClientInstance mQClientFactory, String groupName) {
          this.mQClientFactory = mQClientFactory;
          this.groupName = groupName;
          this.storePath = LOCAL_OFFSET_STORE_DIR + File.separator +
              this.mQClientFactory.getClientId() + File.separator +
              this.groupName + File.separator +
              "offsets.json";
      }
  
      @Override
      public void load() throws MQClientException {
  
          //从本地文件获取
          OffsetSerializeWrapper offsetSerializeWrapper = this.readLocalOffset();
          if (offsetSerializeWrapper != null && offsetSerializeWrapper.getOffsetTable() != null) {
              offsetTable.putAll(offsetSerializeWrapper.getOffsetTable());
  
              for (MessageQueue mq : offsetSerializeWrapper.getOffsetTable().keySet()) {
                  AtomicLong offset = offsetSerializeWrapper.getOffsetTable().get(mq);
                  log.info("load consumer's offset, {} {} {}",
                      this.groupName,
                      mq,
                      offset.get());
              }
          }
      }
  
  
      /**
       * 跟新消息消费进度
       * @param mq
       * @param offset
       * @param increaseOnly
       */
      @Override
      public void updateOffset(MessageQueue mq, long offset, boolean increaseOnly) {
          if (mq != null) {
              AtomicLong offsetOld = this.offsetTable.get(mq);
              if (null == offsetOld) {
                  offsetOld = this.offsetTable.putIfAbsent(mq, new AtomicLong(offset));
              }
  
              if (null != offsetOld) {
                  if (increaseOnly) {
                      MixAll.compareAndIncreaseOnly(offsetOld, offset);
                  } else {
                      offsetOld.set(offset);
                  }
              }
          }
      }
  
      @Override
      public long readOffset(final MessageQueue mq, final ReadOffsetType type) {
          if (mq != null) {
              switch (type) {
                  case MEMORY_FIRST_THEN_STORE:
                  case READ_FROM_MEMORY: {
                      AtomicLong offset = this.offsetTable.get(mq);
                      if (offset != null) {
                          return offset.get();
                      } else if (ReadOffsetType.READ_FROM_MEMORY == type) {
                          return -1;
                      }
                  }
                  case READ_FROM_STORE: {
                      OffsetSerializeWrapper offsetSerializeWrapper;
                      try {
                          offsetSerializeWrapper = this.readLocalOffset();
                      } catch (MQClientException e) {
                          return -1;
                      }
                      if (offsetSerializeWrapper != null && offsetSerializeWrapper.getOffsetTable() != null) {
                          AtomicLong offset = offsetSerializeWrapper.getOffsetTable().get(mq);
                          if (offset != null) {
                              this.updateOffset(mq, offset.get(), false);
                              return offset.get();
                          }
                      }
                  }
                  default:
                      break;
              }
          }
  
          return -1;
      }
  
      public static void main(String[] args) {
          try {
              MixAll.string2File("hello","/Users/lishijie/Documents/a.txt");
          } catch (IOException e) {
              e.printStackTrace();
          }
      }
  
  
      @Override
      public void persistAll(Set<MessageQueue> mqs) {
          if (null == mqs || mqs.isEmpty())
              return;
  
          //持久化消息进度就 是将 ConcurrentMap<MessageQueue, AtomicLong> offsetTable序列化到磁盘文件中
          OffsetSerializeWrapper offsetSerializeWrapper = new OffsetSerializeWrapper();
          for (Map.Entry<MessageQueue, AtomicLong> entry : this.offsetTable.entrySet()) {
              if (mqs.contains(entry.getKey())) {
                  AtomicLong offset = entry.getValue();
                  offsetSerializeWrapper.getOffsetTable().put(entry.getKey(), offset);
              }
          }
  
          String jsonString = offsetSerializeWrapper.toJson(true);
          if (jsonString != null) {
              try {
                  MixAll.string2File(jsonString, this.storePath);
              } catch (IOException e) {
                  log.error("persistAll consumer offset Exception, " + this.storePath, e);
              }
          }
      }
  
      @Override
      public void persist(MessageQueue mq) {
      }
  
      @Override
      public void removeOffset(MessageQueue mq) {
  
      }
  
      @Override
      public void updateConsumeOffsetToBroker(final MessageQueue mq, final long offset, final boolean isOneway)
          throws RemotingException, MQBrokerException, InterruptedException, MQClientException {
  
      }
  
      @Override
      public Map<MessageQueue, Long> cloneOffsetTable(String topic) {
          Map<MessageQueue, Long> cloneOffsetTable = new HashMap<MessageQueue, Long>();
          for (Map.Entry<MessageQueue, AtomicLong> entry : this.offsetTable.entrySet()) {
              MessageQueue mq = entry.getKey();
              if (!UtilAll.isBlank(topic) && !topic.equals(mq.getTopic())) {
                  continue;
              }
              cloneOffsetTable.put(mq, entry.getValue().get());
  
          }
          return cloneOffsetTable;
      }
  
      private OffsetSerializeWrapper readLocalOffset() throws MQClientException {
          String content = null;
          try {
              //1. 获取storePath路径
              content = MixAll.file2String(this.storePath);
          } catch (IOException e) {
              log.warn("Load local offset store file exception", e);
          }
  
  
          //2. 从 storePath ＋” .bak" 尝试加载， 如果还是未找到，则返回 null
          if (null == content || content.length() == 0) {
              return this.readLocalOffsetBak();
          } else {
              //1.1 从storePath中读取消息消费进度
              OffsetSerializeWrapper offsetSerializeWrapper = null;
              try {
                  offsetSerializeWrapper =
                      OffsetSerializeWrapper.fromJson(content, OffsetSerializeWrapper.class);
              } catch (Exception e) {
                  log.warn("readLocalOffset Exception, and try to correct", e);
                  return this.readLocalOffsetBak();
              }
  
              return offsetSerializeWrapper;
          }
      }
  
      private OffsetSerializeWrapper readLocalOffsetBak() throws MQClientException {
          String content = null;
          try {
              content = MixAll.file2String(this.storePath + ".bak");
          } catch (IOException e) {
              log.warn("Load local offset store bak file exception", e);
          }
          if (content != null && content.length() > 0) {
              OffsetSerializeWrapper offsetSerializeWrapper = null;
              try {
                  offsetSerializeWrapper =
                      OffsetSerializeWrapper.fromJson(content, OffsetSerializeWrapper.class);
              } catch (Exception e) {
                  log.warn("readLocalOffset Exception", e);
                  throw new MQClientException("readLocalOffset Exception, maybe fastjson version too low"
                      + FAQUrl.suggestTodo(FAQUrl.LOAD_JSON_EXCEPTION),
                      e);
              }
              return offsetSerializeWrapper;
          }
  
          return null;
      }
  }
  
  ```

* 对于persistAll方法的调用，在MQC!ientlnstance 中会启动 个定时任务，默认每 5s 持久化 次，可通过 persistConsumerOffsetlnterval 设置

  ```java
  this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
  
    @Override
    public void run() {
      try {
        //底层是persistAll
        MQClientInstance.this.persistAllConsumerOffset();
      } catch (Exception e) {
        log.error("ScheduledTask persistAllConsumerOffset exception", e);
      }
    }
  }, 1000 * 10, this.clientConfig.getPersistConsumerOffsetInterval(), TimeUnit.MILLISECONDS);
  ```



##### 集群模式消费进度存储

* 集群模式消息进度存储文件存放在消息服务端 roker 消息消费进度集群模式实现类：org apache.rocketmq.client.consumer.store.RemoteBrokerOffsetStore

* 原理图

  <img src="../../../../照片/typora/image-20210818162919742.png" alt="image-20210818162919742" style="zoom:50%;" />

* 集群模式消息进度

  * 如果从内存中读取消费进度，则从 RemoteBrokerOffsetStore 的ConcurrentMap<MessageQueue, AtomicLong> offsetTable =new ConcurrentHashMap<Messag Qu ue, AtomicLong＞（）中根据消息消费队列获取其消息消费进度；
  * 如果从磁盘读取，则发送网络请求，请求命令为 QUERY_CONSUMER_OFFSET 持久化消息进度，则请求命令为 UPDATE_CONSUMER _OFFSET, 更新 ConsumerOffsetManager的 ConcurrentMap<Str ng/* topic@group， ConcurrentMap<Integer /*消息队列 ID * /， Long/＊消息消费进度＊／>offsetTable, 
    * Broker 端默认 10s 持久化一次消息进度，存储文件名：${RocketMQ_ HOME}/store/config/consumerOffset.json

* 思考

  * 消费者线程池每处理完一个消息消费任务（ ConsumeRequest ）时会从 ProceeQueue中移除本批消 的消息 ，并返回 ProcessQueue中最小的偏移量，用该偏移量更新消息队列消费进度，也就是说更新消费进度与消费任务中的消息没什么关系。
    * 例如现在两个消费任务 task1( queueOffset 分别为 20,40 ), task2 ( 50,70 ），并且ProceeQueue 中当前包含最小消息偏移量为 10 的消息，则task2 消费结束后，将使用 10 去更新消费进度， 并不会是70。
    * 当task1 消费结束后 ，还是以 10 去更新消费队列消息进度，消息消费进度的推进取决于ProceeQueue 中偏移量最小的消息消费速度。 如果偏移量为 10 的消息消费成功后，假如ProceeQueue 中包含消息偏移量为 100 消息， 则消息偏移量为 10 的消息消费成功后，将直接 100 更新消息消费进度。
    * 那如果在消费消息偏移量为 10 的消息时发送了死锁导致一直无法被消费， 那岂不是消息进度无法向前推进，是的，为了避免这种情况， RocketMQ引入了一种消息拉取流控措施： DefaultMQPushConsumer#consumeConcurrentlyMaxSpan=2000 ，消息处理队列ProceeQueue 最大消息偏移与最小偏移量不能超过该值，如超过该值，触发流控，将延迟该消息队列的消息拉取
  * 触发消息消费进度更新的另外一个是在进行消息负载时，如果消息消费队列被分配给其他消 费者时，此时会将该 ProceeQueue 状态设置为 drop时，持久化该消息队列的消费进度，并从内存中移除

  

