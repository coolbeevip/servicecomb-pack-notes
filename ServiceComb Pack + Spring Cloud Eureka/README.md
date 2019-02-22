# ServiceComb Pack + Spring Cloud Eureka

​	Alpha在0.4.0版本中支持将服务实例注册到发现服务Eureka中，Oemga只需要配置Eureka地址后就可以动态获取Alpha集群所有实例的gRPC地址

## 快速体验

因为在Alpha的默认发行版中不包含Eureka的集成支持，所以你需要从源代码编译一个支持的版本

1. 编译

   在编译时增加 `-Pspring-cloud-eureka,spring-boot-1` 参数，默认情况下，编译的版本兼容spring boot 2.x，当使用omega的项目是基于spring boot 1.x时请在编译的时使用`-Pspring-cloud-eureka,spring-boot-1`参数

   ```bash
   git clone https://github.com/apache/servicecomb-pack.git
   cd servicecomb-pack
   mvn clean install -DskipTests=true -Pspring-cloud-eureka,spring-boot-1
   ```

2. 启动

   启动时请添加启动参数 `--spring.profiles.active=spring-cloud-eureka`

   ```bash
   java -jar alpha-server-0.4.0-SNAPSHOT-exec.jar \
     --spring.datasource.url="jdbc:postgresql://127.0.0.1:5432/saga?useSSL=false" \
     --spring.datasource.username=saga-user \
     --spring.datasource.password=saga-password \
     --spring.profiles.active=spring-cloud-eureka \
     --eureka.client.service-url.defaultZone=http://127.0.0.1:8761/eureka
   ```

3. 验证是否注册成功

   访问Eureka的注册实例查询接口`curl http://127.0.0.1:8761/eureka/apps/`可以看到如下注册信息，在你metadata中可以看到Alpha的gRPC访问地址`<servicecomb-alpha-server>0.0.0.0:8080</servicecomb-alpha-server>`已经注册

   ```xml
   <applications>
     <versions__delta>1</versions__delta>
     <apps__hashcode>UP_1_</apps__hashcode>
     <application>
       <name>SERVICECOMB-ALPHA-SERVER</name>
       <instance>
         <instanceId>0.0.0.0::servicecomb-alpha-server:8090</instanceId>
         <hostName>0.0.0.0</hostName>
         <app>SERVICECOMB-ALPHA-SERVER</app>
         <ipAddr>0.0.0.0</ipAddr>
         <status>UP</status>
   	  ...
         <metadata>
           <management.port>8090</management.port>
           <servicecomb-alpha-server>0.0.0.0:8080</servicecomb-alpha-server>
         </metadata>
         ...
       </instance>
     </application>
   </applications>
   ```

   **注意:** 默认情况下注册的服务名是`SERVICECOMB-ALPHA-SERVER`,如果你需要自定义服务名可以在运行Alpha的时候通过命令行参数`spring.application.name`配置

4. 配置omega

   在项目中引入依赖包`omega-spring-cloud-starter` ,**注意** 0.4.0 版本中的 @EnableOmega 不在使用

   ```xml
   <dependency>
       <groupId>org.apache.servicecomb.pack</groupId>
       <artifactId>omega-spring-starter</artifactId>
       <version>0.4.0-SNAPSHOT</version>
   </dependency>
   <dependency>
   	<groupId>org.apache.servicecomb.pack</groupId>
   	<artifactId>omega-spring-cloud-starter</artifactId>
   	<version>0.4.0-SNAPSHOT</version>
   </dependency>
   ```

   在 `application.yaml` 添加下面的配置项：

   ```
   eureka:
     client:
       service-url:
         defaultZone: http://127.0.0.1:8761/eureka
   alpha:
     cluster:
       register:
         type: spring-cloud
   ```

   - `eureka.client.service-url.defaultZone` 配置Eureka注册中心的地址，其他Eureka客户端配置可以参考[Spring Cloud Netflix 2.x](https://cloud.spring.io/spring-cloud-netflix/multi/multi__service_discovery_eureka_clients.html#netflix-eureka-client-starter) 或 [Spring Cloud Netflix 1.x](https://cloud.spring.io/spring-cloud-netflix/1.4.x/multi/multi__service_discovery_eureka_clients.html#netflix-eureka-client-starter)
   - `alpha.cluster.register.type=spring-cloud` 配置Omega获取Alpha的方式是通过Eureka的注册中心

   **注意:** 如果你在启动Alpha的时候通过命令行参数`spring.application.name`自定义了服务名，那么那么你需要在Omega中通过参数`alpha.cluster.serviceId`指定这个服务名

## 源代码说明

### Alpha集成Eureka

1. 在pom.xml中增加Spring Cloud 版本定义

   ```xml
   	<spring.cloud1.version>1.4.6.RELEASE</spring.cloud1.version>
       <spring.cloud2.version>2.0.2.RELEASE</spring.cloud2.version>
       <spring.cloud.version>${spring.cloud2.version}</spring.cloud.version>
   ```

   在pom.xml中定义编译profile

   ```xml
       <profile>
         <id>spring-boot-1</id>
         <properties>
           <spring.boot.version>${spring.boot1.version}</spring.boot.version>
           <spring.cloud.version>${spring.cloud1.version}</spring.cloud.version>
         </properties>
       </profile>
       <profile>
         <id>spring-boot-2</id>
         <properties>
           <spring.boot.version>${spring.boot2.version}</spring.boot.version>
           <spring.cloud.version>${spring.cloud2.version}</spring.cloud.version>
         </properties>
       </profile>
   ```

2. 添加依赖

   在 alpha-server/pom.xm 中增加eureka的依赖包

   ```xml
   <dependency>
       <groupId>org.springframework.cloud</groupId>
       <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
       <version>${spring.cloud.version}</version>
   </dependency>
   ```

   在 alpha-server/pom.xm 中增加变异profile

   ```xml
   <profile>
   	<id>spring-cloud-eureka</id>
   	<dependencies>
   		<dependency>
   			<groupId>org.springframework.cloud</groupId>
   			<artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
   		</dependency>
   	</dependencies>
   </profile>
   ```

3. 增加Eureka相关属性配置

   application.yaml 定义  eureka 实例的meta信息，gRPC地址就是这样传递给Eureka的

   ```yaml
   ---
   spring:
     profiles: spring-cloud-eureka
   eureka:
     instance:
       metadataMap:
         servicecomb-alpha-server: ${alpha.server.host}:${alpha.server.port}
   ```

   bootstrap.yaml 增加Eureka实例信息

   ```yaml
   info:
     app:
       name: ServiceComb Alpha Server
       description: ServiceComb Alpha Server
       version: 0.4.0-SNAPSHOT
   spring:
     application:
       name: servicecomb-alpha-server
   ```

### Omeage集成Eureka

1. omega/omega-spring-cloud-starter 

   此模块为omega提供Eureka发现功能，使用 `spring.factories` 实现 `OmegaSpringEurekaConfig.java` 的自动配置。

   spring.factories

   ```java
   org.springframework.boot.autoconfigure.EnableAutoConfiguration=org.apache.servicecomb.pack.omega.spring.cloud.OmegaSpringEurekaAutoConfiguration
   ```

   `OmegaSpringEurekaConfig.java` 会创建 `AlphaClusterDiscovery.java`  实例，这个实例负责和Eureka通信，根据serviceID获取Alpha集群实例列表

   ```java
       @Bean
       AlphaClusterDiscovery alphaClusterAddress(
               @Value("${alpha.cluster.serviceId:servicecomb-alpha-server}") String serviceId,
               @Value("${alpha.cluster.address:localhost:8080}") String[] addresses) {
           StringBuffer eurekaServiceUrls = new StringBuffer();
           String[] zones = eurekaClientConfig.getAvailabilityZones(eurekaClientConfig.getRegion());
           for (String zone : zones) {
               eurekaServiceUrls.append(String.format(" [%s]:%s,", zone, eurekaClientConfig.getEurekaServerServiceUrls(zone)));
           }
           LOG.info("Eureka address{}", eurekaServiceUrls.toString());
           String[] alphaAddresses = this.getAlphaAddress(serviceId);
           if (alphaAddresses.length > 0) {
               AlphaClusterDiscovery alphaClusterDiscovery = AlphaClusterDiscovery.builder()
                       .discoveryType(AlphaClusterDiscovery.DiscoveryType.SPRING_CLOUD_EUREKA)
                       .discoveryInfo(eurekaServiceUrls.toString())
                       .addresses(alphaAddresses)
                       .build();
               return alphaClusterDiscovery;
           } else {
               AlphaClusterDiscovery alphaClusterDiscovery = AlphaClusterDiscovery.builder()
                       .discoveryType(AlphaClusterDiscovery.DiscoveryType.DEFAULT)
                       .addresses(addresses)
                       .build();
               return alphaClusterDiscovery;
           }
       }
   ```

   

2. omega/omega-spring-starter

   在 `OmegaSpringConfig.java` 的alphaClusterConfig实例化方法中使用 `alphaClusterDiscovery.getAddresses()`  获取Alpha集群地址，并传递给 `AlphaClusterConfig` 的 `addresses` 方法

   ```java
     @Bean
     AlphaClusterConfig alphaClusterConfig(
         @Value("${alpha.cluster.ssl.enable:false}") boolean enableSSL,
         @Value("${alpha.cluster.ssl.mutualAuth:false}") boolean mutualAuth,
         @Value("${alpha.cluster.ssl.cert:client.crt}") String cert,
         @Value("${alpha.cluster.ssl.key:client.pem}") String key,
         @Value("${alpha.cluster.ssl.certChain:ca.crt}") String certChain,
         @Lazy AlphaClusterDiscovery alphaClusterDiscovery,
         @Lazy MessageHandler handler,
         @Lazy TccMessageHandler tccMessageHandler) {
   
       LOG.info("Discovery alpha cluster address from {} {}",alphaClusterDiscovery.getDiscoveryType().name()
               ,alphaClusterDiscovery.getDiscoveryInfo() == null ? "" : alphaClusterDiscovery.getDiscoveryInfo());
       MessageFormat messageFormat = new KryoMessageFormat();
       AlphaClusterConfig clusterConfig = AlphaClusterConfig.builder()
           .addresses(ImmutableList.copyOf(alphaClusterDiscovery.getAddresses()))
           .enableSSL(enableSSL)
           .enableMutualAuth(mutualAuth)
           .cert(cert)
           .key(key)
           .certChain(certChain)
           .messageDeserializer(messageFormat)
           .messageSerializer(messageFormat)
           .messageHandler(handler)
           .tccMessageHandler(tccMessageHandler)
           .build();
       return clusterConfig;
     }
   ```

