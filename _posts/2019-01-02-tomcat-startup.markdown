---
layout:     post
title:      "Tomcat源码分析（二）：启动流程和webapp目录应用部署实现"
subtitle:   "Tomcat Startup"
date:       2019-01-02 08:12:00
author:     "XYZ"
header-img: "img/article-bg1.jpg"
tags:
    - Tomcat源码分析
---

### 启动脚本
* tomcat的启动和关闭分别是通过执行bin目录下的startup.sh和shutdown.sh来实现的，而startup.sh和shutdown.sh里面会执行catalina.sh来完成实际的启动和关闭。在catalina.sh里面会获取或设置启动相关的环境变量，然后配置启动的各种参数，如下为启动脚本：可以看到是执行Bootstrap类。
```
touch "$CATALINA_OUT"
if [ "$1" = "-security" ] ; then
  if [ $have_tty -eq 1 ]; then
    echo "Using Security Manager"
  fi
  shift
  eval $_NOHUP "\"$_RUNJAVA\"" "\"$LOGGING_CONFIG\"" $LOGGING_MANAGER $JAVA_OPTS $CATALINA_OPTS \
    -D$ENDORSED_PROP="\"$JAVA_ENDORSED_DIRS\"" \
    -classpath "\"$CLASSPATH\"" \
    -Djava.security.manager \
    -Djava.security.policy=="\"$CATALINA_BASE/conf/catalina.policy\"" \
    -Dcatalina.base="\"$CATALINA_BASE\"" \
    -Dcatalina.home="\"$CATALINA_HOME\"" \
    -Djava.io.tmpdir="\"$CATALINA_TMPDIR\"" \
    org.apache.catalina.startup.Bootstrap "$@" start \
    >> "$CATALINA_OUT" 2>&1 "&"

else
  eval $_NOHUP "\"$_RUNJAVA\"" "\"$LOGGING_CONFIG\"" $LOGGING_MANAGER $JAVA_OPTS $CATALINA_OPTS \
    -D$ENDORSED_PROP="\"$JAVA_ENDORSED_DIRS\"" \
    -classpath "\"$CLASSPATH\"" \
    -Dcatalina.base="\"$CATALINA_BASE\"" \
    -Dcatalina.home="\"$CATALINA_HOME\"" \
    -Djava.io.tmpdir="\"$CATALINA_TMPDIR\"" \
    org.apache.catalina.startup.Bootstrap "$@" start \
    >> "$CATALINA_OUT" 2>&1 "&"
fi
```
* Bootstrap类在加载时，会初始化类加载器体系，主要是commonLoader，catalinaLoader，sharedLoader这三个类加载器（默认为同一个），并使用catalinaLoader来通过放射的方式加载和创建org.apache.catalina.startup.Catalina实现，最后在main方法中，根据脚本执行参数来执行启动或停止。其中启动start时，依次调用Catalina的load和start：
   1. load为加载解析conf/server.xml文件并创建StandardServer，StanderService，StandardHost对象实例，即只要server.xml中存在这些节点，则会创建对应的Container接口的实现类对象实例；
   2. start为从StanderServer开始整个Catalina容器的启动。
```
类加载器：
ClassLoader commonLoader = null;
ClassLoader catalinaLoader = null;
ClassLoader sharedLoader = null;
// -------------------------------------------------------- Private Methods
private void initClassLoaders() {
    try {
        commonLoader = createClassLoader("common", null);
        if( commonLoader == null ) {
            // no config file, default to this loader - we might be in a 'single' env.
            commonLoader=this.getClass().getClassLoader();
        }
        catalinaLoader = createClassLoader("server", commonLoader);
        sharedLoader = createClassLoader("shared", commonLoader);
    } catch (Throwable t) {
        handleThrowable(t);
        log.error("Class loader creation threw exception", t);
        System.exit(1);
    }
}

Catalina启动：
private Object catalinaDaemon = null;
daemon.setAwait(true);
daemon.load(args);
daemon.start();
if (null == daemon.getServer()) {
    System.exit(1);
}
```

### 启动流程
#### server.xml文件解析
* tomcat中对xml配置文件的解析是通过Digester来完成的。xml文件的解析一般有Dom4j和SAX两种方式组成，其中Dom4j为在内存中构建一棵Dom树来解析的，而SAX则是读取输入流，然后在读到某个xml节点时，触发相应的节点事件并回调相应的处理方法来实现的。而Digester在SAX的基础上，进行了封装，使得更加方便使用，目前Digester已经是一个Apache commons子项目：[Digester](http://commons.apache.org/proper/commons-digester/)
* Catalina的load方法的核心实现：

```
public void load() {
       // 创建digester实例，定义xml节点的解析规则
       // Create and execute our Digester
       Digester digester = createStartDigester();

       // 读取server.xml文件
       InputSource inputSource = null;
       InputStream inputStream = null;
       File file = null;
       try {
           try {
               file = configFile();
               inputStream = new FileInputStream(file);
               inputSource = new InputSource(file.toURI().toURL().toString());
           } catch (Exception e) {
               if (log.isDebugEnabled()) {
                   log.debug(sm.getString("catalina.configFail", file), e);
               }
           }
           
         ...
		 // 解析server.xml文件输入流并创建xml节点对应的对象实例
           try {
               inputSource.setByteStream(inputStream);
               digester.push(this);
               digester.parse(inputSource);
           }
           
         ...
}
```
* 以上代码核心方法为createStartDigester，在这个方法中定义server.xml文件的解析规则，即对每个xml节点，根据pattern定义该xml节点匹配的tomcat中的类，以及为该类对象实例填充属性对象实例。最后在digester.parse(inputSource)方法中，根据解析规则，完成对象的创建和属性值的注入。
   1. 简单节点解析规则定义：以Server节点为例
		
		第一步：
		```
		// public void addObjectCreate(String pattern, String className, String attributeName)
		// pattern：匹配到Server节点
		// className：Server节点默认对应的tomcat的类
		// attributeName：在Server节点中是否存在className属性指定了某个类，
		// 有则使用该类，否则使用默认的org.apache.catalina.core.StandardServer
		digester.addObjectCreate("Server",
		                                 "org.apache.catalina.core.StandardServer",
		                                 "className");
		 ```
		第二步：       
		```
		// public void addSetProperties(String pattern)
		// pattern：读取Server节点的所有节点属性值，然后填充到org.apache.catalina.core.StandardServer的属性中
		digester.addSetProperties("Server");
		```
		第三步：
	  ```	 
		// public void addSetNext(String pattern, String methodName, String paramType)
		// pattern：匹配到Server节点
		// methodName：调用创建了StandardServer实例的parent类的setServer方法，
		// 将StandardServer对象赋值到该parent类的属性值StandardServer。
		// 在这里是调用Catalina的setServer方法。
		// paramType：setServer的方法参数类型为org.apache.catalina.Server
		digester.addSetNext("Server",
		                    "setServer",
		                    "org.apache.catalina.Server");
		```
   2. 复杂节点解析规则定义：通过Rule来定义复杂节点，即节点里面还包含多级节点。addRuleSet方法和addRule，其中addRule为具体添加一条解析规则。
		```
		// Add RuleSets for nested elements
		digester.addRuleSet(new NamingRuleSet("Server/GlobalNamingResources/"));
		digester.addRuleSet(new EngineRuleSet("Server/Service/"));
		digester.addRuleSet(new HostRuleSet("Server/Service/Engine/"));
		digester.addRuleSet(new ContextRuleSet("Server/Service/Engine/Host/"));
		addClusterRuleSet(digester, "Server/Service/Engine/Host/Cluster/");
		digester.addRuleSet(new NamingRuleSet("Server/Service/Engine/Host/Context/"));
		```
   3. HostRuleSet：定义Host节点的解析和创建Host对象实例的规则，其中addRule为该pattern匹配到的节点，在这里是Server/Service/Engine/Host，定义一条具体的解析规则，多次调用则定义多条规则，规则接口为Rule，在Rule节点的begin方法中定义具体的回调方法。
		```
		public void addRuleInstances(Digester digester) {
		    digester.addObjectCreate(prefix + "Host",
		                             "org.apache.catalina.core.StandardHost",
		                             "className");
		    digester.addSetProperties(prefix + "Host");
		    digester.addRule(prefix + "Host",
		                     new CopyParentClassLoaderRule());
		                     
		    // 核心关注这个Rule，org.apache.catalina.startup.HostConfig为
		    // 完成 webapp目录下的应用的解析，然后创建Host的child，即Context                 
		    digester.addRule(prefix + "Host",
		                     new LifecycleListenerRule
		                     ("org.apache.catalina.startup.HostConfig",
		                      "hostConfigClass"));
		    digester.addSetNext(prefix + "Host",
		                        "addChild",
		                        "org.apache.catalina.Container");
		    ...
		}
		```
  4. LifecycleListenerRule：添加监听器Listener，构造函数如下。上面那个addRule的listenerClass为：org.apache.catalina.startup.HostConfig，attributeName为：hostConfigClass，即为StanderHost对象添加了一个HostConfig到Listener列表，其中HostConfig是一个LifeCycleListener。
		```
		/**
		 * Construct a new instance of this Rule.
		 * @param listenerClass Default name of the LifecycleListener
		 *  implementation class to be created
		 * @param attributeName Name of the attribute that optionally
		 *  includes an override name of the LifecycleListener class
		 */
		public LifecycleListenerRule(String listenerClass, String attributeName) {
		    this.listenerClass = listenerClass;
		    this.attributeName = attributeName;
		}
		    
		public void begin(String namespace, String name, Attributes attributes)
		    throws Exception {
		    Container c = (Container) digester.peek();
		    
		    ... 
		    
		    // Use the default
		    if (className == null) {
		        className = listenerClass;
		    }
		
		    // Instantiate a new LifecycleListener implementation object
		    Class<?> clazz = Class.forName(className);
		    LifecycleListener listener = (LifecycleListener) clazz.getConstructor().newInstance();
		
		    // Add this LifecycleListener to our associated component
		    c.addLifecycleListener(listener);
		}
		```

   
#### listener事件监听机制完成Container接口组件的创建
* 由上分析可知在Host节点对应的StanderHost类的生命周期监听器LifeCycleListener列表，添加了一个org.apache.catalina.startup.HostConfig监听器实现。
* 在Catalina体系的核心组件的生命周期是通过LifeCycle来管理的，即初始化，启动，停止都对应相关的状态和listener事件产生，并交给相应的监听器Listener处理。对于核心组件Host，Context的启动，分别是交给org.apache.catalina.startup包的HostConfig和ContextConfig来完成的。
1. StandardHost启动实现：startInternal
```
代码1：
protected synchronized void startInternal() throws LifecycleException {
    ... 
    super.startInternal();
}

代码2：
super.startInternal()：即ContainerBase类的startInternal
protected synchronized void startInternal() throws LifecycleException {

    ...
    // 设置STARTING状态
    setState(LifecycleState.STARTING);

   ...
}

代码3：
LifeCycleBase的fireLifecycleEvent：调用listener的lifecycleEvent方法。
protected void fireLifecycleEvent(String type, Object data) {
   LifecycleEvent event = new LifecycleEvent(this, type, data);
    for (LifecycleListener listener : lifecycleListeners) {
        listener.lifecycleEvent(event);
    }
}
```

2. HostConfig的lifecycleEvent方法处理Lifecycle.START_EVENT：调用deployApps方法部署webapp目录的应用。底层实现主要为遍历docBase指定的目录，默认为webapp，然后对每个xxx.war文件，创建一个StandardContext实例，即一个应用，并添加到StandardHost的children列表中。注意这里并不进行web.xm文件的解析，只是如果在xxx.war包内存在META-INF/context.xml文件，则解析并用来填充这个StandardContext的属性。

```
/**
 * Process a "start" event for this Host.
 */
public void start() {
    ...
    
    if (!host.getAppBaseFile().isDirectory()) {
        log.error(sm.getString("hostConfig.appBase", host.getName(),
                host.getAppBaseFile().getPath()));
        host.setDeployOnStartup(false);
        host.setAutoDeploy(false);
    }

    if (host.getDeployOnStartup())
        deployApps();
        
    ...
}

添加到host中的实现：
Class<?> clazz = Class.forName(host.getConfigClass());
LifecycleListener listener = (LifecycleListener) clazz.getConstructor().newInstance();
context.addLifecycleListener(listener);
context.setName(cn.getName());
context.setPath(cn.getPath());
context.setWebappVersion(cn.getVersion());
context.setDocBase(cn.getBaseName() + ".war");
host.addChild(context);
```
#### webapp目录应用部署
* 由上可知，HostConfig底层调用deployWAR方法来遍历webapp目录，并解压war包，解析META-INF/context.xml文件，创建StandardContext对象，然后加到StandardHost的children，即子容器列表。之后在StandardHost的startInternal中遍历已经填充好的children，调用StandardContext的start方法，从而对每个Context进行启动，初始化机制也是跟StandardHost一样，使用listener事件监听机制来完成实际创建，具体为交给ContextConfig。
```
// Notify our interested LifecycleListeners
fireLifecycleEvent(Lifecycle.CONFIGURE_START_EVENT, null);

 // Start our child containers, if not already started
 for (Container child : findChildren()) {
     if (!child.getState().isAvailable()) {
         child.start();
     }
 }
```

* ContextConfig：与HostConfig类似，不过是在StandardContext的startInternal方法，触发ContextConfig的lifecycleEvent方法：

```
StandardContext的startInternal：
// Notify our interested LifecycleListeners
fireLifecycleEvent(Lifecycle.CONFIGURE_START_EVENT, null);
```

* ContextConfig底层调用configureStart来启动应用，核心方法为webConfig。
```
/**
  * Process a "contextConfig" event for this Context.
  */
 protected synchronized void configureStart() {
     // Called from StandardContext.start()

	... 
	// 核心逻辑，解析web.xml，填充StanderContext的属性
     webConfig();

     if (!context.getIgnoreAnnotations()) {
         applicationAnnotationsConfig();
     }
     if (ok) {
         validateSecurityRoles();
     }

     // Configure an authenticator if we need one
     if (ok) {
         authenticatorConfig();
     }
     
     ...
     
     // Make our application available if no problems were encountered
     if (ok) {
         context.setConfigured(true);
     } else {
         log.error(sm.getString("contextConfig.unavailable"));
         context.setConfigured(false);
     }
 }
```
* webConfig：是tomcat非常重要的一个方法，主要是根据servlet规范，解析web.xml，以及合并包含的jar包里面的web-fragment.xml文件到web.xml，获取注解信息等等工作，完成对应java servlet的ServletContext的创建，代表一个应用的配置和启动。具体在下一篇文章分析。
[Tomcat源码分析：web.xml解析与JavaConfig实现基础ServletContainerInitializer的处理](https://blog.csdn.net/u010013573/article/details/86417019)