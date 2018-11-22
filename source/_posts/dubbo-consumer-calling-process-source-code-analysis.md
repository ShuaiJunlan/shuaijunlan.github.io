---
title: Dubbo消费者调用过程源码分析
date: 2018-08-05 14:11:36
tags:
    - dubbo
---

在分析Dubbo RPC服务调用过程之前，我们先写一个基于Dubbo实现的[**Consumer-Provider的Demo**](https://github.com/shuaijunlan/Spring-Learning/tree/master/dubbo)，通过这个Demo来分析具体的RPC调用栈。

先定义一个接口：

```java
/**
 * @author Junlan Shuai[shuaijunlan@gmail.com].
 * @date Created on 11:02 AM 2018/07/19.
 */
public interface ITestService {
    String sayHello(String msg);
}
```

我们基于zookeeper注册中心，服务端配置如下：

```xml
<dubbo:application name="dubbo-server" owner="Junlan" />
<dubbo:registry address="zookeeper://127.0.0.1:2181"/>
<!--protocal configuration-->
<dubbo:protocol name="dubbo" port="20881"/>
<dubbo:provider server="netty"/>
<!--service configuration-->
<dubbo:service interface="cn.shuaijunlan.dubbo.learning.service.ITestService" ref="testService" protocol="dubbo" loadbalance="roundrobin"/>
<bean class="cn.shuaijunlan.dubbo.learning.service.impl.TestServiceImpl" name="testService" />
```

客户端配置如下：

```xml
<dubbo:application name="dubbo-client" owner="Junlan"/>
<dubbo:consumer client="netty"/>
<dubbo:registry address="zookeeper://127.0.0.1:2181"/>
<dubbo:reference interface="cn.shuaijunlan.dubbo.learning.service.ITestService" id="testService" check="false"/>
```

<!-- more -->

项目的部分依赖如下：

```xml
<dependency>
    <groupId>org.apache.dubbo</groupId>
    <artifactId>dubbo</artifactId>
    <version>2.7.0-SNAPSHOT</version>
</dependency>
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-framework</artifactId>
    <version>2.12.0</version>
</dependency>
<dependency>
    <groupId>org.apache.zookeeper</groupId>
    <artifactId>zookeeper</artifactId>
    <version>3.4.9</version>
</dependency>
```

具体的服务提供者实现参照[**service-provider-a**](https://github.com/shuaijunlan/Spring-Learning/tree/master/dubbo/service-provider-a) ，服务消费者实现参照[**dubbo-client-demo**](https://github.com/shuaijunlan/Spring-Learning/tree/master/dubbo/dubbo-client-demo)。

先启动服务提供者服务，下面分析在哪打断点：

我们看下`org.apache.dubbo.remoting.transport.netty4.NettyChannel`的`send`方法：

```java
@Override
public void send(Object message, boolean sent) throws RemotingException {
    super.send(message, sent);

    boolean success = true;
    int timeout = 0;
    try {
        //向远程服务发送消息，因此我们在这句打个断点
        ChannelFuture future = channel.writeAndFlush(message);
        if (sent) {
            timeout = getUrl().getPositiveParameter(Constants.TIMEOUT_KEY, Constants.DEFAULT_TIMEOUT);
            success = future.await(timeout);
        }
        Throwable cause = future.cause();
        if (cause != null) {
            throw cause;
        }
    } catch (Throwable e) {
        throw new RemotingException(this, "Failed to send message " + message + " to " + getRemoteAddress() + ", cause: " + e.getMessage(), e);
    }

    if (!success) {
        throw new RemotingException(this, "Failed to send message " + message + " to " + getRemoteAddress()
                                    + "in timeout(" + timeout + "ms) limit");
    }
}
```

我们在`ChannelFuture future = channel.writeAndFlush(message);`这句打个断点，Debug服务消费者，得到如下线程栈信息：

```
"main@1" prio=5 tid=0x1 nid=NA runnable
  java.lang.Thread.State: RUNNABLE
	  at org.apache.dubbo.remoting.transport.netty4.NettyChannel.send(NettyChannel.java:101)
	  at org.apache.dubbo.remoting.transport.AbstractClient.send(AbstractClient.java:265)
	  at org.apache.dubbo.remoting.transport.AbstractPeer.send(AbstractPeer.java:53)
	  at org.apache.dubbo.remoting.exchange.support.header.HeaderExchangeChannel.request(HeaderExchangeChannel.java:116)
	  at org.apache.dubbo.remoting.exchange.support.header.HeaderExchangeClient.request(HeaderExchangeClient.java:90)
	  at org.apache.dubbo.rpc.protocol.dubbo.ReferenceCountExchangeClient.request(ReferenceCountExchangeClient.java:83)
	  at org.apache.dubbo.rpc.protocol.dubbo.DubboInvoker.doInvoke(DubboInvoker.java:108)
	  at org.apache.dubbo.rpc.protocol.AbstractInvoker.invoke(AbstractInvoker.java:154)
	  at org.apache.dubbo.rpc.listener.ListenerInvokerWrapper.invoke(ListenerInvokerWrapper.java:77)
	  at org.apache.dubbo.monitor.support.MonitorFilter.invoke(MonitorFilter.java:75)
	  at org.apache.dubbo.rpc.protocol.ProtocolFilterWrapper$1.invoke(ProtocolFilterWrapper.java:72)
	  at org.apache.dubbo.rpc.protocol.dubbo.filter.FutureFilter.invoke(FutureFilter.java:47)
	  at org.apache.dubbo.rpc.protocol.ProtocolFilterWrapper$1.invoke(ProtocolFilterWrapper.java:72)
	  at org.apache.dubbo.rpc.filter.ConsumerContextFilter.invoke(ConsumerContextFilter.java:50)
	  at org.apache.dubbo.rpc.protocol.ProtocolFilterWrapper$1.invoke(ProtocolFilterWrapper.java:72)
	  at org.apache.dubbo.rpc.protocol.InvokerWrapper.invoke(InvokerWrapper.java:56)
	  at org.apache.dubbo.rpc.cluster.support.FailoverClusterInvoker.doInvoke(FailoverClusterInvoker.java:78)
	  at org.apache.dubbo.rpc.cluster.support.AbstractClusterInvoker.invoke(AbstractClusterInvoker.java:243)
	  at org.apache.dubbo.rpc.cluster.support.wrapper.MockClusterInvoker.invoke(MockClusterInvoker.java:75)
	  at org.apache.dubbo.rpc.proxy.InvokerInvocationHandler.invoke(InvokerInvocationHandler.java:70)
	  at org.apache.dubbo.common.bytecode.proxy0.sayHello(proxy0.java:-1)
	  at cn.shuaijunlan.dubbo.learning.main.Main.main(Main.java:16)
```

自底向上，可以直观的看到服务消费者调用要经过的类和方法，下面将进行一步步分析，对每一个类的创建过程和调用过程做出解析：

* 第一行栈信息

```
at cn.shuaijunlan.dubbo.learning.main.Main.main(Main.java:16)
```

Main.java 类是消费者端的启动类，可以忽略。

* 第二行栈信息

```
at org.apache.dubbo.common.bytecode.proxy0.sayHello(proxy0.java:-1)
```

`org.apache.dubbo.common.bytecode.proxy0`类是一个代理类，它代理了所有RPC服务接口的方法调用。这个类实例是什么时候创建的？类代码是怎样的？

**Dubbo基于Spring的构建分析**（参考文章[**《基于Spring构建Dubbo源码分析》**](https://shuaijunlan.github.io/2018/08/13/dubbo-basing-on-spring-framework-analysis/)）， 代理类的创建是由ReferenceBean类的

```java
public Object getObject() throws Exception {
    return get();
}
```

方法里触发的，具体的实现在ReferenceConfig类createProxy方法里

```java
@SuppressWarnings({"unchecked", "rawtypes", "deprecation"})
private T createProxy(Map<String, String> map) {
    // ...
    // 用于生成invoker的逻辑，关于invoker生成逻辑这里先忽略，后面会说到
    // ...
    // create service proxy
    return (T) proxyFactory.getProxy(invoker);
}
```

proxyFactory变量赋值为

```java
private static final ProxyFactory proxyFactory = ExtensionLoader.getExtensionLoader(ProxyFactory.class).getAdaptiveExtension();
```

**Dubbo SPI机制**(参考文章[**《Dubbo SPI机制源码分析》**](https://shuaijunlan.github.io/2018/08/09/dubbo-spi-analysis/))里可以得到ProxyFactory接口的Adaptive类的getProxy方法源码如下：

```java
package org.apache.dubbo.rpc;

import org.apache.dubbo.common.extension.ExtensionLoader;

public class ProxyFactory$Adaptive implements org.apache.dubbo.rpc.ProxyFactory {
    public java.lang.Object getProxy(org.apache.dubbo.rpc.Invoker arg0) throws org.apache.dubbo.rpc.RpcException {
        if (arg0 == null) throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument == null");
        if (arg0.getUrl() == null)
            throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument getUrl() == null");
        org.apache.dubbo.common.URL url = arg0.getUrl();
        String extName = url.getParameter("proxy", "javassist");
        if (extName == null)
            throw new IllegalStateException("Fail to get extension(org.apache.dubbo.rpc.ProxyFactory) name from url(" + url.toString() + ") use keys([proxy])");
        org.apache.dubbo.rpc.ProxyFactory extension = (org.apache.dubbo.rpc.ProxyFactory) ExtensionLoader.getExtensionLoader(org.apache.dubbo.rpc.ProxyFactory.class).getExtension(extName);
        return extension.getProxy(arg0);
    }

    public java.lang.Object getProxy(org.apache.dubbo.rpc.Invoker arg0, boolean arg1) throws org.apache.dubbo.rpc.RpcException {
        if (arg0 == null) throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument == null");
        if (arg0.getUrl() == null)
            throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument getUrl() == null");
        org.apache.dubbo.common.URL url = arg0.getUrl();
        String extName = url.getParameter("proxy", "javassist");
        if (extName == null)
            throw new IllegalStateException("Fail to get extension(org.apache.dubbo.rpc.ProxyFactory) name from url(" + url.toString() + ") use keys([proxy])");
        org.apache.dubbo.rpc.ProxyFactory extension = (org.apache.dubbo.rpc.ProxyFactory) ExtensionLoader.getExtensionLoader(org.apache.dubbo.rpc.ProxyFactory.class).getExtension(extName);
        return extension.getProxy(arg0, arg1);
    }

    public org.apache.dubbo.rpc.Invoker getInvoker(java.lang.Object arg0, java.lang.Class arg1, org.apache.dubbo.common.URL arg2) throws org.apache.dubbo.rpc.RpcException {
        if (arg2 == null) throw new IllegalArgumentException("url == null");
        org.apache.dubbo.common.URL url = arg2;
        String extName = url.getParameter("proxy", "javassist");
        if (extName == null)
            throw new IllegalStateException("Fail to get extension(org.apache.dubbo.rpc.ProxyFactory) name from url(" + url.toString() + ") use keys([proxy])");
        org.apache.dubbo.rpc.ProxyFactory extension = (org.apache.dubbo.rpc.ProxyFactory) ExtensionLoader.getExtensionLoader(org.apache.dubbo.rpc.ProxyFactory.class).getExtension(extName);
        return extension.getInvoker(arg0, arg1, arg2);
    }
}
```

在如上`ProxyFactory$Adaptive`类中，调用`getProxy(org.apache.dubbo.rpc.Invoker arg0) `方法，其中：

```java
String extName = url.getParameter("proxy", "javassist");
org.apache.dubbo.rpc.ProxyFactory extension = (org.apache.dubbo.rpc.ProxyFactory) ExtensionLoader.getExtensionLoader(org.apache.dubbo.rpc.ProxyFactory.class).getExtension(extName);
```

默认获取ProxyFactory接口的javassist扩展类JavassistProxyFactory，先调用`JavassitProxyFactory`的父类ProxyFactory的`getProxy(Invoker<T> invoker)`方法和`getProxy(Invoker<T> invoker, boolean generic)`方法：

```java
@Override
public <T> T getProxy(Invoker<T> invoker) throws RpcException {
    return getProxy(invoker, false);
}
```

再调用`JavassistProxyFactory`类的`getProxy(Invoker<T> invoker, Class<?>[] interfaces)`方法：

```java
@Override
@SuppressWarnings("unchecked")
public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
    return (T) Proxy.getProxy(interfaces).newInstance(new InvokerInvocationHandler(invoker));
}
```

再到**生成代理类的Proxy类**（具体过程在另一篇文章中详细分析）

```java
/**
     * Get proxy.
     *
     * @param ics interface class array.
     * @return Proxy instance.
     */
public static Proxy getProxy(Class<?>... ics) {
    return getProxy(ClassHelper.getClassLoader(Proxy.class), ics);
}
```

这里直接贴出**通过代码hack生成的代理类**源码，这里动态生成了两个类：

```java
package com.alibaba.dubbo.common.bytecode;

import com.alibaba.dubbo.common.bytecode.ClassGenerator.DC;

import java.lang.reflect.InvocationHandler;

public class Proxy0 extends Proxy implements DC {
    public Object newInstance(InvocationHandler var1) {
        return new proxy01(var1);
    }

    public Proxy0_my() {
    }
}
```

这个类继承抽象类Proxy，实现了它的抽象方法newInstance，**接口DC是Dubbo内部作为动态类标示的接口**；

还有一个proxy01，就是在开始方法栈里看到的代理类，源码如下：

```java
package com.alibaba.dubbo.common.bytecode;

import com.alibaba.dubbo.rpc.service.EchoService;
import demo.dubbo.api.DemoService;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;

public class proxy01 implements ClassGenerator.DC, EchoService, DemoService {
    public static Method[] methods;
    private InvocationHandler handler;
    //实现了接口方法
    public String sayHello(String var1) {
        Object[] var2 = new Object[]{var1};
        Object var3 = null;
        try {
            var3 = this.handler.invoke(this, methods[1], var2);
        } catch (Throwable throwable) {
            throwable.printStackTrace();
        }
        return (String)var3;
    }

    public Object $echo(Object var1) {
        Object[] var2 = new Object[]{var1};
        Object var3 = null;
        try {
            var3 = this.handler.invoke(this, methods[3], var2);
        } catch (Throwable throwable) {
            throwable.printStackTrace();
        }
        return (Object)var3;
    }

    public proxy01() {
    }
    //public 构造函数，这里handler是
    //由Proxy.getProxy(interfaces).newInstance(new InvokerInvocationHandler(invoker))语句传入的InvokerInvocationHandler对象
    public proxy01(InvocationHandler var1) {
        this.handler = var1;
    }
}
```

可以看到代理类实现类三个接口，`ClassGeneratr.DC`是Dubbo动态类标识接口，`DemoService`是实际的业务接口，这样代理就可以调用服务方法了，`EchoService`是回显测试接口，只有一个`$echo(Object var1)`法。

```java
package org.apache.dubbo.rpc.service;

/**
 * Echo service.
 * @export
 */
public interface EchoService {

    /**
     * echo test.
     *
     * @param message message.
     * @return message.
     */
    Object $echo(Object message);

}
```

它能为所有的Dubbo RPC服务加上一个回显测试方法。

```java
// 通过类型强制转换为EchoService，可以测试
EchoService echoService = (EchoService) service;
System.out.println(echoService.$echo("hello"));
```

到这可以了解代理类生成的整个过程，可以看到sayHello方法的调用其实是调用`this.handler.invoke(this, methods[1], var2);`，因此这也解释了**线程栈中第三行信息**：

```
at org.apache.dubbo.rpc.proxy.InvokerInvocationHandler.invoke(InvokerInvocationHandler.java:70)
```

再往下看`org.apache.dubbo.rpc.proxy.InvokerInvocationHandler`类：

```java
public class InvokerInvocationHandler implements InvocationHandler {

    private final Invoker<?> invoker;
	//通过构造函数传入invoker
    public InvokerInvocationHandler(Invoker<?> handler) {
        this.invoker = handler;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        String methodName = method.getName();
        Class<?>[] parameterTypes = method.getParameterTypes();
        // 如果是Object类方法
        if (method.getDeclaringClass() == Object.class) {
            //反射调用
            return method.invoke(invoker, args);
        }
        // 对toString、hashCode、equals三个方法做了处理
        if ("toString".equals(methodName) && parameterTypes.length == 0) {
            return invoker.toString();
        }
        if ("hashCode".equals(methodName) && parameterTypes.length == 0) {
            return invoker.hashCode();
        }
        if ("equals".equals(methodName) && parameterTypes.length == 1) {
            return invoker.equals(args[0]);
        }

        RpcInvocation invocation;
        if (RpcUtils.hasGeneratedFuture(method)) {
            Class<?> clazz = method.getDeclaringClass();
            String syncMethodName = methodName.substring(0, methodName.length() - Constants.ASYNC_SUFFIX.length());
            Method syncMethod = clazz.getMethod(syncMethodName, method.getParameterTypes());
            invocation = new RpcInvocation(syncMethod, args);
            invocation.setAttachment(Constants.FUTURE_GENERATED_KEY, "true");
            invocation.setAttachment(Constants.ASYNC_KEY, "true");
        } else {
            invocation = new RpcInvocation(method, args);
            if (RpcUtils.hasFutureReturnType(method)) {
                invocation.setAttachment(Constants.FUTURE_RETURNTYPE_KEY, "true");
                invocation.setAttachment(Constants.ASYNC_KEY, "true");
            }
        }
        return invoker.invoke(invocation).recreate();
    }


}
```

在这里的invoker对象，是通过InvokerInvocationHandler构造方法传入，而且InvokerInvocationHandler对象是由JavassistProxyFactory类的`getProxy(Invoker<T> invoker, Class<?>[] interfaces)`方法创建，回到调用`proxyFactory.getProxy(invoker);`的地方，即ReferenceConfig类的`createProxy(Map<String, String> map)`方法，以下部分是生成invoker的过程：

```java
if (isJvmRefer) {
    URL url = new URL(Constants.LOCAL_PROTOCOL, NetUtils.LOCALHOST, 0, interfaceClass.getName()).addParameters(map);
    invoker = refprotocol.refer(interfaceClass, url);
    if (logger.isInfoEnabled()) {
        logger.info("Using injvm service " + interfaceClass.getName());
    }
} else {
    if (url != null && url.length() > 0) { // user specified URL, could be peer-to-peer address, or register center's address.
        String[] us = Constants.SEMICOLON_SPLIT_PATTERN.split(url);
        if (us != null && us.length > 0) {
            for (String u : us) {
                URL url = URL.valueOf(u);
                if (url.getPath() == null || url.getPath().length() == 0) {
                    url = url.setPath(interfaceName);
                }
                if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
                    urls.add(url.addParameterAndEncoded(Constants.REFER_KEY, StringUtils.toQueryString(map)));
                } else {
                    urls.add(ClusterUtils.mergeUrl(url, map));
                }
            }
        }
    } else { // assemble URL from register center's configuration
        //从注册中心获取配置URL
        List<URL> us = loadRegistries(false);
        if (us != null && !us.isEmpty()) {
            for (URL u : us) {
                URL monitorUrl = loadMonitor(u);
                if (monitorUrl != null) {
                    map.put(Constants.MONITOR_KEY, URL.encode(monitorUrl.toFullString()));
                }
                urls.add(u.addParameterAndEncoded(Constants.REFER_KEY, StringUtils.toQueryString(map)));
            }
        }
        if (urls.isEmpty()) {
            throw new IllegalStateException("No such any registry to reference " + interfaceName + " on the consumer " + NetUtils.getLocalHost() + " use dubbo version " + Version.getVersion() + ", please config <dubbo:registry address=\"...\" /> to your spring config.");
        }
    }
    //只有一个直连地址或者一个注册中心配置地址
    if (urls.size() == 1) {
        //这里的urls.get(0)，可能是直连地址（默认为dubbo协议），也可能是register注册地址（zookeeper协议）
        //示例中使用了zookeeper注册中心，因此会执行这一步
        invoker = refprotocol.refer(interfaceClass, urls.get(0));
    } else {//多个直连地址或者多个注册中心，甚至是两者的结合，执行这一步
        List<Invoker<?>> invokers = new ArrayList<Invoker<?>>();
        URL registryURL = null;
        for (URL url : urls) {
            //创建invoker放入invokers
            invokers.add(refprotocol.refer(interfaceClass, url));
            if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
                registryURL = url; // use last registry url
            }
        }
        if (registryURL != null) { // registry url is available
            // use AvailableCluster only when register's cluster is available
            //这其中包括直连和注册中心混合或者都是注册中心两种情况，默认使用AvailableCluster集群策略
            URL u = registryURL.addParameter(Constants.CLUSTER_KEY, AvailableCluster.NAME);
            invoker = cluster.join(new StaticDirectory(u, invokers));
        } else { // not a registry url
            //多个直连的URL
            invoker = cluster.join(new StaticDirectory(invokers));
        }
    }
}
```

经过上面的分析，可以发现invoker是通过`refprotocol.refer(interfaceClass, urls.get(0));`、`cluster.join(new StaticDirectory(u, invokers));`和`cluster.join(new StaticDirectory(invokers));`三种构建语句其中之一生成的，这里是经过第一种方式生成invoker的，下面来分析第一种生成invoker的情况，根据SPI机制这里refprotocol对象是`Protocol$Adpative`实例，具体refer实现是：

```java
public org.apache.dubbo.rpc.Invoker refer(java.lang.Class arg0, org.apache.dubbo.common.URL arg1) throws org.apache.dubbo.rpc.RpcException {
    if (arg1 == null) throw new IllegalArgumentException("url == null");
    org.apache.dubbo.common.URL url = arg1;
    String extName = (url.getProtocol() == null ? "MEAT-INF.dubbo" : url.getProtocol());
    if (extName == null)
        throw new IllegalStateException("Fail to get extension(org.apache.MEAT-INF.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");
    org.apache.dubbo.rpc.Protocol extension = (org.apache.dubbo.rpc.Protocol) ExtensionLoader.getExtensionLoader(org.apache.dubbo.rpc.Protocol.class).getExtension(extName);
    return extension.refer(arg0, arg1);
}
```

示例中是通过注册中心，因此这里protocol是register，会走RegistryProtocol类的refer方法

```java
@Override
@SuppressWarnings("unchecked")
public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
    //通过register可以获取具体注册中心协议，这里是zookeeper，因此url的协议值被设置为zookeeper
    url = url.setProtocol(url.getParameter(Constants.REGISTRY_KEY, Constants.DEFAULT_REGISTRY)).removeParameter(Constants.REGISTRY_KEY);
    //获取zookeeper Registry实现，即ZookeeperRegistryFactory，并调用getRegistry方法实现
    //获取zookeeper类型的registry对象
    Registry registry = registryFactory.getRegistry(url);
    if (RegistryService.class.equals(type)) {
        return proxyFactory.getInvoker((T) registry, type, url);
    }

    // group="a,b" or group="*"
    Map<String, String> qs = StringUtils.parseQueryString(url.getParameterAndDecoded(Constants.REFER_KEY));
    String group = qs.get(Constants.GROUP_KEY);
    if (group != null && group.length() > 0) {
        if ((Constants.COMMA_SPLIT_PATTERN.split(group)).length > 1
            || "*".equals(group)) {
            return doRefer(getMergeableCluster(), registry, type, url);
        }
    }
    //根据Dubbo SPI机制，给setXxx方法对应的属性赋值为Xxx$Adaptive，这里就是Cluster$Adaptive
    return doRefer(cluster, registry, type, url);
}
private <T> Invoker<T> doRefer(Cluster cluster, Registry registry, Class<T> type, URL url) {
    //这里的RegistryDirectory和StaticDirectory相对应的，前者是动态从注册中心获取url目录对象，后者是静态指定url目录
    RegistryDirectory<T> directory = new RegistryDirectory<T>(type, url);
    directory.setRegistry(registry);
    directory.setProtocol(protocol);
    // all attributes of REFER_KEY
    Map<String, String> parameters = new HashMap<String, String>(directory.getUrl().getParameters());
    URL subscribeUrl = new URL(Constants.CONSUMER_PROTOCOL, parameters.remove(Constants.REGISTER_IP_KEY), 0, type.getName(), parameters);
    if (!Constants.ANY_VALUE.equals(url.getServiceInterface())
        && url.getParameter(Constants.REGISTER_KEY, true)) {
        registry.register(subscribeUrl.addParameters(Constants.CATEGORY_KEY, Constants.CONSUMERS_CATEGORY,
                                                     Constants.CHECK_KEY, String.valueOf(false)));
    }
    //订阅注册中心，可以获取服务提供方地址等信息
    directory.subscribe(subscribeUrl.addParameter(Constants.CATEGORY_KEY,
                                                  Constants.PROVIDERS_CATEGORY
                                                  + "," + Constants.CONFIGURATORS_CATEGORY
                                                  + "," + Constants.ROUTERS_CATEGORY));
    //通过调用Cluster$Adpative类的join方法返回Invoker对象(基于Dubbo SPI机制实现setXX()方法自动注入属性)
    Invoker invoker = cluster.join(directory);
    ProviderConsumerRegTable.registerConsumer(invoker, url, subscribeUrl, directory);
    return invoker;
}
```

这里看下`Cluster$Adpative`类的join方法实现

```java
package org.apache.dubbo.rpc.cluster;

import org.apache.dubbo.common.extension.ExtensionLoader;

public class Cluster$Adaptive implements org.apache.dubbo.rpc.cluster.Cluster {
    private static final org.apache.dubbo.common.logger.Logger logger = org.apache.dubbo.common.logger.LoggerFactory.getLogger(ExtensionLoader.class);
    private java.util.concurrent.atomic.AtomicInteger count = new java.util.concurrent.atomic.AtomicInteger(0);

    public org.apache.dubbo.rpc.Invoker join(org.apache.dubbo.rpc.cluster.Directory arg0) throws org.apache.dubbo.rpc.RpcException {
        if (arg0 == null) throw new IllegalArgumentException("org.apache.dubbo.rpc.cluster.Directory argument == null");
        if (arg0.getUrl() == null)
            throw new IllegalArgumentException("org.apache.dubbo.rpc.cluster.Directory argument getUrl() == null");
        org.apache.dubbo.common.URL url = arg0.getUrl();
        String extName = url.getParameter("cluster", "failover");
        if (extName == null)
            throw new IllegalStateException("Fail to get extension(org.apache.dubbo.rpc.cluster.Cluster) name from url(" + url.toString() + ") use keys([cluster])");
        org.apache.dubbo.rpc.cluster.Cluster extension = null;
        try {
            extension = (org.apache.dubbo.rpc.cluster.Cluster) ExtensionLoader.getExtensionLoader(org.apache.dubbo.rpc.cluster.Cluster.class).getExtension(extName);
        } catch (Exception e) {
            if (count.incrementAndGet() == 1) {
                logger.warn("Failed to find extension named " + extName + " for type org.apache.dubbo.rpc.cluster.Cluster, will use default extension failover instead.", e);
            }
            extension = (org.apache.dubbo.rpc.cluster.Cluster) ExtensionLoader.getExtensionLoader(org.apache.dubbo.rpc.cluster.Cluster.class).getExtension("failover");
        }
        return extension.join(arg0);
    }
}
```

再看下FailoverCluster的join方法：

```java
@Override
public <T> Invoker<T> join(Directory<T> directory) throws RpcException {
    return new FailoverClusterInvoker<T>(directory);
}
```

由于Cluster SPI实现中有个MockClusterWrapper是包装类，这里牵扯到**Dubbo的AOP机制(后期详细分析)**，这里先调用它的join方法：

```java
@Override
public <T> Invoker<T> join(Directory<T> directory) throws RpcException {
    return new MockClusterInvoker<T>(directory,
                                     this.cluster.join(directory));
}
```

生成MockClusterInvoker之后，由于FailoverClusterInvoker是AbstractClusterInvoker的子类，它的invoke方法实现在其父类中，接下来的调用链是`MockClusterInvoker.invoke()->AbstractClusterInvoker.invoke()->FailoverClusterInvoker.doInvoke()`，下面来一步步分析，首先来看MockClusterInvoker类的invoke()方法：

```java
@Override
public Result invoke(Invocation invocation) throws RpcException {
    Result result = null;

    String value = directory.getUrl().getMethodParameter(invocation.getMethodName(), Constants.MOCK_KEY, Boolean.FALSE.toString()).trim();
    if (value.length() == 0 || value.equalsIgnoreCase("false")) {
        //no mock
        result = this.invoker.invoke(invocation);
    } else if (value.startsWith("force")) {
        if (logger.isWarnEnabled()) {
            logger.warn("force-mock: " + invocation.getMethodName() + " force-mock enabled , url : " + directory.getUrl());
        }
        //force:direct mock
        result = doMockInvoke(invocation, null);
    } else {
        //fail-mock
        try {
            result = this.invoker.invoke(invocation);
        } catch (RpcException e) {
            if (e.isBiz()) {
                throw e;
            } else {
                if (logger.isWarnEnabled()) {
                    logger.warn("fail-mock: " + invocation.getMethodName() + " fail-mock enabled , url : " + directory.getUrl(), e);
                }
                result = doMockInvoke(invocation, e);
            }
        }
    }
    return result;
}
```

然后调用AbstractClusterInvoker的invoke()方法：

```java
@Override
public Result invoke(final Invocation invocation) throws RpcException {
    checkWhetherDestroyed();

    // binding attachments into invocation.
    Map<String, String> contextAttachments = RpcContext.getContext().getAttachments();
    if (contextAttachments != null && contextAttachments.size() != 0) {
        ((RpcInvocation) invocation).addAttachments(contextAttachments);
    }

    List<Invoker<T>> invokers = list(invocation);
    LoadBalance loadbalance = initLoadBalance(invokers, invocation);
    RpcUtils.attachInvocationIdIfAsync(getUrl(), invocation);
    return doInvoke(invocation, invokers, loadbalance);
}
```

最后调用FailoverClusterInvoker的doInvoke()方法：

```java
@Override
@SuppressWarnings({"unchecked", "rawtypes"})
public Result doInvoke(Invocation invocation, final List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
    List<Invoker<T>> copyinvokers = invokers;
    checkInvokers(copyinvokers, invocation);
    String methodName = RpcUtils.getMethodName(invocation);
    //设置重试次数
    int len = getUrl().getMethodParameter(methodName, Constants.RETRIES_KEY, Constants.DEFAULT_RETRIES) + 1;
    if (len <= 0) {
        len = 1;
    }
    // retry loop.
    RpcException le = null; // last exception.
    List<Invoker<T>> invoked = new ArrayList<Invoker<T>>(copyinvokers.size()); // invoked invokers.
    Set<String> providers = new HashSet<String>(len);
    for (int i = 0; i < len; i++) {
        //Reselect before retry to avoid a change of candidate `invokers`.
        //NOTE: if `invokers` changed, then `invoked` also lose accuracy.
        if (i > 0) {
            checkWhetherDestroyed();
            copyinvokers = list(invocation);
            // check again
            checkInvokers(copyinvokers, invocation);
        }
        //根据负载均衡策略选择调用者
        Invoker<T> invoker = select(loadbalance, invocation, copyinvokers, invoked);
        //将使用过的invoker放入invoked
        invoked.add(invoker);
        RpcContext.getContext().setInvokers((List) invoked);
        try {
            Result result = invoker.invoke(invocation);
            if (le != null && logger.isWarnEnabled()) {
                logger.warn("Although retry the method " + methodName
                            + " in the service " + getInterface().getName()
                            + " was successful by the provider " + invoker.getUrl().getAddress()
                            + ", but there have been failed providers " + providers
                            + " (" + providers.size() + "/" + copyinvokers.size()
                            + ") from the registry " + directory.getUrl().getAddress()
                            + " on the consumer " + NetUtils.getLocalHost()
                            + " using the dubbo version " + Version.getVersion() + ". Last error is: "
                            + le.getMessage(), le);
            }
            return result;
        } catch (RpcException e) {
            if (e.isBiz()) { // biz exception.
                throw e;
            }
            le = e;
        } catch (Throwable e) {
            le = new RpcException(e.getMessage(), e);
        } finally {
            providers.add(invoker.getUrl().getAddress());
        }
    }
    throw new RpcException(le.getCode(), "Failed to invoke the method "
                           + methodName + " in the service " + getInterface().getName()
                           + ". Tried " + len + " times of the providers " + providers
                           + " (" + providers.size() + "/" + copyinvokers.size()
                           + ") from the registry " + directory.getUrl().getAddress()
                           + " on the consumer " + NetUtils.getLocalHost() + " using the dubbo version "
                           + Version.getVersion() + ". Last error is: "
                           + le.getMessage(), le.getCause() != null ? le.getCause() : le);
}
```

所以会有如下的线程栈信息：

```
at org.apache.dubbo.rpc.cluster.support.FailoverClusterInvoker.doInvoke(FailoverClusterInvoker.java:78)
at org.apache.dubbo.rpc.cluster.support.AbstractClusterInvoker.invoke(AbstractClusterInvoker.java:243)
at org.apache.dubbo.rpc.cluster.support.wrapper.MockClusterInvoker.invoke(MockClusterInvoker.java:75)
```

这些类都是关于**Dubbo的集群容错机制**（将会写一篇关于Dubbo的集群容错机制）。

再往下看invokers是如何生成的呢？又回到AbstractClusterInvoker的invoke方法实现：

```java
@Override
public Result invoke(final Invocation invocation) throws RpcException {
    checkWhetherDestroyed();
    LoadBalance loadbalance = null;

    // binding attachments into invocation.
    Map<String, String> contextAttachments = RpcContext.getContext().getAttachments();
    if (contextAttachments != null && contextAttachments.size() != 0) {
        ((RpcInvocation) invocation).addAttachments(contextAttachments);
    }
	//会调用directory的list方法 返回要调用invokers集合。
    //其实是AbstractDirectory的list方法，这个方法里就是利用路由规则（如果有），从所有
    //提供者中，遴选出符合规则的提供者,接下里才是，集群容错和负载均衡。
    List<Invoker<T>> invokers = list(invocation);
    if (invokers != null && !invokers.isEmpty()) {
        //通过key（loadbalance）从url中取值，默认值为random
        loadbalance = ExtensionLoader.getExtensionLoader(LoadBalance.class).getExtension(invokers.get(0).getUrl()                                                                                .getMethodParameter(RpcUtils.getMethodName(invocation), Constants.LOADBALANCE_KEY, Constants.DEFAULT_LOADBALANCE));
    }
    RpcUtils.attachInvocationIdIfAsync(getUrl(), invocation);
    return doInvoke(invocation, invokers, loadbalance);
}
```

再来看一下list方法：

```java
protected List<Invoker<T>> list(Invocation invocation) throws RpcException {
    //directory.list(invocation)获取invokers，这里directory是RegistryDirectory
    List<Invoker<T>> invokers = directory.list(invocation);
    return invokers;
}
```

跟到RegistryDirectory类的list方法，实现在其父类AbstractDirectory中，主要是**生成路由规则**（将会在另一篇文章中详细讲解，敬请期待）：

```java
@Override
public List<Invoker<T>> list(Invocation invocation) throws RpcException {
    if (destroyed) {
        throw new RpcException("Directory already destroyed .url: " + getUrl());
    }
    //获取所有的提供者
    List<Invoker<T>> invokers = doList(invocation);
    //本地路由规则
    List<Router> localRouters = this.routers; // local reference
    if (localRouters != null && !localRouters.isEmpty()) {
        for (Router router : localRouters) {
            try {
                if (router.getUrl() == null || router.getUrl().getParameter(Constants.RUNTIME_KEY, false)) {
                    //Router接口，实现route的方法，路由获取服务提供者
                    invokers = router.route(invokers, getConsumerUrl(), invocation);
                }
            } catch (Throwable t) {
                logger.error("Failed to execute router: " + getUrl() + ", cause: " + t.getMessage(), t);
            }
        }
    }
    return invokers;
}
```

再来看一下doList方法，它是个抽象方法具体实现在RegistryDirectory类中：

```java
@Override
public List<Invoker<T>> doList(Invocation invocation) {
    //没有服务提供者或者服务提供者被禁用
    if (forbidden) {
        // 1. No service provider 2. Service providers are disabled
        throw new RpcException(RpcException.FORBIDDEN_EXCEPTION,
                               "No provider available from registry " + getUrl().getAddress() + " for service " + getConsumerUrl().getServiceKey() + " on consumer " + NetUtils.getLocalHost()
                               + " use dubbo version " + Version.getVersion() + ", please check status of providers(disabled, not registered or in blacklist).");
    }
    List<Invoker<T>> invokers = null;
    //从这里搜索methodInvokerMap赋值，在refreshInvoker方法里
    Map<String, List<Invoker<T>>> localMethodInvokerMap = this.methodInvokerMap; // local reference
    if (localMethodInvokerMap != null && localMethodInvokerMap.size() > 0) {
        String methodName = RpcUtils.getMethodName(invocation);
        Object[] args = RpcUtils.getArguments(invocation);
        if (args != null && args.length > 0 && args[0] != null
            && (args[0] instanceof String || args[0].getClass().isEnum())) {
            invokers = localMethodInvokerMap.get(methodName + "." + args[0]); // The routing can be enumerated according to the first parameter
        }
        if (invokers == null) {
            invokers = localMethodInvokerMap.get(methodName);
        }
        if (invokers == null) {
            invokers = localMethodInvokerMap.get(Constants.ANY_VALUE);
        }
        if (invokers == null) {
            Iterator<List<Invoker<T>>> iterator = localMethodInvokerMap.values().iterator();
            if (iterator.hasNext()) {
                invokers = iterator.next();
            }
        }
    }
    return invokers == null ? new ArrayList<Invoker<T>>(0) : invokers;
}
```

下面是`refreshInvoker(List<URL> invokerUrls)`方法的实现：

```java
/**
     * Convert the invokerURL list to the Invoker Map. The rules of the conversion are as follows:
     * 1.If URL has been converted to invoker, it is no longer re-referenced and obtained directly from the cache, and notice that any parameter changes in the URL will be re-referenced.
     * 2.If the incoming invoker list is not empty, it means that it is the latest invoker list
     * 3.If the list of incoming invokerUrl is empty, It means that the rule is only a override rule or a route rule, which needs to be re-contrasted to decide whether to re-reference.
     *
     * @param invokerUrls this parameter can't be null
     */
// TODO: 2017/8/31 FIXME The thread pool should be used to refresh the address, otherwise the task may be accumulated.
private void refreshInvoker(List<URL> invokerUrls) {
    if (invokerUrls != null && invokerUrls.size() == 1 && invokerUrls.get(0) != null
        && Constants.EMPTY_PROTOCOL.equals(invokerUrls.get(0).getProtocol())) {
        //禁止访问
        this.forbidden = true; // Forbid to access
        //置空列表
        this.methodInvokerMap = null; // Set the method invoker map to null
        //关闭所有invokers
        destroyAllInvokers(); // Close all invokers
    } else {
        //允许访问
        this.forbidden = false; // Allow to access
        Map<String, Invoker<T>> oldUrlInvokerMap = this.urlInvokerMap; // local reference
        if (invokerUrls.isEmpty() && this.cachedInvokerUrls != null) {
            invokerUrls.addAll(this.cachedInvokerUrls);
        } else {
            this.cachedInvokerUrls = new HashSet<URL>();
            //缓存invokerUrls列表，便于交叉对比
            this.cachedInvokerUrls.addAll(invokerUrls);//Cached invoker urls, convenient for comparison
        }
        if (invokerUrls.isEmpty()) {
            return;
        }
        //生成invoker方法 toInvokers
        //将url列表转换成invoker列表
        Map<String, Invoker<T>> newUrlInvokerMap = toInvokers(invokerUrls);// Translate url list to Invoker map
        //换方法名映射invoker列表
        Map<String, List<Invoker<T>>> newMethodInvokerMap = toMethodInvokers(newUrlInvokerMap); // Change method name to map Invoker Map
        // state change
        // If the calculation is wrong, it is not processed.
        //如果计算错误则不进行处理
        if (newUrlInvokerMap == null || newUrlInvokerMap.size() == 0) {
            logger.error(new IllegalStateException("urls to invokers error .invokerUrls.size :" + invokerUrls.size() + ", invoker.size :0. urls :" + invokerUrls.toString()));
            return;
        }
        this.methodInvokerMap = multiGroup ? toMergeMethodInvokerMap(newMethodInvokerMap) : newMethodInvokerMap;
        this.urlInvokerMap = newUrlInvokerMap;
        try {
            //关闭不使用的invoker
            destroyUnusedInvokers(oldUrlInvokerMap, newUrlInvokerMap); // Close the unused Invoker
        } catch (Exception e) {
            logger.warn("destroyUnusedInvokers error. ", e);
        }
    }
}
```

可以知道refreshInvoker()方法会在RegistryDirectory类的notify()方法里调用，这个方法是**订阅注册中心的回调方法**。下面来看一下toInvokers()的方法实现：

```java
/**
     * Turn urls into invokers, and if url has been refer, will not re-reference.
     * 将urls转换成invokers，如果url已经被refer过则不在重新引用
     * @param urls
     * @return invokers
     */
private Map<String, Invoker<T>> toInvokers(List<URL> urls) {
    Map<String, Invoker<T>> newUrlInvokerMap = new HashMap<String, Invoker<T>>();
    if (urls == null || urls.isEmpty()) {
        return newUrlInvokerMap;
    }
    Set<String> keys = new HashSet<String>();
    String queryProtocols = this.queryMap.get(Constants.PROTOCOL_KEY);
    for (URL providerUrl : urls) {
        // If protocol is configured at the reference side, only the matching protocol is selected
        //若果reference端配置了protocol，则只选择匹配的protocol
        if (queryProtocols != null && queryProtocols.length() > 0) {
            boolean accept = false;
            String[] acceptProtocols = queryProtocols.split(",");
            for (String acceptProtocol : acceptProtocols) {
                if (providerUrl.getProtocol().equals(acceptProtocol)) {
                    accept = true;
                    break;
                }
            }
            if (!accept) {
                continue;
            }
        }
        if (Constants.EMPTY_PROTOCOL.equals(providerUrl.getProtocol())) {
            continue;
        }
        if (!ExtensionLoader.getExtensionLoader(Protocol.class).hasExtension(providerUrl.getProtocol())) {
            logger.error(new IllegalStateException("Unsupported protocol " + providerUrl.getProtocol() + " in notified url: " + providerUrl + " from registry " + getUrl().getAddress() + " to consumer " + NetUtils.getLocalHost()
                                                   + ", supported protocol: " + ExtensionLoader.getExtensionLoader(Protocol.class).getSupportedExtensions()));
            continue;
        }
        URL url = mergeUrl(providerUrl);
		//url参数是排序的
        String key = url.toFullString(); // The parameter urls are sorted
        //跳过重复的url
        if (keys.contains(key)) { // Repeated url
            continue;
        }
        keys.add(key);
        // Cache key is url that does not merge with consumer side parameters, regardless of how the consumer combines parameters, if the server url changes, then refer again
        //缓存key为没有合并消费端参数的URL，不管消费端如何合并参数，如果服务端URL发生变化，则重新refer
        Map<String, Invoker<T>> localUrlInvokerMap = this.urlInvokerMap; // local reference
        Invoker<T> invoker = localUrlInvokerMap == null ? null : localUrlInvokerMap.get(key);
        if (invoker == null) { // Not in the cache, refer again
            try {
                boolean enabled = true;
                if (url.hasParameter(Constants.DISABLED_KEY)) {
                    enabled = !url.getParameter(Constants.DISABLED_KEY, false);
                } else {
                    enabled = url.getParameter(Constants.ENABLED_KEY, true);
                }
                if (enabled) {
                    //创建invoker（这里创建invoker）
                    invoker = new InvokerDelegate<T>(protocol.refer(serviceType, url), url, providerUrl);
                }
            } catch (Throwable t) {
                logger.error("Failed to refer invoker for interface:" + serviceType + ",url:(" + url + ")" + t.getMessage(), t);
            }
            if (invoker != null) { // Put new invoker in cache
                //将新的引用放入缓存
                newUrlInvokerMap.put(key, invoker);
            }
        } else {
            newUrlInvokerMap.put(key, invoker);
        }
    }
    keys.clear();
    return newUrlInvokerMap;
}
```

找到了invoker的创建地方，来看下InvokerDelegate，它是RegistryDirectory的内部类：

```java
/**
     * The delegate class, which is mainly used to store the URL address sent by the registry,and can be reassembled on the basis of providerURL queryMap overrideMap for re-refer.
     * 代理类，主要用于存储注册中心下发的URL地址
     * 用于重新refer时能够根据providerURL queryMap overrideMap重新组装
     * @param <T>
     */
private static class InvokerDelegate<T> extends InvokerWrapper<T> {
    private URL providerUrl;

    public InvokerDelegate(Invoker<T> invoker, URL url, URL providerUrl) {
        //调用父类构造方法
        super(invoker, url);
        this.providerUrl = providerUrl;
    }

    public URL getProviderUrl() {
        return providerUrl;
    }
}
```

invoke方法在父类InvokerWrapper里实现的：

```java
@Override
public Result invoke(Invocation invocation) throws RpcException {
    //这里的invoker是从它的构造方法里传入的
    return invoker.invoke(invocation);
}
```

所以在调用栈里可以看到如下一条信息：

```
at org.apache.dubbo.rpc.protocol.InvokerWrapper.invoke(InvokerWrapper.java:56)
```

InvokerDelegete构造方法调用的父类InvokerWrapper的构造方法并传入invoker实例，回头看`new InvokerDelegate<T>(protocol.refer(serviceType, url), url, providerUrl);`这句，可知上面的invoker的是由`protocol.refer(serviceType, url)`创建的。

通过debug可得知这里的protocol是Protocol$Adaptive类型的，这里的url的protocol是dubbo，通过Dubbo SPI机制可以得到这里最后走DubboProtocol类的refer()方法，但是由于Protocol接口实现中由两个包装类：

```
filter=org.apache.dubbo.rpc.protocol.ProtocolFilterWrapper
listener=org.apache.dubbo.rpc.protocol.ProtocolListenerWrapper
```

所以这里先执行ProtocolFilterWrapper的refer方法，再执行ProtocolListenerWrapper的refer方法，最后才执行DubboProtocol类的refer方法，ProtocolFilterWrapper类的refer方法如下：

```java
@Override
public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
    if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
        return protocol.refer(type, url);
    }
    return buildInvokerChain(protocol.refer(type, url), Constants.REFERENCE_FILTER_KEY, Constants.CONSUMER);
}
```

方法里调用了buildInvokerChain()方法：

```java
private static <T> Invoker<T> buildInvokerChain(final Invoker<T> invoker, String key, String group) {
    Invoker<T> last = invoker;
    //先获取激活的过滤器，我们这里手动配置了monitor MonitorFilter过滤器
    //另外两个自动激活的过滤器是FutureFilter，ConsumerContextFilter
    //这里需要看SPI机制的getActivateExtension方法的相关代码
    List<Filter> filters = ExtensionLoader.getExtensionLoader(Filter.class).getActivateExtension(invoker.getUrl(), key, group);
    if (!filters.isEmpty()) {
        for (int i = filters.size() - 1; i >= 0; i--) {
            final Filter filter = filters.get(i);
            final Invoker<T> next = last;
            last = new Invoker<T>() {

                @Override
                public Class<T> getInterface() {
                    return invoker.getInterface();
                }

                @Override
                public URL getUrl() {
                    return invoker.getUrl();
                }

                @Override
                public boolean isAvailable() {
                    return invoker.isAvailable();
                }
				//实现invoker的invoke方法
                @Override
                public Result invoke(Invocation invocation) throws RpcException {
                    //嵌套进过滤器链
                    return filter.invoke(next, invocation);
                }

                @Override
                public void destroy() {
                    invoker.destroy();
                }

                @Override
                public String toString() {
                    return invoker.toString();
                }
            };
        }
    }
    return last;
}
```

所以有如下的调用栈信息：

```
at org.apache.dubbo.monitor.support.MonitorFilter.invoke(MonitorFilter.java:75)
at org.apache.dubbo.rpc.protocol.ProtocolFilterWrapper$1.invoke(ProtocolFilterWrapper.java:72)
at org.apache.dubbo.rpc.protocol.dubbo.filter.FutureFilter.invoke(FutureFilter.java:47)
at org.apache.dubbo.rpc.protocol.ProtocolFilterWrapper$1.invoke(ProtocolFilterWrapper.java:72)
at org.apache.dubbo.rpc.filter.ConsumerContextFilter.invoke(ConsumerContextFilter.java:50)
at org.apache.dubbo.rpc.protocol.ProtocolFilterWrapper$1.invoke(ProtocolFilterWrapper.java:72)
```

接着ProtocolListenerWrapper的refer方法：

```java
@Override
public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
    if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
        return protocol.refer(type, url);
    }
    //获取激活的监听器，目前dubbo没有提供合适的监听器，只有一个DeprecatedInvokerListener实现类，并且还是Deprecated的
    return new ListenerInvokerWrapper<T>(protocol.refer(type, url),
                                         Collections.unmodifiableList(
                                             ExtensionLoader.getExtensionLoader(InvokerListener.class)
                                             .getActivateExtension(url, Constants.INVOKER_LISTENER_KEY)));
}
```

所以会出现如下栈信息：

```
at org.apache.dubbo.rpc.listener.ListenerInvokerWrapper.invoke(ListenerInvokerWrapper.java:77)
```



最后看一下DubboProtocol类的refer方法，这里创建了DubboInvoker对象：

```java
@Override
public <T> Invoker<T> refer(Class<T> serviceType, URL url) throws RpcException {
    optimizeSerialization(url);
    // create rpc invoker.
    DubboInvoker<T> invoker = new DubboInvoker<T>(serviceType, url, getClients(url), invokers);
    invokers.add(invoker);
    return invoker;
}
```

DubboInvoker的父类AbstractInvoker实现了invoke方法：

```java
@Override
public Result invoke(Invocation inv) throws RpcException {
    if (destroyed.get()) {
        throw new RpcException("Rpc invoker for service " + this + " on consumer " + NetUtils.getLocalHost()
                               + " use dubbo version " + Version.getVersion()
                               + " is DESTROYED, can not be invoked any more!");
    }
    RpcInvocation invocation = (RpcInvocation) inv;
    invocation.setInvoker(this);
    if (attachment != null && attachment.size() > 0) {
        invocation.addAttachmentsIfAbsent(attachment);
    }
    Map<String, String> contextAttachments = RpcContext.getContext().getAttachments();
    if (contextAttachments != null && contextAttachments.size() != 0) {
        /**
             * invocation.addAttachmentsIfAbsent(context){@link RpcInvocation#addAttachmentsIfAbsent(Map)}should not be used here,
             * because the {@link RpcContext#setAttachment(String, String)} is passed in the Filter when the call is triggered
             * by the built-in retry mechanism of the Dubbo. The attachment to update RpcContext will no longer work, which is
             * a mistake in most cases (for example, through Filter to RpcContext output traceId and spanId and other information).
             */
        invocation.addAttachments(contextAttachments);
    }
    if (getUrl().getMethodParameter(invocation.getMethodName(), Constants.ASYNC_KEY, false)) {
        invocation.setAttachment(Constants.ASYNC_KEY, Boolean.TRUE.toString());
    }
    RpcUtils.attachInvocationIdIfAsync(getUrl(), invocation);


    try {
        //doInvoke()方法具体实现在子类中
        return doInvoke(invocation);
    } catch (InvocationTargetException e) { // biz exception
        Throwable te = e.getTargetException();
        if (te == null) {
            return new RpcResult(e);
        } else {
            if (te instanceof RpcException) {
                ((RpcException) te).setCode(RpcException.BIZ_EXCEPTION);
            }
            return new RpcResult(te);
        }
    } catch (RpcException e) {
        if (e.isBiz()) {
            return new RpcResult(e);
        } else {
            throw e;
        }
    } catch (Throwable e) {
        return new RpcResult(e);
    }
}
```

看一下DubboInvoker实现的doInvoke方法：

```java
@Override
protected Result doInvoke(final Invocation invocation) throws Throwable {
    RpcInvocation inv = (RpcInvocation) invocation;
    final String methodName = RpcUtils.getMethodName(invocation);
    inv.setAttachment(Constants.PATH_KEY, getUrl().getPath());
    inv.setAttachment(Constants.VERSION_KEY, version);

    ExchangeClient currentClient;
    if (clients.length == 1) {
        currentClient = clients[0];
    } else {
        currentClient = clients[index.getAndIncrement() % clients.length];
    }
    try {
        boolean isAsync = RpcUtils.isAsync(getUrl(), invocation);
        boolean isAsyncFuture = RpcUtils.isGeneratedFuture(inv) || RpcUtils.isFutureReturnType(inv);
        boolean isOneway = RpcUtils.isOneway(getUrl(), invocation);
        int timeout = getUrl().getMethodParameter(methodName, Constants.TIMEOUT_KEY, Constants.DEFAULT_TIMEOUT);
        if (isOneway) {
            boolean isSent = getUrl().getMethodParameter(methodName, Constants.SENT_KEY, false);
            currentClient.send(inv, isSent);
            RpcContext.getContext().setFuture(null);
            return new RpcResult();
        } else if (isAsync) {
            ResponseFuture future = currentClient.request(inv, timeout);
            // For compatibility
            FutureAdapter<Object> futureAdapter = new FutureAdapter<>(future);
            RpcContext.getContext().setFuture(futureAdapter);

            Result result;
            if (isAsyncFuture) {
                // register resultCallback, sometimes we need the asyn result being processed by the filter chain.
                result = new AsyncRpcResult(futureAdapter, futureAdapter.getResultFuture(), false);
            } else {
                result = new SimpleAsyncRpcResult(futureAdapter, futureAdapter.getResultFuture(), false);
            }
            return result;
        } else {
            RpcContext.getContext().setFuture(null);
            return (Result) currentClient.request(inv, timeout).get();
        }
    } catch (TimeoutException e) {
        throw new RpcException(RpcException.TIMEOUT_EXCEPTION, "Invoke remote method timeout. method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
    } catch (RemotingException e) {
        throw new RpcException(RpcException.NETWORK_EXCEPTION, "Failed to invoke remote method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
    }
}
```

所以会有这两句线程栈输出：

```
at org.apache.dubbo.rpc.protocol.dubbo.DubboInvoker.doInvoke(DubboInvoker.java:108)
at org.apache.dubbo.rpc.protocol.AbstractInvoker.invoke(AbstractInvoker.java:154)
```

接下来就是用于发送请求的currentClient对象的实现了，它的逻辑可以追踪到DubboProtocol类的refer方法里：

```java
@Override
public <T> Invoker<T> refer(Class<T> serviceType, URL url) throws RpcException {
    optimizeSerialization(url);
    // create rpc invoker.
    DubboInvoker<T> invoker = new DubboInvoker<T>(serviceType, url, getClients(url), invokers);
    invokers.add(invoker);
    return invoker;
}
```

具体的获取逻辑在getClients()方法中：

```java
private ExchangeClient[] getClients(URL url) {
    // whether to share connection
    //是否共享连接
    boolean service_share_connect = false;
    int connections = url.getParameter(Constants.CONNECTIONS_KEY, 0);
    // if not configured, connection is shared, otherwise, one connection for one service
    //如果没有配置connection，那么就创建共享连接，否则一个服务一个连接？
    if (connections == 0) {
        service_share_connect = true;
        connections = 1;
    }

    ExchangeClient[] clients = new ExchangeClient[connections];
    for (int i = 0; i < clients.length; i++) {
        if (service_share_connect) {
            clients[i] = getSharedClient(url);
        } else {
            clients[i] = initClient(url);
        }
    }
    return clients;
}
/**
     * Get shared connection
     */
private ExchangeClient getSharedClient(URL url) {
    String key = url.getAddress();
    ReferenceCountExchangeClient client = referenceClientMap.get(key);
    if (client != null) {
        if (!client.isClosed()) {
            client.incrementAndGetCount();
            return client;
        } else {
            referenceClientMap.remove(key);
        }
    }

    locks.putIfAbsent(key, new Object());
    synchronized (locks.get(key)) {
        if (referenceClientMap.containsKey(key)) {
            return referenceClientMap.get(key);
        }

        ExchangeClient exchangeClient = initClient(url);
        client = new ReferenceCountExchangeClient(exchangeClient, ghostClientMap);
        referenceClientMap.put(key, client);
        ghostClientMap.remove(key);
        locks.remove(key);
        return client;
    }
}
/**
     * Create new connection
     */
private ExchangeClient initClient(URL url) {

    // client type setting.
    String str = url.getParameter(Constants.CLIENT_KEY, url.getParameter(Constants.SERVER_KEY, Constants.DEFAULT_REMOTING_CLIENT));

    url = url.addParameter(Constants.CODEC_KEY, DubboCodec.NAME);
    // enable heartbeat by default
    //默认开启heartbeat
    url = url.addParameterIfAbsent(Constants.HEARTBEAT_KEY, String.valueOf(Constants.DEFAULT_HEARTBEAT));

    // BIO is not allowed since it has severe performance issue.
    //BIO存在严重的性能问题，因此不能使用
    if (str != null && str.length() > 0 && !ExtensionLoader.getExtensionLoader(Transporter.class).hasExtension(str)) {
        throw new RpcException("Unsupported client type: " + str + "," +
                               " supported client type is " + StringUtils.join(ExtensionLoader.getExtensionLoader(Transporter.class).getSupportedExtensions(), " "));
    }

    ExchangeClient client;
    try {
        // connection should be lazy
        if (url.getParameter(Constants.LAZY_CONNECT_KEY, false)) {
            client = new LazyConnectExchangeClient(url, requestHandler);
        } else {
            //通过 Exchangers.connect(url, requestHandler); 构建client ，接下来跟踪Exchangers.connect方法
		    //这里会传入一个requestHandler，这个是客户端解救服务端方法返回回调的
            client = Exchangers.connect(url, requestHandler);
        }
    } catch (RemotingException e) {
        throw new RpcException("Fail to create remoting client for service(" + url + "): " + e.getMessage(), e);
    }
    return client;
}
```

这里用到了**Facade设计模式，Exchangers是个门面类**，封装了具体查找合适的Exchanger实现，并调用connect方法返回ExchangeClient的过程，相关代码如下：

```java
public static ExchangeClient connect(URL url, ExchangeHandler handler) throws RemotingException {
    if (url == null) {
        throw new IllegalArgumentException("url == null");
    }
    if (handler == null) {
        throw new IllegalArgumentException("handler == null");
    }
    url = url.addParameterIfAbsent(Constants.CODEC_KEY, "exchange");
    //把codec key设置为exchange
    return getExchanger(url).connect(url, handler);
}

public static Exchanger getExchanger(URL url) {
    String type = url.getParameter(Constants.EXCHANGER_KEY, Constants.DEFAULT_EXCHANGER);
    //通过exchanger key 获取 Exchanger的spi实现，默认是header，这里是HeaderExchanger类
    return getExchanger(type);
}

public static Exchanger getExchanger(String type) {
    //这里返回Exchanger接口的header扩展类的HeaderExchanger
    return ExtensionLoader.getExtensionLoader(Exchanger.class).getExtension(type);
}
```

看一下HeaderExchanger类的connect方法：

```java
//客户端的连接操作
@Override
public ExchangeClient connect(URL url, ExchangeHandler handler) throws RemotingException {
    return new HeaderExchangeClient(Transporters.connect(url, new DecodeHandler(new HeaderExchangeHandler(handler))), true);
}
```

所以有栈信息：

```
at org.apache.dubbo.remoting.exchange.support.header.HeaderExchangeClient.request(HeaderExchangeClient.java:90)
```

再来看HeaderExchnageClient的request()方法：

```java
public HeaderExchangeClient(Client client, boolean needHeartbeat) {
    if (client == null) {
        throw new IllegalArgumentException("client == null");
    }
    this.client = client;
    //初始化HeaderExchangeChannel
    this.channel = new HeaderExchangeChannel(client);
    String dubbo = client.getUrl().getParameter(Constants.DUBBO_VERSION_KEY);
    this.heartbeat = client.getUrl().getParameter(Constants.HEARTBEAT_KEY, dubbo != null && dubbo.startsWith("1.0.") ? Constants.DEFAULT_HEARTBEAT : 0);
    this.heartbeatTimeout = client.getUrl().getParameter(Constants.HEARTBEAT_TIMEOUT_KEY, heartbeat * 3);
    if (heartbeatTimeout < heartbeat * 2) {
        throw new IllegalStateException("heartbeatTimeout < heartbeatInterval * 2");
    }
    if (needHeartbeat) {
        startHeartbeatTimer();
    }
}

@Override
public ResponseFuture request(Object request) throws RemotingException {
    //调用Channel的request方法
    return channel.request(request);
}
```

因此会有如下的栈信息：

```
at org.apache.dubbo.remoting.exchange.support.header.HeaderExchangeChannel.request(HeaderExchangeChannel.java:116)
```

再来看一下HeaderExchangeChannel的request方法：

```java
@Override
public ResponseFuture request(Object request) throws RemotingException {
    return request(request, channel.getUrl().getPositiveParameter(Constants.TIMEOUT_KEY, Constants.DEFAULT_TIMEOUT));
}

@Override
public ResponseFuture request(Object request, int timeout) throws RemotingException {
    if (closed) {
        throw new RemotingException(this.getLocalAddress(), null, "Failed to send request " + request + ", cause: The channel " + this + " is closed!");
    }
    // create request.
    Request req = new Request();
    req.setVersion(Version.getProtocolVersion());
    req.setTwoWay(true);
    req.setData(request);
    DefaultFuture future = new DefaultFuture(channel, req, timeout);
    try {
        //发送请求
        channel.send(req);
    } catch (RemotingException e) {
        future.cancel();
        throw e;
    }
    return future;
}
```

`channel.send(req);`中channel实例是通过`HeaderExchangeChannel(Channel channel)`构造函数初始化的，继续往上看是通过构造函数`public HeaderExchangeClient(Client client, boolean needHeartbeat)`传进来的，最终生成channel的代码是在类`HeaderExchanger`中：

```java
@Override
public ExchangeClient connect(URL url, ExchangeHandler handler) throws RemotingException {
    return new HeaderExchangeClient(Transporters.connect(url, new DecodeHandler(new HeaderExchangeHandler(handler))), true);
}
```

调用`Transporters.connect(url, new DecodeHandler(new HeaderExchangeHandler(handler)))`生成channel实例，**这里Transporters也是个门面类，是Facade设计模式的实现**，继续分析Transporters的connect方法：

```Java
public static Client connect(URL url, ChannelHandler... handlers) throws RemotingException {
    if (url == null) {
        throw new IllegalArgumentException("url == null");
    }
    ChannelHandler handler;
    if (handlers == null || handlers.length == 0) {
        handler = new ChannelHandlerAdapter();
    } else if (handlers.length == 1) {
        handler = handlers[0];
    } else {
        handler = new ChannelHandlerDispatcher(handlers);
    }
    //所以这里默认返回的NettyClient
    return getTransporter().connect(url, handler);
}
//这个方法根据spi返回NettyTransporter扩展类
public static Transporter getTransporter() {
    //生成Transporter$Adaptive类
    return ExtensionLoader.getExtensionLoader(Transporter.class).getAdaptiveExtension();
}
```

所以最后是通过NettyClient类实例的send方法发送具体请求，NettyClient类的send方法在其祖先类AbstractPeer中：

```java
@Override
public void send(Object message) throws RemotingException {
    send(message, url.getParameter(Constants.SENT_KEY, false));
}
```

这个实现又调用NettyClient父类AbstractClient的send方法实现：

```Java
@Override
public void send(Object message, boolean sent) throws RemotingException {
    if (send_reconnect && !isConnected()) {
        connect();
    }
    //获取具体的channel实例
    Channel channel = getChannel();
    //TODO Can the value returned by getChannel() be null? need improvement.
    if (channel == null || !channel.isConnected()) {
        throw new RemotingException(this, "message can not send, because channel is closed . url:" + getUrl());
    }
    channel.send(message, sent);
}
```

这里的getChannel()方法由NettyClient自身实现，如下：

```java
@Override
protected org.apache.dubbo.remoting.Channel getChannel() {
    Channel c = channel;
    if (c == null || !c.isActive())
        return null;
    return NettyChannel.getOrAddChannel(c, getUrl(), this);
}
```

所以有如下栈信息：

```
at org.apache.dubbo.remoting.transport.netty4.NettyChannel.send(NettyChannel.java:101)
at org.apache.dubbo.remoting.transport.AbstractClient.send(AbstractClient.java:265)
at org.apache.dubbo.remoting.transport.AbstractPeer.send(AbstractPeer.java:53)
```

最后就走到NettyChannel的send方法，即到了断点处：

```java
@Override
public void send(Object message, boolean sent) throws RemotingException {
    super.send(message, sent);

    boolean success = true;
    int timeout = 0;
    try {
        //断点处
        ChannelFuture future = channel.writeAndFlush(message);
        if (sent) {
            timeout = getUrl().getPositiveParameter(Constants.TIMEOUT_KEY, Constants.DEFAULT_TIMEOUT);
            success = future.await(timeout);
        }
        Throwable cause = future.cause();
        if (cause != null) {
            throw cause;
        }
    } catch (Throwable e) {
        throw new RemotingException(this, "Failed to send message " + message + " to " + getRemoteAddress() + ", cause: " + e.getMessage(), e);
    }

    if (!success) {
        throw new RemotingException(this, "Failed to send message " + message + " to " + getRemoteAddress()
                                    + "in timeout(" + timeout + "ms) limit");
    }
}
```

到此，整个消费者调用过程就分析完了，文章中提到的一些关于Dubbo的核心feature将会写文章进一步分析，敬请期待。
