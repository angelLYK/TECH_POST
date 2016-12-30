# springMVC与字符编解码
## 背景介绍
通过网络交互的程序，逃不开的话题就是字符编码，这里只讨论设计springMVC的字符编码。

## 产生乱码的情况
产生乱码的原因很简单，编码的字符集和解码的字符集不一致时，就会产生乱码。

数据在网络中都是通过字节流来传输的（TCP协议--面向字节流），发送字符串的一方在发送数据时，首先要进行encode，如果没有指定字符集，那么就会按照系统默认的字符集来进行encode。同理，字符串的接收方必须要进行decode，decode的时候也要指定字符集，如果不指定，也会按照系统默认的字符集。各种不通的系统，字符集是不定的，因而为了让程序更加的健壮，还是不要使用系统的默认字符集。大家都知道根基不稳定，大厦总会坍塌的，依赖系统字符集就是这个道理。一般在企业级开发时，都会指定字符集，我不知道其他项目组怎么样，但是我们组是必须要指定字符集的，且为UTF-8。

## springMVC中设置字符集的方法
涉及到字符集的地方包含两处：
1. 接受到客户端的请求，解析其中的参数信息
2. 返回给客户端时，编码返回的数据

### 针对问题1，普遍的配置方式为：

* web.xml中包含下面的filter

```java
<filter>  
    <filter-name>encodingFilter</filter-name>  
    <filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>  
    <init-param>  
       <param-name>encoding</param-name>  
       <param-value>UTF-8</param-value>  
    </init-param>  
    <init-param>  
       <param-name>forceEncoding</param-name>  
       <param-value>true</param-value>  
    </init-param>  
</filter>  
<filter-mapping>  
    <filter-name>encodingFilter</filter-name>  
    <url-pattern>/*</url-pattern>  
</filter-mapping> 
```

有的小伙伴认为，添加上这个Filter，返回的字符编码不用管了，也会是UTF-8。但是实际情况还真得用管，返回的字符编码会被springMVC默认的字符编码覆盖。我们看一下CharacterEncodingFilter的javaDoc怎么讲：

```java
/**
 * Servlet Filter that allows one to specify a character encoding for requests.
 * This is useful because current browsers typically do not set a character
 * encoding even if specified in the HTML page or form.
 *
 * <p>This filter can either apply its encoding if the request does not already
 * specify an encoding, or enforce this filter's encoding in any case
 * ("forceEncoding"="true"). In the latter case, the encoding will also be
 * applied as default response encoding (although this will usually be overridden
 * by a full content type set in the view).
 *
 * @author Juergen Hoeller
 * @since 15.03.2004
 * @see #setEncoding
 * @see #setForceEncoding
 * @see javax.servlet.http.HttpServletRequest#setCharacterEncoding
 * @see javax.servlet.http.HttpServletResponse#setCharacterEncoding
 */
```

重点是最后一句：although this will usually be overridden by a full content type set in the view。也就是说：CharacterEncodingFilter确实是会改变返回的字符编码，但是会被view覆盖。所以CharacterEncodingFilter只能控制请求的数据编码，对于返回的数据编码，其是无能为力的。

* 或者通过java config的方式

```java
public class WebAppInitializer implements WebApplicationInitializer {

	@Override
	public void onStartup(ServletContext servletContext) throws ServletException {
		
		//set request encoding utf-8
		CharacterEncodingFilter characterEncodingFilter = new CharacterEncodingFilter();
    characterEncodingFilter.setEncoding("UTF-8");
    characterEncodingFilter.setForceEncoding(true);

    FilterRegistration.Dynamic characterEncoding = servletContext.addFilter("characterEncoding", characterEncodingFilter);
    characterEncoding.addMappingForUrlPatterns(EnumSet.of(DispatcherType.REQUEST, DispatcherType.FORWARD), true, "/*");

		
		System.setProperty("cloud.rootPath", servletContext.getRealPath("/"));
		System.out.println(servletContext.getRealPath("/"));
		
		// Create the 'root' Spring application context
    AnnotationConfigWebApplicationContext rootContext = new AnnotationConfigWebApplicationContext();
    rootContext.register(RootConfiguration.class);

     // Manage the lifecycle of the root application context
    servletContext.addListener(new ContextLoaderListener(rootContext));

    // Create the dispatcher servlet's Spring application context
    AnnotationConfigWebApplicationContext dispatcherServlet = new AnnotationConfigWebApplicationContext();
    dispatcherServlet.register(WebConfiguration.class);

    // Register and map the dispatcher servlet
    ServletRegistration.Dynamic dispatcher = servletContext.addServlet("dispatcher", new DispatcherServlet(dispatcherServlet));
    dispatcher.setLoadOnStartup(1);
    dispatcher.addMapping("/");
	}

}

```

### 针对问题2，则有多种解决方案了，最简单的方式如下：

```java
@RequestMapping(value = "/api/path/to/resource", produces = "application/json;charset=UTF-8")
@ResponseBody
public void create(Document document, HttpServletRespone respone) {
}
```
但是这样子写需要在所有的controller中的所有api方法中都添加：produces = "application/json;charset=UTF-8"，如果哪一天不想用UTF-8了，改用其他的了，那改起来就吃力了。

有没有更简单的更优雅的解决方法？通常问这个问题，答案当然存在了，直接上代码：

```java
public class WebConfiguration extends WebMvcConfigurerAdapter{

	@Override
	public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
		converters.add(responseBodyConverter());
		super.configureMessageConverters(converters);
	}

	@Bean
	public HttpMessageConverter<?> responseBodyConverter() {
		StringHttpMessageConverter converter = new StringHttpMessageConverter();
        converter.setSupportedMediaTypes(Arrays.asList(MediaType.APPLICATION_JSON_UTF8));
        return converter;
	}
}

```

# 参考链接
* [How to overwrite StringHttpMessageConverter DEFAULT_CHARSET to use UTF8 in spring 4](http://stackoverflow.com/questions/27606769/how-to-overwrite-stringhttpmessageconverter-default-charset-to-use-utf8-in-sprin)
* [Spring MVC UTF-8 Encoding](http://stackoverflow.com/questions/5928046/spring-mvc-utf-8-encoding)
* [Spring MVC Java config](http://stackoverflow.com/questions/22225040/spring-mvc-java-config)

