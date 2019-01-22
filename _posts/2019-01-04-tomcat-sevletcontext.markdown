---
layout:     post
title:      "Tomcat源码分析（四）：ServletContext应用启动之核心组件初始化"
subtitle:   "Tomcat servletContext initializing"
date:       2019-01-04 08:12:00
author:     "XYZ"
header-img: "img/article-bg1.jpg"
tags:
    - Tomcat源码分析
---
### 概述
* 由我的上篇文章[Tomcat源码分析（三）：ServletContext应用启动之配置解析](https://blog.csdn.net/u010013573/article/details/86417019)可知，在Tomcat启动应用StandardContext时，是先通过listener事件机制的方式，交给ContextConfig，解析web.xml类来获取应用的配置信息，包含过滤器filter，监听器listener，如spring的ContextLoaderListener，servlet，如spring的DispatchServlet，以及session配置等。但是配置解析这边只是创建一个包装类，如过滤器的FilterDef，servlet的StandardWrapper，获取这些组件对应的类的全限定名以及init params等数据保存到包装类中，而不进行实际类对象的创建。
* 其次在Servlet3.0之后，提供了WebApplicationInitializer接口来支持以编程方式配置web.xml的内容，故在ContextConfig中也会查找应用中的WebApplicationInitializer的实例类，然后保存到StandardContext的initializers中。注意在ContextConfig并不会对WebApplicationInitializer进行解析，即调用其onStartup方法，而是留在StardardContext中执行。

    ```java
    /**
     * The ordered set of ServletContainerInitializers for this web application.
     */
     // map的value为WebApplicationInitializer的集合
    private Map<ServletContainerInitializer,Set<Class<?>>> initializers =
        new LinkedHashMap<>();
    ```

* 具体以上对象实例的创建，如servlet, filter，listener等，是之后在StandardContext中继续完成。即在StandardContext的startInternal方法中，以如下顺序继续完成启动和相关类对象实例的创建。

### 1. context初始化参数加载
* context初始化参数主要包括两种，一种是在web.xml（或者WebApplicationInitializer，以下提到web.xml的类似）配置context-param设置，如下：

    ```xml
    <context-param>  
        <param-name>contextConfigLocation</param-name>  
        <param-value>classpath:spring/applicationContext.xml</param-value>  
    </context-param>
    ```
* 另外一种则是在Server.xml的Context节点下面的Parameter节点配置，或者在Context.xml中配置，这些是提供某个param默认值的方式，可以被web.xml中的覆盖。然后在Digester解析规则会添加Parameter解析器，具体为在ContextRuleSet中定义，如下：
    ```java
    digester.addObjectCreate(prefix + "Context/Parameter",
                             "org.apache.tomcat.util.descriptor.web.ApplicationParameter");
    digester.addSetProperties(prefix + "Context/Parameter");
    digester.addSetNext(prefix + "Context/Parameter",
                        "addApplicationParameter",
                        "org.apache.tomcat.util.descriptor.web.ApplicationParameter");
    ```
* 在ContextConfig中已经解析好了在web.xml中配置的param，然后接着在StandardContext中，将以上两种方式获取的param填充到servletContext中：

	```java
	/**
	 * Merge the context initialization parameters specified in the application
	 * deployment descriptor with the application parameters described in the
	 * server configuration, respecting the <code>override</code> property of
	 * the application parameters appropriately.
	 */
	private void mergeParameters() {
	    Map<String,String> mergedParams = new HashMap<>();
	    
	    // web.xml
	    String names[] = findParameters();
	    for (int i = 0; i < names.length; i++) {
	        mergedParams.put(names[i], findParameter(names[i]));
	    }
	
	    // Parameter节点
	    ApplicationParameter params[] = findApplicationParameters();
	    for (int i = 0; i < params.length; i++) {
	        if (params[i].getOverride()) {
	            if (mergedParams.get(params[i].getName()) == null) {
	                mergedParams.put(params[i].getName(),
	                        params[i].getValue());
	            }
	        } else {
	            mergedParams.put(params[i].getName(), params[i].getValue());
	        }
	    }
	
	    // 填充到ServletContext中
	    ServletContext sc = getServletContext();
	    for (Map.Entry<String,String> entry : mergedParams.entrySet()) {
	        sc.setInitParameter(entry.getKey(), entry.getValue());
	    }
	
	}
	```
### 2. SCI（ServletContainerInitializers）启动onStartup
* 在ContextConfig查找到应用中的WebApplicationInitializer实现类中，填充保存在StandardContext的initializers集合中，然后在这步调用WebApplicationInitializer的onStartup方法，获取编程方式的应用配置信息。

	```java
	// Call ServletContainerInitializers
	for (Map.Entry<ServletContainerInitializer, Set<Class<?>>> entry :
	    initializers.entrySet()) {
	    try {
	    
	    // 在spring中，则是遍历WebApplicationInitializer集合，
	    // 即entry.getValue()，
	    // 调用WebApplicationInitializer的onStartup方法;
	    // getServletContext()为传递当前应用的servletContext
	    
	        entry.getKey().onStartup(entry.getValue(),
	                getServletContext());
	    } catch (ServletException e) {
	        log.error(sm.getString("standardContext.sciFail"), e);
	        ok = false;
	        break;
	    }
	}
	```

### 3. Listener监听器配置与调用
* 这些一般是应用在web.xml和WebApplicationInitializer中配置的监听器，而不是Tomcat自身添加的监听器，如ContextConfig是Tomcat自身使用Digester解析server.xml时添加的监听器，用于解析配置文件。而这里说的监听器是应用在web.xml或者WebApplicationInitializer中，即配置文件中添加的，类型包括ServletContextListener，HttpSessionListener，ServletRequestListener等，如spring的ContextLoaderListener就是一个ServletContextListener接口的实现类。
* 实现在StandardContext的listenerStart方法，核心实现如下：即包括：
   1. EventListener和LifeCycleListener的创建
   2. 生命周期监听器LifeCycleListener的调用

    ```java
    /**
     * Configure the set of instantiated application event listeners
     * for this Context.
     * @return <code>true</code> if all listeners wre
     * initialized successfully, or <code>false</code> otherwise.
     */
    public boolean listenerStart() {
    
        ...
    
        // 1. 创建listener对象实例
        // Instantiate the required listeners
        String listeners[] = findApplicationListeners();
        Object results[] = new Object[listeners.length];
        boolean ok = true;
        for (int i = 0; i < results.length; i++) {
            
            ...
            
            String listener = listeners[i];
            results[i] = getInstanceManager().newInstance(listener);
            
            ...
        }
        
        ...
    
        // 2. 分类：分为EventListener和LifeCycleListener，
        // 并保存到StandardContext中
        // EventListener是在应用运行过程中调用的
        // LifetCycleListener是应用启动，关闭等中调用的
        
        // Sort listeners in two arrays
        ArrayList<Object> eventListeners = new ArrayList<>();
        ArrayList<Object> lifecycleListeners = new ArrayList<>();
        for (int i = 0; i < results.length; i++) {
            if ((results[i] instanceof ServletContextAttributeListener)
                || (results[i] instanceof ServletRequestAttributeListener)
                || (results[i] instanceof ServletRequestListener)
                || (results[i] instanceof HttpSessionIdListener)
                || (results[i] instanceof HttpSessionAttributeListener)) {
                eventListeners.add(results[i]);
            }
            if ((results[i] instanceof ServletContextListener)
                || (results[i] instanceof HttpSessionListener)) {
                lifecycleListeners.add(results[i]);
            }
        }
    
        // Listener instances may have been added directly to this Context by
        // ServletContextInitializers and other code via the pluggability APIs.
        // Put them these listeners after the ones defined in web.xml and/or
        // annotations then overwrite the list of instances with the new, full
        // list.
        for (Object eventListener: getApplicationEventListeners()) {
            eventListeners.add(eventListener);
        }
        setApplicationEventListeners(eventListeners.toArray());
        for (Object lifecycleListener: getApplicationLifecycleListeners()) {
            lifecycleListeners.add(lifecycleListener);
            if (lifecycleListener instanceof ServletContextListener) {
                noPluggabilityListeners.add(lifecycleListener);
            }
        }
        setApplicationLifecycleListeners(lifecycleListeners.toArray());
    
        ...
        
        // 3. 调用LifetCycleListener，处理应用启动初始化事件
        // 这里只调用LifetCycleListener，
        // 不调用EventListener
        
        // Send application start events
        // Ensure context is not null
        getServletContext();
        context.setNewServletContextListenerAllowed(false);
    
        Object instances[] = getApplicationLifecycleListeners();
        if (instances == null || instances.length == 0) {
            return ok;
        }
    
        ServletContextEvent event = new ServletContextEvent(getServletContext());
        ServletContextEvent tldEvent = null;
        if (noPluggabilityListeners.size() > 0) {
            noPluggabilityServletContext = new NoPluggabilityServletContext(getServletContext());
            tldEvent = new ServletContextEvent(noPluggabilityServletContext);
        }
        for (int i = 0; i < instances.length; i++) {
            // 只调用ServletContextListener
            // spring的ContextLoaderListener就是
            // ServletContextListener的实现类
            if (!(instances[i] instanceof ServletContextListener))
                continue;
            ServletContextListener listener =
                (ServletContextListener) instances[i];
            try {
                fireContainerEvent("beforeContextInitialized", listener);
                if (noPluggabilityListeners.contains(listener)) {
                    listener.contextInitialized(tldEvent);
                } else {
                    // 调用contextInitialized方法
                    // 可以通过event获取ServletContext
                    // spring的ContextLoaderListener
                    // 是在这里加载spring的root WebApplicationContet
                    // 并作为一个attribute填充到ServletContext中
                    
                    listener.contextInitialized(event);
                }
                fireContainerEvent("afterContextInitialized", listener);
            }
            
            ...
            
        }
        return (ok);
    }
    ```
### 4. 过滤器Filter的实例化
* 遍历StandardContext的filterDefs集合，创建ApplicationFilterConfig，然后填充到filterConfigs中。其中在创建ApplicationFilterConfig时，会创建filter实例并进行初始化。

	```java
	/**
	 * Configure and initialize the set of filters for this Context.
	 * @return <code>true</code> if all filter initialization completed
	 * successfully, or <code>false</code> otherwise.
	 */
	public boolean filterStart() {
	
	    if (getLogger().isDebugEnabled()) {
	        getLogger().debug("Starting filters");
	    }
	    // Instantiate and record a FilterConfig for each defined filter
	    boolean ok = true;
	    synchronized (filterConfigs) {
	        filterConfigs.clear();
	        for (Entry<String,FilterDef> entry : filterDefs.entrySet()) {
	            String name = entry.getKey();
	            if (getLogger().isDebugEnabled()) {
	                getLogger().debug(" Starting filter '" + name + "'");
	            }
	            try {
	            
	            // 创建ApplicationFilterConfig实例，
	            // 并放到StandardContext的filterConfigs中
	                ApplicationFilterConfig filterConfig =
	                        new ApplicationFilterConfig(this, entry.getValue());
	                filterConfigs.put(name, filterConfig);
	            } catch (Throwable t) {
	                t = ExceptionUtils.unwrapInvocationTargetException(t);
	                ExceptionUtils.handleThrowable(t);
	                getLogger().error(sm.getString(
	                        "standardContext.filterStart", name), t);
	                ok = false;
	            }
	        }
	    }
	
	    return ok;
	}
	
	
	/**
	 * Construct a new ApplicationFilterConfig for the specified filter
	 * definition.
	 *
	 * @param context The context with which we are associated
	 * @param filterDef Filter definition for which a FilterConfig is to be
	 *  constructed
	 *
	 * @exception ClassCastException if the specified class does not implement
	 *  the <code>javax.servlet.Filter</code> interface
	 * @exception ClassNotFoundException if the filter class cannot be found
	 * @exception IllegalAccessException if the filter class cannot be
	 *  publicly instantiated
	 * @exception InstantiationException if an exception occurs while
	 *  instantiating the filter object
	 * @exception ServletException if thrown by the filter's init() method
	 * @throws NamingException
	 * @throws InvocationTargetException
	 * @throws SecurityException
	 * @throws NoSuchMethodException
	 * @throws IllegalArgumentException
	 */
	ApplicationFilterConfig(Context context, FilterDef filterDef)
	        throws ClassCastException, ClassNotFoundException, IllegalAccessException,
	        InstantiationException, ServletException, InvocationTargetException, NamingException,
	        IllegalArgumentException, NoSuchMethodException, SecurityException {
	
	    super();
	
	    this.context = context;
	    this.filterDef = filterDef;
	    // Allocate a new filter instance if necessary
	    if (filterDef.getFilter() == null) {
	        getFilter();
	    } else {
	        this.filter = filterDef.getFilter();
	        
	        // 创建filter实例
	        getInstanceManager().newInstance(filter);
	        // 初始化filter，如添加filter配置的param等
	        initFilter();
	    }
	}
	```

### 5. Servlet的实例化
* 主要为查找配置的Servlet中onStartup > 0的serlvet，然后加载Servlet对应的类，创建servlet实例并初始化。具体为通过StandardWrapper的load方法来完成。

	```java
	/**
	 * Load and initialize all servlets marked "load on startup" in the
	 * web application deployment descriptor.
	 *
	 * @param children Array of wrappers for all currently defined
	 *  servlets (including those not declared load on startup)
	 * @return <code>true</code> if load on startup was considered successful
	 */
	public boolean loadOnStartup(Container children[]) {
	
	    // 筛选loadOnStartup > 0的servlet
	    
	    // Collect "load on startup" servlets that need to be initialized
	    TreeMap<Integer, ArrayList<Wrapper>> map = new TreeMap<>();
	    for (int i = 0; i < children.length; i++) {
	        Wrapper wrapper = (Wrapper) children[i];
	        int loadOnStartup = wrapper.getLoadOnStartup();
	        if (loadOnStartup < 0)
	            continue;
	        Integer key = Integer.valueOf(loadOnStartup);
	        ArrayList<Wrapper> list = map.get(key);
	        if (list == null) {
	            list = new ArrayList<>();
	            map.put(key, list);
	        }
	        list.add(wrapper);
	    }
	
	    // Load the collected "load on startup" servlets
	    for (ArrayList<Wrapper> list : map.values()) {
	        for (Wrapper wrapper : list) {
	            try {
	            
	            // 通过StandardWrapper的load方法来完成
	            // 类加载，实例创建，实例初始化
	            
	                wrapper.load();
	            } catch (ServletException e) {
	                getLogger().error(sm.getString("standardContext.loadOnStartup.loadException",
	                      getName(), wrapper.getName()), StandardWrapper.getRootCause(e));
	                // NOTE: load errors (including a servlet that throws
	                // UnavailableException from the init() method) are NOT
	                // fatal to application startup
	                // unless failCtxIfServletStartFails="true" is specified
	                if(getComputedFailCtxIfServletStartFails()) {
	                    return false;
	                }
	            }
	        }
	    }
	    return true;
	
	}
	
	
	/**
	 * Load and initialize an instance of this servlet, if there is not already
	 * at least one initialized instance.  This can be used, for example, to
	 * load servlets that are marked in the deployment descriptor to be loaded
	 * at server startup time.
	 * <p>
	 * <b>IMPLEMENTATION NOTE</b>:  Servlets whose classnames begin with
	 * <code>org.apache.catalina.</code> (so-called "container" servlets)
	 * are loaded by the same classloader that loaded this class, rather than
	 * the classloader for the current web application.
	 * This gives such classes access to Catalina internals, which are
	 * prevented for classes loaded for web applications.
	 *
	 * @exception ServletException if the servlet init() method threw
	 *  an exception
	 * @exception ServletException if some other loading problem occurs
	 */
	@Override
	public synchronized void load() throws ServletException {
	    instance = loadServlet();
	
	    if (!instanceInitialized) {
	        initServlet(instance);
	    }
	
	    if (isJspServlet) {
	        StringBuilder oname = new StringBuilder(getDomain());
	
	        oname.append(":type=JspMonitor");
	
	        oname.append(getWebModuleKeyProperties());
	
	        oname.append(",name=");
	        oname.append(getName());
	
	        oname.append(getJ2EEKeyProperties());
	
	        try {
	            jspMonitorON = new ObjectName(oname.toString());
	            Registry.getRegistry(null, null)
	                .registerComponent(instance, jspMonitorON, null);
	        } catch( Exception ex ) {
	            log.info("Error registering JSP monitoring with jmx " +
	                     instance);
	        }
	    }
	}
	```