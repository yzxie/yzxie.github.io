---
layout:     post
title:      "Dubbo源码分析：全透明接入spring和ServiceBean和Referencebean的接口设计与实现"
subtitle:   "Dubbo Spring"
date:       2018-12-07 22:12:00
author:     "XYZ"
header-img: "img/article-bg1.jpg"
tags:
    - Dubbo源码分析
---
### Spring Schema XML拓展机制：dubbo全透明融入spring的实现基础

* 在spring项目启动，会加载并解析resources目录下的xml，然后将xml配置文件中的配置加载成spring容器的bean，dubbo需要定义dubbo_provider.xml或consumer_provider.xml并加入到applicationContext.xml中。

```
<?xml version="1.0"encoding="UTF-8"?>  
<web-app id="WebApp_ID" version="2.4"xmlns="http://java.sun.com/xml/ns/j2ee"xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"xsi:schemaLocation="http://java.sun.com/xml/ns/j2eehttp://java.sun.com/xml/ns/j2ee/web-app_2_4.xsd">  
    <context-param>  
        <param-name>contextConfigLocation</param-name>  
        <param-value>/WEB-INF/applicationContext.xml,/WEB-INF/controllers.xml,/WEB-INF/service.xml</param-value>  
    </context-param> 
    
    <listener>  
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>  
    </listener>  
     
    <servlet>  
        <servlet-name>dispatch</servlet>  
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>  
        <init-param>  
            <param-name>contextConfigLocation</param-name>  
            <param-value>/WEB-INF/applicationContext.xml</param-value>  
        </init-param>  
    </servlet>  
    <servlet-mapping>  
        <servlet-name>dispatch</servlet-name>  
        <servlet-pattern>*.*</servlet-pattern>  
    </servlet-mapping>  
</web-app>  
```
如applicationContext.xml，如下：

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
xmlns:p="http://www.springframework.org/schema/p"
xmlns:context="http://www.springframework.org/schema/context"
xmlns:mvc="http://www.springframework.org/schema/mvc"
xmlns:dubbo="http://dubbo.apache.org/schema/dubbo"
xsi:schemaLocation="
http://www.springframework.org/schema/beans
http://www.springframework.org/schema/beans/spring-beans.xsd
http://www.springframework.org/schema/context
http://www.springframework.org/schema/context/spring-context.xsd
http://www.springframework.org/schema/mvc
http://www.springframework.org/schema/mvc/spring-mvc.xsd">
    <import resource="provider.xml" />
    <!--組件扫描 -->
    <context:component-scan base-package="org.niugang" />
    <!--注解扫描 -->
    <mvc:annotation-driven />
</beans>
```
通过import标签导入provider.xml到applicationContext.xml

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
	xsi:schemaLocation="http://www.springframework.org/schema/beans 
	http://www.springframework.org/schema/beans/spring-beans.xsd 
     http://code.alibabatech.com/schema/dubbo
	 http://code.alibabatech.com/schema/dubbo/dubbo.xsd">
 
	<!-- http://dubbo.apache.org/schema/dubbo/dubbo.xsd 上面配置为这个一直报错，改为 http://code.alibabatech.com/schema/dubbo/dubbo.xsd -->
 
	<!--用于配置当前应用信息，不管该应用是提供者还是消费者 -->
	<dubbo:application name="hello-world-app" />
 
	<!-- 用于配置连接注册中心相关信息 -->
	<dubbo:registry address="redis://localhost:6379"
		timeout="30000">
		<!--配置redis连接参数 -->
		<!--具体参数配置见com.alibaba.dubbo.registry.redis.RedisRegistry.class  -->
		<dubbo:parameter key="max.idle" value="10" />
		<dubbo:parameter key="min.idle" value="5" />
		<dubbo:parameter key="max.active" value="20" />
		<dubbo:parameter key="max.total" value="100" />
	</dubbo:registry>
 
	<!-- 用于配置提供服务的协议信息，协议由提供方指定，消费方被动接受 -->
	<dubbo:protocol name="dubbo" port="20880" accesslog="true" />
	
	<!-- 实现类 -->
	<bean id="defaultService" class="org.niugang.service.DefaultServiceImpl" />
	<!--用于暴露一个服务，定义服务的元信息，一个服务可以用多个协议暴露，一个服务也可以注册到多个注册中心 -->
	<dubbo:service interface="org.niugang.service.DefaultApiService"
		ref="defaultService" />
</beans>
```
2. 如果是spring boot项目，则通过在@Configuration的config类中使用@ImportResource注解加载xml文件，如下：
```
@Configuration
@PropertySource("classpath:dubbo.properties")
@ImportResource({ "classpath:dubbo_consumer.xml","classpath:dubbo_provider.xml" })
public class DubboConfig
```
实现原理是：spring项目启动的时候，加载applicationContext.xml，然后加载dubbo的provider.xml，在加载provider.xml时，解析
http://code.alibabatech.com/schema/dubbo，
即dubbo的标签定义时，会使用dubbo jar包的
dubbo-config-spring的META-INF的spring.handlers的配置：
```
http\://dubbo.apache.org/schema/dubbo=org.apache.dubbo.config.spring.schema.DubboNamespaceHandler
http\://code.alibabatech.com/schema/dubbo=org.apache.dubbo.config.spring.schema.DubboNamespaceHandler
```
即使用dubbo提供的DubboNamespaceHandler来完成解析和生成spring的bean，从而正式进入dubbo框架。

* DubboNamespaceHandler：xml文件的解析和注册bean

```
public class DubboNamespaceHandler extends NamespaceHandlerSupport {

    static {
        Version.checkDuplicate(DubboNamespaceHandler.class);
    }

    @Override
    public void init() {
        registerBeanDefinitionParser("application", new DubboBeanDefinitionParser(ApplicationConfig.class, true));
        registerBeanDefinitionParser("module", new DubboBeanDefinitionParser(ModuleConfig.class, true));
        registerBeanDefinitionParser("registry", new DubboBeanDefinitionParser(RegistryConfig.class, true));
        registerBeanDefinitionParser("monitor", new DubboBeanDefinitionParser(MonitorConfig.class, true));
        registerBeanDefinitionParser("provider", new DubboBeanDefinitionParser(ProviderConfig.class, true));
        registerBeanDefinitionParser("consumer", new DubboBeanDefinitionParser(ConsumerConfig.class, true));
        registerBeanDefinitionParser("protocol", new DubboBeanDefinitionParser(ProtocolConfig.class, true));
        registerBeanDefinitionParser("service", new DubboBeanDefinitionParser(ServiceBean.class, true));
        registerBeanDefinitionParser("reference", new DubboBeanDefinitionParser(ReferenceBean.class, false));
        registerBeanDefinitionParser("annotation", new AnnotationBeanDefinitionParser());
    }

}
```
使用registerBeanDefinitionParser方法解析xml文件配置，生成spring容器的beanDefinition并注册到BeanDefinitionRegistry中。beanDefinition是bean的一个描述类，之后spring容器从BeanDefinitionRegistry中取出beanDefinition，并使用InstantiationStrategy来真正实例化，创建bean实例。


### ServiceBean和Referencebean的接口设计

ServiceBean和ReferenceBean继承了InitializingBean，DisposableBean，FactoryBean等，故当spring使用InstantiationStrategy来进行实例化，创建bean实例时，会根据这里的接口定义，实现bean的自定义实例化过程。
1. ReferenceBean
```
public class ReferenceBean<T> extends ReferenceConfig<T> implements FactoryBean, ApplicationContextAware, InitializingBean, DisposableBean
```
2. ServiceBean
```
public class ServiceBean<T> extends ServiceConfig<T> implements InitializingBean, DisposableBean, ApplicationContextAware, ApplicationListener<ContextRefreshedEvent>, BeanNameAware 
```
以下是这些接口的作用：
#### InitializingBean：在bean的所有属性都设好值后，自定义bean的实例化过程
```
package org.springframework.beans.factory;

/**
 * Interface to be implemented by beans that need to react once all their
 * properties have been set by a BeanFactory: for example, to perform custom
 * initialization, or merely to check that all mandatory properties have been set.
 *
 * <p>An alternative to implementing InitializingBean is specifying a custom
 * init-method, for example in an XML bean definition.
 * For a list of all bean lifecycle methods, see the BeanFactory javadocs.
 *
 * @author Rod Johnson
 * @see BeanNameAware
 * @see BeanFactoryAware
 * @see BeanFactory
 * @see org.springframework.beans.factory.support.RootBeanDefinition#getInitMethodName
 * @see org.springframework.context.ApplicationContextAware
 */
public interface InitializingBean {

	/**
	 * Invoked by a BeanFactory after it has set all bean properties supplied
	 * (and satisfied BeanFactoryAware and ApplicationContextAware).
	 * <p>This method allows the bean instance to perform initialization only
	 * possible when all bean properties have been set and to throw an
	 * exception in the event of misconfiguration.
	 * @throws Exception in the event of misconfiguration (such
	 * as failure to set an essential property) or if initialization fails.
	 */
	void afterPropertiesSet() throws Exception;

}

```
* 提供了一个afterPropertiesSet方法，在bean的所有属性通过Spring的beanFactory设值之后，可以在afterPropertiesSet方法中，自定义一些初始化操作，即

* 在ServiceBean中是初始化application，module，registries，monitor，path。最后调用export方法进行注册。

* 在ReferenceBean中，除了进行application，module，monitor等设值外，会调用FactoryBean的getObject生成服务代理proxy。具体看FactoryBean。

#### FactoryBean：spring容器BeanFactory获取bean的实例对象
```
package org.springframework.beans.factory;

/**
 * Interface to be implemented by objects used within a {@link BeanFactory} which
 * are themselves factories for individual objects. If a bean implements this
 * interface, it is used as a factory for an object to expose, not directly as a
 * bean instance that will be exposed itself.
 *
 * <p><b>NB: A bean that implements this interface cannot be used as a normal bean.</b>
 * A FactoryBean is defined in a bean style, but the object exposed for bean
 * references ({@link #getObject()}) is always the object that it creates.
 *
 * <p>FactoryBeans can support singletons and prototypes, and can either create
 * objects lazily on demand or eagerly on startup. The {@link SmartFactoryBean}
 * interface allows for exposing more fine-grained behavioral metadata.
 *
 * <p>This interface is heavily used within the framework itself, for example for
 * the AOP {@link org.springframework.aop.framework.ProxyFactoryBean} or the
 * {@link org.springframework.jndi.JndiObjectFactoryBean}. It can be used for
 * custom components as well; however, this is only common for infrastructure code.
 *
 * <p><b>{@code FactoryBean} is a programmatic contract. Implementations are not
 * supposed to rely on annotation-driven injection or other reflective facilities.</b>
 * {@link #getObjectType()} {@link #getObject()} invocations may arrive early in
 * the bootstrap process, even ahead of any post-processor setup. If you need access
 * other beans, implement {@link BeanFactoryAware} and obtain them programmatically.
 *
 * <p>Finally, FactoryBean objects participate in the containing BeanFactory's
 * synchronization of bean creation. There is usually no need for internal
 * synchronization other than for purposes of lazy initialization within the
 * FactoryBean itself (or the like).
 *
 * @author Rod Johnson
 * @author Juergen Hoeller
 * @since 08.03.2003
 * @see org.springframework.beans.factory.BeanFactory
 * @see org.springframework.aop.framework.ProxyFactoryBean
 * @see org.springframework.jndi.JndiObjectFactoryBean
 */
public interface FactoryBean<T> {

	/**
	 * Return an instance (possibly shared or independent) of the object
	 * managed by this factory.
	 * <p>As with a {@link BeanFactory}, this allows support for both the
	 * Singleton and Prototype design pattern.
	 * <p>If this FactoryBean is not fully initialized yet at the time of
	 * the call (for example because it is involved in a circular reference),
	 * throw a corresponding {@link FactoryBeanNotInitializedException}.
	 * <p>As of Spring 2.0, FactoryBeans are allowed to return {@code null}
	 * objects. The factory will consider this as normal value to be used; it
	 * will not throw a FactoryBeanNotInitializedException in this case anymore.
	 * FactoryBean implementations are encouraged to throw
	 * FactoryBeanNotInitializedException themselves now, as appropriate.
	 * @return an instance of the bean (can be {@code null})
	 * @throws Exception in case of creation errors
	 * @see FactoryBeanNotInitializedException
	 */
	T getObject() throws Exception;

	/**
	 * Return the type of object that this FactoryBean creates,
	 * or {@code null} if not known in advance.
	 * <p>This allows one to check for specific types of beans without
	 * instantiating objects, for example on autowiring.
	 * <p>In the case of implementations that are creating a singleton object,
	 * this method should try to avoid singleton creation as far as possible;
	 * it should rather estimate the type in advance.
	 * For prototypes, returning a meaningful type here is advisable too.
	 * <p>This method can be called <i>before</i> this FactoryBean has
	 * been fully initialized. It must not rely on state created during
	 * initialization; of course, it can still use such state if available.
	 * <p><b>NOTE:</b> Autowiring will simply ignore FactoryBeans that return
	 * {@code null} here. Therefore it is highly recommended to implement
	 * this method properly, using the current state of the FactoryBean.
	 * @return the type of object that this FactoryBean creates,
	 * or {@code null} if not known at the time of the call
	 * @see ListableBeanFactory#getBeansOfType
	 */
	Class<?> getObjectType();

	/**
	 * Is the object managed by this factory a singleton? That is,
	 * will {@link #getObject()} always return the same object
	 * (a reference that can be cached)?
	 * <p><b>NOTE:</b> If a FactoryBean indicates to hold a singleton object,
	 * the object returned from {@code getObject()} might get cached
	 * by the owning BeanFactory. Hence, do not return {@code true}
	 * unless the FactoryBean always exposes the same reference.
	 * <p>The singleton status of the FactoryBean itself will generally
	 * be provided by the owning BeanFactory; usually, it has to be
	 * defined as singleton there.
	 * <p><b>NOTE:</b> This method returning {@code false} does not
	 * necessarily indicate that returned objects are independent instances.
	 * An implementation of the extended {@link SmartFactoryBean} interface
	 * may explicitly indicate independent instances through its
	 * {@link SmartFactoryBean#isPrototype()} method. Plain {@link FactoryBean}
	 * implementations which do not implement this extended interface are
	 * simply assumed to always return independent instances if the
	 * {@code isSingleton()} implementation returns {@code false}.
	 * @return whether the exposed object is a singleton
	 * @see #getObject()
	 * @see SmartFactoryBean#isPrototype()
	 */
	boolean isSingleton();

}
```
* 这个接口定义如何来获取指定类型的对象，可以是单例或原型，通过实现这个接口，可以控制该类型bean实例的获取。FactoryBean实例主要是Spring框架自身的BeanFactory调用。所以如果我们想控制某个类型bean实例的获取，即对spring容器BeanFactory的getBean方法如何获取某个类型的bean实例做控制，则可以实现接口，实现它的getObject()，isSingleton方法。

* ReferenceBean
1. 由于在客户端是对服务端的Service的引用reference，不是真实的service bean，故获取这个reference时，其实是获取了一个到服务端Service的proxy。故通过spring容器获取这个Service的bean时，需要返回一个proxy。ReferenceBean实现FactoryBean接口，isSingleton方法返回为true，getObject()创建并返回客户端的service proxy，即refer。
2. 为了之后客户端调用的性能考虑，在spring启动时，dubbo提前初始化好refer代理，之后spring容器获取该bean，如在BeanFactory调用getBean方法时，getObject可以直接返回refer代理了，而不是等到调用时再初始化生成refer代理。提前初始化refer的具体实现为在InitializingBean的afterPropertiesSet的方法中调用getObject，getObject最终会调用com.alibaba.dubbo.config.ReferenceConfig#get方法完成远程服务调用代理类的创建。

* ServiceBean：没有实现FactoryBean接口，原因是服务端的Service bean就是提高功能的bean了，就跟用@Service描述的bean一样，所以spring就按照正常流程，使用spring内置的FactoryBean接口创建即可。但是由于是rpc服务，跟其他service不一样的是，需要注册到注册中心，并且开启监听端口，监听客户端的连接请求，故需要实现InitializingBean接口，在Service bean的属性设置好之后，在InitializingBean的afterPropertiesSet方法中，调用com.alibaba.dubbo.config.ServiceConfig#export方法进行服务暴露。
* 更多关于FactoryBean可以看：https://www.cnblogs.com/davidwang456/p/3688250.html

#### DisposableBean：自定义spring容器管理的单实例bean的销毁操作
```
/**
 * Interface to be implemented by beans that want to release resources
 * on destruction. A BeanFactory is supposed to invoke the destroy
 * method if it disposes a cached singleton. An application context
 * is supposed to dispose all of its singletons on close.
 *
 * <p>An alternative to implementing DisposableBean is specifying a custom
 * destroy-method, for example in an XML bean definition.
 * For a list of all bean lifecycle methods, see the BeanFactory javadocs.
 *
 * @author Juergen Hoeller
 * @since 12.08.2003
 * @see org.springframework.beans.factory.support.RootBeanDefinition#getDestroyMethodName
 * @see org.springframework.context.ConfigurableApplicationContext#close
 */
public interface DisposableBean {

	/**
	 * Invoked by a BeanFactory on destruction of a singleton.
	 * @throws Exception in case of shutdown errors.
	 * Exceptions will get logged but not rethrown to allow
	 * other beans to release their resources too.
	 */
	void destroy() throws Exception;

}
```
* 这个接口主要是针对spring单实例缓存中singleton bean的释放，即在关闭Spring容器，BeanFactory释放和销毁容器中的单例bean时，会调用这个接口的destroy方法。故类可以通过实现这个接口，在destroy方法中自定义如何销毁该类单例实例。

* ServiceBean

```
@Override
public void destroy() throws Exception {
    // This will only be called for singleton scope bean, and expected to be called by spring shutdown hook when BeanFactory/ApplicationContext destroys.
    // We will guarantee dubbo related resources being released with dubbo shutdown hook.
    //unexport();
}
```
ServiceBean的bean的释放最初设计是通过destroy方法来unexport的，后来被spring孵化后，是通过添加spring的关闭钩子shutdown hook来做的，不在destroy方法中执行（如上代码unexport()注释掉了），如下为spring的shutdown hook：

```
/**
 * The shutdown hook thread to do the clean up stuff.
 * This is a singleton in order to ensure there is only one shutdown hook registered.
 * Because {@link ApplicationShutdownHooks} use {@link java.util.IdentityHashMap}
 * to store the shutdown hooks.
 */
public class DubboShutdownHook extends Thread {

    private static final Logger logger = LoggerFactory.getLogger(DubboShutdownHook.class);

    private static final DubboShutdownHook dubboShutdownHook = new DubboShutdownHook("DubboShutdownHook");

    public static DubboShutdownHook getDubboShutdownHook() {
        return dubboShutdownHook;
    }

    /**
     * Has it already been destroyed or not?
     */
    private final AtomicBoolean destroyed;

    private DubboShutdownHook(String name) {
        super(name);
        this.destroyed = new AtomicBoolean(false);
    }

    @Override
    public void run() {
        if (logger.isInfoEnabled()) {
            logger.info("Run shutdown hook now.");
        }
        destroyAll();
    }

    /**
     * Destroy all the resources, including registries and protocols.
     */
    public void destroyAll() {
        if (!destroyed.compareAndSet(false, true)) {
            return;
        }
        // destroy all the registries
        AbstractRegistryFactory.destroyAll();
        // destroy all the protocols
        destroyProtocols();
    }

    /**
     * Destroy all the protocols.
     */
    private void destroyProtocols() {
        ExtensionLoader<Protocol> loader = ExtensionLoader.getExtensionLoader(Protocol.class);
        for (String protocolName : loader.getLoadedExtensions()) {
            try {
                Protocol protocol = loader.getLoadedExtension(protocolName);
                if (protocol != null) {
                    protocol.destroy();
                }
            } catch (Throwable t) {
                logger.warn(t.getMessage(), t);
            }
        }
    }

}
```
DubboShutdownHook是在DubboBootstrap中注册的：

```
/**
 * A bootstrap class to easily start and stop Dubbo via programmatic API.
 * The bootstrap class will be responsible to cleanup the resources during stop.
 */
public class DubboBootstrap
核心方法：
public void start() {
    if (registerShutdownHookOnStart) {
        registerShutdownHook();
    } else {
        // DubboShutdown hook has been registered in AbstractConfig,
        // we need to remove it explicitly
        removeShutdownHook();
    }
    for (ServiceConfig serviceConfig: serviceConfigList) {
        serviceConfig.export();
    }
}

public void stop() {
    for (ServiceConfig serviceConfig: serviceConfigList) {
        serviceConfig.unexport();
    }
    shutdownHook.destroyAll();
    if (registerShutdownHookOnStart) {
        removeShutdownHook();
    }
}

/**
 * Register the shutdown hook
 */
public void registerShutdownHook() {
    Runtime.getRuntime().addShutdownHook(shutdownHook);
}

/**
 * Remove this shutdown hook
 */
public void removeShutdownHook() {
    try {
        Runtime.getRuntime().removeShutdownHook(shutdownHook);
    }
    catch (IllegalStateException ex) {
        // ignore - VM is already shutting down
    }
}
```
而DubboBootstrap自身实例化和启动，是通过在web-fragment.xml注册context-param，在spring启动时，注册ApplicationListener，在spring启动完成后，调用dubboBootstrap.start()，注册钩子，在关闭时，调用dubboBootstrap.stop()，移除钩子：
dubbo-config-spring下的web-fragment.xml

```
<web-fragment version="3.0" xmlns="http://java.sun.com/xml/ns/javaee"
              xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
              xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-fragment_3_0.xsd">

    <name>dubbo-fragment</name>

    <ordering>
        <before>
            <others/>
        </before>
    </ordering>

    <context-param>
        <param-name>contextInitializerClasses</param-name>
        <param-value>org.apache.dubbo.config.spring.initializer.DubboApplicationContextInitializer</param-value>
    </context-param>

    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>

</web-fragment>
```
DubboApplicationContextInitializer和DubboApplicationListener：

```
/**
 * Automatically register {@link DubboApplicationListener} to Spring context
 * A {@link org.springframework.web.context.ContextLoaderListener} class is defined in
 * src/main/resources/META-INF/web-fragment.xml
 * In the web-fragment.xml, {@link DubboApplicationContextInitializer} is defined in context params.
 * This file will be discovered if running under a servlet 3.0+ container.
 * Even if user specifies {@link org.springframework.web.context.ContextLoaderListener} in web.xml,
 * it will be merged to web.xml.
 * If user specifies <metadata-complete="true" /> in web.xml, this will no take effect,
 * unless user configures {@link DubboApplicationContextInitializer} explicitly in web.xml.
 */
public class DubboApplicationContextInitializer implements ApplicationContextInitializer {

    @Override
    public void initialize(ConfigurableApplicationContext applicationContext) {
        applicationContext.addApplicationListener(new DubboApplicationListener());
    }
}

/**
 * An application listener that listens the ContextClosedEvent.
 * Upon the event, this listener will do the necessary clean up to avoid memory leak.
 */
public class DubboApplicationListener implements ApplicationListener<ApplicationEvent> {

    private DubboBootstrap dubboBootstrap;

    public DubboApplicationListener() {
        dubboBootstrap = new DubboBootstrap(false);
    }

    public DubboApplicationListener(DubboBootstrap dubboBootstrap) {
        this.dubboBootstrap = dubboBootstrap;
    }

    @Override
    public void onApplicationEvent(ApplicationEvent applicationEvent) {
        if (applicationEvent instanceof ContextRefreshedEvent) {
            dubboBootstrap.start();
        } else if (applicationEvent instanceof ContextClosedEvent) {
            dubboBootstrap.stop();
        }
    }
}
```

* ReferenceBean
自身的destroy不做任何事情，是在ReferenceConfig中实现destroy：

```
public synchronized void destroy() {
    if (ref == null) {
        return;
    }
    if (destroyed) {
        return;
    }
    destroyed = true;
    try {
        invoker.destroy();
    } catch (Throwable t) {
        logger.warn("Unexpected err when destroy invoker of ReferenceConfig(" + url + ").", t);
    }
    invoker = null;
    ref = null;
}
```
#### ApplicationListener：spring容器监听器
* 观察者设计模式的一种实现，bean可以实现这个接口，并通过指定泛型的实际类型，即ApplicationEvent的实现类，来监听spring容器产生的特定事件，在onApplicationEvent方法中，自定义实现对该特定事件的响应和处理逻辑。

```
/**
 * Interface to be implemented by application event listeners.
 * Based on the standard {@code java.util.EventListener} interface
 * for the Observer design pattern.
 *
 * <p>As of Spring 3.0, an ApplicationListener can generically declare the event type
 * that it is interested in. When registered with a Spring ApplicationContext, events
 * will be filtered accordingly, with the listener getting invoked for matching event
 * objects only.
 *
 * @author Rod Johnson
 * @author Juergen Hoeller
 * @param <E> the specific ApplicationEvent subclass to listen to
 * @see org.springframework.context.event.ApplicationEventMulticaster
 */
public interface ApplicationListener<E extends ApplicationEvent> extends EventListener {

	/**
	 * Handle an application event.
	 * @param event the event to respond to
	 */
	void onApplicationEvent(E event);

}

/**
 * Event raised when an {@code ApplicationContext} gets initialized or refreshed.
 *
 * @author Juergen Hoeller
 * @since 04.03.2003
 * @see ContextClosedEvent
 */
@SuppressWarnings("serial")
public class ContextRefreshedEvent extends ApplicationContextEvent {

	/**
	 * Create a new ContextRefreshedEvent.
	 * @param source the {@code ApplicationContext} that has been initialized
	 * or refreshed (must not be {@code null})
	 */
	public ContextRefreshedEvent(ApplicationContext source) {
		super(source);
	}

}
```
* ServiceBean：实现这个接口，并注册ContextRefreshedEvent事件。ContextRefreshedEvent是在spring容器初始化或者刷新时，产生的事件。ServiceBean对该事件的处理是，执行export导出服务，export方法保证了只导出和注册一次，是可以重复调用的。

```
@Override
public void onApplicationEvent(ContextRefreshedEvent event) {
    if (isDelay() && !isExported() && !isUnexported()) {
        if (logger.isInfoEnabled()) {
            logger.info("The service ready on spring started. service: " + getInterface());
        }
        export();
    }
}

ServiceConfig的export实现：
public synchronized void export() {
    if (provider != null) {
        if (export == null) {
            export = provider.getExport();
        }
        if (delay == null) {
            delay = provider.getDelay();
        }
    }
    if (export != null && !export) {
        return;
    }

    if (delay != null && delay > 0) {
        delayExportExecutor.schedule(new Runnable() {
            @Override
            public void run() {
                doExport();
            }
        }, delay, TimeUnit.MILLISECONDS);
    } else {
        doExport();
    }
}
```


#### ApplicationContextAware：bean获取ApplicationContext对象，从而可以通过ApplicationContext与其他bean交互或者访问其他资源，如resource文件
bean实现这个接口，并在setApplicationContext方法中自定义该bean需要通过ApplicationContext做什么东西。setApplicationContext方法会在bean的属性赋值完成，但在初始化callback调用之前，如InitializingBean的afterPropertiesSet，调用之前，调用。

```
/**
 * Interface to be implemented by any object that wishes to be notified
 * of the {@link ApplicationContext} that it runs in.
 *
 * <p>Implementing this interface makes sense for example when an object
 * requires access to a set of collaborating beans. Note that configuration
 * via bean references is preferable to implementing this interface just
 * for bean lookup purposes.
 *
 * <p>This interface can also be implemented if an object needs access to file
 * resources, i.e. wants to call {@code getResource}, wants to publish
 * an application event, or requires access to the MessageSource. However,
 * it is preferable to implement the more specific {@link ResourceLoaderAware},
 * {@link ApplicationEventPublisherAware} or {@link MessageSourceAware} interface
 * in such a specific scenario.
 *
 * <p>Note that file resource dependencies can also be exposed as bean properties
 * of type {@link org.springframework.core.io.Resource}, populated via Strings
 * with automatic type conversion by the bean factory. This removes the need
 * for implementing any callback interface just for the purpose of accessing
 * a specific file resource.
 *
 * <p>{@link org.springframework.context.support.ApplicationObjectSupport} is a
 * convenience base class for application objects, implementing this interface.
 *
 * <p>For a list of all bean lifecycle methods, see the
 * {@link org.springframework.beans.factory.BeanFactory BeanFactory javadocs}.
 *
 * @author Rod Johnson
 * @author Juergen Hoeller
 * @author Chris Beams
 * @see ResourceLoaderAware
 * @see ApplicationEventPublisherAware
 * @see MessageSourceAware
 * @see org.springframework.context.support.ApplicationObjectSupport
 * @see org.springframework.beans.factory.BeanFactoryAware
 */
public interface ApplicationContextAware extends Aware {

	/**
	 * Set the ApplicationContext that this object runs in.
	 * Normally this call will be used to initialize the object.
	 * <p>Invoked after population of normal bean properties but before an init callback such
	 * as {@link org.springframework.beans.factory.InitializingBean#afterPropertiesSet()}
	 * or a custom init-method. Invoked after {@link ResourceLoaderAware#setResourceLoader},
	 * {@link ApplicationEventPublisherAware#setApplicationEventPublisher} and
	 * {@link MessageSourceAware}, if applicable.
	 * @param applicationContext the ApplicationContext object to be used by this object
	 * @throws ApplicationContextException in case of context initialization errors
	 * @throws BeansException if thrown by application context methods
	 * @see org.springframework.beans.factory.BeanInitializationException
	 */
	void setApplicationContext(ApplicationContext applicationContext) throws BeansException;

}

```
* ServiceBean：两个作用
1. 将applicationContext放到dubbo自身维护的一个SpringExtensionFactory，即在dubbo中保存spring容器applicationContext的引用，从而可以在dubbo中通过applicationContext引用查找spring容器中的bean。
2. 在applicationContext中注册容器监听器ApplicationListener，监听ContextRefreshedEvent事件，即自身实例，因为ServiceBean实现了ApplicationListener接口，在ApplicationListener接口的onApplicationEvent中调用export（主要是spring容器重启时调用）。
```
@Override
public void setApplicationContext(ApplicationContext applicationContext) {
    this.applicationContext = applicationContext;
    SpringExtensionFactory.addApplicationContext(applicationContext);
    if (applicationContext != null) {
        SPRING_CONTEXT = applicationContext;
        try {
            Method method = applicationContext.getClass().getMethod("addApplicationListener", ApplicationListener.class); // backward compatibility to spring 2.0.1
            method.invoke(applicationContext, this);
            supportedApplicationListener = true;
        } catch (Throwable t) {
            if (applicationContext instanceof AbstractApplicationContext) {
                try {
                    Method method = AbstractApplicationContext.class.getDeclaredMethod("addListener", ApplicationListener.class); // backward compatibility to spring 2.0.1
                    if (!method.isAccessible()) {
                        method.setAccessible(true);
                    }
                    method.invoke(applicationContext, this);
                    supportedApplicationListener = true;
                } catch (Throwable t2) {
                }
            }
        }
    }
}
```

* ReferenceBean
与ServiceBean的第一个作用一样。
```
@Override
public void setApplicationContext(ApplicationContext applicationContext) {
    this.applicationContext = applicationContext;
    SpringExtensionFactory.addApplicationContext(applicationContext);
}
```