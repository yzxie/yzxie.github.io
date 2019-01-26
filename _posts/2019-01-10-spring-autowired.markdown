---
layout:     post
title:      "Spring实现依赖注入的三个注解：@Autowired，@Resource，@Inject"
subtitle:   "Spring Bean Inject @Autowired, @Resource, @Inject"
date:       2019-01-10 10:49:00
author:     "XYZ"
header-img: "img/article-bg1.jpg"
tags:
    - Spring源码分析
---

#### 注入实现方式
* @Autowired主要可以在set方法，field，构造函数中完成bean注入，注入方式为byType的，如果存在多个同一类型的bean，则使用@Qualifier来指定注入哪个beanName的bean。
* 与JDK的@Resource的区别：@Resource是基于beanName来查找bean注入的，而@Autowried是基于类型来查找bean注入的。
* 与JDK的@Inject的区别：@Inject也是基于类型来查找bean注入的，如果需要指定名称，则可以结合使用@Named注解。

#### 实现原理
* 注解实现注入主要是通过BeanPostProcessor来生效的。BeanPostProcessor是在spring容器启动，创建bean对象实例后，马上执行的。
* @Autowired是通过AutowiredAnnotationBeanPostProcessor来实现注入；
* @Resource和@Inject是通过CommonAnnotationBeanPostProcessor来实现的，其中CommonAnnotationBeanPostProcessor是spring中统一处理JDK中定义的注解的一个BeanPostProcessor。其他注解包含@PostConstruct，@PreDestroy等。
* AutowiredAnnotationBeanPostProcessor和CommonAnnotationBeanPostProcessor添加到spring容器的BeanPostProcessor的条件，即激活这些处理器的条件：在对应的spring容器的配置xml文件中，如applicationContext.xml，添加<context:annotation-config />和<context:component-scan />，或者只使用<context:component-scan />。
* 两者的区别是<context:annotation-config />只查找并激活已经存在的bean，如通过xml文件的bean标签生成加载到spring容器的，而不会去扫描如@Controller等注解的bean，查找到之后进行注入；而<context:component-scan />除了具有<context:annotation-config />的功能之外，还会去加载通过basePackages属性指定的包下面的，默认为扫描@Controller，@Service，@Component，@Repository注解的类。不指定basePackages则是类路径下面，或者如果使用注解@ComponentScan方式，则是当前类所在包及其子包下面。详细可参看[：Spring 开启Annotation context:annotation-config 和 context:component-scan诠释及区别](https://www.cnblogs.com/leiOOlei/p/3713989.html)
* 基于注解的注入先于基于XML的注入，后者会覆盖前者的注入。

#### 总结
1. @Autowired是Spring自带的，@Inject和@Resource都是JDK提供的，其中@Inject是JSR330规范实现的，@Resource是JSR250规范实现的，而Spring通过BeanPostProcessor来提供对JDK规范的支持。
2. @Autowired、@Inject用法基本一样，不同之处为@Autowired有一个required属性，表示该注入是否是必须的，即如果为必须的，则如果找不到对应的bean，就无法注入，无法创建当前bean。
3. @Autowired、@Inject是默认按照类型匹配的，@Resource是按照名称匹配的。如在spring-boot-data项目中自动生成的redisTemplate的bean，是需要通过byName来注入的。如果需要注入该默认的，则需要使用@Resource来注入，而不是@Autowired。可参看[：SpringBoot中注入RedisTemplate实例异常解决](https://blog.csdn.net/zhaoheng314/article/details/81564166)
4. 对于@Autowire和@Inject，如果同一类型存在多个bean实例，则需要指定注入的beanName。@Autowired和@Qualifier一起使用，@Inject和@Name一起使用。