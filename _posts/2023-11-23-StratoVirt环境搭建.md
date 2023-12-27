---
1title: StratoVirt环境搭建

date: 2023-11-23 17:00:00 +0800

categories: [StratoVirt]

tags: [riscv, qemu]

description: 
---

# 1 StratoVirt介绍



# 2 StratoVirt使用

该系列实验的宿主机环境为：

![image-20231130132249488](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311301322581.png)

## 2.1 QEMU中运行openEuler RISC-V

### 下载qemu源代码并构建

下载qemu并解压，速度慢可选择：https://mirrors.aliyun.com/blfs/conglomeration/qemu/

```shell
wget https://download.qemu.org/qemu-7.2.0.tar.xz
tar xvJf qemu-7.2.0.tar.xz
```

安装编译所需的依赖包：

```shell
sudo apt install autoconf automake autotools-dev curl libmpc-dev libmpfr-dev libgmp-dev \
              gawk build-essential bison flex texinfo gperf libtool patchutils bc \
              zlib1g-dev libexpat-dev pkg-config  libglib2.0-dev libpixman-1-dev libsdl2-dev \
              git tmux python3 python3-pip ninja-build
```

构建并安装qemu：

```shell
cd qemu-7.2.0
./configure --target-list=riscv64-softmmu,riscv64-linux-user
make -j$(nproc)
sudo make install
qemu-system-riscv64 --version
```

### 准备openEuler RISC-V磁盘映像

首先需要下载：

* 启动固件 (`fw_payload_oe_uboot_2304.bin`)
* 磁盘映像(`openEuler-23.09-RISC-V-qemu-riscv64.qcow2.xz`)
* 启动脚本(`start_vm.sh`)

官方仓库：https://repo.openeuler.org/openEuler-23.09/virtual_machine_img/riscv64/

### 启动并登录openEuler RISC-V虚拟机

启动虚拟机步骤如下：

1. 确认当前目录内包含 `fw_payload_oe_uboot_2304.bin`，磁盘映像压缩包，以及启动脚本；
2. 解压映像压缩包 `xz -dk openEuler-23.09-RISC-V-qemu-riscv64.qcow2.xz` 调整启动参数；
3. 执行启动脚本 `bash start_vm.sh`

可根据需求调整参数：

- `vcpu` 为 QEMU 运行线程数，与 CPU 核数没有严格对应。当设定的 `vcpu` 值大于宿主机核心值时，可能导致运行阻塞和速度严重降低。默认为 `4`。
- `memory` 为虚拟机内存大小，可随需要调整。默认为 `2`。
- `drive` 为虚拟磁盘路径，如果在上文中配置了 COW 映像，此处填写创建的新映像。
- `fw` 为 U-Boot 镜像路径。
- `ssh_port` 为转发的 SSH 端口，默认为 `12055`。设定为空以关闭该功能。

也可利用 `ssh -p 12055 root@localhost` 登录，用户名和密码为：

- 用户名: `root` 或 `openeuler`
- 默认密码: `openEuler12#$`

![image-20231130142323220](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311301423296.png)

## 2.2 QEMU环境运行TeleVM RISC-V虚拟机

TeleVM 是由电信云团队开发的用Rust语言编写的基于StratoVirt的轻量级RISC-V用户态虚拟机，需要接入KVM，从而构成完整的虚拟化方案。

### QEMU中运行openEuler RISC-V虚拟机 (2.1)

同2.1，镜像换成了 `openEuler-23.03-V1-base-qemu-preview.qcow2`，该版本内核已加载 `kvm`。

https://mirror.iscas.ac.cn/openeuler-sig-riscv/openEuler-RISC-V/testing/20230331_openEuler-23.03-V1-riscv64/QEMU/

### 虚拟机中运行TeleVM

#### 基础环境搭建

你需要完成两件事：

* Rust开发环境配置
* 构建RISC-V的GCC交叉工具链

> ***Rust开发环境配置***

首先安装 Rust 版本管理器 `rustup` 和 Rust 包管理器 `cargo`。官方的脚本在运行时可能会出现网络速度较慢的问题，可以通过修改 `rustup` 的镜像地址（修改为中国科学技术大学的镜像服务器）来加速：

```shell
export RUSTUP_DIST_SERVER=https://mirrors.ustc.edu.cn/rust-static
export RUSTUP_UPDATE_ROOT=https://mirrors.ustc.edu.cn/rust-static/rustup
curl https://sh.rustup.rs -sSf | sh
```

安装完成后，我们可以重新打开一个终端来让之前设置的环境变量生效。我们也可以手动将环境变量设置应用到当前终端，只需要输入以下命令：

```shell
source $HOME/.cargo/env
```

接下来，我们可以确认一下我们正确安装了 Rust 工具链：

```shell
rustc --version
```

我们最好把软件包管理器 cargo 所用的软件包镜像地址 crates.io 也换成中国科学技术大学的镜像服务器来加速三方库的下载。我们打开（如果没有就新建） `~/.cargo/config` 文件，并把内容修改为：

```shell
[source.crates-io]
registry = "https://github.com/rust-lang/crates.io-index"
replace-with = 'ustc'
[source.ustc]
registry = "git://mirrors.ustc.edu.cn/crates.io-index"
```

接下来安装一些Rust相关的软件包：

```shell
rustup target add riscv64gc-unknown-none-elf
cargo install cargo-binutils
rustup component add llvm-tools-preview
rustup component add rust-src
```

> ***构建RISC-V的GCC交叉工具链***

需要安装的依赖：

```shell
sudo apt install autoconf automake autotools-dev curl libmpc-dev libmpfr-dev libgmp-dev \
                 gawk build-essential bison flex texinfo gperf libtool patchutils bc \
                 zlib1g-dev libexpat-dev git \
                 libglib2.0-dev libfdt-dev libpixman-1-dev \
                 libncurses5-dev libncursesw5-dev
```

下载工具链并进入源码目录：

```shell
git clone https://gitee.com/mirrors/riscv-gnu-toolchain
cd riscv-gnu-toolchain
```

注意上面 clone 的主仓库并不包含子仓库的内容，所以需要继续更新子仓库。注意这里首先排除了 `qemu` 这个子仓库，一来因为 `qemu` 完整下载太大；二来 `qemu` 对 toolchain 的编译本身来说其实并不需要：

```shell
git rm qemu
git submodule update --init --recursive
```

有点慢，耐心等待子仓库下载完成。

子仓库下载完成后安装，注意配置时指定安装到 `/opt/riscv64`，所以 make 时需要 sudo：

```shell
./configure --prefix=/opt/riscv64 
sudo make linux -j $(nproc)
```

编译也比较慢。。完成设置环境变量：

```shell
export PATH="$PATH:/opt/riscv64/bin"
```

测试下 toolchain 是否安装成功：

```shell
riscv64-unknown-linux-gnu-gcc -v
```

#### 在宿主机环境中编译TeleVM

下载源码并构建：

```shell
git clone https://github.com/ltz0302/TeleVM
cd televm
cargo build
```

`cargo build` 后可能会报出很多错误，这里给出一些解决错误的关键点：

* 先把 `Cargo lock` 删了
* 适当调整各模块目录下的 `Cargo.toml` 的选项，根据提示修改即可

构建成功后，在 `target/riscv64gc-unknown-linux-gnu/debug` 下生成TeleVM二进制文件。 

![image-20231201161023380](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202312011610453.png)

#### 制作riscv-linux-kernel/rootfs镜像

> ***编译Linux内核***

需要安装的依赖：

```shell
sudo apt install gcc g++ gcc-riscv64-linux-gnu wget flex bison bc cpio make pkg-config libncurses-dev
```

下载内核源码，太大了指定最近一次commit吧：

```shell
git clone --depth=1 https://github.com/kvm-riscv/linux.git
```

接着，创建编译目录并配置处理器架构和交叉编译器等环境变量：

```shell
export ARCH=riscv
export CROSS_COMPILE=riscv64-linux-gnu-
mkdir build-riscv64
```

接着，通过 menuconfig 配置内核选项。在配置之前，需要注意最新版 Linux 内核默认关闭 RISC-V SBI 相关选项，需要通过以下命令手动配置开启，相关讨论参见该 issue，具体细节参见 [此文](https://zhuanlan.zhihu.com/p/539390400)。

```shell
make -C linux O=`pwd`/build-riscv64 menuconfig
```

最后一个环节就是编译了：

```shell
make -C linux O=`pwd`/build-riscv64  -j`nproc`
```

编译完，咱们获得了两个重要的二进制文件：

- 内核映像：`build-riscv64/arch/riscv/boot/Image`
- KVM 内核模块：`build-riscv64/arch/riscv/kvm/kvm.ko`

> ***RootFS文件系统***

RootFS 包括 `KVM kernel module`,  `kernel image` 两部分：

```shell
export ARCH=riscv
export CROSS_COMPILE=riscv64-linux-gnu-
git clone https://github.com/kvm-riscv/howto.git
wget https://busybox.net/downloads/busybox-1.33.1.tar.bz2
tar -C . -xvf ./busybox-1.33.1.tar.bz2
mv ./busybox-1.33.1 ./busybox-1.33.1-kvm-riscv64
cp -f ./howto/configs/busybox-1.33.1_defconfig busybox-1.33.1-kvm-riscv64/.config
make -C busybox-1.33.1-kvm-riscv64 oldconfig
make -C busybox-1.33.1-kvm-riscv64 install
mkdir -p busybox-1.33.1-kvm-riscv64/_install/etc/init.d
mkdir -p busybox-1.33.1-kvm-riscv64/_install/dev
mkdir -p busybox-1.33.1-kvm-riscv64/_install/proc
mkdir -p busybox-1.33.1-kvm-riscv64/_install/sys
mkdir -p busybox-1.33.1-kvm-riscv64/_install/apps
ln -sf /sbin/init busybox-1.33.1-kvm-riscv64/_install/init
cp -f ./howto/configs/busybox/fstab busybox-1.33.1-kvm-riscv64/_install/etc/fstab
cp -f ./howto/configs/busybox/rcS busybox-1.33.1-kvm-riscv64/_install/etc/init.d/rcS
cp -f ./howto/configs/busybox/motd busybox-1.33.1-kvm-riscv64/_install/etc/motd
cp -f ./build-riscv64/arch/riscv/boot/Image busybox-1.33.1-kvm-riscv64/_install/apps
cp -f ./build-riscv64/arch/riscv/kvm/kvm.ko busybox-1.33.1-kvm-riscv64/_install/apps
cd busybox-1.33.1-kvm-riscv64/_install; find ./ | cpio -o -H newc > ../../rootfs_kvm_riscv64.img; cd -
```

#### //TODO: 运行TeleVM

将内核、根文件系统镜像和TeleVM二进制文件打包发送至虚拟机中；

首先查看环境支持：

```shell
# 看到 `h` 字符说明支持H扩展
cat /proc/cpuinfo
# 输出 `kvm` 说明内核已加载kvm
ls /dev/ | grep kvms
```

启动TeleVM：

```shell
./target/riscv64gc-unknown-linux-gnu/debug/televm \
./TeleVM	
	-machine microvm \
	-smp 1 \
	-m 2g \
	-kernel /tmp/zq/rCore-learn/other/televm/vmlinux.bin \
	-drive id=rootfs,file=/tmp/zq/rCore-learn/other/televm/initrd.img \
	-device virtio-blk-device,drive=rootfs,id=blk1 \
	-serial stdio \
	-append "root=/dev/vda rw console=console=ttyS0" \
```

```shell
./TeleVM -machine microvm -smp 1 -m 1g -kernel Image -initrd initrd.img -qmp unix:sock,server,nowait -device virtio-blk-device,drive=rootfs,id=blk1 -serial stdio -append "root=/dev/vda rw console=console=ttyS0"
```

-qmp unix:sock,server,nowait

-qmp unix:/home/ubuntu/stratovirt.sock,server,nowait

let kernel_path = String::from("vmlinux.bin");
let initrd_path = String::from("initrd.img");

### 问题汇总

[unknown feature `proc_macro_span_shrink`//could not compile xxx lib问题解决，此解决方案不管是在哪一个系统都可以解决此问题。_Rock姜的博客-CSDN博客](https://blog.csdn.net/jl19861101/article/details/132868128)

## 2.3 在x86主机中直接运行StratoVirt X86虚拟机

### 环境准备

在编译StratoVirt前，请确保Rust语言环境和Cargo软件已经安装成功。如果没有安装，请参考以下链接的指导进行安装：

https://www.rust-lang.org/tools/install

### 编译软件

为了编译StratoVirt，需要先克隆代码工程，然后执行编译命令，如下：

```sh
$ git clone https://gitee.com/openeuler/stratovirt.git
$ cd stratovirt
$ make build
```

可以在`target/release/stratovirt`路径下找到生成的二进制文件

### 使用StratoVirt启动虚拟机

为了快速上手StratoVirt，需要准备

* PE格式或bzImage格式(仅x86_64)的Linux内核镜像
* ext4文件系统，raw格式rootfs的镜像

可以通过以下链接获取我们准备好的linux内核镜像和rootfs镜像：

https://repo.openeuler.org/openEuler-22.03-LTS/stratovirt_img/

启动标准机型的虚拟机需要指定遵循UEFI的edk2固件文件。

```shell
# 如果-qmp的socket文件已经存在，请先删除它

# 启动microvm机型
$ ./target/release/stratovirt \
    -machine microvm \
    -kernel /path/to/kernel \
    -append "console=ttyS0 root=/dev/vda reboot=k panic=1" \
    -drive file=/path/lsto/rootfs,id=rootfs,readonly=off \
    -device virtio-blk-device,drive=rootfs,id=rootfs \
    -qmp unix:/path/to/socket,server,nowait \
    -serial stdio

# x86_64上启动标准机型
$ ./target/release/stratovirt \
    -machine q35 \
    -kernel /path/to/kernel \
    -append "console=ttyS0 root=/dev/vda reboot=k panic=1" \
    -drive file=/path/to/firmware,if=pflash,unit=0,readonly=true \
    -device pcie-root-port,port=0x0,addr=0x1.0x0,bus=pcie.0,id=pcie.1 \
    -drive file=/path/to/rootfs,id=rootfs,readonly=off \
    -device virtio-blk-pci,drive=rootfs,bus=pcie.1,addr=0x0.0x0,id=blk-0 \
    -qmp unix:/path/to/socket,server,nowait \
    -serial stdio
```

关于制作rootfs镜像、编译内核镜像以及编译StratoVirt的详细指导，请参考[StratoVirt Quickstart](./docs/quickstart.md)。

StratoVirt所支持更多特性，详细指导请参考[Configuration Guidebook](docs/config_guidebook.md)。

如果你想获取更多关于StratoVirt的信息，请参考https://gitee.com/openeuler/stratovirt/wikis

## 2.4 其它方案对比 (RISC-V)

### QEMU/KVM方案

### TeleVM/KVM方案

