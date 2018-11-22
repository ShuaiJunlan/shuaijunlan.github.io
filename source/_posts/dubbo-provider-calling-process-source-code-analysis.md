---
title: Dubbo服务提供者发布及注册过程源码分析
date: 2018-09-09 20:24:34
tags:
    - dubbo
---

Dubbo服务端在启动服务时会经历怎样的调用过程？在收到消费者发送的请求后会经历怎样的调用过程？这篇文章主要针对以上两个调用过程并结合Dubbo源码进行分析。

我们采用的是[**Consumer-Provider的Demo**](https://github.com/shuaijunlan/Spring-Learning/tree/master/dubbo)提供的示例，并按照[《Dubbo消费者调用过程源码分析》](https://shuaijunlan.github.io/2018/08/05/dubbo-consumer-calling-process-source-code-analysis/)中的分析思路，下面将对两种过程进行进一步分析，先来看一张服务发布过程的时序图(图片太大建议在新的窗口打开查看)，对服务发布与注册有个大致的了解：

![](https://github.com/shuaijunlan/shuaijunlan.github.io/blob/master/images/dubbo-provider.png?raw=true)

<!-- more -->

### Dubbo服务发布过程

首先来看一下服务发布过程调用栈，我们将围绕这个调用栈一步步进行分析：

```
"main@1" prio=5 tid=0x1 nid=NA runnable
  java.lang.Thread.State: RUNNABLE
	  at org.apache.dubbo.remoting.transport.netty4.NettyServer.doOpen(NettyServer.java:97)
	  at org.apache.dubbo.remoting.transport.AbstractServer.<init>(AbstractServer.java:63)
	  at org.apache.dubbo.remoting.transport.netty4.NettyServer.<init>(NettyServer.java:65)
	  at org.apache.dubbo.remoting.transport.netty4.NettyTransporter.bind(NettyTransporter.java:32)
	  at org.apache.dubbo.remoting.Transporter$Adaptive.bind(Transporter$Adaptive.java:-1)
	  at org.apache.dubbo.remoting.Transporters.bind(Transporters.java:56)
	  at org.apache.dubbo.remoting.exchange.support.header.HeaderExchanger.bind(HeaderExchanger.java:44)
	  at org.apache.dubbo.remoting.exchange.Exchangers.bind(Exchangers.java:70)
	  at org.apache.dubbo.rpc.protocol.dubbo.DubboProtocol.createServer(DubboProtocol.java:306)
	  at org.apache.dubbo.rpc.protocol.dubbo.DubboProtocol.openServer(DubboProtocol.java:283)
	  - locked <0x9ee> (a org.apache.dubbo.rpc.protocol.dubbo.DubboProtocol)
	  at org.apache.dubbo.rpc.protocol.dubbo.DubboProtocol.export(DubboProtocol.java:267)
	  at org.apache.dubbo.rpc.protocol.ProtocolListenerWrapper.export(ProtocolListenerWrapper.java:57)
	  at org.apache.dubbo.qos.protocol.QosProtocolWrapper.export(QosProtocolWrapper.java:63)
	  at org.apache.dubbo.rpc.protocol.ProtocolFilterWrapper.export(ProtocolFilterWrapper.java:100)
	  at org.apache.dubbo.rpc.Protocol$Adaptive.export(Protocol$Adaptive.java:-1)
	  at org.apache.dubbo.registry.integration.RegistryProtocol.doLocalExport(RegistryProtocol.java:170)
	  - locked <0x9e2> (a java.util.concurrent.ConcurrentHashMap)
	  at org.apache.dubbo.registry.integration.RegistryProtocol.export(RegistryProtocol.java:133)
	  at org.apache.dubbo.rpc.protocol.ProtocolListenerWrapper.export(ProtocolListenerWrapper.java:55)
	  at org.apache.dubbo.qos.protocol.QosProtocolWrapper.export(QosProtocolWrapper.java:61)
	  at org.apache.dubbo.rpc.protocol.ProtocolFilterWrapper.export(ProtocolFilterWrapper.java:98)
	  at org.apache.dubbo.rpc.Protocol$Adaptive.export(Protocol$Adaptive.java:-1)
	  at org.apache.dubbo.config.ServiceConfig.doExportUrlsFor1Protocol(ServiceConfig.java:513)
	  at org.apache.dubbo.config.ServiceConfig.doExportUrls(ServiceConfig.java:358)
	  at org.apache.dubbo.config.ServiceConfig.doExport(ServiceConfig.java:317)
	  - locked <0x848> (a org.apache.dubbo.config.spring.ServiceBean)
	  at org.apache.dubbo.config.ServiceConfig.export(ServiceConfig.java:216)
	  at org.apache.dubbo.config.spring.ServiceBean.onApplicationEvent(ServiceBean.java:123)
	  at org.apache.dubbo.config.spring.ServiceBean.onApplicationEvent(ServiceBean.java:49)
	  at org.springframework.context.event.SimpleApplicationEventMulticaster.doInvokeListener(SimpleApplicationEventMulticaster.java:172)
	  at org.springframework.context.event.SimpleApplicationEventMulticaster.invokeListener(SimpleApplicationEventMulticaster.java:165)
	  at org.springframework.context.event.SimpleApplicationEventMulticaster.multicastEvent(SimpleApplicationEventMulticaster.java:139)
	  at org.springframework.context.support.AbstractApplicationContext.publishEvent(AbstractApplicationContext.java:393)
	  at org.springframework.context.support.AbstractApplicationContext.publishEvent(AbstractApplicationContext.java:347)
	  at org.springframework.context.support.AbstractApplicationContext.finishRefresh(AbstractApplicationContext.java:883)
	  at org.springframework.context.support.AbstractApplicationContext.refresh(AbstractApplicationContext.java:546)
	  - locked <0xab7> (a java.lang.Object)
	  at org.springframework.context.support.ClassPathXmlApplicationContext.<init>(ClassPathXmlApplicationContext.java:139)
	  at org.springframework.context.support.ClassPathXmlApplicationContext.<init>(ClassPathXmlApplicationContext.java:83)
	  at cn.shuaijunlan.dubbo.learning.main.Main.main(Main.java:13)
```

关于Dubbo是如何基于Spring解析xml文件中配置的ServiceBean这里不再赘述，可以参考我之前写的文章[《基于Spring构建Dubbo源码分析》](https://shuaijunlan.github.io/2018/08/13/dubbo-basing-on-spring-framework-analysis/)，这里将从发布服务的起始点开始将其，我们来看下ServiceBean类的部分函数实现：

```java
//这个函数是实现了ApplicationListener接口的onApplicationEvent(E event)函数
//
@Override
public void onApplicationEvent(ContextRefreshedEvent event) {
    if (isDelay() && !isExported() && !isUnexported()) {
        if (logger.isInfoEnabled()) {
            logger.info("The service ready on spring started. service: " + getInterface());
        }
        //发布服务的起始点
        export();
    }
}
```

上面调用的export()函数的实现在其子类ServiceConfig中：

```java
//这是一个同步方法，保证了多线程环境下的安全性
public synchronized void export() {
    //获取export和delay属性
    if (provider != null) {
        if (export == null) {
            export = provider.getExport();
        }
        if (delay == null) {
            delay = provider.getDelay();
        }
    }
    if (export != null && !export) {
        return;
    }
	//判断是否延迟发布
    if (delay != null && delay > 0) {
        //延迟delay时间后执行
        delayExportExecutor.schedule(new Runnable() {
            @Override
            public void run() {
                doExport();
            }
        }, delay, TimeUnit.MILLISECONDS);
    } else {
        //发布服务
        doExport();
    }
}
//使用synchonized进行同步，保证了线程安全性
protected synchronized void doExport() {
    if (unexported) {
        throw new IllegalStateException("Already unexported!");
    }
    if (exported) {
        return;
    }
    //将exported状态改为true
    exported = true;
    if (interfaceName == null || interfaceName.length() == 0) {
        throw new IllegalStateException("<dubbo:service interface=\"\" /> interface not allow null!");
    }
    checkDefault();
    if (provider != null) {
        if (application == null) {
            application = provider.getApplication();
        }
        if (module == null) {
            module = provider.getModule();
        }
        if (registries == null) {
            registries = provider.getRegistries();
        }
        if (monitor == null) {
            monitor = provider.getMonitor();
        }
        if (protocols == null) {
            protocols = provider.getProtocols();
        }
    }
    if (module != null) {
        if (registries == null) {
            registries = module.getRegistries();
        }
        if (monitor == null) {
            monitor = module.getMonitor();
        }
    }
    if (application != null) {
        if (registries == null) {
            registries = application.getRegistries();
        }
        if (monitor == null) {
            monitor = application.getMonitor();
        }
    }
    if (ref instanceof GenericService) {
        interfaceClass = GenericService.class;
        if (StringUtils.isEmpty(generic)) {
            generic = Boolean.TRUE.toString();
        }
    } else {
        try {
            interfaceClass = Class.forName(interfaceName, true, Thread.currentThread()
                                           .getContextClassLoader());
        } catch (ClassNotFoundException e) {
            throw new IllegalStateException(e.getMessage(), e);
        }
        checkInterfaceAndMethods(interfaceClass, methods);
        checkRef();
        generic = Boolean.FALSE.toString();
    }
    if (local != null) {
        if ("true".equals(local)) {
            local = interfaceName + "Local";
        }
        Class<?> localClass;
        try {
            localClass = ClassHelper.forNameWithThreadContextClassLoader(local);
        } catch (ClassNotFoundException e) {
            throw new IllegalStateException(e.getMessage(), e);
        }
        if (!interfaceClass.isAssignableFrom(localClass)) {
            throw new IllegalStateException("The local implementation class " + localClass.getName() + " not implement interface " + interfaceName);
        }
    }
    if (stub != null) {
        if ("true".equals(stub)) {
            stub = interfaceName + "Stub";
        }
        Class<?> stubClass;
        try {
            stubClass = ClassHelper.forNameWithThreadContextClassLoader(stub);
        } catch (ClassNotFoundException e) {
            throw new IllegalStateException(e.getMessage(), e);
        }
        if (!interfaceClass.isAssignableFrom(stubClass)) {
            throw new IllegalStateException("The stub implementation class " + stubClass.getName() + " not implement interface " + interfaceName);
        }
    }
    checkApplication();
    checkRegistry();
    checkProtocol();
    appendProperties(this);
    checkStubAndMock(interfaceClass);
    if (path == null || path.length() == 0) {
        path = interfaceName;
    }
    doExportUrls();
    ProviderModel providerModel = new ProviderModel(getUniqueServiceName(), this, ref);
    ApplicationModel.initProviderModel(getUniqueServiceName(), providerModel);
}
```

在doExport()函数中调用了doExportUrls()方法，我们将进一步分析doExportUrls()方法：

```java
@SuppressWarnings({"unchecked", "rawtypes"})
private void doExportUrls() {
    //支持多个注册中心
    List<URL> registryURLs = loadRegistries(true);
    for (ProtocolConfig protocolConfig : protocols) {
        //执行doExportUrlsFor1Protocol方法
        doExportUrlsFor1Protocol(protocolConfig, registryURLs);
    }
}

private void doExportUrlsFor1Protocol(ProtocolConfig protocolConfig, List<URL> registryURLs) {
    String name = protocolConfig.getName();
    if (name == null || name.length() == 0) {
        name = "dubbo";
    }

    Map<String, String> map = new HashMap<String, String>();
    map.put(Constants.SIDE_KEY, Constants.PROVIDER_SIDE);
    map.put(Constants.DUBBO_VERSION_KEY, Version.getProtocolVersion());
    map.put(Constants.TIMESTAMP_KEY, String.valueOf(System.currentTimeMillis()));
    if (ConfigUtils.getPid() > 0) {
        map.put(Constants.PID_KEY, String.valueOf(ConfigUtils.getPid()));
    }
    appendParameters(map, application);
    appendParameters(map, module);
    appendParameters(map, provider, Constants.DEFAULT_KEY);
    appendParameters(map, protocolConfig);
    appendParameters(map, this);
    if (methods != null && !methods.isEmpty()) {
        for (MethodConfig method : methods) {
            appendParameters(map, method, method.getName());
            String retryKey = method.getName() + ".retry";
            if (map.containsKey(retryKey)) {
                String retryValue = map.remove(retryKey);
                if ("false".equals(retryValue)) {
                    map.put(method.getName() + ".retries", "0");
                }
            }
            List<ArgumentConfig> arguments = method.getArguments();
            if (arguments != null && !arguments.isEmpty()) {
                for (ArgumentConfig argument : arguments) {
                    // convert argument type
                    if (argument.getType() != null && argument.getType().length() > 0) {
                        Method[] methods = interfaceClass.getMethods();
                        // visit all methods
                        if (methods != null && methods.length > 0) {
                            for (int i = 0; i < methods.length; i++) {
                                String methodName = methods[i].getName();
                                // target the method, and get its signature
                                if (methodName.equals(method.getName())) {
                                    Class<?>[] argtypes = methods[i].getParameterTypes();
                                    // one callback in the method
                                    if (argument.getIndex() != -1) {
                                        if (argtypes[argument.getIndex()].getName().equals(argument.getType())) {
                                            appendParameters(map, argument, method.getName() + "." + argument.getIndex());
                                        } else {
                                            throw new IllegalArgumentException("argument config error : the index attribute and type attribute not match :index :" + argument.getIndex() + ", type:" + argument.getType());
                                        }
                                    } else {
                                        // multiple callbacks in the method
                                        for (int j = 0; j < argtypes.length; j++) {
                                            Class<?> argclazz = argtypes[j];
                                            if (argclazz.getName().equals(argument.getType())) {
                                                appendParameters(map, argument, method.getName() + "." + j);
                                                if (argument.getIndex() != -1 && argument.getIndex() != j) {
                                                    throw new IllegalArgumentException("argument config error : the index attribute and type attribute not match :index :" + argument.getIndex() + ", type:" + argument.getType());
                                                }
                                            }
                                        }
                                    }
                                }
                            }
                        }
                    } else if (argument.getIndex() != -1) {
                        appendParameters(map, argument, method.getName() + "." + argument.getIndex());
                    } else {
                        throw new IllegalArgumentException("argument config must set index or type attribute.eg: <dubbo:argument index='0' .../> or <dubbo:argument type=xxx .../>");
                    }

                }
            }
        } // end of methods for
    }

    if (ProtocolUtils.isGeneric(generic)) {
        map.put(Constants.GENERIC_KEY, generic);
        map.put(Constants.METHODS_KEY, Constants.ANY_VALUE);
    } else {
        String revision = Version.getVersion(interfaceClass, version);
        if (revision != null && revision.length() > 0) {
            map.put("revision", revision);
        }

        String[] methods = Wrapper.getWrapper(interfaceClass).getMethodNames();
        if (methods.length == 0) {
            logger.warn("NO method found in service interface " + interfaceClass.getName());
            map.put(Constants.METHODS_KEY, Constants.ANY_VALUE);
        } else {
            map.put(Constants.METHODS_KEY, StringUtils.join(new HashSet<String>(Arrays.asList(methods)), ","));
        }
    }
    if (!ConfigUtils.isEmpty(token)) {
        if (ConfigUtils.isDefault(token)) {
            map.put(Constants.TOKEN_KEY, UUID.randomUUID().toString());
        } else {
            map.put(Constants.TOKEN_KEY, token);
        }
    }
    if (Constants.LOCAL_PROTOCOL.equals(protocolConfig.getName())) {
        protocolConfig.setRegister(false);
        map.put("notify", "false");
    }
    // export service
    String contextPath = protocolConfig.getContextpath();
    if ((contextPath == null || contextPath.length() == 0) && provider != null) {
        contextPath = provider.getContextpath();
    }

    String host = this.findConfigedHosts(protocolConfig, registryURLs, map);
    Integer port = this.findConfigedPorts(protocolConfig, name, map);
    URL url = new URL(name, host, port, (contextPath == null || contextPath.length() == 0 ? "" : contextPath + "/") + path, map);

    if (ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class)
        .hasExtension(url.getProtocol())) {
        url = ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class)
            .getExtension(url.getProtocol()).getConfigurator(url).configure(url);
    }

    String scope = url.getParameter(Constants.SCOPE_KEY);
    // don't export when none is configured
    if (!Constants.SCOPE_NONE.equalsIgnoreCase(scope)) {

        // export to local if the config is not remote (export to remote only when config is remote)
        if (!Constants.SCOPE_REMOTE.equalsIgnoreCase(scope)) {
            //本地暴露（分析1）
            exportLocal(url);
        }
        // export to remote if the config is not local (export to local only when config is local)
        //如果配置不是local则暴露为远程服务
        if (!Constants.SCOPE_LOCAL.equalsIgnoreCase(scope)) {
            if (logger.isInfoEnabled()) {
                logger.info("Export dubbo service " + interfaceClass.getName() + " to url " + url);
            }
            if (registryURLs != null && !registryURLs.isEmpty()) {
                //多个注册中心
                for (URL registryURL : registryURLs) {
                    url = url.addParameterIfAbsent(Constants.DYNAMIC_KEY, registryURL.getParameter(Constants.DYNAMIC_KEY));
                    URL monitorUrl = loadMonitor(registryURL);
                    if (monitorUrl != null) {
                        url = url.addParameterAndEncoded(Constants.MONITOR_KEY, monitorUrl.toFullString());
                    }
                    if (logger.isInfoEnabled()) {
                        logger.info("Register dubbo service " + interfaceClass.getName() + " url " + url + " to registry " + registryURL);
                    }

                    // For providers, this is used to enable custom proxy to generate invoker
                    String proxy = url.getParameter(Constants.PROXY_KEY);
                    if (StringUtils.isNotEmpty(proxy)) {
                        registryURL = registryURL.addParameter(Constants.PROXY_KEY, proxy);
                    }
					//根据Java SPI机制得到ProxyFactory$Adaptive类的实例proxyFactory
                    Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(Constants.EXPORT_KEY, url.toFullString()));
                    //获取包装类??
                    DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);
					//调用Protocol$Adaptive的export()方法
                    Exporter<?> exporter = protocol.export(wrapperInvoker);
                    exporters.add(exporter);
                }
            } else {
                //没有注册中心，只在本机IP打开服务端口生成服务代理，并不注册到注册中心
                Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, url);
                DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);

                Exporter<?> exporter = protocol.export(wrapperInvoker);
                exporters.add(exporter);
            }
        }
    }
    this.urls.add(url);
}
```

#### 本地服务发布过程

首先是进入exportLocal()函数：

```java
private void exportLocal(URL url) {
    if (!Constants.LOCAL_PROTOCOL.equalsIgnoreCase(url.getProtocol())) {
        URL local = URL.valueOf(url.toFullString())
            .setProtocol(Constants.LOCAL_PROTOCOL)
            .setHost(LOCALHOST)
            .setPort(0);
        ServiceClassHolder.getInstance().pushServiceClass(getServiceClass(ref));
        //这是核心步骤，先是执行proxyFactory.getInvoker()方法生成invoker，然后是执行protocol.export()方法暴露服务，下面将会分别介绍
        Exporter<?> exporter = protocol.export(
            proxyFactory.getInvoker(ref, (Class) interfaceClass, local));
        exporters.add(exporter);
        logger.info("Export dubbo service " + interfaceClass.getName() + " to local registry");
    }
}
```

##### 分析proxyFactory.getInvoker()过程

通过Dubbo SPI机制（详见[《Dubbo SPI机制源码分析》](https://shuaijunlan.github.io/2018/08/09/dubbo-spi-analysis/)），proxyFactory是ProxyFactory$Adaptive类的实例，我们来看它的getInvoker()方法：

```java
//传入三个参数，分别是ref、interfaceClass和url
public org.apache.dubbo.rpc.Invoker getInvoker(java.lang.Object arg0, java.lang.Class arg1, org.apache.dubbo.common.URL arg2) throws org.apache.dubbo.rpc.RpcException {
    if (arg2 == null) throw new IllegalArgumentException("url == null");
    org.apache.dubbo.common.URL url = arg2;
    //默认是使用Javassist生成代理
    String extName = url.getParameter("proxy", "javassist");
    if (extName == null)
        throw new IllegalStateException("Fail to get extension(org.apache.MEAT-INF.dubbo.rpc.ProxyFactory) name from url(" + url.toString() + ") use keys([proxy])");
    //根据Dubbo SPI机制得到JavassistProxyFactory扩展类
    org.apache.dubbo.rpc.ProxyFactory extension = (org.apache.dubbo.rpc.ProxyFactory) ExtensionLoader.getExtensionLoader(org.apache.dubbo.rpc.ProxyFactory.class).getExtension(extName);
    return extension.getInvoker(arg0, arg1, arg2);
}
```

然后调用extension.getInvoker()方法，这里的extension，默认是JavassistProxyFactory类的实例（也是基于Java SPI机制），然后调用它的getInvoker()方法：

```java
@Override
public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) {
    // TODO Wrapper cannot handle this scenario correctly: the classname contains '$'
    //获取包装类,具体代码是怎样的？
    final Wrapper wrapper = Wrapper.getWrapper(proxy.getClass().getName().indexOf('$') < 0 ? proxy.getClass() : type);
    return new AbstractProxyInvoker<T>(proxy, type, url) {
        @Override
        protected Object doInvoke(T proxy, String methodName,
                                  Class<?>[] parameterTypes,
                                  Object[] arguments) throws Throwable {
            return wrapper.invokeMethod(proxy, methodName, parameterTypes, arguments);
        }
    };
}
```

这里以文章开头的Demo为例，通过Wrapper.getWrapper()返回的类代码，这里需要代码hack：

```java
package com.alibaba.dubbo.common.bytecode;

import com.alibaba.dubbo.common.DemoService;
import java.lang.reflect.InvocationTargetException;
import java.util.Map;

public class Wrapper0 extends Wrapper
  implements ClassGenerator.DC
{
  public static String[] pns;
  public static Map pts;
  public static String[] mns;
  public static String[] dmns;
  public static Class[] mts0;

  public String[] getPropertyNames()
  {
    return pns;
  }

  public boolean hasProperty(String paramString)
  {
    return pts.containsKey(paramString);
  }

  public Class getPropertyType(String paramString)
  {
    return (Class)pts.get(paramString);
  }

  public String[] getMethodNames()
  {
    return mns;
  }

  public String[] getDeclaredMethodNames()
  {
    return dmns;
  }

  public void setPropertyValue(Object paramObject1, String paramString, Object paramObject2)
  {
    try
    {
      DemoService localDemoService = (DemoService)paramObject1;
    }
    catch (Throwable localThrowable)
    {
      throw new IllegalArgumentException(localThrowable);
    }
    throw new NoSuchPropertyException("Not found property \"" + paramString + "\" filed or setter method in class com.alibaba.dubbo.common.DemoService.");
  }

  public Object getPropertyValue(Object paramObject, String paramString)
  {
    try
    {
      DemoService localDemoService = (DemoService)paramObject;
    }
    catch (Throwable localThrowable)
    {
      throw new IllegalArgumentException(localThrowable);
    }
    throw new NoSuchPropertyException("Not found property \"" + paramString + "\" filed or setter method in class com.alibaba.dubbo.common.DemoService.");
  }
   
  //(**看这里，关键方法实现****)
  public Object invokeMethod(Object paramObject, String paramString, Class[] paramArrayOfClass, Object[] paramArrayOfObject)
    throws InvocationTargetException
  {
    DemoService localDemoService;
    try
    {
    //赋值执行实例，这里是接口实现类，DemoServiceImpl对象
      localDemoService = (DemoService)paramObject;
    }
    catch (Throwable localThrowable1)
    {
      throw new IllegalArgumentException(localThrowable1);
    }
    try
    {
    //根据传入的要调用的方法名paramString,方法参数值，调用执行实例方法
      if (("sayHello".equals(paramString)) || (paramArrayOfClass.length == 1))
        return localDemoService.sayHello((String)paramArrayOfObject[0]);
    }
    catch (Throwable localThrowable2)
    {
      throw new InvocationTargetException(localThrowable2);
    }
    throw new NoSuchMethodException("Not found method \"" + paramString + "\" in class com.alibaba.dubbo.common.DemoService.");
  }
}
```

到这就比较清楚了解具体的代理的过程了。

##### 分析protocol.export()过程

上面的过程已经生成好了invoker对象，接下来就要通过Protocol$Adaptive的export()方法暴露服务：

```java
public org.apache.dubbo.rpc.Exporter export(org.apache.dubbo.rpc.Invoker arg0) throws org.apache.dubbo.rpc.RpcException {
    if (arg0 == null) throw new IllegalArgumentException("org.apache.MEAT-INF.dubbo.rpc.Invoker argument == null");
    if (arg0.getUrl() == null)
        throw new IllegalArgumentException("org.apache.MEAT-INF.dubbo.rpc.Invoker argument getUrl() == null");
    org.apache.dubbo.common.URL url = arg0.getUrl();
    String extName = (url.getProtocol() == null ? "MEAT-INF.dubbo" : url.getProtocol());
    if (extName == null)
        throw new IllegalStateException("Fail to get extension(org.apache.MEAT-INF.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");
    //因为这是本地服务发布，因此protocol为injvm
    //所以这里会走到InjvmProtocol的export()方法
    org.apache.dubbo.rpc.Protocol extension = (org.apache.dubbo.rpc.Protocol) ExtensionLoader.getExtensionLoader(org.apache.dubbo.rpc.Protocol.class).getExtension(extName);
    return extension.export(arg0);
}
```

看下InjvmProtocol的export()方法

```java
@Override
public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
    //返回InjvmExporter对象
    return new InjvmExporter<T>(invoker, invoker.getUrl().getServiceKey(), exporterMap);
}
```

InjvmExporter的构造函数：

```java
InjvmExporter(Invoker<T> invoker, String key, Map<String, Exporter<?>> exporterMap) {
    super(invoker);
    this.key = key;
    this.exporterMap = exporterMap;
    //存的形式，serviceKey:自身(exporter) put到map关联起来，这样可以通过servciekey找到exporterMap然后找到invoker
    exporterMap.put(key, this);
}
```

这里的exporterMap是由InjvmProtocol实例拥有，而InjvmProtocol又是单例的，因为InjvmProtocol类有如下实例和方法：

```java
//静态自身成员变量
private static InjvmProtocol INSTANCE;
//构造方法，把自己赋值给INSTANCE对象
public InjvmProtocol() {
    INSTANCE = this;
}
```

所以exporterMap对象也是单例的，同时这里顺便看下InjvmProtocol的refer()方法，本地服务的引用查找也是通过自身的exporterMap对象：

```java
@Override
public <T> Invoker<T> refer(Class<T> serviceType, URL url) throws RpcException {
    //把exporterMap对象赋值给InjvmInvoker
    return new InjvmInvoker<T>(serviceType, url, url.getServiceKey(), exporterMap);
}
//具体查找过程
@Override
public Result doInvoke(Invocation invocation) throws Throwable {
    //通过exporterMap获取exporter
    Exporter<?> exporter = InjvmProtocol.getExporter(exporterMap, getUrl());
    if (exporter == null) {
        throw new RpcException("Service [" + key + "] not found.");
    }
    RpcContext.getContext().setRemoteAddress(NetUtils.LOCALHOST, 0);
    return exporter.getInvoker().invoke(invocation);
}
```

以上的所有步骤就是本地服务的发布和引用过程。

#### 远程服务发布

上面调用了proFactory的getInvoker()方法，我们来看一下`ProxyFactory$Adaptive.getInvoker()`的代码：

```java
//传入三个参数，分别是ref、interfaceClass和url
public org.apache.dubbo.rpc.Invoker getInvoker(java.lang.Object arg0, java.lang.Class arg1, org.apache.dubbo.common.URL arg2) throws org.apache.dubbo.rpc.RpcException {
    if (arg2 == null) throw new IllegalArgumentException("url == null");
    org.apache.dubbo.common.URL url = arg2;
    //默认是使用Javassist生成代理
    String extName = url.getParameter("proxy", "javassist");
    if (extName == null)
        throw new IllegalStateException("Fail to get extension(org.apache.MEAT-INF.dubbo.rpc.ProxyFactory) name from url(" + url.toString() + ") use keys([proxy])");
    //根据Dubbo SPI机制得到JavassistProxyFactory扩展类
    org.apache.dubbo.rpc.ProxyFactory extension = (org.apache.dubbo.rpc.ProxyFactory) ExtensionLoader.getExtensionLoader(org.apache.dubbo.rpc.ProxyFactory.class).getExtension(extName);
    return extension.getInvoker(arg0, arg1, arg2);
}
```

再来看一下`JavassistProxyFactory.getInvoker()`方法：

```java
@Override
public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) {
    // TODO Wrapper cannot handle this scenario correctly: the classname contains '$'
    //获取包装类,具体代码是怎样的？
    final Wrapper wrapper = Wrapper.getWrapper(proxy.getClass().getName().indexOf('$') < 0 ? proxy.getClass() : type);
    return new AbstractProxyInvoker<T>(proxy, type, url) {
        @Override
        protected Object doInvoke(T proxy, String methodName,
                                  Class<?>[] parameterTypes,
                                  Object[] arguments) throws Throwable {
            return wrapper.invokeMethod(proxy, methodName, parameterTypes, arguments);
        }
    };
}
```

在doExportUrlsFor1Protocol()方法中还调用了`Protocol$Adaptive.export()`方法：

```java
public org.apache.dubbo.rpc.Exporter export(org.apache.dubbo.rpc.Invoker arg0) throws org.apache.dubbo.rpc.RpcException {
    if (arg0 == null) throw new IllegalArgumentException("org.apache.MEAT-INF.dubbo.rpc.Invoker argument == null");
    if (arg0.getUrl() == null)
        throw new IllegalArgumentException("org.apache.MEAT-INF.dubbo.rpc.Invoker argument getUrl() == null");
    org.apache.dubbo.common.URL url = arg0.getUrl();
    String extName = (url.getProtocol() == null ? "MEAT-INF.dubbo" : url.getProtocol());
    if (extName == null)
        throw new IllegalStateException("Fail to get extension(org.apache.MEAT-INF.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");
    org.apache.dubbo.rpc.Protocol extension = (org.apache.dubbo.rpc.Protocol) ExtensionLoader.getExtensionLoader(org.apache.dubbo.rpc.Protocol.class).getExtension(extName);
    return extension.export(arg0);
}
```

根据Dubbo SPI机制可以看出调用`RegistryProtocol.export()`方法，Protocol还定义了ProtocolFilterWrapper、QosProtocolWrapper和ProtocolListenerWrapper三个Wrapper扩展点，根据ExtensionLoader的加载规则，他会返回`ProtocolFilterWrapper->QosProtocolWrapper->ProtocolListenerWrapper->RegistryProtocol`（对象链调用顺序还待进一步求证）对象链：

```java
@Override
public <T> Exporter<T> export(final Invoker<T> originInvoker) throws RpcException {
    //export invoker
    //暴露invoker（暴露服务过程从这里开始，看doLocalExport()）
    final ExporterChangeableWrapper<T> exporter = doLocalExport(originInvoker);
	
    URL registryUrl = getRegistryUrl(originInvoker);

    //registry provider
    //获取对应注册中心操作对象
    final Registry registry = getRegistry(originInvoker);
    //获取要注册到注册中心的地址
    final URL registeredProviderUrl = getRegisteredProviderUrl(originInvoker);

    //to judge to delay publish whether or not
    boolean register = registeredProviderUrl.getParameter("register", true);

    ProviderConsumerRegTable.registerProvider(originInvoker, registryUrl, registeredProviderUrl);
	//判断是否注册服务
    if (register) {
        //执行注册(服务注册过程从这里开始)
        register(registryUrl, registeredProviderUrl);
        ProviderConsumerRegTable.getProviderWrapper(originInvoker).setReg(true);
    }

    // Subscribe the override data
    // FIXME When the provider subscribes, it will affect the scene : a certain JVM exposes the service and call the same service. Because the subscribed is cached key with the name of the service, it causes the subscription information to cover.
    final URL overrideSubscribeUrl = getSubscribedOverrideUrl(registeredProviderUrl);
    final OverrideListener overrideSubscribeListener = new OverrideListener(overrideSubscribeUrl, originInvoker);
    overrideListeners.put(overrideSubscribeUrl, overrideSubscribeListener);
    registry.subscribe(overrideSubscribeUrl, overrideSubscribeListener);
    //Ensure that a new exporter instance is returned every time export
    //保证每次export都返回一个新的exporter实例
    return new DestroyableExporter<T>(exporter, originInvoker, overrideSubscribeUrl, registeredProviderUrl);
}
```

因此有如下的调用栈：

```
at org.apache.dubbo.registry.integration.RegistryProtocol.export(RegistryProtocol.java:133)
at org.apache.dubbo.rpc.protocol.ProtocolListenerWrapper.export(ProtocolListenerWrapper.java:55)
at org.apache.dubbo.qos.protocol.QosProtocolWrapper.export(QosProtocolWrapper.java:61)
at org.apache.dubbo.rpc.protocol.ProtocolFilterWrapper.export(ProtocolFilterWrapper.java:98)
at org.apache.dubbo.rpc.Protocol$Adaptive.export(Protocol$Adaptive.java:-1)
```

继续看执行服务暴露的函数`doLocalExport()`:

```java
@SuppressWarnings("unchecked")
private <T> ExporterChangeableWrapper<T> doLocalExport(final Invoker<T> originInvoker) {
    //通过原始originInvoker构造缓存key
    String key = getCacheKey(originInvoker);
    ExporterChangeableWrapper<T> exporter = (ExporterChangeableWrapper<T>) bounds.get(key);
    //没有缓存，走具体暴露流程
    if (exporter == null) {
        synchronized (bounds) {
            exporter = (ExporterChangeableWrapper<T>) bounds.get(key);
            if (exporter == null) {
                //InvokerDelegete是RegistryProtocol类的静态内部类，继承自InvokerWrapper，
		    	//通过构造器赋值持有代理originInvoker和服务暴露协议url对象,算是包装一层
                //而url 是通过getProviderUrl(originInvoker)返回的，此时url的协议已是dubbo，即服务暴露的协议
                final Invoker<?> invokerDelegete = new InvokerDelegete<T>(originInvoker, getProviderUrl(originInvoker));
                
                //ExporterChangeableWrapper是RegistryProtocol的私有内部类实现了Exporter接口。
                //通过调用它的构造方法(Exporter<T> exporter, Invoker<T> originInvoker)构造exporterWrapper实例
		    	//而这里传入的exporter是通过(Exporter<T>) protocol.export(invokerDelegete)语句创建
		    	//由上一步知道，这里的invokerDelegete里url属性的protocol协议已经是dubbo
                //下面具体看下protocol.export(invokerDelegete)方法。
                exporter = new ExporterChangeableWrapper<T>((Exporter<T>) protocol.export(invokerDelegete), originInvoker);
                bounds.put(key, exporter);
            }
        }
    }
    return exporter;
}
```

再继续往下看，通过调用Protocol$Adaptive类的export()方法，然后再调用DubboProtocol的export()方法，同理在这里也会生成`ProtocolFilterWrapper->QosProtocolWrapper->ProtocolListenerWrapper->DubboProtocol`对象链：

```java
@Override
public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
    URL url = invoker.getUrl();

    // export service.
    //获取service key
    //key的组成group/service:version:port
    String key = serviceKey(url);
    //构造服务的exporter
    //如同InjvmProtocol一样，DubboProtocol也是单例的，所以这里exporterMap也是单例的
    DubboExporter<T> exporter = new DubboExporter<T>(invoker, key, exporterMap);
    //通过key放入exporterMap，把持有invoker的exporter 和serviceKey关联
    //这个在后面服务调用时，可以通过key找到对应的exporter进而找到invoker提供服务
    exporterMap.put(key, exporter);

    //export an stub service for dispatching event
    Boolean isStubSupportEvent = url.getParameter(Constants.STUB_EVENT_KEY, Constants.DEFAULT_STUB_EVENT);
    Boolean isCallbackservice = url.getParameter(Constants.IS_CALLBACK_SERVICE, false);
    if (isStubSupportEvent && !isCallbackservice) {
        String stubServiceMethods = url.getParameter(Constants.STUB_EVENT_METHODS_KEY);
        if (stubServiceMethods == null || stubServiceMethods.length() == 0) {
            if (logger.isWarnEnabled()) {
                logger.warn(new IllegalStateException("consumer [" + url.getParameter(Constants.INTERFACE_KEY) +
                                                      "], has set stubproxy support event ,but no stub methods founded."));
            }
        } else {
            stubServiceMethodsMap.put(url.getServiceKey(), stubServiceMethods);
        }
    }
	//根据url开启一个服务，比如绑定端口，开始接受请求
    openServer(url);
    optimizeSerialization(url);
    return exporter;
}
```

到这里就可以对应下面的调用栈：

```
at org.apache.dubbo.rpc.protocol.dubbo.DubboProtocol.export(DubboProtocol.java:267)
at org.apache.dubbo.rpc.protocol.ProtocolListenerWrapper.export(ProtocolListenerWrapper.java:57)
at org.apache.dubbo.qos.protocol.QosProtocolWrapper.export(QosProtocolWrapper.java:63)
at org.apache.dubbo.rpc.protocol.ProtocolFilterWrapper.export(ProtocolFilterWrapper.java:100)
at org.apache.dubbo.rpc.Protocol$Adaptive.export(Protocol$Adaptive.java:-1)
at org.apache.dubbo.registry.integration.RegistryProtocol.doLocalExport(RegistryProtocol.java:170)
```

再继续看openServer()方法

```java
private void openServer(URL url) {
    // find server.
    //key=host:port 用于定位server
    String key = url.getAddress();
    //client can export a service which's only for server to invoke
    //client也可以暴露一个只有server可以调用的服务
    boolean isServer = url.getParameter(Constants.IS_SERVER_KEY, true);
    if (isServer) {
        //服务实例放到serverMap中，key是host:port
        //这里serverMap也是单例的
        ExchangeServer server = serverMap.get(key);
        if (server == null) {
            synchronized (this) {
                server = serverMap.get(key);
                if (server == null) {
                    //通过createServer(url)方法获取server
                    serverMap.put(key, createServer(url));
                }
            }
        } else {
            // server supports reset, use together with override
            //server支持reset，配合override使用
            server.reset(url);
        }
    }
}
```

再继续看createServer()代码：

```java
private ExchangeServer createServer(URL url) {
    // send readonly event when server closes, it's enabled by default
    //默认开启server关闭时关闭readonly事件
    url = url.addParameterIfAbsent(Constants.CHANNEL_READONLYEVENT_SENT_KEY, Boolean.TRUE.toString());
    // enable heartbeat by default
    //默认开启heartbeat
    url = url.addParameterIfAbsent(Constants.HEARTBEAT_KEY, String.valueOf(Constants.DEFAULT_HEARTBEAT));
    String str = url.getParameter(Constants.SERVER_KEY, Constants.DEFAULT_REMOTING_SERVER);
	//通过server key检查是否是dubbo目前spi扩展支持的传输框架，默认是netty
    if (str != null && str.length() > 0 && !ExtensionLoader.getExtensionLoader(Transporter.class).hasExtension(str))
        throw new RpcException("Unsupported server type: " + str + ", url: " + url);
	//通过codec key获取编码方法，默认是dubbo
    url = url.addParameter(Constants.CODEC_KEY, DubboCodec.NAME);
    ExchangeServer server;
    try {
        //构造具体服务实例，
	    //Exchangers是门面类，里面封装了具体交换层实现，并调用它的bind方法
        server = Exchangers.bind(url, requestHandler);
    } catch (RemotingException e) {
        throw new RpcException("Fail to start server(url: " + url + ") " + e.getMessage(), e);
    }
    //这里会验证一下客户端传输实现
    //如果没有对应的实现，会抛出异常
    str = url.getParameter(Constants.CLIENT_KEY);
    if (str != null && str.length() > 0) {
        Set<String> supportedTypes = ExtensionLoader.getExtensionLoader(Transporter.class).getSupportedExtensions();
        if (!supportedTypes.contains(str)) {
            throw new RpcException("Unsupported client type: " + str);
        }
    }
    return server;
}
```

继续看Exchanges类的bind()方法：

```java
public static ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {
    if (url == null) {
        throw new IllegalArgumentException("url == null");
    }
    if (handler == null) {
        throw new IllegalArgumentException("handler == null");
    }
    url = url.addParameterIfAbsent(Constants.CODEC_KEY, "exchange");
    return getExchanger(url).bind(url, handler);
}

public static Exchanger getExchanger(URL url) {
    String type = url.getParameter(Constants.EXCHANGER_KEY, Constants.DEFAULT_EXCHANGER);
    return getExchanger(type);
}

public static Exchanger getExchanger(String type) {
    return ExtensionLoader.getExtensionLoader(Exchanger.class).getExtension(type);
}

//HeaderExchanger类的bind()方法
@Override
public ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {
    return new HeaderExchangeServer(Transporters.bind(url, new DecodeHandler(new HeaderExchangeHandler(handler))));
}
```

继续跟进Transporters.bind()方法：

```java
public static Server bind(URL url, ChannelHandler... handlers) throws RemotingException {
    if (url == null) {
        throw new IllegalArgumentException("url == null");
    }
    if (handlers == null || handlers.length == 0) {
        throw new IllegalArgumentException("handlers == null");
    }
    ChannelHandler handler;
    if (handlers.length == 1) {
        handler = handlers[0];
    } else {
        handler = new ChannelHandlerDispatcher(handlers);
    }
    //根据Dubbo SPI机制，这里走NettyTransporter.bind()方法
    return getTransporter().bind(url, handler);
}
public static Transporter getTransporter() {
    return ExtensionLoader.getExtensionLoader(Transporter.class).getAdaptiveExtension();
}

//NettyTransporter的bind方法
public Server bind(URL url, ChannelHandler listener) throws RemotingException {
    //可以看到这里是NettyServer实例
    return new NettyServer(url, listener);
}

//NettyServer构造器
public NettyServer(URL url, ChannelHandler handler) throws RemotingException {
    //调用父类AbstractServer构造器
    //注意下这里的ChannelHandlers.wrap()方法，生成MultiMessageHandler->HeartbeatHandler->AllChannelHandler的调用链
    super(url, ChannelHandlers.wrap(handler, ExecutorUtil.setThreadName(url, SERVER_THREAD_POOL_NAME)));
}
```

看一下父类AbstractServer()的构造函数：

```java
public AbstractServer(URL url, ChannelHandler handler) throws RemotingException {
    super(url, handler);
    localAddress = getUrl().toInetSocketAddress();

    String bindIp = getUrl().getParameter(Constants.BIND_IP_KEY, getUrl().getHost());
    int bindPort = getUrl().getParameter(Constants.BIND_PORT_KEY, getUrl().getPort());
    if (url.getParameter(Constants.ANYHOST_KEY, false) || NetUtils.isInvalidLocalHost(bindIp)) {
        bindIp = NetUtils.ANYHOST;
    }
    bindAddress = new InetSocketAddress(bindIp, bindPort);
    this.accepts = url.getParameter(Constants.ACCEPTS_KEY, Constants.DEFAULT_ACCEPTS);
    this.idleTimeout = url.getParameter(Constants.IDLE_TIMEOUT_KEY, Constants.DEFAULT_IDLE_TIMEOUT);
    try {
        //打开端口，启动服务
        doOpen();
        if (logger.isInfoEnabled()) {
            logger.info("Start " + getClass().getSimpleName() + " bind " + getBindAddress() + ", export " + getLocalAddress());
        }
    } catch (Throwable t) {
        throw new RemotingException(url.toInetSocketAddress(), null, "Failed to bind " + getClass().getSimpleName()
                                    + " on " + getLocalAddress() + ", cause: " + t.getMessage(), t);
    }
    //fixme replace this with better method
    DataStore dataStore = ExtensionLoader.getExtensionLoader(DataStore.class).getDefaultExtension();
    executor = (ExecutorService) dataStore.get(Constants.EXECUTOR_SERVICE_COMPONENT_KEY, Integer.toString(url.getPort()));
}
```

再来看NettyServer的doOpen()方法：

```java
@Override
protected void doOpen() throws Throwable {
    bootstrap = new ServerBootstrap();

    bossGroup = new NioEventLoopGroup(1, new DefaultThreadFactory("NettyServerBoss", true));
    workerGroup = new NioEventLoopGroup(getUrl().getPositiveParameter(Constants.IO_THREADS_KEY, Constants.DEFAULT_IO_THREADS),
                                        new DefaultThreadFactory("NettyServerWorker", true));

    final NettyServerHandler nettyServerHandler = new NettyServerHandler(getUrl(), this);
    channels = nettyServerHandler.getChannels();

    bootstrap.group(bossGroup, workerGroup)
        .channel(NioServerSocketChannel.class)
        .childOption(ChannelOption.TCP_NODELAY, Boolean.TRUE)
        .childOption(ChannelOption.SO_REUSEADDR, Boolean.TRUE)
        .childOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT)
        .childHandler(new ChannelInitializer<NioSocketChannel>() {
            @Override
            protected void initChannel(NioSocketChannel ch) throws Exception {
                NettyCodecAdapter adapter = new NettyCodecAdapter(getCodec(), getUrl(), NettyServer.this);
                ch.pipeline()//.addLast("logging",new LoggingHandler(LogLevel.INFO))//for debug
                    .addLast("decoder", adapter.getDecoder())
                    .addLast("encoder", adapter.getEncoder())
                    .addLast("handler", nettyServerHandler);
            }
        });
    // bind
    //bind地址，开启端口
    ChannelFuture channelFuture = bootstrap.bind(getBindAddress());
    channelFuture.syncUninterruptibly();
    channel = channelFuture.channel();

}
```

分析到这里可以对应如下的调用栈：

```java
at org.apache.dubbo.remoting.transport.netty4.NettyServer.doOpen(NettyServer.java:97)
    at org.apache.dubbo.remoting.transport.AbstractServer.<init>(AbstractServer.java:63)
    at org.apache.dubbo.remoting.transport.netty4.NettyServer.<init>(NettyServer.java:65)
    at org.apache.dubbo.remoting.transport.netty4.NettyTransporter.bind(NettyTransporter.java:32)
    at org.apache.dubbo.remoting.Transporter$Adaptive.bind(Transporter$Adaptive.java:-1)
    at org.apache.dubbo.remoting.Transporters.bind(Transporters.java:56)
    at org.apache.dubbo.remoting.exchange.support.header.HeaderExchanger.bind(HeaderExchanger.java:44)
    at org.apache.dubbo.remoting.exchange.Exchangers.bind(Exchangers.java:70)
    at org.apache.dubbo.rpc.protocol.dubbo.DubboProtocol.createServer(DubboProtocol.java:306)
    at org.apache.dubbo.rpc.protocol.dubbo.DubboProtocol.openServer(DubboProtocol.java:283)
```

分析到这里Dubbo服务提供者服务发布过程源码分析已经完成了，下面将继续分析服务注册过程。

### Dubbo服务注册过程

在之前的分析中，我们知道注册服务的过程是从RegistryProtocol的export()方法开始的，我们来看一下export()方法：

```java
@Override
public <T> Exporter<T> export(final Invoker<T> originInvoker) throws RpcException {
    //export invoker
    final ExporterChangeableWrapper<T> exporter = doLocalExport(originInvoker);

    URL registryUrl = getRegistryUrl(originInvoker);

    //registry provider
    final Registry registry = getRegistry(originInvoker);
    final URL registeredProviderUrl = getRegisteredProviderUrl(originInvoker);

    //to judge to delay publish whether or not
    boolean register = registeredProviderUrl.getParameter("register", true);

    ProviderConsumerRegTable.registerProvider(originInvoker, registryUrl, registeredProviderUrl);

    if (register) {
        //注册服务从这里开始
        register(registryUrl, registeredProviderUrl);
        ProviderConsumerRegTable.getProviderWrapper(originInvoker).setReg(true);
    }

    // Subscribe the override data
    // FIXME When the provider subscribes, it will affect the scene : a certain JVM exposes the service and call the same service. Because the subscribed is cached key with the name of the service, it causes the subscription information to cover.
    final URL overrideSubscribeUrl = getSubscribedOverrideUrl(registeredProviderUrl);
    final OverrideListener overrideSubscribeListener = new OverrideListener(overrideSubscribeUrl, originInvoker);
    overrideListeners.put(overrideSubscribeUrl, overrideSubscribeListener);
    registry.subscribe(overrideSubscribeUrl, overrideSubscribeListener);
    //Ensure that a new exporter instance is returned every time export
    return new DestroyableExporter<T>(exporter, originInvoker, overrideSubscribeUrl, registeredProviderUrl);
}
```

我们再继续看register()方法：

```java
public void register(URL registryUrl, URL registedProviderUrl) {
    //registryFactory是由Dubbo SPI机制生成的RegistryFactory$Adaptive的实例
    //调用其的getRegistry()方法获得registry
    Registry registry = registryFactory.getRegistry(registryUrl);
    registry.register(registedProviderUrl);
}
```

我们来看一下RegistryFactory$Adaptive类的getRegistry()方法：

```java
public org.apache.dubbo.registry.Registry getRegistry(org.apache.dubbo.common.URL arg0) {
    if (arg0 == null) throw new IllegalArgumentException("url == null");
    org.apache.dubbo.common.URL url = arg0;
    String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());
    if (extName == null)
        throw new IllegalStateException("Fail to get extension(org.apache.dubbo.registry.RegistryFactory) name from url(" + url.toString() + ") use keys([protocol])");
    org.apache.dubbo.registry.RegistryFactory extension = null;
    try {
		//得到ZookeeperRegistryFactory的实例extension
        extension = (org.apache.dubbo.registry.RegistryFactory) ExtensionLoader.getExtensionLoader(org.apache.dubbo.registry.RegistryFactory.class).getExtension(extName);
    } catch (Exception e) {
        if (count.incrementAndGet() == 1) {
            logger.warn("Failed to find extension named " + extName + " for type org.apache.dubbo.registry.RegistryFactory, will use default extension dubbo instead.", e);
        }
        extension = (org.apache.dubbo.registry.RegistryFactory) ExtensionLoader.getExtensionLoader(org.apache.dubbo.registry.RegistryFactory.class).getExtension("dubbo");
    }
    return extension.getRegistry(arg0);
}
```

先调用ZookeeperRegister的父类FailbackRegistry的register()方法：

```java
@Override
public void register(URL url) {
    super.register(url);
    failedRegistered.remove(url);
    failedUnregistered.remove(url);
    try {
        // Sending a registration request to the server side
        doRegister(url);
    } catch (Exception e) {
        Throwable t = e;

        // If the startup detection is opened, the Exception is thrown directly.
        boolean check = getUrl().getParameter(Constants.CHECK_KEY, true)
            && url.getParameter(Constants.CHECK_KEY, true)
            && !Constants.CONSUMER_PROTOCOL.equals(url.getProtocol());
        boolean skipFailback = t instanceof SkipFailbackWrapperException;
        if (check || skipFailback) {
            if (skipFailback) {
                t = t.getCause();
            }
            throw new IllegalStateException("Failed to register " + url + " to registry " + getUrl().getAddress() + ", cause: " + t.getMessage(), t);
        } else {
            logger.error("Failed to register " + url + ", waiting for retry, cause: " + t.getMessage(), t);
        }

        // Record a failed registration request to a failed list, retry regularly
        failedRegistered.add(url);
    }
}
```

最后执行ZookeeperRegistry的doRegister()方法，向服务端发送注册请求：

```java
@Override
protected void doRegister(URL url) {
    try {
        zkClient.create(toUrlPath(url), url.getParameter(Constants.DYNAMIC_KEY, true));
    } catch (Throwable e) {
        throw new RpcException("Failed to register " + url + " to zookeeper " + getUrl() + ", cause: " + e.getMessage(), e);
    }
}
```

服务注册调用栈：

```
"main@1" prio=5 tid=0x1 nid=NA runnable
  java.lang.Thread.State: RUNNABLE
	  at org.apache.dubbo.registry.zookeeper.ZookeeperRegistry.doRegister(ZookeeperRegistry.java:114)
	  at org.apache.dubbo.registry.support.FailbackRegistry.register(FailbackRegistry.java:137)
	  at org.apache.dubbo.registry.integration.RegistryProtocol.register(RegistryProtocol.java:127)
	  at org.apache.dubbo.registry.integration.RegistryProtocol.export(RegistryProtocol.java:147)
	  at org.apache.dubbo.rpc.protocol.ProtocolListenerWrapper.export(ProtocolListenerWrapper.java:55)
	  at org.apache.dubbo.qos.protocol.QosProtocolWrapper.export(QosProtocolWrapper.java:61)
	  at org.apache.dubbo.rpc.protocol.ProtocolFilterWrapper.export(ProtocolFilterWrapper.java:98)
	  at org.apache.dubbo.rpc.Protocol$Adaptive.export(Protocol$Adaptive.java:-1)
	  at org.apache.dubbo.config.ServiceConfig.doExportUrlsFor1Protocol(ServiceConfig.java:513)
	  at org.apache.dubbo.config.ServiceConfig.doExportUrls(ServiceConfig.java:358)
	  at org.apache.dubbo.config.ServiceConfig.doExport(ServiceConfig.java:317)
	  - locked <0x9bb> (a org.apache.dubbo.config.spring.ServiceBean)
	  at org.apache.dubbo.config.ServiceConfig.export(ServiceConfig.java:216)
	  at org.apache.dubbo.config.spring.ServiceBean.onApplicationEvent(ServiceBean.java:123)
	  at org.apache.dubbo.config.spring.ServiceBean.onApplicationEvent(ServiceBean.java:49)
```


