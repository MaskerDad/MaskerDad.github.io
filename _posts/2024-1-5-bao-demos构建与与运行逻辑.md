---
title: bao-demos构建与与运行逻辑

date: 2024-1-5 17:00:00 +0800

categories: [bao-project学习]

tags: [hypervisor]

description: 
---

# 0 参考

* [bao_hyperviosr构建](https://zq.org.edu.kg/2023/11/27/Bao_hypervisor%E5%AD%A6%E4%B9%A0/)

* [bao-project](https://github.com/bao-project)

---

之前了解更多的可能是KVM/Xen这种数据中心使用的虚拟化方案，而 `bao-project` 是一个嵌入式虚拟化的开源项目，其核心实现是 `bao-hypervisor`，代码量其实并不算太大，RISC-V相关的代码差不多1w多行。但构建流程还是有一定复杂性的，有必要学习一下。

[bao-project](https://github.com/bao-project) 包含了一系列子项目，核心就是 [bao-hypervisor](https://github.com/bao-project/bao-hypervisor)，其余的包含：

* [bao-demos](https://github.com/bao-project/bao-demos)：包含baremetal、freeRTOS、linux的组合运行，支持系统间的通信；
* [bao-helloworld](https://github.com/bao-project/bao-helloworld)：非常详细的bao-guests构建文档；
* [bao-baremetal-guest](https://github.com/bao-project/bao-baremetal-guest)：用于测试bao-hypervisor的裸金属简易guest实现；

bao 是一个轻量级的基于静态分区架构的 Type-I hypervisor，[论文](https://drops.dagstuhl.de/storage/01oasics/oasics-vol077-ng-res2020/OASIcs.NG-RES.2020.3/OASIcs.NG-RES.2020.3.pdf)以及[翻译](https://blog.csdn.net/jingyu_1/article/details/134577212)等有时间再看吧！

# 1 bao-demos测试

> 1）安装依赖

```shell
sudo apt install build-essential bison flex git libssl-dev ninja-build \
    u-boot-tools pandoc libslirp-dev pkg-config libglib2.0-dev libpixman-1-dev \
    gettext-base curl xterm cmake python3-pip xilinx-bootgen

pip3 install pykwalify packaging pyelftools
```

> 2）下载并设置编译工具链

RISC-V，使用 **riscv64-unknown-elf-** 工具链：[SiFive's Freedom Tools](https://github.com/sifive/freedom-tools/releases) 

下载并安装，然后设置  **CROSS_COMPILE**  环境变量为工具链所在路径的前缀：

```shell
export CROSS_COMPILE=/path/to/toolchain/install/dir/bin/your-toolchain-prefix-
```

> 3）搭建基础编译环境

```shell
git clone https://github.com/bao-project/bao-demos
cd bao-demos
export PLATFORM=qemu-riscv64-virt
export DEMO=baremetal
```

> 4）编译运行

```shell
make -j$(nproc)
make run
```

---

以上直接使用 `bao-demos` 提供的Makefile进行自动化构建，项目还提供了一个[step-by-step](https://github.com/bao-project/bao-demos#b-follow-the-step-by-step-guide)的文档。

---

跑一遍编译运行过程，以两种guest组合为例：`baremetal/linux+freertos`

> DEMO=baremetal

`make -j$(nproc)`

```shell
make -C /home/zq/contents/objs/bao-project/bao-demos/wrkdir/srcs/baremetal PLATFORM=qemu-riscv64-virt  
make[1]: warning: jobserver unavailable: using -j1.  Add '+' to parent make rule.
make[1]: Entering directory '/home/zq/contents/objs/bao-project/bao-demos/wrkdir/srcs/baremetal'
make[1]: Nothing to be done for 'all'.
make[1]: Leaving directory '/home/zq/contents/objs/bao-project/bao-demos/wrkdir/srcs/baremetal'
cp /home/zq/contents/objs/bao-project/bao-demos/wrkdir/srcs/baremetal/build/qemu-riscv64-virt/baremetal.bin /home/zq/contents/objs/bao-project/bao-demos/wrkdir/imgs/qemu-riscv64-virt/baremetal/baremetal.bin
cp -L /home/zq/contents/objs/bao-project/bao-demos/demos/baremetal/configs/qemu-riscv64-virt.c /home/zq/contents/objs/bao-project/bao-demos/wrkdir/imgs/qemu-riscv64-virt/baremetal/config/baremetal.c
make -C /home/zq/contents/objs/bao-project/bao-demos/wrkdir/srcs/bao\
	PLATFORM=qemu-riscv64-virt\
	CONFIG_REPO=/home/zq/contents/objs/bao-project/bao-demos/wrkdir/imgs/qemu-riscv64-virt/baremetal/config\
	CONFIG=baremetal \
	CPPFLAGS=-DBAO_DEMOS_WRKDIR_IMGS=/home/zq/contents/objs/bao-project/bao-demos/wrkdir/imgs/qemu-riscv64-virt/baremetal
make[1]: Entering directory '/home/zq/contents/objs/bao-project/bao-demos/wrkdir/srcs/bao'
Creating directory	 build/qemu-riscv64-virt/baremetal/arch/riscv/inc
Creating directory	 build/qemu-riscv64-virt/baremetal/scripts
Creating directory	 build/qemu-riscv64-virt/baremetal/config
Creating directory	 build/qemu-riscv64-virt/baremetal/platform
Creating directory	 build/qemu-riscv64-virt/baremetal
Creating directory	 build/qemu-riscv64-virt/baremetal/platform/drivers/sbi_uart
Creating directory	 build/qemu-riscv64-virt/baremetal/platform/qemu-riscv64-virt
Creating directory	 build/qemu-riscv64-virt/baremetal/core/mmu
Creating directory	 build/qemu-riscv64-virt/baremetal/core
Creating directory	 build/qemu-riscv64-virt/baremetal/lib
Generating header	 build/qemu-riscv64-virt/baremetal/arch/riscv/inc/asm_defs.h
Compiling generator	 build/qemu-riscv64-virt/baremetal/scripts/config_defs_gen
Compiling generator	 build/qemu-riscv64-virt/baremetal/scripts/platform_defs_gen
Generating header	 build/qemu-riscv64-virt/baremetal/config/config_defs_gen.h
Generating header	 build/qemu-riscv64-virt/baremetal/platform/platform_defs_gen.h
Creating dependency	 build/qemu-riscv64-virt/baremetal/config/baremetal
Creating dependency	 build/qemu-riscv64-virt/baremetal/arch/riscv/inc/asm_defs.h
Creating dependency	 src/linker.ld
Creating dependency	 src/platform/drivers/sbi_uart/sbi_uart.c
Creating dependency	 src/platform/qemu-riscv64-virt/virt_desc.c
Creating dependency	 src/core/mmu/vm.c
Creating dependency	 src/core/mmu/vmm.c
Creating dependency	 src/core/mmu/io.c
Creating dependency	 src/core/mmu/mem.c
Creating dependency	 src/core/hypercall.c
Creating dependency	 src/core/objpool.c
Creating dependency	 src/core/ipc.c
Creating dependency	 src/core/console.c
Creating dependency	 src/core/config.c
Creating dependency	 src/core/vm.c
Creating dependency	 src/core/vmm.c
Creating dependency	 src/core/cpu.c
Creating dependency	 src/core/interrupts.c
Creating dependency	 src/core/cache.c
Creating dependency	 src/core/mem.c
Creating dependency	 src/core/init.c
Creating dependency	 src/lib/printk.c
Creating dependency	 src/lib/bitmap.c
Creating dependency	 src/lib/string.c
Creating dependency	 src/arch/riscv/relocate.S
Creating dependency	 src/arch/riscv/iommu.c
Creating dependency	 src/arch/riscv/cache.c
Creating dependency	 src/arch/riscv/cpu.c
Creating dependency	 src/arch/riscv/sync_exceptions.c
Creating dependency	 src/arch/riscv/vplic.c
Creating dependency	 src/arch/riscv/interrupts.c
Creating dependency	 src/arch/riscv/plic.c
Creating dependency	 src/arch/riscv/vmm.c
Creating dependency	 src/arch/riscv/vm.c
Creating dependency	 src/arch/riscv/mem.c
Creating dependency	 src/arch/riscv/page_table.c
Creating dependency	 src/arch/riscv/sbi.c
Creating dependency	 src/arch/riscv/root_pt.S
Creating dependency	 src/arch/riscv/exceptions.S
Creating dependency	 src/arch/riscv/boot.S
Generating header	 build/qemu-riscv64-virt/baremetal/arch/riscv/inc/asm_defs.h
Creating dependency	 src/arch/riscv/relocate.S
Creating dependency	 src/arch/riscv/exceptions.S
Creating dependency	 src/arch/riscv/boot.S
Compiling source	 src/arch/riscv/boot.S
Compiling source	 src/arch/riscv/exceptions.S
Compiling source	 src/arch/riscv/root_pt.S
Compiling source	 src/arch/riscv/sbi.c
Compiling source	 src/arch/riscv/page_table.c
Compiling source	 src/arch/riscv/mem.c
Compiling source	 src/arch/riscv/vm.c
Compiling source	 src/arch/riscv/vmm.c
Compiling source	 src/arch/riscv/plic.c
Compiling source	 src/arch/riscv/interrupts.c
Compiling source	 src/arch/riscv/sync_exceptions.c
Compiling source	 src/arch/riscv/vplic.c
Compiling source	 src/arch/riscv/cpu.c
Compiling source	 src/arch/riscv/cache.c
Compiling source	 src/arch/riscv/iommu.c
Compiling source	 src/arch/riscv/relocate.S
Compiling source	 src/lib/string.c
Compiling source	 src/lib/printk.c
Compiling source	 src/lib/bitmap.c
Compiling source	 src/core/init.c
Compiling source	 src/core/mem.c
Compiling source	 src/core/cache.c
Compiling source	 src/core/interrupts.c
Compiling source	 src/core/cpu.c
Compiling source	 src/core/vmm.c
Compiling source	 src/core/vm.c
Compiling source	 src/core/config.c
Compiling source	 src/core/console.c
Compiling source	 src/core/ipc.c
Compiling source	 src/core/objpool.c
Compiling source	 src/core/hypercall.c
Compiling source	 src/core/mmu/mem.c
Compiling source	 src/core/mmu/io.c
Compiling source	 src/core/mmu/vmm.c
Compiling source	 src/core/mmu/vm.c
Compiling source	 src/platform/qemu-riscv64-virt/virt_desc.c
Compiling source	 src/platform/drivers/sbi_uart/sbi_uart.c
Compiling source	/home/zq/contents/objs/bao-project/bao-demos/wrkdir/imgs/qemu-riscv64-virt/baremetal/config/baremetal.c
Pre-processing		 src/linker.ld
Creating directory	 bin/qemu-riscv64-virt/baremetal
Linking			 bin/qemu-riscv64-virt/baremetal/bao.elf
Generating binary	 bin/qemu-riscv64-virt/baremetal/bao.bin
make[1]: Leaving directory '/home/zq/contents/objs/bao-project/bao-demos/wrkdir/srcs/bao'
cp /home/zq/contents/objs/bao-project/bao-demos/wrkdir/srcs/bao/bin/qemu-riscv64-virt/baremetal/bao.bin /home/zq/contents/objs/bao-project/bao-demos/wrkdir/imgs/qemu-riscv64-virt/baremetal/bao.bin
make -C /home/zq/contents/objs/bao-project/bao-demos/wrkdir/srcs/opensbi PLATFORM=generic FW_PAYLOAD=y FW_PAYLOAD_FDT_ADDR=0x80100000 FW_PAYLOAD_PATH=/home/zq/contents/objs/bao-project/bao-demos/wrkdir/imgs/qemu-riscv64-virt/baremetal/bao.bin
make[1]: warning: jobserver unavailable: using -j1.  Add '+' to parent make rule.
make[1]: Entering directory '/home/zq/contents/objs/bao-project/bao-demos/wrkdir/srcs/opensbi'
 AS        platform/generic/firmware/fw_payload.o
 ELF       platform/generic/firmware/fw_payload.elf
 OBJCOPY   platform/generic/firmware/fw_payload.bin
make[1]: Leaving directory '/home/zq/contents/objs/bao-project/bao-demos/wrkdir/srcs/opensbi'
cp /home/zq/contents/objs/bao-project/bao-demos/wrkdir/srcs/opensbi/build/platform/generic/firmware/fw_payload.elf /home/zq/contents/objs/bao-project/bao-demos/wrkdir/imgs/qemu-riscv64-virt/baremetal/opensbi.elf

To start qemu execute "make PLATFORM=qemu-riscv64-virt DEMO=baremetal run"
```

`make run`

```shell
--------------------------------------------------------------------------------
/bin/bash: line 1: pandoc: command not found
--------------------------------------------------------------------------------
qemu-system-riscv64 -nographic\
	-M virt -cpu rv64 -m 4G -smp 4\
	-bios /home/zq/contents/objs/bao-project/bao-demos/wrkdir/imgs/qemu-riscv64-virt/baremetal/opensbi.elf\
	-device virtio-net-device,netdev=net0\
	-netdev user,id=net0,net=192.168.42.0/24,hostfwd=tcp:127.0.0.1:5555-:22\
	-device virtio-serial-device -chardev pty,id=serial3 -device virtconsole,chardev=serial3\
	-S
char device redirected to /dev/pts/4 (label serial3)
...
```

开启一个新终端，然后建立连接：

```shell
pyserial-miniterm --filter=direct /dev/pts/4
```

在qemu窗口中按住 `CTRL + A + C`，可以看到：

[Bao-Baremetal](https://asciinema.org/a/613609)

> DEMO=linux+freertos

[Bao-Baremetal + Linux](https://asciinema.org/a/620804)

# 2 bao-demos构建运行逻辑

`bao-demos` 跑起来只需要执行几条命令，但项目编译运行的过程中有着比较复杂的依赖关系，将整个逻辑梳理清楚是非常必要的。

## 2.1 项目结构

> 本文只关注 qemu-riscv64-virt + baremetal/linux+freertos

`bao-demos` 是由多层Makefile构成的，展开结构如下：

```c
bao-demos
+-> demos
   	+-> baremetal
    	+-> configs
    		qemu-riscv64-virt.c
    	make.mk
    +-> linux+freertos
    	+-> configs
    		qemu-riscv64-virt.c
    	+-> devicetrees
    		+-> qemu-riscv64-virt
    			linux.dts
    	make.mk
+-> guests
    +-> baremetal
    	make.k
    +-> freertos
    	make.k
    +-> linux
    	+-> buildroot
    	+-> configs
    	+-> lloader
    	+-> patches
    	make.mk
+-> platforms
    +-> qemu-riscv64-virt
    	make.mk
    opensbi.mk
    qemu.mk
    uboot.mk
Makefile
```

* `demos`：混合嵌入式系统的各种组合，包括每种系统在虚拟化层的分区参数，以及向 `guests/*/make.mk` 传递一些编译参数；
* `guests`：客户机系统的仓库分支克隆、编译构建；
* `platforms`：运行平台、固件的仓库分支克隆、编译构建，对于qemu+riscv64来说需要克隆并编译 `OpenSBI/Qemu`代码； 

---

> 必须设置的环境变量

这些Makefile划分为: `master_mk`、`demos_mk/guest_mk/platform_mk` ，平台部分还涉及 `opensbi.mk/uboot.mk` 等。先看一下项目最开始的环境变量设置：

```makefile
ifeq ($(filter clean distclean, $(MAKECMDGOALS)),)
ifndef CROSS_COMPILE
 $(error No CROSS_COMPILE prefix defined)
endif
endif

ifeq ($(filter distclean, $(MAKECMDGOALS)),)

ifndef PLATFORM
 $(error No target PLATFORM defined)
endif

ifeq ($(wildcard $(platform_dir)),)
 $(error Target platform $(PLATFORM) is not supported)
endif

ifndef DEMO
 $(error No target DEMO defined.)
endif
    
ifeq ($(wildcard $(demo_dir)),)
 $(error Target demo $(DEMO) is not supported)
endif

ifeq ($(wildcard $(demo_dir)/configs/$(PLATFORM).c),)
 $(error The $(DEMO) demo is not supported by the $(PLATFORM) platform)
endif

endif 
```

* 必须先定义项目支持的 `CROSS_COMPILE prefix/PLATFORM/DEMO`；

---

> 设置工作目录

```makefile
# setup working directories

wrkdir:=$(bao_demos)/wrkdir
wrkdir_src:=$(wrkdir)/srcs
wrkdir_bin:=$(wrkdir)/bin
wrkdir_imgs:=$(wrkdir)/imgs
wrkdir_plat_imgs:=$(wrkdir_imgs)/$(PLATFORM)
wrkdir_demo_imgs:=$(wrkdir_plat_imgs)/$(DEMO)
wrkdirs=$(wrkdir) $(wrkdir_src) $(wrkdir_bin) $(wrkdir_plat_imgs) $(wrkdir_demo_imgs)

environment:=BAO_DEMOS=$(bao_demos)
environment+=BAO_DEMOS_WRKDIR=$(wrkdir)
environment+=BAO_DEMOS_WRKDIR_SRC=$(wrkdir_src)
environment+=BAO_DEMOS_WRKDIR_PLAT=$(wrkdir_plat_imgs)
environment+=BAO_DEMOS_WRKDIR_IMGS=$(wrkdir_demo_imgs)
environment+=BAO_DEMOS_SDCARD_DEV=/dev/yoursdcarddev
environment+=BAO_DEMOS_SDCARD=/media/$$USER/boot
```

---

之后的构建运行使用以下环境设置：

```shell
export PLATFORM=qemu-riscv64-virt
export DEMO=baremetal
```

## 2.2 编译

> make -j$(nproc)

```makefile
all: platform

bao_repo:=https://github.com/bao-project/bao-hypervisor
bao_version:=demo
bao_src:=$(wrkdir_src)/bao
bao_cfg_repo:=$(wrkdir_demo_imgs)/config
wrkdirs+=$(bao_cfg_repo)
bao_cfg:=$(bao_cfg_repo)/$(DEMO).c
bao_image:=$(wrkdir_demo_imgs)/bao.bin

include $(platform_dir)/make.mk
include $(demo_dir)/make.mk

ifeq ($(filter clean distclean, $(MAKECMDGOALS)),)
$(shell mkdir -p $(wrkdirs))
endif

guests: $(guest_images)

$(bao_src):
	git clone --branch $(bao_version) $(bao_repo) $(bao_src)

$(bao_cfg): | $(bao_cfg_repo)
	cp -L $(bao_demos)/demos/$(DEMO)/configs/$(PLATFORM).c $(bao_cfg)

bao $(bao_image): $(guest_images) $(bao_cfg) $(bao_src) 
	$(MAKE) -C $(bao_src)\
		PLATFORM=$(PLATFORM)\
		CONFIG_REPO=$(bao_cfg_repo)\
		CONFIG=$(DEMO) \
		CPPFLAGS=-DBAO_DEMOS_WRKDIR_IMGS=$(wrkdir_demo_imgs)
	cp $(bao_src)/bin/$(PLATFORM)/$(DEMO)/bao.bin $(bao_image)

platform: $(bao_image)
```

编译顺序梳理如下：

```c
all
   <- platform
   	  <- $(bao_image)
    	/*
    		$(MAKE) -C $(bao_src)\
				PLATFORM=$(PLATFORM)\
				CONFIG_REPO=$(bao_cfg_repo)\
				CONFIG=$(DEMO) \
				CPPFLAGS=-DBAO_DEMOS_WRKDIR_IMGS=$(wrkdir_demo_imgs)
			cp $(bao_src)/bin/$(PLATFORM)/$(DEMO)/bao.bin $(bao_image)
    	*/
         <- $(guest_images)
    	 <- $(bao_cfg)
    	 <- $(bao_src)
```

可以看到，主Makefile最后会拉取核心的 `bao-hypervisor` 源码并编译生成 `$(bao_image)`，但在此之前依赖三个对象：

*  `$(guest_images)`
* `$(bao_cfg)`
* `$(bao_src)`

我们先来看这三个对象是怎么生成的，最后再分析 `bao-hypervisor` 的编译过程。

### bao_cfgs/bao_src

```makefile
$(bao_src):
	git clone --branch $(bao_version) $(bao_repo) $(bao_src)

$(bao_cfg): | $(bao_cfg_repo)
	cp -L $(bao_demos)/demos/$(DEMO)/configs/$(PLATFORM).c $(bao_cfg)
```

### guest_images

主Makefile中引用了子目录的mk `demos/baremetal/make.mk`：

```makefile
include $(bao_demos)/guests/baremetal/make.mk

baremetal_image:=$(wrkdir_demo_imgs)/baremetal.bin

ifeq ($(ARCH_PROFILE),armv8-r)
baremetal_args:=MEM_BASE=0x10000000
fvpr_image_data:=$(baremetal_image)@0x10000000
endif

$(eval $(call build-baremetal, $(baremetal_image), $(baremetal_args)))

guest_images:=$(baremetal_image)
```

为了生成 `guest_images/baremetal_image(baremetal.bin)`，进一步引用了 `guests/baremetal/make.mk`：

```makefile
baremetal_src:=$(wrkdir_src)/baremetal
baremetal_repo:=https://github.com/bao-project/bao-baremetal-guest.git 
baremetal_branch:=demo

$(baremetal_src):
	git clone $(baremetal_repo) $@ --branch $(baremetal_branch)

baremetal_bin:=$(baremetal_src)/build/$(PLATFORM)/baremetal.bin

define build-baremetal
$(strip $1): $(baremetal_src)
	$(MAKE) -C $(baremetal_src) PLATFORM=$(PLATFORM) $(strip $2) 
	cp $(baremetal_bin) $$@
endef
```

可以看到调用了 `build-baremetal` 函数，该函数将生成 `baremetal_image`，依赖 `baremetal_src`。克隆指定仓库分支后，最终编译生成 `baremetal.bin` 客户机镜像。

### bao-hypervisor编译

`bao-demos` 的主Makefile编译参数为：

```makefile
$(MAKE) -C $(bao_src)\
	PLATFORM=$(PLATFORM)\
	CONFIG_REPO=$(bao_cfg_repo)\
	CONFIG=$(DEMO) \
	CPPFLAGS=-DBAO_DEMOS_WRKDIR_IMGS=$(wrkdir_demo_imgs)
```

- [ ] TODO: `bao-hypervisor` 的编译过程

## 2.3 运行

> make run

将视线转回到 `bao-demos/Makefile` 中，其引用了 `$(platform_dir)/make.mk`：

```makefile
ARCH:=riscv64

include $(bao_demos)/platforms/qemu.mk
include $(bao_demos)/platforms/opensbi.mk

opensbi_image:=$(wrkdir_demo_imgs)/opensbi.elf
$(eval $(call build-opensbi-payload, $(opensbi_image), $(bao_image)))

platform: $(opensbi_image)

instructions:=$(bao_demos)/platforms/$(PLATFORM)/README.md
run: qemu platform
	$(call print-instructions, $(instructions), 1, true)
	$(qemu_cmd) -nographic\
		-M virt -cpu rv64 -m 4G -smp 4\
		-bios $(opensbi_image)\
		-device virtio-net-device,netdev=net0\
		-netdev user,id=net0,net=192.168.42.0/24,hostfwd=tcp:127.0.0.1:5555-:22\
		-device virtio-serial-device -chardev pty,id=serial3 -device virtconsole,chardev=serial3\
		-S
```

run目标依赖 `qemu/platform`，说一下 `opensbi_image` 的构建。这里调用了 `build-opensbi-payload`，并将之前编译好的 `bao_image` 传入：

```makefile
opensbi_repo:=https://github.com/bao-project/opensbi.git
opensbi_version:=bao/demo
opensbi_src:=$(wrkdir_src)/opensbi

$(opensbi_src):
	git clone --depth 1 --branch $(opensbi_version) $(opensbi_repo) $(opensbi_src)

define build-opensbi-payload
$(strip $1): $(strip $2) $(opensbi_src) 
	$(MAKE) -C $(opensbi_src) PLATFORM=generic \
		FW_PAYLOAD=y \
		FW_PAYLOAD_FDT_ADDR=0x80100000\
		FW_PAYLOAD_PATH=$(strip $2)
	cp $(opensbi_src)/build/platform/generic/firmware/fw_payload.elf $$@
endef
```

上面是一种OpenSBI的编译方式 `FW_PAYLOAD`，也就是将 `bao_image` 和 `opensbi.image` 打包编译成一个ELF文件，[之前的文章](https://zq.org.edu.kg/2023/11/27/Bao_hypervisor%E5%AD%A6%E4%B9%A0/)介绍过。

- [ ] `bao-demos/demos/baremetal/configs/qemu-riscv64-virt.c` 将baremetal.bin链接到bao.bin中？？

