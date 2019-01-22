---
layout:     post
title:      "Spring源码分析（五）：ApplicationContext体系结构设计之自底向上分析"
subtitle:   "Spring ApplicationContext design from bottom to up"
date:       2019-01-08 20:49:00
author:     "XYZ"
header-img: "img/article-bg1.jpg"
tags:
    - Spring源码分析
---


### spring-context包
#### 1. ApplicationContext接口
```java
public interface ApplicationContext extends EnvironmentCapable, ListableBeanFactory, HierarchicalBeanFactory,
		MessageSource, ApplicationEventPublisher, ResourcePatternResolver
```
* 最底层接口，通过继承BeanFactory接口的方法，定义了与BeanFactory的关联绑定，以及其他功能组件，如Environment，MessageSource等的关联。
* ApplicationContext是bean容器的一个运行环境，而实际的bean容器为内部绑定的BeanFactory，由BeanFactory来存放bean的元数据beanDefinitions，具体存放在BeanFactory的实现类的一个类型为ConcurrentHashMap的map中，其中key为beanName，value为BeanDefinition；以及bean实例的创建。

#### 2. ConfigurableApplicationContext接口

```java
public interface ConfigurableApplicationContext extends ApplicationContext, Lifecycle, Closeable
```

* 继承于ApplicationContext接口，提供与applicationListener，environment，beanFactoryProcessor等相关的get/set方法，还有启动入口方法refresh。
* 从ApplicationContext接口额外派生这个接口，而不是直接在ApplicationContext接口声明这些的原因：这些组件都是ApplicationContext接口的实现类在内部自身使用的，而ApplicationContext接口主要是定义对外的功能和方法声明，故在ConfigurableApplicationContext接口中声明这些方法，保证接口的清晰和职责的明确。

#### 3. AbstractApplicationContext抽象类

```java
public abstract class AbstractApplicationContext extends DefaultResourceLoader
		implements ConfigurableApplicationContext
```

* 实现ConfigurableApplicationContext接口。
* refresh方法：容器启动的骨架实现，使用了模板设计模式。提供对ConfigurableApplicationContext接口的refresh方法的模板实现，即定义了ApplicationContext的启动步骤，但是不提供具体每步的实现，由子类提供。
* 成员变量定义：定义了applicationListener，environment，beanFactoryProcessor等相关的成员变量。

#### 4. AbstractRefreshableApplicationContext抽象类

```java
public abstract class AbstractRefreshableApplicationContext extends AbstractApplicationContext
```

* 继承于AbstractApplicationContext抽象类，主要提供可多次重复调用refresh方法，刷新容器，销毁之前的beanFactory和beans，重新创建beanFactory和beans，如在对xml配置文件修改过了之后重新加载。ConfigurableApplicationContext接口的定义，默认是不支持多次调用refresh方法的，多次调用则抛IllegalStateException异常。

#### 5. AbstractRefreshableConfigApplicationContext抽象类

```java
public abstract class AbstractRefreshableConfigApplicationContext extends AbstractRefreshableApplicationContext
		implements BeanNameAware, InitializingBean
```

* 继承于AbstractRefreshableApplicationContext抽象类，定义数组类型的configLocations成员变量，以及set/get方法，用于保存xml配置文件存放的地址。故可以在外部定义多个xml文件来配置需要注册的bean。

#### 6. AbstractXmlApplicationContext抽象类

```java
public abstract class AbstractXmlApplicationContext extends AbstractRefreshableConfigApplicationContext
```

* 继承于AbstractRefreshableConfigApplicationContext，指定使用xml文件保存配置。
   * ClassPathXmlApplicationContext：具体实现类，继承于AbstractXmlApplicationContext抽象类，指定从类路径下加载xml配置文件。
   * FileSystemXmlApplicationContext：具体实现类，继承于AbstractXmlApplicationContext，指定从文件系统加载xml配置文件。

#### 7. GenericApplicationContext：通用具体实现类

```java
public class GenericApplicationContext extends AbstractApplicationContext implements BeanDefinitionRegistry
```

* 继承于AbstractApplicationContext抽象类，故不具备上面的Config，Refreshable功能。而Bean的注册，是通过实现BeanDefinitionRegistry接口来提供往内部的beanFactory注入beanDefinitions，而beanDefinitions的来源则是通过BeanDefinitionParser解析，如xml文件来获取的。不支持重复调用refresh。例子如下：

    ```java
    GenericApplicationContext ctx = new GenericApplicationContext();
    
    XmlBeanDefinitionReader xmlReader = new XmlBeanDefinitionReader(ctx);
    xmlReader.loadBeanDefinitions(new ClassPathResource("applicationContext.xml"));
    
    PropertiesBeanDefinitionReader propReader = new PropertiesBeanDefinitionReader(ctx);
    propReader.loadBeanDefinitions(new ClassPathResource("otherBeans.properties"));
    
    ctx.refresh();
    ```
    * GenericXmlApplicationContext：继承与GenericApplicationContext，比ClassPathXmlApplicationContext，FileSystemXmlApplicationContext更加通用的基于xml配置文件的ApplicationContext。即可以在构造函数中指定配置数来源，使用的Resource类型的数组参数。而前两者都是使用String类型的configLocations数组，即路径数组。
    
#### 8. AnnotationConfigApplicationContext：注解驱动的实现类

```java
public class AnnotationConfigApplicationContext extends GenericApplicationContext implements AnnotationConfigRegistry 
```

* 这个也是用来处理我们项目中使用的注解的类，即将入@Controller,@Component等注解的类注册成bean。继承于GenericApplicationContext，并实现AnnotationConfigRegistry接口（AnnotationConfigRegistry接口主要实现将给定的beanDefinition注册到绑定的beanFactory中）。
* 使用注解类型的数组作为构造函数参数，表示将使用这些注解修饰的类将是需要注册的bean，注解类型包含@Configuration，@Component，JSR-330的inject，以及相关的派生注解。
   
   ```java
    // 注册指定注解的类作为bean，如@Configuration，@Component等
    
	/**
	 * Create a new AnnotationConfigApplicationContext, deriving bean definitions
	 * from the given annotated classes and automatically refreshing the context.
	 * @param annotatedClasses one or more annotated classes,
	 * e.g. {@link Configuration @Configuration} classes
	 */
	public AnnotationConfigApplicationContext(Class<?>... annotatedClasses) {
		this();
		register(annotatedClasses);
		refresh();
	}
   ```
   
* 使用包的数组作为构造函数参数，表示扫描这些包下面的类，将需要注册的类注册成bean。处理basePackages的包下面的@Component，以及派生的如@Service，@Controller，@Repository等，具体可以通过includeFilters和excludeFilters来配置需要哪些注解的类和排除哪些注解的类。includeFilters和excludeFilters的相关配置可以参考[：includeFilters和excludeFilters](https://blog.csdn.net/lindiwo/article/details/53406624)，[context:exclude-filter 与 context:include-filter](https://blog.csdn.net/liuwenbo0920/article/details/7260013)
    
    ```java
   
	/**
	 * Create a new AnnotationConfigApplicationContext, scanning for bean definitions
	 * in the given packages and automatically refreshing the context.
	 * @param basePackages the packages to check for annotated classes
	 */
	public AnnotationConfigApplicationContext(String... basePackages) {
		this();
		scan(basePackages);
		refresh();
	}
    ```

### spring-web包
* spring-web包提供了一个ApplicationContext的继承接口：WebApplicationContext的类体系设计与ApplicationContext的基本一致，不同之处为：在ApplicationContext的基础上，添加web应用运行相关的一些性质。主要与web容器、servlet相关，核心性质包括：

* ServletContext
   1. 即该应用在web容器的中运行环境类，ServletContext的启动，触发spring容器WebApplicationContext的启动，即spring容器的创建。
   2. 将ServletContext中的context-param的键值对数据，放到WebApplicationContext的environment中。servlet相关的则是servletConfig的init-param的键值对数据。
    
* WebApplicationContext关联到ServletContext：Spring容器WebApplicationContext对象，作为ServletContext的一个属性关联到ServletContext，属性key为：
   1. root WebApplicationContext：即包含ROOT
   
    ```java
    String ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE = WebApplicationContext.class.getName() + ".ROOT";
    ```
   2. spring-mvc包的FrameworkServlet（或者是DispatcherServlet）所绑定WebApplicationContext为：即包含CONTEXT
   
    ```java
    /** Should we publish the context as a ServletContext attribute?. */
    private boolean publishContext = true;
    	
    protected WebApplicationContext initWebApplicationContext() {
    	
    	...
    
    	if (this.publishContext) {
    		// Publish the context as a servlet context attribute.
    		String attrName = getServletContextAttributeName();
    		getServletContext().setAttribute(attrName, wac);
    	}
    
    	return wac;
    }
    
    /**
     * Return the ServletContext attribute name for this servlet's WebApplicationContext.
     * <p>The default implementation returns
     * {@code SERVLET_CONTEXT_PREFIX + servlet name}.
     * @see #SERVLET_CONTEXT_PREFIX
     * @see #getServletName
     */
    public String getServletContextAttributeName() {
    	return SERVLET_CONTEXT_PREFIX + getServletName();
    }
    
    /**
     * Prefix for the ServletContext attribute for the WebApplicationContext.
     * The completion is the servlet name.
     */
    public static final String SERVLET_CONTEXT_PREFIX = FrameworkServlet.class.getName() + ".CONTEXT.";
    ```

    从而可以ServletContext可以访问WebApplicationContext中的bean。
    
* ServletConfig
   1. spring-web包的WebApplicationContext与spring-context包的ApplicationContext一样，通常也是支持层次化的。在web包中，通常包含一个root WebApplicationContext，每个DispatcherServlet绑定一个独立的WebApplicationContext。
   2. 而作为一个servlet，在web容器中，通常会绑定一个servletConfig来指定该servlet的一些属性，如在web.xml配置这个servlet时，通过init-param标签来指定。所以在WebApplicationContext中，需要在DispatcherServlet绑定的WebApplicationContext中，将与DispatcherServlet绑定的servletConfig中相关的键值对数据，放到该WebApplicationContext的environment中。

* bean配置configLocations固定：
   1. spring容器的配置通常放在类路径的WEB-INF/applicationContext.xml或者如果是DispatcherServlet，通常为WEB-INF/"servletName"-servlet.xml。或者在web.xml中通过context-param或者servlet的init-param标签，使用contextConfigLocation作为param-name，param-value为具体文件位置或者WebApplicationInitializer接口实现类全限定名。
   2. 如果是编程方式，则通常是从WebApplicationInitializer接口的实现类中指定并注入。
