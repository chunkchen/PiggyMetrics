[![Build Status](https://travis-ci.org/sqshq/PiggyMetrics.svg?branch=master)](https://travis-ci.org/sqshq/PiggyMetrics)
[![codecov.io](https://codecov.io/github/sqshq/PiggyMetrics/coverage.svg?branch=master)](https://codecov.io/github/sqshq/PiggyMetrics?branch=master)
[![GitHub license](https://img.shields.io/github/license/mashape/apistatus.svg)](https://github.com/sqshq/PiggyMetrics/blob/master/LICENCE)
[![Join the chat at https://gitter.im/sqshq/PiggyMetrics](https://badges.gitter.im/sqshq/PiggyMetrics.svg)](https://gitter.im/sqshq/PiggyMetrics?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

# Piggy Metrics

**一种管理个人财务状况的简单方法** 

这是一个 概念验证型（[proof-of-concept](http://my-piggymetrics.rhcloud.com)）应用，通过使用Spring Boot, Spring Cloud 和 Docker，用简洁的用户界面对微服务体系模式（[Microservice Architecture Pattern](http://martinfowler.com/microservices/)）进行论证和呈现。


![](https://cloud.githubusercontent.com/assets/6069066/13864234/442d6faa-ecb9-11e5-9929-34a9539acde0.png)
![Piggy Metrics](https://cloud.githubusercontent.com/assets/6069066/13830155/572e7552-ebe4-11e5-918f-637a49dff9a2.gif)

## 功能服务

PiggyMetrics可以被分解成三个核心的微服务。这三个微服务是独立部署的应用，围绕特定的业务能力进行组织。

<img width="880" alt="Functional services" src="https://cloud.githubusercontent.com/assets/6069066/13900465/730f2922-ee20-11e5-8df0-e7b51c668847.png">

#### 帐户服务
包括常规的用户输入逻辑和校验：个人收益/开支明细、存款和账户设置。

Method	| Path	| Description	| User authenticated	| Available from UI
------------- | ------------------------- | ------------- |:-------------:|:----------------:|
GET	| /accounts/{account}	| 获取指定账户数据	|  | 	
GET	| /accounts/current	| 获取当前账户数据	| × | ×
GET	| /accounts/demo	| 获取演示账户数据（例如预先输入的收益/开支明细）	|   | 	×
PUT	| /accounts/current	| 保存当前账户数据	| × | ×
POST	| /accounts/	| 注册新账户	|   | ×


#### 统计服务
在主要的统计参数和获取时间序列方面为每个账户执行统计操作。数据点包括标准化后的基础货币、时间序列等值。这些数据可用于在账户生命周期内动态的跟踪现金流。（用户界面中暂未实现绚丽的图标分析）

Method	| Path	| Description	| User authenticated	| Available from UI
------------- | ------------------------- | ------------- |:-------------:|:----------------:|
GET	| /statistics/{account}	| 获取指定账户统计数据 |  | 	
GET	| /statistics/current	| 获取当前账户统计数据	| × | × 
GET	| /statistics/demo	| 获取演示账户统计数据	|   | × 
PUT	| /statistics/{account}	| 创建或更新指定账户的具体时间元数据	|   | 


#### 通知服务
存储用户联系信息和通知设置（比如提醒周期和备份周期），根据预定的计划从其他微服务收集必要的信息并发送给订阅客户。

Method	| Path	| Description	| User authenticated	| Available from UI
------------- | ------------------------- | ------------- |:-------------:|:----------------:|
GET	| /notifications/settings/current	| 获取当前账户通知设置信息	| × | ×	
PUT	| /notifications/settings/current	| 保存当前通知设置信息	| × | ×

#### Notes
- 每个微服务都有自己的数据库，不使用API将无法直接访问持久化数据。
- 本工程中，我使用MongoDB 作为每个微服务的主数据库。也许支持多种持久性结构显得更有意义（可选择数据库的类型是服务最适合的方式）。
- 服务之间的通讯非常简单：微服务只使用同步的REST API进行通信。现实情况中，常见的做法是结合使用不同的交互方式。例如：为了解耦服务和缓存消息，GET操作中一般使用同步的方式请求和获取数据；而在创建和更新操作中则通过消息代理实现异步处理。这些方法将达成最终一致（[eventual consistency](http://martinfowler.com/articles/microservice-trade-offs.html#consistency)）的目标。


## 基础设施服务
分布式系统中存在大量的公共模式，它们可以帮助我们专注于核心的服务实现。[Spring cloud](http://projects.spring.io/spring-cloud/)提供了一系列强有力的工具增强了Spring Boot应用，使得上述公共模式得以实现。我将简要介绍它们。

<img width="880" alt="Infrastructure services" src="https://cloud.githubusercontent.com/assets/6069066/13906840/365c0d94-eefa-11e5-90ad-9d74804ca412.png">
### Config service
[Spring Cloud Config](http://cloud.spring.io/spring-cloud-config/spring-cloud-config.html) 是为分布式系统提供的水平可伸缩的集中配置服务。它使用一个可插入的仓库层,目前支持本地存储,Git和Subversion。

本项目中,我使用可轻松加载到本地classpath中的 `native profile`。你可以在 [Config service resources](https://github.com/sqshq/PiggyMetrics/tree/master/config/src/main/resources)中看到共享目录。当通知服务发起访问自身配置的请求时, Config service以`shared/notification-service.yml` 和`shared/application.yml`文件的方式进行响应（在所有客户端程序中共享）。


##### 客户端使用
通过`spring-cloud-starter-config`依赖构建 Spring Boot应用，剩下的部分将会自动构建。

在`bootstrap.yml`中定义好应用的名称和Config service 的url后，你的应用中将不再需要任何内置的配置文件。

```yml
spring:
  application:
    name: notification-service
  cloud:
    config:
      uri: http://config:8888
      fail-fast: true
```

##### 使用Spring Cloud Config，您可以动态更改应用配置。 
例如，[EmailService bean](https://github.com/sqshq/PiggyMetrics/blob/master/notification-service/src/main/java/com/piggymetrics/notification/service/EmailServiceImpl.java)用`@RefreshScope`注释，这意味着您可以在不重新编译和重启通知服务应用的情况下，更改电子邮件文本和主题。


首先，在Config服务器中更改所需的属性。然后，对Notification服务执行刷新请求:
`curl -H "Authorization: Bearer #token#" -XPOST http://127.0.0.1:8000/notifications/refresh`

此外，您可以使用webhooks库自动执行此过程 [webhooks to automate this process](http://cloud.spring.io/spring-cloud-config/spring-cloud-config.html#_push_notifications_and_spring_cloud_bus)

##### 说明
- 动态刷新仍然有一些限制。注解 `@RefreshScope` 不能与`@Configuration`在类中一起使用，并且在 `@Scheduled`注解的方法中不起作用。
- 如果它无法连接到Config Service，fail-fast属性意味着Spring Boot应用程序将立即失败启动。这在[启动所有应用程序](https://github.com/sqshq/PiggyMetrics#how-to-run-all-the-things)时非常有用。 
- 下面是一些重要的[安全说明](https://github.com/sqshq/PiggyMetrics#security)


### 授权服务
授权责任完全抽取到单独的服务器，它为后端资源服务授予[OAuth2 tokens](https://tools.ietf.org/html/rfc6749)令牌。授权服务器在边界内为用户授权提供安全的机对机通讯。

本项目中，我使用密码凭据类型（[`Password credentials`](https://tools.ietf.org/html/rfc6749#section-4.3) ）实现用户授权（因为它仅由本地PiggyMetrics UI使用），使用客户端凭证（[`Client Credentials`](https://tools.ietf.org/html/rfc6749#section-4.4)）实现微服务授权。

Spring Cloud Security提供了方便的注释和自动配置，使得从服务器端和客户端都很容易实现。您可以在[文档](http://cloud.spring.io/spring-cloud-security/spring-cloud-security.html) 中了解更多信息，并在[Auth Server代码](https://github.com/sqshq/PiggyMetrics/tree/master/auth-service/src/main/java/com/piggymetrics/auth)中检查配置详细信息。

就客户端而言，一切工作与传统的基于会话的授权相同。您可以从请求中检索`Principal`对象，通过基于表达式的访问控制和`@PreAuthorize`注解检查用户的角色和其他内容。


PiggyMetrics中的每个客户端（帐户服务，统计服务，通知服务和浏览器）都有一个范围：服务器用于后端服务，ui  - 用于浏览器。因此，我们还可以保护控制器免受外部访问，例如：

``` java
@PreAuthorize("#oauth2.hasScope('server')")
@RequestMapping(value = "accounts/{name}", method = RequestMethod.GET)
public List<DataPoint> getStatisticsByAccountName(@PathVariable String name) {
	return statisticsService.findByAccountName(name);
}
```

### API网关
可以看到，有三个核心服务向客户端公开外部API。现实系统中，这个数字随着系统的复杂性增加将会快速增长。实际上，可能数百种服务都会参与渲染某个复杂网页。

理论上，客户端可以直接向每个微服务器发出请求。但很显然，在类似需要知道所有端点地址，分别执行每一个分离的http请求，并且在客户端合并结果数据方面，这种方式存在相当的挑战和限制。另一个问题是网络友好协议可能在后端使用。

更好的方法是使用API网关。它是进入系统的单一入口，通过路由将请求分发到恰当的后端服务或直接调用多个后端服务进行处理，[并将结果进行聚合](http://techblog.netflix.com/2013/01/optimizing-netflix-api.html)。此外，它可用于身份验证，insights，压力测试和canary 测试，服务迁移，静态响应处理，主动流量管理等。

Netflix开源了一个[边界服务](http://techblog.netflix.com/2013/06/announcing-zuul-edge-service-in-cloud.html)，在Spring Cloud中我们可以用一个@EnableZuulProxy 注解使用它。在这个项目中，我使用Zuul来存储静态内容（ui应用程序）并且将请求路由到适当的微服务。以下是一个通知服务中基于前缀的简单路由配置：


```yml
zuul:
  routes:
    notification-service:
        path: /notifications/**
        serviceId: notification-service
        stripPrefix: false

```

这意味着所有以/ notifications开头的请求都将路由到通知服务。你可以看到这里并没有硬编码的地址。Zuul使用服务发现机制( [Service discovery](https://github.com/sqshq/PiggyMetrics/blob/master/README.md#service-discovery))来定位通知服务实例、断路器以及负载平衡器，以下将进行阐述。


### 服务发现
另一个公知的架构模式是服务发现。它允许自动检测服务实例的网络位置，由于自动扩展，故障和升级，可能会动态分配地址。

服务发现的关键部分是注册。我在这个项目中使用Netflix Eureka。当客户端负责决定可用的服务实例（使用注册表服务器）的位置和加载负载均衡请求时，Eureka是客户端发现模式的一个很好的例子。

Spring Boot中，您可以轻松地使用spring-cloud-starter-eureka-server依赖，@EnableEurekaServer注解和简单的配置属性来构建Eureka 注册。

客户端支持在bootstrap.yml中使用@EnableDiscoveryClient注解，以应用名识别：

``` yml
spring:
  application:
    name: notification-service
```

现在，在应用程序启动时，它将注册到Eureka服务器并提供元数据，如主机和端口，健康指示器URL，主页等。Eureka从属于一个服务的每个实例接收心跳消息。如果在可配置的时间表发生心跳故障，则实例将从注册表中删除。

此外，Eureka提供了一个简单的界面，您可以跟踪运行的服务和可用实例数：

`http://localhost:8761`

### 负载均衡器，断路器和Http客户端

Netflix OSS提供了一系列强大的工具集。 

#### Ribbon
Ribbon是一个可以对HTTP和TCP客户端的行为进行控制客户端负载均衡器。与传统的负载均衡器相比，Ribbon无需对每个线上调用都处理心跳，您可以直接访问所需的服务。

它与Spring Cloud和服务发现本身集成，因此可开箱即用。 [Eureka Client](https://github.com/sqshq/PiggyMetrics#service-discovery)提供了可用服务器的动态列表，以便Ribbon可以在它们之间进行平衡。


#### Hystrix
Hystrix是断路器模式（[Circuit Breaker pattern](http://martinfowler.com/bliki/CircuitBreaker.html)）的实现，它提供了对通过网络访问的依赖性的延迟和故障的控制。主要实现思路是在具有大量微服务的分布式环境中停止级联故障。在容错系统的自我治愈方面，它帮助错误尽可能的快速发现并且恢复。

除了断路器控制，使用Hystrix，您可以添加一个后备方法，在主命令失败的情况下调用该方法以获取默认值。

此外，Hystrix生成每个命令的执行结果和延迟的指标，我们可以用它来监视系统行为（[monitor system behavior](https://github.com/sqshq/PiggyMetrics#monitor-dashboard)）。


#### Feign

Feign是一个与Ribbon和Hystrix无缝集成的声明式Http客户端。实际上，使用一个`spring-cloud-starter-feign`依赖和`@EnableFeignClients`注解，您拥有一组完整的负载均衡器，断路器和Http客户端以及合理的即用型默认配置。
以下是帐户服务的示例：


你需要的只是一个接口： 

``` java
@FeignClient(name = "statistics-service")
public interface StatisticsServiceClient {

	@RequestMapping(method = RequestMethod.PUT, value = "/statistics/{accountName}", consumes = MediaType.APPLICATION_JSON_UTF8_VALUE)
	void updateStatistics(@PathVariable("accountName") String accountName, Account account);

}
```

- 你需要的只是一个接口
- 你可以在Spring MVC控制器和Feign方法之间共享`@RequestMapping` 
- 通过Eureka自动发现，上面的示例指定只需要服务id  - 统计服务，（显然，您可以访问任何资源与特定的URL）

### Monitor dashboard

In this project configuration, each microservice with Hystrix on board pushes metrics to Turbine via Spring Cloud Bus (with AMQP broker). The Monitoring project is just a small Spring boot application with [Turbine](https://github.com/Netflix/Turbine) and [Hystrix Dashboard](https://github.com/Netflix/Hystrix/tree/master/hystrix-dashboard).

See below [how to get it up and running](https://github.com/sqshq/PiggyMetrics#how-to-run-all-the-things).

Let's see our system behavior under load: Account service calls Statistics service and it responses with a vary imitation delay. Response timeout threshold is set to 1 second.

<img width="880" src="https://cloud.githubusercontent.com/assets/6069066/14194375/d9a2dd80-f7be-11e5-8bcc-9a2fce753cfe.png">

<img width="212" src="https://cloud.githubusercontent.com/assets/6069066/14127349/21e90026-f628-11e5-83f1-60108cb33490.gif">	| <img width="212" src="https://cloud.githubusercontent.com/assets/6069066/14127348/21e6ed40-f628-11e5-9fa4-ed527bf35129.gif"> | <img width="212" src="https://cloud.githubusercontent.com/assets/6069066/14127346/21b9aaa6-f628-11e5-9bba-aaccab60fd69.gif"> | <img width="212" src="https://cloud.githubusercontent.com/assets/6069066/14127350/21eafe1c-f628-11e5-8ccd-a6b6873c046a.gif">
--- |--- |--- |--- |
| `0 ms delay` | `500 ms delay` | `800 ms delay` | `1100 ms delay`
| Well behaving system. The throughput is about 22 requests/second. Small number of active threads in Statistics service. The median service time is about 50 ms. | The number of active threads is growing. We can see purple number of thread-pool rejections and therefore about 30-40% of errors, but circuit is still closed. | Half-open state: the ratio of failed commands is more than 50%, the circuit breaker kicks in. After sleep window amount of time, the next request is let through. | 100 percent of the requests fail. The circuit is now permanently open. Retry after sleep time won't close circuit again, because the single request is too slow.

### Log analysis

Centralized logging can be very useful when attempting to identify problems in a distributed environment. Elasticsearch, Logstash and Kibana stack lets you search and analyze your logs, utilization and network activity data with ease.
Ready-to-go Docker configuration described [in my other project](http://github.com/sqshq/ELK-docker).

## Security

An advanced security configuration is beyond the scope of this proof-of-concept project. For a more realistic simulation of a real system, consider to use https, JCE keystore to encrypt Microservices passwords and Config server properties content (see [documentation](http://cloud.spring.io/spring-cloud-config/spring-cloud-config.html#_security) for details).

## Infrastructure automation

Deploying microservices, with their interdependence, is much more complex process than deploying monolithic application. It is important to have fully automated infrastructure. We can achieve following benefits with Continuous Delivery approach:

- The ability to release software anytime
- Any build could end up being a release
- Build artifacts once - deploy as needed

Here is a simple Continuous Delivery workflow, implemented in this project:

<img width="880" src="https://cloud.githubusercontent.com/assets/6069066/14159789/0dd7a7ce-f6e9-11e5-9fbb-a7fe0f4431e3.png">

In this [configuration](https://github.com/sqshq/PiggyMetrics/blob/master/.travis.yml), Travis CI builds tagged images for each successful git push. So, there are always `latest` image for each microservice on [Docker Hub](https://hub.docker.com/r/sqshq/) and older images, tagged with git commit hash. It's easy to deploy any of them and quickly rollback, if needed.

## How to run all the things?

Keep in mind, that you are going to start 8 Spring Boot applications, 4 MongoDB instances and RabbitMq. Make sure you have `4 Gb` RAM available on your machine. You can always run just vital services though: Gateway, Registry, Config, Auth Service and Account Service.

#### Before you start
- Install Docker and Docker Compose.
- Export environment variables: `CONFIG_SERVICE_PASSWORD`, `NOTIFICATION_SERVICE_PASSWORD`, `STATISTICS_SERVICE_PASSWORD`, `ACCOUNT_SERVICE_PASSWORD`, `MONGODB_PASSWORD`

#### Production mode
In this mode, all latest images will be pulled from Docker Hub. Just copy `docker-compose.yml` and hit `docker-compose up -d`.

#### Development mode
If you'd like to build images yourself (with some changes in the code, for example), you have to clone all repository and build artifacts with maven. Then, run `docker-compose -f docker-compose.yml -f docker-compose.dev.yml up -d`

`docker-compose.dev.yml` inherits `docker-compose.yml` with additional possibility to build images locally and expose all containers ports for convenient development.

#### Important endpoints
- http://DOCKER-HOST:80 - Gateway
- http://DOCKER-HOST:8761 - Eureka Dashboard
- http://DOCKER-HOST:9000/hystrix - Hystrix Dashboard
- http://DOCKER-HOST:8989 - Turbine stream (source for the Hystrix Dashboard)
- http://DOCKER-HOST:15672 - RabbitMq management (default login/password: guest/guest)

#### Notes
All Spring Boot applications require already running [Config Server](https://github.com/sqshq/PiggyMetrics#config-service) for startup. But we can start all containers simultaneously because of `fail-fast` Spring Boot property and `restart: always` docker-compose option. That means all dependent containers will try to restart until Config Server will be up and running.

Also, Service Discovery mechanism needs some time after all applications startup. Any service is not available for discovery by clients until the instance, the Eureka server and the client all have the same metadata in their local cache, so it could take 3 heartbeats. Default heartbeat period is 30 seconds.

## Feedback welcome

PiggyMetrics is open source, and would greatly appreciate your help. Feel free to contact me with any questions.
