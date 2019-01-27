---
layout:     post
title:      "Spring源码分析（六）：Bean工厂体系结构设计"
subtitle:   "Spring BeanFactory"
date:       2019-01-12 20:49:00
author:     "XYZ"
header-img: "img/article-bg1.jpg"
tags:
    - Spring源码分析
---
### 一. 概述
* Spring容器通常指的是ApplicationContext的体系结构设计，即整个Spring框架的IOC功能，是通过ApplicationContext接口实现类来提供给应用程序使用的。应用程序通过ApplicationContext提供方法来间接与内部Bean工厂交互，如获取Bean对象实例等。
* 在Spring框架内部设计当中，ApplicationContext是Spring容器所管理、维护的beans对象的一个运行环境，即ApplicationContext包含一些功能组件：保存外部属性文件（properties文件，yml文件等）的属性键值对集合的Environment，容器配置的位置contextConfigLocation等等，用于创建bean对象需要的一些外部依赖；
* 而ApplicationContext内部最重要的组件，就是BeanFactory体系结构，ApplicationContext通过BeanFactory来维护Spring容器所管理的类对应的BeanDefintions，通过BeanFactory来获取类对象bean。与BeanFactory配套的就是ApplicationContext维护多个BeanFactoryPostProcessor，BeanPostProcessor来对BeanFactory进行拓展，对BeanFactory自身或所创建的bean对象实例进行加工、功能拓展，实现整体设计的高拓展性。

### 二. BeanFactory接口设计
#### BeanFactory
* 顶层接口，主要提供getBean方法，从该BeanFactory获取给定beanName，对应的bean对象实例；

#### ListableBeanFactory
* 继承于BeanFactory，主要提供根据给定条件，如type，Annotation，获取对应的所有beans列表的接口；

#### HierarchicalBeanFactory
* 继承于BeanFactory，提供获取parentBeanFactory的方法，实现BeanFactory的层次化功能；

#### ConfigurableBeanFactory
* 继承于HierarchicalBeanFactory接口，主要提供对BeanFactory进行相关配置的接口，如类加载器classLoader，beanPostProcessor，类型转换器，属性编辑器等在加载、创建和初始化bean实例时，需要用到的一些功能组件；

#### AutowireCapableBeanFactory
* 继承于BeanFactory，主要提供对于BeanFactory创建的bean，使用autowire的方式对其所依赖的beans进行依赖注入。

#### 接口具体实现类
##### 1. DefaultListableBeanFactory
* BeanFactory接口体系的默认实现类，实现以上接口的功能，提供BeanDefinition的存储map，Bean对象对象的存储map。
* 其中Bean对象实例的存储map，定义在FactoryBeanRegistrySupport，FactoryBeanRegistrySupport实现了SingletonBeanRegistry接口，而DefaultListableBeanFactory的基类AbstractBeanFactory，继承于FactoryBeanRegistrySupport。

	```java
	public class DefaultListableBeanFactory extends AbstractAutowireCapableBeanFactory
			implements ConfigurableListableBeanFactory, BeanDefinitionRegistry, Serializable {
	    
	    ...
	
		/** Map from serialized id to factory instance. */
		private static final Map<String, Reference<DefaultListableBeanFactory>> serializableFactories =
				new ConcurrentHashMap<>(8);
	
	    ...
	
		/** Map from dependency type to corresponding autowired value. */
		private final Map<Class<?>, Object> resolvableDependencies = new ConcurrentHashMap<>(16);
	
		/** Map of bean definition objects, keyed by bean name. */
		private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>(256);
	
		/** Map of singleton and non-singleton bean names, keyed by dependency type. */
		private final Map<Class<?>, String[]> allBeanNamesByType = new ConcurrentHashMap<>(64);
	
		/** Map of singleton-only bean names, keyed by dependency type. */
		private final Map<Class<?>, String[]> singletonBeanNamesByType = new ConcurrentHashMap<>(64);
	
		/** List of bean definition names, in registration order. */
		private volatile List<String> beanDefinitionNames = new ArrayList<>(256);
	
		/** List of names of manually registered singletons, in registration order. */
		private volatile Set<String> manualSingletonNames = new LinkedHashSet<>(16);
	
		/** Cached array of bean definition names in case of frozen configuration. */
		@Nullable
		private volatile String[] frozenBeanDefinitionNames;
	
		/** Whether bean definition metadata may be cached for all beans. */
		private volatile boolean configurationFrozen = false;
		
		...
		
	}
	```
##### 2. StaticListableBeanFactory
* 用于存储给定的bean对象实例，不支持动态注册功能，是ListableBeanFactory接口的简单实现。

    ```java
    public class StaticListableBeanFactory implements ListableBeanFactory {
    
    	/** Map from bean name to bean instance. */
    	private final Map<String, Object> beans;
    	
    	...
    	
    }
    ```
#### Bean的获取与注册的接口设计隔离
* 以上BeanFactory接口层次主要是以Bean的获取为主的设计。而注册是在BeanDefinitionRegistr和SingletonBeanRegistry中设计的。
##### bean的获取
* 在BeanFactory的接口继承体系中，主要是提供获取bean，如getBean；列举bean，如ListableBeanFactory；提供BeanFactory创建bean需要的组件的ConfigurableBeanFactory；以及对bean注入其他beans的AutowireCapableBeanFactory。
* 但是没有提供注册bean的方法声明，即将BeanDefinition注册到BeanFactory实现类内部维护的ConcurrentHashMap类型的map中。

##### bean的注册
###### 1. BeanDefinition的注册
* 提供BeanDefinition注册功能的是BeanDefinitionRegistry接口，在这个接口定义注册beanDefinition到BeanFactory的方法声明。
* BeanFactory的实现类会实现BeanDefinitionRegistry，并实现BeanDefinitionRegistry接口的registerBeanDefinition系列方法来将给定的BeanDefinition注册到BeanFactory中。

###### 2. Bean对象实例的注册
* 在SingletonBeanRegistry接口的实现类中提供存储的map和注册方法，BeanFactory实现SingletonBeanRegistry接口。

### 三. BeanDefinitionRegistry接口
* 注册BeanDefinitions。提供registerBeanDefinition，removeBeanDefinition等方法，用来从BeanFactory注册或移除BeanDefinition。
* 通常BeanFactory接口的实现类需要实现这个接口。
* 实现类（通常为BeanFactory接口实现类）的对象实例，被DefinitionReader接口实现类引用，DefinitionReader将BeanDefintion注册到该对象实例中。

### 四. SingletonBeanRegistry接口
* 用于注册单例Bean对象实例，实现类定义存储Bean对象实例的map，BeanFactory的类层次结构中需要实现这个接口，来提供Bean对象的注册和从Bean对象实例的map获取bean对象。

    ```java
    /**
     * Support base class for singleton registries which need to handle
     * {@link org.springframework.beans.factory.FactoryBean} instances,
     * integrated with {@link DefaultSingletonBeanRegistry}'s singleton management.
     *
     * <p>Serves as base class for {@link AbstractBeanFactory}.
     *
     * @author Juergen Hoeller
     * @since 2.5.1
     */
    public abstract class FactoryBeanRegistrySupport extends DefaultSingletonBeanRegistry {
    
    	/** Cache of singleton objects created by FactoryBeans: FactoryBean name to object. */
    	private final Map<String, Object> factoryBeanObjectCache = new ConcurrentHashMap<>(16);
    	
    	...
    	
    }
    ```
### 五. BeanFactoryPostProcessor接口设计
* 主要是BeanFactory加载、创建好BeanDefinitions之后，对BeanFactory保存的这些BeanDefinitions进行加工处理。除此之外也可以对BeanFactory进行其他拓展实现。
* 典型实现类：ConfigurationClassPostProcessor，用于从BeanFactory中检测使用了@Configuration注解的类，对于这些类对应的BeanDefinitions集合，遍历并依次交给ConfigurationClassParser，ConfigurationClassBeanDefinitionReader处理，分别是处理与@Configuration同时使用的其他注解和将类内部的使用@Bean注解的方法，生成BeanDefinition，注册到BeanFactory。

### 六. BeanPostProcessor接口设计
* 通常用于在创建好bean对象实例后，处理这个bean上面的注解。同时也可以对bean对象进行其他功能拓展。
* BeanPostProcessor的注册
   1. 定义：在注解配置工具类AnnotationConfigUtils的静态方法registerAnnotationConfigProcessors方法中，定义注解的处理器的注册逻辑。
   2. 调用：在BeanFactoryPostProcessor中调用这个静态方法来完成将特定的BeanPostProcessor实现类，注册到ApplicationContext的BeanPostProcessor列表。

1. AutowiredAnnotationBeanPostProcessor：处理bean对象的依赖注入关系，即从BeanFactory获取该bean所依赖的bean，然后注入到该bean对应的成员变量中。

2. CommonAnnotationBeanPostProcessor：该bean中所使用了的JDK定义的注解的处理，如方法中的@PostConstruct，@PreDestroy，成员变量上的@Resource等。

3. PersistenceAnnotationBeanPostProcessor（JPA时添加）：JPA相关bean的持久化处理。

### 七. BeanDefinitionReader接口
* 从xml文件、类路径下使用了@Component系列注解的类、或者从@Configuration注解的配置类，获取BeanDefintiions，然后注册到BeanFactory中。

#### 1. XmlBeanDefinitionReader：基于XML文件
* 读取解析xml文件，通过Parser解析xml文件的标签。
* 针对beans标签，生成对应的BeanDefintions，然后注册到BeanFactory中。
* 针对其他有特殊功能的标签，如context:component-scan，context:anotation-config，还可以生成BeanFactoryPostProcessor，BeanPostProcessor接口实现类的bean等；除了可以生成BeanDefinitions之外，还可以实现其他功能。

##### NamespaceHandler
* 被XmlBeanDefinitionReader使用，XmlBeanDefinitionReader通过一个DefaultNamespaceHandlerResolver来获取给定标签对应的Parser。
* NamespaceHandler维护了xml的标签与标签处理器Parser之间的映射关系，默认在META-INF/spring.handlers文件中，定义标签名称空间与NamespaceHandler实现类之间的映射关系：
    ```
    http\://www.springframework.org/schema/c=org.springframework.beans.factory.xml.SimpleConstructorNamespaceHandler
    http\://www.springframework.org/schema/p=org.springframework.beans.factory.xml.SimplePropertyNamespaceHandler
    http\://www.springframework.org/schema/util=org.springframework.beans.factory.xml.UtilNamespaceHandler
    ```
* 主要是在BeanDefinitionDocumentReader中在解析Document对象的Element元素时调用，如下：

    ```java
    public BeanDefinition parseCustomElement(Element ele, @Nullable BeanDefinition containingBd) {
    	String namespaceUri = getNamespaceURI(ele);
    	if (namespaceUri == null) {
    		return null;
    	}
    	
    	// 获取一个NamespaceHandler
    	
    	NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
    	if (handler == null) {
    		error("Unable to locate Spring NamespaceHandler for XML schema namespace [" + namespaceUri + "]", ele);
    		return null;
    	}
    	
    	// 通过这个NamespaceHandler
    	// 在它里面维护的parser集合中找到
    	// 与该标签对应的parser，由该parser来执行解析
    	
    	return handler.parse(ele, new ParserContext(this.readerContext, this, containingBd));
    }
    ```
* 以上已经说明了通过XmlBeanDefinitionReader的NamespaceHandlerResolver维护：命名空间和NamespaceHandler的映射；通过NamespaceHandler来维护标签和Parser的映射，那么两者是什么时候初始化注册的呢？
* 其实NamespaceHandler和Parser的初始化，使用的是懒加载机制，即当调用了NamespaceHandlerResolver的resolve方法时，才进行加载，加载之后进行缓存。如下：
   1. NamespaceHandler的初始化：在DefaultNamespaceHandlerResolver的getHandlerMappings方法实现。在resolve方法中调用getHandlerMappings方法。
   
        ```java
    	/**
    	 * Load the specified NamespaceHandler mappings lazily.
    	 */
    	private Map<String, Object> getHandlerMappings() {
    		Map<String, Object> handlerMappings = this.handlerMappings;
    		if (handlerMappings == null) {
    			synchronized (this) {
    				handlerMappings = this.handlerMappings;
    				if (handlerMappings == null) {
    					if (logger.isTraceEnabled()) {
    						logger.trace("Loading NamespaceHandler mappings from [" + this.handlerMappingsLocation + "]");
    					}
    					try {
    						Properties mappings =
    								PropertiesLoaderUtils.loadAllProperties(this.handlerMappingsLocation, this.classLoader);
    						if (logger.isTraceEnabled()) {
    							logger.trace("Loaded NamespaceHandler mappings: " + mappings);
    						}
    						handlerMappings = new ConcurrentHashMap<>(mappings.size());
    						CollectionUtils.mergePropertiesIntoMap(mappings, handlerMappings);
    						this.handlerMappings = handlerMappings;
    					}
    					catch (IOException ex) {
    						throw new IllegalStateException(
    								"Unable to load NamespaceHandler mappings from location [" + this.handlerMappingsLocation + "]", ex);
    					}
    				}
    			}
    		}
    		return handlerMappings;
    	}
        ```
        1. 从META-INF/spring.handlers文件加载键值对，并缓存在类型为ConcurrentHashMap的handlerMappings中；
        2. 注意这里并没有初始化NamespaceHandler，即handlerMappings的value还是String类型。
       
   2. NamespaceHandler包含的parsers的初始化：在resolve方法中进行懒加载初始化。
   
   
	     ```java
	      /**
	       * Locate the {@link NamespaceHandler} for the supplied namespace URI
	       * from the configured mappings.
	       * @param namespaceUri the relevant namespace URI
	       * @return the located {@link NamespaceHandler}, or {@code null} if none found
	       */
	      @Override
	      @Nullable
	      public NamespaceHandler resolve(String namespaceUri) {
	      
	          // 懒加载NamespaceHandler的handlerMappings
	          
	      	Map<String, Object> handlerMappings = getHandlerMappings();
	      	Object handlerOrClassName = handlerMappings.get(namespaceUri);
	      	if (handlerOrClassName == null) {
	      		return null;
	      	}
	      	
	      	// 不是第一次调用，则已经是NamespaceHandler类型了，可以直接返回
	      	
	      	else if (handlerOrClassName instanceof NamespaceHandler) {
	      		return (NamespaceHandler) handlerOrClassName;
	      	}
	      	
	      	// 第一次调用，由上面分析可知：
	      	// 刚开始从META-INF/spring.handlers
	      	// 读出时，handlerMappings的value是字符串
	      	
	      	else {
	      		String className = (String) handlerOrClassName;
	      		try {
	      		
	      		// 加载NamespaceHandler
	      		
	      			Class<?> handlerClass = ClassUtils.forName(className, this.classLoader);
	      			if (!NamespaceHandler.class.isAssignableFrom(handlerClass)) {
	      				throw new FatalBeanException("Class [" + className + "] for namespace [" + namespaceUri +
	      						"] does not implement the [" + NamespaceHandler.class.getName() + "] interface");
	      			}
	      			NamespaceHandler namespaceHandler = (NamespaceHandler) BeanUtils.instantiateClass(handlerClass);
	      			
	      			// 调用init方法完成parsers的初始化
	      			
	      			namespaceHandler.init();
	      			handlerMappings.put(namespaceUri, namespaceHandler);
	      			return namespaceHandler;
	      		}
	      		catch (ClassNotFoundException ex) {
	      			throw new FatalBeanException("Could not find NamespaceHandler class [" + className +
	      					"] for namespace [" + namespaceUri + "]", ex);
	      		}
	      		catch (LinkageError err) {
	      			throw new FatalBeanException("Unresolvable class definition for NamespaceHandler class [" +
	      					className + "] for namespace [" + namespaceUri + "]", err);
	      		}
	      	}
	      }
	      
	  	protected final void registerBeanDefinitionParser(String elementName, BeanDefinitionParser parser) {
	       this.parsers.put(elementName, parser);
	   }
	   ```
     3. NamespaceHandler的init方法实现：各个NamespaceHandler接口实现类，在init方法中注册xml的标签和Parser之间的映射关系：如下为context标签的名称空间处理器ContextNamespaceHandler：
        
        ```java
        /**
         * {@link org.springframework.beans.factory.xml.NamespaceHandler}
         * for the '{@code context}' namespace.
         *
         * @author Mark Fisher
         * @author Juergen Hoeller
         * @since 2.5
         */
        public class ContextNamespaceHandler extends NamespaceHandlerSupport {
        
        	@Override
        	public void init() {
        		registerBeanDefinitionParser("property-placeholder", new PropertyPlaceholderBeanDefinitionParser());
        		registerBeanDefinitionParser("property-override", new PropertyOverrideBeanDefinitionParser());
        		registerBeanDefinitionParser("annotation-config", new AnnotationConfigBeanDefinitionParser());
        		
        		// <context:component-scan />
        	
            	registerBeanDefinitionParser("component-scan", new ComponentScanBeanDefinitionParser());
        		registerBeanDefinitionParser("load-time-weaver", new LoadTimeWeaverBeanDefinitionParser());
        		registerBeanDefinitionParser("spring-configured", new SpringConfiguredBeanDefinitionParser());
        		registerBeanDefinitionParser("mbean-export", new MBeanExportBeanDefinitionParser());
        		registerBeanDefinitionParser("mbean-server", new MBeanServerBeanDefinitionParser());
        	}
        
        }
        ```
##### BeanDefinitionDocumentReader
* 被XmlBeanDefinitionReader使用，专门用于处理xml文件的beans标签的标签处理器。即XmlBeanDefinitionReader读取xml文件，创建Document对象，然后交给BeanDefinitionDocumentReader处理。
* BeanDefinitionDocumentReader解析Document对象的Element节点，然后创建BeanDefinitions集合，通过XmlBeanDefinitionReader注册到XmlBeanDefinitionReader所在的BeanFactory。

#### 2. AnnotatedBeanDefinitionReader与ClassPathBeanDefinitionScanner：基于注解的扫描
* 都是根据类来决定是否需要生成该类的BeanDefinition注册到BeanFactory，而不是像xml文件一样已经通过bean标签显示说明这个就是bean。
* 主要是被AnnotationConfigApplicationContext使用，即基于注解的ApplicationContext。基于XML的ApplicationContext使用基于注解的扫描是通过添加context:component-scan标签来实现的，底层使用ComponentScanBeanDefinitionParser这个parser来处理。
* ClassPathBeanDefinitionScanner：扫描给定的类路径下面的类，检测是否存在@Component注解及其子注解，从而生成BeanDefinition，然后注册到BeanFactory。
* AnnotatedBeanDefinitionReader：显示指定将哪些类注册到BeanFactory，如先获取所有使用了@Configuration注解的类，然后通过AnnotatedBeanDefinitionReader生成与这些类对应的BeanDefinitions，并注册到BeanFactory。

#### 3. ConfigurationClassBeanDefinitionReader：基于@Configuration的注解配置
* 处理@Configuration注解的配置类，上面的注解，即与@Configuration一起使用的注解，如@ComponentScan，@PropertySource，@Import等。

### 八. BeanDefinitionParser接口
* xml文件的标签的解析处理器，通过实现 BeanDefinitionParser接口，来针对每个标签进行特定。
* 典型用途包括：生成BeanDefintion对象，或BeanFactoryPostProcessor对象，或BeanPostProcessor对象，或者为针对标签定义特定的功能，自定义该标签的用途。
* Parser可以同时生成这三种类型中的一个或多个。如ComponentScanBeanDefinitionParser既生成BeanDefinitions，又生成ConfigurationClassPostProcessor（BeanFactoryPostProcessor接口实现类）。AnnotationConfigBeanDefinitionParser既生成BeanPostProcessor，又生成ConfigurationClassPostProcessor。

#### 1. 直接创建BeanDefintion

##### DefaultBeanDefinitionDocumentReader：xml文件解析
* 本身不是BeanDefinitionParser接口的实现类，而是利用BeanDefinitionParser接口的实现类，解析xml文件的beans标签，及其里边嵌套的bean标签，获取BeanDefinitions。

##### ComponentScanBeanDefinitionParser
* 处理xml文件的context:component-scan标签，扫描basePackages属性指定的路径下面的类，检测并获取使用了@Component注解及其子注解的类，生成对应的BeanDefinitions。

##### ComponentScanAnnotationParser
* 处理@ComponentScan注解，与ComponentScanBeanDefinitionParser功能一样，一般是与@Configuration注解一起使用。在ConfigurationClassParser中创建ComponentScanAnnotationParser对象实例并在parse方法中调用，ConfigurationClassParser为处理@Configuration注解的类。

#### 2. 创建BeanFactoryPostProcessor
##### ConfigurationClassPostProcessor：间接创建BeanDefinition
* ConfigurationClassPostProcessor是一个BeanFactoryPostProcessor接口的实现类。
* 在ComponentScanBeanDefinitionParser和AnnotationConfigBeanDefinitionParser中会创建ConfigurationClassPostProcessor对象实例：这样由这些parser创建的BeanDefinitions，就可以被ConfigurationClassPostProcessor进一步处理，创建更多的BeanDefintions。
   1. ComponentScanBeanDefinitionParser：对应context:component-scan，在parse方法调用；
   2. AnnotationConfigBeanDefinitionParser：对应context:annotation-config，在parse方法调用；
   3. ClassPathBeanDefinitionScanner：在scan方法调用，该类在AnnotationConfigApplicationContext中使用；
   4. AnnotatedBeanDefinitionReader：在构造函数调用，该类在AnnotationConfigApplicationContext中使用。

* 从BeanFactory的BeanDefintions集合，过滤获取使用了@Configuration注解的类，然后从这些类的@Bean方法获取BeanDefintions。

#### 3. 创建BeanPostProcessor
* AnnotationConfigBeanDefinitionParser：处理context:annotation-config标签，产生相关的BeanPostProcessor：
* ComponentScanBeanDefinitionParser：处理xml文件的context:component-scan标签。
* 以上两个parser都会产生以下两个BeanPostProcessor：
   1. CommonAnnotationBeanPostProcessor：处理bean对象及其方法中的JDK自身的注解；
   2. AutowiredAnnotationBeanPostProcessor：bean对象的依赖注入处理，如@Autowired。

#### 4. 自定义功能拓展
* 可以通过自定义标签和自定义Parser接口的实现类，并结合NamespaceHandler来实现自动融入spring框架。如dubbo框架就利用了这个特性来实现自动融入spring框架。步骤如下：
   1. 在自身项目，自定义标签xsd，自定义Parser实现和NamespaceHandler实现（继承NamespaceHandlerSupport），在init方法中注册该自定义parser；
   2. 在自身项目的jar包的添加META-INF/spring.handlers文件，里面定义自身命名空间xsd的位置和自定义的NamespaceHandler的映射关系。