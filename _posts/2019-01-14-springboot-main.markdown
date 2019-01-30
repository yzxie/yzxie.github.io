---
layout:     post
title:      "SpringBoot学习（二）：为什么main方法启动类SpringApplication需要在项目根目录"
subtitle:   "SpringApplication lies in root directory"
date:       2019-01-14 20:49:00
author:     "XYZ"
header-img: "img/article-bg1.jpg"
tags:
    - SpringBoot
---
* 在基于SpringBoot的应用应用中，通常需要将包含main方法的启动类（即在main方法中通过执行SpringApplication.run方法来启动应用）放在项目的根目录，即与所有包平级。原因主要是启动类自身是一个基于注解的配置类，一般使用@SpringBootApplication注解，而这个注解由三个注解组成，分别是：@SpringBootConfiguration，@ComonentScan，@EnableAutoConfiguration。
   1. @SpringBootConfiguration：继承于@Configuration，本身只是说明这是一个SpringBoot项目的配置；
   2. @ComponentScan注解：用于进行包扫描获取需要注册到Spring容器的Bean；
   3. @EnableAutoConfiguration注解：扫描项目的所有包，检测项目中是否存在与SpringBoot自动添加的功能组件，实现了相同的接口或者继承了相同的父类，有则使用项目自身提供的该功能组件，没有则使用SpringBoot自动添加的该功能组件。SpringBoot自动添加的这些功能组件对应的类通常是使用了@Configuration注解和@Conditional注解。
 
* @ComponentScan注解有个特性就是：如果不指定需要扫描的包或者需要注册的类，则会扫描该使用@ComponentScan注解的类所在的包以及子包，所以放在项目根目录，则会扫描项目的所有包。

* 除了@ComponentScan注解之外，@EnableAutoConfiguration也是扫描使用了这个注解的类所在的包及其子包，故放在项目根目录，则可以扫描项目所有的包，对所有的类（具体为使用Spring容器管理的）进行检测。