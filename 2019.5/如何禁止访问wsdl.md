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

