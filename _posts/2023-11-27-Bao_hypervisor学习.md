---
title: Bao_hypervisor学习

date: 2023-11-27 17:00:00 +0800

categories: [开源项目, Bao_hypervisor]

tags: [Bao_hypervisor, riscv64]

description: 
---

# 0 引言

Bao_hypervisor是一个轻量级的开源Type1-hypervisor，主要应用于嵌入式设备，本文主要记录一下该项目的构建、运行过程。构建项目之前，需要一些前置知识以及嵌入式虚拟化技术的相关背景，特此记录。

## 0.1 嵌入式虚拟化

### 介绍

随着物联网设备的爆炸式增长和万物互联应用的快速发展，虚拟化技术在嵌入式系统上受到了业界越来越多的关注、重视和实际应用。嵌入式系统与虚拟化技术这个跨界创新组合应运而生，其典型的应用场景包括软件定义汽车驾驶舱、工业领域的工作负载整合等。

能够促使嵌入式设备支持虚拟化技术的原因有如下几点：

* 随着半导体技术的发展，摩尔定律推动硬件的性能提升，成本下降。今天的嵌入式SoC的性能甚至可能超过了昨天的服务器。
* 无处不在的CPU多核技术的发展自然地能够支持多个系统。
* 是不同业务的负载整合、数字化互联的需求。
* 节约硬件成本考虑，减少硬件系统的互连，降低整体硬件系统的复杂度。
* 系统需要重用已有的软件系统，降低移植工作量。还可以把多个 “异构” 的操作系统（实时系统和非实时系统、安全系统和非安全系统）整合在一套系统上。

### 带来的好处

> 整合功能，提高嵌入式系统的利用率

采用新的嵌入式系统后，处理能力成倍增加。从旧设备迁移过来的嵌入式应用将只需要小部分处理能力，导致设备的利用率不高。借助虚拟化技 术，新设备上可以构造出多个虚拟机，分别部署嵌入式应用。由于虚拟机之间彼此隔离，嵌入式应用的运行互不干扰。通过这样的功能整合，提高了设备的利用率。 

对于基于多核芯片的嵌入式系统，虚拟化技术有两方面作用：一方面是构造出 “单核” 的虚拟机， 使得旧的嵌入式应用能不加修改地运行其中，不必担心多核带来的变化；另一方面，虚拟化技术能够进行多核优化，屏蔽细节，向操作系统和应用程序提供更高级的虚拟硬件接口。 

> 降低嵌入式系统的成本 

嵌入式系统的成本包括硬件成本和软件成本。硬件成本减少的例子如TI的C6474产 品。该产品的DSP利用虚拟化产品VLX，使得以前需要两个专用 DSP与一个通用处理器才可支持的各种任务，只需单个性能更高的DSP便可实现。软件成本得以降低的原因在于虚拟化使得老版软件能在新设备上运行。老版软件不需要修改，这就重新利用了现有软件资产的价值。 

> 减少嵌入式系统功耗、重量和机板尺寸

上面的例子中，处理器数量从３个减到了１个。减少处理器芯片将减少嵌入式系统功耗、重量和机板尺寸。对移动终端而言，这意味着携带更轻 便和更长的待机时间。 

> 增加嵌入式系统的可靠性和安全性

通过在不同的虚拟机内分别部署关键任务和非关键任务，彼此的隔离使得非关键任务的任何异常都不会导致关键任务受到干扰。例如，在手机中，通话是关键任务，玩游戏是非关键任务。因游戏软件故障导致通话无法正常进行是不可接受的。把语音呼叫软件与游戏软件分别部署在隔离的虚拟机内，就不会发生上述情形。这增加了嵌入式系统的可靠性。任务或数据的隔离带来更好的安全性，依赖隔离特性，位于某个虚拟机（如运行游戏软件的虚拟机）中的恶意程序无法侵害其他虚拟机（如运行理财软件的虚拟机）的程序或数据。实际上，恶意程序根本无法感知其他虚拟机的存在。 

> 对嵌入式系统的生产商而言，除了能够减少成本，还能够缩短产品上市周期 

这得益于硬件子系统和软件子系统的开发和调试时间的缩减。硬件方面，软件所依赖的硬件平台是在性能较强的处理器上虚拟出来的，比开发对等功能的硬件模块更节省时间。软件方面，老版软件不加修改就能运行，免除了代码移植和调试的时间开销。

### 关键问题

嵌入式虚拟化技术有着诸多好处，但是也面临以下主要问题。部分问题在通用计算机系统上不存在，使得适用于桌面系统或者服务器领域的虚拟化技术方案无法满足需求。

1. **如何保证运行在虚拟机内的实时应用满足实时要求？** 

   运行于嵌入式系统内的应用大多是实时应用。 在虚拟机内运行时，实时应用有可能无法满足实时要求，也就是说实时任务有可能无法在其时限内完成执行。在宿主虚拟机无法占用足够CPU带宽的情形下，这就会发生。修改实时应用或操作系统无法解决问题，VMM调度虚拟机的策略是关键。

2. **如何适应嵌入式软硬件平台的多样性？**

   嵌入式系统种类多样，包括移动终端、医疗设施、工业控制器、无线电通信设备、网络设备、机器人、车载电脑和数字家电等。嵌入式硬件平台的种数比通用计算机硬件平台的种数多出几个数量级。 操作系统和应用软件也是多种多样，甚至没有操作系统。嵌入式虚拟化技术要得以广泛应用，就需面向多种软硬件平台，这无疑为嵌入式虚拟化技术的发展设置了障碍。 

3. **如何有效管理电源？** 

   用电池供电的嵌入式系统，需采用有效管理电源的手段。在没有运用虚拟化技术的嵌入式系统中，操作系统或应用软件对此采取了专门的节电措施。运用虚拟化技术后，因客户操作系统依托的是VMM构造的虚拟机，独立的电源管理决策不再适用。例如，操作系统不能因为自己内部没有工作负荷就把整个硬件系统置于待机状态，因为其他虚拟机还可能在运行。VMM有必要进行全局的电源管理。不过，VMM在缺乏了解虚拟机内部状态的情况下，也不能有效完成电源管理决策。

4. **如何在保证安全的前提下，运行于不同虚拟机的进程之间能够进行通信？**

   不同虚拟机的进程之间需要协作的时候，彼此将进行通信。例如，手机视频处理就可能涉及不同虚拟机的进程：运行在处理基带业务的客户操作系统内的进程通过移动通信网络下载视频文件，运行在处理常规应用的客户操作系统内的播放器完成视频播放。两者之间将进行大数据量传输。虚拟机之间可以建立网络连接，但通过网络进行通信是低效的。既然在同一物理平台上运行，一个高效的办法是共享内存。不过这与资源隔离的原则相冲突。过于自由的内存共享将破坏隔离性和安全性。

### //TODO: 具体应用



## 0.2 关于OpenSBI

### 概述

OpenSBI提供了针对特定平台的固件构建。支持不同类型的固件来处理不同平台早期启动阶段的差异。所有的固件都会根据平台特定的代码以及OpenSBI通用库代码执行平台硬件的相同的初始化过程。所支持的固件类型将因平台早期引导阶段传递的参数的处理方式以及固件之后的引导阶段的处理和执行方式而有所不同。

早期的启动阶段将通过RISC-V CPU的以下寄存器传递信息：

* hardid通过 `a0` 寄存器传递。

* 通过 `a1` 寄存器在内存中的设备树blob地址。地址必须对齐到8个字节。

### OpenSBI目前支持三种不同类型的固件

#### 带有动态信息的固件(FW_DYNMIC)

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311291350213.png" alt="img" style="zoom:50%;" />

FW_DYNAMIC固件在运行时从上一个引导阶段获取下一个引导阶段入口的信息，例如引导加载程序或操作系统内核。

1. 下一个启动阶段 (如bootloader) 和FW_DYNAMIC被上一个启动阶段加载 (如LOADER)；
2. 上一个启动阶段 (即LOADER) 通过 `a2` 寄存器将 `struct fw_dynamic_info` 的位置传递给FW_DYNAMIC

之前的启动阶段 (即LOADER) 需要知道 `struct fw_dynamic_info` ：

```c
struct fw_dynamic_info {
    /** Info magic */
    unsigned long magic;
    /** Info version */
    unsigned long version;
    /** Next booting stage address */
    unsigned long next_addr;
    /** Next booting stage mode */
    unsigned long next_mode;
    /** Options for OpenSBI library */
    unsigned long options;
    /**
     * Preferred boot HART id
     *
     * It is possible that the previous booting stage uses same link
     * address as the FW_DYNAMIC firmware. In this case, the relocation
     * lottery mechanism can potentially overwrite the previous booting
     * stage while other HARTs are still running in the previous booting
     * stage leading to boot-time crash. To avoid this boot-time crash,
     * the previous booting stage can specify last HART that will jump
     * to the FW_DYNAMIC firmware as the preferred boot HART.
     *
     * To avoid specifying a preferred boot HART, the previous booting
     * stage can set it to -1UL which will force the FW_DYNAMIC firmware
     * to use the relocation lottery mechanism.
     */
    unsigned long boot_hart;
} __packed;
```

#### 带跳转地址的固件(FW_JUMP)

<img src="https://img-blog.csdnimg.cn/132065e0895e46829c388726619e6f91.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcmljaGFyZC5kYWk=,size_20,color_FFFFFF,t_70,g_se,x_16" alt="img" style="zoom:50%;" />

FW_JUMP固件假设下一个引导阶段入口的地址是固定的，例如引导加载程序或操作系统内核，而不直接包含下一个阶段的二进制代码。

1. 下一个启动阶段 (bootloader)，上一个启动阶段 (LOADER) 加载FW_JUMP (这对QEMU非常有用，因为我们可以使用预编译的FW_JUMP)
2. 上一个启动阶段 (即LOADER) 必须在一个固定的位置加载下一个启动阶段 (即bootloader)
3. 没有机制将参数从之前的启动阶段 (即LOADER) 传递给FW_JUMP

#### 带负载的固件(FW_PAYLOAD)

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311291359443.png" alt="image-20231129135920396" style="zoom:50%;" />

FW_PAYLOAD固件直接包含了OpenSBI固件执行后启动阶段的二进制代码。通常，这个有效负载将是一个引导加载程序或一个OS内核。

1. OpenSBI FW_PAYLOAD可以是任何S-mode的 `bootloader/os` 镜像文件
2. 允许重写设备树blob(即DTB)
3. 非常适合 `sifve unmached` 板子
4. 每当OpenSBI或BOOTLOADER (U-Boot)改变时，我们必须重新创建FW_PAYLOAD映像
5. 上一个启动阶段 (LOADER) 不会传递参数给下一个启动阶段

### 固件配置和编译

所有固件类型都支持以下通用编译时配置参数:

* `FW_TEXT_ADDR`:  定义OpenSBI固件的执行地址。必选参数。
* `FW_FDT_PATH`:  外部扁平设备树二进制文件的路径，该文件将被嵌入到最终固件的 `.rodata` 部分中。如果没有提供这个选项，那么固件将期望FDT作为参数在之前的引导阶段传递。
* `FW_FDT_PADDING`:  FW_FDT_PATH选项指定嵌入式扁平设备树二进制文件，可选零字节填充。
* `FW_PIC`:  FW_PIC=y生成位置独立的可执行固件镜像。OpenSBI可以运行在任意地址，并保持适当的对齐。因此，原有的迁移机制(FW_PIC=n)将被跳过。换句话说，OpenSBI将直接运行在加载地址，而不需要任何代码移动。这个选项需要一个支持PIE的工具链，默认情况下是开启的。

### 为OpenSBI提供不同的负载

OpenSBI固件可以使用编译时选项接受各种有效负载。通常，这些有效负载指的是下一阶段引导加载程序 (例如U-Boot) 或操作系统内核映像 (例如Linux)。默认情况下，如果在编译时没有指定特定的负载，那么OpenSBI会自动提供一个测试负载。

要在编译时指定有效负载，需要使用make变量 `FW_PAYLOAD_PATH`。

```shell
make PLATFORM=<platform_subdir> FW_PAYLOAD_PATH=<payload path>
```

### OpenSBI固件行为选项

一个可选的编译时标志 `FW_OPTIONS` 可以用来控制OpenSBI固件的运行时行为。

```shell
make PLATFORM=<platform_subdir> FW_OPTIONS=<options>
```

FW_OPTIONS是不同选项的按位值，例如：`FW_OPTIONS=0x1` 代表OpenSBI库禁止启动打印。

对于所有支持的选项，请查看 `include/sbi/sbi_scratch.h` 头文件中的 `enum sbi_scratch_options`。

---

# 1 Bao_hypervisor构建流程

参考：[bao-project/bao-helloworld (github.com)](https://github.com/bao-project/bao-helloworld)

在本指南中，我们将介绍使用Bao虚拟机构建设置所需的不同组件，并学习这些组件之间的相互作用。为此，本指南包含以下主题：

* **一个入门部分，**帮助用户准备我们将构建目标设置的环境。我们还提供了有关虚拟机监控程序实施方面的额外详细文档注释；
* **一个初始设置部分，**探讨系统的不同组件，并获得本指南的第一个实际示例；
* **一个交互式教程，**介绍如何更改在Bao之上运行的虚拟机；
* 一个**更改正在运行的设置**的实际示例；
* 一个展示**不同虚拟机如何共存并相互交互**的示例；

## 1.1 环境搭建

> 建议使用Linux (Ubuntu22.04)

我们建议使用基于Linux的操作系统来充分利用本教程和Bao虚拟机监视器。尽管这些指令可能适用于其他平台，但本指南是在基于Linux的机器上设置的，具体来说是Ubuntu 22.04。这将确保兼容性并在整个教程中提供最佳体验。

> 安装依赖项

在我们深入了解Bao的世界之前，我们需要安装一些依赖项，以实现无缝的设置过程。打开您的终端并运行以下命令来安装必要的软件包：

```shell
sudo apt install build-essential bison flex git libssl-dev ninja-build \
  u-boot-tools pandoc libslirp-dev pkg-config libglib2.0-dev libpixman-1-dev \
  gettext-base curl xterm cmake python3-pip
```

该命令将安装构建和运行Bao所需的基本工具和库。接下来，我们需要安装一些Python包。执行以下命令来完成此操作：

```shell
pip3 install pykwalify packaging pyelftools
```

> 下载并设置工具链

在我们深入探讨之前，让我们确保您拥有适当的工具。我们将指导您获取和配置适用于您目标架构的适当交叉编译工具链：

| Architecture    | Toolchain Name       | Download Link                                                |
| --------------- | -------------------- | ------------------------------------------------------------ |
| Armv8 Aarch64   | aarch64-none-elf-    | [Arm Developer](https://developer.arm.com/downloads/-/arm-gnu-toolchain-downloads) |
| Armv7/8 Aarch32 | arm-none-eabi-       | [Arm Developer](https://developer.arm.com/downloads/-/arm-gnu-toolchain-downloads) |
| RISC-V          | riscv64-unknown-elf- | [SiFive Tools](https://github.com/sifive/freedom-tools/releases) |

下载适用于目标架构的预构建二进制工具链包。然后，使用参考工具链前缀路径设置CROSS_COMPILE环境变量：

```shell
export CROSS_COMPILE=/path/to/toolchain/install/dir/bin/your-toolchain-prefix
```

> 确保你的主机有足够的存储空间

请注意，拥有足够的空闲存储空间对于顺畅的体验非常重要，特别是因为将为Linux客户机虚拟机构建的Linux映像。为了避免任何与存储相关的问题，我们建议在您的系统上至少有13 GiB的可用存储空间。现在，您已经完成了环境设置并安装了所有依赖项，可以开始探索Bao虚拟化程序并创建您的第一个虚拟化环境了。

| Component         | Required storage | Percentage of storage Required |
| ----------------- | ---------------- | ------------------------------ |
| Bao               | 155.8 MiB        | 1.28%                          |
| Guest (Linux)     | 10.5 GiB         | 86.78%                         |
| Guest (freeRTOS)  | 24.8 MiB         | 0.20%                          |
| Guest (baremetal) | 4.2 MiB          | 0.03%                          |
| Tools (QEMU)      | 1.3 GiB          | 10.74%                         |
| Tools (OpenSBI)   | 114 MiB          | 0.92%                          |

## 1.2 初始设置

现在，您已经准备好了，是时候开始这次导览的第一步了。在初始设置部分，我们将探索系统的不同组件，并通过一个实际示例来引导您建立坚实的基础。

我们将从设置工作环境开始。首先，克隆Bao hello world存储库：

```shell
git clone https://github.com/bao-project/bao-helloworld.git
cd bao-helloworld
```

接下来，开始配置开发环境并建立一个目录树来组织所需的各个组件。打开终端并执行以下命令：

```shell
export ROOT_DIR=$(realpath .)
export PRE_BUILT_IMGS=$ROOT_DIR/bin
export SETUP_BUILD=$ROOT_DIR/build
export PATCHES_DIR=$ROOT_DIR/patches
export TOOLS_DIR=$ROOT_DIR/tools
export BUILD_GUESTS_DIR=$SETUP_BUILD/guests
export BUILD_BAO_DIR=$SETUP_BUILD/bao
export BUILD_FIRMWARE_DIR=$SETUP_BUILD/firmware
mkdir -p $BUILD_GUESTS_DIR
mkdir -p $BUILD_BAO_DIR
mkdir -p $BUILD_FIRMWARE_DIR
mkdir -p $TOOLS_DIR/bin
```

完成这些命令后，您的目录树应该如下所示：

```markdown
├── bin
│   ├── bao
│   ├── firmware
│   └── guests
├── configs
│   ├──...
├── img
│  ├──...
└──README.md
```

### 构建第一个Bao-Guest

让我们开始建立您的第一个Bao客户的旅程。在这里，您将获得创建Baremetal客户的实践经验。在我们进入实际方面之前，让我们首先了解我们正在构建的设置。我们的目标是在Bao虚拟化程序的顶部部署一个Baremetal客户，如下图所示：

![image-20231128161913404](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311281619830.png)

为了简化和方便起见，我们将使用QEMU模拟器（不用担心，在教程后面我们会指导您安装它）。然而，请记住您可以将这些步骤应用于其他各种平台。

首先，让我们为裸金属应用程序的源代码定义一个环境变量：

```shell
export BAREMETAL_SRCS=$ROOT_DIR/baremetal
```

然后，克隆我们准备好的Bao裸机客户应用程序（如果您已经有自己的裸机源代码，可以跳过此步骤）：

```shell
git clone https://github.com/bao-project/bao-baremetal-guest.git\
   --branch demo $BAREMETAL_SRCS
```

现在，让我们编译它（为了简单起见，我们的例子包含一个Makefile来编译裸机编译）：

```shell
git -C $BAREMETAL_SRCS apply $PATCHES_DIR/baremetal.patch
make -C $BAREMETAL_SRCS PLATFORM=qemu-riscv64-virt
```

完成这些步骤后，您将在BAREMETAL_SRCS目录中找到一个二进制文件。如果您按照我们提供的Makefile操作，该二进制文件的名称为`baremetal.bin`。现在，请将该二进制文件移动到您的构建目录（BUILD_GUESTS_DIR）中。

```shell
mkdir -p $BUILD_GUESTS_DIR/baremetal-setup
cp $BAREMETAL_SRCS/build/qemu-riscv64-virt/baremetal.bin \
   $BUILD_GUESTS_DIR/baremetal-setup/baremetal.bin
```

### 构建Bao-Hypervisor

接下来，我们将指导您构建Bao Hypervisor本身。第一步是使用Bao的配置文件配置虚拟机监控程序。对于这个特定的设置，我们提供了配置文件以简化过程。如果您对探索不同的配置选项感兴趣，我们详细的Bao配置文档可以帮助您。

如果您使用的目录结构与教程中所示的不同，请确保在配置文件中更新以下代码。

```c
VM_IMAGE(baremetal_image, XSTR(BAO_WRKDIR_IMGS/baremetal-setup/baremetal.bin));
```

下面开始构建过程：

> Cloning the Bao Hypervisor

您进入无缝虚拟化的门户是通过克隆Bao Hypervisor存储库开始的。在终端中执行以下命令以启动这个关键步骤：

```shell
export BAO_SRCS=$ROOT_DIR/bao
git clone https://github.com/bao-project/bao-hypervisor $BAO_SRCS\
   --branch demo
```

> Cloning the Bao Hypervisor

我们已经准备就绪！是时候让Bao虚拟机开始运行了。现在，我们只需要将其编译即可！

```shell
make -C $BAO_SRCS\
   PLATFORM=qemu-riscv64-virt\
   CONFIG_REPO=$ROOT_DIR/configs\
   CONFIG=baremetal\
   CPPFLAGS=-DBAO_WRKDIR_IMGS=$BUILD_GUESTS_DIR
```

完成这些步骤后，您将在BAO_SRCS目录中找到一个名为 `bao.bin` 的二进制文件。现在，让我们将该二进制文件移动到您的构建目录中：

```shell
cp $BAO_SRCS/bin/qemu-riscv64-virt/baremetal/bao.bin $BUILD_BAO_DIR/bao.bin
```

## 1.3 构建固件

固件对于驱动您的虚拟世界至关重要。这就是为什么我们在这里，协助您获取针对目标平台定制的必要固件（您可以在这里找到构建其他平台固件的说明）。

> 将QEMU作为底层平台

QEMU提供了一个方便的硬件平台替代方案。如果您还没有安装它，不用担心。我们将指导您完成构建和安装的过程。在本指南中，我们将针对riscv64架构进行操作。

如果您已经安装了 `qemu-system-riscv64`，或者您更喜欢使用软件包管理器或其他方法直接安装它，请确保您使用的是7.2.0版本或更高版本。在这种情况下，请继续进行下一步。

要安装QEMU，只需运行以下命令：

```shell
export QEMU_DIR=$ROOT_DIR/tools/qemu-riscv64
git clone https://github.com/qemu/qemu.git $QEMU_DIR --depth 1\
  --branch v7.2.0
cd $QEMU_DIR
./configure --target-list=riscv64-softmmu --enable-slirp
make -j$(nproc)
sudo make install
```

> 克隆OpenSBI

要使OpenSBI运行起来，只需执行以下：

```shell
export OPENSBI_DIR=$ROOT_DIR/tools/OpenSBI
git clone https://github.com/bao-project/opensbi.git $OPENSBI_DIR\
   --depth 1 --branch bao/demo
```

## 1.4 运行baremetal-guest

现在一切都已经设置好了，让我们回顾一下已经执行的所有步骤：

✅ Build guest (baremetal)

✅ Build bao hypervisor

✅ Build firmware (qemu)

一切就绪后，我们将继续进行QEMU启动。以下是运行命令：

```shell
make -C $TOOLS_DIR/OpenSBI PLATFORM=generic \
   FW_PAYLOAD=y \
   FW_PAYLOAD_FDT_ADDR=0x80100000\
   FW_PAYLOAD_PATH=$BUILD_BAO_DIR/bao.bin
   
cp $TOOLS_DIR/OpenSBI/build/platform/generic/firmware/fw_payload.elf \
   $TOOLS_DIR/bin/opensbi.elf
   
qemu-system-riscv64 -nographic\
   -M virt -cpu rv64 -m 4G -smp 4\
   -bios $TOOLS_DIR/bin/opensbi.elf\
   -device virtio-net-device,netdev=net0 \
   -netdev user,id=net0,net=192.168.42.0/24,hostfwd=tcp:127.0.0.1:5555-:22\
   -device virtio-serial-device -chardev pty,id=serial3 -device
virtconsole,chardev=serial3
```

> **你或许对qemu的启动命令有所疑惑，因为只出现了一个 `opensbi.elf`，关于OpenSBI的几种编译方式请看0.2小节。**

现在，您应该看到OpenSBI初始化日志。现在，让我们建立连接并进入Bao。这里是一个示例：

您应该得到以下输出（视频在此处）。

![baremetal](https://github.com/bao-project/bao-helloworld/raw/risc-v/img/.gif/baremetal_1vCPU.gif)

## 1.5 调整虚拟机配置

在我们继续前进的过程中，让我们专注于对你的Bao设置进行微调。拥有修改虚拟环境的能力非常有用。在接下来的章节中，我们将为您提供详细的逐步说明，以实现对客户机的各种更改。

通过本节的学习，您将对不同组件之间的相互作用有更深入的了解，并且能够熟练地进行必要的调整以满足您的需求。

### 修改裸金属虚拟机配置

让我们从修改裸金属虚拟机开始。如下图所示，我们将增加分配给裸金属的虚拟中央处理器（vCPUs）数量。

![image-20231128165248429](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311281652720.png)

> 更新虚拟机vCPU数量

让我们从更改分配给虚拟机的vCPU数量开始（更改应用于新的配置文件中）：

```c
-       .cpu_num = 1,                                       | line 17
+       .cpu_num = 4,                                       | line 17
```

> 更新虚拟机内存

1. 打开位于 `baremetal/src/platform/qemu-riscv64-virt/inc/plat.h` 的平台配置文件。

2. 将平台内存大小从64MiB（0x4000000）修改为128 MiB（0x8000000）：

   ```c
   #define PLAT_MEM_BASE 0x80200000
   -#define PLAT_MEM_SIZE 0x4000000
   +#define PLAT_MEM_SIZE 0x8000000
   ```

在更新裸金属之后，您需要修改Bao配置文件。使用新的内存大小更新配置( `configs/baremetal.c` )：

```c
          .regions = (struct vm_mem_region[]) {     | line 20
              {                                       | line 21
                  .base = 0x80200000,                 | line 22
-                   .size = 0x4000000                   | line 23
+                   .size = 0x8000000                   | line 23
              }
          }
```

> 重新编译裸机程序

运行以下命令以重新构建裸金属镜像：

```shell
make -C $BAREMETAL_SRCS PLATFORM=qemu-riscv64-virt
cp $BAREMETAL_SRCS/build/qemu-riscv64-virt/baremetal.bin \
   $BUILD_GUESTS_DIR/baremetal-setup/baremetal.bin
```

> 重新编译Bao

```shell
make -C $BAO_SRCS\
   PLATFORM=qemu-riscv64-virt\
   CONFIG_REPO=$ROOT_DIR/configs\
   CONFIG=baremetal_mod\
   CPPFLAGS=-DBAO_WRKDIR_IMGS=$BUILD_GUESTS_DIR
cp $BAO_SRCS/bin/qemu-riscv64-virt/baremetal_mod/bao.bin $BUILD_BAO_DIR/bao.bin
```

> 启动QEMU

```shell
make -C $TOOLS_DIR/OpenSBI PLATFORM=generic \
   FW_PAYLOAD=y \
   FW_PAYLOAD_FDT_ADDR=0x80100000\
   FW_PAYLOAD_PATH=$BUILD_BAO_DIR/bao.bin

cp $TOOLS_DIR/OpenSBI/build/platform/generic/firmware/fw_payload.elf \
   $TOOLS_DIR/bin/opensbi.elf
   
qemu-system-riscv64 -nographic\
   -M virt -cpu rv64 -m 4G -smp 4\
   -bios $TOOLS_DIR/bin/opensbi.elf\
   -device virtio-net-device,netdev=net0 \
   -netdev user,id=net0,net=192.168.42.0/24,hostfwd=tcp:127.0.0.1:5555-:22\
   -device virtio-serial-device -chardev pty,id=serial3 -device
virtconsole,chardev=serial3
```

现在，您应该看到OpenSBI初始化日志。现在，让我们建立连接并跳转到Bao。这里是一个示例：

您应该得到以下输出（视频在此处）：

![baremetal_4vCPU.gif](https://github.com/bao-project/bao-helloworld/blob/risc-v/img/.gif/baremetal_4vCPU.gif?raw=true)

### 添加第二个虚拟机 - freeRTOS

在本节中，我们将深入探讨各种情景，并演示如何使用Bao配置特定环境。让我们开始加入一个运行FreeRTOS的第二个虚拟机。

![image-20231128170531049](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311281705346.png)

首先，我们可以使用从第一次设置中编译的裸金属系统：

```shell
mkdir -p $BUILD_GUESTS_DIR/baremetal-freeRTOS-setup
cp $BAREMETAL_SRCS/build/qemu-riscv64-virt/baremetal.bin \
   $BUILD_GUESTS_DIR/baremetal-freeRTOS-setup/baremetal.bin
```

#### 编译freeRTOS

接下来，让我们编译我们的新客户机：

```shell
export FREERTOS_SRCS=$ROOT_DIR/freertos
export FREERTOS_PARAMS="STD_ADDR_SPACE=y"

git clone --recursive --shallow-submodules\
    https://github.com/bao-project/freertos-over-bao.git\
    $FREERTOS_SRCS --branch demo
git -C $FREERTOS_SRCS apply $PATCHES_DIR/freeRTOS.patch
make -C $FREERTOS_SRCS PLATFORM=qemu-riscv64-virt $FREERTOS_PARAMS
```

完成这些步骤后，您将在FREERTOS_SRCS目录中找到一个名为freertos.bin的二进制文件。将该二进制文件移动到您的构建目录（BUILD_GUESTS_DIR）：

```shell
cp $FREERTOS_SRCS/build/qemu-riscv64-virt/freertos.bin \
    $BUILD_GUESTS_DIR/baremetal-freeRTOS-setup/freertos.bin
```

#### 整合新虚拟机

现在，我们已经准备好了双客户机设置所需的两个客户机镜像。然而，需要一些步骤来适应我们的平台。让我们了解第一个设置和第二个设置的配置之间的差异。

首先，我们需要添加第二个虚拟机镜像：

```c
- VM_IMAGE(baremetal_image, XSTR(BAO_WRKDIR_IMGS/guests/baremetal-setup/baremetal.bin));
+ VM_IMAGE(baremetal_image, XSTR(BAO_WRKDIR_IMGS/guests/baremetal-freeRTOS-setup/baremetal.bin));
+ VM_IMAGE(freertos_image, XSTR(BAO_WRKDIR_IMGS/guests/baremetal-freeRTOS-setup/freertos.bin));
```

另外，由于我们现在有两个虚拟机，我们需要在配置中更改vm_list_size的值。

```c
- .vmlist_size = 1,
+ .vmlist_size = 2,
```

接下来，我们需要考虑资源。在第一个设置中，我们为裸金属分配了4个vCPU。但是这一次，我们需要将vCPU在两个虚拟机之间进行分割。对于裸金属，我们将使用3个CPU：

```c
- .cpu_num = 4,
+ .cpu_num = 3,
```

对于freeRTOS虚拟机，我们将仅分配一个CPU：

```c
+ .cpu_num = 1,
```

此外，我们还需要包括第二个虚拟机的所有配置。（为简化起见，细节被省略了，但您可以在配置文件中查看更多细节）：

```c
+        {
+            .image = {
+                .base_addr = 0x0,
+                .load_addr = VM_IMAGE_OFFSET(freertos_image),
+                .size = VM_IMAGE_SIZE(freertos_image)
+            },
+
+            ...        // omitted for simplicity
+        },
```

#### 重新构建Bao

正如我们所看到的，更改客户机包括更改配置文件。因此，我们需要重复构建Bao的过程。请注意，标志CONFIG定义了在编译Bao时要使用的配置文件。要编译它，请使用以下命令：

```shell
make -C $BAO_SRCS\
    PLATFORM=qemu-riscv64-virt\
    CONFIG_REPO=$ROOT_DIR/configs\
    CONFIG=baremetal-freeRTOS\
    CPPFLAGS=-DBAO_WRKDIR_IMGS=$BUILD_GUESTS_DIR
```

完成这些步骤后，您将在BAO_SRCS目录中找到一个名为bao.bin的二进制文件。将该二进制文件移动到您的构建目录（BUILD_BAO_DIR）中。

```shell
cp $BAO_SRCS/bin/qemu-riscv64-virt/baremetal-freeRTOS/bao.bin \
    $BUILD_BAO_DIR/bao.bin
```

#### 启动QMEU

现在，您已经配置好了测试新设置所需的一切！只需运行以下命令：

```shell
make -C $TOOLS_DIR/OpenSBI PLATFORM=generic \
    FW_PAYLOAD=y \
    FW_PAYLOAD_FDT_ADDR=0x80100000\
    FW_PAYLOAD_PATH=$BUILD_BAO_DIR/bao.bin

cp $TOOLS_DIR/OpenSBI/build/platform/generic/firmware/fw_payload.elf \
    $TOOLS_DIR/bin/opensbi.elf

qemu-system-riscv64 -nographic\
    -M virt -cpu rv64 -m 4G -smp 4\
    -bios $TOOLS_DIR/bin/opensbi.elf\
    -device virtio-net-device,netdev=net0 \
    -netdev user,id=net0,net=192.168.42.0/24,hostfwd=tcp:127.0.0.1:5555-:22\
    -device virtio-serial-device -chardev pty,id=serial3 -device virtconsole,chardev=serial3
```

现在，您应该有以下输出（视频在此处）：

### 将Linux加入到混合系统集

现在，让我们介绍第三个运行Linux操作系统的虚拟机。

![image-20231128170835278](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202311281708586.png)

首先，我们可以重复利用前一次设置中的来宾。

```shell
mkdir -p $BUILD_GUESTS_DIR/baremetal-linux-setup
cp $BAREMETAL_SRCS/build/qemu-riscv64-virt/baremetal.bin \
   $BUILD_GUESTS_DIR/baremetal-linux-setup/baremetal.bin
```

#### Step-1: 构建Linux-Guest

现在让我们从构建我们的Linux虚拟机开始。设置Linux环境变量：

```shell
export LINUX_DIR=$ROOT_DIR/linux
export LINUX_REPO=https://github.com/torvalds/linux.git
export LINUX_VERSION=v6.1

export LINUX_SRCS=$LINUX_DIR/linux-$LINUX_VERSION

mkdir -p $LINUX_DIR/linux-$LINUX_VERSION
mkdir -p $LINUX_DIR/linux-build

git clone $LINUX_REPO $LINUX_SRCS\
   --depth 1 --branch $LINUX_VERSION
cd $LINUX_SRCS
git apply $ROOT_DIR/srcs/patches/$LINUX_VERSION/*.patch
```

如果您希望跳过这些步骤并使用预构建的Linux镜像，则执行以下命令并继续进行 `Step-2`。

```shell
mkdir -p $BUILD_GUESTS_DIR/baremetal-linux-setup
cp $PRE_BUILT_IMGS/guests/baremetal-linux-setup/linux.bin \
   $BUILD_GUESTS_DIR/baremetal-linux-setup/linux.bin
```

设置一个指向目标架构和特定平台配置的环境变量，以供buildroot使用：

```shell
export LINUX_CFG_FRAG=$(ls $ROOT_DIR/srcs/configs/base.config\
   $ROOT_DIR/srcs/configs/riscv64.config\
   $ROOT_DIR/srcs/configs/qemu-riscv64-virt.config 2> /dev/null)
```

设置Buildroot环境变量：

```shell
export BUILDROOT_SRCS=$LINUX_DIR/buildroot-riscv64-$LINUX_VERSION
export BUILDROOT_DEFCFG=$ROOT_DIR/srcs/buildroot/riscv64.config
export LINUX_OVERRIDE_SRCDIR=$LINUX_SRCS
```

在最新的稳定版本下克隆最新的Buildroot。

```shell
git clone https://github.com/buildroot/buildroot.git $BUILDROOT_SRCS\
   --depth 1 --branch 2022.11
cd $BUILDROOT_SRCS
```

请使用我们提供的buildroot defconfig，其中包含指向Linux内核defconfig的补丁和构建选项。

```shell
make defconfig BR2_DEFCONFIG=$BUILDROOT_DEFCFG
make linux-reconfigure all
mv $BUILDROOT_SRCS/output/images/Image\
   $BUILDROOT_SRCS/output/images/Image-qemu-riscv64-virt
```

这个设置的设备树可以在 `srcs/devicetrees/qemu-riscv64-virt` 目录中找到。对于名为 `linux.dts` 的设备树文件，定义一个环境变量并进行构建：

```shell
export LINUX_VM=linux
dtc $ROOT_DIR/srcs/devicetrees/qemu-riscv64-virt/$LINUX_VM.dts >\
   $LINUX_DIR/linux-build/$LINUX_VM.dtb
```

将内核镜像和设备树二进制包装成一个单一的二进制文件：

```shell
make -j $(nproc) -C $ROOT_DIR/srcs/lloader\
   ARCH=riscv64\
   IMAGE=$BUILDROOT_SRCS/output/images/Image-qemu-riscv64-virt\
   DTB=$LINUX_DIR/linux-build/$LINUX_VM.dtb\
   TARGET=$LINUX_DIR/linux-build/$LINUX_VM
```

最后，将二进制文件复制到（已编译）的客户文件夹中：

```shell
cp $LINUX_DIR/linux-build/$LINUX_VM.bin \
   $BUILD_GUESTS_DIR/baremetal-linux-setup/linux.bin
```

#### Step-2: 整合到系统中

在构建了我们的新客户机之后，现在是将其整合到我们的设置中的时候了。

首先，我们需要加载我们的客户机：

```c
- VM_IMAGE(baremetal_image, XSTR(BAO_WRKDIR_IMGS/guests/baremetal-freeRTOSsetup/baremetal.bin));
+ VM_IMAGE(baremetal_image, XSTR(BAO_WRKDIR_IMGS/guests/baremetal-linuxsetup/baremetal.bin));
+ VM_IMAGE(linux_image, XSTR(BAO_WRKDIR_IMGS/guests/baremetal-linuxsetup/linux.bin));
```

让我们现在更新虚拟机列表的大小，以便整合我们的新客户端：

```shell
-   .vmlist_size = 1,
+   .vmlist_size = 2,
```

然后，我们需要重新调整vCPU的数量：

```c
// baremetal configuration
  {
-       .cpu_num = 2,
+       .cpu_num = 1,
      ...
  },
  // linux configuration
  {
+       .cpu_num = 3,
  }
```

此外，您还可以选择配置Linux虚拟机以集成各种设备，甚至内存区域。有关此设置的具体细节，请参阅配置文件。

#### Step-3: 重新构建Bao

正如前文所述，我们改变了客户机的配置文件，因此用以下命令重新构建Bao：

```shell
make -C $BAO_SRCS\
    PLATFORM=qemu-riscv64-virt\
    CONFIG_REPO=$ROOT_DIR/configs\
    CONFIG=baremetal-linux\
    CPPFLAGS=-DBAO_WRKDIR_IMGS=$BUILD_GUESTS_DIR
```

完成以上步骤后，你将在 `BAO_SRCS` 目录中看到一个二进制文件，即 `bao.bin`，将其移动至对应的构建目录 `BUILD_BAO_DIR`：

```shell
cp $BAO_SRCS/bin/qemu-riscv64-virt/baremetal-linux/bao.bin \
    $BUILD_BAO_DIR/bao.bin
```

#### Step-4: 启动QEMU

必需的组件已经就绪，启动QEMU：

```shell
make -C $TOOLS_DIR/OpenSBI PLATFORM=generic \
    FW_PAYLOAD=y \
    FW_PAYLOAD_FDT_ADDR=0x80100000\
    FW_PAYLOAD_PATH=$BUILD_BAO_DIR/bao.bin

cp $TOOLS_DIR/OpenSBI/build/platform/generic/firmware/fw_payload.elf \
    $TOOLS_DIR/bin/opensbi.elf

qemu-system-riscv64 -nographic\
    -M virt -cpu rv64 -m 4G -smp 4\
    -bios $TOOLS_DIR/bin/opensbi.elf\
    -device virtio-net-device,netdev=net0 \
    -netdev user,id=net0,net=192.168.42.0/24,hostfwd=tcp:127.0.0.1:5555-:22\
    -device virtio-serial-device -chardev pty,id=serial3 -device virtconsole,chardev=serial3
```

为建立连接，请打开一个新的终端窗口并连接到指定的伪终端。具体步骤如下：

```shell
pyserial-miniterm --filter=direct /dev/pts/4
```

请注意，每个用户的pts端口可能会有所不同。要找到正确的pts端口，请参考qemu输出控制台。
最后，您应该会看到以下输出（视频在此处）。

### 虚拟机间通信

在特定的情景下，建立一个通信渠道对于客户机来说是很重要的。为了实现这一点，我们将利用共享内存对象和进程间通信（IPC）机制，使Linux虚拟机能够与系统无缝交互。

#### 为虚拟机添加共享内存和IPC功能

让我们从修改我们的裸机程序以处理通过IPC发送的消息开始。我们已经准备了一个可以直接应用于裸机源代码的补丁。要应用这些更改，请使用以下命令：

```shell
git -C $BAREMETAL_SRCS apply $PATCHES_DIR/baremetal_shmem.patch
```

接下来，重新构建裸金属环境：

```shell
make -C $BAREMETAL_SRCS PLATFORM=qemu-riscv64-virt

mkdir -p $BUILD_GUESTS_DIR/baremetal-linux-shmem-setup
cp $BAREMETAL_SRCS/build/qemu-riscv64-virt/baremetal.bin \
	$BUILD_GUESTS_DIR/baremetal-linux-shmem-setup/baremetal.bin
```

现在，让我们将一个IPC集成到Linux中。为了简化，`linux-shmem.dts` 文件已经包含了以下更改：

```c
+ bao-ipc@f0000000 {
+ 	compatible = "bao,ipcshmem";
+ 	reg = <0x0 0xf0000000 0x0 0x00010000>;
+ 	read-channel = <0x0 0x2000>;
+ 	write-channel = <0x2000 0x2000>;
+ 	interrupts = <0 52 1>;
+ 	id = <0>;
+ };
```

现在，让我们生成更新后的设备树：

```shell
export LINUX_VM=linux-shmem
dtc $ROOT_DIR/srcs/devicetrees/qemu-riscv64-virt/$LINUX_VM.dts >\
	$BUILD_GUESTS_DIR/baremetal-linux-shmem-setup/$LINUX_VM.dtb
```

为了正确引入这些变化，您需要确保按照之前描述的方法将补丁应用到Linux上。将内核镜像和设备树二进制文件捆绑成一个单一的二进制文件：

```shell
make -j $(nproc) -C $ROOT_DIR/srcs/lloader\
    ARCH=riscv64\
    IMAGE=$PRE_BUILT_IMGS/guests/baremetal-linux-shmem-setup/Image-qemu-riscv64-virt\
    DTB=$BUILD_GUESTS_DIR/baremetal-linux-shmem-setup/$LINUX_VM.dtb\
    TARGET=$BUILD_GUESTS_DIR/baremetal-linux-shmem-setup/$LINUX_VM
```

#### 重新构建Bao

鉴于您已经修改了其中一个客户机，现在重建Bao是非常必要的。因此，请使用以下命令编译它：

```shell
make -C $BAO_SRCS\
    PLATFORM=qemu-riscv64-virt\
    CONFIG_REPO=$ROOT_DIR/configs\
    CONFIG=baremetal-linux-shmem\
    CPPFLAGS=-DBAO_WRKDIR_IMGS=$BUILD_GUESTS_DIR
```

成功完成后，您将在BAO_SRCS目录中找到一个名为bao.bin的二进制文件。将其移动到您的构建目录（BUILD_BAO_DIR）中。

```shell
cp $BAO_SRCS/bin/qemu-riscv64-virt/baremetal-linux-shmem/bao.bin \
    $BUILD_BAO_DIR/bao.bin
```

#### 最终的运行

现在，您已经准备好执行最后的设置：

```shell
make -C $TOOLS_DIR/OpenSBI PLATFORM=generic \
    FW_PAYLOAD=y \
    FW_PAYLOAD_FDT_ADDR=0x80100000\
    FW_PAYLOAD_PATH=$BUILD_BAO_DIR/bao.bin

cp $TOOLS_DIR/OpenSBI/build/platform/generic/firmware/fw_payload.elf \
    $TOOLS_DIR/bin/opensbi.elf

qemu-system-riscv64 -nographic\
    -M virt -cpu rv64 -m 4G -smp 4\
    -bios $TOOLS_DIR/bin/opensbi.elf\
    -device virtio-net-device,netdev=net0 \
    -netdev user,id=net0,net=192.168.42.0/24,hostfwd=tcp:127.0.0.1:5555-:22\
    -device virtio-serial-device -chardev pty,id=serial3 -device virtconsole,chardev=serial3
```

如果一切按计划进行，您应该能够通过运行以下命令在Linux上找到IPC。

```shell
ls /dev
```

您将会看到以下的IPC（视频在此）：[Bao Hello World - Baremetal+Linux +shmem qemu-riscv64 - asciinema](https://asciinema.org/a/620803)

从这里，您可以在Linux上使用IPC将消息分发到Baremetal，通过写入 `/dev/baoipc0`。

```shell
echo "Hello, Bao!" > /dev/baoipc0
```

