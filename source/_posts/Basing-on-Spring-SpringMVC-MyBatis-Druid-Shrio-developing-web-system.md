---
title: Basing on Spring SpringMVC MyBatis Druid Shrio developing web system
date: 2017-11-03 11:46:39
tags:
    - WebSystem
categories:
    - WebSystem
---
源码下载地址：[https://github.com/shuaijunlan/Autumn-Framework](https://github.com/shuaijunlan/Autumn-Framework)

在线Demo：[http://autumn.shuaijunlan.cn](http://autumn.shuaijunlan.cn)

#### 项目介绍
`Autumn-Framework`旨在提供通用的web系统解决方案，目前由作者本人一个人维护，更新速度缓慢，但是会持续更新，此项目适合初学者学习使用，也欢迎您加入我一起维护整个项目。

<!-- more -->

#### 效果图
* 登录界面
  ![登录界面][login]
* 系统主界面
  ![系统主界面][main]
* 菜单管理
  ![菜单管理][menu]
* 日志管理
  ![日志管理][log]

  ![日志管理][log1]

#### 技术选型
前端以`Layui`为主要框架，并使用了`ECharts`、`editor.md`等其他第三方插件</br>
后端主要使用`Spring`、`SpringMVC`、`MyBatis`、`Shiro`、`Druid`、`Ehcache`构建整个web系统，并使用Maven管理项目，使用Mysql存储数据，使用tomcat部署web系统。

#### 代码结构
```
.
└── src-------------------------------------------源码根目录
    └── main
        ├── java
        │   └── com
        │       └── autumnframework
        │           └── cms
        │               ├── architect-------------包含常用的工具类和常量
        │               │   ├── conf
        │               │   ├── constant
        │               │   ├── filter
        │               │   ├── interceptor
        │               │   └── utils
        │               ├── controller------------控制器层
        │               │   └── system
        │               ├── dao-------------------dao层
        │               │   ├── bomapper
        │               │   └── vomapper
        │               │       ├── impl
        │               │       └── interfaces
        │               ├── model-----------------model层
        │               │   ├── bo
        │               │   ├── po
        │               │   └── vo
        │               ├── service---------------service层
        │               │   ├── impl
        │               │   └── interfaces
        │               └── shiroconfig-----------shiro配置
        │                   ├── filter
        │                   └── realm
        ├── resources----------------------------资源文件目录
        │   ├── mapperxml------------------------mapper映射文件
        │   ├── mybatis-generator----------------mybatis-generator配置文件
        │   └── spring---------------------------所有与spring相关的配置文件
        └── webapp-------------------------------前端源码文件
            ├── BasePlu--------------------------公共库
            ├── comm
            ├── Lib------------------------------第三方库
            │   ├── Echarts-3.7.2
            │   ├── editor.md
            │   ├── jquery
            │   └── layui_v2.1.2
            ├── static--------------------------静态资源
            ├── Sys-----------------------------系统功能插件目录
            │   ├── js
            │   └── plugin
            └── WEB-INF
                └── views
                    ├── error-------------------异常目录
                    └── main--------------------系统主界面目录
```

#### 运行系统
* 拷贝代码到本地`git clone git@github.com:shuaijunlan/Autumn-Framework.git`
* 进入Autumn-Framework目录`cd Autumn-Framework`
* 执行`mvn install`
* 再进入cms目录`cd cms`
* 在执行`mvn tomcat7:run`
* 最后在浏览器中访问`localhost:8081`，就可以看到登录界面
* Tips：以上所有操作基于您的电脑已经安装了`jdk8`、`maven`和`git`环境

#### FAQ


#### 联系作者
您有任何问题都可以随时联系我！ 

Email：shuaijunlan@gmail.com

-----------------------------
[login]:https://github.com/shuaijunlan/shuaijunlan.github.io/blob/master/images/login-page.png?raw=true "登录界面"
[main]:https://github.com/shuaijunlan/shuaijunlan.github.io/blob/master/images/main-page.png?raw=true  "系统主界面""
[menu]:https://github.com/shuaijunlan/shuaijunlan.github.io/blob/master/images/menu-manage.png?raw=true  "菜单管理""
[log]:https://github.com/shuaijunlan/shuaijunlan.github.io/blob/master/images/log-manage1.png?raw=true "日志管理"
[log1]:https://github.com/shuaijunlan/shuaijunlan.github.io/blob/master/images/log-manage2.png?raw=true  "日志管理"

