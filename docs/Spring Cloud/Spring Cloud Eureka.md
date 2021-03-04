## 1.Spring Cloud Eureka 

Spring Cloud Eureka是SpringCloud Netflix微服务套件中的一部分，它基于netflix-eureka做了二次封装，主要负责微服务架构中的服务治理。

服务治理必须实现服务注册和服务发现的功能。服务注册和发现指每个微服务向注册中心汇报自己的服务地址、服务内容、端口和服务状态等信息。当有服务依赖此服务时，从注册中心获取可用的服务列表井发起请求，服务消费者不用关心服务所处的位置和状态，具体用哪个实例对外提供服则由注册中心决定。

## 2.Spring Cloud Eureka的原理

Eureka实现了服务注册和服务发现的功能。Eureka角色分为注册中心（Eureka Server）和客户端（Eureka Client)。 客户端指注册到注册中心的具体的服务实例，又可抽象分为服务提供者和服务消费者。服务提供者主要用于将自己的服务注册到服务中心供服务消费者使用，服务消费者从注册中心获取相应的服务提供者对应的服务地址并调用该服务。

![](D:\workspace\Java\images\Springcloud001.png)

在Eureka中，同一个实例可能既是服务提供者，也是服务消费者。Eureka服务端也会向另一个服务端实例注册自己的信息，从而实现Eureka Server集群。其核心概念有服务注册、服务发现、服务同步和服务续约等。

1.服务注册

在服务启动时，服务提供者会将自己的信息注测到Eureka Server, Eureka Server收到信息后，会将数据信息存储在一个双层结构的Map中，其中，第一层的Key是服务名，第二层的Key是具体服务的实例名。

```
／／注册表数据存储InstanceInfo 
private final ConcurrentHashMap<String, Map<String, Lease＜InstanceInfo>> registry= new ConcurrentHashMap<String,Map<String, Lease<InstanceInfo>>>();／／双层嵌套的HashMap
／／第一层存储的Key为AppName应用名称（同一个微服务节点可能会有多个实例）／／第二层存储的Key为InstanceName的实例名称
```

2.服务同步

在Eureka Server集群中，当一个服务提供者向其中一个Eureka Server注册服务后，该Eureka Server会向集群中的其他Eureka Server转发这个服务提供者的注信息，从而实现Eureka Server之间的服务同步。

3.服务续约

当服务提供者在注册中心完成注册后，会维护一个续约请求来持续发送信息给该Eureka Server表示其正常运行，当Eureka Server长时间收不到续约请求时，会将该服务实例从服务列表中剔除。

4.服务启动

当一个Eureka Server初始化或重启时，本地注册服务为空。Eureka Server首先调syncUp（）从别的服务节点获取所有的注册服务，然后执行Register操作，完成初始化，从而同步所有的服务信息。

5.服务下线

当服务实例正常关闭时，服务提供者会给注册中心发送一个服务下线的消息，注册中心收到信息后，会将该服务实例的状态置为下线，并把该服务下线的信息传播给其他服务实例。

6.服务发现

个服务例依赖另个服务时，这个服务实例作为服务消费者会发个信息给注册中心，请求获取注册的服务清单，注册中心会维护份只读的服务清单返回给服务消费者

7.失效剔除

注册中心为每个服务设定一个服务失效的时间。当服务提供者无法正常提供服务，而注册中心又没有收到服务下线的信息时，注册中心会创建一个定时任务，将超过一定时间而没有收到服务续约消息的服务实例从服务清单中剔除。失效时间可以通过eureka. instance. leaseExpirationDurationlnSeconds进行配置，定期扫描时间可以通过eureka.server.evictionlntervalTimerlnMs进行配

置。

## 3.Spring Cloud Eureka的使用

### 注册中心的定义

Eureka Server是服务的注册中心，服务提供者在启动时会将服务信息注册到注册中心。它维护了集群中的服务列表和状态，要实现一个注册中心分为4步：首先在pom.xml中引入spring-cloud-starter-eureka-server依赖，然后通过@EnableEurekaServer注解开启服务注册、发现功能，接下来配置appication.properties配置文件，最后一步是服务的访问和使用。

( 1 ) pomxml添加依赖

( 2）通过＠EnableEurekaServer注解开启对注册中心的支持

```
@SpringBootApplicaton
@EnableEurekaServer 
public class EurekaserverApplication{ 
	public static void main(String[]args) { 
		SpringApplication.run(EurekaserverApplication.class, args); 
	}
}
```

( 3) application.properties配置

配置EurekaServer的服务名称、端口和服务地址。在默认情况下，服务注册中心会将自己业作为客户端来尝试注册，因此，应用程序需要禁用其注册行为。具体参数为eureka.client. register-with-eureka=flase和eureka.client.fetch-registry=false。

```
spring.application.name=eureka-server 
server.port=9001 
eureka.instance.hostname=localhost 
eureka.client.register-with-eureka=false
eureka.client.fetch-registry=false
spring.main.allow-bean-definition-overriding=true
```

Eureka Server的常用配置

- eureka. instance. prefer-ip-address ：将指定的IP地址注册到Eureka Server上，如果不配置，默认为机器的主机名
- eureka.server.enable-self-preservation：设置开启或关闭自我保护
- eureka.server. renewal-percent-threshoId：设置自我保护系数，默认为0.85
- eureka.client.register-with-eureka ：设置是否将自己注册到Eureka Server，默认为true
- eureka.client.fetch-registry：设置是否从Eureka Server获取注册信息，默认为true
- eureka.server.eviction-interval-timer-in-ms ：检测服务状态间隔时间，默认为60000ms，即60s
- eureka.server. wait-time-in-ms-when-sync-empty：设置同步为空的等待时间，默认为5min，在同步等待期间，注册中心暂停向客户端提供服务的注册信息
- eureka.server.number-of-replication-retries：设置Eureka Server服务的注册信息同步失败的重试次数，默认为5次。

（4 ）访问服务地址

启动应用程序，在浏览器地址栏中输入http://localhost:9001访问注册中心.

### 服务提供者的定义

服务提供者（Service Provider）即Eureka Client，是服务的定义者。服务提供者在启动时，将自身的服务注册到注册中心，服费消费者从注册中心请求到服务列表后，便会调用具体的服务提供者的服务接口，实现远程过程调用。要实现一个服务提供者分为5步：首先在pom.xml中引入spring-cloud-starter-eureka依赖，然后通过@EnableEurekaClient注解开启服务发现的功能，接着配置application.properties配置文件，再定义服务接口，最后一步是服务的访问和使用

( 1) pom.xml添加依赖

( 2) 通过@EnabeEurekaClient注解开启服务发现的功能

```
@SpringBootApplication
@EnableEurekaClient 
public class EurekaclientApplication { 
	public static void main(String[]args) { 
		SpringApplication.run(EurekaclientApplication.class,args); 
	}
}
```

( 3 ) application.properties配置

配置Eureka Server的服务名称、端口和注册中心的地址

```
spring.application.name=eureka-client 
server.port=9002 eureka.client.serviceUrl.defaultZone=http://localhost:9001/eureka/ 
```

Eureka客户端配置：

- service-uiI ：配置服务注册中心的地址，如果服务注册中心为高可用集册，则多个注册中心的地址用逗号分割。如果服务注册中心加入了安全验证，则在IP地址前加上域名密码：http://<username>:<password>@localhost: 8761 /eureka 
- fetch-registry ：是否从Eureka Server获取注册信息，默认为true
- register-with-eureka：是否将自身的实例信息注册到Eureka Server，默认为true
- eureka-connection-idIe-timeout-seconds：Eureka Server连接空闲的关闭时间，单位：s，默认为30
- eureka-server-connect-timeout-seconds：连接Eureka Server的超时时间，单位：s，默认为5
- eureka-server-read-timeout-seconds：读取Eureka Server信息的超时时间，单位：s，默认为8
- eureka-server-total-connections：从Eureka Client到所有Eureka Server的连接总数，默认为200
- eureka-server-total-connections-per-host：从Eureka Client到每个Eureka Server的连接总数，默认为50
- eureka-service-url-poll-interval-seconds：轮询Eureka Server地址更改的间隔时间，单位：s，默认为300
- initial-instance-info-replication-interval-seconds：初始化实例信息到Eureka Server的间隔时间，单位：s，默认为40
- instance-info-replication-interval-seconds：更新实例信息的变化到Eureka Server的间隔时间，单位：s，默认为30
- registry-fetch-interval-seconds ：从Eureka Server获取注册信息的间隔时间，单位：s，默认为30

(4 ）定义服务

```
@RestController
public class DiscoveryController { 
	@Autowired 
	private DiscoveryClient discoveryClient;
    @Value (”${server.port}”) 
    private String port;
    @GetMapping(”/serviceProducer")
    public Map serviceProducer() { 
		//1服务提供者的信息
		String services =”Services:”+ discoveryClient.getServices() +”port :”+port; 
		//2：服务提者返回的数据
		Map result ＝new HashMap(); 
		result. put (”serviceProducer”,services); 
		result. put (”time ”，System.currentTimeM11lis()); 
		return result;
	}
}
```

上述代码定义了一个名为serviceProducer的服务，服务地址为“／serviceProducer"，其中，通过DiscoveryClient获取服务列表中可用的服务地址。

(5） 调用服务

在浏览器地址输入http://localhost:9002/serviceProducer访问服务

### 服务消费者的定义

服务消费者（Service Consumer）即Eureka Consumer，是服务的具体使用者，一个服务实例常常即是服务消费者也是服务提供者。要实现一个服务消费者分为5步：首先在pom.xm中引人spring-cloud-starter-eureka依赖，然后通过@EnabeEurekaClient注解开启服务发现的功能，如果通过Feign调用，则需要通过@EnabeFeignClients开启对Feign的支持，接下来配置application. properties配置文件，最后调用服务提供者暴露出来的服务接口。

( 1) pom.xml添加依赖

( 2 )通过@EnabeEurekaClient注解开启服务发现的功能，如果通过Feign远程调用，则需要通过@EnableFeignClients开启对Feign的支持；如果通过RestTemplate调用，则需要定义RestTemplate实例

( 3 ) application.properties配置

配置服务名称、端口和注册中心的地址

( 4)调用RestTemplate服务

```
@RestController 
@Autowired 
private RestTemplate restTemplate; 
@RequestMapping(value =”/consume/remote ”,method = RequestMethod.GET) 
public String service() {
	return restTemplate.getForEntity(”http://EUREKACLIENT/serviceProducer”,String.class) .getBody(); 
```

上述代码定义了一个名为service的服务，服务地址为“/consume/remote”，通过RestTemplate调用远程服务。其中，“/EUREKA-CLIENT”为服务提供者的实例名称，serviceProducer为服务＠RequestMapping映射后的地址。

( 5 ）调用服务

在浏览器地址栏中输入http://localhost: 9003 /consume/remote访问服务

( 6）定义基于Feign接口的服务

基于Feign接口的服务调用简单方便，上述代码在EurekaconsumerApplication中已经通过加入＠EnableFeignClients开启了对Feign的支持，这里只需要应用程序定义Feign接口便可。

定义Feign接口分为2步：首先通过@FeignClient（“serviceProducerName”）定义需要代理的服务实例名（“EUREKA-CLIENT”即服务提供者的实例名称），然后定义一个调用远程服务的方法。方法中@RequestMapping的value（这里为“／serviceProducer"）需要与服务提供者的地址相对应，这样，Feign才能知道代理需要具体调用服务提供者的哪个方法

```
/** 
 * @FeignClient用于通知Feign组件对该接口进行代理（不需要编写接口实现），使用者可直接通过＠Autowired注入
 *＠RequestMapping表示在调用该方法需要向／serviceProducer发送GET请求
 */ 
@FeignClient("EUREKA-CLIENT”) 
public interface Servers { 
	@RequestMapping(value="/serviceProducer”，method= RequestMethod.GET) 
	Map serviceProducer();
}
```

(7）调用基于Feign接口的服务

```
@Autowired Servers server; 
@RquestMapping(value=”/consume/feign”,method = RequestMethod.GET} 
public Map serviceFeign() { 
	return server.serviceProducer();
}
```

上述代码通过注入Servers并调用其serviceProducer（）方法来实现服务的调用。在浏览器地址中输入http://Iocalhost:9003/consume/feign查看服务结果