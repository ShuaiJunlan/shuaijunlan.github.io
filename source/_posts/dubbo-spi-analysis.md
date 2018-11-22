---
title: Dubbo SPI机制源码分析
date: 2018-08-09 15:03:57
tags:
    - dubbo
---

Dubbo是微内核架构，还是开闭原则的应用，把核心流程架构固定，但是流程的各个节点对重新改进是开放的。具体的实现机制就是SPI(Service Provider Interface)机制，Dubbo基于Java SPI机制（不了解Java SPI机制的可以参考这篇文章[《深入理解Java SPI机制》](https://shuaijunlan.github.io/2018/08/03/java-spi-introduction/)），在其基础上做了改进和扩展。

根据SPI规范，接口由框架定义，具体实现可以由不同的厂商提供，在Dubbo jar包可以发现在`/META-INF/dubbo/internal`目录下有许多接口命名的文件，文件里面的内容就是文件名代表的接口的各种实现类，这就是Dubbo SPI机制的配置基础，以`org.apache.dubbo.rpc.Protocol`文件为例，内容如下（dubbo-2.7.0-SNAPSHOT 版本）：

```
filter=org.apache.dubbo.rpc.protocol.ProtocolFilterWrapper
listener=org.apache.dubbo.rpc.protocol.ProtocolListenerWrapper
mock=org.apache.dubbo.rpc.support.MockProtocol
dubbo=org.apache.dubbo.rpc.protocol.dubbo.DubboProtocol
injvm=org.apache.dubbo.rpc.protocol.injvm.InjvmProtocol
rmi=org.apache.dubbo.rpc.protocol.rmi.RmiProtocol
hessian=org.apache.dubbo.rpc.protocol.hessian.HessianProtocol
http=org.apache.dubbo.rpc.protocol.http.HttpProtocol

org.apache.dubbo.rpc.protocol.webservice.WebServiceProtocol
thrift=org.apache.dubbo.rpc.protocol.thrift.ThriftProtocol
memcached=org.apache.dubbo.rpc.protocol.memcached.MemcachedProtocol
redis=org.apache.dubbo.rpc.protocol.redis.RedisProtocol
rest=org.apache.dubbo.rpc.protocol.rest.RestProtocol
registry=org.apache.dubbo.registry.integration.RegistryProtocol
qos=org.apache.dubbo.qos.protocol.QosProtocolWrapper
```

<!-- more -->

在Dubbo SPI机制中，`org.apache.dubbo.rpc.Protocol`接口由以上那么多的具体实现，`=`前面是扩展名，后面是扩展类的实现；

SPI的启动的入口类是ExtensionLoader，这个类没定义public构造函数，只有一个privae的，而且public的静态方法也只有一个`public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type)`，这个方法也是SPI的入口方法，**若想获取某个接口类型的扩展，先必须获取其对应的ExtensionLoader**

```java
//私有构造器
private ExtensionLoader(Class<?> type) {
    this.type = type;
    //objectFactory 对象 ，ExtensionFactory本身也是spi的
    //如果是ExtensionFactory本身的ExtensionLoader实例，objectFactory字段为null
    //否则，是ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension()；关于getAdaptiveExtension()方法返回的实例，后面会看到
    objectFactory = (type == ExtensionFactory.class ? null : ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());
}

private static <T> boolean withExtensionAnnotation(Class<T> type) {
    return type.isAnnotationPresent(SPI.class);
}
//获取某个接口的ExtensionLoader
@SuppressWarnings("unchecked")
public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
    if (type == null)
        throw new IllegalArgumentException("Extension type == null");
    if (!type.isInterface()) {
        throw new IllegalArgumentException("Extension type(" + type + ") is not interface!");
    }
    //判断接口是否有SPI注解，Dubbo里所有需要SPI扩展的接口都需要添加@SPI注解
    if (!withExtensionAnnotation(type)) {
        throw new IllegalArgumentException("Extension type(" + type +
                                           ") is not extension, because WITHOUT @" + SPI.class.getSimpleName() + " Annotation!");
    }
	//判断是否已经存在
    ExtensionLoader<T> loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
    if (loader == null) {
        //利用私有构造器创建ExtensionLoader，并且放入缓存
        EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));
        loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
    }
    return loader;
}
```

创建了ExtensionLoader实例，我们就可以通过SPI机制获取想要的接口扩展类实例了，下面就以`org.apache.dubbo.rpc.Protocol`接口获取名为Dubbo的扩展实例为例：

```java
ExtensionLoader.getExtensionLoader(Protocol.class).getExtension(DubboProtocol.NAME); 
```

跟进getExtension方法：

```java
/**
     * Find the extension with the given name. If the specified name is not found, then {@link IllegalStateException}
     * will be thrown.
     */
@SuppressWarnings("unchecked")
public T getExtension(String name) {
    if (name == null || name.length() == 0)
        throw new IllegalArgumentException("Extension name == null");
    //获取默认扩展
    if ("true".equals(name)) {
        return getDefaultExtension();
    }
    //指定扩展实例，判断是否已经缓存
    Holder<Object> holder = cachedInstances.get(name);
    if (holder == null) {
        //创建Holder实例，放入缓存
        cachedInstances.putIfAbsent(name, new Holder<Object>());
        holder = cachedInstances.get(name);
    }
    Object instance = holder.get();
    //加锁技巧，保证线程安全
    if (instance == null) {
        synchronized (holder) {
            instance = holder.get();
            if (instance == null) {
                //根据扩展名，获取具体扩展实例，放入缓存holder中
                instance = createExtension(name);
                holder.set(instance);
            }
        }
    }
    //返回具体的扩展实例
    return (T) instance;
}
```

这里有两个获取扩展的相关方法，一个是`getDefaultExtension()`获取默认扩展，另一个是`createExtension(name)`根据扩展名获取扩展实例，下面分析这两个方法的具体实现：

```java
/***
     * 这个方法，总结起来有3个步骤，
     * 1，通过扩展名，找到扩展实现类，这过程可能触发spi文件加载解析
     * 2，利用反射机制，获取扩展类实例，并完成依赖注入
     * 3，如果接口扩展有包装类，实例化包装类
     * 最后返回经由以上3步流程后，产生的对象。
     * 这3步，前一步都是后一步的基础，要顺序完成
     */
@SuppressWarnings("unchecked")
private T createExtension(String name) {
    //根据扩展名，获取扩展实现类的class（完成第1步）
    Class<?> clazz = getExtensionClasses().get(name);
    if (clazz == null) {
        throw findException(name);
    }
    try {
        //从缓存里，获取实现类的实例
        T instance = (T) EXTENSION_INSTANCES.get(clazz);
        if (instance == null) {
            //利用newInstance()反射，构造类实例，病放入缓存
            EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.newInstance());
            instance = (T) EXTENSION_INSTANCES.get(clazz);
        }
        //完成接口实现类依赖注入，依赖组件先从SPI机制构造查找，再从Spring容器查找（完成第2步）
        injectExtension(instance);
        //如果这接口的实现，还有wrapper类，（有接口类型的构造函数）
        //还有把当前实例instance，注入到包装类，包装类有多个，依次层层，循环构造注入
        //最后返回的是，最后一个包装类实例，这也是dubbo的aop实现机制（完成第3步）
        Set<Class<?>> wrapperClasses = cachedWrapperClasses;
        if (wrapperClasses != null && !wrapperClasses.isEmpty()) {
            for (Class<?> wrapperClass : wrapperClasses) {
                instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
            }
        }
        return instance;
    } catch (Throwable t) {
        throw new IllegalStateException("Extension instance(name: " + name + ", class: " +
                                        type + ")  could not be instantiated: " + t.getMessage(), t);
    }
}
```

#### 第一步，加载扩展实现类

```java
//获取某个接口所有实现，按照扩展名：扩展实现，存储在map中
private Map<String, Class<?>> getExtensionClasses() {
    Map<String, Class<?>> classes = cachedClasses.get();
    if (classes == null) {
        synchronized (cachedClasses) {
            classes = cachedClasses.get();
            if (classes == null) {
                classes = loadExtensionClasses();
                cachedClasses.set(classes);
            }
        }
    }
    return classes;
}

// synchronized in getExtensionClasses
//加载类路径中的spi配置文件，构造cachedClasses
private Map<String, Class<?>> loadExtensionClasses() {
    final SPI defaultAnnotation = type.getAnnotation(SPI.class);
    //获取spi 注解  SPI(value="xxx")，默认实现xxx
    if (defaultAnnotation != null) {
        String value = defaultAnnotation.value();
        if ((value = value.trim()).length() > 0) {
            String[] names = NAME_SEPARATOR.split(value);
            //默认实现只能有一个
            if (names.length > 1) {
                throw new IllegalStateException("more than 1 default extension name on extension " + type.getName()
                                                + ": " + Arrays.toString(names));
            }
            //获取spi默认实现值
            if (names.length == 1) cachedDefaultName = names[0];
        }
    }

    Map<String, Class<?>> extensionClasses = new HashMap<String, Class<?>>();
    //读取三个目录下的spi 配置文件;/META-INF/dubbo/internal, /META-INF/dubbo, /META-INF/services
    //构造 扩展名:实现类 map
    loadDirectory(extensionClasses, DUBBO_INTERNAL_DIRECTORY, type.getName());
    loadDirectory(extensionClasses, DUBBO_INTERNAL_DIRECTORY, type.getName().replace("org.apache", "com.alibaba"));
    loadDirectory(extensionClasses, DUBBO_DIRECTORY, type.getName());
    loadDirectory(extensionClasses, DUBBO_DIRECTORY, type.getName().replace("org.apache", "com.alibaba"));
    loadDirectory(extensionClasses, SERVICES_DIRECTORY, type.getName());
    loadDirectory(extensionClasses, SERVICES_DIRECTORY, type.getName().replace("org.apache", "com.alibaba"));
    return extensionClasses;
}

private void loadDirectory(Map<String, Class<?>> extensionClasses, String dir, String type) {
    //拼接接口名作为文件名，例如：/META-INF/dubbo/internal/org.apache.dubbo.rpc.Protocol
    String fileName = dir + type;
    try {
        Enumeration<java.net.URL> urls;
        //获取加载ClassLoader类的类加载器
        ClassLoader classLoader = findClassLoader();
        if (classLoader != null) {
            urls = classLoader.getResources(fileName);
        } else {
            urls = ClassLoader.getSystemResources(fileName);
        }
        if (urls != null) {
            while (urls.hasMoreElements()) {
                java.net.URL resourceURL = urls.nextElement();
                //加载资源
                loadResource(extensionClasses, classLoader, resourceURL);
            }
        }
    } catch (Throwable t) {
        logger.error("Exception when load extension class(interface: " +
                     type + ", description file: " + fileName + ").", t);
    }
}
```

我们继续来看loadResource()方法

```java
private void loadResource(Map<String, Class<?>> extensionClasses, ClassLoader classLoader, java.net.URL resourceURL) {
    try {
        BufferedReader reader = new BufferedReader(new InputStreamReader(resourceURL.openStream(), "utf-8"));
        try {
            String line;
            //读取文件每一行
            while ((line = reader.readLine()) != null) {
                final int ci = line.indexOf('#');
                if (ci >= 0) line = line.substring(0, ci);
                line = line.trim();
                if (line.length() > 0) {
                    try {
                        String name = null;
                        int i = line.indexOf('=');
                        if (i > 0) {
                            //name 是扩展名
                            name = line.substring(0, i).trim();
                            //扩展实现类全名
                            line = line.substring(i + 1).trim();
                        }
                        if (line.length() > 0) {
                            //根据line加载类
                            loadClass(extensionClasses, resourceURL, Class.forName(line, true, classLoader), name);
                        }
                    } catch (Throwable t) {
                        IllegalStateException e = new IllegalStateException("Failed to load extension class(interface: " + type + ", class line: " + line + ") in " + resourceURL + ", cause: " + t.getMessage(), t);
                        exceptions.put(line, e);
                    }
                }
            }
        } finally {
            reader.close();
        }
    } catch (Throwable t) {
        logger.error("Exception when load extension class(interface: " +
                     type + ", class file: " + resourceURL + ") in " + resourceURL, t);
    }
}

private void loadClass(Map<String, Class<?>> extensionClasses, java.net.URL resourceURL, Class<?> clazz, String name) throws NoSuchMethodException {
    //盘判断实现类是否实现了type接口
    if (!type.isAssignableFrom(clazz)) {
        throw new IllegalStateException("Error when load extension class(interface: " +
                                        type + ", class line: " + clazz.getName() + "), class "
                                        + clazz.getName() + "is not subtype of interface.");
    }
    //判断实现类是否有Adaptive注解
    if (clazz.isAnnotationPresent(Adaptive.class)) {
        if (cachedAdaptiveClass == null) {
            //赋值
            cachedAdaptiveClass = clazz;
        } else if (!cachedAdaptiveClass.equals(clazz)) {
            //一个接口的SPI实现，只能有一个实现类是Adaptive的
            throw new IllegalStateException("More than 1 adaptive class found: "
                                            + cachedAdaptiveClass.getClass().getName()
                                            + ", " + clazz.getClass().getName());
        }
    } else if (isWrapperClass(clazz)) { //判断是否为包装类
        //一个接口的SPI实现可以有多个包装类
        Set<Class<?>> wrappers = cachedWrapperClasses;
        if (wrappers == null) {
            cachedWrapperClasses = new ConcurrentHashSet<Class<?>>();
            wrappers = cachedWrapperClasses;
        }
        wrappers.add(clazz);
    } else {
        clazz.getConstructor();
        if (name == null || name.length() == 0) {
            name = findAnnotationName(clazz);
            if (name.length() == 0) {
                throw new IllegalStateException("No such extension name for the class " + clazz.getName() + " in the config " + resourceURL);
            }
        }
        String[] names = NAME_SEPARATOR.split(name);
        if (names != null && names.length > 0) { //？？？
            //实现类是否有Active注解
            Activate activate = clazz.getAnnotation(Activate.class);
            if (activate != null) {
                //如果有，加入cachedActivates map（扩展名：实现类class）
                cachedActivates.put(names[0], activate);
            } else {
                // support com.alibaba.dubbo.common.extension.Activate
                com.alibaba.dubbo.common.extension.Activate oldActivate = clazz.getAnnotation(com.alibaba.dubbo.common.extension.Activate.class);
                if (oldActivate != null) {
                    cachedActivates.put(names[0], oldActivate);
                }
            }
            for (String n : names) {
                if (!cachedNames.containsKey(clazz)) {
                    //实现类:扩展名 map 放入缓存
                    cachedNames.put(clazz, n);
                }
                Class<?> c = extensionClasses.get(n);
                if (c == null) {
                    //Adaptive 和wapper类都不在extensionClasses里!!!
                    extensionClasses.put(n, clazz);
                } else if (c != clazz) {
                    throw new IllegalStateException("Duplicate extension " + type.getName() + " name " + n + " on " + c.getName() + " and " + clazz.getName());
                }
            }
        }
    }
}
private boolean isWrapperClass(Class<?> clazz) {
    try {
        //实现类里，是否有，参数是接口类型的（比如 com.alibaba.dubbo.rpc.Protocol类型，并且1个参数）的构造函数
        //表示它是个接口包装类
        clazz.getConstructor(type);
        return true;
    } catch (NoSuchMethodException e) {
        return false;
    }
}
```

#### 第二步，依赖注入流程分析

首先来看injectExtension(T instance)的实现：

```java
//实例对象，字段依赖注入。字段类型可以是spi 接口类型，或者是Spring bean 类型
// 依赖注入的字段对象，是通过ExtensionLoader的objectFactory属性完成的，
// objectFacotry 会根据先后通过spi机制和从spring 容器里获取属性对象并注入。
// objectFactory 是在ExtensionLoader私有构造函数中赋值
private T injectExtension(T instance) {
    try {
        if (objectFactory != null) {
            for (Method method : instance.getClass().getMethods()) {
                if (method.getName().startsWith("set")
                    && method.getParameterTypes().length == 1
                    && Modifier.isPublic(method.getModifiers())) { //获取所有public类型，并且只有一个参数的以set开头的方法
                    Class<?> pt = method.getParameterTypes()[0];
                    try {
                        //根据驼峰命名法，根据方法名，构造set方法要赋值的属性名
                        String property = method.getName().length() > 3 ? method.getName().substring(3, 4).toLowerCase() + method.getName().substring(4) : "";
                        //通过getExtension的方法获取属性对象，所以还要看getExtension的实现。
                        Object object = objectFactory.getExtension(pt, property);
                        if (object != null) {
                            //利用反射机制，赋值对象属性
                            method.invoke(instance, object);
                        }
                    } catch (Exception e) {
                        logger.error("fail to inject via method " + method.getName()
                                     + " of interface " + type.getName() + ": " + e.getMessage(), e);
                    }
                }
            }
        }
    } catch (Exception e) {
        logger.error(e.getMessage(), e);
    }
    return instance;
}
```

看下ExtensionLoader定义的私有构造函数，可以看到objectFactory是通过`ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension()`赋值的，它是ExtensionFactory接口的Adaptive扩展实现，看下getAdaptiveExtension()方法：

```Java
//获取一个SPI接口的Adaptive(实现类有Adaptive注解的)类型扩展实现
public T getAdaptiveExtension() {
    //先取缓存
    Object instance = cachedAdaptiveInstance.get();
    if (instance == null) {
        if (createAdaptiveInstanceError == null) {
            synchronized (cachedAdaptiveInstance) {
                instance = cachedAdaptiveInstance.get();
                if (instance == null) {
                    try {
                        //缓存不在，就创建Adaptive扩展实例
                        instance = createAdaptiveExtension();
                        //对象放入缓存中
                        cachedAdaptiveInstance.set(instance);
                    } catch (Throwable t) {
                        createAdaptiveInstanceError = t;
                        throw new IllegalStateException("fail to create adaptive instance: " + t.toString(), t);
                    }
                }
            }
        } else {
            throw new IllegalStateException("fail to create adaptive instance: " + createAdaptiveInstanceError.toString(), createAdaptiveInstanceError);
        }
    }

    return (T) instance;
}
@SuppressWarnings("unchecked")
private T createAdaptiveExtension() {
    try {
        //获取AdaptiveExtensionClass的class 通过反射获取实例，同时要走依赖注入流程
        //AdaptiveExtensionClass 已在spi 文件解析时赋值
        return injectExtension((T) getAdaptiveExtensionClass().newInstance());
    } catch (Exception e) {
        throw new IllegalStateException("Can not create adaptive extension " + type + ", cause: " + e.getMessage(), e);
    }
}

private Class<?> getAdaptiveExtensionClass() {
    //如果有必要，触发spi加载流程，
	//找到类上有Adaptive注解的class,赋值给cachedAdaptiveClass
    getExtensionClasses();
    if (cachedAdaptiveClass != null) {
        return cachedAdaptiveClass;
    }
    //Adaptive注解不在扩展实现类上，而是在待扩展接口方法上
	//这种情况，就是dubbo动态生成生成java类字串，动态编译生成想要的class
	//这个下面再分析下
    return cachedAdaptiveClass = createAdaptiveExtensionClass();
}
```

目前ExtensionFactory接口3个实现类，只有AdaptiveExtensionFactory类是Adaptive的：

```java
/**
 * AdaptiveExtensionFactory
 */
@Adaptive
public class AdaptiveExtensionFactory implements ExtensionFactory {

    private final List<ExtensionFactory> factories;
	//无参构造函数中，把其他实现类实例加入到factories list中
    public AdaptiveExtensionFactory() {
        ExtensionLoader<ExtensionFactory> loader = ExtensionLoader.getExtensionLoader(ExtensionFactory.class);
        List<ExtensionFactory> list = new ArrayList<ExtensionFactory>();
        //getSupportedExtensions()返回的是 非包装类扩展，非Adaptive扩展，防止无限循环
        for (String name : loader.getSupportedExtensions()) {
            list.add(loader.getExtension(name));
        }
        factories = Collections.unmodifiableList(list);
    }

    @Override
    public <T> T getExtension(Class<T> type, String name) {
        for (ExtensionFactory factory : factories) {
            T extension = factory.getExtension(type, name);
            if (extension != null) {
                return extension;
            }
        }
        return null;
    }

}
```

另外两个实现类是`SpiExtensionFactory`、`SpringExtensionFactory`：

```java
/**
 * SpiExtensionFactory
 */
public class SpiExtensionFactory implements ExtensionFactory {
	//SPI机制获取type扩展接口
    @Override
    public <T> T getExtension(Class<T> type, String name) {
        if (type.isInterface() && type.isAnnotationPresent(SPI.class)) {
            ExtensionLoader<T> loader = ExtensionLoader.getExtensionLoader(type);
            if (!loader.getSupportedExtensions().isEmpty()) {
                //获取的是接口的Adaptive实现
                return loader.getAdaptiveExtension();
            }
        }
        return null;
    }

}

/**
 * SpringExtensionFactory
 */
public class SpringExtensionFactory implements ExtensionFactory {
    private static final Logger logger = LoggerFactory.getLogger(SpringExtensionFactory.class);

    private static final Set<ApplicationContext> contexts = new ConcurrentHashSet<ApplicationContext>();
	//手动将spring容器传入
    public static void addApplicationContext(ApplicationContext context) {
        contexts.add(context);
    }

    public static void removeApplicationContext(ApplicationContext context) {
        contexts.remove(context);
    }

    // currently for test purpose
    public static void clearContexts() {
        contexts.clear();
    }

    @Override
    @SuppressWarnings("unchecked")
    public <T> T getExtension(Class<T> type, String name) {

        //SPI should be get from SpiExtensionFactory
        if (type.isInterface() && type.isAnnotationPresent(SPI.class)) {
            return null;
        }
		//遍历spring容器
        for (ApplicationContext context : contexts) {
            if (context.containsBean(name)) {
                Object bean = context.getBean(name);
                if (type.isInstance(bean)) {
                    return (T) bean;
                }
            }
        }

        logger.warn("No spring extension(bean) named:" + name + ", try to find an extension(bean) of type " + type.getName());

        for (ApplicationContext context : contexts) {
            try {
                return context.getBean(type);
            } catch (NoUniqueBeanDefinitionException multiBeanExe) {
                throw multiBeanExe;
            } catch (NoSuchBeanDefinitionException noBeanExe) {
                if (logger.isDebugEnabled()) {
                    logger.debug("Error when get spring extension(bean) for type:" + type.getName(), noBeanExe);
                }
            }
        }

        logger.warn("No spring extension(bean) named:" + name + ", type:" + type.getName() + " found, stop get bean.");

        return null;
    }

}

```



#### 第三步，实例化包装类流程分析

代码上面createExtension方法里已贴出，为了更好的理解，我们可以看下Protocol接口的实现中，ProtocolFIlterWrapper和ProtocolListenerWrapper两个包装类，可以看到他们都有参数为Protocol类型的public构造函数，实例化时，把上层的protocol对象作为参数传入构造函数作为内部属性，同时包装类本身会实现Protocol接口，所以这就可以做些类似aop的操作，如ProtocolFilterWrapper：

```java
/**
 * ListenerProtocol
 */
public class ProtocolFilterWrapper implements Protocol {

    private final Protocol protocol;

    public ProtocolFilterWrapper(Protocol protocol) {
        if (protocol == null) {
            throw new IllegalArgumentException("protocol == null");
        }
        this.protocol = protocol;
    }
	//实例化过滤器链
    private static <T> Invoker<T> buildInvokerChain(final Invoker<T> invoker, String key, String group) {
        Invoker<T> last = invoker;
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

                    @Override
                    public Result invoke(Invocation invocation) throws RpcException {
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

    @Override
    public int getDefaultPort() {
        return protocol.getDefaultPort();
    }
	//暴露过程前执行过滤器链
    @Override
    public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
        if (Constants.REGISTRY_PROTOCOL.equals(invoker.getUrl().getProtocol())) {
            return protocol.export(invoker);
        }
        return protocol.export(buildInvokerChain(invoker, Constants.SERVICE_FILTER_KEY, Constants.PROVIDER));
    }
	//调用前执行过滤器链
    @Override
    public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
        if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
            return protocol.refer(type, url);
        }
        return buildInvokerChain(protocol.refer(type, url), Constants.REFERENCE_FILTER_KEY, Constants.CONSUMER);
    }

    @Override
    public void destroy() {
        protocol.destroy();
    }

}
```

到此Dubbo SPI机制的三个步骤分析完了。

上面提到的Adaptive类的另一种配置方式，即Adaptive注解配置在方法上，dubbo里，配置Adaptive类有两种方式，一种在就扣实现里，类本身有Adaptive注解，还有一种配置实在接口定义的方法级上有Adaptive注解，这两种方式第一种优先，没有第一种，dubbo自动完成第二种Adaptive类的生成，以Protocol接口为例：

```java
@SPI("dubbo")
public interface Protocol {

    /**
     * 获取缺省端口，当用户没有配置端口时使用。
     *
     * @return 缺省端口
     */
    int getDefaultPort();

    /**
     * 暴露远程服务：<br>
     * 1. 协议在接收请求时，应记录请求来源方地址信息：RpcContext.getContext().setRemoteAddress();<br>
     * 2. export()必须是幂等的，也就是暴露同一个URL的Invoker两次，和暴露一次没有区别。<br>
     * 3. export()传入的Invoker由框架实现并传入，协议不需要关心。<br>
     *
     * @param <T>     服务的类型
     * @param invoker 服务的执行体
     * @return exporter 暴露服务的引用，用于取消暴露
     * @throws RpcException 当暴露服务出错时抛出，比如端口已占用
     */
    @Adaptive
    <T> Exporter<T> export(Invoker<T> invoker) throws RpcException;

    /**
     * 引用远程服务：<br>
     * 1. 当用户调用refer()所返回的Invoker对象的invoke()方法时，协议需相应执行同URL远端export()传入的Invoker对象的invoke()方法。<br>
     * 2. refer()返回的Invoker由协议实现，协议通常需要在此Invoker中发送远程请求。<br>
     * 3. 当url中有设置check=false时，连接失败不能抛出异常，并内部自动恢复。<br>
     *
     * @param <T>  服务的类型
     * @param type 服务的类型
     * @param url  远程服务的URL地址
     * @return invoker 服务的本地代理
     * @throws RpcException 当连接服务提供方失败时抛出
     */
    @Adaptive
    <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException;
    
    /**
     * 释放协议：<br>
     * 1. 取消该协议所有已经暴露和引用的服务。<br>
     * 2. 释放协议所占用的所有资源，比如连接和端口。<br>
     * 3. 协议在释放后，依然能暴露和引用新的服务。<br>
     */
    void destroy();

}
```

在export和refer方法上所有Adaptive注解，根据上面的分析，我们跟踪一下createAdaptiveExtensionClass方法：

```java
private Class<?> createAdaptiveExtensionClass() {
    //生成Adaptive；类源码
    String code = createAdaptiveExtensionClassCode();
    ClassLoader classLoader = findClassLoader();
    //通过SPI获取java 编译器
    org.apache.dubbo.common.compiler.Compiler compiler = ExtensionLoader.getExtensionLoader(org.apache.dubbo.common.compiler.Compiler.class).getAdaptiveExtension();
    //编译源码返回class
    return compiler.compile(code, classLoader);
}
```

`createAdaptiveExtensionClassCode();`方法就是实现字符串拼接, 不同的接口，生成的code会有不同，默认使用javassist对代码进行编译。 这里贴出Protocal生成的Adaptive类的源代码。**体现的思想是，所谓Adaptive方法，其实现，内部的对象类型都是参数（url）和spi机制动态决定的**。

```java
package org.apache.dubbo.rpc;

import org.apache.dubbo.common.extension.ExtensionLoader;

public class Protocol$Adaptive implements org.apache.dubbo.rpc.Protocol {
    public org.apache.dubbo.rpc.Invoker refer(java.lang.Class arg0, org.apache.dubbo.common.URL arg1) throws org.apache.dubbo.rpc.RpcException {
        if (arg1 == null) throw new IllegalArgumentException("url == null");
        org.apache.dubbo.common.URL url = arg1;
        String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());
        if (extName == null)
            throw new IllegalStateException("Fail to get extension(org.apache.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");
        org.apache.dubbo.rpc.Protocol extension = (org.apache.dubbo.rpc.Protocol) ExtensionLoader.getExtensionLoader(org.apache.dubbo.rpc.Protocol.class).getExtension(extName);
        return extension.refer(arg0, arg1);
    }

    public org.apache.dubbo.rpc.Exporter export(org.apache.dubbo.rpc.Invoker arg0) throws org.apache.dubbo.rpc.RpcException {
        if (arg0 == null) throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument == null");
        if (arg0.getUrl() == null)
            throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument getUrl() == null");
        org.apache.dubbo.common.URL url = arg0.getUrl();
        String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());
        if (extName == null)
            throw new IllegalStateException("Fail to get extension(org.apache.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");
        org.apache.dubbo.rpc.Protocol extension = (org.apache.dubbo.rpc.Protocol) ExtensionLoader.getExtensionLoader(org.apache.dubbo.rpc.Protocol.class).getExtension(extName);
        return extension.export(arg0);
    }

    public void destroy() {
        throw new UnsupportedOperationException("method public abstract void org.apache.dubbo.rpc.Protocol.destroy() of interface org.apache.dubbo.rpc.Protocol is not adaptive method!");
    }

    public int getDefaultPort() {
        throw new UnsupportedOperationException("method public abstract int org.apache.dubbo.rpc.Protocol.getDefaultPort() of interface org.apache.dubbo.rpc.Protocol is not adaptive method!");
    }
}
```

