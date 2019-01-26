---
layout:     post
title:      "Spring Bean对象初始化和销毁相关回调的用法和源码实现"
subtitle:   "Spring Bean init and destroy callback"
date:       2019-01-10 20:49:00
author:     "XYZ"
header-img: "img/article-bg1.jpg"
tags:
    - Spring源码分析
---
### 概述
* 构造后置处理：在spring容器启动，加载并创建bean对象实例的时候调用，通常为在构造bean对象实例，将相关属性赋值好了调用。
* 销毁前置处理：在spring容器关闭，在销毁其所创建并管理的bean对象实例之前，执行销毁前置处理，通常可以用来释放外部资源等。
### 使用方法
##### 1. JDK注解方式
1. 构造后置处理：使用@PostConstruct注解对应的方法
2. 销毁前置处理：使用@PreDestroy注解对应的方法。

* 可以应用在一个或多个方法上面，但是推荐使用一个方法。方法的可见性没有严格要求，即public，default（包可见），protected，private都可以。
* 基于JDK的注解方式是一个在spring容器外配置的方式，即不是使用spring自身注解。spring对这种方式的支持，主要是通过spring的CommonAnnotationBeanPostProcessor，这个BeanPostProcessor在spring容器创建bean之后和销毁bean之前调用对应的方法。
##### 2. Spring的接口方式
1. 构造后置处理：实现InitializingBean接口的afterPropertiesSet方法；
2. 销毁前置处理：实现DisposableBean接口的destroy方法。
3. 除了使用JDK提供的@PostConstruct和@PreDestroy，在spring中也可以自定义注解来实现，具体为通过InitDestroyAnnotationBeanPostProcessor来实现。CommonAnnotationBeanPostProcessor就是InitDestroyAnnotationBeanPostProcessor的子类，指定注解@PostConstruct和@PreDestroy。
##### 3. XML配置或者@Bean注解
1. 构造后置处理：在bean标签的init-method中指定处理方法；或者使用@Bean注解的initMethod指定，其中@Bean通常为@Configuration（或者是@Component或@Component的子注解，如@Service，@Configuration是@Component的一个子注解）注解里面的@Bean方法；
2. 销毁前置处理：在bean标签的destroy-method中指定处理方法；或者使用@Bean注解的destroyMethod指定。
### 源码实现
#### 三种实现方式在BeanFactory中的调用过程
* 在以上用法中：（1）JDK注解@PostContruct，@PreDestroy（2）InitializingBean和DisposableBean接口（3）XML配置或者@Bean注解，这三种方式，都是在BeanFactory在创建某个bean对象实例时，检查并调用初始化回调方法；或者spring容器关闭，BeanFactory销毁其所管理的bean（具体为singleton bean），回调销毁回调方法。
##### 1. 构造后置处理
* BeanFactory在创建一个bean对象实例是，从上到下实现源码如下，其中从最顶层的getBean方法开始，调用顺序为：getBean -> doGetBean -> createBean -> doCreateBean：
   *  doCreateBean：核心逻辑包含：创建bean对象实例 -> bean的属性赋值populateBean -> 初始化bean：initializeBean -> 注册销毁拦截的disposableBean：registerDisposableBeanIfNecessary，将该bean保存在一个以beanName为key，以包装了bean引用的DisposableBeanAdapter，为value的map中，在spring容器关闭时，遍历这个map来获取bean。
		```java
			protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
					throws BeanCreationException {
					
				...
		
				// Initialize the bean instance.
				Object exposedObject = bean;
				try {
					populateBean(beanName, mbd, instanceWrapper);
					exposedObject = initializeBean(beanName, exposedObject, mbd);
				}
				catch (Throwable ex) {
					if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
						throw (BeanCreationException) ex;
					}
					else {
						throw new BeanCreationException(
								mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
					}
				}
				
				...
		
				// Register bean as disposable.
				try {
					registerDisposableBeanIfNecessary(beanName, bean, mbd);
				}
				catch (BeanDefinitionValidationException ex) {
					throw new BeanCreationException(
							mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
				}
		
				return exposedObject;
			}
		```
   * initializeBean：这里关注initializeBean方法，在这个方法里完成bean的初始化，分别完成：调用BeanPostProcessor的postProcessBeforeInitialization，调用invokeInitMethods，调用BeanPostProcessor的postProcessAfterInitialization方法。
		```java
		/**
		 * Initialize the given bean instance, applying factory callbacks
		 * as well as init methods and bean post processors.
		 * <p>Called from {@link #createBean} for traditionally defined beans,
		 * and from {@link #initializeBean} for existing bean instances.
		 * @param beanName the bean name in the factory (for debugging purposes)
		 * @param bean the new bean instance we may need to initialize
		 * @param mbd the bean definition that the bean was created with
		 * (can also be {@code null}, if given an existing bean instance)
		 * @return the initialized bean instance (potentially wrapped)
		 * @see BeanNameAware
		 * @see BeanClassLoaderAware
		 * @see BeanFactoryAware
		 * @see #applyBeanPostProcessorsBeforeInitialization
		 * @see #invokeInitMethods
		 * @see #applyBeanPostProcessorsAfterInitialization
		 */
		protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
			if (System.getSecurityManager() != null) {
				AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
					invokeAwareMethods(beanName, bean);
					return null;
				}, getAccessControlContext());
			}
			else {
				invokeAwareMethods(beanName, bean);
			}
		
			Object wrappedBean = bean;
			if (mbd == null || !mbd.isSynthetic()) {

				// 调用BeanPostProcessor的postProcessBeforeInitialization
				
				wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
			}
		
			try {

				// 调用invokeInitMethods
				
				invokeInitMethods(beanName, wrappedBean, mbd);
			}
			catch (Throwable ex) {
				throw new BeanCreationException(
						(mbd != null ? mbd.getResourceDescription() : null),
						beanName, "Invocation of init method failed", ex);
			}
			if (mbd == null || !mbd.isSynthetic()) {
				
				// 调用BeanPostProcessor的postProcessAfterInitialization方法
				
				wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
			}
		
			return wrappedBean;
		}
		```

   * invokeInitMethods：检测当前Bean是否实现了InitializingBean接口和定义了init-method，并调用：
		```java
		/**
		 * Give a bean a chance to react now all its properties are set,
		 * and a chance to know about its owning bean factory (this object).
		 * This means checking whether the bean implements InitializingBean or defines
		 * a custom init method, and invoking the necessary callback(s) if it does.
		 * @param beanName the bean name in the factory (for debugging purposes)
		 * @param bean the new bean instance we may need to initialize
		 * @param mbd the merged bean definition that the bean was created with
		 * (can also be {@code null}, if given an existing bean instance)
		 * @throws Throwable if thrown by init methods or by the invocation process
		 * @see #invokeCustomInitMethod
		 */
		protected void invokeInitMethods(String beanName, final Object bean, @Nullable RootBeanDefinition mbd)
				throws Throwable {
			
			// InitializingBean的检测并执行
			
			boolean isInitializingBean = (bean instanceof InitializingBean);
			if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
				if (logger.isTraceEnabled()) {
					logger.trace("Invoking afterPropertiesSet() on bean with name '" + beanName + "'");
				}
				if (System.getSecurityManager() != null) {
					try {
						AccessController.doPrivileged((PrivilegedExceptionAction<Object>) () -> {
							((InitializingBean) bean).afterPropertiesSet();
							return null;
						}, getAccessControlContext());
					}
					catch (PrivilegedActionException pae) {
						throw pae.getException();
					}
				}
				else {
					((InitializingBean) bean).afterPropertiesSet();
				}
			}
		
			// init-method的检测并执行
			
			if (mbd != null && bean.getClass() != NullBean.class) {
				String initMethodName = mbd.getInitMethodName();
				if (StringUtils.hasLength(initMethodName) &&
						!(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&
						!mbd.isExternallyManagedInitMethod(initMethodName)) {
					invokeCustomInitMethod(beanName, bean, mbd);
				}
			}
		}
		```

##### 2. 销毁前置处理
* 由上面的源码分析可知，在doCreateBean方法中，会通过registerDisposableBeanIfNecessary方法来注册disposable bean，在beanRegistry中通过一个map来保存：其中key为beanName，value为DisposableBeanAdapter。

	```java
	/** Disposable bean instances: bean name to disposable instance. */
	private final Map<String, Object> disposableBeans = new LinkedHashMap<>();
	
	// registerDisposableBeanIfNecessary的实现
	
	protected void registerDisposableBeanIfNecessary(String beanName, Object bean, RootBeanDefinition mbd) {
		AccessControlContext acc = (System.getSecurityManager() != null ? getAccessControlContext() : null);
		if (!mbd.isPrototype() && requiresDestruction(bean, mbd)) {
			if (mbd.isSingleton()) {
				// Register a DisposableBean implementation that performs all destruction
				// work for the given bean: DestructionAwareBeanPostProcessors,
				// DisposableBean interface, custom destroy method.
				registerDisposableBean(beanName,

						// 使用DisposableBeanAdapter封装bean
				
						new DisposableBeanAdapter(bean, beanName, mbd, getBeanPostProcessors(), acc));
			}
			else {
				// A bean with a custom scope...
				Scope scope = this.scopes.get(mbd.getScope());
				if (scope == null) {
					throw new IllegalStateException("No Scope registered for scope name '" + mbd.getScope() + "'");
				}
				scope.registerDestructionCallback(beanName,
						new DisposableBeanAdapter(bean, beanName, mbd, getBeanPostProcessors(), acc));
			}
		}
	}
	```
	
* 在spring容器关闭时，从AbstractApplicationContext的doClose开始，调用BeanFactory的destroySingletons，最终销毁每个singleton的时候，从以上disposableBeans获取该bean对应的DisposableBeanAdapter，然后调用DisposableBeanAdapter的destroy。
	
	```java
	/**
	 * Destroy the given bean. Delegates to {@code destroyBean}
	 * if a corresponding disposable bean instance is found.
	 * @param beanName the name of the bean
	 * @see #destroyBean
	 */
	public void destroySingleton(String beanName) {
		// Remove a registered singleton of the given name, if any.
		removeSingleton(beanName);
	
		// Destroy the corresponding DisposableBean instance.
		DisposableBean disposableBean;
		synchronized (this.disposableBeans) {
			disposableBean = (DisposableBean) this.disposableBeans.remove(beanName);
		}
		destroyBean(beanName, disposableBean);
	}
	```
* DisposableBeanAdapter的destroy方法实现如下：先调用BeanPostProcessor，再调用DisposableBean，最后调用destroy-method。
	
	```java
	@Override
	public void destroy() {
		
		// 调用BeanPostProcessor的postProcessBeforeDestruction
		// 如基于注解实现的销毁前置处理：InitDestroyAnnotationBeanPostProcessor
		
		if (!CollectionUtils.isEmpty(this.beanPostProcessors)) {
			for (DestructionAwareBeanPostProcessor processor : this.beanPostProcessors) {
				processor.postProcessBeforeDestruction(this.bean, this.beanName);
			}
		}
	
		// 调用bean的DisposableBean接口实现的destroy
		
		if (this.invokeDisposableBean) {
			if (logger.isTraceEnabled()) {
				logger.trace("Invoking destroy() on bean with name '" + this.beanName + "'");
			}
			try {
				if (System.getSecurityManager() != null) {
					AccessController.doPrivileged((PrivilegedExceptionAction<Object>) () -> {
						((DisposableBean) this.bean).destroy();
						return null;
					}, this.acc);
				}
				else {
					((DisposableBean) this.bean).destroy();
				}
			}
			catch (Throwable ex) {
				String msg = "Invocation of destroy method failed on bean with name '" + this.beanName + "'";
				if (logger.isDebugEnabled()) {
					logger.info(msg, ex);
				}
				else {
					logger.info(msg + ": " + ex);
				}
			}
		}
	
		// 调用destroy-method，即在bean标签中声明的destroy-method
		
		if (this.destroyMethod != null) {
			invokeCustomDestroyMethod(this.destroyMethod);
		}
		else if (this.destroyMethodName != null) {
			Method methodToCall = determineDestroyMethod(this.destroyMethodName);
			if (methodToCall != null) {
				invokeCustomDestroyMethod(methodToCall);
			}
		}
	}
	```
#### 基于BeanPostProcessor接口的实现
* 第三种方式，使用JDK的@PostConstruct和@PreDestroy注解的实现，是基于BeanPostProcessor来实现的。
* 由上面的代码分析可知，基于BeanPostProcessor实现的构造后置处理也是在initializeBean方法中调用，同时是在InitializingBean和init-method之前调用。
* 基于BeanPostProcessor的销毁前置处理，也是在DisposableBeanAdapter的destroy方法执行的，同时是在DisposableBean的destroy和destroy-method之前调用的。
##### 1. 自定义注解实现
* InitDestroyAnnotationBeanPostProcessor：这个类在spring-beans包定义，可以自定义指定需要处理的注解，构造后置处理的注解，是通过setInitAnnotationType方法来出传入的；销毁前置处理的注解，是通过setDestroyAnnotationType来传入的。
##### 2. 使用JDK的@PostConstruct和@PreDestroy注解实现
* CommonAnnotationBeanPostProcessor：这个类在spring-context包定义，继承于InitDestroyAnnotationBeanPostProcessor，主要用于处理JDK规范提供的注解，如@PostConstruct和@PreDestroy。在构造函数中，通过InitDestroyAnnotationBeanPostProcessor的setInitAnnotationType和setDestroyAnnotationType分别注册@PostConstruct和@PreDestroy：

	```java
	public CommonAnnotationBeanPostProcessor() {
		setOrder(Ordered.LOWEST_PRECEDENCE - 3);
		setInitAnnotationType(PostConstruct.class);
		setDestroyAnnotationType(PreDestroy.class);
		ignoreResourceType("javax.xml.ws.WebServiceContext");
	}
	```

### 总结
* 由以上分析可知，不过是哪种方式实现的构造后置处理和销毁前置处理，在BeanFactory层面：
* 对应构造后置处理，是在BeanFactory的initializeBean方法实现的，依次调用BeanPostProcessor定义的构造后置处理，InitializingBean的afterPropertiesSet，init-method指定的自定义方法；
* 对应销毁前置处理，是从destroySingletons方法发起，针对每个bean，是在DisposableBeanAdapter的destroy方法中实现的，依次调用BeanPostProcessor的销毁前置处理，调用bean的DisposableBean接口实现的destroy方法，调用destroy-method中指定的自定义方法。
