# 前言

用zuul 1.*开发iptv3a网关的时候发现与预期不一致，因此仔细研究了zuul实现机制，发现路由规则匹配不到的路径根据不执行filter，然后又去研究了一把spring webmvc的handlermapping，handleradapter才明白里面的机制，特此记录一下。



# zuul filter实现机制

参考网上博文，并阅读了源代码

[Zuul详解](https://cloud.tencent.com/developer/article/1334258)

[Spring Cloud Zuul中DispatcherServlet和ZuulServlet](https://www.jianshu.com/p/f97096b8a39f)



zuul的filters不是servlet filter，不能在web第一层进行过滤。 虽然zuul core包里有ZuulServletFilter这个servlet filter，但是spring cloud netflix zuul没有用ZuulServletFilter。 zull filter是都由ZuulServlet里的zuulRunner的FilterProcessor来负责的。 

ZuulServerAutoConfiguration负责ZuulServlet的初始化，这里初始了两次，一次是直接加载ZuulServlet：

```
@Bean
	@ConditionalOnMissingBean(name = "zuulServlet")
	public ServletRegistrationBean zuulServlet() {
		ServletRegistrationBean<ZuulServlet> servlet = new ServletRegistrationBean<>(new ZuulServlet(),
				this.zuulProperties.getServletPattern());
		// The whole point of exposing this servlet is to provide a route that doesn't
		// buffer requests.
		servlet.addInitParameter("buffer-requests", "false");
		return servlet;
	}
```

这是直接匹配/zuul/路径的，而且不会buffer request，适合上传文件等multipart操作。

另一次是装载了ZuulHandlerMapping

```
@Bean
	public ZuulHandlerMapping zuulHandlerMapping(RouteLocator routes) {
		ZuulHandlerMapping mapping = new ZuulHandlerMapping(routes, zuulController());
		mapping.setErrorController(this.errorController);
		return mapping;
	}
```

，直接把路由规则的路径匹配给spring webmvc的dispatchServlet在/根路径进行匹配，zuulController封装代理了zuulservlet。

```
Spring Controller implementation that wraps a servlet instance which it manages internally. Such a wrapped servlet is not known outside of this controller; its entire lifecycle is covered here (in contrast to ServletForwardingController). 
Useful to invoke an existing servlet via Spring's dispatching infrastructure, for example to apply Spring HandlerInterceptors to its requests. 

```





# spring webmvc mapping机制

参考网文，阅读源代码，对照TRACE级别的spring日志

[[SpringMVC工作原理之二：HandlerMapping和HandlerAdapter](https://www.cnblogs.com/tengyunhao/p/7658952.html)]

根据日志对照源代码发现：

spring 会管理若干HandlerMapping（比如SimpleUrlHandlerMapping/WebMvcEndpointHandlerMapping/ControllerEndpointHandlerMapping/RequestMappingHandlerMapping/ZuulHandlerMapping/BeanNameUrlHandlerMapping）

一个请求进来，spring会在web一层根据HandlerMapping列表匹配一遍，这里如果根据SimpleUrlHandlerMapping匹配到了配置的servlet，就转给servlet运行结束（比如zuulservlet，hystrixMetricsCheck），其它即使匹配到了也继续转到dispatchServlet，dispatchServlet根据handlermapping列表再匹配一遍。



匹配时会遍历handlermapping，每个handlermapping都会根据路径匹配handler以及对应的handlerAdapter。由于zuulfiter是zuulservlet管理的，通过ZuulHandlerMapping把路由规则的路径和对应的handler zuulController发布到web容器里了。当非zuul路由规则内的路径请求进来时，根据zuulHandlerMapping匹配不到handler，就会去尝试其它handlermapping，就不会进zuulController，就不会进zuulservlet，也就不会经过zuul filter。 所以这就是order最前的pre logfilter打印不出那些请求日志的原因了。



由于zk config无法采用yaml配置文件方式，所以无法配置路由规则的顺序，所以无法在最后配置一条默认的全路径规则（需要对每一条精确匹配），所以不能把所有请求都通过ZuulHandlerMapping引入进来。



所以要打印所有请求日志，思路是 写个servlet filter，对/过滤，request中加入logBO，在处理流程中添加信息到logBO， filter回程中打印日志。