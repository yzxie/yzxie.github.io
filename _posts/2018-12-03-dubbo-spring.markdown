---
layout:     post
title:      "Dubbo源码分析：dubbo框架与spring融合"
subtitle:   "Start dubbo through Spring"
date:       2018-12-03 22:12:00
author:     "XYZ"
header-img: "img/article-bg1.jpg"
tags:
    - Dubbo源码分析
---
### 概述
* Dubbo框架主要是用于分布式系统中服务之间的远程调用。而分布式系统中的每个服务一般为采用spring框架搭建，通过spring容器管理beans，通过spring mvc提供restful接口，在service层进行业务逻辑处理。而不管是服务消费者引用的bean，还是服务提供者需要对外提供服务、进行注册的bean，都需要一种机制来触发其进行初始化，生成JVM堆的一个对象实例，同时由spring容器管理，可以通过@Autowired进行注入，从而可以在运行时进行调用。
* 在dubbo源码结构中，主要在dubbo-config模块进行服务建模。具体为在dubbo-config-api子模块对服务进行各粒度的建模，依次为应用、提供者、消费者、注册中心、服务模块、服务提供类和引用类、方法、方法参数；在dubbo-config-spring子模块将消费者配置引用的服务，提供者配置对外提供调用的服务分别融合到相应项目的spring容器中，从而可以在service层通过@Autowired的机制进行注入。
![在这里插入图片描述](https://img-blog.csdnimg.cn/2018120522351678.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)

### 配置解析规则定义
1. 在dubbo.xsd定义dubbo相关标签
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181205223539491.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
2. 在dubbo.schemas定义标签的位置location
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181205223603409.png)
3. dubbo标签处理器器配置：在spring.handlers中定义了dubbo的命名空间和标签处理器DubboNamespaceHandler的对应关系，即当遇到dubbo标签的时候，使用DubboNamespaceHandler进行处理，解析生成beanDefinition，其中DubboNamespaceHandler继承于NamespaceHandlerSupport。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181205223628730.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181205223653553.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
### 解析配置文件并生成beanDefinition
在spring扫描和解析xml文件遇到dubbo标签的时候，会使用DubboNamespaceHandler进行标签处理。
1. DubboNamespaceHandler：自定义了BubboBeanDefinitionParser，使用相应的解析器，解析配置文件的标签，并转换成相应的beanDefinition。
![在这里插入图片描述](https://img-blog.csdnimg.cn/2018120522371894.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181205224904407.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
3. 通过registerBeanDefinitionParser将标签名称和解析器的对应关系注册到NamespaceHandlerSupport中的parsers中，从而在spring扫描和解析xml文件遇到对应的dubbo标签时，会使用相应的DubboBeanDefinitionParser进行标签解析，生成beanDefinition。通过registerBeanDefinitionParser将标签名称和解析器的对应关系注册到NamespaceHandlerSupport中的parsers中，从而在spring扫描和解析xml文件遇到对应的dubbo标签时，会使用相应的DubboBeanDefinitionParser进行标签解析，生成beanDefinition。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181205223745272.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
如果是使用注解而不是配置文件，则使用AnnotationBeanDefinitionParser。
![在这里插入图片描述](https://img-blog.csdnimg.cn/2018120522380842.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)

### 服务提供者提供的服务Service：ServiceBean
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181205223855591.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
1. ServiceBean实现ApplicationListener接口，泛型参数为ContextRefreshedEvent，从而在spring容器启动或重启时会触发。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181205223918387.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
2. ServiceBean继承于ServiceConfig，调用ServiceConfig的export方法，该方法会产生该service的Invoker，即通过该Invoker调用serviceImpl的方法，其中invoker是从该service的具体实现代理ref获取：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181205223941837.png)
包含setRef方法，通过ExtensionLoader注入。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181205224004442.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
3. 由之前的文章分析可知，export最终会调用RegistryProtocol的export方法完成服务导出，注册到注册中心，同时会调用doLocalExport，在提供者本地生成消费者请求处理invoker。具体为根据URL确定所使用的、对应与dubbo-rpc模块的具体rpc协议protocol，如DubboProtocol，调用protocol.export创建export对象和实际的监听URL和请求处理器，DubboProtocol请求处理器的远程通讯由dubbo-remoting模块实现，源码如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181205224027340.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181205224051833.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
其中invoker为2中，注册导出服务时创建的invoker。
 
### 服务消费者引用的服务Reference：ReferenceBean
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181205224116327.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
1. 在解析dubbo:reference时，会产生ReferenceBean的实例。ReferenceBean实现了spring的FactoryBean接口，故spring在注入bean时，在程序使用如@Autowired时，会调用getObject方法，获取对象实例，如果是singleton类，则放到spring单实例缓存池。
2. ReferenceBean继承于ReferenceConfig，getObject调用ReferenceConfig的get方法，其中get为使用synchronized修饰，保证只初始化一次：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181205224141591.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
init方法底层实现核心源码如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181205224204857.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
其中map为该对象实例的属性和属性值，通过createProxy创建消费者调用的代理对象。createProxy的核心源码实现如下：
（1）refprotocol：根据URL的protocol获取到dubbo-rpc模块的protocol实现，调用refprotocol.refer获取对应的消费者端的调用；通过这个代理来对服务提供者或者说服务端发起RPC请求；
（2）cluster.join：如果包含多个注册中心，则调用cluster.join进行负载均衡获取一个调用；
（3）proxyFactory.getProxy(invoker)：获取最终的消费者调用代理，代理提供了一个模板，最终的调用逻辑是定义在对应的Invoker的doInvoke方法，如DubboInvoker，在doInvoker中再调用dubbo-remoting模块的client进行远程通讯。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181205224231303.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)

### 创建spring bean实例
1. 通过context-param的方式将dubbo引入到宿主应用中
web-fragment.xml文件：相当于web.xml的一个小片段，通过合并这个文件到web.xml，从而在宿主应用启动时，初始化DubboApplicationContextInitializer。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181205224258168.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)

2. 监听spring框架启动：在DubboApplicationContextInitializer初始化时，将DubboApplicationListener作为监听器，监听宿主应用的启动。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181205224320292.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
DubboApplicationListener监听宿主应用的启动和关闭，调用DubboBootstrap进行dubbo框架的启动和关闭。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181205224342713.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)

