---
layout:     post
title:      "Tomcat源码分析（一）：整体架构设计"
subtitle:   "Tomcat Architecture"
date:       2019-01-01 08:12:00
author:     "XYZ"
header-img: "img/article-bg1.jpg"
tags:
    - Tomcat源码分析
---
### 源码结构与核心接口设计
#### 架构图
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190106161944467.gif)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190106172207428.jpeg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
#### catalina包
Tomcat框架作为servlet容器的实现，主要以Container接口来定义。
* core包：servlet容器的分层实现，容器从上到下为：Engine，Host，Context，Wrapper。其中核心容器为Context和Wrapper，Context代表一个应用，对应一个ServletContext。Wrapper是对应用的Servlet的封装，进行实际请求处理。
* connector包：Tomcat框架的Connector连接器的定义，负责连接Servlet容器，将底层传上来的，网络协议相关的数据封装成符合servlet规范的数据，然后传递到servlet容器进行处理。
* startup包：Tomcat框架以脚本形式启动和关闭的实现。主要是通过Bootstrap类的main，接收启动脚本startup.sh和停止脚本shutdown.sh的调用，根据脚本参数进行进行Tomcat的启动和关闭。Bootstrap类，调用Catalina类，进行Tomcat框架Server类的启动，然后根据LifeCycle架构设计，依次启动Server -> Service -> Connecotrs，Engine。
其中Engine中和Connector中再依次启动其Child：
Engine -> Host -> Context -> Wrapper；
Connector -> Endpoint -> Socket。

#### coyote包
Tomcat的网络协议实现，主要提供http1.1，http2.0，apr协议的实现。核心接口为ProtocolHandler。

#### tomcat.util.net包
Tomcat的底层Socket IO处理的实现，以及整体框架的线程模型的实现，核心基类为AbstractEndpoint。

#### AbstractEndpoint：底层socket IO处理和线程模型设计
* 建立连接请求处理：acceptor，在accept方法中阻塞等待客户端连接请求到来，然后注册到其中一个poller线程中。
* 事件轮询线程池：poller，每个从acceptor接收并创建的连接socket都注册到一个poller线程的selector中，每个poller线程包含一个selector。线程个数一般是2和处理器个数两者之间的最小值。
* 工作线程池：executor，线程个数默认为200，可以通过server.xml的Connector的maxThreads属性来修改。作用是进行实际请求处理，针对每个请求，所做的工作包括：从Socket中获取底层数据，然后交给协议实现ProtocolHandler解析成与协议格式，如http协议，相关的数据。再通过适配器CoyoteAdapter转为servelt规范的数据并交给catalina容器。

#### ProtocolHandler：应用层协议实现
* 负责将底层Socket接收到的字节数据，解析成对应应用层协议的数据，目前提供了http1.1，http2.0，apr的协议实现。

#### CoyoteAdapter：应用层协议数据和servlet规范数据的适配器，也是Connector连接Container的实现
* 负责将ProtocolHandler解析出的针对特定应用层协议的数据request和reponse，转为符合servlet规范的request和reponse，即HttpServletRequest和HttpServletResponse。从而再进一步交给Catalina容器处理。

### 请求处理流程：从上到下
1. Connector包含所在的Service的引用，通过Service可以获取该Service所包含的Engine，从而可以建立Connector和Engine，或者说跟Catalina容器的联系；
2. Connector包含一个ProtocolHandler和一个CoyteAdapter，并将CoyteAdapter引用传给PrototcolHandler，从而PrototcolHandler可以通过CoyteAdapter将数据往上传给Connector；
3. PrototcolHandler通过AbstractEndpoint来处理底层Socket的IO，并生成应用层协议相关的request和response，然后传给CoyteAdapter；
4. CoyteAdapter将应用层相关的request和response，转为sevlet规范的request和response，然后通过Connector传给Catalina容器。

### 总结
1. Container，即servelt容器，主要负责对多应用的部署的支持，即采用分层架构来实现。针对各个应用的请求，调用对应应用的servlet进行处理。
1. CoyteAdapter承担了建立应用协议控制器ProtocolHandler和servlet容器连接器Connector之间的联系；Connector承担了建立与Catalina容器的联系，从而最终完成从底层socket，即TCP层的数据，到生成应用层协议数据，再到servlet容器的数据传输，最终在servlet容器进行实际的请求处理。
2. Endpoint进行底层Socket连接、读写IO的处理，并负责整体框架的线程模型设计。针对一个请求，从工作线程池中获取一个线程，在该线程中执行以上请求处理。而相关设置参数，如线程池大小等，则是通过ProtocolHandler在启动时，获取相关配置参数，传给底层的Endpoint对象。