---
layout:     post
title:      "SpringBoot学习（三）：Filter过滤器等的配置方法和SpringBoot源码实现原理"
subtitle:   "SpringBoot Filter/Servlet/Listener Configuration"
date:       2019-01-15 20:49:00
author:     "XYZ"
header-img: "img/article-bg1.jpg"
tags:
    - SpringBoot
---
### Servlet，Filter，Listener的注册
* 在SpringBoot应用来说，由于是自身启动了一个Servlet引擎，并且需要创建一个与应用关联ServletContext对象绑定到Servlet引擎，从而使得Servlet引擎接收到请求可以分发到该应用来处理。
* ServletContext内部通常会包含Servlet规范中的Servlet，Filter，Listener等组件，而将这些组件注册到ServletContext，在SpringBoot中主要通过三步来完成，分别是：
   1. 在应用代码定义和配置组件；
   2. 应用启动，获取这些组件，并生成对应的BeanDefinition注册到Spring容器；
   3. 从Spring容器取出这些组件Bean（取出过程中完成有BeanFactory调用getBean方法，基于BeanDefinition完成Bean对象的创建）并绑定到该ServletContext中。

#### 应用代码配置方法

##### 1. RegistrationBean实现类结合@Configuration注解的类的@Bean注解方法
* RegistrationBean是SpringBoot提供的一个抽象类，是ServletContextInitializer接口的实现类，故在应用启动创建应用对应的内嵌的ServletContext时，会从Spring容器获取已经加载好的ServletContextInitializer接口实现类对象，然后对ServletContext进行初始化。
* ServletRegistrationBean：相当于Servlet 3.0+的ServletContext#addServlet(String, Servlet)方法，主要用于对dispatcherServlet进行自定义，如对处理的url进行修改，默认为所有包含“/”的请求都由dispatcherServlet处理，可以通过setUrlMappings方法来修改，类定义如下：

    ```java
    /**
     * A {@link ServletContextInitializer} to register {@link Servlet}s in a Servlet 3.0+
     * container. Similar to the {@link ServletContext#addServlet(String, Servlet)
     * registration} features provided by {@link ServletContext} but with a Spring Bean
     * friendly design.
     * <p>
     * The {@link #setServlet(Servlet) servlet} must be specified before calling
     * {@link #onStartup}. URL mapping can be configured used {@link #setUrlMappings} or
     * omitted when mapping to '/*' (unless
     * {@link #ServletRegistrationBean(Servlet, boolean, String...) alwaysMapUrl} is set to
     * {@code false}). The servlet name will be deduced if not specified.
     *
     * @param <T> the type of the {@link Servlet} to register
     * @author Phillip Webb
     * @since 1.4.0
     * @see ServletContextInitializer
     * @see ServletContext#addServlet(String, Servlet)
     */
    public class ServletRegistrationBean<T extends Servlet>
    		extends DynamicRegistrationBean<ServletRegistration.Dynamic> {
    
    	private static final String[] DEFAULT_MAPPINGS = { "/*" };
    
    	private T servlet;
    
    	private Set<String> urlMappings = new LinkedHashSet<>();
    
    	private boolean alwaysMapUrl = true;
    
    	private int loadOnStartup = -1;
    
    	private MultipartConfigElement multipartConfig;
    	
    	...
    	
    }
    ```

* FilterRegistrationBean：相当于Servlet 3.0+的ServletContext#addFilter(String, Filter)方法，主要用于自定义Filter过滤器来添加到当前的ServletContext，对请求进行过滤，类定义如下：

    ```java
    /**
     * A {@link ServletContextInitializer} to register {@link Filter}s in a Servlet 3.0+
     * container. Similar to the {@link ServletContext#addFilter(String, Filter) registration}
     * features provided by {@link ServletContext} but with a Spring Bean friendly design.
     * <p>
     * The {@link #setFilter(Filter) Filter} must be specified before calling
     * {@link #onStartup(ServletContext)}. Registrations can be associated with
     * {@link #setUrlPatterns URL patterns} and/or servlets (either by {@link #setServletNames
     * name} or via a {@link #setServletRegistrationBeans ServletRegistrationBean}s. When no
     * URL pattern or servlets are specified the filter will be associated to '/*'. The filter
     * name will be deduced if not specified.
     *
     * @param <T> the type of {@link Filter} to register
     * @author Phillip Webb
     * @since 1.4.0
     * @see ServletContextInitializer
     * @see ServletContext#addFilter(String, Filter)
     * @see DelegatingFilterProxyRegistrationBean
     */
    public class FilterRegistrationBean<T extends Filter>
    		extends AbstractFilterRegistrationBean<T> {
    
    	/**
    	 * Filters that wrap the servlet request should be ordered less than or equal to this.
    	 * @deprecated since 2.1.0 in favor of
    	 * {@code OrderedFilter.REQUEST_WRAPPER_FILTER_MAX_ORDER}
    	 */
    	@Deprecated
    	public static final int REQUEST_WRAPPER_FILTER_MAX_ORDER = AbstractFilterRegistrationBean.REQUEST_WRAPPER_FILTER_MAX_ORDER;
    
    	private T filter;
    	
    	...
    	
    }
    ```
* ServletListenerRegistrationBean：相当于Servlet 3.0+的ServletContext#addListener(EventListener)方法，用于注册Servlet规范相关的Listener，接口定义如下：

    ```java
    /**
     * A {@link ServletContextInitializer} to register {@link EventListener}s in a Servlet
     * 3.0+ container. Similar to the {@link ServletContext#addListener(EventListener)
     * registration} features provided by {@link ServletContext} but with a Spring Bean
     * friendly design.
     *
     * This bean can be used to register the following types of listener:
     * <ul>
     * <li>{@link ServletContextAttributeListener}</li>
     * <li>{@link ServletRequestListener}</li>
     * <li>{@link ServletRequestAttributeListener}</li>
     * <li>{@link HttpSessionAttributeListener}</li>
     * <li>{@link HttpSessionListener}</li>
     * <li>{@link ServletContextListener}</li>
     * </ul>
     *
     * @param <T> the type of listener
     * @author Dave Syer
     * @author Phillip Webb
     * @since 1.4.0
     */
    public class ServletListenerRegistrationBean<T extends EventListener>
    		extends RegistrationBean {
    
    	private static final Set<Class<?>> SUPPORTED_TYPES;
    
    	static {
    		Set<Class<?>> types = new HashSet<>();
    		types.add(ServletContextAttributeListener.class);
    		types.add(ServletRequestListener.class);
    		types.add(ServletRequestAttributeListener.class);
    		types.add(HttpSessionAttributeListener.class);
    		types.add(HttpSessionListener.class);
    		types.add(ServletContextListener.class);
    		SUPPORTED_TYPES = Collections.unmodifiableSet(types);
    	}
    
    	private T listener;
    	
    	...
    	
    }
    ```

* 使用示例：IP白名单过滤器

    ```java
    @SpringBootApplication
    public class LogWebApplication {
    
        /**
         * IP白名单过来
         * @param redisTemplate
         * @return
         */
        @Bean
        public FilterRegistrationBean ipFilter(RedisTemplate redisTemplate) {
            FilterRegistrationBean filterRegistrationBean = new FilterRegistrationBean();
            Filter filter = new IpWhiteListFilter(redisTemplate);
            filterRegistrationBean.setFilter(filter);
            filterRegistrationBean.addUrlPatterns(ipWhiteList);
            return filterRegistrationBean;
        }
    
        public static void main(String[] args) {
            SpringApplication.run(LogWebApplication.class, args);
        }
    }
    
    public class IpWhiteListFilter extends GenericFilterBean {
    
        private RedisTemplate<String, Object> redisTemplate;
    
        @Autowired
        public IpWhiteListFilter(RedisTemplate<String, Object> redisTemplate) {
            this.redisTemplate = redisTemplate;
        }
    
        @Override
        public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain)
                throws IOException, ServletException {
    
            HttpServletResponse response = (HttpServletResponse)servletResponse;
            HttpServletRequest request = (HttpServletRequest)servletRequest;
            
            String ip = HttpUtils.getIpAddr(request);
            if (!validIp(ip)) {
                response.setStatus(FORBIDDEN);
                LOG.warn("forbidden ip {} for {}", ip, uri);
                return;
            }
            
            filterChain.doFilter(request, servletResponse);
            } catch (Exception e) {
                LOG.error("doFilter", e);
                response.setStatus(BAD_REQUEST);
                return;
            }
        }
        
        private boolean validIp(String ip) {
            ...
        }
    }
    ```

##### 2. JSR-330的注解@WebServlet，@WebFilter，@WebListener与注解扫描@ServletComponentScan
* 可以使用@WebServlet，@WebFilter，@WebListener注解运用在对应的组件类上面，注意组件类自身实现基于Servlet规范，如实现Filter接口，ServletListener接口等。然后需要在@Configuration注解的配置类中，加上@ServletComponentScan注解，用于扫描@WebServlet，@WebFilter，@WebListener这些注解的类来注册到Spring容器中。
* 其中创建BeanDefinition注册到Spring容器时，也是使用RegistrationBean的实现类，即ServletRegistrationBean，FilterRegistrationBean，ServletListenerRegistrationBean，对组件进行封装的，故也是基于RegistrationBean实现的，只是SpringBoot在内部完成封装，而不需要像上面一种方式一样在应用代码显示使用以上三个RegistrationBean的实现类进行操作，使用@WebServlet，@WebFilter，@WebListener注解定义，使用@ServletComponentScan来扫描即可。

* 使用示例

    ```java
    @SpringBootApplication
    @ServletComponentScan // 使@WebListener生效
    public class LogWebApplication {
    
        /**
         * IP白名单过来
         * @param redisTemplate
         * @return
         */
        @Bean
        public FilterRegistrationBean ipFilter(RedisTemplate redisTemplate) {
            FilterRegistrationBean filterRegistrationBean = new FilterRegistrationBean();
            Filter filter = new IpWhiteListFilter(redisTemplate);
            filterRegistrationBean.setFilter(filter);
            filterRegistrationBean.addUrlPatterns(ipWhiteList);
            return filterRegistrationBean;
        }
    
        public static void main(String[] args) {
            SpringApplication.run(LogWebApplication.class, args);
        }
    }
    
    /**
     * @author xyz
     * @date 11/11/2018 22:03
     * @description: netty服务启动监听器
     */
    @WebListener
    public class NettyServerListener implements ServletContextListener {
        private static final Logger LOG = LoggerFactory.getLogger(NettyServerListener.class);
    
        @Autowired
        private NettyServer nettyServer;
    
        @Override
        public void contextInitialized(ServletContextEvent servletContextEvent) {
            LOG.info("NettyServerListener: spring context inited.");
            Thread nettyServerThread = new Thread(new NettyServerThread());
            nettyServerThread.start();
        }
    
        @Override
        public void contextDestroyed(ServletContextEvent servletContextEvent) {
            LOG.info("NettyServerListener: spring context closed.");
        }
    
        /**
         * netty服务启动线程
         */
        private class NettyServerThread implements Runnable {
    
            @Override
            public void run() {
                nettyServer.start();
            }
        }
    }
    ```

#### SpringBoot内部源码实现
##### ApplicationContext的启动方法refresh
* Spring容器是通过ApplicationContext的refresh方法来定义启动步骤的。
    ```java
    @Override
    public void refresh() throws BeansException, IllegalStateException {
    	synchronized (this.startupShutdownMonitor) {
    		// Prepare this context for refreshing.
    		prepareRefresh();
    
    		// Tell the subclass to refresh the internal bean factory.
    		ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
    
    		// Prepare the bean factory for use in this context.
    		prepareBeanFactory(beanFactory);
    
    		try {
    			// Allows post-processing of the bean factory in context subclasses.
    			postProcessBeanFactory(beanFactory);
    
    			// Invoke factory processors registered as beans in the context.
    			invokeBeanFactoryPostProcessors(beanFactory);
    
    			// Register bean processors that intercept bean creation.
    			registerBeanPostProcessors(beanFactory);
    
    			// Initialize message source for this context.
    			initMessageSource();
    
    			// Initialize event multicaster for this context.
    			initApplicationEventMulticaster();
    
    			// Initialize other special beans in specific context subclasses.
    			onRefresh();
    
    			// Check for listener beans and register them.
    			registerListeners();
    
    			// Instantiate all remaining (non-lazy-init) singletons.
    			finishBeanFactoryInitialization(beanFactory);
    
    			// Last step: publish corresponding event.
    			finishRefresh();
    		}
    
    		...
    		
    	}
    }
    ```
    依次按顺序执行：
   1. obtainFreshBeanFactory：创建BeanFactory和加载BeanDefintion；
   2. invokeBeanFactoryPostProcessors：调用BeanFactoryPostProcessor，即BeanFactory后置处理器。ComponentScan进行相关类扫描是在这里完成的。以上两种方法创建Servlet，Filter和Listener，对应的BeanDefinition的创建并注册到BeanFactory是在这步完成的；
   3. onRefresh：完成有特殊功能的bean实例的创建。从BeanFactory获取Servlet，Filter和Listener对应的BeanDefinition并创建Bean对象实例，然后绑定到ServletContext是在这步完成的。

##### Servlet，Filter和Listener，对应的BeanDefinition的创建并注册到BeanFactory
* RegistrationBean实现类结合@Configuration注解的类的@Bean方法实现：
   * 这种方式跟普通的，通过@Configuration注解配置类和@Bean注解方法来创建BeanDefintioin，然后注册到BeanFactory的实现方式一样，都是通过ConfigurationClassPostProcessor这个BeanFactoryPostProcessor来处理，具体可看我的另外一篇文章分析[：Spring基于@Configuration的类配置的内部源码实现](https://blog.csdn.net/u010013573/article/details/86663467)
   * 在这里的特殊之处为注册的Bean类型为：ServletRegistrationBean，FilterRegistrationBean，ServletListenerRegistrationBean，即@Bean注解的方法返回值类型为以上四种类型之一。

* JSR-330的注解@WebServlet，@WebFilter，@WebListener与注解扫描@ServletComponentScan：
   * 这种方式主要是通过SpringBoot提供的一个BeanFactoryPostProcessor实现：ServletComponentRegisteringPostProcessor。
   * ServletComponentRegisteringPostProcessor：扫描@ServletComponentScan指定的包或者没指定则扫描包含这个注解的类所在的包及其子包，获取使用了@WebServlet，@WebFilter或@WebListener注解的类，并创建BeanDefintion，并由注解的handler加工BeanDefintion后，再注册到BeanFactory；核心方法为scanPackage：扫描指定的包路径，
   
        ```java
        private void scanPackage(
        		ClassPathScanningCandidateComponentProvider componentProvider,
        		String packageToScan) {
        		
            // componentProvider.findCandidateComponents(packageToScan)方法：
            // 查找并创建BeanDefintions集合，但是还没注册到BeanFactory中
        	for (BeanDefinition candidate : componentProvider
        			.findCandidateComponents(packageToScan)) {
        			
        		if (candidate instanceof ScannedGenericBeanDefinition) {
        			for (ServletComponentHandler handler : HANDLERS) {
        			
        			    // 由注解的handler完成加工再注册到BeanFactory
        				handler.handle(((ScannedGenericBeanDefinition) candidate),
        						(BeanDefinitionRegistry) this.applicationContext);
        			}
        		}
        	}
        }
        ```
        HANDLERS的定义如下：
        
        ```java
        static {
        	List<ServletComponentHandler> servletComponentHandlers = new ArrayList<>();
        	servletComponentHandlers.add(new WebServletHandler());
        	servletComponentHandlers.add(new WebFilterHandler());
        	servletComponentHandlers.add(new WebListenerHandler());
        	HANDLERS = Collections.unmodifiableList(servletComponentHandlers);
        }
        ```
   * 对于每个注解，都有相应的handler来注册到BeanFactory，具体为：WebServletHandler，WebFilterHandler，WebListenerHandler。
   * 以下以WebFilterHandler为例：可见在内部使用FilterRegistrationBean来保证Filter接口的实现类：
   
        ```java
        /**
         * Handler for {@link WebFilter}-annotated classes.
         *
         * @author Andy Wilkinson
         */
        class WebFilterHandler extends ServletComponentHandler {
        
        	WebFilterHandler() {
        		super(WebFilter.class);
        	}
        
        	@Override
        	public void doHandle(Map<String, Object> attributes,
        			ScannedGenericBeanDefinition beanDefinition,
        			BeanDefinitionRegistry registry) {
        		
        		// 指定FilterRegistrationBean
        		
        		BeanDefinitionBuilder builder = BeanDefinitionBuilder
        				.rootBeanDefinition(FilterRegistrationBean.class);
        		builder.addPropertyValue("asyncSupported", attributes.get("asyncSupported"));
        		builder.addPropertyValue("dispatcherTypes", extractDispatcherTypes(attributes));
        		builder.addPropertyValue("filter", beanDefinition);
        		builder.addPropertyValue("initParameters", extractInitParameters(attributes));
        		String name = determineName(attributes, beanDefinition);
        		builder.addPropertyValue("name", name);
        		builder.addPropertyValue("servletNames", attributes.get("servletNames"));
        		builder.addPropertyValue("urlPatterns", extractUrlPatterns(attributes));
        		
        		// 创建BeanDefinition并注册到BeanFactory
        		
        		registry.registerBeanDefinition(name, builder.getBeanDefinition());
        	}
        	
        	...
        	
        }
        ```


##### 创建Servlet，Filter和Listener的bean对象实例并绑定到ServletContext
* 由以上分析可知，ApplicationContext的refresh方法在调用完所有BeanFactoryPostProcessor之后，接下来会调用onRefresh方法，在这个方法中注册具有特殊含义的bean对象。
* 创建和启动应用内嵌的Servlet引擎WebServer，创建内嵌的ServletContext对象绑定到WebServer，创建Servlet，Filter和Listener对应的bean对象绑定到ServletContext就是在**ServletWebServerApplicationContext**类的**onRefresh**方法实现的。onRefresh方法的实现如下：

    ```java
    @Override
    protected void onRefresh() {
    	super.onRefresh();
    	try {
    		createWebServer();
    	}
    	catch (Throwable ex) {
    		throw new ApplicationContextException("Unable to start web server", ex);
    	}
    }
    
    @Override
    protected void finishRefresh() {
    	super.finishRefresh();
    	WebServer webServer = startWebServer();
    	if (webServer != null) {
    		publishEvent(new ServletWebServerInitializedEvent(webServer, this));
    	}
    }
    
    private void createWebServer() {
		WebServer webServer = this.webServer;
		ServletContext servletContext = getServletContext();
		if (webServer == null && servletContext == null) {
			ServletWebServerFactory factory = getWebServerFactory();
			
			// 创建webServer，ServletContext，
			// 以及获取ServletContext的ServletContextInitializer
			// 并执行其onStartup方法
			this.webServer = factory.getWebServer(getSelfInitializer());
		}
		else if (servletContext != null) {
			try {
				getSelfInitializer().onStartup(servletContext);
			}
			catch (ServletException ex) {
				throw new ApplicationContextException("Cannot initialize servlet context",
						ex);
			}
		}
		initPropertySources();
	}
	
	// getSelfInitializer调用该方法
	// 从BeanFactory获取ServletContextInitializer接口的实现类
	private void selfInitialize(ServletContext servletContext) throws ServletException {	
		prepareWebApplicationContext(servletContext);
		registerApplicationScope(servletContext);
		WebApplicationContextUtils.registerEnvironmentBeans(getBeanFactory(),
				servletContext);
		// 在getServletContextInitializerBeans方法内部实现：
	    // 从BeanDefiniton创建bean对象实例，具体为调用了BeanFactory的getBean
		for (ServletContextInitializer beans : getServletContextInitializerBeans()) {
			beans.onStartup(servletContext);
		}
	}
    ```
* 从以上源码可知，在selfInitialize方法中，调用getServletContextInitializerBeans方法来从BeanFactory获取ServletContextInitializer接口的实现类，创建bean对象实例并执行onStartup方法。
* 其中RegistrationBean就实现了 ServletContextInitializer接口。RegistrationBean的onStartup方法实现如下：具体由子类实现register方法完成业务逻辑。对Servlet规范相关的Servlet，Filter，Listener，则是绑定到ServletContext。

    ```java
    public abstract class RegistrationBean implements ServletContextInitializer, Ordered {
    
    	private boolean enabled = true;
    
    	@Override
    	public final void onStartup(ServletContext servletContext) throws ServletException {
    		String description = getDescription();
    		if (!isEnabled()) {
    			logger.info(StringUtils.capitalize(description)
    					+ " was not registered (disabled)");
    			return;
    		}
    		// 将当前bean对象注册到servletContext
    		register(description, servletContext);
    	}
    
        // 抽象方法，由子类实现
        
    	/**
    	 * Register this bean with the servlet context.
    	 * @param description a description of the item being registered
    	 * @param servletContext the servlet context
    	 */
    	protected abstract void register(String description, ServletContext servletContext);
    
        ...
        
    }
    ```
    以下以ServletListenerRegistrationBean的register方法实现为例：调用servletContext.addListener完成绑定：
    
    ```java
    @Override
    protected void register(String description, ServletContext servletContext) {
    	try {
    		servletContext.addListener(this.listener);
    	}
    	catch (RuntimeException ex) {
    		throw new IllegalStateException(
    				"Failed to add listener '" + this.listener + "' to servlet context",
    				ex);
    	}
    }
    ```