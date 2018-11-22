---
title: CentOS7中配置KVM教程
date: 2016-11-20 10:07:15
tags:
    - KVM
    - CentOS
---

> 基本环境：CentOS7.0

1. [[root@localhost /]# ]()egrep '(vmx|svm)' /proc/cpuinfo  
    > 和 Xen 不同，KVM 需要有 CPU 的支持（Intel VT 或 AMD SVM），在安装 KVM 之前检查一下 CPU 是否提供了虚拟技术的支持,可以通过下面命令查询是否支持，如果输出有相关的vmx或者svm，表明CPU支持，否则就不支持。

<!-- more -->

2. [[root@localhost /]# ]()yum install qemu-kvm qemu-img virt-manager libvirt libvirt-python python-virtinst libvirt-client virt-install virt-viewer

    > kvm相关安装包及其作用（按需选择安装）
    > * qemu-kvm：qemu模拟器,主要的KVM程序包
    > * qemu-img：qemu磁盘image管理器
    > * virt-install：基于libvirt服务的虚拟机创建命令，用来创建虚拟机的命令行工具
    > * libvirt：提供libvirtd daemon来管理虚拟机和控制hypervisor
    > * libvirt-client：提供客户端API用来访问server和提供管理虚拟机命令行工具的virsh实体
    > * python-virtinst：创建虚拟机所需要的命令行工具和程序库
    > * virt-manager：GUI虚拟机管理工具
    > * virt-top：虚拟机统计命令
    > * virt-viewer：GUI连接程序，连接到已配置好的虚拟机
    > * bridge-utils：创建和管理桥接设备的工具

3. 验证是否安装成功

    * 验证内核模块是否加载
        > [[root@localhost /]# ]()lsmod | grep kvm

    * 启动服务(同时设置了开机自启)

        > [[root@localhost /]# ]()systemctl start libvirtd.service

    * 重启服务

        > [[root@localhost /]# ]()systemctl restart libvirtd.service

    * 设置可用

        > [[root@localhost /]# ]()systemctl enable libvirtd.service

    * 查看服务基本信息

        > [[root@localhost /]# ]()systemctl status libvirtd.service

    ![](http://learningnotes-1251679769.costj.myqcloud.com/virtual/6.png)

3. 配置虚拟机网络

    > 见<a href="http://www.blog.shuaijunlan.cn/2016/12/07/libvirt%20kvm%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%BD%91%E7%BB%9C%E9%85%8D%E7%BD%AE/" target="\_blank">《libvirt kvm虚拟机网络配置》</a>一文

4. 在CentOS上安装vnc服务，通过vnc客户端远程连接CentOS，通过图形化界面在宿主机上安装客户机，安装教程见《使用VNC工具初级教程》一文

5. 最后安装客户机

    > * 下载虚拟机要安装的ISO系统镜像文件，之后需创建存储池，指定在宿主机上虚拟机磁盘的存储位置，创建存储目录：(目录随便设定，按照自己的需求设定)
        * [[root@localhost /]# ]()mkdir -p /home/kvm1

    > * 定义一个储存池和绑定目录：
        * [[root@localhost /]# ]()virsh pool-define-as vmspool --type dir --target /home/kvm1

    > * 建立并激活存储池：
        * [[root@localhost /]# ]()virsh pool-build vmspool
        * [[root@localhost /]# ]()virsh pool-start vmspool

    > * virsh-install:
        * 1、输入虚拟机名称
        * 2、分配多少内存
        * 3、处理器的个数
        * 4、此步可以直接输入iso的位置或是url
        * 5、虚拟机类型KVM
        * 6、定义虚拟机磁盘映像的位置
        * 7、磁盘的大小
        * 6、指定哪个桥或者可以指定多个桥
        * 7、额外的控制台和KS文件
        * 8、连接到系统参数
        * 参数说明注意每行都要空格
        * -n 虚拟机名称
        * -r 分配虚拟机内存大小
        * --vcpus 分配虚拟cpu个数
        * -c 镜像文件位置
        * --vnc --vncport=5901 --vnclisten=0.0.0.0 启动图形安装界面
        * --virt-type 虚拟机模式
        * -f 虚拟机系统文件存储目录
        * -s 分配磁盘大小（GB）
        * -w 联网方式（birdge bridge:br0/nat bridge:virbr0）
        * --os-type='windows' --os-variant=win2k3 安装windows最好加上这个否则会报错
        * virt-install工具安装虚拟机后，在目录/etc/libvirt/qemu/下生成xml配置文件
        * -s 用来指定虚拟磁盘的大小单位为GB
        * -m 指定虚拟网卡的硬件地址默认virt-install自动产生
        * -p 以半虚拟化方式建立虚拟机
        * -l 指定安装来源
        * -x EXTRA, --extra-args=EXTRA当执行从"--location"选项指定位置的客户机安装时，附加内核命令行参数到安装程序。
        * -v, --hvm 设置全虚拟化
        </br>
        </br>

    > * 创建第一个guest：
        * [[root@localhost /]# ]()virt-install --name=CentOS7-1 --ram=4096 --vcpus=1 --cdrom=/home/kvm1/CentOS-7-x86_64-Minimal-1511.iso --virt-type=kvm --disk  path=/home/kvm1/centos7-1.img,device=disk,format=qcow2,bus=virtio,cache=writeback,size=250 --graphics vnc,listen=0.0.0.0,port=5920,password=123456 --network bridge:br0 --accelerate --force --autostart

    > * 创建第二个guest：
        * [[root@localhost /]# ]()virt-install --name=CentOS7-2 --ram=7168 --vcpus=2,maxvcpus=4 --cdrom=/home/kvm1/CentOS-7-x86_64-Minimal-1511.iso --virt-type=kvm --disk  path=/home/kvm1/centos7-2.img,device=disk,format=qcow2,bus=virtio,cache=writeback,size=400 --graphics vnc,listen=0.0.0.0,port=5921,password=123456 --network bridge:br0 --accelerate --force --autostart

    > * 创建第三个guest：
        * [[root@localhost /]# ]()virt-install --name=CentOS7-3 --ram=7168 --vcpus=6,maxvcpus=9 --cpuset=6,7,8,9,10,11 --cdrom=/home/kvm1/CentOS-7-x86_64-Minimal-1511.iso --virt-type=kvm --disk  path=/home/kvm1/centos7-3.img,device=disk,format=qcow2,bus=virtio,cache=writeback,size=400 --graphics vnc,listen=0.0.0.0,port=5922,password=123456 --network bridge:br0 --accelerate --force --autostart

    > * 创建第四个guest：
        * [[root@localhost /]# ]()virt-install --name=CentOS7-4 --ram=7168 --vcpus=6,maxvcpus=9 --cpuset=12,13,14,15,16,17 --cdrom=/home/kvm1/CentOS-7-x86_64-Minimal-1511.iso --virt-type=kvm --disk  path=/home/kvm1/centos7-4.img,device=disk,format=qcow2,bus=virtio,cache=writeback,size=400 --graphics vnc,listen=0.0.0.0,port=5923,password=123456 --network bridge:br0 --accelerate --force --autostart

#### 持续更新中......
-----

* REFERENCES

    1. <a href="http://blog.csdn.net/chdhust/article/details/7931717" target="\_blank">KVM虚拟机创建功能详细讲解</a>
    2. <a href="http://blog.csdn.net/zhaihaifei/article/details/51163750" target="\_blank">绑定KVM虚拟机的vcpu与物理CPU</a>
    3. <a href="https://www.teakki.com/p/57dbd24a1b9882de17e7fd37" target="\_blank">基于Linux命令行KVM虚拟机的安装配置与基本使用</a>
    4. <a href="http://kulezhaizhuren.lofter.com/post/1cca7553_7e20b47" target="\_blank">CENTOS7 安装KVM笔记之安装</a>
    5. <a href="https://www.52os.net/articles/linux-kvm-install-virtual-machines.html" target="\_blank">linux下配置和安装KVM虚拟机</a>
