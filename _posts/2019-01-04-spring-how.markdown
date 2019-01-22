---
layout:     post
title:      "Spring源码分析（一）：从哪里开始看spring源码（系列文章基于Spring5.0）"
subtitle:   "How to start the study of Spring source code"
date:       2019-01-04 20:49:00
author:     "XYZ"
header-img: "img/article-bg1.jpg"
tags:
    - Spring源码分析
---

### 概述
* 对于大多数第一次看spring源码的人来说，都会感觉不知从哪开始看起，因为spring项目源码由多个子项目组成，如spring-beans，spring-context，spring-core，spring-aop，spring-web，spring-webmvc等，整个项目结构如图：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190118233901339.png)
* 可能有人会觉得，既然spring是一个IOC容器或者说是一个bean的容器，那么应该从spring-beans看起，先了解spring是如何从xml文件配置获取需要创建的bean的信息，但是这里有个问题就是虽然知道怎么遍历初始化，但是不知道哪里用到或者说哪里让这些初始化开始，而且像BeanFactory，FactoryBean，Environment，PropertySource等接口还是比较抽象的，比较难看懂，所以很容易让人感觉枯燥，然后就放弃了。
* 我们可以换个思路，从能接触到的角度开始，即我们通常会使用spring-mvc来进行web开发，如@Controller，@RequestMapping都是再熟悉不过的了。如果搭过spring-mvc项目都知道，通常需要在web.xml文件中，配置一个ContextLoaderListener，contextConfigLocation，DispatcherServlet，可能很多人都是从网上copy了一份配置过来或者知道contextConfigLocation是指定spring配置文件的位置，DispatcherServlet是接收所有请求的前端控制器，需要指定拦截路由：“/”，从而拦截所有URL中带“/”的请求，但是在spring源码中是怎么使用这些组件的呢？以及怎么配置了一个@Controller，@RequestMapping中指定了一个url，就可以访问了呢？还有就是通常我们的web项目都会部署在web容器，如tomcat当中，那么tomcat和spring有啥关系呢？所以我们可以带着这些问题去查看spring源码找到答案。
* 所以我推荐是从spring-mvc开始看spring源码，因为这个是我们使用得比较多，比较容易理解的一个模块，然后一层一层往上剥，找到与spring-context，spring-beans，spring-aop等的关系。如果真的对JavaWeb开发，Java EE很感兴趣，或者更容易读懂spring的源码，可以先看servlet规范和Tomcat的设计与Tomcat的请求处理工作流。我目前也在结合这两个方面看，也可以看下我的Tomcat源码分析系列。
### Servlet规范
* 在servlet的规范当中，servlet容器或者叫web容器，如tomcat，中运行的每个应用都由一个ServletContext表示，在web容器中可以包含多个ServletContext，即可以有多个web应用在web容器中运行。如在tomcat的webapp目录下，每个war包都对应一个web应用，tomcat启动时会解压war包，并启动相关的应用。
* 在web容器启动的时候，会初始化web应用，即创建ServletContext对象，加载解析web.xml文件，获取该应用的Filters，Listener，Servlet等组件的配置并创建对象实例，作为ServletContext的属性，保存在ServletContext当中。之后web容器接收到客户端请求时，则会根据请求信息，匹配到处理这个请求的Servlet，同时在交给servlet处理之前，会先使用应用配置的Filters对这个请求先进行过滤，最后才交给servlet处理。
* 了解web容器启动，之后接受客户端请求这些知识有啥用处呢？这里我们需要回过头来看我们的spring项目。我们在日常开发中，直接接触的是spring相关的组件，然后打成war包，放到web容器中，如拷贝到tomcat的webapp目录，并不会直接和web容器打交道。经过以上的分析，其实一个spring项目就是对应web容器里的一个ServletContext，所以在ServletContext对象的创建和初始化的时候，就需要一种机制来触发spring相关组件的创建和初始化，如包含@Controller和@RequestMapping注解的类和方法，这样才能处理请求。
### Listener监听器机制：ContextLoaderListener
* servlet规范当中，使用了Listener监听器机制来进行web容器相关组件的生命周期管理以及Event事件监听器来实现组件之间的交互。
* 其中一个重要的生命周期监听器是ServletContextListener。web容器在创建和初始化ServletContext的时候，会产生一个ServletContextEvent事件，其中ServletContextEvent包含该ServletContext的引用。然后交给在web.xml中配置的，注册到这个ServletContext的监听器ServletContextListener。ServletContextListener在其contextInitialized方法中定义处理逻辑，接口定义如下：
	```java
	/**
	 * Implementations of this interface receive notifications about changes to the
	 * servlet context of the web application they are part of. To receive
	 * notification events, the implementation class must be configured in the
	 * deployment descriptor for the web application.
	 *
	 * @see ServletContextEvent
	 * @since v 2.3
	 */
	public interface ServletContextListener extends EventListener {
	
	    /**
	     ** Notification that the web application initialization process is starting.
	     * All ServletContextListeners are notified of context initialization before
	     * any filter or servlet in the web application is initialized.
	     * @param sce Information about the ServletContext that was initialized
	     */
	    public void contextInitialized(ServletContextEvent sce);
	
	    /**
	     ** Notification that the servlet context is about to be shut down. All
	     * servlets and filters have been destroy()ed before any
	     * ServletContextListeners are notified of context destruction.
	     * @param sce Information about the ServletContext that was destroyed
	     */
	    public void contextDestroyed(ServletContextEvent sce);
	}
	```
	从contextInitialized的注释可知：通知所有的ServletContextListeners，当前的web应用正在启动，而且这些ServletContextListeners是在Filters和Servlets创建之前接收到通知的。所以在这个时候，web应用还不能接收请求，故可以在这里完成底层处理请求的组件的加载，这样等之后接收请求的Filters和Servlets创建时，则可以使用这些创建好的组件了。spring相关的bean就是这里所说的底层处理请求的组件，如数据库连接池，数据库事务管理器等。
* ContextLoaderListener：spring-web包的ContextLoaderListener就是一个ServletContextListener的实现类。ContextLoaderListener主要用来获取spring项目的整体配置信息，并创建对应的WebApplicationContext来保存bean的信息，以及创建这些bean的对象实例。默认去WEB-INF下加载applicationContext.xml配置，如果applicationContext.xml放在其他位置，或者使用其他不同的名称，或者使用多个xml文件，则与指定contextConfigLocation。具体spring源码的实现过程后续文章详细分析。
	```xml
	<listener>
	    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
	</listener>
	<!-- 修改配置文件路径 -->
	<context-param>
	    <param-name>contextConfigLocation</param-name>
	    <param-value>classpath:spring/applicationContext.xml</param-value>
	</context-param>
	```

### DispatcherServlet：前端控制器
* 在web容器中，web.xml中的加载顺序：context-param -> listener -> filter -> servlet。其中ContextLoaderListener是属于listener阶段。我们通常需要在项目的web.xml中配置一个DispatcherServlet，并配置拦截包含“/”路径的请求，即拦截所有请求。这样在web容器启动应用时，在servlet阶段会创建这个servlet，由Servlet规范中servlet的生命周期方法可知：
	```java
	public interface Servlet {
	    void init(ServletConfig var1) throws ServletException;
	
	    ServletConfig getServletConfig();
	
	    void service(ServletRequest var1, ServletResponse var2) throws ServletException, IOException;
	
	    String getServletInfo();
	
	    void destroy();
	}
	```
	web容器在创建这个servlet的时候，会调用其init方法，故可以在DispatcherServlet的init方法中定义初始化逻辑，核心实现了创建DispatcherServlet自身的一个WebApplicationContext，注意在spring中每个servlet可以包含一个独立的WebApplicationContext来维护自身的组件，而上面通过ContextLoaderListener创建的WebApplicationContext为共有的，通常也是最顶层，即root WebApplicationContext，servlet的WebApplicationContext可以通过setParent方法设值到自身的一个属性。DispatcherServlet默认是加载WEB-INF下面的“servletName”-servlet.xml，来获取配置信息的，也可以与ContextLoaderListener一样通过contextLoaderConfig来指定位置。DispatcherServlet具体的源码设计在之后文章详细分析。

### 总结
* 从上面的分析，可知spring相关配置解析和组件创建其实是在web容器中，启动一个web应用的时候，即在其ServletContext组件创建的时候，首先解析web.xml获取该应用配置的listeners列表和servlet列表，然后保存在自身的一个属性中，然后通过分发生命周期事件ServletContextEvent给这些listeners，从而在listeners感知到应用在启动了，然后自定义自身的处理逻辑，如spring的ContextLoaderListener就是解析spring的配置文件并创建相关的bean，这样其实也是实现了一种代码的解耦；其次是创建配置的servlet列表，调用servlet的init方法，这样servlet可以自定义初始化逻辑，DispatcherServlet就是其中一个servlet。
* 所以在ContextLoaderListener和DispatcherServlet的创建时，都会进行WebApplicationContext的创建，这里其实就是IOC容器的创建了，即会交给spring-context，spring-beans包相关的类进行处理了，故可以从这里作为一个入口，一层一层地剥spring的源码了。
