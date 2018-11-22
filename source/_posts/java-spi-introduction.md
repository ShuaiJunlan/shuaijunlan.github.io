---
title: 深入理解Java SPI机制
date: 2018-08-03 18:38:48
tags:
    - java
    - SPI
---

 SPI的全名为Service Provider Interface，在java.util.ServiceLoader的[文档:https://docs.oracle.com/javase/6/docs/api/java/util/ServiceLoader.html](https://docs.oracle.com/javase/6/docs/api/java/util/ServiceLoader.html)中有比较详细的介绍。究其思想，其实和`Callback`差不多。`Callback`的思想是我们在调用API的时候，我们可以写入一段逻辑代码传到API里面，API内部在合适的时候会调用它，从而实现某种程度上的“定制”。

典型的是`Collections.sort(List<T> list,Comparator<? super T> c)`这个方法，它的第二个参数是一个实现Comparator接口的实例。我们可以根据自己的排序规则写一个类，实现此接口，传入此方法，那么这个方法就会根据我们的规则对list进行排序。

#### Java SPI的具体约定如下

当服务的提供者，提供了服务接口的一种实现之后，在jar包的META-INF/services/目录里同时创建一个以服务接口命名的文件。该文件里就是实现该服务接口的具体实现类。而当外部程序装配这个模块的时候，就能通过该jar包META-INF/services/里的配置文件找到具体的实现类名，并装载实例化，完成模块的注入。 

基于这样一个约定就能很好的找到服务接口的实现类，而不需要再代码里制定。

JDK提供服务实现查找的一个工具类：**java.util.ServiceLoader**

<!-- more -->

#### 实现一个Java SPI示例

假设我们有一个日志服务`ILogService`，其只定义了一个`warn`方法用于输出日志信息，我们希望把它作为SPI，然后具体的实现由对应的服务提供者去实现。ILogService的定义如下:

```java
package cn.shuaijunlan.spi;

/**
 * @author Junlan Shuai[shuaijunlan@gmail.com].
 * @date Created on 7:13 PM 2018/08/04.
 */
public interface ILogService {
    void warn(String msg);
}
```

然后基于这个服务接口实现了两个类，分别是`ConsoleLogServiceImpl`、`FileLogServiceImpl`，代码如下：

```java
package cn.shuaijunlan.spi.impl;

import cn.shuaijunlan.spi.ILogService;

/**
 * @author Junlan Shuai[shuaijunlan@gmail.com].
 * @date Created on 7:14 PM 2018/08/04.
 */
public class ConsoleLogServiceImpl implements ILogService {
    @Override
    public void warn(String msg) {
        System.out.println("Console log:"+ msg + "!");
    }
}
=======================================================================================

package cn.shuaijunlan.spi.impl;

import cn.shuaijunlan.spi.ILogService;

/**
 * @author Junlan Shuai[shuaijunlan@gmail.com].
 * @date Created on 7:15 PM 2018/08/04.
 */
public class FileLogServiceImpl implements ILogService {
    @Override
    public void warn(String msg) {
        System.out.println("File log:" + msg +"!");
    }
}
```

根据SPI的规范我们的服务实现类必须有一个无参构造方法。我们的SPI服务提供者需要将其在`classpath`下的`META-INF/services`目录下以服务接口全路径名命名的文件中写对应的实现类的全路径名称，每一行代表一个实现，如果需要注释信息可以使用**#**进行注释，根据官方的要求，这个文件的编码格式必须是UTF-8。我们示例中的ILogService的全路径名是`cn.shuaijunlan.spi.ILogService`，所以我们需要在类路径下的`META-INF/services`目录下创建一个名称为`cn.shuaijunlan.spi.ILogService`文件。在本示例中我们一个提供了两个实现，所以该文件的内容如下：

```
# Console log & File log

cn.shuaijunlan.spi.impl.ConsoleLogServiceImpl
cn.shuaijunlan.spi.impl.FileLogServiceImpl
```

ServiceLoader是实现了java.util.Iterator接口的，而且是基于我们所使用的服务的实现，所以可以通过ServiceLoader的实例来遍历其中的服务实现者，从而调用对应的服务提供者。测试函数如下：

```java
package cn.shuaijunlan.spi;

import java.util.Iterator;
import java.util.ServiceLoader;

/**
 * @author Junlan Shuai[shuaijunlan@gmail.com].
 * @date Created on 7:28 PM 2018/08/04.
 */
public class Main {
    private static ServiceLoader<ILogService> services = ServiceLoader.load(ILogService.class);

    public static void main(String[] args) {
        Iterator<ILogService> iterator = services.iterator();
        while (iterator.hasNext()){
            iterator.next().warn("Hello SPI");
        }
    }
}

```

控制台输出结果如下：

```
Console log:Hello SPI!
File log:Hello SPI!
```

基于SPI规范，我们最终实现了想要的结果。

#### ServiceLoader源码分析

在调用`ServiceLoader.load(ILogService.class);`时，代码进入：

```java
public static <S> ServiceLoader<S> load(Class<S> service) {
    //获取当前线程上下文类加载器
    ClassLoader cl = Thread.currentThread().getContextClassLoader();
    return ServiceLoader.load(service, cl);
}
```

执行之后，程序进入`ServiceLoader.load(service, cl);`方法

```java
public static <S> ServiceLoader<S> load(Class<S> service,
                                        ClassLoader loader)
{
    return new ServiceLoader<>(service, loader);
}
```

返回一个ServiceLoader的实例

在调用`services.iterator();`方法时，返回一个Iterator容器

```java
public Iterator<S> iterator() {
    return new Iterator<S>() {

        Iterator<Map.Entry<String,S>> knownProviders
            = providers.entrySet().iterator();
        //第一次执行hasNext方法时，knownProviders的size为0，会继续执行lookupIterator.hasNext()，最后进入到hasNextService方法中
        public boolean hasNext() {
            if (knownProviders.hasNext())
                return true;
            return lookupIterator.hasNext();
        }

        public S next() {
            if (knownProviders.hasNext())
                return knownProviders.next().getValue();
            return lookupIterator.next();
        }

        public void remove() {
            throw new UnsupportedOperationException();
        }

    };
}
```

我们来看hasNextService方法：

```java
private boolean hasNextService() {
    if (nextName != null) {
        return true;
    }
    if (configs == null) {
        try {
            //获取接口的全名
            String fullName = PREFIX + service.getName();
            if (loader == null)
                configs = ClassLoader.getSystemResources(fullName);
            else
                configs = loader.getResources(fullName);
        } catch (IOException x) {
            fail(service, "Error locating configuration files", x);
        }
    }
    while ((pending == null) || !pending.hasNext()) {
        if (!configs.hasMoreElements()) {
            return false;
        }
        //获取所有实现类的全名，具体的解析函数查看parse函数
        pending = parse(service, configs.nextElement());
    }
    nextName = pending.next();
    return true;
}
```

当调用`iterator.next()`方法时，会进入到：

```java
private S nextService() {
    if (!hasNextService())
        throw new NoSuchElementException();
    String cn = nextName;
    nextName = null;
    Class<?> c = null;
    try {
        //生成名称为cn的Class对象，不进行初始化
        c = Class.forName(cn, false, loader);
    } catch (ClassNotFoundException x) {
        //没找到类则会抛出异常
        fail(service,
             "Provider " + cn + " not found");
    }
    if (!service.isAssignableFrom(c)) {
        fail(service,
             "Provider " + cn  + " not a subtype");
    }
    try {
        //进行实例化
        S p = service.cast(c.newInstance());
        providers.put(cn, p);
        //返回实例
        return p;
    } catch (Throwable x) {
        fail(service,
             "Provider " + cn + " could not be instantiated",
             x);
    }
    throw new Error();          // This cannot happen
}
```

 在上述分析中我们可以看到ServiceLoader不是一实例化以后立马就去读配置文件中的服务实现者，并且进行对应的实例化工作的，而是会等到需要通过其Iterator实现获取对应的服务提供者时才会加载对应的配置文件进行解析，具体来说是在调用Iterator的hasNext方法时会去加载配置文件进行解析，在调用next方法时会将对应的服务提供者进行实例化并进行缓存。所有的配置文件只加载一次，服务提供者也只实例化一次，如需要重新加载配置文件可调用ServiceLoader的reload方法。

#### 框架案例

**1.common-logging**

apache最早提供的日志的门面接口。只有接口，没有实现。具体方案由各提供商实现，发现日志提供商是通过扫描 META-INF/services/org.apache.commons.logging.LogFactory配置文件，通过读取该文件的内容找到日志提供商实现类。只要我们的日志实现里包含了这个文件，并在文件里指定 LogFactory工厂接口的实现类即可。

**2.jdbc**

jdbc4.0以前，开发人员还需要基于Class.forName("xxx")的方式来装载驱动，jdbc4也基于spi的机制来发现驱动提供商了，可以通过META-INF/services/java.sql.Driver文件里指定实现类的方式来暴露驱动提供者。

#### SPI不足之处

* 通过上面的解析，可以发现，我们使用SPI查找具体的实现的时候，需要遍历所有的实现，并实例化，然后我们在循环中才能找到我们需要实现。这应该也是最大的缺点，需要把所有的实现都实例化了，即便我们不需要，也都给实例化了。
* 获取某个实现类的方式不够灵活，只能通过Iterator的形式获取，不能根据某个参数来获取对应的实现类。

#### REFERENCES

1.[https://docs.oracle.com/javase/tutorial/sound/SPI-intro.html](https://docs.oracle.com/javase/tutorial/sound/SPI-intro.html)

2.[https://cxis.me/2017/04/17/Java%E4%B8%ADSPI%E6%9C%BA%E5%88%B6%E6%B7%B1%E5%85%A5%E5%8F%8A%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90](https://cxis.me/2017/04/17/Java%E4%B8%ADSPI%E6%9C%BA%E5%88%B6%E6%B7%B1%E5%85%A5%E5%8F%8A%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/#comments)

