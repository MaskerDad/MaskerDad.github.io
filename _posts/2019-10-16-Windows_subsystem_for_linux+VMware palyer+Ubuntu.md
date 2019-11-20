---
layout: post
title: "Windows Subsystem for Linux+VMware palyer+Ubuntu"
description: ""
categories: []
tags: [Essential Skill,Linux]
redirect_from:
  - /2019/10/17/
typora-root-url: ..
---

自从Win10更新之后，我之前装的VMware workstation就不能用了，必须更新。所以今晚不想学习就搞了一下，并且发现了Windows Subsystem for Linux这个好玩意。

## 1.下载安装VMware和Ubuntu

这里推荐用[VMware Workstation 15 Player](https://www.vmware.com/cn/products/workstation-player/workstation-player-evaluation.html)，VMware Workstation  Player是免费版的虚拟机，已经支持很多操作了，付费订阅之后会有更完整的功能。VMware Workstation  Pro是专业版的是要付费的。VMware Workstation  Player这个免费版的已经满足大部分人的需求了，秉着支持正版的态度，我没有去找VMware Workstation  Pro的破解版。需要的自己百度或者淘宝。

1. 在官网下载最新版安装包

[https://www.vmware.com/cn/products/workstation-player/workstation-player-evaluation.html](https://www.vmware.com/cn/products/workstation-player/workstation-player-evaluation.html)

2. 下载Ubuntu安装镜像，这推荐在各个开源镜像站，比如清华源，中科大源，西电也有开源镜像站，自己学校的下载速度会快很多。(西电镜像需要用校园网下载)
3. 得到iso镜像。安装iso到VMware就很简单的傻瓜式操作了，网上也有大把教程，这里不再赘述。

## 2. Windows Subsystem for Linux

WSL是Win10才加入了，就是在Windows下安装Linux子系统。我试了一下，也很好用，全命令行的，相当于用了个SSH客户端，但IP地址什么的跟主机的一样，这样的话就很方便的自己搞网站了，比如把jekyll的本地环境部署到这个Linux下。

安装方法微软官网有[Windows Subsystem for Linux Installation Guide for Windows 10](https://docs.microsoft.com/en-us/windows/wsl/install-win10)

1. 以管理员权限打开Powershell

2. 执行下面的代码开启子系统功能

   ```powershell
   Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Windows-Subsystem-Linux
   ```

3. 重启电脑

4. 在Microsoft Store搜索Linux，安装其中某一个即可

   ![](/images/posts/2019-10-17/store.png)

   可以看到，还是可以安装很多个版本的Linux的

5. 像正常启动软件一样启动Ubuntu这个“软件”就行, 打开就是下面这样的命令行界面。有需要的自己捣鼓一下安装个桌面。

   ![](/images/posts/2019-10-17/ubuntu.png)

**注意**

- VMware Workstation Player和Windows Subsystem for Linux安装过程中均需要多次重启
- Windows Subsystem for Linux的Ubuntu中主机的文件是挂载到/mnt下了分别以c、d...命名，就是代表Windows下的各个盘

## 3. VMware中虚拟机网络的配置

此部分主要参考 [Vmware虚拟机三种网络模式详解](https://www.cnblogs.com/linjiaxin/p/6476480.html)，里面讲的很详细。我主要说的理解

- **Bridged（桥接模式）**也就是虚拟机跟主机在一个网段，相当于在主机所在局域网里增加了一个机器。不过桥接模式不能用DHCP自动获取IP地址，需要自己配置IP, 网关DNS。
- **NAT（网络地址转换模式）**也就是新建了一个虚拟局域网，把主机和虚拟机放在一个虚拟局域网里。并且通过虚拟NAT设备能让虚拟机直接通过主机的网卡访问外网。这个就不需要自己配置任何东西了。(主机有一个虚拟网卡VMnet8，这个网卡的IP地址就是主机在虚拟局域网中的IP)
- **Host-Only（仅主机模式）**只新建一个虚拟局域网，并没有虚拟NAT能直接访问外网，需要自己另外设置。(在虚拟局域网中的IP用虚拟网卡VMnet1控制)

在主机cmd界面执行

```shell
ipconfig /all
```

得到下面的信息

> 以太网适配器 VMware Network Adapter VMnet1:
>
>    连接特定的 DNS 后缀 . . . . . . . :
>    描述. . . . . . . . . . . . . . . : VMware Virtual Ethernet Adapter for VMnet1
>    物理地址. . . . . . . . . . . . . : 00-50-56-C0-00-01
>    DHCP 已启用 . . . . . . . . . . . : 否
>    自动配置已启用. . . . . . . . . . : 是
>    本地链接 IPv6 地址. . . . . . . . : fe80::5510:f6e3:665d:c391%2(首选)
>    IPv4 地址 . . . . . . . . . . . . : 192.168.238.1(首选)
>    子网掩码  . . . . . . . . . . . . : 255.255.255.0
>    默认网关. . . . . . . . . . . . . :
>    DHCPv6 IAID . . . . . . . . . . . : 838881366
>    DHCPv6 客户端 DUID  . . . . . . . : 00-01-00-01-1F-C9-20-DB-FC-45-96-45-9D-35
>    DNS 服务器  . . . . . . . . . . . : fec0:0:0:ffff::1%1
>                                        fec0:0:0:ffff::2%1
>                                        fec0:0:0:ffff::3%1
>    TCPIP 上的 NetBIOS  . . . . . . . : 已启用
>
> 以太网适配器 VMware Network Adapter VMnet8:
>
>    连接特定的 DNS 后缀 . . . . . . . :
>    描述. . . . . . . . . . . . . . . : VMware Virtual Ethernet Adapter for VMnet8
>    物理地址. . . . . . . . . . . . . : 00-50-56-C0-00-08
>    DHCP 已启用 . . . . . . . . . . . : 否
>    自动配置已启用. . . . . . . . . . : 是
>    本地链接 IPv6 地址. . . . . . . . : fe80::a4de:c891:14b4:560e%20(首选)
>    IPv4 地址 . . . . . . . . . . . . : 192.168.111.1(首选)
>    子网掩码  . . . . . . . . . . . . : 255.255.255.0
>    默认网关. . . . . . . . . . . . . :
>    DHCPv6 IAID . . . . . . . . . . . : 855658582
>    DHCPv6 客户端 DUID  . . . . . . . : 00-01-00-01-1F-C9-20-DB-FC-45-96-45-9D-35
>    DNS 服务器  . . . . . . . . . . . : fec0:0:0:ffff::1%1
>                                        fec0:0:0:ffff::2%1
>                                        fec0:0:0:ffff::3%1
>    TCPIP 上的 NetBIOS  . . . . . . . : 已启用
>
> 无线局域网适配器 WLAN:
>
>    连接特定的 DNS 后缀 . . . . . . . :
>    描述. . . . . . . . . . . . . . . : Qualcomm Atheros QCA9377 Wireless Network Adapter
>    物理地址. . . . . . . . . . . . . : 58-00-E3-46-5F-B5
>    DHCP 已启用 . . . . . . . . . . . : 是
>    自动配置已启用. . . . . . . . . . : 是
>    本地链接 IPv6 地址. . . . . . . . : fe80::85ed:faf5:d6c1:a2a0%8(首选)
>    IPv4 地址 . . . . . . . . . . . . : 10.177.177.95(首选)
>    子网掩码  . . . . . . . . . . . . : 255.255.0.0
>    获得租约的时间  . . . . . . . . . : 2019年10月18日 9:40:50
>    租约过期的时间  . . . . . . . . . : 2019年10月21日 12:57:32
>    默认网关. . . . . . . . . . . . . : 10.177.255.254
>    DHCP 服务器 . . . . . . . . . . . : 10.177.255.254
>    DHCPv6 IAID . . . . . . . . . . . : 89653475
>    DHCPv6 客户端 DUID  . . . . . . . : 00-01-00-01-1F-C9-20-DB-FC-45-96-45-9D-35
>    DNS 服务器  . . . . . . . . . . . : 202.117.112.3
>                                        218.30.19.40
>    TCPIP 上的 NetBIOS  . . . . . . . : 已启用

可以看到我的主机真实网卡的IP是`10.177.177.95`，NAT模式的虚拟网卡(VMnet8)的IP地址`192.168.111.1`。

我觉得NAT模式是最好用的，根据自己的需求选择即可

在虚拟机中执行

```shell
ifconfig
```

得到虚拟机的IP地址`192.168.111.130`

这样的话，`192.168.111.1`和`192.168.111.130`在同一网段，可以实现虚拟机与主机的通信。虚拟NAT设备又能让虚拟机访问外网(暂时还不知道NAT是什么原理)。

## 4. 主机访问虚拟机环境下的jekyll博客

之前做过jekyll的环境搭建，我们知道`jekyll serve`会让博客网站运行在localhost的4000端口。但是这只能在虚拟机里通过`http://127.0.0.1:4000/`访问。我也想在主机里访问虚拟机的博客网站。但是我在主机的浏览器访问`192.168.111.130:4000`时，提示我`192.168.111.130`拒绝了我的连接请求，查了一下发现jekyll只把博客绑定到了127.0.0.1，其他IP访问都会被拒绝的。所以只需要改成全IP可访问就好。

```shell
jekyll serve -w --host=0.0.0.0
```

这里`-w`是持续监控文件的修改情况、编译成HTML

`--host=0.0.0.0`是让所有的IP地址均可访问。

这样就可以在主机访问虚拟机环境下的jekyll博客了。



