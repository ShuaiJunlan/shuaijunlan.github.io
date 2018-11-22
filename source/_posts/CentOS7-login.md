---
title: CentOS7设置免密登陆
date: 2016-11-26 10:07:15
tags:
    - CentOS
---

* 基本环境
    * master(centOS7-4:192.168.1.75)
    * slave1(CentOS7-1:192.168.1.21)
    * slave2(CentOS7-2:192.168.1.129)

* 前提条件

    > 要保证这三台机器之间可以互相ping通

<!-- more -->

* 基本配置
    > 在slave1机器上输入命令：vi /etc/ssh/sshd_config

    > 在master机器上输入命令：vi /etc/ssh/sshd_config

    ![image]( http://learningnotes-1251679769.costj.myqcloud.com/linux/8.png  "图片")

* 在master和slave1上建立相同的用户，此文章以root用户为例，读者可以自行创建其他用户。

    1. 登陆到master机器上
    2. 执行命令：mkdir .ssh（创建.ssh文件夹），如果存在此文件夹可以不用创建
    3. 进入到.ssh目录（执行命令：cd .ssh）,并执行命令：ssh-keygen -t rsa，并一直回车，出现以下结果：

        ![image]( http://learningnotes-1251679769.costj.myqcloud.com/linux/9.png  "图片")

        > 可以看到在.ssh目录下面生成了两个文件：id_rsa（私钥）和id_rsa.pub（公钥）两个文件

    4. 使用root用户登陆slave1，同样执行1-3步。

    5. 合并id_rsa.pub，追加到authorized_key文件中
        * root登录master, 在“.ssh”文件夹下，执行命令：scp id_rsa.pub  root@slave1:~/.ssh/authorized_keys

            ![image]( http://learningnotes-1251679769.costj.myqcloud.com/linux/10.png  "图片")

            > 拷贝master的公钥id_rsa.pub到slave1的.ssh/authorized_keys。此过程会要求输入密码。

    6. test登录slave,在“.ssh”文件夹下，输入命令：cat id_rsa.pub >> authorized.keys,把slave1的公钥id_rsa.pub追加到slave的authorized_keys文件。

    7. 在slave1的“.ssh”文件夹下，复制authorized_keys到master的root，命令“scp authorized_keys root@master:~/.ssh/"，此时，master “.ssh”文件夹下，已经存在与slave1相同的authorized_keys文件

* 测试登陆

    >输入命令：ssh master，登陆到master系统

    ![image]( http://learningnotes-1251679769.costj.myqcloud.com/linux/11.png  "图片")

    > 输入命令：exit，退出系统
