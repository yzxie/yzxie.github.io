---
layout:     post
title:      "Spring源码分析（五）：spring-context包的ApplicationContext体系自底向上的设计"
subtitle:   "Spring ApplicationContext design from bottom to up"
date:       2019-01-08 20:49:00
author:     "XYZ"
header-img: "img/article-bg1.jpg"
tags:
    - Spring源码分析
---
#### ApplicationContext接口
```java
public interface ApplicationContext extends EnvironmentCapable, ListableBeanFactory, HierarchicalBeanFactory,
		MessageSource, ApplicationEventPublisher, ResourcePatternResolver
```
* 最底层接口，通过继承BeanFactory接口的方法，定义了与BeanFactory的关联绑定，以及其他功能组件，如Environment，MessageSource等的关联。

#### ConfigurableApplicationContext接口

```java
public interface ConfigurableApplicationContext extends ApplicationContext, Lifecycle, Closeable
```

* 继承于ApplicationContext接口，提供与applicationListener，environment，beanFactoryProcessor等相关的get/set方法，还有启动入口方法refresh。
* 从ApplicationContext接口额外派生这个接口，而不是直接在ApplicationContext接口声明这些的原因：这些组件都是ApplicationContext接口的实现类在内部自身使用的，而ApplicationContext接口主要是定义对外的功能和方法声明，故在ConfigurableApplicationContext接口中声明这些方法，保证接口的清晰和职责的明确。

#### AbstractApplicationContext抽象类

```java
public abstract class AbstractApplicationContext extends DefaultResourceLoader
		implements ConfigurableApplicationContext
```

* 实现ConfigurableApplicationContext接口。
* refresh方法：容器启动的骨架实现，使用了模板设计模式。提供对ConfigurableApplicationContext接口的refresh方法的模板实现，即定义了ApplicationContext的启动步骤，但是不提供具体每步的实现，由子类提供。
* 成员变量定义：定义了applicationListener，environment，beanFactoryProcessor等相关的成员变量。

#### AbstractRefreshableApplicationContext抽象类

```java
public abstract class AbstractRefreshableApplicationContext extends AbstractApplicationContext
```

* 继承于AbstractApplicationContext抽象类，主要提供可多次重复调用refresh方法，刷新容器，销毁之前的beanFactory和beans，重新创建beanFactory和beans，如在对xml配置文件修改过了之后重新加载。ConfigurableApplicationContext接口的定义，默认是不支持多次调用refresh方法的，多次调用则抛IllegalStateException异常。

#### AbstractRefreshableConfigApplicationContext抽象类

```java
public abstract class AbstractRefreshableConfigApplicationContext extends AbstractRefreshableApplicationContext
		implements BeanNameAware, InitializingBean
```

* 继承于AbstractRefreshableApplicationContext抽象类，定义数组类型的configLocations成员变量，以及set/get方法，用于保存xml配置文件存放的地址。故可以在外部定义多个xml文件来配置需要注册的bean。

#### AbstractXmlApplicationContext抽象类

```java
public abstract class AbstractXmlApplicationContext extends AbstractRefreshableConfigApplicationContext
```

* 继承于AbstractRefreshableConfigApplicationContext，指定使用xml文件保存配置。
   * ClassPathXmlApplicationContext：具体实现类，继承于AbstractXmlApplicationContext抽象类，指定从类路径下加载xml配置文件。
   * FileSystemXmlApplicationContext：具体实现类，继承于AbstractXmlApplicationContext，指定从文件系统加载xml配置文件。

#### GenericApplicationContext：通用具体实现类

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
    
#### AnnotationConfigApplicationContext：注解驱动的实现类

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