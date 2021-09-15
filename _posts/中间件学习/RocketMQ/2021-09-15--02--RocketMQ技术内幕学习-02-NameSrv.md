---
layout:     post
title:      "RocketMQ技术内幕学习-针对点-02--ServiceThread"
date:       2021-09-01 12:00:00
author:     "LSJ"
header-img: "img/post-bg-2015.jpg"
tags:
    RocketMQ
    NameServer
---







## 探索点

* 作用：
  * 分布式服务SOA的注册中心主要提供服务调用的解析服务，指引服务调用方法（消费方）找到远处的服务提供者
* 问题
  * 问题一：存储什么数据？
    * 存储着Topic和Broker的关联信息，具体是不是呢？
  * 问题二：如果避免单点故障，提高高可用？？？
    * 相当于热备，部署多个互不通讯的NameServer，多个节点部署保证高可用





## 知识点

![RocketMQ技术内幕-02--NameSrv](../../../img/RocketMQ技术内幕-02--NameSrv.png)



## 启动流程

### 时序图

![image-20210728160328983](../../../img/image-20210728160328983.png)



#### 具体源码

##### NamesrvStartup

1. NamesrvStartup.main

   ```java
   public static void main(String[] args) {
     main0(args);
   }
   
   public static NamesrvController main0(String[] args) {
   
     try {
   
       //解析配置文件，创建填充NameServerConfig,NettyServerConfig属性值
       NamesrvController controller =     public static void main(String[] args) {
           main0(args);
       }
   
       public static NamesrvController main0(String[] args) {
   
           try {
   
               //解析配置文件，创建填充NameServerConfig,NettyServerConfig属性值
               NamesrvController controller = createNamesrvController(args);
               start(controller);
               String tip = "The Name Server boot success. serializeType=" + RemotingCommand.getSerializeTypeConfigInThisServer();
               log.info(tip);
               System.out.printf("%s%n", tip);
               return controller;
           } catch (Throwable e) {
               e.printStackTrace();
               System.exit(-1);
           }
   
           return null;
       }(args);
       start(controller);
       String tip = "The Name Server boot success. serializeType=" + RemotingCommand.getSerializeTypeConfigInThisServer();
       log.info(tip);
       System.out.printf("%s%n", tip);
       return controller;
     } catch (Throwable e) {
       e.printStackTrace();
       System.exit(-1);
     }
   
     return null;
   }
   ```

2. createNamesrvController

   ```java
   public static NamesrvController createNamesrvController(String[] args) throws IOException, JoranException {
     System.setProperty(RemotingCommand.REMOTING_VERSION_KEY, Integer.toString(MQVersion.CURRENT_VERSION));
     //PackageConflictDetect.detectFastjson();
   
     Options options = ServerUtil.buildCommandlineOptions(new Options());
     commandLine = ServerUtil.parseCmdLine("mqnamesrv", args, buildCommandlineOptions(options), new PosixParser());
     if (null == commandLine) {
       System.exit(-1);
       return null;
     }
   
   
     //1. 创建NamesrvConfig配置文件
     final NamesrvConfig namesrvConfig = new NamesrvConfig();
     final NettyServerConfig nettyServerConfig = new NettyServerConfig();
     nettyServerConfig.setListenPort(9876);
   
     //配置文件的目录地址
     if (commandLine.hasOption('c')) {
       String file = commandLine.getOptionValue('c');
       if (file != null) {
         InputStream in = new BufferedInputStream(new FileInputStream(file));
         properties = new Properties();
         properties.load(in);
         
         //此工具类，将namesrvConfig中的基本类型，使用properties进行注入
         MixAll.properties2Object(properties, namesrvConfig);
         MixAll.properties2Object(properties, nettyServerConfig);
   
         namesrvConfig.setConfigStorePath(file);
   
         System.out.printf("load config properties file OK, %s%n", file);
         in.close();
       }
     }
   
     if (commandLine.hasOption('p')) {
       InternalLogger console = InternalLoggerFactory.getLogger(LoggerName.NAMESRV_CONSOLE_NAME);
       MixAll.printObjectProperties(console, namesrvConfig);
       MixAll.printObjectProperties(console, nettyServerConfig);
       System.exit(0);
     }
   
     MixAll.properties2Object(ServerUtil.commandLine2Properties(commandLine), namesrvConfig);
   
     if (null == namesrvConfig.getRocketmqHome()) {
       System.out.printf("Please set the %s variable in your environment to match the location of the RocketMQ installation%n", MixAll.ROCKETMQ_HOME_ENV);
       System.exit(-2);
     }
   
     LoggerContext lc = (LoggerContext) LoggerFactory.getILoggerFactory();
     JoranConfigurator configurator = new JoranConfigurator();
     configurator.setContext(lc);
     lc.reset();
     configurator.doConfigure(namesrvConfig.getRocketmqHome() + "/conf/logback_namesrv.xml");
   
     log = InternalLoggerFactory.getLogger(LoggerName.NAMESRV_LOGGER_NAME);
   
     MixAll.printObjectProperties(log, namesrvConfig);
     MixAll.printObjectProperties(log, nettyServerConfig);
   
     final NamesrvController controller = new NamesrvController(namesrvConfig, nettyServerConfig);
   
     // remember all configs to prevent discard
     controller.getConfiguration().registerConfig(properties);
   
     return controller;
   }
   ```

3. start()

   ```java
   public static NamesrvController start(final NamesrvController controller) throws Exception {
   
     if (null == controller) {
       throw new IllegalArgumentException("NamesrvController is null");
     }
   
   
     //1. NameSrvController初始化
     boolean initResult = controller.initialize();
     if (!initResult) {
       controller.shutdown();
       System.exit(-3);
     }
   
     //2. 注册JVM钩子函数
     //优点：如果代码中使用了线程池，一种优雅停机的方式是注册一个JVM钩子函数，在JVM进程关闭之前，先将线程池关闭，及时释放资源
     Runtime.getRuntime().addShutdownHook(new ShutdownHookThread(log, new Callable<Void>() {
       @Override
       public Void call() throws Exception {
         controller.shutdown();
         return null;
       }
     }));
   
     controller.start();
   
     return controller;
   }
   
   ```

##### NamesrvController

* Initialize()

  ```java
  public boolean initialize() {
  
    //1. 加载KV配置
    this.kvConfigManager.load();
  
    //2. 创建Netty网络处理对象
    this.remotingServer = new NettyRemotingServer(this.nettyServerConfig, this.brokerHousekeepingService);
  
    this.remotingExecutor =
      Executors.newFixedThreadPool(nettyServerConfig.getServerWorkerThreads(), new ThreadFactoryImpl("RemotingExecutorThread_"));
  
    this.registerProcessor();
  
  
    //3. 开启定时任务，每隔10s扫描一个Broker，移除处于不激活状态的broker
    this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
  
      @Override
      public void run() {
        NamesrvController.this.routeInfoManager.scanNotActiveBroker();
      }
    }, 5, 10, TimeUnit.SECONDS);
  
  
    //4. 开启单线程的定时任务，每隔10分钟打印一次KV配置
    this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
  
      @Override
      public void run() {
        NamesrvController.this.kvConfigManager.printAllPeriodically();
      }
    }, 1, 10, TimeUnit.MINUTES);
  
    if (TlsSystemConfig.tlsMode != TlsMode.DISABLED) {
      // Register a listener to reload SslContext
      try {
        fileWatchService = new FileWatchService(
          new String[] {
            TlsSystemConfig.tlsServerCertPath,
            TlsSystemConfig.tlsServerKeyPath,
            TlsSystemConfig.tlsServerTrustCertPath
          },
          new FileWatchService.Listener() {
            boolean certChanged, keyChanged = false;
            @Override
            public void onChanged(String path) {
              if (path.equals(TlsSystemConfig.tlsServerTrustCertPath)) {
                log.info("The trust certificate changed, reload the ssl context");
                reloadServerSslContext();
              }
              if (path.equals(TlsSystemConfig.tlsServerCertPath)) {
                certChanged = true;
              }
              if (path.equals(TlsSystemConfig.tlsServerKeyPath)) {
                keyChanged = true;
              }
              if (certChanged && keyChanged) {
                log.info("The certificate and private key changed, reload the ssl context");
                certChanged = keyChanged = false;
                reloadServerSslContext();
              }
            }
            private void reloadServerSslContext() {
              ((NettyRemotingServer) remotingServer).loadSslContext();
            }
          });
      } catch (Exception e) {
        log.warn("FileWatchService created error, can't load the certificate dynamically");
      }
    }
  
    return true;
  }
  ```

* start()

  ```java
  public void start() throws Exception {
    //netty服务端的启动
    this.remotingServer.start();
  
    if (this.fileWatchService != null) {
      this.fileWatchService.start();
    }
  }
  ```

### 总结

* 可以学习到的点
  * 理解启动流程
  * 开发技巧：将启动的配置做成一个对象，包括NameSrvConfig和NettyConfig，当使用的时候直接获取，先通过配置文件解析，转换成property然后赋值给config
  * 工具类
  * JVM的钩子函数，特别是线程池，当JVM停止时使用钩子函数来进行shutdown
  * 通过JDK最基本的定时任务进行Broker和KV打印的定时任务
* 问题点
  * 没有解决broker宕机之后的高可用





## NameServer路由

### 路由元数据

* 主要保存与Broker有直接关系和间接关系的数据，方便Producer和Consumer获取这些信息

* 源码

  ```java
  public class RouteInfoManager {
     
      /**
       * 此处注意点：
       *  brokerAddrTable中是不包含，心跳检测的结果的
       *  brokerLiveTable中才包含了心跳检测的结果，因为基础信息时各个broker上报的，而健康检查是NameServer去定时任务检查出来的
       * 工作：
       *  工作中，也涉及到健康检查的数据，但是因为健康检查的状态保存在了服务节点的基础信息中，导致，需要重新覆盖基础信息，那么结果就是
       *  如果健康检查是老数据，当健康检查完毕后回写数据的时候，实际上节点信息已经修改，导致修改的被心跳检测的老数据覆盖。更新无效
       */
  
      //消息队列路由信息，消息发送时更具路由表进行负载均衡
      private final HashMap<String/* topic */, List<QueueData>> topicQueueTable;
  
      //Broker基础信息，包含brokerName,所属集群名称，主备Broker地址
      private final HashMap<String/* brokerName */, BrokerData> brokerAddrTable;
  
      //Broker集群信息，存储着集群中所有Broker名称
      private final HashMap<String/* clusterName */, Set<String/* brokerName */>> clusterAddrTable;
  
      //Broker状态信息。NameServer每次收到心跳包时会替换该消息
      private final HashMap<String/* brokerAddr */, BrokerLiveInfo> brokerLiveTable;
  
      //Broker 上的 FilterServer 列表，用于类模式消息滤
      private final HashMap<String/* brokerAddr */, List<String>/* Filter Server */> filterServerTable;
    
     
      ......
    
  }  
  ```

* QueueData：Topic队列数据

  <img src="../../../img/image-20210728163242184.png" alt="image-20210728163242184" style="zoom:50%;" />

  ```java
  public class QueueData implements Comparable<QueueData> {
  
      //存储的broker名称
      private String brokerName;
  
      //读队列数
      private int readQueueNums;
  
      //写队列数
      private int writeQueueNums;
  
      //读写权限
      private int perm;
  }  
  ```

  * 一个topic有多个消息队列，这些消息队列分布在不同的broker上，一个Queuedata对应一个broker上某topic对应读写队列的结构

    <img src="../../../img/image-20210728190403645.png" alt="image-20210728190403645" style="zoom:50%;" />

* BrokerData

  <img src="../../../img/image-20210728163326955.png" alt="image-20210728163326955" style="zoom:50%;" />

  ```java
  public class BrokerData implements Comparable<BrokerData> {
      //集群
      private String cluster;
  
      //名称
      private String brokerName;
  
      //所有broker的地址信息，brokerId=0表示Master,大于0表示Slave
      private HashMap<Long/* brokerId */, String/* broker address */> brokerAddrs;
   		
    	......
  }  
  ```

* BrokerLiveInfo

  <img src="../../../img/image-20210728163439489.png" alt="image-20210728163439489" style="zoom:50%;" />

  ```java
  /**
   * Broker存活信息，值对象
   */
  class BrokerLiveInfo {
      //上次收到Broker心跳包的时间
      private long lastUpdateTimestamp;
  
      //数据版本
      private DataVersion dataVersion;
  
      //心跳检测通道，基于Netty
      private Channel channel;
  
      //消息过滤器地址
      private String haServerAddr;
     
      ......
  }  
  ```

  

### 路由注册

* 过程

  * 启动，Broker发送心跳包给NameSrv
  * 运行中
    * Broker每隔60s发送一次
    * NameSrv检测brokerLiveTable，长时间未收到心跳包，则删除信息，关闭socket

* 时序图

  ![RocketMQ--NameSrv--路由注册](../../../../VPProjects/RocketMQ--NameSrv--路由注册.jpg)

#### Broker发送心跳包

##### BrokerController

* BrokerController#start

  ```java
  this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
  
    @Override
    public void run() {
      try {
        BrokerController.this.registerBrokerAll(true, false, brokerConfig.isForceRegister());
      } catch (Throwable e) {
        log.error("registerBrokerAll Exception", e);
      }
    }
  }, 1000 * 10, Math.max(10000, Math.min(brokerConfig.getRegisterNameServerPeriod(), 60000)), TimeUnit.MILLISECONDS);
  ```



##### BrokerOuterAPI

* registerBrokerAll，每一个请求，RocketMQ都会定义一个RequestCode,然后再服务端会对应的网络处理器(processor包中

  ```java
  public List<RegisterBrokerResult> registerBrokerAll(
    final String clusterName,
    final String brokerAddr,
    final String brokerName,
    final long brokerId,
    final String haServerAddr,
    final TopicConfigSerializeWrapper topicConfigWrapper,
    final List<String> filterServerList,
    final boolean oneway,
    final int timeoutMills,
    final boolean compressed) {
  
    final List<RegisterBrokerResult> registerBrokerResultList = Lists.newArrayList();
    List<String> nameServerAddressList = this.remotingClient.getNameServerAddressList();
    if (nameServerAddressList != null && nameServerAddressList.size() > 0) {
  
      //封装请求包头(Header)
      final RegisterBrokerRequestHeader requestHeader = new RegisterBrokerRequestHeader();
      requestHeader.setBrokerAddr(brokerAddr);//broker地址
      requestHeader.setBrokerId(brokerId);//0:Master,大于0：Slave
      requestHeader.setBrokerName(brokerName);//Broker名称
      requestHeader.setClusterName(clusterName);//集群名称
      requestHeader.setHaServerAddr(haServerAddr);//master地址，
      requestHeader.setCompressed(compressed);
  
      RegisterBrokerBody requestBody = new RegisterBrokerBody();
      requestBody.setTopicConfigSerializeWrapper(topicConfigWrapper);
      requestBody.setFilterServerList(filterServerList);
      final byte[] body = requestBody.encode(compressed);
      final int bodyCrc32 = UtilAll.crc32(body);
      requestHeader.setBodyCrc32(bodyCrc32);
      final CountDownLatch countDownLatch = new CountDownLatch(nameServerAddressList.size());
      
      //向所有的NameServer发送心跳包
      for (final String namesrvAddr : nameServerAddressList) {
        brokerOuterExecutor.execute(new Runnable() {
          @Override
          public void run() {
            try {
              
              //内部为Netty的发送方式，发送的协议对象为RemotingCommand
              //每个RemotingCommand会有一个Code，用来说明此消息是什么类型，让消费端使用
              //特定的处理器来处理
              RegisterBrokerResult result = registerBroker(namesrvAddr,oneway, timeoutMills,requestHeader,body);
              if (result != null) {
                registerBrokerResultList.add(result);
              }
  
              log.info("register broker[{}]to name server {} OK", brokerId, namesrvAddr);
            } catch (Exception e) {
              log.warn("registerBroker Exception, {}", namesrvAddr, e);
            } finally {
              countDownLatch.countDown();
            }
          }
        });
      }
  
      
      //异步转同步
      try {
        countDownLatch.await(timeoutMills, TimeUnit.MILLISECONDS);
      } catch (InterruptedException e) {
      }
    }
  
    return registerBrokerResultList;
  }
  ```

  * 处理逻辑在NameServer中





#### NameServer处理心跳包

* 上面Broker已经将心跳信息发送给所有的NameServer了，那么NameServer如何处理呢？？？



##### DefaultRequestProcessor

* 网络处理器解析请求类型

  ![image-20210728184459240](../../../img/image-20210728184459240.png)

* 当有请求进来之后，查看processRequest方法

  ```java
  @Override
  public RemotingCommand processRequest(ChannelHandlerContext ctx,
                                        RemotingCommand request) throws RemotingCommandException {
  
    if (ctx != null) {
      log.debug("receive request, {} {} {}",
                request.getCode(),
                RemotingHelper.parseChannelRemoteAddr(ctx.channel()),
                request);
    }
  
  
    switch (request.getCode()) {
      case RequestCode.PUT_KV_CONFIG:
        return this.putKVConfig(ctx, request);
      case RequestCode.GET_KV_CONFIG:
        return this.getKVConfig(ctx, request);
      case RequestCode.DELETE_KV_CONFIG:
        return this.deleteKVConfig(ctx, request);
      case RequestCode.QUERY_DATA_VERSION:
        return queryBrokerTopicConfig(ctx, request);
  
        //注册broker
      case RequestCode.REGISTER_BROKER:
        Version brokerVersion = MQVersion.value2Version(request.getVersion());
        if (brokerVersion.ordinal() >= MQVersion.Version.V3_0_11.ordinal()) {
          return this.registerBrokerWithFilterServer(ctx, request);
        } else {
          return this.registerBroker(ctx, request);
        }
      case RequestCode.UNREGISTER_BROKER:
        return this.unregisterBroker(ctx, request);
  
        //路由信息是客户端，即消费者和提供者来主动发起拉取，而不是NameServer推送给他们
        //是非实时的
      case RequestCode.GET_ROUTEINTO_BY_TOPIC:
        return this.getRouteInfoByTopic(ctx, request);
      case RequestCode.GET_BROKER_CLUSTER_INFO:
        return this.getBrokerClusterInfo(ctx, request);
      case RequestCode.WIPE_WRITE_PERM_OF_BROKER:
        return this.wipeWritePermOfBroker(ctx, request);
      case RequestCode.GET_ALL_TOPIC_LIST_FROM_NAMESERVER:
        return getAllTopicListFromNameserver(ctx, request);
      case RequestCode.DELETE_TOPIC_IN_NAMESRV:
        return deleteTopicInNamesrv(ctx, request);
      case RequestCode.GET_KVLIST_BY_NAMESPACE:
        return this.getKVListByNamespace(ctx, request);
      case RequestCode.GET_TOPICS_BY_CLUSTER:
        return this.getTopicsByCluster(ctx, request);
      case RequestCode.GET_SYSTEM_TOPIC_LIST_FROM_NS:
        return this.getSystemTopicListFromNs(ctx, request);
      case RequestCode.GET_UNIT_TOPIC_LIST:
        return this.getUnitTopicList(ctx, request);
      case RequestCode.GET_HAS_UNIT_SUB_TOPIC_LIST:
        return this.getHasUnitSubTopicList(ctx, request);
      case RequestCode.GET_HAS_UNIT_SUB_UNUNIT_TOPIC_LIST:
        return this.getHasUnitSubUnUnitTopicList(ctx, request);
      case RequestCode.UPDATE_NAMESRV_CONFIG:
        return this.updateConfig(ctx, request);
      case RequestCode.GET_NAMESRV_CONFIG:
        return this.getConfig(ctx, request);
      default:
        break;
    }
    return null;
  }
  ```

* registerBroker -> routeInfoManager.registerBroker



##### RouteInfoManager

* registerBroker：注册的主要逻辑

  ```java
  public RegisterBrokerResult registerBroker(
    final String clusterName,
    final String brokerAddr,
    final String brokerName,
    final long brokerId,
    final String haServerAddr,
    final TopicConfigSerializeWrapper topicConfigWrapper,
    final List<String> filterServerList,
    final Channel channel) {
    RegisterBrokerResult result = new RegisterBrokerResult();
    try {
      try {
  
  
        //1. 加锁，防止并发修改路由表  clusterAddrTable
        this.lock.writeLock().lockInterruptibly();
  
        //1.1 首先判断Broker所属集群是否存在，如果不存在，则创建，然后将broker加入到集群中
        Set<String> brokerNames = this.clusterAddrTable.get(clusterName);
        if (null == brokerNames) {
          brokerNames = new HashSet<String>();
          this.clusterAddrTable.put(clusterName, brokerNames);
        }
        brokerNames.add(brokerName);
  
  
  
        //2. 维护BrokerData的信息
        boolean registerFirst = false;
        BrokerData brokerData = this.brokerAddrTable.get(brokerName);
        if (null == brokerData) {
          registerFirst = true;
          brokerData = new BrokerData(clusterName, brokerName, new HashMap<Long, String>());
          this.brokerAddrTable.put(brokerName, brokerData);
        }
        Map<Long, String> brokerAddrsMap = brokerData.getBrokerAddrs();
        //Switch slave to master: first remove <1, IP:PORT> in namesrv, then add <0, IP:PORT>
        //The same IP:PORT must only have one record in brokerAddrTable
        Iterator<Entry<Long, String>> it = brokerAddrsMap.entrySet().iterator();
        while (it.hasNext()) {
          Entry<Long, String> item = it.next();
          if (null != brokerAddr && brokerAddr.equals(item.getValue()) && brokerId != item.getKey()) {
            it.remove();
          }
        }
  
        String oldAddr = brokerData.getBrokerAddrs().put(brokerId, brokerAddr);
        registerFirst = registerFirst || (null == oldAddr);
  
  
        //3. 如果Broker为master，并且Broker Topic配置信息发生变化或者初次注册，则需要创建
        //   或跟新Topic路由元数据，填充topicQueueTable，其实就是为默认主题自动注册路由信息，
        //   包含MixAll.DEFAULT_TOPIC路由信息
        if (null != topicConfigWrapper
            && MixAll.MASTER_ID == brokerId) {
          if (this.isBrokerTopicConfigChanged(brokerAddr, topicConfigWrapper.getDataVersion())
              || registerFirst) {
            ConcurrentMap<String, TopicConfig> tcTable =
              topicConfigWrapper.getTopicConfigTable();
            if (tcTable != null) {
              for (Map.Entry<String, TopicConfig> entry : tcTable.entrySet()) {
                //Broker -> Queue的跟新 -> topicQueueTable
                //跟新主题，和对列信息
                this.createAndUpdateQueueData(brokerName, entry.getValue());
              }
            }
          }
        }
  
  
        // Broker -> brokerLiveInfo
        //4. 更新BrokerLiveInfo，存活Broker信息表，BrokerLiveInfo是执行路由删除的重要依据
        BrokerLiveInfo prevBrokerLiveInfo = this.brokerLiveTable.put(brokerAddr,
                                                                     new BrokerLiveInfo(
                                                                       System.currentTimeMillis(),//记录最近一次的心跳时间，到lastUpdateTimestamp
                                                                       topicConfigWrapper.getDataVersion(),
                                                                       channel,
                                                                       haServerAddr));
        if (null == prevBrokerLiveInfo) {
          log.info("new broker registered, {} HAServer: {}", brokerAddr, haServerAddr);
        }
  
        if (filterServerList != null) {
          if (filterServerList.isEmpty()) {
            this.filterServerTable.remove(brokerAddr);
          } else {
            this.filterServerTable.put(brokerAddr, filterServerList);
          }
        }
  
  
        //5. 注册Broker的过滤器Server地址列表，一个Broker上会关联多个FilterServer消息过滤服务器
        if (MixAll.MASTER_ID != brokerId) {
          String masterAddr = brokerData.getBrokerAddrs().get(MixAll.MASTER_ID);
          if (masterAddr != null) {
            BrokerLiveInfo brokerLiveInfo = this.brokerLiveTable.get(masterAddr);
            if (brokerLiveInfo != null) {
              result.setHaServerAddr(brokerLiveInfo.getHaServerAddr());
              result.setMasterAddr(masterAddr);
            }
          }
        }
      } finally {
  
        //读写锁
        this.lock.writeLock().unlock();
      }
    } catch (Exception e) {
      log.error("registerBroker Exception", e);
    }
  
    return result;
  }
  ```

  * 其中主要就是更新，NameServer中路由表的信息，下面是更新Topic与QueueData的关系

    ```java
    private void createAndUpdateQueueData(final String brokerName, final TopicConfig topicConfig) {
      QueueData queueData = new QueueData();
      queueData.setBrokerName(brokerName);
      queueData.setWriteQueueNums(topicConfig.getWriteQueueNums());
      queueData.setReadQueueNums(topicConfig.getReadQueueNums());
      queueData.setPerm(topicConfig.getPerm());
      queueData.setTopicSynFlag(topicConfig.getTopicSysFlag());
    
      List<QueueData> queueDataList = this.topicQueueTable.get(topicConfig.getTopicName());
    
      //该topic是第一次注册，新增
      if (null == queueDataList) {
        queueDataList = new LinkedList<QueueData>();
        queueDataList.add(queueData);
        this.topicQueueTable.put(topicConfig.getTopicName(), queueDataList);
        log.info("new topic registered, {} {}", topicConfig.getTopicName(), queueData);
      } else {
        boolean addNewOne = true;
    
        Iterator<QueueData> it = queueDataList.iterator();
        while (it.hasNext()) {
          QueueData qd = it.next();
          if (qd.getBrokerName().equals(brokerName)) {
            //topic对应的Queue没变，不做处理
            if (qd.equals(queueData)) {
              addNewOne = false;
            } else {
              //topic对应的queueData信息更改，先移除然后进行新增
              log.info("topic changed, {} OLD: {} NEW: {}", topicConfig.getTopicName(), qd,
                       queueData);
              it.remove();
            }
          }
        }
    
        if (addNewOne) {
          queueDataList.add(queueData);
        }
      }
    }
    ```

* 亮点
  * NameServe与Broker保持长连接
  * Broker 状态存储在 brokerLiveTable 中，NameServer 每收到一个心跳包，将更新 brokerL iveT ble 中关于 Broker 的状态信息以及路由表（ topicQueueTable brokerAddrTab le brokerLiveTabl fi lterServerTable）
  * 读写锁的使用：因为路由信息，存在着读，Producer和Consumer都使用其中的数据，但是必须保证这些路由信息的原子性，所以，使用了读写锁来提高性能。



### 路由删除

* RocketMQ有两个触发点来触发路由删除

  * NameServer定时扫描brokerLiveTable检测上次心跳包与当前系统时间的时间差，如果时间戳大于120s，则需要移除该Broker信息
  * Broker在正常被关闭的情况下，会执行unregisterBroker指令

* 第一个触发点入口

  ```java
  //3. 开启定时任务，每隔10s扫描一个Broker，移除处于不激活状态的broker
  this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
  
    @Override
    public void run() {
      NamesrvController.this.routeInfoManager.scanNotActiveBroker();
    }
  }, 5, 10, TimeUnit.SECONDS);
  ```

#### NamesrvController

##### initialize

```java
public boolean initialize() {

  ......

  //3. 开启定时任务，每隔10s扫描一个Broker，移除处于不激活状态的broker
  this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

    @Override
    public void run() {
      NamesrvController.this.routeInfoManager.scanNotActiveBroker();
    }
  }, 5, 10, TimeUnit.SECONDS);


  ....
}

```



#### RouteInfoManager

##### scanNotActiveBroker

```java
/**
     * 浏览并且清理未生效的Broker
     *
     *
     * 问题点？？？
     *
     *      1. 什么时候去更新时间呢？？？
     *        心跳注册会更新时间点
     *        
     *      2. 为何要分开操作呢？？？
     *          更新是由broker心跳方式触发
     *          删除，则是有NameServer自动去检测，这样分开好处，不同的动作交给不同的执行者执行，如果心跳响应也进行删除操作，那么就会影响响应
     *          而且删除操作，是由NameServer来进行维护的和Broker上传心跳包，进行跟新无关
     */
public void scanNotActiveBroker() {
  Iterator<Entry<String, BrokerLiveInfo>> it = this.brokerLiveTable.entrySet().iterator();
  while (it.hasNext()) {
    Entry<String, BrokerLiveInfo> next = it.next();
    long last = next.getValue().getLastUpdateTimestamp();

    //距离上次收到心跳包的事件如果超过了当前时间的120s，NameServer则认为该Broker已不可用，需要将它移除
    if ((last + BROKER_CHANNEL_EXPIRED_TIME) < System.currentTimeMillis()) {

      //关闭通道
      RemotingUtil.closeChannel(next.getValue().getChannel());
      it.remove();
      log.warn("The broker channel expired, {} {}ms", next.getKey(), BROKER_CHANNEL_EXPIRED_TIME);

      //删除涉及Broker数据的Map，brokerLiveTable  filterServer
      this.onChannelDestroy(next.getKey(), next.getValue().getChannel());
    }
  }
}
```







### 路由发现

* 路由发现是非实时的，当Topic路由出现变化后，NameServer不主动推送给客户端，而是由客户端定时拉取最新的路由

* 路由结果对象

  * TopicRouteData

    ![image-20210728200022882](../../../img/image-20210728200022882.png)

    ```java
    /**
     * 主题路由元信息
     *
     *   主题和哪些角色有关
     *        message queue
     *        broker
     *          broker filter
     */
    public class TopicRouteData extends RemotingSerializable {
    
        //顺序消息配置内容
        private String orderTopicConf;
    
        //主题所在topic对列元数据
        private List<QueueData> queueDatas;
    
        //主题所在的broker元数据
        private List<BrokerData> brokerDatas;
    
        //broker上过滤服务器地址列表，broker元信息中既然没有路由信息
        private HashMap<String/* brokerAddr */, List<String>/* Filter Server */> filterServerTable;
    
    
    }
    ```



#### DefaultRequestProcessor

* Netty请求的统一入口

##### processRequest

```java
@Override
public RemotingCommand processRequest(ChannelHandlerContext ctx,
                                      RemotingCommand request) throws RemotingCommandException {

  if (ctx != null) {
    log.debug("receive request, {} {} {}",
              request.getCode(),
              RemotingHelper.parseChannelRemoteAddr(ctx.channel()),
              request);
  }


  switch (request.getCode()) {
    case RequestCode.PUT_KV_CONFIG:
      return this.putKVConfig(ctx, request);
    case RequestCode.GET_KV_CONFIG:
      return this.getKVConfig(ctx, request);
    case RequestCode.DELETE_KV_CONFIG:
      return this.deleteKVConfig(ctx, request);
    case RequestCode.QUERY_DATA_VERSION:
      return queryBrokerTopicConfig(ctx, request);

      //注册broker
    case RequestCode.REGISTER_BROKER:
      Version brokerVersion = MQVersion.value2Version(request.getVersion());
      if (brokerVersion.ordinal() >= MQVersion.Version.V3_0_11.ordinal()) {
        return this.registerBrokerWithFilterServer(ctx, request);
      } else {
        return this.registerBroker(ctx, request);
      }
    case RequestCode.UNREGISTER_BROKER:
      return this.unregisterBroker(ctx, request);

      
      //---------------获取Topic信息------------
      //路由信息是客户端，即消费者和提供者来主动发起拉取，而不是NameServer推送给他们
      //是非实时的
    case RequestCode.GET_ROUTEINTO_BY_TOPIC:
      return this.getRouteInfoByTopic(ctx, request);
    case RequestCode.GET_BROKER_CLUSTER_INFO:
      return this.getBrokerClusterInfo(ctx, request);
    case RequestCode.WIPE_WRITE_PERM_OF_BROKER:
      return this.wipeWritePermOfBroker(ctx, request);
    case RequestCode.GET_ALL_TOPIC_LIST_FROM_NAMESERVER:
      return getAllTopicListFromNameserver(ctx, request);
    case RequestCode.DELETE_TOPIC_IN_NAMESRV:
      return deleteTopicInNamesrv(ctx, request);
    case RequestCode.GET_KVLIST_BY_NAMESPACE:
      return this.getKVListByNamespace(ctx, request);
    case RequestCode.GET_TOPICS_BY_CLUSTER:
      return this.getTopicsByCluster(ctx, request);
    case RequestCode.GET_SYSTEM_TOPIC_LIST_FROM_NS:
      return this.getSystemTopicListFromNs(ctx, request);
    case RequestCode.GET_UNIT_TOPIC_LIST:
      return this.getUnitTopicList(ctx, request);
    case RequestCode.GET_HAS_UNIT_SUB_TOPIC_LIST:
      return this.getHasUnitSubTopicList(ctx, request);
    case RequestCode.GET_HAS_UNIT_SUB_UNUNIT_TOPIC_LIST:
      return this.getHasUnitSubUnUnitTopicList(ctx, request);
    case RequestCode.UPDATE_NAMESRV_CONFIG:
      return this.updateConfig(ctx, request);
    case RequestCode.GET_NAMESRV_CONFIG:
      return this.getConfig(ctx, request);
    default:
      break;
  }
  return null;
}
```



##### getRoutelnfoByTopic

```java
public RemotingCommand getRouteInfoByTopic(ChannelHandlerContext ctx,
                                           RemotingCommand request) throws RemotingCommandException {
  final RemotingCommand response = RemotingCommand.createResponseCommand(null);
  final GetRouteInfoRequestHeader requestHeader =
    (GetRouteInfoRequestHeader) request.decodeCommandCustomHeader(GetRouteInfoRequestHeader.class);


  //1. 调用 RouterlnfoManager 的方法，从路由 topicQueueTable brokerAddrTable
  //fiterServerTable 中分别填充 TopicRouteData 中的 List<QueueData＞、 List<BrokerData＞和filterServer 地址表
  TopicRouteData topicRouteData = this.namesrvController.getRouteInfoManager().pickupTopicRouteData(requestHeader.getTopic());


  //2. 如果找到主题对应的路由信息并且该主题为顺序消息，则从 NameServer KVconfig 中获取关于顺序消息相 的配置填充路由信息
  //如果找不到路由信息 CODE 则使用 TOPIC NOT_EXISTS ，表示没有找到对应的路由
  if (topicRouteData != null) {
    if (this.namesrvController.getNamesrvConfig().isOrderMessageEnable()) {
      String orderTopicConf =
        this.namesrvController.getKvConfigManager().getKVConfig(NamesrvUtil.NAMESPACE_ORDER_TOPIC_CONFIG,
                                                                requestHeader.getTopic());
      topicRouteData.setOrderTopicConf(orderTopicConf);
    }

    byte[] content = topicRouteData.encode();
    response.setBody(content);
    response.setCode(ResponseCode.SUCCESS);
    response.setRemark(null);
    return response;
  }

  response.setCode(ResponseCode.TOPIC_NOT_EXIST);
  response.setRemark("No topic route info in name server for the topic: " + requestHeader.getTopic()
                     + FAQUrl.suggestTodo(FAQUrl.APPLY_TOPIC_URL));
  return response;
}
```



### 总结

* 总体流程图

![image-20210728200538275](../../../img/image-20210728200538275.png)



## 工具类

### MixAll

* 执行property与对象的转换

```java
public static void printObjectProperties(final InternalLogger logger, final Object object) {
  printObjectProperties(logger, object, false);
}

public static void printObjectProperties(final InternalLogger logger, final Object object,
                                         final boolean onlyImportantField) {
  Field[] fields = object.getClass().getDeclaredFields();
  for (Field field : fields) {
    if (!Modifier.isStatic(field.getModifiers())) {
      String name = field.getName();
      if (!name.startsWith("this")) {
        Object value = null;
        try {
          field.setAccessible(true);
          value = field.get(object);
          if (null == value) {
            value = "";
          }
        } catch (IllegalAccessException e) {
          log.error("Failed to obtain object properties", e);
        }

        if (onlyImportantField) {
          Annotation annotation = field.getAnnotation(ImportantField.class);
          if (null == annotation) {
            continue;
          }
        }

        if (logger != null) {
          logger.info(name + "=" + value);
        } else {
        }
      }
    }
  }
}

public static String properties2String(final Properties properties) {
  StringBuilder sb = new StringBuilder();
  for (Map.Entry<Object, Object> entry : properties.entrySet()) {
    if (entry.getValue() != null) {
      sb.append(entry.getKey().toString() + "=" + entry.getValue().toString() + "\n");
    }
  }
  return sb.toString();
}
```

