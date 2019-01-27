---
layout:     post
title:      "Spring基于@Configuration的类配置的内部源码实现"
subtitle:   "Spring @Configuration source code implementation"
date:       2019-01-11 20:49:00
author:     "XYZ"
header-img: "img/article-bg1.jpg"
tags:
    - Spring源码分析
---

### 概述
* Spring容器启动时，即执行refresh方法时，主要是通过执行ConfigurationClassPostProcessor这个BeanFactoryPostProcessor，来开启整个@Configuration注解的系列类的加载的，即开启基于@Configuration的类配置代替beans标签的容器配置的相关bean的加载。
* 而ConfigurationClassPostProcessor注册到Spring容器的BeanFactoryPostProcessor列表，主要是在AnnotationConfigUtils的registerAnnotationConfigProcessors定义的。如下：

	```java
	/**
	 * Register all relevant annotation post processors in the given registry.
	 * @param registry the registry to operate on
	 * @param source the configuration source element (already extracted)
	 * that this registration was triggered from. May be {@code null}.
	 * @return a Set of BeanDefinitionHolders, containing all bean definitions
	 * that have actually been registered by this call
	 */
	public static Set<BeanDefinitionHolder> registerAnnotationConfigProcessors(
			BeanDefinitionRegistry registry, @Nullable Object source) {
	
	    ...
	    
		Set<BeanDefinitionHolder> beanDefs = new LinkedHashSet<>(8);
	
		if (!registry.containsBeanDefinition(CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME)) {
		
		    // 注册ConfigurationClassPostProcessor
		    // 用于处理@Configuration注解的类
		    
			RootBeanDefinition def = new RootBeanDefinition(ConfigurationClassPostProcessor.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME));
		}
	
		if (!registry.containsBeanDefinition(AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
		
		    // 注册AutowiredAnnotationBeanPostProcessor
		    
			RootBeanDefinition def = new RootBeanDefinition(AutowiredAnnotationBeanPostProcessor.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
		}
	
		// Check for JSR-250 support, and if present add the CommonAnnotationBeanPostProcessor.
		if (jsr250Present && !registry.containsBeanDefinition(COMMON_ANNOTATION_PROCESSOR_BEAN_NAME)) {
		
		    // 注册CommonAnnotationBeanPostProcessor
		    
			RootBeanDefinition def = new RootBeanDefinition(CommonAnnotationBeanPostProcessor.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, COMMON_ANNOTATION_PROCESSOR_BEAN_NAME));
		}
	}
	```

* 而AnnotationConfigUtils的registerAnnotationConfigProcessors的被调用，则可以通过多种方式：
   1. ComponentScanBeanDefinitionParser：对应context:component-scan，在parse方法调用；
   2. AnnotationConfigBeanDefinitionParser：对应context:annotation-config，在parse方法调用；
   3. ClassPathBeanDefinitionScanner：在scan方法调用，该类在AnnotationConfigApplicationContext中使用；
   4. AnnotatedBeanDefinitionReader：在构造函数调用，该类在AnnotationConfigApplicationContext中使用。

* 由以上四种方式可知，当在xml文件中配置了context:component-scan或者context:annotation-config；或者使用AnnotationConfigApplicationContext作为spring容器的ApplicationContext，这种方式通常在结合实现WebApplicationInitializer接口，来实现全基于Java类配置时用到；则可以自动激活@Configuration注解系列配置类，及其内部的@Bean方法对应bean的加载。

### 类结构设计
#### 1. ConfigurationClassPostProcessor
* 作为BeanFactoryPostProcessor，在spring容器ApplicationContext启动调用refresh方法时，遍历并调用BeanFactoryPostProcessor时被调用，执行postProcessBeanDefinitionRegistry方法，如下：

    ```java
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
    
                // 调用BeanFactoryPostProcessor
                
    			// Invoke factory processors registered as beans in the context.
    			invokeBeanFactoryPostProcessors(beanFactory);
    
    			// Register bean processors that intercept bean creation.
    			registerBeanPostProcessors(beanFactory);
        
        ...
        
    	}
    }
    ```
    invokeBeanFactoryPostProcessors方法实现：
    
    ```java
    /**
     * Instantiate and invoke all registered BeanFactoryPostProcessor beans,
     * respecting explicit order if given.
     * <p>Must be called before singleton instantiation.
     */
    protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
    
        // 在这里会调用到
        // ConfigurationClassPostProcessor的postProcessBeanDefinitionRegistry方法
        
    	PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());
    
    	// Detect a LoadTimeWeaver and prepare for weaving, if found in the meantime
    	// (e.g. through an @Bean method registered by ConfigurationClassPostProcessor)
    	if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
    		beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
    		beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
    	}
    }
    ```

* 在该方法内先从BeanFactory中获取使用了@Configuration注解的类对应的beanDefinitions集合，即configCandidates。然后将configCandidates交给ConfigurationClassParser处理；
* ConfigurationClassParser处理会获取这些类对应的ConfigurationClass类型的集合，然后交给ConfigurationClassBeanDefinitionReader处理，从中处理@Bean方法，将对应的bean注册到BeanFactory。
* ConfigurationClassPostProcessor的processConfigBeanDefinitions相关代码如下：其中刚开始能从BeanFactory获取@Configuration的类对应beanDefinition，是因为@Configuration是@Component的一个子类，所以通常在配置component-scan时，需要能加载到@Configuration注解的类所在的包。

    ```java
    /**
     * Build and validate a configuration model based on the registry of
     * {@link Configuration} classes.
     */
    public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
    	List<BeanDefinitionHolder> configCandidates = new ArrayList<>();
    	String[] candidateNames = registry.getBeanDefinitionNames();
        
        // 从BeanFactory中获取使用@Configuration注解的类
        
    	for (String beanName : candidateNames) {
    		BeanDefinition beanDef = registry.getBeanDefinition(beanName);
    		if (ConfigurationClassUtils.isFullConfigurationClass(beanDef) ||
    				ConfigurationClassUtils.isLiteConfigurationClass(beanDef)) {
    			if (logger.isDebugEnabled()) {
    				logger.debug("Bean definition has already been processed as a configuration class: " + beanDef);
    			}
    		}
    		else if (ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory)) {
    			configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));
    		}
    	}
    
        // 没有则直接返回
        
    	// Return immediately if no @Configuration classes were found
    	if (configCandidates.isEmpty()) {
    		return;
    	}
    
    	// Sort by previously determined @Order value, if applicable
    	configCandidates.sort((bd1, bd2) -> {
    		int i1 = ConfigurationClassUtils.getOrder(bd1.getBeanDefinition());
    		int i2 = ConfigurationClassUtils.getOrder(bd2.getBeanDefinition());
    		return Integer.compare(i1, i2);
    	});
    
    	...
    	
    	// 使用ConfigurationClassParser处理这些类上面的注解，
    	// 如@ComponentScan，@Import，@PropertySource等
    	// 并从这些类生成一个ConfigurationClass类型的列表
    
    	// Parse each @Configuration class
    	ConfigurationClassParser parser = new ConfigurationClassParser(
    			this.metadataReaderFactory, this.problemReporter, this.environment,
    			this.resourceLoader, this.componentScanBeanNameGenerator, registry);
    
    	Set<BeanDefinitionHolder> candidates = new LinkedHashSet<>(configCandidates);
    	Set<ConfigurationClass> alreadyParsed = new HashSet<>(configCandidates.size());
    	do {
    	
    	    // 执行解析
    	    
    		parser.parse(candidates);
    		parser.validate();
    
    		Set<ConfigurationClass> configClasses = new LinkedHashSet<>(parser.getConfigurationClasses());
    		configClasses.removeAll(alreadyParsed);
    
            // 交给ConfigurationClassBeanDefinitionReader
            // 完成@Configuration注解的类内部注解的处理
            // 如@Bean，嵌套@Configuration等。
            
    		// Read the model and create bean definitions based on its content
    		if (this.reader == null) {
    			this.reader = new ConfigurationClassBeanDefinitionReader(
    					registry, this.sourceExtractor, this.resourceLoader, this.environment,
    					this.importBeanNameGenerator, parser.getImportRegistry());
    		}
    		
    		// 注册@Bean方法对应的bean到BeanFactory
    		
    		this.reader.loadBeanDefinitions(configClasses);
    	
    	...
    	
    }
    ```
#### 2. ConfigurationClassParser
* 遍历并处理configCandidates集合：
   1. 针对集合中的每个@Configuration注解的类时，会处理该类上同时配置的其他注解，如@ComponentScan，扫描指定的包并注册有相关注解的类到BeanFactory中；还包括其他注解的处理，如@Import，@PropertySource等。
   2. 同时会创建该@Configuration注解的类对应的一个类型为ConfigurationClass的对象，将该类中@Bean注解的方式生成BeanMethod，放到类型为ConfigurationClass的对象内部的一个集合beanMethods（类型为LinkedHashSet）里面。并将该类型为ConfigurationClass的对象，放到ConfigurationClassParser的一个map（类型为LinkedHashMap），其中key和value，均为该类型为ConfigurationClass的对象。
* 处理方法源码如下：
	```java
	/**
	 * Apply processing and build a complete {@link ConfigurationClass} by reading the
	 * annotations, members and methods from the source class. This method can be called
	 * multiple times as relevant sources are discovered.
	 * @param configClass the configuration class being build
	 * @param sourceClass a source class
	 * @return the superclass, or {@code null} if none found or previously processed
	 */
	@Nullable
	protected final SourceClass doProcessConfigurationClass(ConfigurationClass configClass, SourceClass sourceClass)
			throws IOException {
		if (configClass.getMetadata().isAnnotated(Component.class.getName())) {
			// Recursively process any member (nested) classes first
			processMemberClasses(configClass, sourceClass);
		}
	
		// Process any @PropertySource annotations
		for (AnnotationAttributes propertySource : AnnotationConfigUtils.attributesForRepeatable(
				sourceClass.getMetadata(), PropertySources.class,
				org.springframework.context.annotation.PropertySource.class)) {
			if (this.environment instanceof ConfigurableEnvironment) {
				processPropertySource(propertySource);
			}
			else {
				logger.info("Ignoring @PropertySource annotation on [" + sourceClass.getMetadata().getClassName() +
						"]. Reason: Environment must implement ConfigurableEnvironment");
			}
		}
	
		// Process any @ComponentScan annotations
		Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(
				sourceClass.getMetadata(), ComponentScans.class, ComponentScan.class);
		if (!componentScans.isEmpty() &&
				!this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(), ConfigurationPhase.REGISTER_BEAN)) {
			for (AnnotationAttributes componentScan : componentScans) {
				// The config class is annotated with @ComponentScan -> perform the scan immediately
				Set<BeanDefinitionHolder> scannedBeanDefinitions =
						this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());
				// Check the set of scanned definitions for any further config classes and parse recursively if needed
				for (BeanDefinitionHolder holder : scannedBeanDefinitions) {
					BeanDefinition bdCand = holder.getBeanDefinition().getOriginatingBeanDefinition();
					if (bdCand == null) {
						bdCand = holder.getBeanDefinition();
					}
					if (ConfigurationClassUtils.checkConfigurationClassCandidate(bdCand, this.metadataReaderFactory)) {
						parse(bdCand.getBeanClassName(), holder.getBeanName());
					}
				}
			}
		}
	
		// Process any @Import annotations
		processImports(configClass, sourceClass, getImports(sourceClass), true);
	
		// Process any @ImportResource annotations
		AnnotationAttributes importResource =
				AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(), ImportResource.class);
		if (importResource != null) {
			String[] resources = importResource.getStringArray("locations");
			Class<? extends BeanDefinitionReader> readerClass = importResource.getClass("reader");
			for (String resource : resources) {
				String resolvedResource = this.environment.resolveRequiredPlaceholders(resource);
				configClass.addImportedResource(resolvedResource, readerClass);
			}
		}
	
		// Process individual @Bean methods
		Set<MethodMetadata> beanMethods = retrieveBeanMethodMetadata(sourceClass);
		for (MethodMetadata methodMetadata : beanMethods) {
			configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
		}
	
		// Process default methods on interfaces
		processInterfaces(configClass, sourceClass);
	
		// Process superclass, if any
		if (sourceClass.getMetadata().hasSuperClass()) {
			String superclass = sourceClass.getMetadata().getSuperClassName();
			if (superclass != null && !superclass.startsWith("java") &&
					!this.knownSuperclasses.containsKey(superclass)) {
				this.knownSuperclasses.put(superclass, configClass);
				// Superclass found, return its annotation metadata and recurse
				return sourceClass.getSuperClass();
			}
		}
		// No superclass -> processing is complete
		return null;
	}
	```

#### 3. ConfigurationClassBeanDefinitionReader
* 调用ConfigurationClassParser的getConfigurationClasses方法，获取以上所说的map的keySet，即configCandidates对应的类型为ConfigurationClass的列表。
* 然后遍历这个ConfigurationClass列表，针对每个ConfigurationClass，使用ConfigurationClassBeanDefinitionReader的loadBeanDefinitions方法，从该ConfigurationClass中获取其内部的@Bean注解的方法列表，即BeanMethod列表，或者嵌套@Configuration等包含的BeanMethod，然后将生成的bean，注册到BeanFactory中。
* 实现源码如下：
	```java
	/**
	 * Read a particular {@link ConfigurationClass}, registering bean definitions
	 * for the class itself and all of its {@link Bean} methods.
	 */
	private void loadBeanDefinitionsForConfigurationClass(
			ConfigurationClass configClass, TrackedConditionEvaluator trackedConditionEvaluator) {
	
		if (trackedConditionEvaluator.shouldSkip(configClass)) {
			String beanName = configClass.getBeanName();
			if (StringUtils.hasLength(beanName) && this.registry.containsBeanDefinition(beanName)) {
				this.registry.removeBeanDefinition(beanName);
			}
			this.importRegistry.removeImportingClass(configClass.getMetadata().getClassName());
			return;
		}
	
		if (configClass.isImported()) {
			registerBeanDefinitionForImportedConfigurationClass(configClass);
		}
	
		// 将@Bean注解的方法注册到BeanFactory中
		
		for (BeanMethod beanMethod : configClass.getBeanMethods()) {
			loadBeanDefinitionsForBeanMethod(beanMethod);
		}
	
		loadBeanDefinitionsFromImportedResources(configClass.getImportedResources());
		loadBeanDefinitionsFromRegistrars(configClass.getImportBeanDefinitionRegistrars());
	}
	```
