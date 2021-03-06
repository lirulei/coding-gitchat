# 第10课：消息总线

其实在上一课我们已经接触过了消息总线，那就是 Spring Cloud Bus，这一课我们将继续深入研究 Spring Cloud Bus 的一些特性。

### 局部刷新

Spring Cloud Bus 用于实现在集群中传播一些状态变化（例如：配置变化），它常常与 Spring Cloud Config 联合实现热部署。上一课我们体验了配置的自动刷新，但每次都会刷新所有微服务，有些时候我们只想刷新部分微服务的配置，这时就需要通过 `/bus/refresh` 断点的 destination 参数来定位要刷新的应用程序。

它的基本用法如下：

> /bus/refresh?destination=application:port

其中，application 为各微服务指定的名字，port 为端口，如果我们要刷新所有指定微服务名字下的配置，则 destination 可以设置为 application：**例如：`/bus/refresh?destination=eurekaclient:`**，代表刷新所有名字为 EurekaClient 的微服务配置。

### 改进架构

在前面的示例中，我们是通过某一个微服务的 `/bus/refesh` 断点来实现配置刷新，但是这种方式并不优雅，它有以下弊端：

1. 破坏了微服务的单一职责原则，微服务客户端理论上应只关注自身业务，而不应该负责配置刷新。
2. 破坏了微服务各节点的对等性。
3. 有一定的局限性。在微服务迁移时，网络地址时常会发生改变，这时若想自动刷新配置，就不得不修改 Git 仓库的 WebHook 配置。

因此，我们应考虑改进架构，将 ConfigServer 也加入到消息总线来，将其 `/bus/refresh` 用于实现配置的自动刷新。这样，各个微服务节点就只需关注自身业务，无需再承担配置自动刷新的任务（具体代码已上传到 Github 上，此时不再列举）。

我们来看看此时的架构图：

<img src="https://tva1.sinaimg.cn/large/007S8ZIlgy1ggcrif2nb0j30vi0ok41f.jpg" style="zoom:45%;" />

**注意：** 所有需要刷新配置的服务都需要添加以下依赖。

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-bus-amqp</artifactId>
</dependency>
```

并且需要在配置文件设置 rabbitmq 信息：

```yaml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest
```

### 消息总线事件

在某些场景下，我们需要知道 Spring Cloud Bus 的事件传播细节，这时就需要跟踪消息总线事件。

要实现跟踪消息总线事件是一件很容易的事情，只需要修改配置文件，如下所示：

```yaml
server:
  port: 8888
spring:
  application:
    name: config
  profiles:
    active: dev
  cloud:
    bus:
      trace:
        enable: true
    config:
      server:
        git:
          uri: https://github.com/lynnlovemin/SpringCloudLesson.git # 配置git仓库地址
          searchPaths: 第09课/config # 配置仓库路径
          username: lynnlovemin # 访问git仓库的用户名
          password: ********** # 访问git仓库的用户密码
      label: master # 配置仓库的分支
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest
eureka:
  instance:
    hostname: ${spring.cloud.client.ipAddress}
    instanceId: ${spring.cloud.client.ipAddress}:${server.port}
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
management:
  security:
    enabled: false
```

我们将 spring.cloud.bus.trace.enabled 设置为 true 即可，这样我们在 POST 请求 `/bus/refresh` 后，浏览器访问 `/trace` 端点即可看到如下数据：

```json
[
  {
    "timestamp": 1527299528556,
    "info": {
      "method": "GET",
      "path": "/eurekaclient/dev/master",
      "headers": {
        "request": {
          "accept": "application/json, application/*+json",
          "user-agent": "Java/1.8.0_40",
          "host": "192.168.31.218:8888",
          "connection": "keep-alive"
        },
        "response": {
          "X-Application-Context": "config:dev:8888",
          "Content-Type": "application/json;charset=UTF-8",
          "Transfer-Encoding": "chunked",
          "Date": "Sat, 26 May 2018 01:52:08 GMT",
          "status": "200"
        }
      },
      "timeTaken": "4200"
    }
  },
  {
    "timestamp": 1527299524802,
    "info": {
      "method": "POST",
      "path": "/bus/refresh",
      "headers": {
        "request": {
          "host": "localhost:8888",
          "user-agent": "curl/7.54.0",
          "accept": "*/*"
        },
        "response": {
          "X-Application-Context": "config:dev:8888",
          "status": "200"
        }
      },
      "timeTaken": "1081"
    }
  }
]
```

这样就可以清晰的看到传播细节了。