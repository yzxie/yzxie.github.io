---
layout:     post
title:      "Spring源码分析（三）：SpringMVC的DispatcherServlet的设计与实现"
subtitle:   "SpringMVC DispatcherServlet"
date:       2019-01-06 20:49:00
author:     "XYZ"
header-img: "img/article-bg1.jpg"
tags:
    - Spring源码分析
---

### 概述
* DispatcherServlet是SpringMVC的一个前端控制器，是MVC架构中的C，即controller的实现，用于拦截这个web应用的所有请求，具体为在web.xml中配置这个servlet，对应的url-pattern设置为“/”，或者使用servlet3.0之后的WebApplicationInitializer来配置，在web容器启动这个应用时，会创建和初始化这个DispatcherServlet对象实例。
* DispatcherServlet在接收到请求之后，会根据请求的uri信息，找到对应的某个controller的某个方法来处理这个请求。通常在controller对应的类中使用@Controller和@RequestMapping来唯一确定类的一个方法处理哪些uri请求，具体包括路径，http请求方法等信息。同时还需要处理主题theme，本地化locale，multipart请求，以及响应到View视图的映射。
* 基于以上需求背景，DispatcherServlet需要定义相关的子组件来完成这些功能。由于Spring的ApplicationContext体系结构设计当中是支持层次化的，即整个spring应用包含一个root WebApplicationContext，多个子WebApplicationContext，子WebApplicationContext共享这个root WebApplicationContext。每个DispatcherServlet可以使用一个独立的，只与这个DispatcherServlet实例绑定的WebApplicationContext来创建和管理这些子组件。

### DispatcherServlet的类结构设计
* DispatcherServlet既是servlet规范中的一个普通的servlet，在service方法中定义对请求的处理；又通过spring的WebApplicationContext，即使用spring的beans容器机制，来管理其相关的子组件bean，并在自身中维护对spring容器中的子组件bean的引用，从而方便调用。所以在DispatcherServlet类结构设计当中，就需要考虑这两个方面的设计，如图：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190119171940993.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
* 从上图可以看出，在DispatcherServlet的类的继承体系中，从下到上依次为：DispatcherServlet -> FrameworkServlet -> HttpServletBean。
#### HttpServletBean
* 继承于HttpServlet，实现了EnvironmentAware（注入Environment对象）和EnvironmentCapable（访问Environment对象）接口，其中Environment主要从类路径的属性文件，运行参数，@PropertySource注解等获取应用的相关属性值，以提供给spring容器相关组件访问，或者写入属性到Environment来给其他组件访问。HttpServletBean的主要作用就是将于该servlet相关的init-param，封装成bean属性，然后保存到Environment当中，从而可以在spring容器中被其他bean访问。
#### FrameworkServlet
* 继承于HttpServletBean，因为DispatcherServlet通常包含一个独立的WebApplication，而普通的servlet则只是通过servletContext获取spring容器的root WebApplicationContext，从而从中获取相关bean，所以在DispatcherServlet类层次结构中，增加FrameworkServlet这层设计。作用就是用于获取，创建与管理，DispatcherServlet所绑定的WebApplicationContext对象，即完成WebApplicationContext的创建相关的：从contextConfigLocation获取xml或WebApplicationInitializer配置信息，根据contextClass创建WebApplicationContext，以及获取ApplicationContextInitializer来对WebApplicationContext进行初始化，最后调用refresh完成DispatcherServlet绑定的这个WebApplicationContext的创建。
#### DispatcherServlet
* 从FrameworkServlet中获取WebApplicationContext，然后从WebApplicationContext中获取DispatcherServlet的相关功能子组件bean，然后在自身维护一个引用。实现doService方法并使用这些功能子组件来完成请求的处理和生成响应。
#### 三者的关联
* 在HttpServletBean的init方法中，定义initServletBean模板方法，供子类实现，其中FrameworkServlet的initServletBean方法实现为创建WebApplicationContext，即调用initWebApplicationContext方法来完成WebApplicationContext的创建，并在initWebApplicationContext方法中定义onRefresh模板方法由子类实现，其中DispatcherServlet的onRefresh方法实现为从initWebApplicationContext的WebApplicationContext获取其功能子组件的bean，保存在自身的引用中。

### DispatcherServlet的功能子组件
* 在DispatcherServlet的onRefresh方法中，完成从WebApplicationContext获取对应的bean，赋值到自身的引用当中，源码实现如下：

	```java
	/**
	 * This implementation calls {@link #initStrategies}.
	 */
	@Override
	protected void onRefresh(ApplicationContext context) {
		initStrategies(context);
	}
	
	/**
	 * Initialize the strategy objects that this servlet uses.
	 * <p>May be overridden in subclasses in order to initialize further strategy objects.
	 */
	protected void initStrategies(ApplicationContext context) {
		initMultipartResolver(context);
		initLocaleResolver(context);
		initThemeResolver(context);
		initHandlerMappings(context);
		initHandlerAdapters(context);
		initHandlerExceptionResolvers(context);
		initRequestToViewNameTranslator(context);
		initViewResolvers(context);
		initFlashMapManager(context);
	}
	```
* DispatcherServlet中对应的bean引用如下：即核心功能组件，主要包括multipart请求处理器multipartResolver，本地化处理器localeResolver，主题（JSP的css样式）处理器themeResolver，请求URI和处理方法映射handlerMappings，请求处理器适配器handlerAdapters，请求异常处理器handlerExceptionResolvers，视图名称解析器viewNameTranslator，视图处理器viewResolvers，转发请求数据存储器flashMapManager。
	```java
	/** MultipartResolver used by this servlet. */
	@Nullable
	private MultipartResolver multipartResolver;
	
	/** LocaleResolver used by this servlet. */
	@Nullable
	private LocaleResolver localeResolver;
	
	/** ThemeResolver used by this servlet. */
	@Nullable
	private ThemeResolver themeResolver;
	
	/** List of HandlerMappings used by this servlet. */
	@Nullable
	private List<HandlerMapping> handlerMappings;
	
	/** List of HandlerAdapters used by this servlet. */
	@Nullable
	private List<HandlerAdapter> handlerAdapters;
	
	/** List of HandlerExceptionResolvers used by this servlet. */
	@Nullable
	private List<HandlerExceptionResolver> handlerExceptionResolvers;
	
	/** RequestToViewNameTranslator used by this servlet. */
	@Nullable
	private RequestToViewNameTranslator viewNameTranslator;
	
	/** FlashMapManager used by this servlet. */
	@Nullable
	private FlashMapManager flashMapManager;
	
	/** List of ViewResolvers used by this servlet. */
	@Nullable
	private List<ViewResolver> viewResolvers;
	```
* 重点需要关注initHandlerMappings的实现，主要是从WebApplicationContext获取HandlerMapping接口实现类的对象实例，然后从HandlerMapping中获取请求URI和请求方法的处理器映射，其中一个重要的实现为RequestMappingHandlerMapping（包为：org.springframework.web.servlet.mvc.method.annotation），这个是处理@Controller和@RequestMapping，然后生成HandlerMethod对象实现了映射URI的（URI的信息由RequestMappingInfo维护），具体在后续文章中详细分析。
### DispatcherServlet的请求处理
* DispatcherServlet作为前端控制器，接收所有的请求（即从客户端发送请求到tomcat，tomcat定位到该应用，分配给DispatcherServlet），在doService方法中定义对请求的处理逻辑。doService方法对请求request进行一些加工，即添加一些attribute，包含当前DispatcherServlet的WebApplicationContext，本地化处理器localeResolver，主题处理器themeResolver等功能组件，而选择哪个controler的哪个方法来处理的逻辑，则由子方法doDispatch来实现，并且生成response响应客户端，源码如下：

	```java
	/**
	 * Process the actual dispatching to the handler.
	 * <p>The handler will be obtained by applying the servlet's HandlerMappings in order.
	 * The HandlerAdapter will be obtained by querying the servlet's installed HandlerAdapters
	 * to find the first that supports the handler class.
	 * <p>All HTTP methods are handled by this method. It's up to HandlerAdapters or handlers
	 * themselves to decide which methods are acceptable.
	 * @param request current HTTP request
	 * @param response current HTTP response
	 * @throws Exception in case of any kind of processing failure
	 */
	protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
		HttpServletRequest processedRequest = request;
		HandlerExecutionChain mappedHandler = null;
		boolean multipartRequestParsed = false;
	
		WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
	
		try {
			ModelAndView mv = null;
			Exception dispatchException = null;
	
			try {
			
				// muiltpart请求处理，如果是则进行封装加工
				
				processedRequest = checkMultipart(request);
				multipartRequestParsed = (processedRequest != request);
	
				// 获取处理这个请求的那个controler的那个方法
				
				// Determine handler for the current request.
				mappedHandler = getHandler(processedRequest);
				if (mappedHandler == null) {
					noHandlerFound(processedRequest, response);
					return;
				}
	
				// 使用请求处理器适配器HandlerAdapter来定义
				// 请求的DispatcherServlet的统一处理格式
				// 即定义一种统一的模板来调用不同的Handler实现请求处理
				// 其中Handler由拦截器链和请求处理方法HandlerMethod组成
				
				// Determine handler adapter for the current request.
				HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
	
				// Process last-modified header, if supported by the handler.
				String method = request.getMethod();
				boolean isGet = "GET".equals(method);
				if (isGet || "HEAD".equals(method)) {
					long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
					if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
						return;
					}
				}
				
				// 调用拦截器链各个拦截器的preHandle，进行预处理
				if (!mappedHandler.applyPreHandle(processedRequest, response)) {
					return;
				}
				
				// 请求的实际处理，返回一个ModelAndView
				// 其中Model表示模型，即包含数据在视图渲染中使用
				// View为定义到具体的JSP视图
				
				// Actually invoke the handler.
				mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
	
				if (asyncManager.isConcurrentHandlingStarted()) {
					return;
				}
	
				applyDefaultViewName(processedRequest, mv);
				mappedHandler.applyPostHandle(processedRequest, response, mv);
			}
			catch (Exception ex) {
				dispatchException = ex;
			}
			catch (Throwable err) {
				// As of 4.3, we're processing Errors thrown from handler methods as well,
				// making them available for @ExceptionHandler methods and other scenarios.
				dispatchException = new NestedServletException("Handler dispatch failed", err);
			}
			
			// 对客户端进行响应
			
			processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
		}
		catch (Exception ex) {
			triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
		}
		catch (Throwable err) {
			triggerAfterCompletion(processedRequest, response, mappedHandler,
					new NestedServletException("Handler processing failed", err));
		}
		finally {
			if (asyncManager.isConcurrentHandlingStarted()) {
				// Instead of postHandle and afterCompletion
				if (mappedHandler != null) {
					mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
				}
			}
			else {
				// Clean up any resources used by a multipart request.
				if (multipartRequestParsed) {
					cleanupMultipart(processedRequest);
				}
			}
		}
	}
	```
### 总结
* 通过以上分析可知，DispatcherServlet作为springMVC框架的一个统一前端控制器，需要接收所有发送到这个应用的请求，然后在自身启动时，已经加载好的URI和请求处理器映射，获取对应的请求处理器，由请求处理器进行实际的请求处理。
* 所以基于此，DispatcherServlet需要多种子功能组件来完成请求处理，由于应用也可以使用多个DispatcherServlet来接收请求，为了对众多子功能组件的封装和多个DispatcherServlet的子组件的隔离性，每个DispatcherServlet使用了一个自身独立spring子容器WebApplicationContext来管理自身的子功能组件。然后共享同一个root WebApplicationContext（即WEB-INF/applicationContext.xml）来获取公用组件，如数据库连接池等。
* 在这些子功能组件中，我们需要核心关注HandlerMappings，即请求URI和请求处理器映射这个组件的设计，即DispatcherServlet的WebApplicationContext是如何产生这个映射的，映射的设计是怎么样的，请求处理器具体是什么，如我们应用代码通常是使用@Controller和@RequestMapping来定义的，这些在底层源码是怎么实用的；当一个请求到来时，DispatcherServlet是如何从里面查找的。这些问题其实就需要到spring的IOC实现了，即spring-context和spring-beans包的实现，具体在后续文章分析。