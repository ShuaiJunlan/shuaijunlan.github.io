---
title: Dubbo基于Javassist实现动态代理分析
date: 2018-08-02 16:22:08
tags:
    - dubbo
    - javassist
---

```
org.apache.dubbo.common.bytecode.Proxy	//生成代理类
org.apache.dubbo.common.bytecode.ClassGenerator  //基于javassist封装

org.apache.dubbo.common.utils.ClassHelper;  //工具类
org.apache.dubbo.common.utils.ReflectUtils;	//工具类
```

```java
// create ProxyInstance class.
//生成代理类的实例
String pcn = pkg + ".proxy" + id;
ccp.setClassName(pcn);
ccp.addField("public static java.lang.reflect.Method[] methods;");
ccp.addField("private " + InvocationHandler.class.getName() + " handler;");
ccp.addConstructor(Modifier.PUBLIC, new Class<?>[]{InvocationHandler.class}, new Class<?>[0], "handler=$1;");
ccp.addDefaultConstructor();
Class<?> clazz = ccp.toClass();
clazz.getField("methods").set(null, methods.toArray(new Method[0]));

// create Proxy class.
//生成代理类，通过调用newInstance()方法获取实例
String fcn = Proxy.class.getName() + id;
ccm = ClassGenerator.newInstance(cl);
ccm.setClassName(fcn);
ccm.addDefaultConstructor();
ccm.setSuperClass(Proxy.class);
ccm.addMethod("public Object newInstance(" + InvocationHandler.class.getName() + " h){ return new " + pcn + "($1); }");
Class<?> pc = ccm.toClass();
proxy = (Proxy) pc.newInstance();
```