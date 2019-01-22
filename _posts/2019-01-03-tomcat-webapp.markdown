---
layout:     post
title:      "Tomcat源码分析（三）：ServletContext应用启动之配置解析"
subtitle:   "Tomcat servletcontext startup"
date:       2019-01-03 08:12:00
author:     "XYZ"
header-img: "img/article-bg1.jpg"
tags:
    - Tomcat源码分析
---

### 概述
* 这篇是接我上篇文章：[Tomcat源码分析（二）：启动流程和webapp目录应用部署实现](https://blog.csdn.net/u010013573/article/details/86407922)，主要来分析一下tomcat启动时，创建和初始化应用的ServletContext的详细过程。
* 我们知道在servlet规范设计当中，每个应用在servlet容器中，如tomcat，是使用一个servletContext来代表的，即servletContext包含了该应用的相关配置，如session配置信息，sercurity相关的authentication，constraints，过滤器filters，用来处理客户端http请求的servlet集合，以及与url的映射关系等。如果应用使用spring框架，还有需要配置spring对servlet容器的启动监听器ContextLoaderListener，请求拦截器DispatchServlet等。
### web.xml和JavaConfig
#### web.xml配置文件配置
* ServeltContext的配置信息，一般可以通过在web.xml文件中配置，tomcat在加载应用的时候会查找并解析web.xml，将相关信息填充到ServeltContext中，一个典型的web.xml配置如下：

```xml
出自：https://blog.csdn.net/u010796790/article/details/52098258

<?xml version="1.0" encoding="UTF-8"?>  
<web-app version="3.0" xmlns="http://java.sun.com/xml/ns/javaee"  
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"  
         xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd">  

    <!-- 在Spring框架中是如何解决从页面传来的字符串的编码问题的呢？
    下面我们来看看Spring框架给我们提供过滤器CharacterEncodingFilter  
     这个过滤器就是针对于每次浏览器请求进行过滤的，
     然后再其之上添加了父类没有的功能即处理字符编码。  
      其中encoding用来设置编码格式，forceEncoding用来设置是否理会 request.getCharacterEncoding()方法，设置为true则强制覆盖之前的编码格式。-->  
    <filter>  
        <filter-name>characterEncodingFilter</filter-name>  
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
        <filter-name>characterEncodingFilter</filter-name>  
        <url-pattern>/*</url-pattern>  
    </filter-mapping>  
    
    <!-- 项目中使用Spring 时，applicationContext.xml配置文件中并没有BeanFactory，
    要想在业务层中的class 文件中直接引用Spring容器管理的bean可通过以下方式-->  
    <!--1、在web.xml配置监听器ContextLoaderListener-->  
    <!--ContextLoaderListener的作用就是启动Web容器时，
    自动装配ApplicationContext的配置信息。
    因为它实现了ServletContextListener这个接口，
    在web.xml配置这个监听器，启动容器时，就会默认执行它实现的方法。  
    在ContextLoaderListener中关联了ContextLoader这个类，
    所以整个加载配置过程由ContextLoader来完成。  
    它的API说明  
    第一段说明ContextLoader可以由 ContextLoaderListener和ContextLoaderServlet生成。  
    如果查看ContextLoaderServlet的API，
    可以看到它也关联了ContextLoader这个类而且它实现了HttpServlet这个接口  
    第二段，ContextLoader创建的是 XmlWebApplicationContext这样一个类，
    它实现的接口是WebApplicationContext->ConfigurableWebApplicationContext->ApplicationContext->  
    BeanFactory这样一来spring中的所有bean都由这个类来创建  
     IUploaddatafileManager uploadmanager = (IUploaddatafileManager)    ContextLoaderListener.getCurrentWebApplicationContext().getBean("uploadManager");
     -->  
     
    <listener>  
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>  
    </listener>  
    
    <!--2、部署applicationContext的xml文件-->  
    <!--如果在web.xml中不写任何参数配置信息，默认的路径是"/WEB-INF/applicationContext.xml，  
    在WEB-INF目录下创建的xml文件的名称必须是applicationContext.xml。  
    如果是要自定义文件名可以在web.xml里加入contextConfigLocation这个context参数：  
    在<param-value> </param-value>里指定相应的xml文件名，如果有多个xml文件，
    可以写在一起并以“,”号分隔。  
    也可以这样applicationContext-*.xml采用通配符，
    比如这那个目录下有applicationContext-ibatis-base.xml，  
    applicationContext-action.xml，applicationContext-ibatis-dao.xml等文件，
    都会一同被载入。在ContextLoaderListener中关联了ContextLoader这个类，
    所以整个加载配置过程由ContextLoader来完成。-->  
    
    <context-param>  
        <param-name>contextConfigLocation</param-name>  
        <param-value>classpath:spring/applicationContext.xml</param-value>  
    </context-param>  
    
    <!--使用Spring MVC,配置DispatcherServlet是第一步。DispatcherServlet是一个Servlet,,所以可以配置多个DispatcherServlet-->  
    <!--DispatcherServlet是前置控制器，配置在web.xml文件中的。拦截匹配的请求，Servlet拦截匹配规则要自已定义，把拦截下来的请求，依据某某规则分发到目标Controller(我们写的Action)来处理。-->  
    
    <servlet>  
        <servlet-name>DispatcherServlet</servlet-name>

		<!--在DispatcherServlet的初始化过程中，框架会在web应用的 WEB-INF文件夹下寻找名为[servlet-name]-servlet.xml 的配置文件，生成文件中定义的bean。-->  
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>  
        <!--指明了配置文件的文件名，不使用默认配置文件名，而使用dispatcher-servlet.xml配置文件。-->  
        
        <init-param>  
            <param-name>contextConfigLocation</param-name>  
            
            <!--其中<param-value>**.xml</param-value> 这里可以使用多种写法-->  
            <!--1、不写,使用默认值:/WEB-INF/<servlet-name>-servlet.xml-->  
            <!--2、<param-value>/WEB-INF/classes/dispatcher-servlet.xml</param-value>-->  
            <!--3、<param-value>classpath*:dispatcher-servlet.xml</param-value>-->  
            <!--4、多个值用逗号分隔-->  
            
            <param-value>classpath:spring/dispatcher-servlet.xml</param-value>  
        </init-param>  
        <load-on-startup>1</load-on-startup>

		<!--是启动顺序，让这个Servlet随Servletp容器一起启动。-->  
		
    </servlet>  
    <servlet-mapping>  
    
        <!--这个Servlet的名字是dispatcher，可以有多个DispatcherServlet，是通过名字来区分的。每一个DispatcherServlet有自己的WebApplicationContext上下文对象。同时保存的ServletContext中和Request对象中.-->  
        <!--ApplicationContext是Spring的核心，Context我们通常解释为上下文环境，我想用“容器”来表述它更容易理解一些，ApplicationContext则是“应用的容器”了:P，Spring把Bean放在这个容器中，在需要的时候，用getBean方法取出-->  
        
        <servlet-name>DispatcherServlet</servlet-name>  
        
        <!--Servlet拦截匹配规则可以自已定义，当映射为@RequestMapping("/user/add")时，为例,拦截哪种URL合适？-->  
        <!--1、拦截*.do、*.htm， 例如：/user/add.do,这是最传统的方式，最简单也最实用。不会导致静态文件（jpg,js,css）被拦截。-->  
        <!--2、拦截/，例如：/user/add,可以实现现在很流行的REST风格。很多互联网类型的应用很喜欢这种风格的URL。弊端：会导致静态文件（jpg,js,css）被拦截后不能正常显示。 -->  
        
        <url-pattern>/</url-pattern> <!--会拦截URL中带“/”的请求。-->  
    </servlet-mapping>  

    <!--如果你的DispatcherServlet拦截"/"，为了实现REST风格，拦截了所有的请求，那么同时对*.js,*.jpg等静态文件的访问也就被拦截了。-->  
    <!--方案一：激活Tomcat的defaultServlet来处理静态文件-->  
    <!--要写在DispatcherServlet的前面， 让 defaultServlet先拦截请求，这样请求就不会进入Spring了，我想性能是最好的吧。-->  
    
    <servlet-mapping>  
        <servlet-name>default</servlet-name>  
        <url-pattern>*.css</url-pattern>  
    </servlet-mapping>  
    <servlet-mapping>  
        <servlet-name>default</servlet-name>  
        <url-pattern>*.swf</url-pattern>  
    </servlet-mapping>  
    <servlet-mapping>  
        <servlet-name>default</servlet-name>  
        <url-pattern>*.gif</url-pattern>  
    </servlet-mapping>  
    <servlet-mapping>  
        <servlet-name>default</servlet-name>  
        <url-pattern>*.jpg</url-pattern>  
    </servlet-mapping>  
    <servlet-mapping>  
        <servlet-name>default</servlet-name>  
        <url-pattern>*.png</url-pattern>  
    </servlet-mapping>  
    <servlet-mapping>  
        <servlet-name>default</servlet-name>  
        <url-pattern>*.js</url-pattern>  
    </servlet-mapping>  
    <servlet-mapping>  
        <servlet-name>default</servlet-name>  
        <url-pattern>*.html</url-pattern>  
    </servlet-mapping>  
    <servlet-mapping>  
        <servlet-name>default</servlet-name>  
        <url-pattern>*.xml</url-pattern>  
    </servlet-mapping>  
    <servlet-mapping>  
        <servlet-name>default</servlet-name>  
        <url-pattern>*.json</url-pattern>  
    </servlet-mapping>  
    <servlet-mapping>  
        <servlet-name>default</servlet-name>  
        <url-pattern>*.map</url-pattern>  
    </servlet-mapping>  

    <welcome-file-list><!--指定欢迎页面-->  
        <welcome-file>login.html</welcome-file>  
    </welcome-file-list>  
    <error-page> <!--当系统出现404错误，跳转到页面nopage.html-->  
        <error-code>404</error-code>  
        <location>/nopage.html</location>  
    </error-page>  
    <error-page> <!--当系统出现java.lang.NullPointerException，跳转到页面error.html-->  
        <exception-type>java.lang.NullPointerException</exception-type>  
        <location>/error.html</location>  
    </error-page>  
    
    <session-config><!--会话超时配置，单位分钟-->  
        <session-timeout>360</session-timeout>  
    </session-config>  
</web-app>

```
#### JavaConfig编程方式配置
* 在servlet 3.0及以上版本，除了可以通过web.xml方式来配置，还可以通过基于Java类的方式来配置，即JavaConfig机制，从而省去web.xml文件。具体为Servlet 3.0提供了一个**ServletContainerInitializer**接口来支持这种编程方式配置的实现，接口定义如下：

```java
package javax.servlet;

import java.util.Set;

/**
 * ServletContainerInitializers (SCIs) are registered via an entry in the
 * file META-INF/services/javax.servlet.ServletContainerInitializer that must be
 * included in the JAR file that contains the SCI implementation.
 * <p>
 * SCI processing is performed regardless of the setting of metadata-complete.
 * SCI processing can be controlled per JAR file via fragment ordering. If an
 * absolute ordering is defined, the only those JARs included in the ordering
 * will be processed for SCIs. To disable SCI processing completely, an empty
 * absolute ordering may be defined.
 * <p>
 * SCIs register an interest in annotations (class, method or field) and/or
 * types via the {@link javax.servlet.annotation.HandlesTypes} annotation which
 * is added to the class.
 *
 * @since Servlet 3.0
 */
public interface ServletContainerInitializer {

    /**
     * Receives notification during startup of a web application of the classes
     * within the web application that matched the criteria defined via the
     * {@link javax.servlet.annotation.HandlesTypes} annotation.
     *
     * @param c     The (possibly null) set of classes that met the specified
     *              criteria
     * @param ctx   The ServletContext of the web application in which the
     *              classes were discovered
     *
     * @throws ServletException If an error occurs
     */
    void onStartup(Set<Class<?>> c, ServletContext ctx) throws ServletException;
}
```
   1. ServletContainerInitializer接口的说明大概意思是说，ServletContainerInitializer接口的实现类需要实现SPI机制，即将ServletContainerInitializer接口的实现类全限定名放到META-INF/services/javax.servlet.ServletContainerInitializer文件内，这样servlet容器启动的时候，在加载这个jar的时候，就会实例化META-INF/services/javax.servlet.ServletContainerInitializer列举的类。
   2.  @HandlesTypes：可以在ServletContainerInitializer接口实现类，加上@HandlesTypes，通过在@HandlesTypes注解中，包含对ServeltContext，即应用启动感兴趣的接口，这样在应用启动过程中，则会通知到这些接口的实现类，这些接口实现类可以获取到ServletContext对象，从而可以调用ServletContext的方法对ServletContext填充数据，如addServlet方法来添加Spring的Dispatcher。

* spring对ServletContainerInitializer的实现：主要是定义了一个WebApplicationInitializer接口。onStartup方法的webAppInitializerClasses参数就是WebApplicationInitializer接口的实现类。
```java
@HandlesTypes(WebApplicationInitializer.class)
public class SpringServletContainerInitializer implements ServletContainerInitializer {
   @Override
	public void onStartup(@Nullable Set<Class<?>> webAppInitializerClasses, ServletContext servletContext)
			throws ServletException {
		List<WebApplicationInitializer> initializers = new LinkedList<>();

		if (webAppInitializerClasses != null) {

            // 排除接口或者抽象类，只处理具体的WebApplicationInitializer实现类
            
			for (Class<?> waiClass : webAppInitializerClasses) {
				// Be defensive: Some servlet containers provide us with invalid classes,
				// no matter what @HandlesTypes says...
				if (!waiClass.isInterface() && !Modifier.isAbstract(waiClass.getModifiers()) &&
						WebApplicationInitializer.class.isAssignableFrom(waiClass)) {
					try {
						initializers.add((WebApplicationInitializer)
								ReflectionUtils.accessibleConstructor(waiClass).newInstance());
					}
					catch (Throwable ex) {
						throw new ServletException("Failed to instantiate WebApplicationInitializer class", ex);
					}
				}
			}
		}

		if (initializers.isEmpty()) {
			servletContext.log("No Spring WebApplicationInitializer types detected on classpath");
			return;
		}
		servletContext.log(initializers.size() + " Spring WebApplicationInitializers detected on classpath");
		
		// 排序
		AnnotationAwareOrderComparator.sort(initializers);
		for (WebApplicationInitializer initializer : initializers) {
		
		    // 将servletContext作为参数，调用WebApplicationInitializer实现类的onStartup方法
		    // 在onStartup方法中，定义具体的处理逻辑
			initializer.onStartup(servletContext);
		}
	}
}
```
* 典型用法：使用一个WebApplicationInitializer接口实现类替换web.xml，如下添加了一个DispatchServlet，与上面在web.xml中，通过servlet标签添加的一样。

```java
public class MyWebAppInitializer implements WebApplicationInitializer {
     @Override
     public void onStartup(ServletContext container) {
       XmlWebApplicationContext appContext = new XmlWebApplicationContext();
       
       // contextConfigLocation
       appContext.setConfigLocation("/WEB-INF/spring/applicationContext.xml");
 		
 		// DispatcherServlet
       ServletRegistration.Dynamic dispatcher =
         container.addServlet("dispatcher", new DispatcherServlet(appContext));
         
       //  load-on-start
       dispatcher.setLoadOnStartup(1);
       
       // servlet-mapping
       dispatcher.addMapping("/");
     }
  }
```
* 以上例子还是需要一个applicationContext.xml，如果想完全通过Java的方式来配置，可以结合spring的@Conifguration注解，以及使用AnnotationConfigWebApplicationContext来加载spring的context，来实现：

```java
import lombok.Data;
import org.springframework.stereotype.Component;

/**
 * @author xieyizun
 * @date 13/1/2019 15:15
 * @description:
 */
@Component
@Data
public class User {
    private String name;
}
```
通过@ComponentScan来进行包扫描：
```java
/**
 * @author xieyizun
 * @date 13/1/2019 15:10
 * @description:
 */
@Slf4j
@Configuration
@ComponentScan(basePackages = "model")
public class MyConfig {
    public MyConfig() {
        log.info("spring init");
    }
}

/**
 * @author xieyizun
 * @date 13/1/2019 15:28
 * @description:
 */
@Configuration
@EnableWebMvc
@ComponentScan("com.yzxie.demo.controller")
public class WebConfig extends WebMvcConfigurationSupport {
    @Bean
    public ViewResolver viewResolver(){
        InternalResourceViewResolver viewResolver = new InternalResourceViewResolver();
        viewResolver.setPrefix("/WEB-INF/views/");
        viewResolver.setSuffix(".jsp");
        return viewResolver;
    }
}
```

```java
public class MyWebAppInitializer implements WebApplicationInitializer {
    @Override
    public void onStartup(ServletContext container) {
      // spring容器
      AnnotationConfigWebApplicationContext rootContext =
        new AnnotationConfigWebApplicationContext();
      rootContext.register(MyConfig.class);
      container.addListener(new ContextLoaderListener(rootContext));
      
       // 注册dispatchServlet
      AnnotationConfigWebApplicationContext dispatcherContext =
        new AnnotationConfigWebApplicationContext();
      dispatcherContext.register(WebConfig.class);
   
      ServletRegistration.Dynamic dispatcher =
        container.addServlet("dispatcher", new DispatcherServlet(dispatcherContext));
      dispatcher.setLoadOnStartup(1);
      dispatcher.addMapping("/");
    }
 }
```
### tomcat对web.xml的解析和ServletContainerInitializer的处理
* 以上也算是对web.xml和JavaConfig的使用做了一个复习吧，接下来结合tomcat源码来分析以上用法的底层实现原理，主要是在org.apache.catalina.startup.ContextConfig的webConfig方法中实现。调用时机为：StandardContext在启动时，调用startInternal方法，在方法内执行：
fireLifecycleEvent(Lifecycle.CONFIGURE_START_EVENT, null);，通知LifeCycleListener，而ContextConfig是其中一个LifeCycleListener。
* 核心注释：

```java
/*
  * Anything and everything can override the global and host defaults.
  * This is implemented in two parts
  * - Handle as a web fragment that gets added after everything else so
  *   everything else takes priority
  * - Mark Servlets as overridable so SCI configuration can replace
  *   configuration from the defaults
  */

 /*
  * The rules for annotation scanning are not as clear-cut as one might
  * think. Tomcat implements the following process:
  * - As per SRV.1.6.2, Tomcat will scan for annotations regardless of
  *   which Servlet spec version is declared in web.xml. The EG has
  *   confirmed this is the expected behaviour.
  * - As per http://java.net/jira/browse/SERVLET_SPEC-36, if the main
  *   web.xml is marked as metadata-complete, JARs are still processed
  *   for SCIs.
  * - If metadata-complete=true and an absolute ordering is specified,
  *   JARs excluded from the ordering are also excluded from the SCI
  *   processing.
  * - If an SCI has a @HandlesType annotation then all classes (except
  *   those in JARs excluded from an absolute ordering) need to be
  *   scanned to check if they match.
  */
```
#### webConfig方法的核心步骤

0. WebXmlParser和WebXml对象的创建

```java
WebXmlParser webXmlParser = new WebXmlParser(context.getXmlNamespaceAware(),
                context.getXmlValidation(), context.getXmlBlockExternal());

Set<WebXml> defaults = new HashSet<>();
defaults.add(getDefaultWebXmlFragment(webXmlParser));

WebXml webXml = createWebXml();

// Parse context level web.xml
InputSource contextWebXml = getContextWebXmlSource();
if (!webXmlParser.parseWebXml(contextWebXml, webXml, false)) {
    ok = false;
}

ServletContext sContext = context.getServletContext();
```
1. 从各个jar包中查找web-fragment.xml，并根据这些web-fragment.xml中定义的规则排序：

```java
// Ordering is important here

// Step 1. Identify all the JARs packaged with the application and those
// provided by the container. If any of the application JARs have a
// web-fragment.xml it will be parsed at this point. web-fragment.xml
// files are ignored for container provided JARs.
Map<String,WebXml> fragments = processJarsForWebFragments(webXml, webXmlParser);

// Step 2. Order the fragments.
Set<WebXml> orderedFragments = null;
orderedFragments =
        WebXml.orderWebFragments(webXml, fragments, sContext);
```
2. 查找ServletContainerInitializer接口的实现，从/WEB-INF/classes和各个jar包中查找@HandlesTypes指定的接口的实现，如spring的**WebApplicationInitializer**，

```java
// Step 3. Look for ServletContainerInitializer implementations
if (ok) {
    processServletContainerInitializers();
}

if  (!webXml.isMetadataComplete() || typeInitializerMap.size() > 0) {
    // Step 4. Process /WEB-INF/classes for annotations and
    // @HandlesTypes matches
    Map<String,JavaClassCacheEntry> javaClassCache = new HashMap<>();

    if (ok) {
        WebResource[] webResources =
                context.getResources().listResources("/WEB-INF/classes");

        for (WebResource webResource : webResources) {
            // Skip the META-INF directory from any JARs that have been
            // expanded in to WEB-INF/classes (sometimes IDEs do this).
            if ("META-INF".equals(webResource.getName())) {
                continue;
            }
            processAnnotationsWebResource(webResource, webXml,
                    webXml.isMetadataComplete(), javaClassCache);
        }
    }

    // Step 5. Process JARs for annotations and
    // @HandlesTypes matches - only need to process those fragments we
    // are going to use (remember orderedFragments includes any
    // container fragments)
    if (ok) {
        processAnnotations(
                orderedFragments, webXml.isMetadataComplete(), javaClassCache);
    }

    // Cache, if used, is no longer required so clear it
    javaClassCache.clear();
}
```
3. 将web-fragment.xml合并到web.xml中，此时是完整的web.xml了，故通过configureContext方法，将web.xml中配置信息添加到StandardContext中。

```java
if (!webXml.isMetadataComplete()) {
  // Step 6. Merge web-fragment.xml files into the main web.xml
    // file.
    if (ok) {
        ok = webXml.merge(orderedFragments);
    }

    // Step 7. Apply global defaults
    // Have to merge defaults before JSP conversion since defaults
    // provide JSP servlet definition.
    webXml.merge(defaults);

    // Step 8. Convert explicitly mentioned jsps to servlets
    if (ok) {
        convertJsps(webXml);
    }

    // Step 9. Apply merged web.xml to Context
    if (ok) {
        configureContext(webXml);
    }
} else {
    webXml.merge(defaults);
    convertJsps(webXml);
    configureContext(webXml);
}

// configureContext的实现：主要是从webxml中取出各种信息，然后填充到StandardContext中。
private void configureContext(WebXml webxml) {
    ...

    for (Entry<String, String> entry : webxml.getContextParams().entrySet()) {
        context.addParameter(entry.getKey(), entry.getValue());
    }
    
    ...
    
    for (ErrorPage errorPage : webxml.getErrorPages().values()) {
        context.addErrorPage(errorPage);
    }
    for (FilterDef filter : webxml.getFilters().values()) {
        if (filter.getAsyncSupported() == null) {
            filter.setAsyncSupported("false");
        }
        context.addFilterDef(filter);
    }
    for (FilterMap filterMap : webxml.getFilterMappings()) {
        context.addFilterMap(filterMap);
    }
    context.setJspConfigDescriptor(webxml.getJspConfigDescriptor());
    for (String listener : webxml.getListeners()) {
        context.addApplicationListener(listener);
    }
    for (Entry<String, String> entry :
            webxml.getLocaleEncodingMappings().entrySet()) {
        context.addLocaleEncodingMappingParameter(entry.getKey(),
                entry.getValue());
    }
    // Prevents IAE
    if (webxml.getLoginConfig() != null) {
        context.setLoginConfig(webxml.getLoginConfig());
    }
    for (MessageDestinationRef mdr :
            webxml.getMessageDestinationRefs().values()) {
        context.getNamingResources().addMessageDestinationRef(mdr);
    }

    // messageDestinations were ignored in Tomcat 6, so ignore here

    context.setIgnoreAnnotations(webxml.isMetadataComplete());
    for (Entry<String, String> entry :
            webxml.getMimeMappings().entrySet()) {
        context.addMimeMapping(entry.getKey(), entry.getValue());
    }
    ...
    for (ContextResource resource : webxml.getResourceRefs().values()) {
        context.getNamingResources().addResource(resource);
    }
    boolean allAuthenticatedUsersIsAppRole =
            webxml.getSecurityRoles().contains(
                    SecurityConstraint.ROLE_ALL_AUTHENTICATED_USERS);
    for (SecurityConstraint constraint : webxml.getSecurityConstraints()) {
        if (allAuthenticatedUsersIsAppRole) {
            constraint.treatAllAuthenticatedUsersAsApplicationRole();
        }
        context.addConstraint(constraint);
    }
    for (String role : webxml.getSecurityRoles()) {
        context.addSecurityRole(role);
    }
    for (ContextService service : webxml.getServiceRefs().values()) {
        context.getNamingResources().addService(service);
    }
    for (ServletDef servlet : webxml.getServlets().values()) {
        Wrapper wrapper = context.createWrapper();
        // Description is ignored
        // Display name is ignored
        // Icons are ignored

        // jsp-file gets passed to the JSP Servlet as an init-param

        if (servlet.getLoadOnStartup() != null) {
            wrapper.setLoadOnStartup(servlet.getLoadOnStartup().intValue());
        }
        if (servlet.getEnabled() != null) {
            wrapper.setEnabled(servlet.getEnabled().booleanValue());
        }
        wrapper.setName(servlet.getServletName());
        Map<String,String> params = servlet.getParameterMap();
        for (Entry<String, String> entry : params.entrySet()) {
            wrapper.addInitParameter(entry.getKey(), entry.getValue());
        }
        wrapper.setRunAs(servlet.getRunAs());
        Set<SecurityRoleRef> roleRefs = servlet.getSecurityRoleRefs();
        for (SecurityRoleRef roleRef : roleRefs) {
            wrapper.addSecurityReference(
                    roleRef.getName(), roleRef.getLink());
        }
        wrapper.setServletClass(servlet.getServletClass());
        MultipartDef multipartdef = servlet.getMultipartDef();
        if (multipartdef != null) {
            if (multipartdef.getMaxFileSize() != null &&
                    multipartdef.getMaxRequestSize()!= null &&
                    multipartdef.getFileSizeThreshold() != null) {
                wrapper.setMultipartConfigElement(new MultipartConfigElement(
                        multipartdef.getLocation(),
                        Long.parseLong(multipartdef.getMaxFileSize()),
                        Long.parseLong(multipartdef.getMaxRequestSize()),
                        Integer.parseInt(
                                multipartdef.getFileSizeThreshold())));
            } else {
                wrapper.setMultipartConfigElement(new MultipartConfigElement(
                        multipartdef.getLocation()));
            }
        }
        if (servlet.getAsyncSupported() != null) {
            wrapper.setAsyncSupported(
                    servlet.getAsyncSupported().booleanValue());
        }
        wrapper.setOverridable(servlet.isOverridable());
        context.addChild(wrapper);
    }
    for (Entry<String, String> entry :
            webxml.getServletMappings().entrySet()) {
        context.addServletMappingDecoded(entry.getKey(), entry.getValue());
    }
    SessionConfig sessionConfig = webxml.getSessionConfig();
    if (sessionConfig != null) {
        if (sessionConfig.getSessionTimeout() != null) {
            context.setSessionTimeout(
                    sessionConfig.getSessionTimeout().intValue());
        }
        SessionCookieConfig scc =
            context.getServletContext().getSessionCookieConfig();
        scc.setName(sessionConfig.getCookieName());
        scc.setDomain(sessionConfig.getCookieDomain());
        scc.setPath(sessionConfig.getCookiePath());
        scc.setComment(sessionConfig.getCookieComment());
        if (sessionConfig.getCookieHttpOnly() != null) {
            scc.setHttpOnly(sessionConfig.getCookieHttpOnly().booleanValue());
        }
        if (sessionConfig.getCookieSecure() != null) {
            scc.setSecure(sessionConfig.getCookieSecure().booleanValue());
        }
        if (sessionConfig.getCookieMaxAge() != null) {
            scc.setMaxAge(sessionConfig.getCookieMaxAge().intValue());
        }
        if (sessionConfig.getSessionTrackingModes().size() > 0) {
            context.getServletContext().setSessionTrackingModes(
                    sessionConfig.getSessionTrackingModes());
        }
    }

    ...
  
}
```
4. 加载jar包里面的META-INF/resources的静态资源

```java
// Always need to look for static resources
// Step 10. Look for static resources packaged in JARs
if (ok) {
   // Spec does not define an order.
   // Use ordered JARs followed by remaining JARs
   Set<WebXml> resourceJars = new LinkedHashSet<>();
   for (WebXml fragment : orderedFragments) {
       resourceJars.add(fragment);
   }
   for (WebXml fragment : fragments.values()) {
       if (!resourceJars.contains(fragment)) {
           resourceJars.add(fragment);
       }
   }
   processResourceJARs(resourceJars);
   // See also StandardContext.resourcesStart() for
   // WEB-INF/classes/META-INF/resources configuration
}
```
5. 将查找到的ServletContainerInitializer的实现类对象，添加到StandardContext的initializers中，其中key为ServletContainerInitializer的接口实现类对象，value为
@HandlesTypes指定的接口的实现类对象的集合，这个之后会在StandardContext的startInternal方法的后续代码中，遍历并调用onStartup方法。

```java
/**
 * The ordered set of ServletContainerInitializers for this web application.
 */
private Map<ServletContainerInitializer,Set<Class<?>>> initializers =
    new LinkedHashMap<>();
```

```java
// Step 11. Apply the ServletContainerInitializer config to the
// context
if (ok) {
    for (Map.Entry<ServletContainerInitializer,
            Set<Class<?>>> entry :
                initializerClassMap.entrySet()) {
        if (entry.getValue().isEmpty()) {
            context.addServletContainerInitializer(
                    entry.getKey(), null);
        } else {
            context.addServletContainerInitializer(
                    entry.getKey(), entry.getValue());
        }
    }
}
```
6. StandardContext的startInternal方法等listener列表执行完返回后继续往下执行，其中上面的ContextConfig为其中一个listener。在以下代码执行ServletContainerInitializer的处理，可以看到是在初始化filter和实例化（配置了load-on-startup）servlet之前执行。
*   遍历ServletContainerInitializer列表，其中entry.getKey()为：ServletContainerInitializer；entry.getValue()对应@HandlerType对应的类实例列表。在spring中@HandlerType的实现接口为WebApplicationInitializer，调用WebApplicationInitializer的onStartup方法。

```java
// Set up the context init params
mergeParameters();

// 遍历ServletContainerInitializer列表，
// entry.getValue()对应@HandlerType对应的类实例，并调用其onStartup方法
// Call ServletContainerInitializers
 for (Map.Entry<ServletContainerInitializer, Set<Class<?>>> entry :
     initializers.entrySet()) {
     try {
         entry.getKey().onStartup(entry.getValue(),
                 getServletContext());
     } catch (ServletException e) {
         log.error(sm.getString("standardContext.sciFail"), e);
         ok = false;
         break;
     }
 }

...
// 初始化过滤器filter
 // Configure and call application filters
 if (ok) {
     if (!filterStart()) {
         log.error(sm.getString("standardContext.filterFail"));
         ok = false;
     }
 }

 // Load and initialize all "load on startup" servlets
 if (ok) {
     if (!loadOnStartup(findChildren())){
         log.error(sm.getString("standardContext.servletFail"));
         ok = false;
     }
 }
```