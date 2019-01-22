---
layout:     post
title:      "Spring源码分析（四）：SpringMVC的HandlerMapping和HandlerAdapter的体系结构设计与实现"
subtitle:   "SpringMVC HandlerMapping & HandlerAdapter"
date:       2019-01-07 20:49:00
author:     "XYZ"
header-img: "img/article-bg1.jpg"
tags:
    - Spring源码分析
---

### 概述
* 在我的上一篇文章：[Spring源码分析（三）：DispatcherServlet的设计与实现](https://blog.csdn.net/u010013573/article/details/86550201)中提到，DispatcherServlet在接收到客户端请求时，会遍历DispatcherServlet自身维护的一个HandlerMapping集合，来查找该请求对应的请求处理器，然后由该请求处理器来执行请求处理。
* 在SpringMVC中，DispatcherServlet通过HandlerMapping接口来定义请求和请求处理器之间的映射关系，通过HandlerAdapter接口来提供一个模板实现，即以统一的方式，调用不同handler来执行请求处理。
* HandlerMapping和HandlerAdapter并不是跟名字所描述的这样，即每个HandlerMapping定义一个请求和请求处理器的映射对，这两个接口不是从请求处理方法的角度来定义的，而是从请求处理器的实现类型的角度来定义的。HandlerMapping包含Url和HandlerMethod两种实现，HandlerAdapter包含HttpMethod，Servlet，Controller，HttpRequestHandler四种实现。
### HandlerMapping：请求与请求处理器映射
* DispatcherServlet通过HandlerMapping获取处理某个请求的请求处理器，请求处理器由请求执行器handler和匹配的拦截器链interceptors两部分组成，具体由HandlerExecutionChain定义请求处理器实现。所以在HandlerMapping接口的实现当中，需要维护两个重要的组件，分别是请求和请求处理器handler的映射map，总拦截器链interceptors，拦截器链在HandlerMapping接口的最顶层抽象实现类AbstractHandlerMapping定义。
* 其次在AbstractHandlerMapping中会定义Cors跨域访问的处理，通常也是在拦截器链添加一个跨域拦截器。然后通过请求的header的Origin的值和应用的跨域配置来执行请求拦截。
#### 请求和请求处理器handler的映射map
##### 1. 基于URL
* 主要在抽象类AbstractUrlHandlerMapping中定义，具体实现子类包含BeanNameUrlHandlerMapping和SimpleUrlHandlerMapping。其中BeanNameUrlHandlerMapping的映射map的key为请求URI，value是beanName，即处理匹配这个uri的请求的bean，在spring容器中的beanName，通常是以“/”开头，如beanName为“/foo”的bean处理URI为“/foo”的请求，同时可以指定beanName的别名alias，这样这些别名也作为匹配的请求URI。获取映射关系的源码如下：

	```java
	public class BeanNameUrlHandlerMapping extends AbstractDetectingUrlHandlerMapping {
	
		/**
		 * Checks name and aliases of the given bean for URLs, starting with "/".
		 */
		@Override
		protected String[] determineUrlsForHandler(String beanName) {
			List<String> urls = new ArrayList<>();
			
			// urls为以这个beanName为value的多个key，即map的key和value
			// beanName以“/”开头
			if (beanName.startsWith("/")) {
				urls.add(beanName);
			}
			// 别名也作为value
			String[] aliases = obtainApplicationContext().getAliases(beanName);
			for (String alias : aliases) {
				if (alias.startsWith("/")) {
					urls.add(alias);
				}
			}
			return StringUtils.toStringArray(urls);
		}
	
	}
	```
	SimpleUrlHandlerMapping与BeanNameUrlHandlerMapping类似，不过映射map的value的类型可以是beanName或者是bean实例。而这个value对应的请求执行器，在SimpleUrlHandlerMapping与BeanNameUrlHandlerMapping中，可以是Servlet，Controller（是Controller接口实现类，不是@Controller注解）HttpRequestHandler，这个是与HandlerAdapter的实现类对应的。
#### 2. 基于HandlerMethod
* 我们在应用代码中通常使用@Controller和@RequestMapping来定义请求和请求处理方法直接的映射关系，这种方式的请求和请求处理方法映射map是基于HandlerMethod来实现的，即将每个处理方法在springMVC中都会抽象成一个HandlerMethod对象，而请求的匹配条件配置，则是通过RequestMappingInfo来定义的。而这个对应HandlerMapping体系结构的设计是RequestMappingHandlerMapping。
###### HandlerMethod：基于方法的请求执行器
* HandlerMethod主要用于封装@Controller注解的类的使用@RequestMapping（或@RequestMapping的变体，如@GetMapping）注解的方法信息，类设计如下：
	```java
	/**
	 * Encapsulates information about a handler method consisting of a
	 * {@linkplain #getMethod() method} and a {@linkplain #getBean() bean}.
	 * Provides convenient access to method parameters, the method return value,
	 * method annotations, etc.
	 *
	 * <p>The class may be created with a bean instance or with a bean name
	 * (e.g. lazy-init bean, prototype bean). Use {@link #createWithResolvedBean()}
	 * to obtain a {@code HandlerMethod} instance with a bean instance resolved
	 * through the associated {@link BeanFactory}.
	 * @since 3.1
	 */
	public class HandlerMethod {
		private final Object bean;
		@Nullable
		private final BeanFactory beanFactory;
		private final Class<?> beanType;
		private final Method method;
		private final Method bridgedMethod;
		private final MethodParameter[] parameters;
		@Nullable
		private HttpStatus responseStatus;
		@Nullable
		private String responseStatusReason;
		@Nullable
		private HandlerMethod resolvedFromHandlerMethod;
		@Nullable
		private volatile List<Annotation[][]> interfaceParameterAnnotations;
		
		...
	
	}
	```
	核心属性为bean，即@Controller注解类对象；method请求方法，主要用于反射调用；parameters方法参数，类型为MethodParameter，MethodParameter封装了参数的位置parameterIndex，参数类型parameterType，参数注解parameterAnnotations等信息。

###### RequestMappingInfo：请求的匹配条件
* 主要是对@RequestMapping注解的相关属性进行封装，然后作为请求和请求处理器映射map的key。通常@RequstMapping可以设置value，params，method，consumes，produces等参数来精确匹配请求。
* value为匹配的路径；params是匹配的请求参数值；method为处理的http请求方法，consumers为请求的mediaType，produces为响应的mediaType。value，params，consumes，produces都可以设置多个，如下为一个用例：
	```java
	@RestController  
	@RequestMapping("/home")  
	public class IndexController {  
	    @RequestMapping(value = {
	     "/head",
	     "/index"
	    }, 
	    method = RequestMethod.GET,
	    headers = {  
	        "content-type=text/plain",  
	        "content-type=text/html"  
	    }, consumers = {
	    	"application/JSON",  
        	"application/XML"  
	    }, produces = {
	    	"application/JSON"
	    }) String post() {  
	        return "Mapping applied along with headers";  
	    }  
	}  
	```
* RequestMappingInfo的类设计如下：
	```java
	public final class RequestMappingInfo implements RequestCondition<RequestMappingInfo> {
		@Nullable
		private final String name;
		private final PatternsRequestCondition patternsCondition;
		private final RequestMethodsRequestCondition methodsCondition;
		private final ParamsRequestCondition paramsCondition;
		private final HeadersRequestCondition headersCondition;
		private final ConsumesRequestCondition consumesCondition;
		private final ProducesRequestCondition producesCondition;
		private final RequestConditionHolder customConditionHolder;
		
		...
	
			/**
		 * Checks if all conditions in this request mapping info match the provided request and returns
		 * a potentially new request mapping info with conditions tailored to the current request.
		 * <p>For example the returned instance may contain the subset of URL patterns that match to
		 * the current request, sorted with best matching patterns on top.
		 * @return a new instance in case all conditions match; or {@code null} otherwise
		 */
		@Override
		@Nullable
		public RequestMappingInfo getMatchingCondition(HttpServletRequest request) {
			RequestMethodsRequestCondition methods = this.methodsCondition.getMatchingCondition(request);
			ParamsRequestCondition params = this.paramsCondition.getMatchingCondition(request);
			HeadersRequestCondition headers = this.headersCondition.getMatchingCondition(request);
			ConsumesRequestCondition consumes = this.consumesCondition.getMatchingCondition(request);
			ProducesRequestCondition produces = this.producesCondition.getMatchingCondition(request);
	
			if (methods == null || params == null || headers == null || consumes == null || produces == null) {
				return null;
			}
	
			PatternsRequestCondition patterns = this.patternsCondition.getMatchingCondition(request);
			if (patterns == null) {
				return null;
			}
	
			RequestConditionHolder custom = this.customConditionHolder.getMatchingCondition(request);
			if (custom == null) {
				return null;
			}
	
			return new RequestMappingInfo(this.name, patterns,
					methods, params, headers, consumes, produces, custom.getCondition());
		}
		
		...
	}
	```

###### RequestMappingHandlerMapping：基于HandlerMethod和RequestMappingInfo的HandlerMapping实现
 * RequestMappingHandlerMapping是HandlerMapping的一个实现，其请求和请求处理器的映射map是以RequestMappingInfo为key，HandlerMethod为value的。这样设计可以通过从客户端请求request对象中获取URI，http method，http header等信息，从而找到匹配的RequestMappingInfo。以该RequestMappingInfo作为key，在映射map中找到请求处理器或者说是请求处理方法。
 * RequestMappingHandlerMapping的类结构设计如下：![在这里插入图片描述](https://img-blog.csdnimg.cn/20190120113007811.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
	核心关注：RequestMappingHandlerMapping -> RequestMappingInfoHandlerMapping -> AbstractHandlerMethodMapping -> AbstractHandlerMapping，InitializingBean这个继承体系设计。同时从请求和请求处理器映射map的创建和初始化；给定请求，从请求和请求处理器映射map查找请求处理器两个角度来分析：
	1. map的创建和初始化
	由类的继承体系可知，实现了InitializingBean接口，故spring容器在创建这个bean时，填充好所有属性之后，会调用InitializingBean的afterPropertiesSet方法，以下为从RequestMappingHandlerMapping -> RequestMappingInfoHandlerMapping的afterPropertiesSet实现：
	RequestMappingHandlerMapping的afterPropertiesSet方法定义：
		```java
			@Override
			public void afterPropertiesSet() {
				this.config = new RequestMappingInfo.BuilderConfiguration();
				this.config.setUrlPathHelper(getUrlPathHelper());
				this.config.setPathMatcher(getPathMatcher());
				this.config.setSuffixPatternMatch(this.useSuffixPatternMatch);
				this.config.setTrailingSlashMatch(this.useTrailingSlashMatch);
				this.config.setRegisteredSuffixPatternMatch(this.useRegisteredSuffixPatternMatch);
				this.config.setContentNegotiationManager(getContentNegotiationManager());
				
				// 调用RequestMappingInfoHandlerMapping的afterPropertiesSet方法
				
				super.afterPropertiesSet();
			}
		```
		RequestMappingInfoHandlerMapping的afterPropertiesSet方法：定义initHandlerMethods完成请求和请求处理器映射map的初始化，故map的初始化是定义在RequestMappingInfoHandlerMapping中的，其整体实现源码如下:
		```java
			// Handler method detection
			/**
			 * Detects handler methods at initialization.
			 * @see #initHandlerMethods
			 */
			@Override
			public void afterPropertiesSet() {
				initHandlerMethods();
			}
		
			/**
			 * Scan beans in the ApplicationContext, detect and register handler methods.
			 * @see #getCandidateBeanNames()
			 * @see #processCandidateBean
			 * @see #handlerMethodsInitialized
			 */
			protected void initHandlerMethods() {
				
				// 遍历所有bean
				
				for (String beanName : getCandidateBeanNames()) {
					if (!beanName.startsWith(SCOPED_TARGET_NAME_PREFIX)) {
						
						// 查找是handler的bean
						
						processCandidateBean(beanName);
					}
				}
				handlerMethodsInitialized(getHandlerMethods());
			}
		```
		获取所有Object类型的bean：getCandidateBeanNames
		```java	
			// 获取所有Object类型的bean
			
			/**
			 * Determine the names of candidate beans in the application context.
			 * @since 5.1
			 * @see #setDetectHandlerMethodsInAncestorContexts
			 * @see BeanFactoryUtils#beanNamesForTypeIncludingAncestors
			 */
			protected String[] getCandidateBeanNames() {
				return (this.detectHandlerMethodsInAncestorContexts ?
						BeanFactoryUtils.beanNamesForTypeIncludingAncestors(obtainApplicationContext(), Object.class) :
						obtainApplicationContext().getBeanNamesForType(Object.class));
			}
		```
			
		processCandidateBean：通过isHandler方法，判断某个bean是否是handler，如果是则通过detectHandlerMethods方法，获取该handler的所有方法，然后保存注册到请求和请求处理器映射map。
		
		```java
			/**
			 * Determine the type of the specified candidate bean and call
			 * {@link #detectHandlerMethods} if identified as a handler type.
			 * <p>This implementation avoids bean creation through checking
			 * {@link org.springframework.beans.factory.BeanFactory#getType}
			 * and calling {@link #detectHandlerMethods} with the bean name.
			 * @param beanName the name of the candidate bean
			 * @since 5.1
			 * @see #isHandler
			 * @see #detectHandlerMethods
			 */
			protected void processCandidateBean(String beanName) {
				Class<?> beanType = null;
				try {
					beanType = obtainApplicationContext().getType(beanName);
				}
				catch (Throwable ex) {
					// An unresolvable bean type, probably from a lazy bean - let's ignore it.
					if (logger.isTraceEnabled()) {
						logger.trace("Could not resolve type for bean '" + beanName + "'", ex);
					}
				}
				
				// 使用isHandler方法，判断bean是否是
				// 请求执行器handler
				// 这个是一个重要方法，在RequestMappingInfoHandlerMapping中实现
				
				if (beanType != null && isHandler(beanType)) {
					detectHandlerMethods(beanName);
				}
			}
				
			/**
			 * Look for handler methods in the specified handler bean.
			 * @param handler either a bean name or an actual handler instance
			 * @see #getMappingForMethod
			 */
			protected void detectHandlerMethods(Object handler) {
				Class<?> handlerType = (handler instanceof String ?
						obtainApplicationContext().getType((String) handler) : handler.getClass());
		
				if (handlerType != null) {
					Class<?> userType = ClassUtils.getUserClass(handlerType);
					Map<Method, T> methods = MethodIntrospector.selectMethods(userType,
							(MethodIntrospector.MetadataLookup<T>) method -> {
								try {
									return getMappingForMethod(method, userType);
								}
								catch (Throwable ex) {
									throw new IllegalStateException("Invalid mapping on handler class [" +
											userType.getName() + "]: " + method, ex);
								}
							});
					if (logger.isTraceEnabled()) {
						logger.trace(formatMappings(userType, methods));
					}
					methods.forEach((method, mapping) -> {
						Method invocableMethod = AopUtils.selectInvocableMethod(method, userType);
						registerHandlerMethod(handler, invocableMethod, mapping);
					});
				}
			}	
		```
		isHandler方法：判断bean是否是handler，这个是在主类RequestMappingHandlerMapping中定义的：判断bean是否使用@Controller或者@RequestMapping注解
		```java
			/**
			 * {@inheritDoc}
			 * <p>Expects a handler to have either a type-level @{@link Controller}
			 * annotation or a type-level @{@link RequestMapping} annotation.
			 */
			@Override
			protected boolean isHandler(Class<?> beanType) {
				return (AnnotatedElementUtils.hasAnnotation(beanType, Controller.class) ||
						AnnotatedElementUtils.hasAnnotation(beanType, RequestMapping.class));
			}
		```
		
	2. 从请求和请求处理器映射map查找请求处理器：主要是在DispatcherServlet中调用，对应RequestMappingHandlerMapping的底层实现如下：主要是在AbstractHandlerMethodMapping的getHandlerInternal方法定义，AbstractHandlerMethodMapping直接继承于AbstractHandlerMapping。getHandlerInternal的源码实现逻辑如下：
		```java
		public abstract class AbstractHandlerMethodMapping<T> extends AbstractHandlerMapping implements InitializingBean {
			/**
			 * Look up a handler method for the given request.
			 */
			@Override
			protected HandlerMethod getHandlerInternal(HttpServletRequest request) throws Exception {
				String lookupPath = getUrlPathHelper().getLookupPathForRequest(request);
				this.mappingRegistry.acquireReadLock();
				try {
					HandlerMethod handlerMethod = lookupHandlerMethod(lookupPath, request);
					return (handlerMethod != null ? handlerMethod.createWithResolvedBean() : null);
				}
				finally {
					this.mappingRegistry.releaseReadLock();
				}
			}
		
			/**
			 * Look up the best-matching handler method for the current request.
			 * If multiple matches are found, the best match is selected.
			 * @param lookupPath mapping lookup path within the current servlet mapping
			 * @param request the current request
			 * @return the best-matching handler method, or {@code null} if no match
			 * @see #handleMatch(Object, String, HttpServletRequest)
			 * @see #handleNoMatch(Set, String, HttpServletRequest)
			 */
			@Nullable
			protected HandlerMethod lookupHandlerMethod(String lookupPath, HttpServletRequest request) throws Exception {
				List<Match> matches = new ArrayList<>();
				List<T> directPathMatches = this.mappingRegistry.getMappingsByUrl(lookupPath);
				if (directPathMatches != null) {
					addMatchingMappings(directPathMatches, matches, request);
				}
				if (matches.isEmpty()) {
					// No choice but to go through all mappings...
					addMatchingMappings(this.mappingRegistry.getMappings().keySet(), matches, request);
				}
		
				if (!matches.isEmpty()) {
					Comparator<Match> comparator = new MatchComparator(getMappingComparator(request));
					matches.sort(comparator);
					Match bestMatch = matches.get(0);
					if (matches.size() > 1) {
						if (logger.isTraceEnabled()) {
							logger.trace(matches.size() + " matching mappings: " + matches);
						}
						if (CorsUtils.isPreFlightRequest(request)) {
							return PREFLIGHT_AMBIGUOUS_MATCH;
						}
						Match secondBestMatch = matches.get(1);
						if (comparator.compare(bestMatch, secondBestMatch) == 0) {
							Method m1 = bestMatch.handlerMethod.getMethod();
							Method m2 = secondBestMatch.handlerMethod.getMethod();
							String uri = request.getRequestURI();
							throw new IllegalStateException(
									"Ambiguous handler methods mapped for '" + uri + "': {" + m1 + ", " + m2 + "}");
						}
					}
					request.setAttribute(BEST_MATCHING_HANDLER_ATTRIBUTE, bestMatch.handlerMethod);
					handleMatch(bestMatch.mapping, lookupPath, request);
					return bestMatch.handlerMethod;
				}
				else {
					return handleNoMatch(this.mappingRegistry.getMappings().keySet(), lookupPath, request);
				}
			}
		```
### DispatcherServlet的handlerMappings集合
#### 1. 初始化
* DispatcherServlet可以包含多个HandlerMapping接口的实现类对象，其中RequestMappingHandlerMapping就是其中一个，用于获取使用@Controller和@RequestMapping注解的类和方法，RequestMappingHandlerMapping获取这些Controller类和其方法，从而生成请求和请求处理器映射map，供DispatcherServlet在接收到请求时查找并调用。
* handlerMappings集合中的HandlerMapping对象的创建和初始化时机是在创建DispatcherServlet自身的bean容器WebApplicationContext的时候，具体为在解析WebApplicationContext的配置xml文件，如WEB-INF/myDispatcherServlet-servlet.xml时，通过spring-webmvc子项目的META-INF/spring.handlers获取命名空间处理器MvcNamespaceHandler，MvcNamespaceHandler注册并使用AnnotationDrivenBeanDefinitionParser来处理xml文件的annotation-driven标签：

	```java
	
	/**
	 * {@link NamespaceHandler} for Spring MVC configuration namespace.
	 *
	 * @author Keith Donald
	 * @author Jeremy Grelle
	 * @author Sebastien Deleuze
	 * @since 3.0
	 */
	public class MvcNamespaceHandler extends NamespaceHandlerSupport {
	
		@Override
		public void init() {
			registerBeanDefinitionParser("annotation-driven", new AnnotationDrivenBeanDefinitionParser());
			
			...
			
		}
	
	}
	```
* RequestMappingHandlerMapping类型的HandlerMapping的创建：AnnotationDrivenBeanDefinitionParser在解析annotation-driven标签时，调用parse方法：此时创建RequestMappingHandlerMapping类型的bean：

	```java
		@Override
		@Nullable
		public BeanDefinition parse(Element element, ParserContext context) {
			
			...
		
			// 创建RequestMappingHandlerMapping的BeanDefinition，
			// 之后创建bean实例
		
			RootBeanDefinition handlerMappingDef = new RootBeanDefinition(RequestMappingHandlerMapping.class);
			handlerMappingDef.setSource(source);
			handlerMappingDef.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
			handlerMappingDef.getPropertyValues().add("order", 0);
			handlerMappingDef.getPropertyValues().add("contentNegotiationManager", contentNegotiationManager);
	
			if (element.hasAttribute("enable-matrix-variables")) {
				Boolean enableMatrixVariables = Boolean.valueOf(element.getAttribute("enable-matrix-variables"));
				handlerMappingDef.getPropertyValues().add("removeSemicolonContent", !enableMatrixVariables);
			}
	
			configurePathMatchingProperties(handlerMappingDef, element, context);
			readerContext.getRegistry().registerBeanDefinition(HANDLER_MAPPING_BEAN_NAME , handlerMappingDef);
	
			RuntimeBeanReference corsRef = MvcNamespaceUtils.registerCorsConfigurations(null, context, source);
			handlerMappingDef.getPropertyValues().add("corsConfigurations", corsRef);
	
			...
			
	}
	```
#### 2. 请求处理
* DispatcherServlet遍历自身的handlerMappings集合，从每个handlerMapping的map中找到处理这个请求的请求处理器：找到一个则返回，其中handlerMappings集合的实例类型，可以是AbstractUrlHandlerMapping的实现，如BeanNameUrlHandlerMapping和SimpleUrlHandlerMapping；或者是AbstractHandlerMethodMapping的实现，如RequestMappingHandlerMapping。
	```java
	/**
	 * Return the HandlerExecutionChain for this request.
	 * <p>Tries all handler mappings in order.
	 * @param request current HTTP request
	 * @return the HandlerExecutionChain, or {@code null} if no handler could be found
	 */
	@Nullable
	protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
		if (this.handlerMappings != null) {
			for (HandlerMapping mapping : this.handlerMappings) {
			
				// 找到一个则返回
				
				HandlerExecutionChain handler = mapping.getHandler(request);
				if (handler != null) {
					return handler;
				}
			}
		}
		return null;
	}
	```
* 调用HandlerMapping的getHandler方法获取一个请求处理器HandlerExecutionChain。
#### 3. 请求执行
* HandlerExecutionChain：由一个请求执行器handler和多个请求拦截器interceptors组成，handler在处理请求之前，需要先调用interceptors的preHandler方法来对请求进行拦截处理，只有通过了全部的拦截器，才能交给请求执行器handler完成真正的请求处理。源码设计如下：
	```java
	/**
	 * Handler execution chain, consisting of handler object and any handler interceptors.
	 * Returned by HandlerMapping's {@link HandlerMapping#getHandler} method.
	 */
	public class HandlerExecutionChain {
		private final Object handler;
	
		@Nullable
		private HandlerInterceptor[] interceptors;
	
		@Nullable
		private List<HandlerInterceptor> interceptorList;
	
		private int interceptorIndex = -1;
	
		...
		
	}
	```
#### 完整请求处理过程	
* DispatcherServlet中使用doDispatch方法完成整个请求处理，核心处理逻辑如下
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

		...
			// multipart请求加工
			processedRequest = checkMultipart(request);
			multipartRequestParsed = (processedRequest != request);

			// 查找请求处理器handler
			
			// Determine handler for the current request.
			mappedHandler = getHandler(processedRequest);
			if (mappedHandler == null) {
				noHandlerFound(processedRequest, response);
				return;
			}

			// 获取请求执行器的执行适配器
			
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

			// 拦截器链前置处理applyPreHandle
			
			if (!mappedHandler.applyPreHandle(processedRequest, response)) {
				return;
			}
			
			// 通过请求执行器适配器来调用请求执行器（如handlerMethod）处理请求
			
			// Actually invoke the handler.
			mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

			if (asyncManager.isConcurrentHandlingStarted()) {
				return;
			}
			
			// 拦截器链后置处理applyPostHandle
			
			applyDefaultViewName(processedRequest, mv);
			mappedHandler.applyPostHandle(processedRequest, response, mv);
		
		...
		
		// 响应客户端
		processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
}
```

### HandlerAdapter：DispatcherServlet的请求执行适配器
* DispatcherServlet通过HandlerAdapter来间接调用实际的请求执行器handler。由上面分析可知请求执行器包含：Servlet，Controller，HttpRequestHandler，HandlerMethod。
* 所以与上面这些请求执行器配套的HandlerAdapter分别为：SimpleServletHandlerAdapter，SimpleControllerHandlerAdapter，HttpRequestHandlerAdapter和RequestMappingHandlerAdapter。
* 如下为SimpleControllerHandlerAdapter的源码：其他类似
	```java
	package org.springframework.web.servlet.mvc;
	/**
	 * Adapter to use the plain {@link Controller} workflow interface with
	 * the generic {@link org.springframework.web.servlet.DispatcherServlet}.
	 * Supports handlers that implement the {@link LastModified} interface.
	 *
	 * <p>This is an SPI class, not used directly by application code.
	 *
	 * @author Rod Johnson
	 * @author Juergen Hoeller
	 * @see org.springframework.web.servlet.DispatcherServlet
	 * @see Controller
	 * @see LastModified
	 * @see HttpRequestHandlerAdapter
	 */
	public class SimpleControllerHandlerAdapter implements HandlerAdapter {
		@Override
		public boolean supports(Object handler) {
			return (handler instanceof Controller);
		}
		@Override
		@Nullable
		public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
				throws Exception {
	
			return ((Controller) handler).handleRequest(request, response);
		}
		@Override
		public long getLastModified(HttpServletRequest request, Object handler) {
			if (handler instanceof LastModified) {
				return ((LastModified) handler).getLastModified(request);
			}
			return -1L;
		}
	}
	```
