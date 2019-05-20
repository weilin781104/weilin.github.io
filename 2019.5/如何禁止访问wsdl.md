# 背景

电信客户方提出安全漏洞：不允许暴露wsdl

需求：不允许暴露wsdl，但不影响访问web service接口



# 思路

获取wsdl是通过http Get， 而访问web service接口是通过http Post实现，所以通过web filter区分http method。

# 解决方案

以TM1为例：

生成filter类：

```
public class SoapFilter implements Filter {

	@Override
	public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse,
			FilterChain filterChain) throws IOException, ServletException {
		HttpServletRequest request = (HttpServletRequest) servletRequest;
		HttpServletResponse response=(HttpServletResponse)servletResponse;
		if("get".equalsIgnoreCase(request.getMethod())&& !("127.0.0.1".equals(request.getRemoteAddr()))) {
			response.setStatus(403);
			PrintWriter out = response.getWriter();	
			out.println("get wsdl is forbidden!!!");
			out.flush();
			out.close();
		}
		
		filterChain.doFilter(servletRequest, servletResponse);
	}
	
	@Override
	public void init(FilterConfig arg0) throws ServletException {
		// TODO Auto-generated method stub

	}
	
	
	@Override
	public void destroy() {
		// TODO Auto-generated method stub

	}

}
```

在web.xml中配置filter mapping：

```
<filter>
		<filter-name>soapFilter</filter-name>
		<filter-class>com.sanss.tm.web.SoapFilter</filter-class>
	</filter>
    <filter-mapping>
		<filter-name>soapFilter</filter-name>
		<url-pattern>/TmWsPort</url-pattern>
	</filter-mapping>
	<filter-mapping>
		<filter-name>soapFilter</filter-name>
		<url-pattern>/InnerWsPort</url-pattern>
	</filter-mapping>		
```



部署后测试：

浏览器不能直接访问了

![](assets\20190518220004.png)

![](assets\20190518220047.png)

为了防止wsdl无法访问，还是允许服务器本地访问：

![](assets\20190518221203.png)



xmlspy能正常访问接口（预先根据wsdl创建新的soap请求）

![](assets\20190518221418.png)



# 问题

虽然xmlspy 访问 <http://116.228.215.9:8088/TM/InnerWsPort?wsdl ，只有一次post请求，预先根据wsdl url创建一个新的soap请求。

![](assets\20190518211742.png)



但是我记得soap客户端访问一次webservice接口是有两次请求的，第一次get wsdl文件，第二次往wsdl地址里post。 如果通过filter禁止get请求，客户端第一次get不到wsdl文件，是否能正常第二次post吗？

以烧写客户端请求抓包为例：

![](assets\20190518211450.png)

可以明显看到两次请求的。

用前面服务器获取到的wsdl文件生成java soap client模拟发现不行，post请求前会get一下wsdl文件，get时报错。

![](assets\20190518223646.png)

从抓包来看，当无法get wsdl时会用server无法识别的格式post数据。

![](assets\20190518224409.png)

![](assets\20190518224605.png)

![](assets\20190518224641.png)



所以实际还要看是否影响机顶盒访问。

# 最终解决方案

通过抓包发现终端访问ETV soap接口是直接post的，之前没有get wsdl，所以可以把get禁掉

内部接口互相调用使用soap 客户端自动生成的，在post之前会get wsdl，把get禁掉之后就会调不通接口，所以只允许平台ip访问的get允许之外，其它外部访问get都禁掉



## 接口说明

| 模块    | 接口            | 接口调用情况                                                 | 处理方法                            |
| ------- | --------------- | ------------------------------------------------------------ | ----------------------------------- |
| TM1     | TmWs            | 面向终端接口，post之前没有get wsdl                           | 把外部的get请求都禁掉               |
|         | InnerWs         | 看日志发现主要是boss过来的清缓存请求，post之前会get          | 除平台本地IP之外的外部get请求都禁掉 |
| ETVBMS1 | TerminalWS      | 面向终端接口，post之前没有get wsdl                           | 把外部的get请求都禁掉               |
|         | InnerWS         | 看日志发现主要是boss或者应用互相调用过来的清缓存请求，post之前会get | 除平台本地IP之外的外部get请求都禁掉 |
|         | BestvWS         | 看日志没有请求了                                             | 把外部的get请求都禁掉               |
| OTTBMS  | CDNtokenDeal    | 面向终端接口，post之前没有get wsdl                           | 把外部的get请求都禁掉               |
|         | M3U8tokenDeal   | 面向CDN的M3u8tokenAuth接口，抓包发现post之前没有get wsdl     | 把外部的get请求都禁掉               |
|         | OTTBMSInnerDeal | 看日志没有请求了                                             | 把外部的get请求都禁掉               |
| TM2     |                 | 只有应急时挂到外网，先不处理                                 |                                     |
| ETVBMS2 |                 | 只有应急时挂到外网，先不处理                                 |                                     |
| TM4     | InnerWS         | 兼容用的，目前不使用                                         | 把外部的get请求都禁掉               |
|         | BestvWS         | 兼容用的，目前不使用                                         | 把外部的get请求都禁掉               |
| ETVBMS3 | TerminalWS      | 兼容用的，目前不使用                                         | 把外部的get请求都禁掉               |
|         | InnerWS         | 兼容用的，目前不使用                                         | 把外部的get请求都禁掉               |
|         | BestvWS         | 兼容用的，目前不使用                                         | 把外部的get请求都禁掉               |



## filter类业务逻辑：

http get方法  && clientip=http header("ipaddr/client-ip") 不为空 &&  clientip不以开头（"10.0.0","124.75","222.68"）     拒绝访问

其它都放开使用

```
private boolean isClientIPLocal(String clientIP){
		if(clientIP.startsWith("10.0.0"))
			return true;
		else if(clientIP.startsWith("124.75"))
			return true;
		else if(clientIP.startsWith("222.68"))
			return true;
		else
			return false;
	}

	@Override
	public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse,
			FilterChain filterChain) throws IOException, ServletException {
		HttpServletRequest request = (HttpServletRequest) servletRequest;
		HttpServletResponse response=(HttpServletResponse)servletResponse;
		if("get".equalsIgnoreCase(request.getMethod())&& !CommonUtil.ifFieldIsNull(request.getHeader("ipaddr"))) {
			String clientIP=request.getHeader("ipaddr").trim();
			if (!isClientIPLocal(clientIP)){			
				response.setStatus(403);
				PrintWriter out = response.getWriter();	
				out.println("get wsdl is forbidden!!!");
				out.flush();
				out.close();
			}
		}
		
		filterChain.doFilter(servletRequest, servletResponse);
	}
```

