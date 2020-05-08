---
title: "centos 7安装kvm教程"
date: 2020-05-08T22:46:52+08:00
lastmod: 2020-05-08T22:46:52+08:00
draft: false
tags: ["centos7","kvm"]
categories: ["centos7","kvm"]
author: "铁血执着的青春"

---

## 前言
之前一直使用单机来搭建k8s集群,但是单机版本,只有一个master和一个node. 虽然可以使用大多数的命令,但是对于调度策略以及灰度发布的逻辑研究不是特别方案。所以这里基于centos 7的主机,搭建了一套集群版本的k8s集群。其中包含了1个master和2个node。搭建过程完全是基于ansible来实现的,还算比较方便。这里详细记录下搭建的过程,便于后面回溯。

## 搭建流程
### 检测机器是否支持虚拟化
**检测cpu是否支持虚拟化**

```
egrep -c '(vmx|svm)' /proc/cpuinfo

```
输出结果:

![http://p.qpic.cn/qqconadmin/0/ea35cc7d79144f3093aef6a8b7d671d3/0](
http://p.qpic.cn/qqconadmin/0/ea35cc7d79144f3093aef6a8b7d671d3/0)


### 安装软件
```
yum  -y  install  qemu-kvm  libvirt  virt-install  bridge-utils
```

### 检测模块是否加载
**检测内核模块是否包含kvm**
```
lsmod | grep kvm 
```
输出结果:
![
http://p.qpic.cn/qqconadmin/0/363ffe78d15b4dcf8aef5a233fcf7cbe/0](
http://p.qpic.cn/qqconadmin/0/363ffe78d15b4dcf8aef5a233fcf7cbe/0)

### 启动服务
```
systemctl  start  libvirtd
systemctl  enable  libvirtd

```

### 创建虚拟机
如果有图形化的操作系统的时候,非常建议使用virt-manager来安装操作系统,这样可以少很多命令行操作,同时可以减少出错。
```
virt-manager
```
生成如下页面:
![
http://p.qpic.cn/qqconadmin/0/d69f44872970434e8311ba5db503692f/0](
http://p.qpic.cn/qqconadmin/0/d69f44872970434e8311ba5db503692f/0)

然后就可以通过iso镜像来安装虚拟机了。上面的centos7.0-node1和centos7.01-node2都是通过这种方式来创建的。

当然也可以通过通过配置的方式来创建,比如参考如下指令:
```
virt-install \
--name centos7 \
--ram 2048 \
--disk path=/var/kvm/images/centos7.img,size=30 \
--vcpus 2 \
--os-type linux \
--os-variant rhel7 \
--network bridge=br0 \
--graphics none \
--console pty,target_type=serial \
--location '/tmp/CentOS-7-x86_64-Minimal-1611.iso' \
--extra-args 'console=ttyS0,115200n8 serial' \
--force
```

具体的方式可以自己摸索。

### virbr0网卡
>virbr0是一种虚拟网络接口,安装和启动了libvirt服务之后,会默认生成一个virtual network switch(virbr0), host上的所有虚拟机通过这个virbr0连接起来,为连接其上的虚机网卡提供 NAT 访问外网的功能。virbr0默认的IP是 `192.168.122.1`,并为连接其上的虚拟网卡提供DHCP服务。

#### 查看虚拟机的挂载情况
```
brctl show
```
![
http://p.qpic.cn/qqconadmin/0/d1ff0616f17f495db4c9c1b4c4b226e1/0](
http://p.qpic.cn/qqconadmin/0/d1ff0616f17f495db4c9c1b4c4b226e1/0)

可以看到有两个桥接设备。
* docker0 主要是docker来实现,宿主机和容器通信用的.
* virbr0 主要是实现kvm宿主机和虚拟机通信的。

从上图可以看到,vnet0和vnet1是挂接到virbr0的两个设备。

#### 查看网卡情况
```
virsh list --all
virsh domiflist  centos7.0-node1
```
输出如下图:
![http://p.qpic.cn/qqconadmin/0/49d27b145e2f40e9ae9f678e593a3d2e/0](
http://p.qpic.cn/qqconadmin/0/49d27b145e2f40e9ae9f678e593a3d2e/0)

可以看到vnet0是centos7.0-node1对应的网卡。

#### dhcp服务
virbr0通过使用dnsmasq来提供dhcp服务,可以在宿主机查看进程信息。
```
ps -ef|grep dnsmasq
```
可以看到整个进程:
/usr/sbin/dnsmasq --conf-file=/var/lib/libvirt/dnsmasq/default.conf --leasefile-ro --dhcp-script=/usr/libexec/libvirt_leaseshelper

一般情况下，虚拟机的ip默认情况下是自动分配的,如果用虚拟机来实现k8s集群,最好的方式是固定ip,那么通过绑定mac地址和ip地址,就可以实现固定ip地址的目的:
```
cat /var/lib/libvirt/dnsmasq/default.conf
```
可以看到配置文件中,可以通过配置dhcp-hostsfile选项,来实现固定ip地址。dnsmasq可以参考专门的介绍文章。输出内容如下:
```
##WARNING:  THIS IS AN AUTO-GENERATED FILE. CHANGES TO IT ARE LIKELY TO BE
##OVERWRITTEN AND LOST.  Changes to this configuration should be made using:
##    virsh net-edit default
## or other application using the libvirt API.
##
## dnsmasq conf file created by libvirt
strict-order
pid-file=/var/run/libvirt/network/default.pid
except-interface=lo
bind-dynamic
interface=virbr0
dhcp-range=192.168.122.2,192.168.122.254
dhcp-no-override
dhcp-authoritative
dhcp-lease-max=253
dhcp-hostsfile=/var/lib/libvirt/dnsmasq/default.hostsfile
addn-hosts=/var/lib/libvirt/dnsmasq/default.addnhosts
```

可以通过配置: /var/lib/libvirt/dnsmasq/default.hostsfile来实现固定ip地址的目的。配置完成之后,可以通过重启虚拟机生效:
```
systemctl restart  libvirtd.service
```

## 虚拟机管理
* 查看虚拟机列表
```
virsh list -all
```

* 启动虚拟机
```
virsh start centos7.0-node1
```

* 关闭虚拟机
```
virsh  shutdown  centos7.0-node1
```

* 挂起虚拟机
```
virsh suspend centos7.0-node1
```
* 恢复挂起虚拟机
```
virsh resume  centos7.0-node1
```
* 开机启动虚拟机
```
virsh autostart centos7.0-node1
```
* 关闭自动启动虚拟机
```
virsh auto start --disable centos7.0-node1
```
* 关闭虚拟机
```
virsh shutdown centos7.0-node1
```
* 强制关机
```
virsh destory centos7.0-node1
```
## 参考文档
[centos 7安装kvm并搭建虚拟机](
https://blog.51cto.com/12173069/2294364)

[centos图形化界面安装kvm虚拟机](
https://blog.51cto.com/12173069/2294364)

[virbr0是什么](
https://blog.csdn.net/sangyi1122/article/details/84983845)

[理解virbr0](
https://www.cnblogs.com/zhaohongtian/p/6811317.html)

[centos7里的虚拟网桥virbr0](
https://my.oschina.net/u/4135649/blog/3068968)

[kvm虚拟机常见管理命令](
https://baijiahao.baidu.com/s?id=1612293596898577753&wfr=spider&for=pc)