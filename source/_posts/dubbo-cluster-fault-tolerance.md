---
title: Dubbo集群容错机制
date: 2018-09-18 15:34:45
tags:
    - dubbo
---

在之前一篇文章[《Dubbo消费者调用过程源码分析》](https://shuaijunlan.github.io/2018/08/05/dubbo-consumer-calling-process-source-code-analysis/)讲到，在创建代理的时候会生成调用对象invoker，这个时候就会绑定集群策略，我们来看生成invoker的代码，在类`ReferenceConfig#createProxy(Map<String, String> map)`方法中：

```java
//直接在配置文件中配置url，实现直接通信（如果既配置了直连地址又配置了注册中心的地址，则自动忽略注册中心的地址）
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
    //获取配置注册中心的url（可以有多个注册中心的url）
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
    //如果urls为空，则抛出异常
    if (urls.isEmpty()) {
        throw new IllegalStateException("No such any registry to reference " + interfaceName + " on the consumer " + NetUtils.getLocalHost() + " use dubbo version " + Version.getVersion() + ", please config <dubbo:registry address=\"...\" /> to your spring config.");
    }
}
//当urls的长度为一时，可能为服务的直连地址也可能为注册中心的地址
if (urls.size() == 1) {
    invoker = refprotocol.refer(interfaceClass, urls.get(0));
} else {//有多个直连地址，或者多个注册中心的地址
    List<Invoker<?>> invokers = new ArrayList<Invoker<?>>();
    URL registryURL = null;
    for (URL url : urls) {
        //获取每个url对应的invoker
        invokers.add(refprotocol.refer(interfaceClass, url));
        if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
            //获取最后一个注册中心的地址
            registryURL = url; // use last registry url
        }
    }
    if (registryURL != null) { // registry url is available
        // use AvailableCluster only when register's cluster is available
        //当有注册中心的地址时，第一层使用AvailableCluster集群策略，第二层使用默认的集群策略
        URL u = registryURL.addParameter(Constants.CLUSTER_KEY, AvailableCluster.NAME);
        invoker = cluster.join(new StaticDirectory(u, invokers));
    } else { // not a registry url
        //如果没有注册中心地址，则使用默认的集群策略
        invoker = cluster.join(new StaticDirectory(invokers));
    }
}
```

我们总结出三种生成集群策略的入口，**当urls的长度为1时，此时该url可能为注册中心的地址也可能是服务的直连地址，则进一步执行`invoker = refprotocol.refer(interfaceClass, urls.get(0));`；当urls的长度不为1时，此时可能为多个注册中心的地址或者多个服务直连地址，当为多个注册中心的地址时，会执行：**

```java
URL u = registryURL.addParameter(Constants.CLUSTER_KEY, AvailableCluster.NAME);
invoker = cluster.join(new StaticDirectory(u, invokers));
```

**当为多个直连地址时，则执行：`invoker = cluster.join(new StaticDirectory(invokers));`**，下面将对这三种方式进行详细的分析。

<!-- more -->

Dubbo中提供了七种集群模式（FailoverCluster、FailfastCluster、FailsafeCluster、FailbackCluster、ForkingCluster、BroadcastCluster、AvailableCluster），来看一下它们的继承关系图：

![Screenshot from 2018-09-18 11-16-13](https://github.com/shuaijunlan/shuaijunlan.github.io/blob/master/images/Cluster.png?raw=true)

下面将分别对每一种集群模式进行分析。

#### FailoverCluster

```java
@Override
@SuppressWarnings({"unchecked", "rawtypes"})
public Result doInvoke(Invocation invocation, final List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
    List<Invoker<T>> copyinvokers = invokers;
    checkInvokers(copyinvokers, invocation);//??
    String methodName = RpcUtils.getMethodName(invocation);
    //获取重试次数
    int len = getUrl().getMethodParameter(methodName, Constants.RETRIES_KEY, Constants.DEFAULT_RETRIES) + 1;
    if (len <= 0) {
        len = 1;
    }
    // retry loop.
    RpcException le = null; // last exception.
    //存放被调用过的invoker
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
        //根据负载均衡策略选择一个invoker
        Invoker<T> invoker = select(loadbalance, invocation, copyinvokers, invoked);
        invoked.add(invoker);
        RpcContext.getContext().setInvokers((List) invoked);
        try {
            //调用invoke
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
            //捕获到远程调用异常，则直接抛出异常
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

Failover集群容错机制，总的逻辑是，以方法重复次数为限制，每次调用如果失败，
就利用负责均衡策略获取下一个提供者（invoker）,直到调用成功，或者最后方法超限，抛出异常，
其中中间如果有业务异常，则不再重试，直接抛出异常。

#### FailfastCluster

```java
@Override
public Result doInvoke(Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
    checkInvokers(invokers, invocation);
    //通过负载均衡策略选择一个invoker
    Invoker<T> invoker = select(loadbalance, invocation, invokers, null);
    try {
        //只执行一次调用
        return invoker.invoke(invocation);
    } catch (Throwable e) {
        if (e instanceof RpcException && ((RpcException) e).isBiz()) { // biz exception.
            throw (RpcException) e;
        }
        throw new RpcException(e instanceof RpcException ? ((RpcException) e).getCode() : 0,
                               "Failfast invoke providers " + invoker.getUrl() + " " + loadbalance.getClass().getSimpleName()
                               + " select from all providers " + invokers + " for service " + getInterface().getName()
                               + " method " + invocation.getMethodName() + " on consumer " + NetUtils.getLocalHost()
                               + " use dubbo version " + Version.getVersion()
                               + ", but no luck to perform the invocation. Last error is: " + e.getMessage(),
                               e.getCause() != null ? e.getCause() : e);
    }
}
```

 快速失败，只发起一次调用，失败立即报错。通常用于非幂等性的写操作，比如新增记录。

#### FailsafeCluster

```java
@Override
public Result doInvoke(Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
    try {
        checkInvokers(invokers, invocation);
        //通过负载均衡策略选择一个invoker
        Invoker<T> invoker = select(loadbalance, invocation, invokers, null);
        //只执行一次调用
        return invoker.invoke(invocation);
    } catch (Throwable e) {
        logger.error("Failsafe ignore exception: " + e.getMessage(), e);
        //遇到异常则返回一个RpcResult
        return new RpcResult(); // ignore
    }
}
```

失败安全，出现异常时，直接忽略。通常用于写入审计日志等操作。

#### FailbackCluster

```java
private void addFailed(Invocation invocation, AbstractClusterInvoker<?> router) {
    if (retryFuture == null) {
        synchronized (this) {
            if (retryFuture == null) {
                //调度线程池，周期性（5秒一次）的调用retryFailed方法
                retryFuture = scheduledExecutorService.scheduleWithFixedDelay(new Runnable() {

                    @Override
                    public void run() {
                        // collect retry statistics
                        try {
                            //执行之前异常方法的调用
                            retryFailed();
                        } catch (Throwable t) { // Defensive fault tolerance
                            logger.error("Unexpected error occur at collect statistic", t);
                        }
                    }
                }, RETRY_FAILED_PERIOD, RETRY_FAILED_PERIOD, TimeUnit.MILLISECONDS);
            }
        }
    }
    //放入map
    failed.put(invocation, router);
}

void retryFailed() {
    if (failed.size() == 0) {
        return;
    }
    //遍历所有的失败执行
    for (Map.Entry<Invocation, AbstractClusterInvoker<?>> entry : new HashMap<>(failed).entrySet()) {
        Invocation invocation = entry.getKey();
        Invoker<?> invoker = entry.getValue();
        try {
            //发起调用
            invoker.invoke(invocation);
            //调用成功则从failed中删除
            failed.remove(invocation);
        } catch (Throwable e) {
            logger.error("Failed retry to invoke method " + invocation.getMethodName() + ", waiting again.", e);
        }
    }
}

@Override
protected Result doInvoke(Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
    try {
        checkInvokers(invokers, invocation);
        //通过负载均衡策略选择一个invoker
        Invoker<T> invoker = select(loadbalance, invocation, invokers, null);
        return invoker.invoke(invocation);
    } catch (Throwable e) {
        //失败后，记录日志，不抛出异常
        logger.error("Failback to invoke method " + invocation.getMethodName() + ", wait for retry in background. Ignored exception: "
                     + e.getMessage() + ", ", e);
        //记录异常信息，key为调用的方法信息，value为invoker本身
        addFailed(invocation, this);
        return new RpcResult(); // ignore
    }
}
```

 此策略失败自动恢复，后台记录失败请求，定时重发。通常用于消息通知操作。

#### ForkingCluster

```java
@Override
@SuppressWarnings({"unchecked", "rawtypes"})
public Result doInvoke(final Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
    try {
        checkInvokers(invokers, invocation);
        final List<Invoker<T>> selected;
        //获取并行调用的个数
        final int forks = getUrl().getParameter(Constants.FORKS_KEY, Constants.DEFAULT_FORKS);
        //超时时间
        final int timeout = getUrl().getParameter(Constants.TIMEOUT_KEY, Constants.DEFAULT_TIMEOUT);
        if (forks <= 0 || forks >= invokers.size()) {
            selected = invokers;
        } else {
            selected = new ArrayList<>();
            //通过负载均衡策略，选出要并行调用的invoker，放入selected列表中
            for (int i = 0; i < forks; i++) {
                // TODO. Add some comment here, refer chinese version for more details.
                Invoker<T> invoker = select(loadbalance, invocation, invokers, selected);
                if (!selected.contains(invoker)) {//防止重复添加
                    //Avoid add the same invoker several times.
                    selected.add(invoker);
                }
            }
        }
        RpcContext.getContext().setInvokers((List) selected);
        final AtomicInteger count = new AtomicInteger();
        final BlockingQueue<Object> ref = new LinkedBlockingQueue<>();
        //遍历selected列表，通过线程池并发调用
        for (final Invoker<T> invoker : selected) {
            executor.execute(new Runnable() {
                @Override
                public void run() {
                    try {
                        Result result = invoker.invoke(invocation);
                        //把结果放入阻塞队列
                        ref.offer(result);
                    } catch (Throwable e) {
                        int value = count.incrementAndGet();
                        //表示所有并发调用都抛出异常，才把异常加入阻塞队列尾部
                        //这就保证了，只要有一个调用成功，ref.poll()方法就能从队列头部取到返回结果  
                        if (value >= selected.size()) {
                            ref.offer(e);
                        }
                    }
                }
            });
        }
        try {
            //设定阻塞时间，从阻塞队列头部获取返回结果，如果是异常则抛出异常
            Object ret = ref.poll(timeout, TimeUnit.MILLISECONDS);
            if (ret instanceof Throwable) {
                Throwable e = (Throwable) ret;
                throw new RpcException(e instanceof RpcException ? ((RpcException) e).getCode() : 0, "Failed to forking invoke provider " + selected + ", but no luck to perform the invocation. Last error is: " + e.getMessage(), e.getCause() != null ? e.getCause() : e);
            }
            return (Result) ret;
        } catch (InterruptedException e) {
            throw new RpcException("Failed to forking invoke provider " + selected + ", but no luck to perform the invocation. Last error is: " + e.getMessage(), e);
        }
    } finally {
        // clear attachments which is binding to current thread.
        RpcContext.getContext().clearAttachments();
    }
}
```

并行调用多个服务器，只要一个成功即返回。通常用于实时性要求较高的读操作，但需要浪费更多服务资源。可通过 forks="2" 来设置最大并行数。

#### BroadcastCluster

```java
@Override
@SuppressWarnings({"unchecked", "rawtypes"})
public Result doInvoke(final Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
    checkInvokers(invokers, invocation);
    RpcContext.getContext().setInvokers((List) invokers);
    RpcException exception = null;
    Result result = null;
    //遍历调用所有的服务列表，并把结果覆盖以前的
    for (Invoker<T> invoker : invokers) {
        try {
            result = invoker.invoke(invocation);
        } catch (RpcException e) {
            exception = e;
            logger.warn(e.getMessage(), e);
        } catch (Throwable e) {
            exception = new RpcException(e.getMessage(), e);
            logger.warn(e.getMessage(), e);
        }
    }
    //其中有一个失败，则直接抛异常
    if (exception != null) {
        throw exception;
    }
    return result;
}
```

这个策略通常用于通知所有提供者更新缓存或日志等本地资源信息。

#### AvailableCluster

Available集群容错机制，主要逻辑是，简单的调用第一个可到达的服务，如果都不可达，则抛出异常：

```java
@Override
public <T> Invoker<T> join(Directory<T> directory) throws RpcException {
	//没有通过继承AbstractClusterInvoker抽象类，而是直接实现它，也没有使用负载均衡策略，而是简单的选择一个可达的服务
    return new AbstractClusterInvoker<T>(directory) {
        @Override
        public Result doInvoke(Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
            for (Invoker<T> invoker : invokers) {
                if (invoker.isAvailable()) {//获取第一个可达的服务提供方
                    return invoker.invoke(invocation);
                }
            }
            throw new RpcException("No provider available in " + invokers);
        }
    };

}
```







