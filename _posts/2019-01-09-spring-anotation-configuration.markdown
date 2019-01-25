---
layout:     post
title:      "基于注解的配置@Configuration详解"
subtitle:   "Spring Anotation @Configuration"
date:       2019-01-09 20:49:00
author:     "XYZ"
header-img: "img/article-bg1.jpg"
tags:
    - Spring源码分析
---
* 使用@Configuration来注解类，在类里面包含多个@Bean注解的方法。这些使用@Bean注解的方法，会被加载为BeanFactory里面的BeanDefinition，其中beanName默认为方法名，并且默认会创建对应的bean对象实例。其实@Configuration注解的类，就相当于一个xml配置文件的beans标签。

* 使@Configuration注解的类生效方式，即被spring容器ApplicationContext感知并加载。
   1. 使用AnnotationConfigApplicationContext，在refresh之前，通过AnnotationConfigApplicationContext的register方法注册这个类，如下：
   
        ```java
         AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();
         ctx.register(AppConfig.class);
         ctx.refresh();
         
         MyBean myBean = ctx.getBean(MyBean.class);
         // use myBean ...
        ```
        在web项目中也可以在web.xml中的contextClass中指定AnnotationConfigWebApplicationContext：
        
        ```xml
        <web-app>
            <context-param>
                <param-name>contextClass</param-name>
                <param-value>
                    org.springframework.web.context.
                    support.AnnotationConfigWebApplicationContext
                </param-value>
            </context-param>
            
            ...
            
            <servlet>
                <servlet-name>sampleServlet</servlet-name>
                <servlet-class>
                    org.springframework.web.servlet.DispatcherServlet
                </servlet-class>
                <init-param>
                    <param-name>contextClass</param-name>
                    <param-value>
                        org.springframework.web.context.
                        support.AnnotationConfigWebApplicationContext
                    </param-value>
                </init-param>
            </servlet>
            
            ...
            
        </web-app>
        ```


   2. 基于Spring的context:annotation-config
   
        ```xml
        <beans>
           <context:annotation-config/>
           <bean class="com.acme.AppConfig"/>
        </beans>
        ```

   3. 基于Spring的componentScan
        
        ```xml
        <context:component-scan/>
        ```
        
    
* @Configuration注解可与以下注解一起使用：
   1. @ComponentScan：指定需要扫描的包，同时启动自身

       ```java
        @Configuration
        @ComponentScan("com.acme.app.services")
        public class AppConfig {
           // various @Bean definitions ...
        }
        ```
   2. @PropertySource：为Environment提供propertySource，即指定的属性源的属性键值对也会加载到Environment中，其中Environment主要用来存放类路径下的相关属性文件，如properties文件，的内容，是spring容器启动时，最先加载的，即在加载bean之前加载，在创建beanDefinition或bean实例对象时就可以直接使用了，如下面的在myBean里面可以直接访问，如下：
   
        ```java
        @Configuration
        @PropertySource("classpath:/com/acme/app.properties")
        public class AppConfig {
            @Inject Environment env;
         
           @Bean
           public MyBean myBean() {
                  return new MyBean(env.getProperty("bean.name"));
            }
        }
        ```
    3. @Import：合并其他Configuration，跟xml的import标签差不多
    
        ```java
         @Configuration
         public class DatabaseConfig {
         
             @Bean
             public DataSource dataSource() {
                 // instantiate, configure and return DataSource
             }
         }
         
         @Configuration
         @Import(DatabaseConfig.class)
         public class AppConfig {
         
             private final DatabaseConfig dataConfig;
        
             public AppConfig(DatabaseConfig dataConfig) {
                 this.dataConfig = dataConfig;
             }
        
             @Bean
             public MyBean myBean() {
                 // reference the dataSource() bean method
                 return new MyBean(dataConfig.dataSource());
             }
        }
        ```

    4. @Value：即将properties的属性填充进来，通常结合<context:property-placeholder/>来激活。
    
        ```java
        @Configuration
        @PropertySource("classpath:/com/acme/app.properties")
        public class AppConfig {
            @Value("${bean.name}") String beanName;
         
            @Bean
            public MyBean myBean() {
                return new MyBean(beanName);  
            }
        }        
        ```
    5. @Profile：既可以在@Configuration级别设置profile，也可以在@Bean级别设置。作用是指定profile激活时，这些配置才生效：
    
        ```java
         @Profile("development")
         @Configuration
         public class EmbeddedDatabaseConfig {
         
             @Bean
             public DataSource dataSource() {
                 // instantiate, configure and return embedded DataSource
             }
         }
         
         @Profile("production")
         @Configuration
         public class ProductionDatabaseConfig {
         
             @Bean
             public DataSource dataSource() {
                 // instantiate, configure and return production DataSource
             }
         }
         
         
         @Configuration
         public class ProfileDatabaseConfig {
         
             @Bean("dataSource")
             @Profile("development")
             public DataSource embeddedDatabase() { ... }
         
             @Bean("dataSource")
             @Profile("production")
             public DataSource productionDatabase() { ... }
         }
        ```
    6. @ImportResource：合并xml文件的配置信息进来，与@Import功能类似，只是@Import是合并配置类。
    
        ```java
        @Configuration
        @ImportResource("classpath:/com/acme/database-config.xml")
         public class AppConfig {
         
             @Inject DataSource dataSource; // from XML
         
             @Bean
             public MyBean myBean() {
                 // inject the XML-defined dataSource bean
                 return new MyBean(this.dataSource);
             }
         }
        ```
        
    7. 嵌套@Configuration：即在类内部使用@Configuration注解来合并其他配置类，功能和@Import一样，只是可以在一个类里面配置多个配置类。这个用法通常与@Profile结合使用，这样可以做到每个profile对应一个配置类。
    
        ```java
        @Configuration
        @Profile("production")
        public class AppConfig {
         
            @Inject DataSource dataSource;
         
            @Bean
            public MyBean myBean() {
                return new MyBean(dataSource);
            }
        
            // 注意方法需要是static的
            
            @Configuration
            static class DatabaseConfig {
                @Bean
                DataSource dataSource() {
                    return new EmbeddedDatabaseBuilder().build();
                }
            }
        }
        ```
    
    8. @Lazy：懒加载支持，类里面的@Bean注解的方法默认是启动就加载创建bean对象实例的，可以使用该注解在@Configuration或者@Bean上，来实现类里面的所有bean或者指定的bean懒加载。
    
    9. @ContextConfiguration：测试支持，即在spring-test包的测试工具中使用，用来加载指定的配置类到当前正在测试的ApplicationContext：
    
        ```java
        @RunWith(SpringRunner.class)
        @ContextConfiguration(classes = {AppConfig.class, DatabaseConfig.class})
        public class MyTests {
        
            @Autowired MyBean myBean;
         
            @Autowired DataSource dataSource;
         
            @Test
            public void test() {
                // assertions against myBean ...
            }
        }
        ```
        
    10. 与@Enable为前戳的系列注解同时使用，开启spring框架的其他功能，如下：
    
        @EnableAsync： 方法异步执行支持
        
        @EnableScheduling：定时任务执行支持
        
        @EnableTransactionManagement：注解驱动的事务管理器支持
        
        @EnableWebMvc：启动spring-mvc功能
    
* @Configuration注解的类，需要以类的形式提供，而不是是实例对象。
   1. 类不能是final
   2. 类不能是内部类
   3. 嵌套的@Configuration定义，对应的方法需要是static的。