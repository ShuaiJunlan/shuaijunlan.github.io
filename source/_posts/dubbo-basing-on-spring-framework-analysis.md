---
title: 基于Spring构建Dubbo源码分析
date: 2018-08-13 21:06:34
tags:
    - dubbo
---

从Dubbo 2.7.0的项目依赖来看，依赖的Spring Framework版本是`4.3.16.RELEASE`：

```xml
<properties>
	<spring_version>4.3.16.RELEASE</spring_version>
</properties>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-framework-bom</artifactId>
    <version>${spring_version}</version>
    <type>pom</type>
    <scope>import</scope>
</dependency>
```

Dubbo是基于Spring构建和运行的，兼容Spring配置，Dubbo利用了SpringFramework的Extensible XML authoring 特性，扩展了Spring标签，关于如何利用Spring扩展标签，可以参考官方文档[ 《Extensible XML authoring》](https://docs.spring.io/spring/docs/4.3.16.RELEASE/spring-framework-reference/htmlsingle/#xml-custom)：

* 编写xml，描述需要扩展的标签的配置属性，dubbo实现放在jar包`META-INF/dubbo.xsd`文件里 同时通过编写`META-INF/spring.handlers`文件，提供给spring，内容如下：

  ```
  http\://dubbo.apache.org/schema/dubbo=org.apache.dubbo.config.spring.schema.DubboNamespaceHandler
  http\://code.alibabatech.com/schema/dubbo=org.apache.dubbo.config.spring.schema.DubboNamespaceHandler
  ```

* 编写一个NamespaceHandler接口实现类，dubbo中的实现类是DubboNamespaceHandler，同时通过编写`META-INF/spring.schemas`文件提供给Spring，内容如下：

  ```
  http\://dubbo.apache.org/schema/dubbo/dubbo.xsd=META-INF/dubbo.xsd
  http\://code.alibabatech.com/schema/dubbo/dubbo.xsd=META-INF/compat/dubbo.xsd
  ```

* 编写一个或多个BeanDefinitionParser实现类，用来解析扩展的元素，Dubbo实现类是DubboBeanDefinitionParaser

* 把以上解析组件注册给Spring

<!-- more -->

#### 解析DubboNamespaceHandler实现

首先来看一下DubboNamespaceHandler类的源代码：

```java
public class DubboNamespaceHandler extends NamespaceHandlerSupport {

    static {
        Version.checkDuplicate(DubboNamespaceHandler.class);
    }

    @Override
    public void init() {
        //注册每个标签对应的解析类
        registerBeanDefinitionParser("application", new DubboBeanDefinitionParser(ApplicationConfig.class, true));
        registerBeanDefinitionParser("module", new DubboBeanDefinitionParser(ModuleConfig.class, true));
        registerBeanDefinitionParser("registry", new DubboBeanDefinitionParser(RegistryConfig.class, true));
        registerBeanDefinitionParser("monitor", new DubboBeanDefinitionParser(MonitorConfig.class, true));
        registerBeanDefinitionParser("provider", new DubboBeanDefinitionParser(ProviderConfig.class, true));
        registerBeanDefinitionParser("consumer", new DubboBeanDefinitionParser(ConsumerConfig.class, true));
        registerBeanDefinitionParser("protocol", new DubboBeanDefinitionParser(ProtocolConfig.class, true));
        registerBeanDefinitionParser("service", new DubboBeanDefinitionParser(ServiceBean.class, true));
        registerBeanDefinitionParser("reference", new DubboBeanDefinitionParser(ReferenceBean.class, false));
        registerBeanDefinitionParser("annotation", new AnnotationBeanDefinitionParser());
    }

}
```

上面提到的10个扩展标签，分别对应10个配置类，类的层次关系如下图(右键-->Open image in new tab)：

![](https://github.com/shuaijunlan/shuaijunlan.github.io/blob/master/images/AbstractConfig.png?raw=true)

如上图可以看出，主要基类是AbstractConfig和AbstractMethodConfig，而AbstractConfig又是所有类的基类，这10个配置类中ReferenceBean，ServiceBean，AnnotationBean 3个类又都实现了若干spring接口，这3个类算是利用spring完成dubbo调用的驱动类，后面要分别看源码。

#### 解析DubboBeanDefinitionParser类

这个类实现了BeanDefinitionParser接口，这是个Spring的原生接口，里面只有一个方法：

```java
public interface BeanDefinitionParser {

	/**
	 * Parse the specified {@link Element} and register the resulting
	 * {@link BeanDefinition BeanDefinition(s)} with the
	 * {@link org.springframework.beans.factory.xml.ParserContext#getRegistry() BeanDefinitionRegistry}
	 * embedded in the supplied {@link ParserContext}.
	 * <p>Implementations must return the primary {@link BeanDefinition} that results
	 * from the parse if they will ever be used in a nested fashion (for example as
	 * an inner tag in a {@code <property/>} tag). Implementations may return
	 * {@code null} if they will <strong>not</strong> be used in a nested fashion.
	 * @param element the element that is to be parsed into one or more {@link BeanDefinition BeanDefinitions}
	 * @param parserContext the object encapsulating the current state of the parsing process;
	 * provides access to a {@link org.springframework.beans.factory.support.BeanDefinitionRegistry}
	 * @return the primary {@link BeanDefinition}
	 */
	BeanDefinition parse(Element element, ParserContext parserContext);

}
```

根据接口的定义，这个方法实现，需要解析Element元素成原生的BeanDefinition类对象，然后利用ParserContext对象的getRegistry()返回的注册器来注册解析后的BeanDefinition类的对象，最后返回这个BeanDefination类对象，下面是Dubbo的实现，主要是完成Spring配置到Spring容器内部BeanDefination转化的过程，下面来分析`parse()`方法：

```java
@SuppressWarnings("unchecked")
private static BeanDefinition parse(Element element, ParserContext parserContext, Class<?> beanClass, boolean required) {
    RootBeanDefinition beanDefinition = new RootBeanDefinition();
    beanDefinition.setBeanClass(beanClass);
    beanDefinition.setLazyInit(false);
    String id = element.getAttribute("id");
    if ((id == null || id.length() == 0) && required) {
        String generatedBeanName = element.getAttribute("name");
        if (generatedBeanName == null || generatedBeanName.length() == 0) {
            if (ProtocolConfig.class.equals(beanClass)) {
                generatedBeanName = "dubbo";
            } else {
                generatedBeanName = element.getAttribute("interface");
            }
        }
        if (generatedBeanName == null || generatedBeanName.length() == 0) {
            generatedBeanName = beanClass.getName();
        }
        id = generatedBeanName;
        int counter = 2;
        while (parserContext.getRegistry().containsBeanDefinition(id)) {
            id = generatedBeanName + (counter++);
        }
    }
    if (id != null && id.length() > 0) {
        if (parserContext.getRegistry().containsBeanDefinition(id)) {
            throw new IllegalStateException("Duplicate spring bean id " + id);
        }
        parserContext.getRegistry().registerBeanDefinition(id, beanDefinition);
        beanDefinition.getPropertyValues().addPropertyValue("id", id);
    }
    if (ProtocolConfig.class.equals(beanClass)) {
        for (String name : parserContext.getRegistry().getBeanDefinitionNames()) {
            BeanDefinition definition = parserContext.getRegistry().getBeanDefinition(name);
            PropertyValue property = definition.getPropertyValues().getPropertyValue("protocol");
            if (property != null) {
                Object value = property.getValue();
                if (value instanceof ProtocolConfig && id.equals(((ProtocolConfig) value).getName())) {
                    definition.getPropertyValues().addPropertyValue("protocol", new RuntimeBeanReference(id));
                }
            }
        }
    } else if (ServiceBean.class.equals(beanClass)) {
        String className = element.getAttribute("class");
        if (className != null && className.length() > 0) {
            RootBeanDefinition classDefinition = new RootBeanDefinition();
            classDefinition.setBeanClass(ReflectUtils.forName(className));
            classDefinition.setLazyInit(false);
            parseProperties(element.getChildNodes(), classDefinition);
            beanDefinition.getPropertyValues().addPropertyValue("ref", new BeanDefinitionHolder(classDefinition, id + "Impl"));
        }
    } else if (ProviderConfig.class.equals(beanClass)) {
        parseNested(element, parserContext, ServiceBean.class, true, "service", "provider", id, beanDefinition);
    } else if (ConsumerConfig.class.equals(beanClass)) {
        parseNested(element, parserContext, ReferenceBean.class, false, "reference", "consumer", id, beanDefinition);
    }
    Set<String> props = new HashSet<String>();
    ManagedMap parameters = null;
    for (Method setter : beanClass.getMethods()) {
        String name = setter.getName();
        if (name.length() > 3 && name.startsWith("set")
            && Modifier.isPublic(setter.getModifiers())
            && setter.getParameterTypes().length == 1) {
            Class<?> type = setter.getParameterTypes()[0];
            String property = StringUtils.camelToSplitName(name.substring(3, 4).toLowerCase() + name.substring(4), "-");
            props.add(property);
            Method getter = null;
            try {
                getter = beanClass.getMethod("get" + name.substring(3), new Class<?>[0]);
            } catch (NoSuchMethodException e) {
                try {
                    getter = beanClass.getMethod("is" + name.substring(3), new Class<?>[0]);
                } catch (NoSuchMethodException e2) {
                }
            }
            if (getter == null
                || !Modifier.isPublic(getter.getModifiers())
                || !type.equals(getter.getReturnType())) {
                continue;
            }
            if ("parameters".equals(property)) {
                parameters = parseParameters(element.getChildNodes(), beanDefinition);
            } else if ("methods".equals(property)) {
                parseMethods(id, element.getChildNodes(), beanDefinition, parserContext);
            } else if ("arguments".equals(property)) {
                parseArguments(id, element.getChildNodes(), beanDefinition, parserContext);
            } else {
                String value = element.getAttribute(property);
                if (value != null) {
                    value = value.trim();
                    if (value.length() > 0) {
                        if ("registry".equals(property) && RegistryConfig.NO_AVAILABLE.equalsIgnoreCase(value)) {
                            RegistryConfig registryConfig = new RegistryConfig();
                            registryConfig.setAddress(RegistryConfig.NO_AVAILABLE);
                            beanDefinition.getPropertyValues().addPropertyValue(property, registryConfig);
                        } else if ("registry".equals(property) && value.indexOf(',') != -1) {
                            parseMultiRef("registries", value, beanDefinition, parserContext);
                        } else if ("provider".equals(property) && value.indexOf(',') != -1) {
                            parseMultiRef("providers", value, beanDefinition, parserContext);
                        } else if ("protocol".equals(property) && value.indexOf(',') != -1) {
                            parseMultiRef("protocols", value, beanDefinition, parserContext);
                        } else {
                            Object reference;
                            if (isPrimitive(type)) {
                                if ("async".equals(property) && "false".equals(value)
                                    || "timeout".equals(property) && "0".equals(value)
                                    || "delay".equals(property) && "0".equals(value)
                                    || "version".equals(property) && "0.0.0".equals(value)
                                    || "stat".equals(property) && "-1".equals(value)
                                    || "reliable".equals(property) && "false".equals(value)) {
                                    // backward compatibility for the default value in old version's xsd
                                    value = null;
                                }
                                reference = value;
                            } else if ("protocol".equals(property)
                                       && ExtensionLoader.getExtensionLoader(Protocol.class).hasExtension(value)
                                       && (!parserContext.getRegistry().containsBeanDefinition(value)
                                           || !ProtocolConfig.class.getName().equals(parserContext.getRegistry().getBeanDefinition(value).getBeanClassName()))) {
                                if ("dubbo:provider".equals(element.getTagName())) {
                                    logger.warn("Recommended replace <dubbo:provider protocol=\"" + value + "\" ... /> to <dubbo:protocol name=\"" + value + "\" ... />");
                                }
                                // backward compatibility
                                ProtocolConfig protocol = new ProtocolConfig();
                                protocol.setName(value);
                                reference = protocol;
                            } else if ("onreturn".equals(property)) {
                                int index = value.lastIndexOf(".");
                                String returnRef = value.substring(0, index);
                                String returnMethod = value.substring(index + 1);
                                reference = new RuntimeBeanReference(returnRef);
                                beanDefinition.getPropertyValues().addPropertyValue("onreturnMethod", returnMethod);
                            } else if ("onthrow".equals(property)) {
                                int index = value.lastIndexOf(".");
                                String throwRef = value.substring(0, index);
                                String throwMethod = value.substring(index + 1);
                                reference = new RuntimeBeanReference(throwRef);
                                beanDefinition.getPropertyValues().addPropertyValue("onthrowMethod", throwMethod);
                            } else if ("oninvoke".equals(property)) {
                                int index = value.lastIndexOf(".");
                                String invokeRef = value.substring(0, index);
                                String invokeRefMethod = value.substring(index + 1);
                                reference = new RuntimeBeanReference(invokeRef);
                                beanDefinition.getPropertyValues().addPropertyValue("oninvokeMethod", invokeRefMethod);
                            } else {
                                if ("ref".equals(property) && parserContext.getRegistry().containsBeanDefinition(value)) {
                                    BeanDefinition refBean = parserContext.getRegistry().getBeanDefinition(value);
                                    if (!refBean.isSingleton()) {
                                        throw new IllegalStateException("The exported service ref " + value + " must be singleton! Please set the " + value + " bean scope to singleton, eg: <bean id=\"" + value + "\" scope=\"singleton\" ...>");
                                    }
                                }
                                reference = new RuntimeBeanReference(value);
                            }
                            beanDefinition.getPropertyValues().addPropertyValue(property, reference);
                        }
                    }
                }
            }
        }
    }
    NamedNodeMap attributes = element.getAttributes();
    int len = attributes.getLength();
    for (int i = 0; i < len; i++) {
        Node node = attributes.item(i);
        String name = node.getLocalName();
        if (!props.contains(name)) {
            if (parameters == null) {
                parameters = new ManagedMap();
            }
            String value = node.getNodeValue();
            parameters.put(name, new TypedStringValue(value, String.class));
        }
    }
    if (parameters != null) {
        beanDefinition.getPropertyValues().addPropertyValue("parameters", parameters);
    }
    return beanDefinition;
}
```

#### ReferenceBean类

ReferenceBean类主要完成早适当的时机（Spring Bean初始化完成或者用户通过Spring容器获取bean）根据服务调用方法配置，生成服务调用代理工作，ReferenceBean类继承如下：

![](https://github.com/shuaijunlan/shuaijunlan.github.io/blob/master/images/ReferenceBean.png?raw=true)

可以看到ReferenceBean实现了FactoryBean、ApplicationContextAware、FactoryBean、InitializingBean及DisposableBean四个接口，通过Spring的回调机制，完成Spring容器的传入，获取Bean类型，Bean初始化和Destory定制等操作，ReferenceBean类里的具体实现如下：

```java
//实现ApplicationContextAware接口的方法，在bean初始化时，回传bean所在容器的引用
@Override
public void setApplicationContext(ApplicationContext applicationContext) {
    this.applicationContext = applicationContext;
    SpringExtensionFactory.addApplicationContext(applicationContext);
}
//实现FactoryBean接口的方法，返回一个Bean实例，在使用Spring API从容器中获取一个bean时调用，
//这里返回的是reference的代理的代理类实例
@Override
public Object getObject() throws Exception {
    return get();
}
//实现FactoryBean接口的方法，返回一个bean的类型
@Override
public Class<?> getObjectType() {
    return getInterfaceClass();
}
//实现FactoryBean接口的方法，返回一个bean是否是单例
@Override
@Parameter(excluded = true)
public boolean isSingleton() {
    return true;
}
//实现InitializingBean的接口方法，在bean所有属性都赋值后，由spring回调执行
//这个方法里可以做些初始化定制
@Override
@SuppressWarnings({"unchecked"})
public void afterPropertiesSet() throws Exception {
    if (getConsumer() == null) {
        //BeanFactoryUtils.beansOfTypeIncludingAncestors()是Spring的一个工具类
        //返回指定容器里，ConsumerConfig.class类及其子类的Bean，如果还没初始化，会触发初始化的过程，依赖注入的概念a
        Map<String, ConsumerConfig> consumerConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, ConsumerConfig.class, false, false);
        if (consumerConfigMap != null && consumerConfigMap.size() > 0) {
            ConsumerConfig consumerConfig = null;
            //遍历map，默认设置ConsumerConfig
            for (ConsumerConfig config : consumerConfigMap.values()) {
                if (config.isDefault() == null || config.isDefault().booleanValue()) {
                    if (consumerConfig != null) {
                        throw new IllegalStateException("Duplicate consumer configs: " + consumerConfig + " and " + config);
                    }
                    consumerConfig = config;
                }
            }
            //设置ConsumerConfig
            if (consumerConfig != null) {
                setConsumer(consumerConfig);
            }
        }
    }
    //设置ApplicationConfig
    if (getApplication() == null
        && (getConsumer() == null || getConsumer().getApplication() == null)) {
        Map<String, ApplicationConfig> applicationConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, ApplicationConfig.class, false, false);
        if (applicationConfigMap != null && applicationConfigMap.size() > 0) {
            ApplicationConfig applicationConfig = null;
            for (ApplicationConfig config : applicationConfigMap.values()) {
                if (config.isDefault() == null || config.isDefault().booleanValue()) {
                    if (applicationConfig != null) {
                        throw new IllegalStateException("Duplicate application configs: " + applicationConfig + " and " + config);
                    }
                    applicationConfig = config;
                }
            }
            if (applicationConfig != null) {
                setApplication(applicationConfig);
            }
        }
    }
    //设置ModuleConfig
    if (getModule() == null
        && (getConsumer() == null || getConsumer().getModule() == null)) {
        Map<String, ModuleConfig> moduleConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, ModuleConfig.class, false, false);
        if (moduleConfigMap != null && moduleConfigMap.size() > 0) {
            ModuleConfig moduleConfig = null;
            for (ModuleConfig config : moduleConfigMap.values()) {
                if (config.isDefault() == null || config.isDefault().booleanValue()) {
                    if (moduleConfig != null) {
                        throw new IllegalStateException("Duplicate module configs: " + moduleConfig + " and " + config);
                    }
                    moduleConfig = config;
                }
            }
            if (moduleConfig != null) {
                setModule(moduleConfig);
            }
        }
    }
    //设置注册中心
    if ((getRegistries() == null || getRegistries().isEmpty())
        && (getConsumer() == null || getConsumer().getRegistries() == null || getConsumer().getRegistries().isEmpty())
        && (getApplication() == null || getApplication().getRegistries() == null || getApplication().getRegistries().isEmpty())) {
        //多个注册中心
        Map<String, RegistryConfig> registryConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, RegistryConfig.class, false, false);
        if (registryConfigMap != null && registryConfigMap.size() > 0) {
            List<RegistryConfig> registryConfigs = new ArrayList<RegistryConfig>();
            for (RegistryConfig config : registryConfigMap.values()) {
                if (config.isDefault() == null || config.isDefault().booleanValue()) {
                    registryConfigs.add(config);
                }
            }
            if (registryConfigs != null && !registryConfigs.isEmpty()) {
                super.setRegistries(registryConfigs);
            }
        }
    }
    //设置监控中心
    if (getMonitor() == null
        && (getConsumer() == null || getConsumer().getMonitor() == null)
        && (getApplication() == null || getApplication().getMonitor() == null)) {
        Map<String, MonitorConfig> monitorConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, MonitorConfig.class, false, false);
        if (monitorConfigMap != null && monitorConfigMap.size() > 0) {
            MonitorConfig monitorConfig = null;
            for (MonitorConfig config : monitorConfigMap.values()) {
                if (config.isDefault() == null || config.isDefault().booleanValue()) {
                    if (monitorConfig != null) {
                        throw new IllegalStateException("Duplicate monitor configs: " + monitorConfig + " and " + config);
                    }
                    monitorConfig = config;
                }
            }
            if (monitorConfig != null) {
                setMonitor(monitorConfig);
            }
        }
    }
    //是否bean创建后就初始化代理
    Boolean b = isInit();
    if (b == null && getConsumer() != null) {
        b = getConsumer().isInit();
    }
    if (b != null && b.booleanValue()) 
        //立即初始化代理  
        getObject();
    }
}
//DisposableBean的方法，做销毁处理
@Override
public void destroy() {
    // do nothing
}
```

#### ServiceBean类

ServiceBean类主要完成在适当时机（Spring容器初始化完成或者服务实例初始化完成）根据服务提供方的配置暴露发布服务的工作，ServiceBean类继承关系如下：

![](https://github.com/shuaijunlan/shuaijunlan.github.io/blob/master/images/ServiceBean.png?raw=true)

可以看到ServiceBean及其基类实现了BeanNameAware、ApplicationContextAware、ApplicationListener、DisposableBean、InitializingBean接口，通过Spring回调机制完成Spring容器引用传入，bean初始化和destory过程定制，以及监听并处理Spring事件的操作，ServiceBean类具体实现如下：

```java
//传入bean所在容器的引用
@Override
public void setApplicationContext(ApplicationContext applicationContext) {
    this.applicationContext = applicationContext;
    //把Spring容器传入SpringExtensionFactory
    SpringExtensionFactory.addApplicationContext(applicationContext);
    //获取容易addApplicationListener方法，把当前类加入到容器监听队列
    if (applicationContext != null) {
        SPRING_CONTEXT = applicationContext;
        try {
            Method method = applicationContext.getClass().getMethod("addApplicationListener", ApplicationListener.class); // backward compatibility to spring 2.0.1
            method.invoke(applicationContext, this);
            supportedApplicationListener = true;
        } catch (Throwable t) {
            if (applicationContext instanceof AbstractApplicationContext) {
                try {
                    Method method = AbstractApplicationContext.class.getDeclaredMethod("addListener", ApplicationListener.class); // backward compatibility to spring 2.0.1
                    if (!method.isAccessible()) {
                        method.setAccessible(true);
                    }
                    method.invoke(applicationContext, this);
                    //设置监听器后设为true
                    supportedApplicationListener = true;
                } catch (Throwable t2) {
                }
            }
        }
    }
}
//设置beanName值
@Override
public void setBeanName(String name) {
    this.beanName = name;
}

/**
     * Gets associated {@link Service}
     *
     * @return associated {@link Service}
     */
public Service getService() {
    return service;
}
//实现ApplicationListener接口方法，接受并处理在容器初始化完成时发布的ContextRefreshedEvent事件
//即容器初始化完成后暴露服务
@Override
public void onApplicationEvent(ContextRefreshedEvent event) {
    if (isDelay() && !isExported() && !isUnexported()) {
        if (logger.isInfoEnabled()) {
            logger.info("The service ready on spring started. service: " + getInterface());
        }
        //执行暴露过程
        export();
    }
}
//判断是否延迟暴露
private boolean isDelay() {
    Integer delay = getDelay();
    ProviderConfig provider = getProvider();
    if (delay == null && provider != null) {
        delay = provider.getDelay();
    }
    return supportedApplicationListener && (delay == null || delay == -1);
}
//InitializinBean接口方法，Bean属性初始化后，操作处理
@Override
@SuppressWarnings({"unchecked", "deprecation"})
public void afterPropertiesSet() throws Exception {
    // 设置ProviderConfig
    if (getProvider() == null) {
        Map<String, ProviderConfig> providerConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, ProviderConfig.class, false, false);
        if (providerConfigMap != null && providerConfigMap.size() > 0) {
            Map<String, ProtocolConfig> protocolConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, ProtocolConfig.class, false, false);
            if ((protocolConfigMap == null || protocolConfigMap.size() == 0)
                && providerConfigMap.size() > 1) { // backward compatibility
                List<ProviderConfig> providerConfigs = new ArrayList<ProviderConfig>();
                for (ProviderConfig config : providerConfigMap.values()) {
                    if (config.isDefault() != null && config.isDefault()) {
                        providerConfigs.add(config);
                    }
                }
                if (!providerConfigs.isEmpty()) {
                    setProviders(providerConfigs);
                }
            } else {
                ProviderConfig providerConfig = null;
                for (ProviderConfig config : providerConfigMap.values()) {
                    if (config.isDefault() == null || config.isDefault()) {
                        if (providerConfig != null) {
                            throw new IllegalStateException("Duplicate provider configs: " + providerConfig + " and " + config);
                        }
                        providerConfig = config;
                    }
                }
                if (providerConfig != null) {
                    setProvider(providerConfig);
                }
            }
        }
    }
    //设置ApplicationConfig
    if (getApplication() == null
        && (getProvider() == null || getProvider().getApplication() == null)) {
        Map<String, ApplicationConfig> applicationConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, ApplicationConfig.class, false, false);
        if (applicationConfigMap != null && applicationConfigMap.size() > 0) {
            ApplicationConfig applicationConfig = null;
            for (ApplicationConfig config : applicationConfigMap.values()) {
                if (config.isDefault() == null || config.isDefault()) {
                    if (applicationConfig != null) {
                        throw new IllegalStateException("Duplicate application configs: " + applicationConfig + " and " + config);
                    }
                    applicationConfig = config;
                }
            }
            if (applicationConfig != null) {
                setApplication(applicationConfig);
            }
        }
    }
    //设置模块
    if (getModule() == null
        && (getProvider() == null || getProvider().getModule() == null)) {
        Map<String, ModuleConfig> moduleConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, ModuleConfig.class, false, false);
        if (moduleConfigMap != null && moduleConfigMap.size() > 0) {
            ModuleConfig moduleConfig = null;
            for (ModuleConfig config : moduleConfigMap.values()) {
                if (config.isDefault() == null || config.isDefault()) {
                    if (moduleConfig != null) {
                        throw new IllegalStateException("Duplicate module configs: " + moduleConfig + " and " + config);
                    }
                    moduleConfig = config;
                }
            }
            if (moduleConfig != null) {
                setModule(moduleConfig);
            }
        }
    }
    //设置注册中心
    if ((getRegistries() == null || getRegistries().isEmpty())
        && (getProvider() == null || getProvider().getRegistries() == null || getProvider().getRegistries().isEmpty())
        && (getApplication() == null || getApplication().getRegistries() == null || getApplication().getRegistries().isEmpty())) {
        Map<String, RegistryConfig> registryConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, RegistryConfig.class, false, false);
        if (registryConfigMap != null && registryConfigMap.size() > 0) {
            List<RegistryConfig> registryConfigs = new ArrayList<RegistryConfig>();
            for (RegistryConfig config : registryConfigMap.values()) {
                if (config.isDefault() == null || config.isDefault()) {
                    registryConfigs.add(config);
                }
            }
            if (!registryConfigs.isEmpty()) {
                super.setRegistries(registryConfigs);
            }
        }
    }
    //设置监控中心
    if (getMonitor() == null
        && (getProvider() == null || getProvider().getMonitor() == null)
        && (getApplication() == null || getApplication().getMonitor() == null)) {
        Map<String, MonitorConfig> monitorConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, MonitorConfig.class, false, false);
        if (monitorConfigMap != null && monitorConfigMap.size() > 0) {
            MonitorConfig monitorConfig = null;
            for (MonitorConfig config : monitorConfigMap.values()) {
                if (config.isDefault() == null || config.isDefault()) {
                    if (monitorConfig != null) {
                        throw new IllegalStateException("Duplicate monitor configs: " + monitorConfig + " and " + config);
                    }
                    monitorConfig = config;
                }
            }
            if (monitorConfig != null) {
                setMonitor(monitorConfig);
            }
        }
    }
    //服务协议，可以有多个
    if ((getProtocols() == null || getProtocols().isEmpty())
        && (getProvider() == null || getProvider().getProtocols() == null || getProvider().getProtocols().isEmpty())) {
        Map<String, ProtocolConfig> protocolConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, ProtocolConfig.class, false, false);
        if (protocolConfigMap != null && protocolConfigMap.size() > 0) {
            List<ProtocolConfig> protocolConfigs = new ArrayList<ProtocolConfig>();
            for (ProtocolConfig config : protocolConfigMap.values()) {
                if (config.isDefault() == null || config.isDefault()) {
                    protocolConfigs.add(config);
                }
            }
            if (!protocolConfigs.isEmpty()) {
                super.setProtocols(protocolConfigs);
            }
        }
    }
    //设置服务路径（类全名）
    if (getPath() == null || getPath().length() == 0) {
        if (beanName != null && beanName.length() > 0
            && getInterface() != null && getInterface().length() > 0
            && beanName.startsWith(getInterface())) {
            setPath(beanName);
        }
    }
    //是否延迟暴露
    if (!isDelay()) {
        //暴露服务
        export();
    }
}
```

#### AnnotationBean类

这个类使得dubbo具有自动包扫描功能支持dubbo通过注解配置service和Reference bean（有些属性不能注解配置），病完成ServiceBean和ReferenceBean相同的功能，类图如下：

![](https://github.com/shuaijunlan/shuaijunlan.github.io/blob/master/images/AnnotationBean.png?raw=true)

可以看到AnnotationBean类实现了接口DisposableBean、BeanFactoryPostProcessor、BeanPostProcessor、ApplicationContextAware，同样用过Spring接口方法回调，实现Bean实例的初始化预处理。

AnnotationBean类是基于ClassPathBeanDefinationScanner类实现的，看下`org.springframework.context.annotation.ClassPathBeanDefinationScanner`类官方解释[ClassPathBeanDefinationScanner](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/context/annotation/ClassPathBeanDefinitionScanner.html)：

```
A bean definition scanner that detects bean candidates on the classpath, registering corresponding bean definitions with a given registry (BeanFactory or ApplicationContext).
Candidate classes are detected through configurable type filters. The default filters include classes that are annotated with Spring's @Component, @Repository, @Service, or @Controller stereotype.

Also supports Java EE 6's ManagedBean and JSR-330's Named annotations, if available.
```

大概意思就是，ClassPathBeanDefinitionScanner将会扫描classpath下的bean，并且向注册器注册（BeanFactory或者ApplicationContext）bean definition，通过配置的过滤器检测bean，默认会检测被@Component, @Repository, @Service, or @Controller注解的类。

下面来分析AnnotationBean类的核心代码：

```java
//设置扫描的包名，以逗号分隔包名
public void setPackage(String annotationPackage) {
    this.annotationPackage = annotationPackage;
    this.annotationPackages = (annotationPackage == null || annotationPackage.length() == 0) ? null
        : Constants.COMMA_SPLIT_PATTERN.split(annotationPackage);
}
//实现Spring回调接口方法，传入容器引用
@Override
public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
    this.applicationContext = applicationContext;
}
//实现BeanFactoryPostProcessor接口，这个方法会在所有的bean definitions已加载，但是还没有实例化之前回调执行
//可以在Bean初始化之前定制化一些操作，这里做的是调用org.springframework.context.annotation.ClassPathBeanDefinitionScanner的scan方法
//扫描注册由Service(Dubbo定义)注解的Bean，都是用反射完成的
@Override
public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory)
    throws BeansException {
    if (annotationPackage == null || annotationPackage.length() == 0) {
        return;
    }
    if (beanFactory instanceof BeanDefinitionRegistry) {
        try {
            // init scanner
            //利用反射构造ClassPathBeanDefinitionScanner实例，用的这个构造方法，
            // useDefaultFilters=true 默认扫描 spring 4种的注解
            // public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry, boolean useDefaultFilters) {
            //		this(registry, useDefaultFilters, getOrCreateEnvironment(registry));
            //	}
            Class<?> scannerClass = ReflectUtils.forName("org.springframework.context.annotation.ClassPathBeanDefinitionScanner");
            Object scanner = scannerClass.getConstructor(new Class<?>[]{BeanDefinitionRegistry.class, boolean.class}).newInstance((BeanDefinitionRegistry) beanFactory, true);
            // add filter
            //通过filter 添加新要扫描的注解，也是用的反射 这里是 AnnotationTypeFilte
            Class<?> filterClass = ReflectUtils.forName("org.springframework.core.type.filter.AnnotationTypeFilter");
            Object filter = filterClass.getConstructor(Class.class).newInstance(Service.class);
            //获取添加filter的方法，并调用
            Method addIncludeFilter = scannerClass.getMethod("addIncludeFilter", ReflectUtils.forName("org.springframework.core.type.filter.TypeFilter"));
            addIncludeFilter.invoke(scanner, filter);
            // scan packages
            //获取ClassPathBeanDefinitionScanner的scan()方法，开始扫描
            String[] packages = Constants.COMMA_SPLIT_PATTERN.split(annotationPackage);
            Method scan = scannerClass.getMethod("scan", String[].class);
            scan.invoke(scanner, new Object[]{packages});
        } catch (Throwable e) {
            // spring 2.0
        }
    }
}
//实现DisposableBean接口，在bean析构时，调用相关方法，释放资源
@Override
public void destroy() {

    //  This will only be called for singleton scope bean, and expected to be called by spring shutdown hook when BeanFactory/ApplicationContext destroys.
    //  We will guarantee dubbo related resources being released with dubbo shutdown hook.

    //  for (ServiceConfig<?> serviceConfig : serviceConfigs) {
    //      try {
    //          serviceConfig.unexport();
    //      } catch (Throwable e) {
    //          logger.error(e.getMessage(), e);
    //      }
    //  }

    for (ReferenceConfig<?> referenceConfig : referenceConfigs.values()) {
        try {
            referenceConfig.destroy();
        } catch (Throwable e) {
            logger.error(e.getMessage(), e);
        }
    }
}
//实现BeanPostProcessor接口方法，在Bean初始化之后，比如在afterPropertiesSet后由Spring回调执行
//这个方法完成类似ServiceBean的工作
@Override
public Object postProcessAfterInitialization(Object bean, String beanName)
    throws BeansException {
    //检查是否匹配包名
    if (!isMatchPackage(bean)) {
        return bean;
    }
    //手动创建ServiceBean并暴露服务
    Service service = bean.getClass().getAnnotation(Service.class);
    if (service != null) {
        ServiceBean<Object> serviceConfig = new ServiceBean<Object>(service);
        serviceConfig.setRef(bean);
        if (void.class.equals(service.interfaceClass())
            && "".equals(service.interfaceName())) {
            if (bean.getClass().getInterfaces().length > 0) {
                serviceConfig.setInterface(bean.getClass().getInterfaces()[0]);
            } else {
                throw new IllegalStateException("Failed to export remote service class " + bean.getClass().getName() + ", cause: The @Service undefined interfaceClass or interfaceName, and the service class unimplemented any interfaces.");
            }
        }
        if (applicationContext != null) {
            serviceConfig.setApplicationContext(applicationContext);
            if (service.registry().length > 0) {
                List<RegistryConfig> registryConfigs = new ArrayList<RegistryConfig>();
                for (String registryId : service.registry()) {
                    if (registryId != null && registryId.length() > 0) {
                        registryConfigs.add(applicationContext.getBean(registryId, RegistryConfig.class));
                    }
                }
                serviceConfig.setRegistries(registryConfigs);
            }
            if (service.provider().length() > 0) {
                serviceConfig.setProvider(applicationContext.getBean(service.provider(), ProviderConfig.class));
            }
            if (service.monitor().length() > 0) {
                serviceConfig.setMonitor(applicationContext.getBean(service.monitor(), MonitorConfig.class));
            }
            if (service.application().length() > 0) {
                serviceConfig.setApplication(applicationContext.getBean(service.application(), ApplicationConfig.class));
            }
            if (service.module().length() > 0) {
                serviceConfig.setModule(applicationContext.getBean(service.module(), ModuleConfig.class));
            }
            if (service.provider().length() > 0) {
                serviceConfig.setProvider(applicationContext.getBean(service.provider(), ProviderConfig.class));
            }
            if (service.protocol().length > 0) {
                List<ProtocolConfig> protocolConfigs = new ArrayList<ProtocolConfig>();
                for (String protocolId : service.protocol()) {
                    if (protocolId != null && protocolId.length() > 0) {
                        protocolConfigs.add(applicationContext.getBean(protocolId, ProtocolConfig.class));
                    }
                }
                serviceConfig.setProtocols(protocolConfigs);
            }
            if (service.tag().length() > 0) {
                serviceConfig.setTag(service.tag());
            }
            try {
                serviceConfig.afterPropertiesSet();
            } catch (RuntimeException e) {
                throw e;
            } catch (Exception e) {
                throw new IllegalStateException(e.getMessage(), e);
            }
        }
        serviceConfigs.add(serviceConfig);
        serviceConfig.export();
    }
    return bean;
}
//实现BeanPostProcessor接口方法，在Bean初始化前，比如在afterPropertiesSet前，由Spring回调执行
//这个方法完成类似ReferenceBean的工作
@Override
public Object postProcessBeforeInitialization(Object bean, String beanName)
    throws BeansException {
    if (!isMatchPackage(bean)) {
        return bean;
    }
    //因为Dubbo Reference注解只能在类的字段或者方法上
    //通过Bean的set方法上找dubbo注解
    Method[] methods = bean.getClass().getMethods();
    for (Method method : methods) {
        String name = method.getName();
        if (name.length() > 3 && name.startsWith("set")
            && method.getParameterTypes().length == 1
            && Modifier.isPublic(method.getModifiers())
            && !Modifier.isStatic(method.getModifiers())) {
            try {
                Reference reference = method.getAnnotation(Reference.class);
                if (reference != null) {
                    Object value = refer(reference, method.getParameterTypes()[0]);
                    if (value != null) {
                        method.invoke(bean, value);
                    }
                }
            } catch (Throwable e) {
                logger.error("Failed to init remote service reference at method " + name + " in class " + bean.getClass().getName() + ", cause: " + e.getMessage(), e);
            }
        }
    }
    //通过bean的字段上找dubbo注解
    Field[] fields = bean.getClass().getDeclaredFields();
    for (Field field : fields) {
        try {
            if (!field.isAccessible()) {
                field.setAccessible(true);
            }
            Reference reference = field.getAnnotation(Reference.class);
            if (reference != null) {
                Object value = refer(reference, field.getType());
                if (value != null) {
                    field.set(bean, value);
                }
            }
        } catch (Throwable e) {
            logger.error("Failed to init remote service reference at filed " + field.getName() + " in class " + bean.getClass().getName() + ", cause: " + e.getMessage(), e);
        }
    }
    return bean;
}
//通过解析Reference注解里的值，去构造服务调用配置，最后调用创建代理的方法
private Object refer(Reference reference, Class<?> referenceClass) { //method.getParameterTypes()[0]
    String interfaceName;
    if (!"".equals(reference.interfaceName())) {
        interfaceName = reference.interfaceName();
    } else if (!void.class.equals(reference.interfaceClass())) {
        interfaceName = reference.interfaceClass().getName();
    } else if (referenceClass.isInterface()) {
        interfaceName = referenceClass.getName();
    } else {
        throw new IllegalStateException("The @Reference undefined interfaceClass or interfaceName, and the property type " + referenceClass.getName() + " is not a interface.");
    }
    String key = reference.group() + "/" + interfaceName + ":" + reference.version();
    ReferenceBean<?> referenceConfig = referenceConfigs.get(key);
    if (referenceConfig == null) {
        referenceConfig = new ReferenceBean<Object>(reference);
        if (void.class.equals(reference.interfaceClass())
            && "".equals(reference.interfaceName())
            && referenceClass.isInterface()) {
            referenceConfig.setInterface(referenceClass);
        }
        if (applicationContext != null) {
            referenceConfig.setApplicationContext(applicationContext);
            if (reference.registry().length > 0) {
                List<RegistryConfig> registryConfigs = new ArrayList<RegistryConfig>();
                for (String registryId : reference.registry()) {
                    if (registryId != null && registryId.length() > 0) {
                        registryConfigs.add(applicationContext.getBean(registryId, RegistryConfig.class));
                    }
                }
                referenceConfig.setRegistries(registryConfigs);
            }
            if (reference.consumer().length() > 0) {
                referenceConfig.setConsumer(applicationContext.getBean(reference.consumer(), ConsumerConfig.class));
            }
            if (reference.monitor().length() > 0) {
                referenceConfig.setMonitor(applicationContext.getBean(reference.monitor(), MonitorConfig.class));
            }
            if (reference.application().length() > 0) {
                referenceConfig.setApplication(applicationContext.getBean(reference.application(), ApplicationConfig.class));
            }
            if (reference.module().length() > 0) {
                referenceConfig.setModule(applicationContext.getBean(reference.module(), ModuleConfig.class));
            }
            if (reference.consumer().length() > 0) {
                referenceConfig.setConsumer(applicationContext.getBean(reference.consumer(), ConsumerConfig.class));
            }
            try {
                referenceConfig.afterPropertiesSet();
            } catch (RuntimeException e) {
                throw e;
            } catch (Exception e) {
                throw new IllegalStateException(e.getMessage(), e);
            }
        }
        referenceConfigs.putIfAbsent(key, referenceConfig);
        referenceConfig = referenceConfigs.get(key);
    }
    return referenceConfig.get();
}

private boolean isMatchPackage(Object bean) {
    if (annotationPackages == null || annotationPackages.length == 0) {
        return true;
    }
    String beanClassName = bean.getClass().getName();
    for (String pkg : annotationPackages) {
        if (beanClassName.startsWith(pkg)) {
            return true;
        }
    }
    return false;
}
```

