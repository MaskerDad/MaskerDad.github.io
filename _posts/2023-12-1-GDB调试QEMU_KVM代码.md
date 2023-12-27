---
title: GDB调试QEMU_KVM代码

date: 2023-12-1 17:00:00 +0800

categories: [GDB]

tags: [gdb, qemu, kvm]

description: 
---

# 0 参考

[QEMU+GDB调试linux内核驱动 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/615578858)

[构建riscv两层qemu的步骤 | Sherlock's blog (wangzhou.github.io)](https://wangzhou.github.io/构建riscv两层qemu的步骤/)

[用 QEMU/Spike+KVM 运行 RISC-V Host/Guest Linux - 泰晓科技 (tinylab.org)](https://tinylab.org/riscv-kvm-qemu-spike-linux-usage/)

[riscv KVM虚拟化分析 | Sherlock's blog (wangzhou.github.io)](https://wangzhou.github.io/riscv-KVM虚拟化分析/)

[无标题文档 (yuque.com)](https://www.yuque.com/weixin-37430058/klw1dv/mlktwrvemh1t7kvx)



# 1 QEMU/GDB调试内核驱动



arch



# 2 riscv两层qemu环境配置

可以构建一个两层qemu的环境来调试问题，第一层qemu启动的时候打开qemu的虚拟化扩展，这个可以作为一个支持虚拟化扩展的riscv硬件平台，第二层qemu启动的时候 打开kvm支持。

> 注意，第二层qemu的编译比较有意思，因为qemu编译需要依赖很多动态库，我用的都是交叉编译编译riscv的程序，所以，需要先交叉编译qemu依赖的动态库，然后再交叉编译qemu，太麻烦了。我们这里用编译buildroot的方式一同编译小文件系统里的qemu, buildroot编译qemu的时候就会一同编译qemu依赖的各种库, 这样编译出的host文件系统里就带了qemu。

## 2.1 在主机环境中完成的工作

具体流程如下：

* qemu的编译安装
* 编译第一层qemu运行的内核，需要开启linux内核的kvm模块
* 编译根文件系统（如果想在第一层qemu启动后的环境中直接运行qemu，需要编译buildroot时一同编译qemu）

### qemu编译安装

创建工作目录：

```shell
mkdir riscv64-kvm
cd riscv64-kvm
```

下载qemu源码：

[blfs-conglomeration-qemu安装包下载_开源镜像站-阿里云 (aliyun.com)](https://mirrors.aliyun.com/blfs/conglomeration/qemu/)

```shell
wget https://download.qemu.org/qemu-8.0.0.tar.xz
tar xvJf qemu-8.0.0.tar.xz
```

编译安装：

```shell
cd qemu-8.0.0/
mkdir build
cd build
../configure --enable-kvm --enable-virtfs --target-list=riscv64-linux-user,riscv64-softmmu \
	--prefix=/opt/software/toolchain/qemu
make -j`nproc`
sudo make install
```

>其中的 ：
>
>* `--target-list` 为将要生成的目标平台：
>
> * `riscv-64-linux-user` 为用户模式，可以运行基于riscv指令集编译的程序文件
>
> * `softmmu` 为系统模式，可以运行基于riscv指令集编译的linux镜像；
>
>* `--enable-kvm` 为把kvm编译进qemu里；
>
>* `--enable-virtfs` 为qemu使用共享文件夹的参数
>
>  * 使用此选项需要安装一些依赖：
>
>    ```shell
>    sudo apt install libcap-ng-dev libcap-dev libcap-ng-utils libattr1 libattr1-dev
>    ```

验证安装是否成功：

```shell
qemu-system-riscv64 --version
```

若输出qemu版本信息则安装成功。

### 内核编译

下载内核源码：

[linux-kernel安装包下载_开源镜像站-阿里云 (aliyun.com)](https://mirrors.aliyun.com/linux-kernel/?spm=a2c6h.13651104.d-4003.5.72a570149KivWf)

这里下载的是5.19内核。

```shell
git clone https://github.com/kvm-riscv/linux.git 
# wget https://mirrors.aliyun.com/linux-kernel/v5.x/linux-5.19.tar.xz
# tar xvJf linux-5.19.tar.xz
```

接着，创建编译目录并配置处理器架构和交叉编译器等环境变量：

```shell
export ARCH=riscv
export CROSS_COMPILE=riscv64-linux-gnu-
mkdir build-riscv64
```

接着，通过 menuconfig 配置内核选项。在配置之前，需要注意最新版 Linux 内核默认关闭 RISC-V SBI 相关选项，需要通过以下命令手动配置开启，相关讨论参见该 issue，具体细节参见 [Linux kernel 配置修改 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/539390400)。

```shell
# change options of kernel compiling to generate build-riscv64/.config (output dir)
make -C linux-5.19 O=`pwd`/build-riscv64 menuconfig 
```

最后一个环节就是编译了：

```shell
make -C linux-5.19 O=`pwd`/build-riscv64  -j`nproc`
```

编译完，咱们获得了两个重要的二进制文件：

- 内核映像：`build-riscv64/arch/riscv/boot/Image`
- KVM 内核模块：`build-riscv64/arch/riscv/kvm/kvm.ko`

### buildroot编译

这一层的qemu我们可以用一种更便捷的方法，用编译buildroot的方式一同编译根文件系统里的qemu, buildroot编译qemu的时候就会一同编译qemu依赖的各种库, 这样编译出的根文件系统里就带了qemu。

首先下载buildroot工具：

```shell
git clone https://github.com/buildroot/buildroot.git
cd buildroot
make menuconfig
```

---

选择RISC-V架构：

`Target options  --->  Target Architecture （i386）--->  (x) RISCV`

![2023-09-01_131109.png](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202312041334764.png)

---

选择ext文件系统：

`Filesystem images ---> [*] ext2/3/4 root filesystem`

![2023-09-01_131351.png](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202312041335497.png)

下方的 `exact size` 可以调整ext文件系统大小配置，默认为60M，这里需要调整到500M以上，因为需要编译qemu文件进去

---

buildroot配置qemu

```shell
BR2_TOOLCHAIN_BUILDROOT_GLIBC=y # Toolchain -> C library (<choice> [=y]) glibc
BR2_USE_WCHAR=y # BR2_TOOLCHAIN_USES_GLIBC=y
BR2_PACKAGE_QEMU=y # QEMU
BR2_TARGET_ROOTFS_CPIO=y # cpio the root filesystem (for use as an initial RAM filesystem) 
BR2_TARGET_ROOTFS_CPIO_GZIP=y
Prompt: gzip                                                                                                                                                                            
  │   Location:                                                                                                                                                                                           
  │     -> Filesystem images                                                                                                                                                                              
  │ (1)   -> cpio the root filesystem (for use as an initial RAM filesystem) (BR2_TARGET_ROOTFS_CPIO [=n])                                                                                              
  │         -> Compression method (<choice> [=n]) 
```

在可视化页面按/ 即可进入搜索模式，在搜索模式分别输入上述参数：

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202312041336099.png" alt="截图 2023-09-22 14-55-44.png" style="zoom: 33%;" />

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202312041337819.png" alt="截图 2023-09-22 14-55-54.png" style="zoom:33%;" />

---

全部开启后保存退出

`make -j` 编译，完成后在output/images目录下得到rootfs.ext2，将它复制到工作目录。**这里编译的qemu版本为8.1.0**。

第二层qemu运行的内核，就使用第一次编译的内核即可。

## 2.2 从主机运行脚本

### 第一层qemu的脚本

```shell
#!/bin/bash

sudo qemu-system-riscv64 \
-M virt \
-cpu 'rv64,h=true' \
-m 2G \
-kernel Image \
-append "rootwait root=/dev/vda ro" \
-drive file=rootfs.ext2,format=raw,id=hd0 \
-device virtio-blk-device,drive=hd0 \
-nographic \
-virtfs local,path=/home/wx/Documents/shared,mount_tag=host0,security_model=passthrough,id=host0 \
-netdev user,id=net0 -device virtio-net-device,netdev=net0
```

> **qemu7.0.0之前的版本使用-cpu rv64,x-h=true能使CPU虚拟化扩展，在qemu v7.0.0以及之后的版本使用-cpu rv64,h=true能使CPU虚拟化扩展**

执行上述命令启动QEMU后，root账号登录Linux系统，然后执行 `mount` 命令挂载宿主机目录，用于文件共享：

```shell
mkdir -p /mnt/shared
mount -t 9p -o trans=virtio,version=9p2000.L host0 /mnt/shared
```

### //error: 第二层qemu的脚本

```shell
#!/bin/sh

/usr/bin/qemu-system-riscv64 \
-M virt --enable-kvm \
-cpu rv64 \
-m 256m  \
-kernel ./Image \
-append "rootwait root=/dev/vda ro" \
-drive file=rootfs.ext2,format=raw,id=hd0 \
-device virtio-blk-device,drive=hd0 \
-nographic 
```

> error => qemu-system-riscv64: Unable to read ISA_EXT KVM register ssaia, error -1

* `-cpu` 未支持 AIA
* `-machine` 位置添加 AIA













