# springMVC���ַ������
## ��������
ͨ�����罻���ĳ����Ӳ����Ļ�������ַ����룬����ֻ�������springMVC���ַ����롣

## ������������
���������ԭ��ܼ򵥣�������ַ����ͽ�����ַ�����һ��ʱ���ͻ�������롣

�����������ж���ͨ���ֽ���������ģ�TCPЭ��--�����ֽ������������ַ�����һ���ڷ�������ʱ������Ҫ����encode�����û��ָ���ַ�������ô�ͻᰴ��ϵͳĬ�ϵ��ַ���������encode��ͬ���ַ����Ľ��շ�����Ҫ����decode��decode��ʱ��ҲҪָ���ַ����������ָ����Ҳ�ᰴ��ϵͳĬ�ϵ��ַ��������ֲ�ͨ��ϵͳ���ַ����ǲ����ģ����Ϊ���ó�����ӵĽ�׳�����ǲ�Ҫʹ��ϵͳ��Ĭ���ַ�������Ҷ�֪���������ȶ��������ܻ�̮���ģ�����ϵͳ�ַ��������������һ������ҵ������ʱ������ָ���ַ������Ҳ�֪��������Ŀ����ô���������������Ǳ���Ҫָ���ַ����ģ���ΪUTF-8��

## springMVC�������ַ����ķ���
�漰���ַ����ĵط�����������
1. ���ܵ��ͻ��˵����󣬽������еĲ�����Ϣ
2. ���ظ��ͻ���ʱ�����뷵�ص�����

### �������1���ձ�����÷�ʽΪ��

* web.xml�а��������filter

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

�е�С�����Ϊ����������Filter�����ص��ַ����벻�ù��ˣ�Ҳ����UTF-8������ʵ�����������ùܣ����ص��ַ�����ᱻspringMVCĬ�ϵ��ַ����븲�ǡ����ǿ�һ��CharacterEncodingFilter��javaDoc��ô����

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

�ص������һ�䣺although this will usually be overridden by a full content type set in the view��Ҳ����˵��CharacterEncodingFilterȷʵ�ǻ�ı䷵�ص��ַ����룬���ǻᱻview���ǡ�����CharacterEncodingFilterֻ�ܿ�����������ݱ��룬���ڷ��ص����ݱ��룬��������Ϊ���ġ�

* ����ͨ��java config�ķ�ʽ

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

### �������2�����ж��ֽ�������ˣ���򵥵ķ�ʽ���£�

```java
@RequestMapping(value = "/api/path/to/resource", produces = "application/json;charset=UTF-8")
@ResponseBody
public void create(Document document, HttpServletRespone respone) {
}
```
����������д��Ҫ�����е�controller�е�����api�����ж���ӣ�produces = "application/json;charset=UTF-8"�������һ�첻����UTF-8�ˣ������������ˣ��Ǹ������ͳ����ˡ�

��û�и��򵥵ĸ����ŵĽ��������ͨ����������⣬�𰸵�Ȼ�����ˣ�ֱ���ϴ��룺

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

# �ο�����
* [How to overwrite StringHttpMessageConverter DEFAULT_CHARSET to use UTF8 in spring 4](http://stackoverflow.com/questions/27606769/how-to-overwrite-stringhttpmessageconverter-default-charset-to-use-utf8-in-sprin)
* [Spring MVC UTF-8 Encoding](http://stackoverflow.com/questions/5928046/spring-mvc-utf-8-encoding)
* [Spring MVC Java config](http://stackoverflow.com/questions/22225040/spring-mvc-java-config)

