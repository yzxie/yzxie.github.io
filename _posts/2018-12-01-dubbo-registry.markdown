## Registry
每个注册中心对应一个Registry实例，包括dubbo，zookeeper，redis, multicast。
（1）Set<URL>类型的registed：记录provider注册过的service url。
（2）ConcurrentMap<URL, Set<NotifyListener>> subscribed：consumer订阅URL，URL有变化时的监听器NotifyListener；其中多个NotifyListener是因为consumer存在多个地方调用这个service，NotifyListener的实现为RegistryDirectory。
（3）ConcurrentMap<URL, Map<String, List<URL>>> notified：
（4）file与properties：consumer将从注册中心获取的provider urls保存到本地file，同时加载到properties，避免注册中心挂了，此时consumer还可以连接provider。

## RegistryFactory
工厂类，负责生成Registry实例，维护Registry实例缓存，根据底层实现技术的不同，分为dubbo，zookeeper，redis，multicast实现。

## RegistryProtocol：注册协议
1. export方法：负责将provider的service注册到注册中心，具体调用了在ServiceConfig中通过ExtensionLoader加载RegistryProtocol。
2. 注册中心实现选择：包括：zookeeper，redis，dubbo。确定RegistoryFactory的实现：一旦RegistoryFactory实现确定，则Registry就确定了。
* 根据ExtensionLoader的分析，通过ServiceConfig确定采用的是RegistryProtocol，通过RegistryProtocol的export方法注册service到注册中心。
* RegistryProtocol通过ExtensionLoader确定RegistryFactory的实现类。
3. 具体过程
（1）在RegistryProtocol的export方法调用了getReigstry(originInvoker)：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181201135103581.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
如下：右下角为originInvoker的url的protocol和相关parameters，registry -> zookeeper，所以通过setProtocol设置值为zookeeper，故最终选择的是ZookeeperRegistoryFactory，具体看（2）中RegistryFactory的SPI和@Adaptive规则。
![在这里插入图片描述](https://img-blog.csdnimg.cn/2018120113560928.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
getRegistry的实现：模板设计模式，最终由具体实现类实现createRegistry。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181201135637768.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
ZookeeperRegistry的定义：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181201135710599.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)

（2）RegistryProtocol中registryFactory实现的确定过程
在获取registryProtocol实例时，调用的getAdaptiveExtension的实现，然后底层继续调用createAdaptiveExtension方法。createAdaptiveExtension再调用injectExtension方法，给通过getAdaptiveExtensionClass().newInstance()获取到的RegistryProtocol实例，注入属性值，这个功能类似于spring ioc的实现，RegistryProtocol实例包含setRegistoryFactory方法：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181201135808462.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181201140009832.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
如上图：其中pt为RegistroyFactory，property为registoryFactory，通过objectFactory.getExtension(pt, property）获取到registoryFactory实现类的实例。objectFactory为SpiExtensionFactory的实例，底层调用getExtension：其中type为RegistryFactory。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181201140206501.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
如下图：RegistoryFactory接口的@SPI注解，value默认为dubbo，表示默认为DubboRegistryFactory
方法getRegistry为使用@Adaptive({"protocol"})，protocol表示调用这个方法的时候，根据URL的protocol对应的值或决定使用哪个RegistryFactory接口实现，由（1）分析protocol为zookeeper。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181201140324732.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
继续debug源码，查找loader.getAdaptiveExtension()最终获取到哪个RegistryFactory实现。最终进入ExtensionLoader的createAdaptiveExtensionClassCode方法，生成的代码如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181201140448560.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAwMTM1NzM=,size_16,color_FFFFFF,t_70)
即：首先根据调用方法getRegistry(URL url)，url的protocol选择RegistryFactory，如果protocol为null，则使用defaultExtName，默认为dubbo。而由（1）分析可知：url的protocol为zookeeper，故最终调用的为ZookeeperRegistryFactory。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181201140527164.png)
动态生成的代码如下：
```
package com.alibaba.dubbo.registry;
import com.alibaba.dubbo.common.extension.ExtensionLoader;
public class RegistryFactory$Adpative implements com.alibaba.dubbo.registry.RegistryFactory {
	public com.alibaba.dubbo.registry.Registry getRegistry(com.alibaba.dubbo.common.URL arg0) {
		if (arg0 == null) throw new IllegalArgumentException("url == null");
		com.alibaba.dubbo.common.URL url = arg0;
		String extName = ( url.getProtocol() == null ? "dubbo" : url.getProtocol() );
		if(extName == null) 
			throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.registry.RegistryFactory) name from url(" + url.toString() + ") use keys([protocol])");
		com.alibaba.dubbo.registry.RegistryFactory extension = (com.alibaba.dubbo.registry.RegistryFactory)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.registry.RegistryFactory.class).getExtension(extName);
		return extension.getRegistry(arg0);
	}
}
```

## RegistryDirectory：消费者订阅监听器