[zuul实现所有接口对于带指定前缀和不带前缀的url均能兼容访问](https://blog.csdn.net/qq_39470742/article/details/83274609)

我们的项目里通过zuul实现路由转发，前几日接到这么一个需求，需要实现所有接口对于带指定前缀和不带前缀的url均能兼容访问，网上这方面的文档并不多，因此为了处理这个需求，捎带着阅读了一下zuul的部分源码
首先说一下结论，zuul本身便实现了这个功能，对于带/zuul的前缀的url会自动去掉该前缀进行转发，完美匹配这次需求。
接着开始理解源码，看一看zuul是怎么实现的。
在springboot项目中使用zuul需要使用@EnableZuulServer或@EnableZuulProxy中的至少一个注解。其中@EnableZuulServer对应ZuulServerAutoConfiguration，@EnableZuulProxy对应ZuulProxyAutoConfiguration，ZuulServerAutoConfiguration是ZuulProxyAutoConfiguration的父类，因此可以简单理解成@EnableZuulServer是@EnableZuulProxy的一个简化版本。
如下图：EnableZuulServer --> ZuulServerMarkerConfiguration --> ZuulServerAutoConfiguration，EnableZuulProxy也类似。

![](assets\2018102214490884.png)



![](assets\20181022144835394.png)





![](assets\20181022144954744.png)



在这里主要针对EnableZuulProxy展开，我们开一下zuul是怎么针对请求的url进行处理的。
在ZuulProxyAutoConfiguration类中，我们注入了PreDecorationFilter进行拦截。

```
// pre filters
@Bean
public PreDecorationFilter preDecorationFilter(RouteLocator routeLocator, ProxyRequestHelper proxyRequestHelper) {
	return new PreDecorationFilter(routeLocator, this.server.getServlet().getServletPrefix(), this.zuulProperties,
			proxyRequestHelper);
}
```


进入这个类我们可以看到该filter的类型是pre，如果已经处理过转发逻辑的请求在不在拦截处理。

```
@Override
public int filterOrder() {
	return PRE_DECORATION_FILTER_ORDER; //5
}

@Override
public String filterType() {
	return PRE_TYPE;
}

@Override
public boolean shouldFilter() {
	RequestContext ctx = RequestContext.getCurrentContext();
	return !ctx.containsKey(FORWARD_TO_KEY) // a filter has already forwarded
			&& !ctx.containsKey(SERVICE_ID_KEY); // a filter has already determined serviceId
}
```


接下来我们进入run方法仔细了解PreDecorationFilter的拦截逻辑。
```
@Override
public Object run() {
	//1.根据请求的url找到对应的路由Route
	RequestContext ctx = RequestContext.getCurrentContext();
	final String requestURI = this.urlPathHelper.getPathWithinApplication(ctx.getRequest());
	Route route = this.routeLocator.getMatchingRoute(requestURI);
	//2.根据Route进行相应的转发
	if (route != null) {
		String location = route.getLocation();
		if (location != null) {
			ctx.put(REQUEST_URI_KEY, route.getPath());
			ctx.put(PROXY_KEY, route.getId());
			if (!route.isCustomSensitiveHeaders()) {
				this.proxyRequestHelper
						.addIgnoredHeaders(this.properties.getSensitiveHeaders().toArray(new String[0]));
			}
			else {
				this.proxyRequestHelper.addIgnoredHeaders(route.getSensitiveHeaders().toArray(new String[0]));
			}

if (route.getRetryable() != null) {
				ctx.put(RETRYABLE_KEY, route.getRetryable());
			}
	

if (location.startsWith(HTTP_SCHEME+":") || location.startsWith(HTTPS_SCHEME+":")) {
				ctx.setRouteHost(getUrl(location));
				ctx.addOriginResponseHeader(SERVICE_HEADER, location);
			}
			else if (location.startsWith(FORWARD_LOCATION_PREFIX)) {
				ctx.set(FORWARD_TO_KEY,
						StringUtils.cleanPath(location.substring(FORWARD_LOCATION_PREFIX.length()) + route.getPath()));
				ctx.setRouteHost(null);
				return null;
			}
			else {
				// set serviceId for use in filters.route.RibbonRequest
				ctx.set(SERVICE_ID_KEY, location);
				ctx.setRouteHost(null);
				ctx.addOriginResponseHeader(SERVICE_ID_HEADER, location);
			}
			if (this.properties.isAddProxyHeaders()) {
				addProxyHeaders(ctx, route);
				String xforwardedfor = ctx.getRequest().getHeader(X_FORWARDED_FOR_HEADER);
				String remoteAddr = ctx.getRequest().getRemoteAddr();
				if (xforwardedfor == null) {
					xforwardedfor = remoteAddr;
				}
				else if (!xforwardedfor.contains(remoteAddr)) { // Prevent duplicates
					xforwardedfor += ", " + remoteAddr;
				}
				ctx.addZuulRequestHeader(X_FORWARDED_FOR_HEADER, xforwardedfor);
			}
			if (this.properties.isAddHostHeader()) {
				ctx.addZuulRequestHeader(HttpHeaders.HOST, toHostHeader(ctx.getRequest()));
			}
		}
	}
	//3.Route为null，进行相应的fallback处理
	else {
		log.warn("No route found for uri: " + requestURI);
	
		String fallBackUri = requestURI;
		String fallbackPrefix = this.dispatcherServletPath; // default fallback
															// servlet is
															// DispatcherServlet
	
		if (RequestUtils.isZuulServletRequest()) {
			// remove the Zuul servletPath from the requestUri
			log.debug("zuulServletPath=" + this.properties.getServletPath());
			fallBackUri = fallBackUri.replaceFirst(this.properties.getServletPath(), "");
			log.debug("Replaced Zuul servlet path:" + fallBackUri);
		}
		else {
			// remove the DispatcherServlet servletPath from the requestUri
			log.debug("dispatcherServletPath=" + this.dispatcherServletPath);
			fallBackUri = fallBackUri.replaceFirst(this.dispatcherServletPath, "");
			log.debug("Replaced DispatcherServlet servlet path:" + fallBackUri);
		}
		if (!fallBackUri.startsWith("/")) {
			fallBackUri = "/" + fallBackUri;
		}
		String forwardURI = fallbackPrefix + fallBackUri;
		forwardURI = forwardURI.replaceAll("//", "/");
		ctx.set(FORWARD_TO_KEY, forwardURI);
	}
	return null;
}
```

其中Route route = this.routeLocator.getMatchingRoute(requestURI)；是根据url路径获取路由的核心逻辑，我们继续跟踪，routeLocator是一个接口，对于getMatchingRoute方法有两个实现类SimpleRouteLocator和CompositeRouteLocator，其中CompositeRouteLocator是遍历routeLocator.getMatchingRoute方法，因此我们进入SimpleRouteLocator类的getMatchingRoute方法

```
//CompositeRouteLocator
@Override
public Route getMatchingRoute(String path) {
	for (RouteLocator locator : routeLocators) {
		Route route = locator.getMatchingRoute(path);
		if (route != null) {
			return route;
		}
	}
	return null;
}
```

```
//SimpleRouteLocator
@Override
public Route getMatchingRoute(final String path) {

	return getSimpleMatchingRoute(path);

}
protected Route getSimpleMatchingRoute(final String path) {
	if (log.isDebugEnabled()) {
		log.debug("Finding route for path: " + path);
}

// This is called for the initialization done in getRoutesMap()
	//1.获取ZuulRoute的映射对
	getRoutesMap();
	
	if (log.isDebugEnabled()) {
		log.debug("servletPath=" + this.dispatcherServletPath);
		log.debug("zuulServletPath=" + this.zuulServletPath);
		log.debug("RequestUtils.isDispatcherServletRequest()="
				+ RequestUtils.isDispatcherServletRequest());
		log.debug("RequestUtils.isZuulServletRequest()="
				+ RequestUtils.isZuulServletRequest());
	}
	
	//2.对url路径预处理
	String adjustedPath = adjustPath(path);
	
	//3.根据路径获取匹配的ZuulRoute
	ZuulRoute route = getZuulRoute(adjustedPath);
	
	//4.根据ZuulRoute组装Route
	return getRoute(route, adjustedPath);
}
```
我们可以看到SimpleRouteLocator提供了protected类型的getSimpleMatchingRoute可以留给子类进行扩展，如果有需要，我们可以定义一个SimpleRouteLocator的子类自定义这部分的逻辑。
getRoutesMap();是这个类的一个核心方法方法，读取我们的路由配置组成映射并保存在线程本地变量中，我们留待下一篇在展开。
String adjustedPath = adjustPath(path);方法队url路径进行了预处理，是我们今天的重点，也是过滤/zuul前缀的逻辑所在。
```
private String adjustPath(final String path) {
	String adjustedPath = path;

	if (RequestUtils.isDispatcherServletRequest()
			&& StringUtils.hasText(this.dispatcherServletPath)) {
		if (!this.dispatcherServletPath.equals("/")) {
			adjustedPath = path.substring(this.dispatcherServletPath.length());
			log.debug("Stripped dispatcherServletPath");
		}
	}
	else if (RequestUtils.isZuulServletRequest()) {
		if (StringUtils.hasText(this.zuulServletPath)
				&& !this.zuulServletPath.equals("/")) {
			adjustedPath = path.substring(this.zuulServletPath.length());
			log.debug("Stripped zuulServletPath");
		}
	}
	else {
		// do nothing
	}
	
	log.debug("adjustedPath=" + adjustedPath);
	return adjustedPath;
}
```

这里有两个个判断分支内都对url路径进行了截取，而且截取的都是字符串的前面一部分，这时候我们应该想到这儿和我们的需求有匹配之处，而这两个判断条件都用到RequestUtils类，我们继续进入
```
public class RequestUtils {

	/**
	 * @deprecated use {@link org.springframework.cloud.netflix.zuul.filters.support.FilterConstants#IS_DISPATCHER_SERVLET_REQUEST_KEY}
	 */
	@Deprecated
	public static final String IS_DISPATCHERSERVLETREQUEST = IS_DISPATCHER_SERVLET_REQUEST_KEY;
	
	public static boolean isDispatcherServletRequest() {
		return RequestContext.getCurrentContext().getBoolean(IS_DISPATCHER_SERVLET_REQUEST_KEY);
	}
	
	public static boolean isZuulServletRequest() {
		//extra check for dispatcher since ZuulServlet can run from ZuulController
		return !isDispatcherServletRequest() && RequestContext.getCurrentContext().getZuulEngineRan();
	}	
}
```

继续进入RequestContext，可以发现RequestContext实际上是一个继承了ConcurrentHashMap<String, Object>的映射对。判断上述两个判断条件是否成立的方法实际上就是判断“isDispatcherServletRequest”和“zuulEngineRan”这两个key值对应的value是否为true。
因此，我们上述的代码对url的请求路径进行预处理的逻辑是：
1.如果isDispatcherServletRequest对应的value值为true，并且路径中包含dispatcherServletPath，直接截取。
2.步骤1不成立，且zuulEngineRan对应的value为true，并且路径中包含zuulServletPath，直接截取。
3.上诉步骤都不成立，不处理。
其中我们一开始所说的解决方案对应的就是步骤2，zuul对应zuulServletPath。
那么dispatcherServletPath和zuulServletPath在哪里设置呢？在ZuulServerAutoConfiguration中我们生成并注入了SimpleRouteLocator类的实例。

```
@Bean
@ConditionalOnMissingBean(SimpleRouteLocator.class)
public SimpleRouteLocator simpleRouteLocator() {
	return new SimpleRouteLocator(this.server.getServlet().getServletPrefix(),
			this.zuulProperties);
}
```
```
//SimpleRouteLocator的构造方法
public SimpleRouteLocator(String servletPath, ZuulProperties properties) {
	this.properties = properties;
	if (StringUtils.hasText(servletPath)) {
		this.dispatcherServletPath = servletPath;
	}

	this.zuulServletPath = properties.getServletPath();
}
```

在ZuulProperties类中servletPath默认值为/zuul，当然我们可以通过配置zuul.servletPath进行修改。
dispatcherServletPath也同理，在ServerProperties的内部类Servlet的path属性获得，
```
/**
 * Path to install Zuul as a servlet (not part of Spring MVC). The servlet is more
 * memory efficient for requests with large bodies, e.g. file uploads.
 */
private String servletPath = "/zuul";
```
```
public String getServletPrefix() {
	String result = this.path;
	int index = result.indexOf('*');
	if (index != -1) {
		result = result.substring(0, index);
	}
	if (result.endsWith("/")) {
		result = result.substring(0, result.length() - 1);
	}
	return result;
}
```

上面我们知道了怎么获取到servletPath和dispatcherServletPath的值，那么zuul在上面时候上面情况下设置RequestContext中对应的两个属性值是否为true呢？

在ZuulServerAutoConfiguration还注入了一个pre类型且order为-3的ZuulFilter ：ServletDetectionFilter，它是最早执行的ZuulFilter，对所有请求生效。从它的的名字我们就可以看出它的主要作用是检测当前请求是通过Spring的DispatcherServlet处理运行，还是通过ZuulServlet来处理运行的。
```
@Override
public Object run() {
	RequestContext ctx = RequestContext.getCurrentContext();
	HttpServletRequest request = ctx.getRequest();
	if (!(request instanceof HttpServletRequestWrapper) 
			&& isDispatcherServletRequest(request)) {
		ctx.set(IS_DISPATCHER_SERVLET_REQUEST_KEY, true);
	} else {
		ctx.set(IS_DISPATCHER_SERVLET_REQUEST_KEY, false);
	}

	return null;
}
```

ServletDetectionFilter设置了isDispatcherServletRequest属性，而ZuulServlet类通过context.setZuulEngineRan();设置了“zuulEngineRan”属性。
```
@Override
   public void service(javax.servlet.ServletRequest servletRequest, javax.servlet.ServletResponse servletResponse) throws ServletException, IOException {
       try {
           init((HttpServletRequest) servletRequest, (HttpServletResponse) servletResponse);

           // Marks this request as having passed through the "Zuul engine", as opposed to servlets
           // explicitly bound in web.xml, for which requests will not have the same data attached
           RequestContext context = RequestContext.getCurrentContext();
           context.setZuulEngineRan();
    
           try {
               preRoute();
           } catch (ZuulException e) {
               error(e);
               postRoute();
               return;
           }
           try {
               route();
           } catch (ZuulException e) {
               error(e);
               postRoute();
               return;
           }
           try {
               postRoute();
           } catch (ZuulException e) {
               error(e);
               return;
           }
    
       } catch (Throwable e) {
           error(new ZuulException(e, 500, "UNHANDLED_EXCEPTION_" + e.getClass().getName()));
       } finally {
           RequestContext.getCurrentContext().unset();
       }
   }
```

37
最后，我们明确一点：
在SpringMvc中，我们将请求交由DispatcherServlet进行处理。
而使用了zuul后，对于/zuul前缀的url会交由ZuulServlet进行处理。
事实上，这两个前缀也对应我们前面说SimpleRouteLocator的dispatcherServletPath和servletPath属性。

除了带/zuul能实现我们一开始的需求以外，如果需要类似的需要在路由转发中对路径进行处理的逻辑，根据上面的分析，我们也可以通过下面几种方式实现：
1.定义一个继承SimpleRouteLocator的子类并注入spring，重写getSimpleMatchingRoute实现自己的自定义逻辑实现路径的预处理。
2.定义一个实现ZuulFilter的类并注入spring，要求filterType为pre，并且order小于PreDecorationFilter的order（5），获取到请求路径后进行处理，其余逻辑可以模仿PreDecorationFilter。
3.定义一个实现ZuulFilter的类并注入spring，要求filterType为pre，并且order大于PreDecorationFilter的order（5），对经过PreDecorationFilter处理后的请求再次拦截，修改RequestContext中的“requestURI”和“proxy”对应的值，这两者对应route的path和id，确定了转换后url路径的值。

