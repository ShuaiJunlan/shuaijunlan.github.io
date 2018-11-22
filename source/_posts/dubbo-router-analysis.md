---
title: Dubbo路由机制分析
date: 2018-10-17 10:03:57
tags:
    - dubbo
---

![](https://github.com/shuaijunlan/shuaijunlan.github.io/blob/master/images/Screenshot from 2018-10-17 16-41-11.png?raw=true)

<!-- more -->

这篇文章主要讲Dubbo的路由特性，当Consumer发起一个请求，Dubbo依据配置的路由规则，计算出那些提供者可以提供这次的请求服务，所以路由机制会在`集群容错策略`和`负载均衡策略`之前被执行，下面我们来开始分析源码。

执行路由机制的入口是在AbstractClusterInvoker类的invoke方法，其中调用了list()方法，然后会执行AbstractDirectory的list()方法：

* AbstractDirectory#list()

```java
@Override
public List<Invoker<T>> list(Invocation invocation) throws RpcException {
    if (destroyed) {
        throw new RpcException("Directory already destroyed .url: " + getUrl());
    }
    //获取所有的提供者
    List<Invoker<T>> invokers = doList(invocation);
    //在哪里设置的routers？稍后揭晓
    List<Router> localRouters = this.routers; // local reference
    if (localRouters != null && !localRouters.isEmpty()) {
        for (Router router : localRouters) {
            try {
                if (router.getUrl() == null || router.getUrl().getParameter(Constants.RUNTIME_KEY, false)) {
                    //依次通过路由规则进行过滤
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

那么路由器是在哪设置的？通过搜索代码发现是则AbstractDirectory类的setRouters()方法里设置的：

* AbstractDirectory#setRouters()

```java
protected void setRouters(List<Router> routers) {
    // copy list
    routers = routers == null ? new ArrayList<Router>() : new ArrayList<Router>(routers);
    // append url router
    String routerKey = url.getParameter(Constants.ROUTER_KEY);
    if (routerKey != null && routerKey.length() > 0) {
        //根据Dubbo SPI机制获取RouterFactory
        RouterFactory routerFactory = ExtensionLoader.getExtensionLoader(RouterFactory.class).getExtension(routerKey);
        routers.add(routerFactory.getRouter(url));
    }
    // append mock invoker selector 为什么添加这个？
	//MockInvokersSelector路由器，是dubbo对mock调用支持的一部分,稍后看下源码
    routers.add(new MockInvokersSelector());
    //进行排序？？
    //排序后MockInvokersSelector会放在最后
    Collections.sort(routers);
    this.routers = routers;
}
```

再进一步分析，setRouters()在哪里被调用了？通过查看方法调用栈，在AbstractDirectory的子类RegisterDirectory里notify()方法里调用了，notify()方法是注册中心通知consumer回调的方法：

* RegisterDirectory#notify()

```java
@Override
public synchronized void notify(List<URL> urls) {
    List<URL> invokerUrls = new ArrayList<URL>();
    List<URL> routerUrls = new ArrayList<URL>();
    List<URL> configuratorUrls = new ArrayList<URL>();
    for (URL url : urls) {
        String protocol = url.getProtocol();
        String category = url.getParameter(Constants.CATEGORY_KEY, Constants.DEFAULT_CATEGORY);
        if (Constants.ROUTERS_CATEGORY.equals(category)
            || Constants.ROUTE_PROTOCOL.equals(protocol)) {
            routerUrls.add(url);
        } else if (Constants.CONFIGURATORS_CATEGORY.equals(category)
                   || Constants.OVERRIDE_PROTOCOL.equals(protocol)) {
            configuratorUrls.add(url);
        } else if (Constants.PROVIDERS_CATEGORY.equals(category)) {
            invokerUrls.add(url);
        } else {
            logger.warn("Unsupported category " + category + " in notified url: " + url + " from registry " + getUrl().getAddress() + " to consumer " + NetUtils.getLocalHost());
        }
    }
    // configurators
    if (configuratorUrls != null && !configuratorUrls.isEmpty()) {
        this.configurators = toConfigurators(configuratorUrls);
    }
    // routers
    if (routerUrls != null && !routerUrls.isEmpty()) {
        //把路由配置转换成路由实例
        List<Router> routers = toRouters(routerUrls);
        if (routers != null) { // null - do nothing
            setRouters(routers);
        }
    }
    List<Configurator> localConfigurators = this.configurators; // local reference
    // merge override parameters
    this.overrideDirectoryUrl = directoryUrl;
    if (localConfigurators != null && !localConfigurators.isEmpty()) {
        for (Configurator configurator : localConfigurators) {
            this.overrideDirectoryUrl = configurator.configure(overrideDirectoryUrl);
        }
    }
    // providers
    refreshInvoker(invokerUrls);
}
```

在上面的方法中，有一个比较重要的方法`toRouters()`，它将路由配置转换成路由实例：

* RegisterDirectory#toRouters()

```java
/**
     * @param urls
     * @return null : no routers ,do nothing
     * else :routers list
     */
private List<Router> toRouters(List<URL> urls) {
    List<Router> routers = new ArrayList<Router>();
    if (urls == null || urls.isEmpty()) {
        return routers;
    }
    if (urls != null && !urls.isEmpty()) {
        for (URL url : urls) {
            if (Constants.EMPTY_PROTOCOL.equals(url.getProtocol())) {
                continue;
            }
            String routerType = url.getParameter(Constants.ROUTER_KEY);
            if (routerType != null && routerType.length() > 0) {
                url = url.setProtocol(routerType);
            }
            try {
                Router router = routerFactory.getRouter(url);
                if (!routers.contains(router)) {
                    routers.add(router);
                }
            } catch (Throwable t) {
                logger.error("convert router url to router error, url: " + url, t);
            }
        }
    }
    return routers;
}
```

下面将分别分析四种路由的具体实现和配置方法：

#### ConditionRouterFactory

创建一个`ConditionRouter`实例，

#### ScriptRouterFactory

创建一个`ScriptRouter`实例，

#### TagRouterFactory

创建一个`TagRouter`实例，

#### FileRouterFactory





