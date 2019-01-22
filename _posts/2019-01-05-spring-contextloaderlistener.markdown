---
layout:     post
title:      "Spring源码分析（二）：ContextLoaderListener的设计与实现"
subtitle:   "Spring ContextLoaderListener"
date:       2019-01-05 20:49:00
author:     "XYZ"
header-img: "img/article-bg1.jpg"
tags:
    - Spring源码分析
---

### ContextLoaderListener
* 在我的上一篇文章：[Spring源码分析（一）：从哪里开始看spring源码](https://blog.csdn.net/u010013573/article/details/86547687)，分析过在web容器，如tomcat，启动web应用时，会通过监听器的方式，通知ServletContextListener，web容器开始启动web应用了，ServletContextListener可以自定义初始化逻辑。ContextLoaderListener就是ServletContextListener接口的一个实现类，主要负责加载spring主容器相关的bean，即默认WEB-INF/applicationContext.xml文件的配置信息。
* ContextLoaderListener通过实现ServletContextListener接口，将spring容器融入web容器当中。这个可以分两个角度来理解：
   1. web项目自身：接收web容器启动web应用的通知，开始自身配置的解析加载，创建bean实例，通过一个WebApplicationContext来维护spring项目的主容器相关的bean，以及其他一些组件。
   2. web容器：web容器使用ServletContext来维护每一个web应用，ContextLoaderListener将spring容器，即WebApplicationContext，作为ServletContext的一个attribute，key为，保存在ServletContext中，从而web容器和spring项目可以通过ServletContext来交互。
* ContextLoaderListener只是作为一个中间层来建立spring容器和web容器的关联关系，而实际完成以上两个角度的工作是通过ContextLoader来进行的，即在ContextLoader中定义以上逻辑，ContextLoaderListener的实现如下：

	```java
	package org.springframework.web.context;
	**
	 * Bootstrap listener to start up and shut down Spring's root {@link WebApplicationContext}.
	 * Simply delegates to {@link ContextLoader} as well as to {@link ContextCleanupListener}.
	 *
	 * <p>As of Spring 3.1, {@code ContextLoaderListener} supports injecting the root web
	 * application context via the {@link #ContextLoaderListener(WebApplicationContext)}
	 * constructor, allowing for programmatic configuration in Servlet 3.0+ environments.
	 * See {@link org.springframework.web.WebApplicationInitializer} for usage examples.
	 */
	public class ContextLoaderListener extends ContextLoader implements ServletContextListener {
		/**
		 * Create a new {@code ContextLoaderListener} that will create a web application
		 * context based on the "contextClass" and "contextConfigLocation" servlet
		 * context-params. See {@link ContextLoader} superclass documentation for details on
		 * default values for each.
		 * <p>This constructor is typically used when declaring {@code ContextLoaderListener}
		 * as a {@code <listener>} within {@code web.xml}, where a no-arg constructor is
		 * required.
		 * <p>The created application context will be registered into the ServletContext under
		 * the attribute name {@link WebApplicationContext#ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE}
		 * and the Spring application context will be closed when the {@link #contextDestroyed}
		 * lifecycle method is invoked on this listener.
		 * @see ContextLoader
		 * @see #ContextLoaderListener(WebApplicationContext)
		 * @see #contextInitialized(ServletContextEvent)
		 * @see #contextDestroyed(ServletContextEvent)
		 */
		public ContextLoaderListener() {
		}
		/**
		 * Create a new {@code ContextLoaderListener} with the given application context. This
		 * constructor is useful in Servlet 3.0+ environments where instance-based
		 * registration of listeners is possible through the {@link javax.servlet.ServletContext#addListener}
		 * API.
		 * <p>The context may or may not yet be {@linkplain
		 * org.springframework.context.ConfigurableApplicationContext#refresh() refreshed}. If it
		 * (a) is an implementation of {@link ConfigurableWebApplicationContext} and
		 * (b) has <strong>not</strong> already been refreshed (the recommended approach),
		 * then the following will occur:
		 * <ul>
		 * <li>If the given context has not already been assigned an {@linkplain
		 * org.springframework.context.ConfigurableApplicationContext#setId id}, one will be assigned to it</li>
		 * <li>{@code ServletContext} and {@code ServletConfig} objects will be delegated to
		 * the application context</li>
		 * <li>{@link #customizeContext} will be called</li>
		 * <li>Any {@link org.springframework.context.ApplicationContextInitializer ApplicationContextInitializer org.springframework.context.ApplicationContextInitializer ApplicationContextInitializers}
		 * specified through the "contextInitializerClasses" init-param will be applied.</li>
		 * <li>{@link org.springframework.context.ConfigurableApplicationContext#refresh refresh()} will be called</li>
		 * </ul>
		 * If the context has already been refreshed or does not implement
		 * {@code ConfigurableWebApplicationContext}, none of the above will occur under the
		 * assumption that the user has performed these actions (or not) per his or her
		 * specific needs.
		 * <p>See {@link org.springframework.web.WebApplicationInitializer} for usage examples.
		 * <p>In any case, the given application context will be registered into the
		 * ServletContext under the attribute name {@link
		 * WebApplicationContext#ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE} and the Spring
		 * application context will be closed when the {@link #contextDestroyed} lifecycle
		 * method is invoked on this listener.
		 * @param context the application context to manage
		 */
		public ContextLoaderListener(WebApplicationContext context) {
			super(context);
		}
		/**
		 * Initialize the root web application context.
		 */
		@Override
		public void contextInitialized(ServletContextEvent event) {
			initWebApplicationContext(event.getServletContext());
		}	
		/**
		 * Close the root web application context.
		 */
		@Override
		public void contextDestroyed(ServletContextEvent event) {
			closeWebApplicationContext(event.getServletContext());
			ContextCleanupListener.cleanupAttributes(event.getServletContext());
		}
	}
	```
### 	ContextLoader
* ContextLoader主要负责加载spring主容器，即root ApplicationContext，在设计层面主要定义了contextId，contextConfigLocation，contextClass，contextInitializerClasses。这些参数都可以在配置中指定，如web.xml的context-param标签，或者是基于Java编程方式配置的WebApplicationInitializer中定义，作为分别为：
   1. contextId：当前容器的id，主要给底层所使用的BeanFactory，在进行序列化时使用。
   2. contextConfigLocation：配置文件的位置，默认为WEB-INF/applicationContext.xml，可以通过在web.xml使用context-param标签来指定其他位置，其他名字或者用逗号分隔指定多个。在配置文件中通过beans作为主标签来定义bean。这样底层的BeanFactory会解析beans标签以及里面的bean，从而来创建BeanDefinitions集合，即bean的元数据内存数据库。
   3. contextClass：当前所使用的WebApplicationContext的类型，如果是在WEB-INF/applicationContext.xml中指定beans，则使用XmlWebApplicationContext，如果是通过注解，如@Configuration，@Component等，则是AnnotationConfigWebApplicationContext，通过扫描basePackages指定的包来创建bean。
   4. contextInitializerClasses：ApplicationContextInitializer的实现类，即在调用ApplicationContext的refresh加载beanDefinition和创建bean之前，对WebApplicationContext进行一些初始化。
* initWebApplicationContext方法：创建和初始化spring主容器对应的WebApplicationContext，主要完成两个操作：
   1. 创建WebApplicationContext对象实例并调用refresh方法完成从contextConfigLocation指定的配置中，加载BeanDefinitions和创建bean实例；核心源码实现如下：
		```java
		// Store context in local instance variable, to guarantee that
		// it is available on ServletContext shutdown.
		if (this.context == null) {
		
		   // 创建一个WebApplicationContext
		   // 具体类型如果是指定了contextClass则使用该指定的；
		   // 默认使用XmlWebApplicationContext
		   
			this.context = createWebApplicationContext(servletContext);
		}
		if (this.context instanceof ConfigurableWebApplicationContext) {
			ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) this.context;
			if (!cwac.isActive()) {
			
			    // 设置parent WebApplicationContext，
			    // 对root WebApplicationContext来说，通常为null
			    
				// The context has not yet been refreshed -> provide services such as
				// setting the parent context, setting the application context id, etc
				if (cwac.getParent() == null) {
					// The context instance was injected without an explicit parent ->
					// determine parent for root web application context, if any.
					ApplicationContext parent = loadParentContext(servletContext);
					cwac.setParent(parent);
				}
				
				// 核心方法，完成配置加载，BeanDefinition定义和bean对象创建
				configureAndRefreshWebApplicationContext(cwac, servletContext);
			}
		}
		```
		configureAndRefreshWebApplicationContext方法主要完成对ApplicationContext配置文件地址contextConfigLocation的设值，调用ApplicationContextInitializers，最后调用ApplicationContext的refresh方法完成bean容器的创建。
		
		```java
		protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac, ServletContext sc) {
				
				...
				
				wac.setServletContext(sc);
				String configLocationParam = sc.getInitParameter(CONFIG_LOCATION_PARAM);
				if (configLocationParam != null) {
					wac.setConfigLocation(configLocationParam);
				}
		
				// The wac environment's #initPropertySources will be called in any case when the context
				// is refreshed; do it eagerly here to ensure servlet property sources are in place for
				// use in any post-processing or initialization that occurs below prior to #refresh
				ConfigurableEnvironment env = wac.getEnvironment();
				if (env instanceof ConfigurableWebEnvironment) {
					((ConfigurableWebEnvironment) env).initPropertySources(sc, null);
				}
				// 使用ApplicationContextInitializer对ApplicationContext进行初始化
				customizeContext(sc, wac);
				
				// ApplicationContext的核心方法：在refresh方法中完成ApplicationContext的启动
				// 即spring容器的各个核心组件的创建，如beanDefinitions，enviromnet等
				
				wac.refresh();
			}
		```

   2. 将创建好的WebApplicationContext实例作为将创建好的WebApplicationContext实例，作为一个attribute保存在ServletContext当中，核心源码实现如下：

		```java
		servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.context);
		```
	   其中		WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE为：
	

		```java
			/**
			 * Context attribute to bind root WebApplicationContext to on successful startup.
			 * <p>Note: If the startup of the root context fails, this attribute can contain
			 * an exception or error as value. Use WebApplicationContextUtils for convenient
			 * lookup of the root WebApplicationContext.
			 * @see org.springframework.web.context.support.WebApplicationContextUtils#getWebApplicationContext
			 * @see org.springframework.web.context.support.WebApplicationContextUtils#getRequiredWebApplicationContext
			 */
			String ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE = WebApplicationContext.class.getName() + ".ROOT";
		```

### 总结
* 通过上面的分析可知，ContextLoader完成spring主容器的创建，工作主要是负责从ServletContext中，具体为ServletContext从web.xml文件或者WebApplicationInitializer的实现类中，获取WebApplicationContext的相关配置信息，如使用contextClass指定使用哪种WebApplicationContext实现，contextConfigLocation指定spring容器的配置文件在哪里，以及获取WebApplicationContextInitializers来在进行spring容器创建之前，对WebApplicationContext进行加工处理。
* 而spring容器的创建，则是使用spring-context包的ApplicationContext，spring-beans包的BeanFactory，来完成从配置中获取beans定义并创建BeanDefinition，获取一些资源属性值，以及完成单例bean的创建等。具体在之后关于ApplicationContext和BeanFactory体系结构的文章中分析。
