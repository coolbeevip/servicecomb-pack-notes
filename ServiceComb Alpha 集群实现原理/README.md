# ServiceComb Alpha 集群动态主节点实现

​	Alpha可以通过扩展部署节点个数实现高可用集群部署，并且通过在节点上设置启动参数 `alpha.event.scanner.enabled=false` 关闭某些节点的后台定时执行的事务扫描，避免多个节点同时扫描事物表可能导致的性能问题。但是当启动事务扫描的进程宕机后会导致没有进程进行事务扫描，从而导致事务补偿失败。

​	在0.4.0版本中，我们通过一种基于数据库表的抢占锁机制，实现Alpha集群中主节点的动态选举，并让事务扫描方法只在主节点上执行。当主节点宕机后其他节点通过抢占的方式选出一个新的主节点，本文将介绍在0.4.0版本相关的代码实现。

## 快速体验

在0.4.0版本中开启动态管理节点支持只需要增加启动参数 `alpha.cluster.master.enabled=true` ，例如：

1. 分别启动两个节点

```
java -jar alpha-server-0.4.0-SNAPSHOT-exec.jar \
  --server.port=8090 \
  --alpha.server.port=8080 \
  --spring.datasource.url="jdbc:postgresql://127.0.0.1:5432/saga?useSSL=false" \
  --spring.datasource.username=saga-user \
  --spring.datasource.password=saga-password \
  --alpha.cluster.master.enabled=true
```

```
java -jar alpha-server-0.4.0-SNAPSHOT-exec.jar \
  --server.port=8091 \
  --alpha.server.port=8081 \
  --spring.datasource.url="jdbc:postgresql://127.0.0.1:5432/saga?useSSL=false" \
  --spring.datasource.username=saga-user \
  --spring.datasource.password=saga-password \
  --alpha.cluster.master.enabled=true
```

2. 启动后可以在日志中看到节点类型信息

在日志中看到 `Master Node` 则表示这个进程是主节点

```
01:31:07.032 [pool-3-thread-1] INFO  org.apache.servicecomb.pack.alpha.server.cluster.master.ClusterLockService - Master Node
```

在日志中看到 `Slave Node` 则表示这个进程是从节点

```
01:31:31.059 [pool-3-thread-1] INFO  org.apache.servicecomb.pack.alpha.server.cluster.master.ClusterLockService - Slave Node
```

3. 节点切换

当主节点进程宕后2秒钟，其他从节点会采用抢占的方式使自己成为主节点。

## 让事务扫描运行在主节点

事务扫描是通过 `EventScanner.java` 实现的，并且在 `AlphaConfig.java` 中进行初始化，可以看到在 `new EventScanner` 代码执行前前进行了eventScannerEnabled判断，这个参数就是通过``alpha.event.scanner.enabled` 指定的（默认是true），然后传入了`nodeStatus` 对象，这个对象就记录着这个节点的状态（主节点或者从节点），后边会讲解 `nodeStatus` 是如何构造的。

```java
  @Bean
  TxConsistentService txConsistentService(
      @Value("${alpha.event.pollingInterval:500}") int eventPollingInterval,
      @Value("${alpha.event.scanner.enabled:true}") boolean eventScannerEnabled,
      ScheduledExecutorService scheduler,
      TxEventRepository eventRepository,
      CommandRepository commandRepository,
      TxTimeoutRepository timeoutRepository,
      OmegaCallback omegaCallback) {
        if (eventScannerEnabled) {
          new EventScanner(scheduler,
              eventRepository, commandRepository, timeoutRepository,
              omegaCallback, eventPollingInterval, nodeStatus).run();
          LOG.info("Starting the EventScanner.");
          }
        TxConsistentService consistentService = new TxConsistentService(eventRepository);
        return consistentService;
  }
```

`EventScanner.java` 的 pollEvents 方法进行定时事务扫描，并使用 `nodeStatus.isMaster()` 判断自己是否是主节点，只有主节点才允许执行。看到这里大家应该知道 `nodeStatus.isMaster()` 是我们判断主节点的关键对象，那么 `NodeStatus.java` 是如何被创建并初始化的呢

```java
  private void pollEvents() {
    scheduler.scheduleWithFixedDelay(
        () -> {
          // only pull the events when working in the master mode
          if(nodeStatus.isMaster()){
            updateTimeoutStatus();
            findTimeoutEvents();
            abortTimeoutEvents();
            saveUncompensatedEventsToCommands();
            compensate();
            updateCompensatedCommands();
            deleteDuplicateSagaEndedEvents();
            updateTransactionStatus();
          }
        },
        0,
        eventPollingInterval,
        MILLISECONDS);
  }
```

我们在 `AlphaConfig.java`  中通过以下方式声明，以确保无论您是否指定了 `alpha.cluster.master.enabled` 参数事务扫描都可以正常工作，在这里可以看到当我们开启了集群模式后节点刚启动的时候状态是Slave，下面会说明状态是如何切换到Master的。

```java
  @Bean
  NodeStatus nodeStatus (){
    if(masterEnabled){
      return new NodeStatus(NodeStatus.TypeEnum.SLAVE);
    }else{
      return new NodeStatus(NodeStatus.TypeEnum.MASTER);
    }
  }

  @Autowired
  NodeStatus nodeStatus;
```

控制节点状态切换的是 `ClusterLockService.java` ,这个服务会定时执行锁抢占，抢占成功后设置本节点为Master，否则为Slave

```java
  @Autowired
  LockProvider lockProvider;
  ...
  ...
  @Scheduled(cron = "0/1 * * * * ?")
  public void masterCheck() {
    if (applicationReady) {
      this.locker = lockProvider.lock(this.getMasterLock());
      if (this.locker.isPresent()) {
        if (!this.locked) {
          this.locked = true;
          nodeStatus.setTypeEnum(NodeStatus.TypeEnum.MASTER);
          LOG.info("Master Node");
        }
        //Keep locked
      } else {
        if (this.locked || !lockExecuted) {
          locked = false;
          nodeStatus.setTypeEnum(NodeStatus.TypeEnum.SLAVE);
          LOG.info("Slave Node");
        }
      }
      lockExecuted = true;
    }
  }
```

## 加锁服务基础类

在前边的说明中可以看到，在 `ClusterLockService.java` 的 `masterCheck` 方法中通过 `this.locker = lockProvider.lock(this.getMasterLock());` 获取一个锁并判断是否锁成功。

`LockProvider.java` 是一个接口，目前我们提供了基于jdbc的实现，包结构以及类依赖关系如下：

![image-20190222113431299](/Volumes/MyWallet/Xunlei/work/servicecomb-pack-notes/ServiceComb Alpha 集群实现原理/assets/image-20190222113431299.png)



依赖关系如下

![image-20190222114741514](/Volumes/MyWallet/Xunlei/work/servicecomb-pack-notes/ServiceComb Alpha 集群实现原理/assets/image-20190222114741514.png)

* LockProvider.java

  接口定义了锁方法 lock

* LockProviderPersistence.java 

   接口定义了以下三个方法，作为持久化锁的接口

  - initLock 创建锁，尝试进行锁定并返回锁定是否成功
  - updateLock 更新锁，进行再次锁定并返回是否成功（更新锁的接口设计的目的是为了非长连接锁设计，例如对于按照固定周期进行加锁的实现）
  - unLock 解锁，取消锁定

* AbstractLockProvider.java 

  抽象类实现了 `LockProvider.java` 接口的lock方法，并调用内部的 ``LockProviderPersistence` 接口进行锁操作  

## 加锁服务 JDBC 实现

> 我喜欢和你们一起跑步，一致的步伐，一样的心跳，这感觉真好，当你掉队了，我带着大家跑

1. JDBC 锁类关系图如下：

![image-20190222122056459](/Volumes/MyWallet/Xunlei/work/servicecomb-pack-notes/ServiceComb Alpha 集群实现原理/assets/image-20190222122056459.png)

* JdbcLockPersistence.java

   `LockProviderPersistence.java` 接口实现，用来实现对数据库表操作

* JdbcLockProvider.java

  继承抽象类 `AbstractLockProvider.java` ，在构造函数中传入 ``LockProviderPersistence.java` 的接库实现  `JdbcLockPersistence`

* LockProviderJdbcConfiguration.java

  锁的JDBC实例构造类

* JPA操作相关类

  通过JPA操作master_lock表的相关功能定义在以下几个类

  MasterLockEntityRepository.java、SpringMasterLockRepository.java、org.apache.servicecomb.pack.alpha.server.cluster.master.provider.jdbc.jpa.*

2. 锁定表结构设计如下

```sql
CREATE TABLE IF NOT EXISTS master_lock (
  serviceName varchar(36) not NULL,
  expireTime timestamp(3) not NULL,
  lockedTime timestamp(3) not NULL,
  instanceId  varchar(255) not NULL,
  PRIMARY KEY (serviceName)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

- serviceName 服务名，这个字段取值  `${spring.application.name}`
- expireTime 锁定过期时间，这个字段取值 lockedTime+5s
- lockedTime 最近一次锁定时间
- instanceId 集群实例ID，这个字段取值 `${alpha.server.host}]:${alpha.server.port}`

## 加锁过程

加锁/更新锁的过程是一个周期性重复执行的动作，步骤如下：

1.  `ClusterLockService.java` 服务中会每秒调用一次 `LockProvider.java` 接口的lock方法

2. `AbstractLockProvider.java` 抽象类中的 lock方法会尝试调用 `JdbcLockPersistence.java` 的 iniLock方法进行加锁，加锁的SQL实现定义在 `MasterLockEntityRepository.java` 中

   * 如果表为空，那么插入一条记录并返回加锁成功
   * 如果表中存在serviceName字段相同的记录，则捕获异常加锁失败

   ```java
     @Transactional
     @Modifying
     @Query(value = "INSERT INTO master_lock "
         + "(serviceName, expireTime, lockedTime, instanceId) "
         + "VALUES "
         + "(?1, ?2, ?3, ?4)", nativeQuery = true)
     int initLock(
         @Param("serviceName") String serviceName,
         @Param("expireTime") Date expireTime,
         @Param("lockedTime") Date lockedTime,
         @Param("instanceId") String instanceId);
   ```

3. 如果 initLock加锁失败，则尝试调用 `MasterLockEntityRepository.java` 中的 updateLock 方法尝试更新锁

   * 表中存在的本服务记录 instanceId 与本实例 instanceId 相同，则更新成功并返回加锁成功（这个表明上一个更新周期也是本服务更新的）
   * 表中存在的本服务记录 expireTime 小于当前锁定时间 ，则更新成功并返回加锁成功（表示上一个锁定周期并没有实例进行锁定更新操作）

   ```java
     @Transactional
     @Modifying(clearAutomatically = true)
     @Query("UPDATE org.apache.servicecomb.pack.alpha.server.cluster.master.provider.jdbc.jpa.MasterLock t "
         + "SET t.expireTime = :expireTime"
         + ",t.lockedTime = :lockedTime "
         + ",t.instanceId = :instanceId "
         + "WHERE t.serviceName = :serviceName AND (t.expireTime <= :lockedTime OR t.instanceId = :instanceId)")
     int updateLock(
         @Param("serviceName") String serviceName,
         @Param("lockedTime") Date lockedTime,
         @Param("expireTime") Date expireTime,
         @Param("instanceId") String instanceId);
   ```

4. 释放锁 unLock

   系统中并没有进行释放锁的调用，因为没有在JDBC层面不会导致事务锁定，对于扩展实现它方式的锁实现预留接口

## 抢占锁的其他实现

分布式锁可以通过扩展 `LockProvider.java` 和 `LockProviderPersistence.java` 接口提供多种实现，例如zookeeper，redis等

## 注意

* 基于数据库表的方式需要集群中多个节点的服务器时钟同步

* 基于mysql数据库时需要在JDBC连接地址中增加时区配置，`serverTimezone=GMT%2b8`

* 关于脑裂问题

  由于网络、进程假死、中间件可能导致的脑裂问题在一定程度上不可避免，我们需要做一种权衡，即脑裂发生时是否可以避免问题进一步扩大，基于JDBC的分布式锁住要避免多个节点同时对事务表进行扫描，两者都是进行数据库操作，因为它们都基于数据库所以可以在一定程度上避免脑裂导致的集群处理问题。但是基于zookeeper,redis的方式可能会导致真实的脑裂发生，如果有什么更好的方法请告诉我。

