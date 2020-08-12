---
layout: post
title: "Windows 10+Ubuntu 20.4双系统安装+向日葵远程控制"
description: ""
categories: []
tags: [Essential Skill,Linux]
redirect_from:
  - /2020/08/12/
typora-root-url: ..
---

* Kramdown table of contents
{:toc .toc}
本文主要参考了博客[windows10安装ubuntu双系统教程（绝对史上最详细）](https://www.cnblogs.com/masbay/p/10745170.html)成功安装，

前半部分则主要转载它的，再根据我安装的时候遇到的问题重新整理一下，不是手把手，适合稍微懂点的人看。

首先提醒以下，先装win10再装Ubuntu，很容易成功，但先装Ubuntu，启动引导很容易出问题，当时什么都不懂，就踩坑了。

# 一、双系统的安装

## 1. 判断启动方式是MBR还是UEFI

按下Win+R, 输入`msinfo32`回车确认

在出现的信息中找到`BIOS模式`看看是传统(即MBR)还是UEFI

## 2. MBR启动方式下安装

1. 用他说的磁盘管理或者[DiskGenius](https://www.diskgenius.cn/)来进行磁盘分区，也就是腾出来地方给Ubuntu。我采用的是[老毛桃](https://www.laomaotao.net/)WinPE下的DiskGenius工具进行的磁盘分区。腾出足够的`空闲空间`即可。

   > 因为我有老毛桃启动盘，之前经常用老毛桃进行分区装系统，对它比较熟悉，怕用windows下的磁盘管理工具踩坑。并且老毛桃可以对[C盘进行扩容](https://diskgenius.cn/help/partresizing.php)，登陆密码重置。所以我比较喜欢这个工具。

2. 下载Ubuntu最新版系统, 推荐[清华源](https://mirrors.tuna.tsinghua.edu.cn/ubuntu-releases/)

3. 制作Ubuntu系统安装盘，用[软碟通](https://cn.ultraiso.net/)(可以试用)或者[Rufus](https://rufus.ie/)制作

4. 然后就用这个启动盘安装系统即可，注意！！！在选择安装类型的时候，最好选择最后一个，自己定义分区，就会虽然麻烦一点，但可以避过很多坑。在之前腾出的空闲空间下新建下列分区

   > /boot   ext4格式， 200MB  用来装系统启动相关文件
   >
   > swap    交换空间   电脑运行内存的2倍即可     用来在电脑内存不足的情况下调用执行程序
   >
   > /             ext4格式   越大越好    系统配置，安装的软件什么的都在这
   >
   > /home   ext4格式  越大越好    自己的用户文件保存在这，根据你的使用情况分

   注意！！！ 然后在`安装启动引导器的设备`这一栏选择`/boot`所在的分区

5. 安装完成后重启回到windows系统，打开[EasyBCD](https://www.techspot.com/downloads/3112-easybcd.html)这个软件，这个是双系统共存的关键，`添加新条目`->`Linux`->选择/boot所在的分区

   > 注意：那个博客没提到要启用一下，在`编辑引导菜单`里启用一下Linux那个条目，然后再重启就能选择是进入Ubuntu还是Windows了

完成啦！！！

## 3. UEFI启动方式安装

> 这个我没一步一步跟着做，仅仅记录一下

跟MBR的不同

4. > efi         efi格式， 200MB  用来装系统启动相关文件， ！！！！**不同就在这**
   >
   > swap    交换空间   电脑运行内存的2倍即可     用来在电脑内存不足的情况下调用执行程序
   >
   > /             ext4格式   越大越好    系统配置，安装的软件什么的都在这
   >
   > /home   ext4格式  越大越好    自己的用户文件保存在这，根据你的使用情况分

   然后在`安装启动引导器的设备`这一栏选择`efi`所在的分区 (大概200MB,Window的启动文件也是efi, 大概200MB, 注意区分)

5. 这样的话就好啦，不用自己设置启动条目，重启即可

# 二、在Ubuntu下安装向日葵远程控制软件

1. 正常情况下在[向日葵](https://sunlogin.oray.com/download/)官网下载deb安装包即可。

   > 建议装图形版本，命令行版本只能被控，不能控制别人。并且用着比较麻烦

2. cd到deb包所在的文件夹，执行

   ```shell
   sudo dpkg -i sunloginclient.deb  #注意更改deb包的名字，善用Tab键自动补全
   ```

3. 上面执行完可能就行了，但也可能报错，缺少依赖，比如

   ![](/images/posts/2020-08-12/error.png)

   这时候可以尝试使用apt来让他自动解决依赖

   ```shell
   sudo apt-get install -f -y
   ```

4. 上面的命令执行完可能也行了，但还可能出现没办法自动解决依赖的问题

   ![](/images/posts/2020-08-12/error2.png)

   他就把sunloginclient（也就是向日葵)给卸载了。（典型的解决不了问题就解决提出问题的人。。。）

5. 这样的话就需要手动解决依赖问题啦

   直接去下载所缺依赖的包，即[libwebkitgtk-3.0-0](https://debian.pkgs.org/9/debian-main-amd64/libwebkitgtk-3.0-0_2.4.11-3_amd64.deb.html), 然后自行安装即可，可能安装他还提示缺少依赖，那就继续下载依赖的包，然后都安装一下就好，大概需要安装10个以内的包就行了。然后最终把libwebkitgtk-3.0-0安装成功即可。

   再执行`sudo dpkg -i sunloginclient.deb`就不会出现依赖问题啦。成功安装

   ![](/images/posts/2020-08-12/ok.png)

   然后就可以在左下角的全部应用程序里找到向日葵并且能成功启动了呢。

   

   > 注意：卸载包的话
   >
   > ```shell
   > sudo dpkg -l | grep sunloginclient #看看安装了没有
   > sudo dpkg -r sunloginclient        #卸载掉它
   > ```
   >
   > 如果真的用命令行版的向日葵，可以参考[向日葵LinuxTerminal帮助文档](https://service.oray.com/question/11017.html)

   









