---
title: Spring 面试
---

+ Spring security 默认 http cache是disable的 。为静态资源添加设置cache 周期 
	+ 对静态资源开启cache 周期，如果要关闭的话，setCachedPeriod（-1）即为关闭缓存
	 
	<pre>
		@Configuration
public class MvcConfigurer extends WebMvcConfigurerAdapter
        implements EmbeddedServletContainerCustomizer {
 	   @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        // Resources without Spring Security. No cache control response headers.
        registry.addResourceHandler("/static/public/**")
            .addResourceLocations("classpath:/static/public/");

        // Resources controlled by Spring Security, which
        // adds "Cache-Control: must-revalidate".
        registry.addResourceHandler("/static/**")
            .addResourceLocations("classpath:/static/")
            .setCachePeriod(3600*24);
  	   }
}
</pre>、
 + 检查 request 的 last-modified的状态，如果没有被修改返回 null，返回null后 这里回去是交给父类处理，。。
<pre>
		long lastModified = // spec by app。
	    if (request.checkNotModified(lastModified)) {
        // 2. shortcut exit - no further processing necessary
        return null;
    }
</pre>




+ CGLIB 个JDK代理的区别：JDK 的动态代理是通过反射，代理的建立发生在编译器（其实是运行期），对动态代理类的字节码进行更改。JDK利用的是反射
 + cglib 实现的关键类`Enhancer` 织入器
+　**@RestController 和@Controller的区别**
 + @RestController注解相当于@ResponseBody和@Controller结合在一起使用。
 + 如果只是使用@RestController注解Controller，则Controller中的方法不经过InternelResourcesViewResolver。如果要返回View 例如JSP，返回一个ModeView对象可以
 + 如果需要返回到指定页面，则需要用 @Controller配合视图解析器InternalResourceViewResolver才行。
 + 如果需要返回JSON，XML或自定义mediaType内容到页面，则需要在对应的方法上加上@ResponseBody注解。