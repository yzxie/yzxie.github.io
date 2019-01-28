---
layout:     post
title:      "SpringBoot学习（一）：SpringApplication的用法与内部源码实现原理"
subtitle:   "SpringBoot SpringApplication"
date:       2019-01-13 20:49:00
author:     "XYZ"
header-img: "img/article-bg1.jpg"
tags:
    - SpringBoot
---
### 概述
* 在基于SpringBoot的web应用中，通常使用一个带有main方法的类，通过命令行执行main方法来启动整个应用。而在main方法中是使用SpringApplication.run这个静态方法或者创建SpringApplication对象，执行成员方法run，以该main方法所在的类作为参数的方式启动的。
* main方法所在的类是一个基于Spring的注解，如@Configuration，@ComponentScan等，的配置类。

### 使用方法示例
* 典型用法1：使用@SpringBootApplication等

    ```java
    @SpringBootApplication
    @ImportResource(locations= {"classpath:dubbo_consumer.xml"})
    @MapperScan("com.yzxie.easy.log.web.dao")
    @ServletComponentScan // 使@WebListener生效
    public class LogWebApplication {
        public static void main(String[] args) {
            SpringApplication.run(LogWebApplication.class, args);
        }
    }
    ```
* 典型用法2：使用@Configuration等

    ```java
    @Configuration
    @EnableAutoConfiguration
    public class MyApplication  {
 
       // ... Bean definitions
     
        public static void main(String[] args) {
         SpringApplication.run(MyApplication.class, args);
       }
    }
    ```
    
* 典型用法3：使用@ComponentScan等注解

    ```java
    @ComponentScan(basePackages = {"com.yzxie.demo.springboot"})
    @EnableAutoConfiguration
    @EnableTransactionManagement
    @EnableWebMvc
    public class WebMainServer extends WebMvcConfigurerAdapter {
    
        @Override
        public void configureContentNegotiation(ContentNegotiationConfigurer config) {
            config.ignoreAcceptHeader(true);
        }
    
        @Bean
        public FilterRegistrationBean sourceApiFilter(RedisTemplate redisTemplate) {
            FilterRegistrationBean filterRegistrationBean = new FilterRegistrationBean();
            Filter filter = new SourceApiFilter(redisTemplate);
            filterRegistrationBean.setFilter(filter);
            filterRegistrationBean.addUrlPatterns(sourcesApis);
            return filterRegistrationBean;
        }
    
    
        public static void main(String[] args) {
            SpringApplication.run(WebMainServer.class, args);
        }
    }
    ```
* 典型用法4：创建SpringApplication对象

    ```java
    @SpringBootApplication
    public class MyApplication {
        public static void main(String[] args) {
            SpringApplication application = new SpringApplication(MyApplication.class);
            // ... customize application settings here
            application.run(args);
        }
    }
    ```

### 源码实现分析
* main方法所在的类，一般为一个基于@Configuration的配置类，然后作为参数传给SpringApplication.run方法。
* SpringApplication内部实现也是跟普通SpringMVC项目差不多，也是需要创建一个WebApplicationContext。那么如何将这个配置类注册到该WebApplicaiontContext当中呢？源码实现如下：
  1. 通过SpringApplication.run静态方法调用，在内部会创建一个SpringApplication对象实例，然后将该main方法传入的类添加到SpringApplication对象实例的source集合：primarySources，然后执行成员方法run：核心关注：prepareContext方法调用，即初始化ApplicationContext。
  
    ```java
	/**
	 * Run the Spring application, creating and refreshing a new
	 * {@link ApplicationContext}.
	 * @param args the application arguments (usually passed from a Java main method)
	 * @return a running {@link ApplicationContext}
	 */
	public ConfigurableApplicationContext run(String... args) {
		StopWatch stopWatch = new StopWatch();
		stopWatch.start();
		ConfigurableApplicationContext context = null;
		Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
		configureHeadlessProperty();
		SpringApplicationRunListeners listeners = getRunListeners(args);
		listeners.starting();
		try {
			ApplicationArguments applicationArguments = new DefaultApplicationArguments(
					args);
			ConfigurableEnvironment environment = prepareEnvironment(listeners,
					applicationArguments);
			configureIgnoreBeanInfo(environment);
			Banner printedBanner = printBanner(environment);
			context = createApplicationContext();
			exceptionReporters = getSpringFactoriesInstances(
					SpringBootExceptionReporter.class,
					new Class[] { ConfigurableApplicationContext.class }, context);
					
		    // prepareContext初始化context
		    
			prepareContext(context, environment, listeners, applicationArguments,
					printedBanner);
			refreshContext(context);
			afterRefresh(context, applicationArguments);
			stopWatch.stop();
			if (this.logStartupInfo) {
				new StartupInfoLogger(this.mainApplicationClass)
						.logStarted(getApplicationLog(), stopWatch);
			}
			listeners.started(context);
			callRunners(context, applicationArguments);
		}
		catch (Throwable ex) {
			handleRunFailure(context, ex, exceptionReporters, listeners);
			throw new IllegalStateException(ex);
		}

		try {
			listeners.running(context);
		}
		catch (Throwable ex) {
			handleRunFailure(context, ex, exceptionReporters, null);
			throw new IllegalStateException(ex);
		}
		return context;
	}
    ```
    2. prepareContext方法实现：核心关注load方法调用。
    
    ```java
    private void prepareContext(ConfigurableApplicationContext context,
    		ConfigurableEnvironment environment, SpringApplicationRunListeners listeners,
    		ApplicationArguments applicationArguments, Banner printedBanner) {
    	context.setEnvironment(environment);
    	postProcessApplicationContext(context);
    	applyInitializers(context);
    	listeners.contextPrepared(context);
    	if (this.logStartupInfo) {
    		logStartupInfo(context.getParent() == null);
    		logStartupProfileInfo(context);
    	}
    	// Add boot specific singleton beans
    	ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
    	beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
    	if (printedBanner != null) {
    		beanFactory.registerSingleton("springBootBanner", printedBanner);
    	}
    	if (beanFactory instanceof DefaultListableBeanFactory) {
    		((DefaultListableBeanFactory) beanFactory)
    				.setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
    	}
    	
    	// main方法对应的类，在静态run方法中
    	// 是注册为一个source
    	
    	// Load the sources
    	Set<Object> sources = getAllSources();
    	Assert.notEmpty(sources, "Sources must not be empty");
    	
    	// 加载source
    	
    	load(context, sources.toArray(new Object[0]));
    	listeners.contextLoaded(context);
    }
    ```
    3. load方法实现：
    
    ```java
    /**
     * Load beans into the application context.
     * @param context the context to load beans into
     * @param sources the sources to load
     */
    protected void load(ApplicationContext context, Object[] sources) {
    	if (logger.isDebugEnabled()) {
    		logger.debug(
    				"Loading source " + StringUtils.arrayToCommaDelimitedString(sources));
    	}
    	BeanDefinitionLoader loader = createBeanDefinitionLoader(
    			getBeanDefinitionRegistry(context), sources);
    	if (this.beanNameGenerator != null) {
    		loader.setBeanNameGenerator(this.beanNameGenerator);
    	}
    	if (this.resourceLoader != null) {
    		loader.setResourceLoader(this.resourceLoader);
    	}
    	if (this.environment != null) {
    		loader.setEnvironment(this.environment);
    	}
    	loader.load();
    }
    ```
    4. BeanDefinitionLoader的load方法实现：核心关注最后一个load方法的代码：this.annotatedReader.register(source);
    
    ```java
    /**
     * Load the sources into the reader.
     * @return the number of loaded beans
     */
    public int load() {
    	int count = 0;
    	for (Object source : this.sources) {
    		count += load(source);
    	}
    	return count;
    }
    
    private int load(Object source) {
    	Assert.notNull(source, "Source must not be null");
    	if (source instanceof Class<?>) {
    		return load((Class<?>) source);
    	}
    	if (source instanceof Resource) {
    		return load((Resource) source);
    	}
    	if (source instanceof Package) {
    		return load((Package) source);
    	}
    	if (source instanceof CharSequence) {
    		return load((CharSequence) source);
    	}
    	throw new IllegalArgumentException("Invalid source type " + source.getClass());
    }
    
    private int load(Class<?> source) {
    	if (isGroovyPresent()
    			&& GroovyBeanDefinitionSource.class.isAssignableFrom(source)) {
    		// Any GroovyLoaders added in beans{} DSL can contribute beans here
    		GroovyBeanDefinitionSource loader = BeanUtils.instantiateClass(source,
    				GroovyBeanDefinitionSource.class);
    		load(loader);
    	}
    	if (isComponent(source)) {
    		this.annotatedReader.register(source);
    		return 1;
    	}
    	return 0;
    }
    ```
    5. this.annotatedReader.register(source)：其中annotatedReader的类型为AnnotatedBeanDefinitionReader。annotatedReader生成source对应的类的BeanDefinition，然后将该BeanDefinition对象注册到底层的BeanFactory当中。
    
    6. main所在的类注册到了底层的BeanFactory当中了，但是这个类的注解如何处理？注意上面是将BeanDefinition注册到了BeanFactory当中。而在ApplicationContext启动时，下一步会调用BeanFactoryPostProcessor来对BeanFactory内部的BeanDefinitions进行加工处理。而是哪个BeanFactoryPostProcessor，还有什么时候注册了这个BeanFactoryPostProcessor呢？
    
    7. 由上分析可知，SpringApplication所使用的BeanDefinition解析注册器为annotatedReader的类型AnnotatedBeanDefinitionReader。AnnotatedBeanDefinitionReader的构造函数实现如下：核心关注AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry)，在这里注册了BeanFactoryPostProcessor。
    
    ```java
    public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry, Environment environment) {
    	Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
    	Assert.notNull(environment, "Environment must not be null");
    	this.registry = registry;
    	this.conditionEvaluator = new ConditionEvaluator(registry, environment, null);
    	AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
    }
    ```
    
    8. AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry)的实现：可以看出注册了ConfigurationClassPostProcessor。
    
    ```java
    public static Set<BeanDefinitionHolder> registerAnnotationConfigProcessors(
    			BeanDefinitionRegistry registry, @Nullable Object source) {
    
            ...
    
    		Set<BeanDefinitionHolder> beanDefs = new LinkedHashSet<>(8);
    
    		if (!registry.containsBeanDefinition(CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME)) {
    			RootBeanDefinition def = new RootBeanDefinition(ConfigurationClassPostProcessor.class);
    			def.setSource(source);
    			beanDefs.add(registerPostProcessor(registry, def, CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME));
    		}
    		
    		...
    		
    }
    ```
    
    9. ConfigurationClassPostProcessor：BeanFactoryPostProcessor接口的一个实现类，用来查找并处理BeanFactory中对应的类使用了 **==@Configuration或（@Component，@ComponentScan，@Import，@ImportResource）==** 的BeanDefinition。所以是在这里使main所在类的所有注解生效的。具体可以看我的另外一篇文章的分析[：Spring基于@Configuration的类配置的内部源码实现
](https://blog.csdn.net/u010013573/article/details/86663467)

    10. @SpringBootApplication注解的定义如下：包含三个注解：SpringBootConfiguration，EnableAutoConfiguration，ComponentScan；其中SpringBootConfiguration继承于@Configuration注解。
    
    ```java
    /**
     * Indicates a {@link Configuration configuration} class that declares one or more
     * {@link Bean @Bean} methods and also triggers {@link EnableAutoConfiguration
     * auto-configuration} and {@link ComponentScan component scanning}. This is a convenience
     * annotation that is equivalent to declaring {@code @Configuration},
     * {@code @EnableAutoConfiguration} and {@code @ComponentScan}.
     *
     * @author Phillip Webb
     * @author Stephane Nicoll
     * @since 1.2.0
     */
    @Target(ElementType.TYPE)
    @Retention(RetentionPolicy.RUNTIME)
    @Documented
    @Inherited
    @SpringBootConfiguration
    @EnableAutoConfiguration
    @ComponentScan(excludeFilters = {
    		@Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
    		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
    public @interface SpringBootApplication {
    ```