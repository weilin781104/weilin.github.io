## zuul 内置参数

zuul集成了ribbon和hystrix，一些参数的优先级高于ribbon和hystrix设置

spring-cloud-netflix-zuul-ver.jar

参数类：org.springframework.cloud.netflix.zuul.filters.ZuulProperties

默认值：spring-configuration-metadata.json





### zuul.host.maxTotalConnections

适用于ApacheHttpClient，如果是okhttp无效。每个服务的http客户端连接池最大连接，默认是200.

### zuul.host.maxPerRouteConnections

适用于ApacheHttpClient，如果是okhttp无效。每个route可用的最大连接数，默认值是20。

### zuul.semaphore.max-semaphores

Hystrix最大的并发请求`execution.isolation.semaphore.maxConcurrentRequests`，这个值并非`TPS`、`QPS`、`RPS`等都是相对值，指的是1秒时间窗口内的事务/查询/请求，`semaphore.maxConcurrentRequests`是一个绝对值，无时间窗口，相当于亚毫秒级的。当请求达到或超过该设置值后，其其余就会被拒绝。默认值是100。参考: [Hystrix semaphore和thread隔离策略的区别及配置参考](https://www.jianshu.com/p/b8d21248c9b1)

这个参数本来直接可以通过Hystrix的命名规则来设置，但被zuul重新设计了，使得在zuul中semaphores的最大并发请求有4个方法的参数可以设置，如果4个参数都存在优先级（1~4）由高到低：

- [优先级1]zuul.eureka.api.semaphore.maxSemaphores
- [优先级2]zuul.semaphore.max-semaphores
- [优先级3]hystrix.command.api.execution.isolation.semaphore.maxConcurrentRequests
- [优先级4]hystrix.command.default.execution.isolation.semaphore.maxConcurrentRequests

需要注意的是：在Camden.SR3版本的zuul中`hystrix.command.default.execution.isolation.semaphore.maxConcurrentRequests`设置不会起作用，这是因为在`org.springframework.cloud.netflix.zuul.filters.ZuulProperties.HystrixSemaphore.maxSemaphores=100`设置了默认值100，因此`zuul.semaphore.max-semaphores`的优先级高于`hystrix.command.default.execution.isolation.semaphore.maxConcurrentRequests`。

zuul.eureka.[commandKey].semaphore.maxSemaphores：
其中commandKey为

![img](D:\Develope\weilin\book\gitbook\weilinqwe.github.io\2019.8\735284-20181107095017274-886930959.png)

 

## 其他Hystrix参数：

`hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds`用来设置thread和semaphore两种隔离策略的超时时间，默认值是1000。

- 建议设置这个参数，在Hystrix 1.4.0之前，semaphore-isolated隔离策略是不能超时的，从1.4.0开始semaphore-isolated也支持超时时间了。
- 建议通过CommandKey设置不同微服务的超时时间,对于zuul而言，CommandKey就是service id：`hystrix.command.[CommandKey].execution.isolation.thread.timeoutInMilliseconds`



 

**zuul 重试配置**
　　Spring Cloud Zuul模块本身就包含了对于hystrix和ribbon的依赖，当我们使用zuul通过path和serviceId的组合来配置路由的时候，可以通过hystrix和ribbon的配置调整路由请求的各种时间超时机制。

 

  

1 ribbon配置举例

  配置连接超时时间1秒，请求处理时间2秒，统一服务server尝试重连1次，切换server重连1次

  ![img](assets\735284-20181107100317466-1124035478.png)

2 hystirx配置举例

　　![img](assets\735284-20181107100407255-2134511771.png)

这里需要注意的是hystrix的配置时间应该大于ribbon全部重试时间的总和，上面我配置的是2次重试，包括首次请求，三次时间是6秒

 

3 打开zuul的重试配置：

  ![img](assets\735284-20181107102014956-427494267.png)

特别注意zuul的重试配置需要依赖spring的retry，不然的话怎么配置都是徒劳

![img](assets\735284-20181107102039761-408851967.png)