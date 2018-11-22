---
title: Spring中Bean的初始化与销毁（基于Spring4.x）
date: 2016-10-26 10:07:15
tags:
    - Spring
---

1. 通过在bean中设置init-method和destroy-method

    > 配置bean</br>
    > spring-lifecycle.xml

    ```xml
    <?xml version="1.0" encoding="UTF-8"?>
    <beans xmlns="http://www.springframework.org/schema/beans"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
        <bean id="beanLifeCycle" class="com.sh.imcdemo.services.impl.BeanLifeCycle" init-method="start" destroy-method="stop"></bean>
    </beans>
    ```

    > com.sh.imcdemo.services.impl 实现类

<!-- more -->

    ```java
    package com.sh.imcdemo.services.impl;
    /**
     * Created by Mr SJL on 2016/11/26.
     *
     * @Author Junlan Shuai
     */
    public class BeanLifeCycle
    {
        public void start()
        {
            System.out.println("Bean start.");
        }
        public void stop()
        {
            System.out.println("Bean stop.");
        }
    }
    ```

2. 通过实现InitializingBean和DisposableBean接口

    > 配置bean</br>
    > spring-lifecycle.xml

    ```xml
    <?xml version="1.0" encoding="UTF-8"?>
    <beans xmlns="http://www.springframework.org/schema/beans"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
        <bean id="beanLifeCycle1" class="com.sh.imcdemo.services.impl.BeanLifeCycle"></bean>
    </beans>
    ```

    > com.sh.imcdemo.services.impl实现类

    ```java
    package com.sh.imcdemo.services.impl;

    import org.springframework.beans.factory.DisposableBean;
    import org.springframework.beans.factory.InitializingBean;

    /**
     * Created by Mr SJL on 2016/11/26.
     *
     * @Author Junlan Shuai
     */
    public class BeanLifeCycle implements InitializingBean, DisposableBean
    {

        public void destroy() throws Exception
        {
            System.out.println("Bean destory.");
        }

        public void afterPropertiesSet() throws Exception
        {
            System.out.println("Bean afterPropertiesSet.");

        }
    }

    ```

3. 通过设置default-destroy-method和default-init-method

    > 对于同一配置文件下的所有的Bean都会使用该默认的初始化和销毁方法（但有特殊情况，见本篇总结部分）

    ```xml
    <?xml version="1.0" encoding="UTF-8"?>
    <beans xmlns="http://www.springframework.org/schema/beans"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd"
            default-destroy-method="defaultDestroy" default-init-method="defaultInit">
        <bean id="beanLifeCycle" class="com.sh.imcdemo.services.impl.BeanLifeCycle"></bean>
    </beans>
    ```

    > com.sh.imcdemo.services.impl

    ```java
    package com.sh.imcdemo.services.impl;

    /**
     * Created by Mr SJL on 2016/11/26.
     *
     * @Author Junlan Shuai
     */
    public class BeanLifeCycle
    {
        public void defaultInit()
        {
            System.out.println("Bean defaultInit.");
        }
        public void defaultDestroy()
        {
            System.out.println("Bean defaultDestory");
        }
    }
    ```
4. 总结

    * 当三种方式同时使用时，我们会发现，第三种方式被覆盖了，另外两种方式的输出先后顺序是：先是2再是1。
    * 当使用第3种方式时，实现类中不一定非要实现该默认方法，如果没有该方法，则没有处理。
    * 当第2种和第3中方式同时使用时，默认方法却没有被覆盖，两者都会输出，但是第1种和第3种同时使用时，默认方法却被覆盖了。（？？？）

### 附录

> 测试基类 com.sh.imcdemo.unitTest

```java
package com.sh.imcdemo.unitTest;
import org.apache.commons.lang.StringUtils;
import org.junit.After;
import org.junit.Before;
import org.springframework.beans.BeansException;
import org.springframework.context.support.ClassPathXmlApplicationContext;
/**
 * Created by Mr SJL on 2016/11/26.
 *
 * @Author Junlan Shuai
 */
public class UnitTestBase
{
    private ClassPathXmlApplicationContext context;

    private String springXmlpath;
    public UnitTestBase()
    {

    }
    public UnitTestBase(String springXmlpath)
    {
        this.springXmlpath = springXmlpath;
    }
    @Before
    public void before()
    {
        if (StringUtils.isEmpty(springXmlpath))
        {
            springXmlpath = "classpath*:spring-*.xml";
        }
        try
        {
            context = new ClassPathXmlApplicationContext(springXmlpath.split("[,\\s]+"));
            context.start();
        }
        catch (BeansException e)
        {
            e.printStackTrace();
        }

    }
    @After
    public void after()
    {
        context.destroy();
    }

    protected <T extends Object> T getBean(String beanId)
    {
        return (T)context.getBean(beanId);
    }
    protected <T extends Object> T getBean(Class<T> clas)
    {
        return context.getBean(clas);
    }

}

```
> 测试类com.sh.imcdemo.unitTest

```java
package com.sh.imcdemo.unitTest;

import org.junit.Test;
import org.junit.runner.RunWith;
import org.junit.runners.BlockJUnit4ClassRunner;

/**
 * Created by Mr SJL on 2016/11/26.
 *
 * @Author Junlan Shuai
 */
@RunWith(BlockJUnit4ClassRunner.class)
public class App3 extends UnitTestBase
{
    public App3()
    {
        super("classpath:spring-lifecycle.xml");
    }
    @Test
    public void test1()
    {
        super.getBean("beanLifeCycle");
    }

    @Test
    public void test2()
    {
        super.getBean("beanLifeCycle1");
    }

}
```

> 依赖包pom.xml

```xml
<spring.version>4.3.2.RELEASE</spring.version>
<junit.version>4.11</junit.version>


<!-- Spring依赖包-->
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-core</artifactId>
    <version>${spring.version}</version>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>${spring.version}</version>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-aop</artifactId>
    <version>${spring.version}</version>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-beans</artifactId>
    <version>${spring.version}</version>
</dependency>

<!-- 单元测试包 -->
<dependency>
    <groupId>junit</groupId>
    <artifactId>junit</artifactId>
    <version>${junit.version}</version>
</dependency>
```
