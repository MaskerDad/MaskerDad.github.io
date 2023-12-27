---
title: 深入浅出系统虚拟化(五)轻量级虚拟化平台StratoVirt

date: 2023-11-24 17:00:00 +0800

categories: [读书笔记, 深入浅出系统虚拟化]

tags: [x86, AArch64, StratoVirt, kvm]

description: 
---

# 0 参考

 [华为 | 基于Rust的下一代虚拟化平台-StratoVirt - Rust精选 (rustmagazine.github.io)](https://rustmagazine.github.io/rust_magazine_2021/chapter_3/hw_rust_stratovirt.html)

[StratoVirt中MicroVM启动过程-CSDN博客](https://blog.csdn.net/u012520854/article/details/125158629)



# 1 StratoVirt概述

StratoVirt是计算产业中面向云数据中心的企业级虚拟化平台，实现了一套统一架构，支持虚拟机、容器和Serverless三种场景，在轻量低噪、软硬协同、安全等方面具备关键技术竞争优势。StratoVirt在架构设计和接口上预留了组件化拼装的能力和接口，因此StratoVirt可以按需灵活组装高级特性，直至演化到支持标准虚拟化，在特性需求、应用场景和轻快灵巧之间找到最佳的平衡点。

本文主要讲述了：

* 介绍StratoVirt的基本能力与特性，使读者了解StratoVirt的使用场景以及技术特点；
* 介绍虚拟机技术的演进；
* 介绍StratoVirt虚拟化技术原理，为后续实现做理论铺垫；
* 结合代码讲解StratoVirt的基本实现，从零开始打造一个具备基本功能的轻量级虚拟化平台。

---

Strato取自地球大气层中的平流层 (stratosphere)。大气层可以保护地球不受外界环境侵害，平流层则是大气层中最稳定的一层；类似的，虚拟化层是比操作系统更为底层的隔离层，既能保护操作系统平台不受上层恶意应用的破坏，又能为正常应用提供稳定可靠的运行环境。StratoVirt中的Strato寓意其为保护openEuler平台上业务平稳运行的轻薄保护层；同时，它也承载了项目的愿景与未来：轻量、灵活、安全和完整的保护能力。

StratoVirt是openEuler平台依托于虚拟化技术打造的稳定和坚固的保护层，它重构了openEuler虚拟化底座，具有以下**六大技术特点。**

1. **强安全性与隔离性。**采用内存安全语言Rust编写，保证语言级安全性；基于硬件辅助虚拟化实现安全多租户隔离，并通过 `seccomp` 进一步约束非必要的系统调用，减小系统攻击面。
2. **轻量低噪。**轻量化场景下冷启动时间 < 50ms，内存开销 < 4MB。
3. **高速稳定的I/O能力。**具有精简的设备模型，并提供了稳定高速的I/O能力。
4. **资源伸缩。**具有毫秒级别的设备伸缩时延，为轻量化负载提供灵活的资源伸缩能力。
5. **全场景支持。**目前支持x86和ARM平台。x86支持VT、鲲鹏支持Kunpeng-V，实现多体系硬件加速；可与容器Kubernetes生态无缝集成，在虚拟机、容器和Serverless场景有广阔的应用空间。
6. **扩展性。**架构设计完备，各个组件可灵活地配置和拆分；设备模型可扩展，可以扩展PCIe等复杂设备规范，向通用标准虚拟机演进。

# 2 发展背景

在开源虚拟化技术的发展历程中，QEMU/KVM 一直是整个虚拟化产业发展的基石和主线。随着多年的发展和迭代，QEMU也沉积了庞大的代码基线和繁多的历史设备。据统计 QEMU 已有 `157` 万行代码，而且其中很大一部分代码是用于历史遗留 (legacy) 功能或者设备的，功能和设备严重耦合在一起，导致在轻量化场景中无法轻装上阵。

> **StratoVirt采用精简的设备模型，提供高速稳定的I/O能力，做到轻量低噪，达到毫秒级的资源伸缩能力，同时架构设计预留了组件化能力，支撑向标准虚拟化方向演进。**

另一方面，在过去十几年QEMU的CVE（Common Vulnerabilities & Exposures，通用漏洞披露）安全问题中，发现其中有将近一半是因为内存问题导致的，例如缓冲区溢出、内存非法访问等。如何有效避免产生内存问题，成为编程语言选型方面的重要考虑。因此，专注于安全的Rust语言脱颖而出。Rust语言拥有强大的类型系统、所有权系统、借用和生命周期等机制，不仅保证内存安全，还保证并发安全，极大地提升软件的质量。在支持安全性的同时，具有零成本抽象的特点，既提升代码的可读性，又不影响代码的运行时性能。同时，Rust语言拥有强大的软件包管理器和项目管理工具—— `Cargo`，不仅能够方便、统一和灵活地管理项目，还提供了大量的代码扫描工具，能进一步提升开发者的编码风格和代码质量。

业界有很多厂商也在尝试使用Rust语言发展虚拟化技术。谷歌公司是最早尝试使用Rust语言进行虚拟化开发的厂商之一，推出了CrosVM项目，它是Chrome操作系统中用于创建虚拟机的应用。后来亚马逊公司基于谷歌公司开源的CrosVM项目的部分功能，也推出了基于Rust语言的轻量级虚拟化项目Firecracker。两个厂商在开发的过程中，将虚拟化软件栈所需的基础组件进行解耦化设计，却发现了很多重复的通用组件，为了不重复造轮子，成立了Rust-VMM开源社区，用于管理所有通用的基础组件，便于构建自定义的Hypervisor。英特尔公司主导的Cloud Hypervisor项目也是基于Rust-VMM来实现对标准虚拟化的支持。

> **StratoVirt项目同样也是基于Rust-VMM开发的，旨在实现一套架构既能满足轻量级虚拟化场景，又能满足标准虚拟化场景的使用。**

# 3 StratoVirt架构设计

如图6-1所示，StratoVirt核心架构自顶向下分为以下三层。

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/微信截图_20231202121525.png" style="zoom: 33%;" />

1. **OCI（Open Container Initiative，开放容器）兼容接口。**兼容QMP（QEMU Monitor Protocol，QEMU监控协议），具有完备的OCI兼容能力。
2. **引导加载程序 (BootLoader)。**抛弃传统的 BIOS+GRUB（GRand Unified Bootloader，多重操作系统启动管理器）启动模式，实现了更轻更快的BootLoader，并达到极限启动时延。
3. **轻量级虚拟机 (MicroVM)。**充分利用软硬件协同能力，精简化设备模型，低时延资源伸缩能力。如下图所示：

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/微信截图_20231202121717.png" style="zoom: 50%;" />

---

StratoVirt源码主要分为四个部分，可参考StratoVirt网站主页（https://gitee.com/openeuler/stratovirt中的标签v0.2.0之前代码）具体如下。

* `address_space`：地址空间模拟，实现地址堆叠等复杂地址管理模式。
* `boot_loader`：内核引导程序，实现快速加载和启动功能。
* `device_model`：仿真各类设备，可扩展、可组合。
* `machine_manager`：提供虚拟机管理接口，兼容QMP等常用协议，可扩展。

当前 StratoVirt 开源代码中实现的是轻量化虚拟机模型，是能实现运行业务负载的最小设备集合，但已经包括虚拟化三大子系统：CPU子系统、内存子系统、I/O设备子系统（包括中断控制器和各类仿真设备、例如virtio设备、串行设备等）。下面分别介绍其基本功能和特性。

## 3.1 CPU子系统

StratoVirt是一套软硬件结合的虚拟化解决方案，其运作依赖于硬件辅助虚拟化的能力（如VT-x或Kunpeng-V）。CPU子系统的实现也是紧密依赖于硬件辅助虚拟化技术（内核KVM模块），例如对于x86架构的CPU而言，硬件辅助虚拟化为CPU增加了一种新的模式，即非根模式，在该模式下，CPU执行的并不是物理机的指令，而是虚拟机的指令。这种指令执行方式消除了大部分性能开销，非常高效。但是敏感指令（如I/O指令）不能通过这种方式执行，而且还是强制将CPU退出到根模式下交给Hypervisor程序（内核态KVM模块/用户态StratoVirt）去处理，处理完再重新返回到非根模式，执行下一条虚拟机的指令。

而StratoVirt中的CPU子系统主要围绕着KVM模块中对CPU的模拟来实现，为了支持KVM模块中对CPU的模拟，CPU子系统主要负责处理退出到根模式的事件，以及在客户机操作系统内核开始运行前对vCPU的寄存器等虚拟硬件的状态进行初始化。整个CPU子系统的设计模型如图6-3所示。

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20231202121917.png" style="zoom: 33%;" />

StratoVirt为每个vCPU创建了一个独立的线程，用来处理退出到根模式的事件，包括I/O的下发、系统关机事件、系统异常事件等，这些事件的处理以及KVM对vCPU接口的run函数独占一个单独线程，用户可以自己通过对vCPU线程进行绑核等方式让虚拟机的vCPU获取物理机CPU近似百分之百的性能。

在客户机操作系统的内核运行前，对vCPU寄存器虚拟硬件状态信息的初始化则是与StratoVirt的另一个模块BootLoader相互结合，在BootLoader中实现了一种快速引导启动Linux内核镜像的方法。在这套启动流程中，BootLoader将主动完成传统BIOS对一些硬件信息的获取，将对应的硬件表保存在虚拟机内存中，同时将提供一定的寄存器设置信息，这些寄存器设置信息将传输给CPU模块，通过设置CPU结构中的寄存器值，让虚拟机CPU跳过实模式直接进入保护模式运行，这样内核就能直接从保护模式的入口开始运行，从而让StratoVirt的启动流程更轻量快速。

CPU子系统另一大职责就是管理vCPU的生命周期，包括创建(new)、使能(realize)、运行(run)、暂停(pause)、恢复(resume)和销毁(destroy)。创建和使能过程就是结构体创建和寄存器初始化的流程，运行过程即实现KVM中vCPU运作和vCPU退出事件处理的流程。同时，得益于Rust语言对线程并发和同步的精密控制，CPU子系统用一种简单的方式实现了暂停与恢复的功能。任意vCPU线程收到暂停或恢复的命令后，就会通过改变信号量的方式，将该线程vCPU的状态变化传递到所有的vCPU线程，实现整台虚拟机的暂停或恢复，流程如图6-4和图6-5所示。

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/微信截图_20231202122047.png" style="zoom:33%;" />

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/微信截图_20231202122108.png" style="zoom:33%;" />

当虚拟机的生命周期结束时，同样要从对vCPU的销毁开始实现，在StratoVirt中，vCPU销毁分为两种情况。

(1)客户机内部正常关机：将通过对VM-Exit事件中的客户机关机(GUEST_SHUTDOWN)事件进行捕获，执行销毁并传递到所有的vCPU。(2)通过外部的QMP接口执行销毁：接收到QMP下发的命令后，将遍历每一个vCPU，依次执行销毁函数。

两种方式最终都会调用每个vCPU实例的destroy函数，让vCPU发生从Running到Stopping的状态转换，同步所有的vCPU状态后，再进入Stopped状态，完成关机流程。正常关机后，所有的vCPU都会处于Stopped状态，非此状态的生命周期结束则是异常关机，将进入错误处理流程。

StratoVirt的CPU模型较为精简，许多CPU特性以及硬件信息都将直接透传到虚拟机中，之后将在现有架构的基础上实现更多的高级CPU特性。

## 3.2 内存子系统

StratoVirt进程运行在用户态，StratoVirt会完成虚拟机启动之前的准备工作，包括虚拟机内存初始化、CPU寄存器初始化、设备初始化等。其中，内存初始化工作和虚拟机的地址空间管理都是由StratoVirt的内存子系统完成的。

### 相关概念

(1)地址空间(AddressSpace)：是地址空间模块的管理结构，负责整个虚拟机的物理地址空间管理。

(2)内存区域(Region)：代表一段地址区间，根据这段地址区间的使用者，可以分为表6-1中的类型。

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202312051754378.png" style="zoom: 50%;" />

(3)平坦地址空间(FlatRange)：如图6-6所示，是根据树状拓扑结构中内存区域的地址范围和优先级(priority)属性形成的线性视图。

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202312051755472.png" alt="image-20231205175530081" style="zoom: 50%;" />

在树状拓扑结构中，每个内存区域都会对应一个优先级属性，如果低优先级和高优先级的内存区域占用的地址区间重叠，则低优先级的内存区域的重叠部分将会被覆盖，即在平坦视图中不可见，具体体现为在平坦视图中没有对应的平坦地址空间。

(4)平坦视图(FlatView)：其中包含多个平坦地址空间。在通过地址空间管理结构地址空间访问设备或者内存时，使用平坦视图可以找到对应的平坦地址空间，平坦地址空间中有指向其所属内存区域的指针。

StratoVirt地址空间模块的设计采用树状结构和平坦视图结合的方案。通过树状结构可以快速了解到各个内存区域之间的拓扑结构关系，这种分层、分类的设计，可以管理并区分存储器域与PCI总线域的地址管理，并形成与PCI设备树相呼应的树状管理结构。平坦视图则是根据这些内存区域的地址范围和优先级属性形成的线性视图，在通过地址空间管理结构内存地址空间访问设备或者内存时，使用平坦视图可以更加方便快捷地找到对应的内存区域。

树状拓扑结构的更新很大可能会带来平坦视图的更新，一些设备或者模块需要获取最新的平坦视图并执行一些相应的操作。例如vhost设备，需要将平坦视图中的全部内存信息同步到内核vhost模块，以便通过共享内存的方式完成消息通知的流程。另外，也需要将已经分配并映射好的GPA和HVA信息注册到KVM模块，这样可以借助硬件辅助虚拟化加速内存访问的性能。基于以上需求，引入了地址空间监听函数链表，该链表允许其他模块添加一个自定义的回调函数，被注册到该链表中的函数将在平坦视图更新后被依次调用，这样即可方便地完成信息同步。

### 虚拟机物理内存初始化

StratoVirt作为用户态虚拟机监控器，实际是运行在宿主机上的用户态进程。在StratoVirt进程的虚拟地址空间中，存在多段StratoVirt进程本身使用的地址区间，该地址区间是宿主机上的VMA（Virtual Memory Area，虚拟内存区间）。

在虚拟机启动前，StratoVirt会初始化一段内存给虚拟机使用，这段虚拟内存区间也是StratoVirt进程的虚拟地址空间中的一段VMA。StratoVirt使用mmap系统调用来实现虚拟机内存资源的分配，得到的内存映射关系如图6-7所示。

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202312051756519.png" style="zoom:50%;" />

mmap映射有两种方式：匿名映射和文件映射。文件映射是将一个文件映射到真实的内存中，通过读写内存的方式直接读写文件。文件映射需要建立FileBackend结构体，包括文件std：：fs：：File和文件内偏移量两个成员。匿名映射则不需要文件，直接分配宿主机上的一段虚拟内存给虚拟机使用。StratoVirt现在支持匿名映射和文件映射两种方式，并利用文件映射的机制实现了大页特性。

前面提到内存映射的信息需要同步到KVM，在StratoVirt内存子系统中同步的方式为：在内存地址空间初始化时，添加默认的监听回调函数KvmMemoryListener。KvmMemoryListener的功能为：当图6-6中的树状拓扑发生变化从而引起平坦视图变化时，KvmMemoryListener中的回调函数被调用，将新增或者删除的内存映射信息同步到KVM。

## 3.3 I/O子系统

I/O子系统负责前端虚拟机内部和后端主机交互，如果把虚拟机当作一个黑盒，那么I/O子系统就是这个黑盒和外界连通的管道。一台物理主机通常会包含基本的I/O设备，如保存数据的磁盘、与外部进行通信的网卡、进行登录操作的串口设备等，这些都是主机与外部进行交互的必要设备。与物理机类似，虚拟机要实现基本的交互功能，也需要实现块设备、网络设备、串口设备等。虚拟这些设备是虚拟化技术I/O子系统的职责，称为I/O虚拟化。

StratoVirt作为一款轻量级的虚拟化软件，也实现了基本的设备交互功能。这些设备除了磁盘(virtio-block)、网卡(virtio-net)、串口(serial)之外，还有用作特定用途的virtio-vsock、virtio-console等设备。按照之前所述的I/O虚拟化方式，分为完全设备模拟、半虚拟化模拟、设备直通和单根I/O虚拟化，StratoVirt采用的是完全设备模拟和半虚拟化模拟的方式虚拟出各个I/O设备。完全设备模拟主要用在串行设备的模拟上，半虚拟化模拟用在磁盘、网卡等设备的模拟上，下文就以串口和磁盘为例分别介绍这两种方式。

### 完全设备模拟: 串口

串口设备主要是管理员或操作人员与虚拟机进行交互的手段，如登录到虚拟机内部执行命令等。StratoVirt实现了对UART（Universal Asynchronous Receiver Transmitter，通用异步收发传输器）16550A串口设备的模拟，该设备在计算机发展历史上出现较早，设备功能比较简单。在物理主机场景，由CPU访问16550A的寄存器实现OS和外界的串口通信，这些寄存器被映射到特定的CPU地址空间，如x86架构下，串口设备被映射到0x3f8起始的8字节的PIO地址空间。16550A设备的寄存器如表6-2所示。

![](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20231202123120.png)

在虚拟化场景中，I/O虚拟化技术负责完全模拟相应PIO端口的寄存器访问，这种模拟方法不需要修改任何前端客户机操作系统代码，虚拟机无感知，所以称为完全设备模拟方式。在6.4.6节的实验中会详细介绍16550A各寄存器的作用，以及StratoVirt如何通过软件模拟该设备。

### 半虚拟化模拟: 磁盘

磁盘设备的作用是存放数据，这个数据可以是OS必需的镜像文件，也可以是用户数据，如物理主机上的SATA盘。StratoVirt模拟的是virtio-block设备，它是一种基于virtio前后端协议的块设备，在虚拟机内部呈现的是vda、vdb等盘符，虚拟机内部可以格式化、读写这些设备，就像在物理主机上操作一样。与物理主机使用的设备驱动不同（如SATA盘驱动），StratoVirt模拟的磁盘设备基于virtio驱动，这种驱动是专门针对I/O虚拟化设计的，所以要求虚拟机内部安装virtio驱动，这种行为会造成虚拟机感知到自己处于虚拟化环境，所以是种半虚拟化的设备模拟方式。virtio驱动是对virtio协议的实现，该协议的作用是将前端的I/O请求发送到后端用户态Hypervisor，即StratoVirt，然后由StratoVirt代替前端做真正的I/O执行，这样就可以控制前端的I/O行为，避免造成逃逸等安全问题。

virtio协议是I/O虚拟化技术中常用的一种协议，本质上是种无锁的前后端数据共享方案，它实现了非常高效的前后端数据传递，当前主流的几种虚拟化软件（如QEMU）都支持virito协议，常见的设备包含virito-block、virito-net、virtio-gpu等。与上面介绍的基于PCI协议的virito不同，StratoVirt基于MMIO协议，MMIO协议定义了设备寄存器的MMIO地址。virtio-mmio部分寄存器如表6-3所示。

![](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20231202123220.png)

StratoVirt实现了最基本的基于MMIO协议的virtio-block设备，相对于标准的PCI协议，MMIO协议具有启动速度快的特点，符合轻量化的使用场景。在实现中，虚拟出一个MMIO总线，所有设备都挂在MMIO总线下。当前端CPU线程访问设备时，经过MMIO总线的读写函数，再到具体设备(virtio-block)的读写函数，然后在virtio-block中模拟各个寄存器的访问。virito协议规定，前端驱动准备就绪后，会往0x70寄存器写入DRIVER_OK标记，当StratoVirt收到该标记时会做磁盘的初始化动作，比如监听前端事件、异步I/O完成事件等。当前端有磁盘I/O需要下发时，会通知到后端，后端将这些I/O请求下发到异步I/O模块(aio)，aio封装请求再通过系统调用接口(io_submit)下发到主机内核。当异步I/O完成后，主机内核会通知StratoVirt给虚拟机内部发送中断，一次完整的I/O流程就完成了。这样就完成了磁盘设备的模拟。

# 4 从零开始构建StratoVirt

## 4.1 总体介绍

从现在开始，将通过一系列的动手实践构建一个精简版StratoVirt，其架构如图6-8所示。

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202312051447793.png" style="zoom:50%;" />

精简虚拟化实践旨在使用 Rust 语言指导零基础开发一个基本功能完备的**用户态虚拟机监控软件，**基于 Linux 提供的硬件辅助虚拟化能力，实现在多平台（x86和鲲鹏平台）的运行。预期的实现结果为**用户可以通过串口登入虚拟机内部并执行基本的Linux命令。**

精简虚拟机实践主要包括KVM模型实现、内存模型实现、CPU模型实现、BootLoader实现以及串口设备实现。下面涉及的代码存放在openEuler社区 StratoVirt 仓库中，对应的分支为 `mini_stratovirt_edu`，该分支的各个提交分别对应 `4.2~4.8` 小节中的内容。

## 4.2 KVM模型

Linux 提供的 KVM 模块可以充分利用硬件辅助虚拟化的特性，并提供了CPU、内存和中断的虚拟化支持。构建一个完整的虚拟机需要 CPU 模型构建、设备模拟等。为简单起见，本节将借助KVM API，构建一个最小化且可运行的虚拟机，运行一段汇编代码。具体流程为：**创建虚拟机和vCPU线程，提供一段汇编指令让vCPU线程执行，并捕获和处理虚拟机退出事件。**

在开始前，先用 Rust 的构建系统和包管理器 Cargo 来创建一个新项目 — StratoVirt-mini：`cargo new StratoVirt-mini`，这行代码新建了一个名为 `StratoVirt-mini` 的目录，该目录名也作为项目的名字。进入目录，可以看见 Cargo 生成了两个文件和一个目录：一个 Cargo.toml 文件，一个 src 目录用来存放代码，当前仅有一个 main.rs 文件，它还在项目目录下初始化了一个 git 仓库，便于管理代码。

### step1: 定义汇编指令

最小化模型暂时并不运行一个操作系统内核，而是提供一段指令给vCPU执行，执行完毕后，vCPU退出。提供给vCPU执行的汇编指令定义在如下`src/main.rs` 中。

`StratoVirt-mini_stratovirt_edu/src/main.rs`

![](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20231202131951.png)

这段代码的逻辑为：

1. 将 0x3f8 存在寄存器 `DX` 中；
2. 将 `AL` 寄存器和 `BL` 寄存器中的值相加，输出到 0x3f8 端口；
3. 并将换行符 `\n` 输出到 0x3f8 端口，最后执行 HLT 指令停止虚拟机的运行。

### step2: 打开KVM模块并创建虚拟机

在打开 KVM 模块之前，需要引入 Rust 的第三方库 `kvm-bindings` 和 `kvm-ioctls`：

* 其中 `kvm-bindings` 使用 Rust 语言对 KVM 模块使用的结构体进行封装；
* `kvm-ioctls` 库对KVM API进行封装。

引入的方法为：①在项目 Cargo.toml 的 `[dependencies]` 中描述第三方库的版本信息；②在需要使用库的文件头部处，通过 `use` 来引入需要使用的第三方库中对应的结构体或函数。代码如下：

`StratoVirt-mini_stratovirt_edu/Cargo.toml`

![image-20231202131750297](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/image-20231202131750297.png)

`StratoVirt-mini_stratovirt_edu/src/main.rs`

![image-20231202131824100](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/image-20231202131824100.png)

---

引入第三方库 `kvm-bindings` 和 `kvm-ioctls` 之后，调用 `kvm_ioctls`：KVM的构造函数打开 `/dev/kvm` 模块，该函数会返回一个kvm对象。通过调用该对象的 `create_vm` 成员函数，可以得到所创建虚拟机的句柄 `vm_fd`。代码如下：

`StratoVirt-mini_stratovirt_edu/src/main.rs`

![image-20231202131858194](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/image-20231202131858194.png)

### step3: 初始化虚拟机内存

> **那么在启动虚拟机之前，如何配置虚拟机的内存大小呢？**

这里固定分配 64 KB的内存给虚拟机使用。下面的介绍中，用 `GPA` 代表客户机物理地址，用 `HVA` 代表宿主机虚拟地址。

首先在 StratoVirt-mini 进程中使用 `mmap` 系统调用分配一段宿主机上的虚拟地址资源，并得到对应的 `HVA`。值得注意的是，Rust 语言中对 `mmap` 的系统调用需要使用第三方库 `libc`，为此在 Cargo.toml 中的 `[dependencies]` 后面添加 `libc` 第三方库以及它的版本信息libc=">=0.2.39"。

---

如下图所示，得到 `HVA` 之后，需要将 `HVA` 与 `GPA` 的映射关系，以及配置的客户机内存大小通知给KVM。其中映射关系和内存大小的信息保存在`kvm-binding` 提供的 `kvm_userspace_memory_region` 结构体中，然后通过 `step2` 中得到的虚拟机句柄 `vm_fd` 的 `set_user_memory_region`成员方法，将内存映射信息通知KVM。

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202312051510610.png" alt="image-20231202132100623" style="zoom: 33%;" />

最后将已定义的汇编机器码写入 `HVA` 起始地址，vCPU在运行时，会退出到KVM，KVM借助硬件辅助虚拟化技术建立页表，从而 vCPU 可以执行这段内存中保存的汇编指令。代码如下：

`StratoVirt-mini_stratovirt_edu/src/main.rs`

![image-20231202132244788](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/image-20231202132244788.png)

### step4: 创建并初始化虚拟机和vCPU

接下来使用虚拟机句柄 `vm_fd` 的 `create_vcpu` 成员函数来创建vCPU。通过得到的 vCPU 句柄设置通用寄存器和段寄存器。其中 `CS`（Code Segment，代码段）寄存器设置为0，`IP`（Instruction Point，指令指针）寄存器为虚拟机内存起始地址，`RAX` 寄存器设置为 2，`RBX` 寄存器设置为 3，这些寄存器的具体功能将在 4.4 节中介绍。代码如下：

`StratoVirt-mini_stratovirt_edu/src/main.rs`

![image-20231202132401822](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/image-20231202132401822.png)

### step5: 处理vCPU退出事件

在 vCPU 执行 `step1`（定义汇编指令）中定义的汇编指令时，会访问 I/O 端口 `0x3f8`，该端口资源没有被映射，因此虚拟机退出，进入KVM。KVM 同样无法处理该访问请求，进一步退出到用户态 `Hypervisor` 程序中。在收到 `VcpuExit::IoOut` 的 vCPU 退出事件时，将访问请求的信息以一定格式打印出来。代码如下：

`StratoVirt-mini_stratovirt_edu/src/main.rs`

![image-20231202132452664](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/image-20231202132452664.png)

编写完最小化虚拟机模型后，执行 `cargo run` 可以编译并运行工程，可以得到以下运行结果。

`Terminal`

![image-20231202132526371](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/image-20231202132526371.png)

根据运行结果可知，共发生了**三次**虚拟机 VM_Exit 事件：

* 前两次虚拟机退出均为访问 I/O 端口 `0x3f8`。其中，数据 `53` 为 `step5` 中设置的RAX寄存器、RBX寄存器以及数字0的ASCII码相加得到的结果；数据10为换行符对应的ASCII码。
* 最后一次虚拟机退出为 HLT 指令，虚拟机结束运行，Hypervisor 进程退出。

## 4.3 内存模型

4.2 节介绍了创建简单虚拟机的方法，并执行了一段简单的汇编指令。为将这段汇编指令保存到虚拟机内存中，分配了 64 KB的地址资源来存放这段汇编指令代码。但是，若要支持 StratoVirt 项目的进一步扩展，`src/main.rs` 文件中的内存实现存在如下很多不足：

* 4.5 节中，将删去测试使用的这段汇编代码，而启动一台标准的虚拟机。如果运行客户机内核，那么 64 KB的**虚拟机内存资源远远不够。**如果新增内存热插拔特性，则需要**新增多段内存映射关系。**
* 在 Intel Q35 芯片组的地址空间布局中，4 GB以下的一部分地址资源被固定分配给Flash、中断控制器、PCI设备等，因此内存可占用的地址资源被**分割成多个区间。**

考虑到以上限制，以及增强地址资源管理灵活性的需求，本节新增了 `memory` 子模块，并在 `src/main.rs` 头部中声明：mod memory。

`memory` 子模块中主要包含**地址资源管理、虚拟机内存管理、内存读写**等功能。

### 地址空间布局

StratoVirt 设置的客户机物理地址空间布局如下图所示：

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202312051555905.png" alt="image-20231202132847182" style="zoom: 80%;" />

StratoVirt 设置的客户机物理地址空间布局如上图所示，其中：

* 内存占用 `0~3GB`、`4GB` 以上的地址资源；
* `3~4GB` 的地址资源提供给设备、中断控制器使用；
* 内存的 `0~1MB` 空间内，固定存放启动客户机内核的相关配置内容，这部分内容将在 4.5 节详细介绍。

为支持更大的虚拟机内存规格、模拟更多的设备类型，将 StratoVirt 项目中各个组件用到的地址资源范围定义在常量中，当新增其他类型的设备时，可以在全局变量中动态添加设备使用的地址资源。上图中定义的客户机物理地址空间布局定义在 `src/memory/mod.rs` 中，其中资源类型定义在 `LayoutEntryType` 枚举结构体中，每个资源的范围定义在常量 `MEM_LAYOUT` 中。代码如下。

`StratoVirt-mini_stratovirt_edu/src/memory/mod.rs`

![image-20231202132936906](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/image-20231202132936906.png)

### 内存地址映射管理

4.2 节构造最小化KVM模型时，初始化了一段宿主机虚拟内存提供给虚拟机使用，并将内存信息注册到KVM。但是，`src/main.rs` 中的实现方式不能够满足项目的可扩展需求，例如添加一个 virtio` 设备。virtio设备在和前端驱动交互过程中遵循virtio协议，通过前后端共享内存的方式通信，后端virtio设备需要从virtqueue取出前端驱动下发的事件处理请求，其中virtqueue就存放在内存中。因此，内存管理模块不仅需要为 HVA 和 GPA映射关系建立管理结构，而且需要提供访问接口供其他模块使用。

如下图所示，管理一段内存映射的结构体定义为 `HostMemMapping`，该结构体通过 `mmap` 系统调用分配宿主机虚拟内存，并与客户机物理地址空间建立映射。映射关系会保存在 `HostMemMapping` 结构体中。在 `HostMemMapping` 结构体的析构函数中，会通过 `unmap` 系统调用释放这段宿主机虚拟内存资源。

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202312051603633.png" alt="image-20231202133326724" style="zoom:50%;" />

作为内存子模块对外的接口，`GuestMemory` 结构体保存所有的内存映射关系。例如，当设置虚拟机内存规格高于 3 GB时，内存将被分割为两部分：0~3GB和高于4GB的部分。这两部分的映射关系将分别保存在两个 `HostMemMapping` 对象中，这两个 `HostMemMapping` 对象将保存在`GuestMemory` 结构体的成员中。

`StratoVirt-mini_stratovirt_edu/src/memory/guest_memory.rs`

![image-20231202133402571](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/image-20231202133402571.png)

`GuestMemory` 提供的构造函数需要传入的参数为创建的客户机句柄、客户机内存规格，该构造函数会根据地址空间布局和内存大小来建立 HVA 和GPA 映射关系，并注册到KVM。

### 内存访问管理

内存访问接口是其他模块访问内存的方式，需要达到简单易用的目的。首先，先实现最基本的接口：

* **写接口：**将长度已知的字节流写入客户机物理内存中的指定地址处，地址在函数参数中指定。
* **读接口：**从客户机物理内存的指定地址处读出一段字节流，并保存在输入参数中。

`StratoVirt-mini_stratovirt_edu/src/memory/guest_memory.rs`

![image-20231202133519313](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/image-20231202133519313.png)

---

在 4.5 节中，BootLoader模块将使用 `read/write` 接口访问客户机内存，并将客户机内核文件保存在内存中。使用上面代码中的接口，示例代码如下：

`StratoVirt-mini_stratovirt_edu BootLoader` 对内存模块的调用：

![image-20231202133619225](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/image-20231202133619225.png)

这段代码会将内核的数据保存在一个 `Vec`（Rust语言定义的数据结构，表示数组）中，然后将 `Vec` 中的数据写入内存。这里共存在两次复制，如果文件过大，复制的时间和空间开销都不容忽视。

> ***Rust 优化设计：***
>
> 那么，内存模块的访问接口参数能否更加灵活，从而支持更多的参数类型？一个可行的方法是使用 `trait`（Rust语言中的类型，为一组方法的集合）和 `trait` 对象（trait object，Rust语言中实现了一组traits的数据对象）接口优化的实现，可以参考 StratoVirt 项目 mini_stratovirt_edu 分支中 `GuestMemory` 的 `read/write` 成员函数。
>
> 内存访问 `read/write` 接口优化后，仍存在一个问题，对于如下代码中的 `SplitVringDesc` 数据结构，如果想将类型为 `SplitVringDesc` 的数据对象写入内存，仍需要先转换为 `Vec`，再将其写入内存。读者可以思考如何优化 `GuestMemory` 的访问接口，增强易用性。针对这个问题的接口优化，可以参考 StratoVirt 项目 `mini_stratovirt_edu` 分支中 `GuestMemory` 的 `read_object/write_object` 成员函数，以及`src/helper/byte_code.rs` 子模块的实现。
>
> `StratoVirt-mini_stratovirt_edu VirtQueue` 中 Vring 的 `Descriptor` 结构体：
>
> ![image-20231202133719118](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/image-20231202133719118.png)
>
> ByteCode trait 定义在 `src/helper/byte_code` 子模块中，该trait主要实现数据结构和slice（Rust语言中的数据类型，数组）的相互转换。

### 错误处理

Rust语言使用 Result（Rust语言中的数据类型，表示函数执行结果）进行错误处理和传递，错误类型可以自定义。在 `src/memory/lib.rs` 文件中定义了 `memory` 子模块相关的错误类型。通过为 Error 枚举类型实现 `std::fmt::Display` trait，可以自定义每种错误发生时的输出信息。通过定义 `Result<T>` 的别名，在本模块或者其他模块中可以直接通过 `use crate::memory::Result` 引入并使用该 Result 类型。代码如下：

`StratoVirt-mini_stratovirt_edu/src/memory/mod.rs`

![image-20231202133819251](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/image-20231202133819251.png)

## 4.4 CPU模型

在之前的章节中学习了如何使用 `kvm_ioctls` 来调用系统的KVM接口，以完成硬件辅助虚拟化的基本功能，实践了如何使用 Rust 语言对虚拟机内存子系统进行设计和抽象。在这一节的实验中，将继续构建 StratoVirt-mini 虚拟化程序，对虚拟机的 CPU 进行进一步的设计和抽象。

在正式开始之前，先在项目的 `src` 目录下创建一个名为 `cpu` 的文件夹，文件夹下创建一个名为 `mod.rs` 的文件，本节的主要编程将在 `mod.rs`文件中进行。

### CPU基本结构的抽象

首先要先确定 CPU 子模块的功能界限。对于物理机来说，CPU的功能主要是解释计算机指令以及处理计算机软件中的数据；而在虚拟化程序中，CPU应该被抽象为两个基本功能：

* **完成对计算机指令的模拟**
* **处理一定的寄存器数据**

计算机指令模拟的部分主要在内核 KVM 模块中进行处理，在虚拟化程序的 CPU 模块中，主要负责对 VM-Exit 退出事件的处理，也就是上一节中对`vcpu_fd.run(...)` 事件的处理，这部分代码将会全部封装在CPU模块中。

同时在虚拟化程序的CPU子模块中，还需要进行一定寄存器数据的处理。为了便于操作，这些寄存器相关的数据，将被直接保存在抽象出的CPU数据结构中，可以通过KVM提供的相关接口，完成内核KVM模块中模拟的vCPU和Hypervisor中CPU结构寄存器信息的同步。

---

沿着上面的思路，可以简单抽象出 CPU 的基本数据结构，代码如下：

`StratoVirt-mini_stratovirt_edu/src/cpu/mod.rs`

![image-20231202134219134](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/image-20231202134219134.png)

虽然目前CPU中的成员还比较少，但已经基本可以把CPU数据结构中的成员分为三类，之后**对CPU结构的一切扩展都是围绕着这三类进行的：**

* **该vCPU本身的相关信息：**如该vCPU的 ID 号等；
* **与内核KVM模块进行交互的接口：**如该vCPU的 `VcpuFd`，通过这个抽象出的文件描述符，可以直接调用到内核KVM模块所提供的 vCPU 相关接口；
* **寄存器的相关信息：**如目前保存的通用寄存器和段寄存器的相关信息，可以根据运行程序的需要，对这些寄存器的信息自由进行修改。

---

#### vCPU初始化

接下来为抽象出的 CPU 结构添加成员函数。首先需要添加一个构造函数 `new` 对CPU结构进行初始化，代码如下：

`StratoVirt-mini_stratovirt_edu/src/cpu/mod.rs`

![image-20231202134353967](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/image-20231202134353967.png)

在Rust语言的习惯中，`new` 一般意味着对数据结构的直接创建，作为单纯的构造函数而不包含别的逻辑，所以直接传入已经初始化完成的 `VcpuFd` 和 `vcpu_id`。`new` 函数运行后，可以直接获得一个初始化完成的 CPU 数据结构。

此时对于该 CPU 数据结构而言，已经有了 vCPU 的唯一标识符 — `vcpu_id`，和内核中KVM模块的接口 `VcpuFd`，但是寄存器状态还都是初始值，所以还需要另一个函数来完成与KVM模块中寄存器数值的同步，代码如下。

`StratoVirt-mini_stratovirt_edu/src/cpu/mod.rs`

![image-20231202134433953](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/image-20231202134433953.png)

---

#### vCPU同步

简单将设备的生命周期中**设备正式启动前的流程**分为两个阶段：

* 第一个阶段是初始化阶段(new)，包含最基本的数据结构的创建；
* 第二个阶段是使能阶段(realize)，包含对设备状态的使能；

在CPU结构中，设备状态的使能主要包含CPU中寄存器信息的设置。在 `realize` 函数中，将获取内核KVM模块中的寄存器数值，并同步到CPU结构的寄存器中。同步完成后，在此基础上对CPU寄存器进行应用程序运行所需要的修改，以前文那段汇编程序为例，需要对通用寄存器和段寄存器中的一些值进行相关设置，代码如下。

`StratoVirt-mini_stratovirt_edu/src/main.rs`

![image-20231202134517604](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/image-20231202134517604.png)

为了成功运行写入内存的汇编代码，对寄存器进行如下设置：

* 对于通用寄存器 `regs` 而言：
  * 会将地址寄存器 `RIP` 设置为该汇编代码在内存中的客户机物理地址，这样，CPU就会沿着地址寄存器设置的地址开始执行指令；
  * 标志寄存器 `RFLAGS` 用于指示处理器状态并控制其操作，标志寄存器一共 32 位，其中第 2 位必须为 1，其他位均不需要设置，所以将标志寄存器设置为 2；
  * `RAX` 和 `RBX` 是两个数据寄存器，在汇编代码中将会把 `RAX` 的值和 `RBX` 的值进行加法计算，这两个寄存器设置的值将会和程序最后的输出结果直接相关，这里设置成 2 和 3，输出结果就将是 5；
* 对于段寄存器 `sregs` 而言，需要设置它的代码段寄存器，该寄存器和运行代码的寻址相关，此处用最简单的寻址方式即可，`base` 和 `selector` 均设置为0。

在设置完 CPU 实例中寄存器的数值之后，还需要将设置完成的寄存器数值同步回内核的 KVM 模块，代码如下：

`StratoVirt-mini_stratovirt_edu/src/cpu/mod.rs`

![image-20231202134606758](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/image-20231202134606758.png)

该函数将 CPU 模块中修改的寄存器值重新设置到了内核KVM模块中，这样在虚拟机vCPU正式运行之前，所有的准备都完成了。（严格来说，CPU的使能步骤应该包括：获取kvm_vcpu中的寄存器值、修改kvm_vcpu中的寄存器值和设置kvm_vcpu中的寄存器值三个过程，为了便于说明，此处分为三个步骤来完成）。

---

#### vCPU运行

下一步就能正式运行抽象出的CPU了，运行中最主要的部分还是内核KVM模块中vCPU的指令模拟和对陷出事件的处理，代码如下：

`StratoVirt-mini_stratovirt_edu/src/cpu/mod.rs`

![image-20231202134654575](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/image-20231202134654575.png)

---

将CPU模块抽象完成后，将 `kvm_vcpu_exec` 函数整合进 `main.rs` 中运行，可以得到如下和 4.2 小节末尾相同的结果：

`Terminal`

![image-20231202135242255](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/image-20231202135242255.png)

至此，成功把CPU最基本的功能封装到了CPU子模块中。

### CPU并发运行多个任务

在完成CPU基本功能的抽象后，继续对比虚拟机CPU和物理机CPU会发现，在实际的物理机中，一般不只有一个CPU，常常会有一个以上CPU的场景，这些CPU可以独立地运行程序指令，它们可以运行不同的程序，也可以运行同一个程序，利用并行计算能力加快程序运行的速度。

为了充分发挥硬件本身的计算能力，需要**添加对多CPU并行任务的支持。**在新的CPU运行模型中，CPU的指令模拟和对陷出事件的处理将不在主线程中进行，每个vCPU都会对应一个单独的线程，通过**分时复用**的方式共享物理CPU。

Rust中同样也对多线程并发编程提供了很好的支持，主要包括 `std::thread` 和 `std::sync` 两个基本模块：

* `thread` 模块中定义了**管理线程的各种函数；**
* `sync` 模块中则定义了并发编程中**常用的锁、条件变量和屏障。**

---

将 CPU 创建线程并运行指令模拟和处理陷入陷出事件的操作封装起来，代码如下：

`StratoVirt-mini_stratovirt_edu/src/cpu/mod.rs`

![image-20231202135348832](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/image-20231202135348832.png)

`StratoVirt-mini_stratovirt_edu/src/main.rs`

![image-20231202135415857](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/image-20231202135415857.png)

修改完成后尝试通过 `cargo run` 运行却失败了，发现如下报错信息。

`Terminal`

![image-20231202135459764](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/image-20231202135459764.png)

这是Rust编程中一个常见的静态生命周期检查失败的错误，原因是在创建进程时使用了一个闭包。闭包在一般情况下会按引用来捕获变量，因为添加了 `move` 关键字，会将 `&self` 的所有权转移到闭包中，而 `&self` 只是一个临时借用，无法通过生命周期检查，所以要将传入函数的参数 `&self` 改为 `self` 才能通过检查。确定了问题后，对该函数进行如下修改。

`StratoVirt-mini_stratovirt_edu/src/cpu/mod.rs`

![image-20231202135522139](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/image-20231202135522139.png)

编译通过，程序成功执行。但是这样改动也会带来一个问题：CPU实例的所有权将完全转移到自己的CPU线程中，在主线程将再也无法获取到CPU实例的所有权，无法对它进行任何查询和访问。这对之后查询和管理CPU信息是极为不利的，

例如想在CPU开始指令模拟后得到CPU对应的ID信息，直接获取CPU实例的 `id` 号后，将会得到如下报错：`value borrowed here after move`。该报错意思是该CPU实例的所有权已经被转移到了CPU线程中，在主线程中因为没有获取到该CPU的所有权，将再无法获取实例的任何信息。那么有没有一种方法可以安全地在主线程和CPU线程中共享CPU实例呢？Rust提供了一种很强大的线程安全同步机制：`Arc <T>`。

---

`Arc <T>` 意思是多线程引用计数指针，是一个线程安全的类型，允许它被传递和共享给别的线程，可以直接通过 `clone` 方法来共享所有权，此时的 `clone` 方法并不是深复制，只是简单地共享所有权的计数。下面再次对分离CPU线程的代码进行如下修改：

`StratoVirt-mini_stratovirt_edu/src/cpu/mod.rs`

![image-20231202135613094](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/image-20231202135613094.png)

`StratoVirt-mini_stratovirt_edu/src/main.rs`

![image-20231202135648962](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/image-20231202135648962.png)

同时将前面的 `reset` 步骤也加入CPU线程中来并发进行，以减少主线程中的时间损耗，提升程序运行的效率。

---

当实现了多个CPU线程同时运行的方式后，可以支持用多个CPU来执行不同的任务。我们可以尝试同时运行两段代码，每段代码由各自的CPU线程并发运行。在上一节汇编代码的基础上添加第二段汇编代码，简单地对 `0x8000` 地址进行mmio读写后，该vCPU执行 `HLT` 指令。代码如下：

`StratoVirt-mini_stratovirt_edu/src/main.rs`

![image-20231202135734292](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/image-20231202135734292.png)

修改程序让两个vCPU运行各自的汇编代码，成功运行后将得到以下输出。

`Terminal`

![image-20231202135757849](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/image-20231202135757849.png)

以上就是一个简单的虚拟机CPU模型的全部内容，介绍了CPU模块的设计思路和功能抽象，以及在StratoVirt程序中CPU线程模型的简单实现。下面将不再运行简单的汇编小程序，而是**通过对 BootLoader 模块的实现来启动一个完整的 Linux 标准内核。**

## 4.5 BootLoader实现

前文中已经实现了对CPU模型的抽象，可以通过启动更多的vCPU线程来并行执行更多的任务。那么是否可以通过它来运行更复杂的程序，而不只是简单的汇编代码，比如一个完整的Linux内核？下面将逐步构建启动引导模块BootLoader，直到能启动一个完整的Linux内核。

### 内核文件读入内存

有了之前章节中运行汇编代码的经验，可以简单总结出通过KVM模块模拟出的vCPU和虚拟机内存来运行代码的方法：

1. **将要运行的代码读进虚拟机内存；**
2. **设置vCPU中的相关寄存器支持代码运行。**

首先获取一个标准PE格式的Linux内核镜像 `vmlinux.bin`，通过Rust中的文件操作打开内核镜像并读入内存中。代码如下：

`StratoVirt-mini_stratovirt_edu/src/main.rs`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202312051757765.png" alt="image-20231202181502124" style="zoom: 33%;" />

> Rust标准库中有对文件操作的封装，封装了打开、关闭、获取文件元数据信息等一系列文件操作，通过引入 `std::fs` 进行使用。

上面这段代码中使用了 `fs` 的两个函数，分别是打开内核镜像文件和获取镜像的大小，通过 `fs::File::open` 函数打开的文件为 File 类型，该类型默认直接实现了标准库中的 `Read` trait，所以可以直接被虚拟机内存模块的GuestMemory 的 `write` 函数调用写入虚拟机内存中，通过调用该函数可以把整个内核镜像写入虚拟机内存中`0x1000` 处，并将 vCPU0 通用寄存器中地址寄存器的值改为内核代码的起始地址 `0x1000`，开始运行。

运行后却发现屏幕上没有任何输出，这代表内核镜像在启动后没有任何的 VM-Exit 陷出，这显然并不符合Linux内核正常启动的情形，到底是哪一步出了问题呢？

### Linux内核引导流程

要想回答这个问题，需要先简单梳理一下由 Linux 启动协议规定的内核在物理机 `Intel x86_64` 平台上的引导-启动流程：

1. 当硬件电源打开后，8086结构的CPU会自动进入实模式，此时仅能访问1MB的内存，并且会加载BIOS到0xffff0到0xfffff的位置；
2. CPU自动从0xffff0开始执行代码运行BIOS，BIOS将会执行某些硬件检测，初始化某些硬件相关的重要信息，并从物理地址0处开始初始化中断向量；
3. 在BIOS执行完成后，该区域已经填充了内核头部所需要的所有参数，常见的BIOS执行完成后实模式低地址位1MB内存布局如下表所示：

![image-20231202181646056](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/image-20231202181646056.png)

---

BIOS执行完成后就到了跳转至内核的入口点，此时内核将会执行两个函数：`go_to_protected_mode` 和`protected_mode_jump`：

* 前者会将CPU设置为保护模式，在实模式下，处理器的中断向量表总是保存在物理地址0处，但是在保护模式下，中断向量表将存储在CPU寄存器 `IDTR` 中，同时两种模式的内存寻址模式也是不同的，保护模式需要使用位于CPU中 `GDTR` 寄存器中的全局描述符表，所以在进入保护模式前还需要调用 `setup_idt` 和 `setup_gdt` 函数来安装临时中断描述符表和全局描述符表。
* 完成后调用 `protected_mode_jump` 函数正式进入保护模式，该函数将设置CPU CR0寄存器中的 `PE` 位来启用保护模式，此时最多可以处理4GB的内存。之后将会跳转到32位内核入口点 `startup_32`，正式进行Linux内核的启动流程，引导完成。

---

了解了Linux内核的基本引导流程后，就可以知道之前内核启动失败的原因了：

> **<font color='red'>StratoVirt-mini 在执行内核时缺少了引导的步骤，也就是类似于物理机启动流程中BIOS功能的模块，没有引导Linux内核在实模式下配置内核头部所需要的参数，也没有进行一些重要寄存器的设置，Linux内核当然无法启动。</font>**

接下来需要设计并实现一个 BootLoader 模块，引导标准PE格式的 Linux 内核启动，实现将内核镜像加载进内存并根据启动协议设置启动所必需的内存布局，并设置相应的CPU寄存器，以跳过实模式直接进入保护模式启动Linux内核。

在正式编码前，先在项目的 `src` 目录下创建一个名为 `boot_loader` 的文件夹作为模块目录，本节的主要编码将在该目录下进行。

首先根据Linux x86启动协议来设计 `boot_loader` 的内存布局，详见下表所示：

![image-20231202181941534](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/image-20231202181941534.png)

* 内存布局中1MB以下的部分主要还是内核启动的相关配置，对照BIOS执行后的1MB内存布局做了相当多的简化；
* 1MB以上的部分开始读入内核保护模式的入口代码；
* 在内存末尾将存入一个简易文件系统 `initrd`，作为内核启动后的内存文件系统来使用；

### 配置Zero Page

Zero Page（零页）是32位内核启动参数的一部分，用来存放内核启动的各种配置和硬件信息。零页也被称作`Boot Params`，它包含了很多配置结构体，在这些结构体中，除了配置一些精简的硬件信息外，还有两个结构体是需要特别处理的，即 `RealModeKernelHeader` 和 `E820Entry`，代码如下：

`StratoVirt-mini_stratovirt_edu/src/boot_loader/zeropage.rs`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202312051758357.png" alt="image-20231202182145747" style="zoom:33%;" />

---

`KernelHeader` 作为实模式下内核镜像的文件头，包含了许多内核的配置信息，其中有几项是需要特别设置的，如下表所示：

![image-20231202182223026](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/image-20231202182223026.png)

---

接下来还需要进行 `e820` 表的配置。在物理机上，`e820` 是一个可以**探测硬件内存分布**的硬件，BIOS一般会通过`0x15` 中断与之通信来获取硬件内存布局，在 `boot_loader` 中不需要再通过这个流程来获取布局，直接根据对`boot_loader` 内存布局中准备好的值来进行 `e820` 表的配置。

`e820` 表中的每一项都是一个 `E820Entry` 数据结构，表示一段内存空间，包含了起始地址、结束地址和类型。这里将根据整个虚拟机的内存布局来进行 `e820` 表的配置，代码如下。

`StratoVirt-mini_stratovirt_edu/src/boot_loader/zeropage.rs`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202312051758957.png" alt="image-20231202182336020" style="zoom: 50%;" />

在对 `e820` 表的配置中，要把虚拟机的内存进行分类，此处的分类较为简单，仅分为两类：

* 一类是可以作为内存使用的 `E820_RAM`；
* 还有一类是已经保留给特定功能使用的内存 `E820_RESERVED`；

低地址段的内存布局和前面的内存布局保持一致，高地址段的内存要注意的地方是在 x86_64 架构虚拟机的内存布局中，一般会存在一个内存空洞，在配置 `e820` 表时，需要根据虚拟机的内存布局跳过内存空洞进行配置。

---

完成对零页的配置后，将调用GuestMemory的 `write_object` 接口将 `BootParams` 数据结构整个写入内存中。为了能成功地将这些数据结构写入内存，需要对上述所有的数据结构都实现 ByteCode Trait，写入地址为预设的`0x7000`。

### 配置MPtable

Linux内核通过零页获取到各种硬件、内存信息和配置项之后，还需要通过某种方式来**获取处理器和中断控制器的信息。**目前共有两种用来获取处理器信息的方式：

* 是Intel x86平台的MP Spec（MultiProcessor Specification，多处理器技术规范）约定的方式；
* 另一种是ACPI的MADT表（Multiple APIC Description Table，多个高级可编程中断控制器描述表）约定的方式。

> 这里选择比较容易实现的MP Spec的方式。

MP Spec的核心数据结构主要包含两个部分：MPF（MP Floating Pointer，多处理器浮点数指针）和MP Table（多处理器结构表），如图6-12所示。

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202312051726387.png" alt="image-20231202182522778" style="zoom: 33%;" />

---

例如可以按如下代码直接初始化MPF：

`StratoVirt-mini_stratovirt_edu/src/boot_loader/mptable.rs`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202312051759050.png" alt="image-20231202182553900" style="zoom:33%;" />

其中关键字 MP 是固定的，用来搜索 MP 结构表，length表示整个结构的长度，以16字节为单位。spec规定版本号固定为1.4版本，除了传入的pointer外，其余项均置为0。成员pointer将直接指向存放MP Table的内存地址。

---

MP Table结构用来真实反映处理器的硬件信息，它由一个Header（表头）和若干个Entry组成，表头信息内容如下表所示：

![image-20231202182622342](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/image-20231202182622342.png)

这一步仅输入基本的硬件厂商相关信息［如OEM_ID（Original Equipment Manufacturer，原始设备制造商ID）、PRODUCT_ID（产品ID）］，支持自定义，以及固定的SPEC版本号、SIGNATURE字符。`BASE_TABLE_LENGTH` 以及 `CHECKSUM` 两项将在MP Table整个处理完毕后填入，其余项在 BootLoader 中不用实现，置为0即可。

紧跟着MP Table表头，是一串Entry结构，每个Entry都代表一个单独的与处理器相关的硬件信息，每种硬件类型都有其特定的Entry结构，它们对应的Entry Type号和长度见下表。

![image-20231202182654153](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/image-20231202182654153.png)

每个种类的 Entry 都有各自的数据结构，这里给出一份它们的Rust实现，代码如下：

`StratoVirt-mini_stratovirt_edu/src/boot_loader/mptable.rs`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202312051759509.png" alt="image-20231202182804842" style="zoom: 50%;" />

---

为了成功启动Linux内核，这 5 种Entry都是必需的，对于每个CPU，都必须写入一次 ProcessEntry，代码如下。

`StratoVirt-mini_stratovirt_edu/src/boot_loader/mptable.rs`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202312051800363.png" alt="image-20231202182853625" style="zoom:33%;" />

其余Entry的配置方法大同小异，此处不再额外说明，读者自行查阅相关SPEC手册和示例代码来了解各项数值的配置。

将全部的MP浮点指针 (MP Floating Pointer) 和MP表 (MP Table) 都写入内存地址 `0x0009fc00` 后，`mptable` 部分的配置全部完成。**此时内核已经能获取启动所必需的硬件信息、配置信息和处理器相关信息。**

### 配置Initrd、Kernel和内核命令行

相关配置全部完成后，还需要将一些启动所必需的资源读入内存，这里**主要有三种资源需要被读入内存：**

* `Initrd`：用来作为内存文件系统，读入内存位置在内存末端；
* `Kernel`：PE格式的Linux内核镜像 `vmlinux.bin` 可以全部读入内存中，读入内存位置为 `0x00100000`；
* 内核命令行：Kernel启动额外的配置项作为输入，以字符串的形式传入，读入内存位置为 `0x00020000`。

内核命令行作为内核启动最后的入参，用来控制一些较为上层的行为，如内核的启动盘、输出设备等，这里给出一个较为简易的支持 `Initrd` 启动的配置：

```shell
console=ttyS0  panic=1  reboot=k  root=/dev/ram  rdinit=/bin/sh
```

其中：

* `console` 用来标识该内核的输出设备；
* `panic` 和 `reboot` 定义了内核处理panic和reboot行为的模式；
* `root` 定义了内核的启动盘，这里使用内存文件系统来启动；
* `rdinit` 定义了内核启动完成后执行的第一个进程的路径。

> 注：
>
> 在x86架构下，0x3f8和ttys0都与串口通信相关。
>
> 1. `0x3f8`：
>    - 0x3f8 是一个十六进制数，表示计算机中串口UART（通用异步收发器）的基地址；
>    - 它是串口通信的I/O端口地址，用于与串口进行数据交换；
>    - 0x3f8对应的是第一个串口（COM1）。
>
> 2. `ttys0`：
>    - ttys0 是Linux操作系统中对应于第一个串口（COM1）的设备文件名；
>    - 在Linux系统中，串口设备通常以 `ttys*`  的形式命名，其中 `*` 代表一个数字（如ttys0、ttys1等）；
>    - ttys0 对应的是第一个串口（COM1）设备文件。
>
> 两者的关系：
> - 0x3f8是硬件层面上的串口I/O端口地址，用于访问串口硬件进行数据收发；
> - ttys0是Linux操作系统中的设备文件名，用于通过文件系统接口访问串口设备；
> - 在Linux系统中，可以通过打开和操作ttys0设备文件来进行对串口的读写操作，实现串口通信。其中，ttys0设备文件与0x3f8对应的串口硬件相连接。
>
> 综上，0x3f8和ttys0可以看作是硬件和软件之间的接口，用于在x86架构下实现与串口相关的数据通信。
>
> ---
>
> 在x86架构下，对0x3f8发送字符会将字符发送到串口设备（如COM1）而不是直接显示在终端设备上。
>
> 终端设备通常是指计算机上的显示器和键盘。串口设备是一种通过串行通信接口进行数据传输的设备，一般用于与外部设备进行数据交换，比如串口连接的终端设备（如终端机或串口控制台）。
>
> 当向0x3f8写入字符时，字符会被发送到串口设备的发送缓冲区中。然后，通过串口的物理连接（如串口线）将数据发送到终端设备（如终端机或串口控制台）。在终端设备上，接收器将串行数据流转换为并行数据，并根据数据的ASCII码值将字符显示在终端上。
>
> 所以，通过向0x3f8发送字符，可以将字符发送到串口设备，然后通过串口连接的终端设备上显示出来。请注意，确保终端设备的设置正确，并且串口通信的参数（如波特率、数据位、奇偶校验位等）与终端设备的设置匹配，以确保正确的数据传输和显示。

### 进入保护模式

在完成所有的配置后，基本已经做完了一遍内核启动流程中实模式所做的工作，接下来就是让CPU从实模式进入保护模式的步骤了。

在实模式和保护模式下，CPU寻址方式是不一样的：

* **在实模式下，**内存被划分为不同的段，每个段的大小为64KB，这样的段地址可以用16位来标识，内存段的处理通过和段寄存器关联的内部机制来进行，即将段寄存器本身的值作为物理地址的一部分，此时：**物理地址 = 左移4位的段地址 + 偏移地址** ；

* **而在保护模式下，**为了能控制更多的内存，内存段被一系列称为描述符表的结构所定义，段寄存器由直接存储地址变为了存储指向这些表的指针。这些描述符表中，最重要就是 `GDT`（Global Descriptor Table，全局描述符表）。`GDT` 是一个段描述符数组，包含了所有应用程序都可以使用的基本描述符。`GDT` 必须是存在且唯一的，它的初始化一般是在实模式中由内核来完成的。想要跳过实模式到保护模式，必须完成对`GDT` 的配置。

---

`GDT` 表的每一项由属性（flags，12位）、基地址（base，32位）和段界限（limit，20位）组成，共64位，可以通过64位数把它表示出来，相关代码如下：

`StratoVirt-mini_stratovirt_edu/src/boot_loader/gdt.rs`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202312051800417.png" alt="image-20231202183148177" style="zoom:33%;" />

在整个虚拟机内存中，GDT 表只有一张，并且可以存放在内存的任何位置。但是对CPU来说，它必须知道 GDT 表的位置，因此CPU中存在一个特定寄存器 `GDTR` 用来存放GDT的入口位置。之后在CPU进入保护模式后，它就能根据该寄存器中的值访问 `GDT`。根据对内存布局的规划，这里将GDT 表存放在 `0x520` 处，将用于中断的 IDT 表存放在 `0x500` 处。但此时 IDT 表的内容并不重要，可以将 64 位全部置 0。

---

设置完GDT表后，需要根据GDT表的入口地址和内容来设置CPU的寄存器，这里主要设置 `GDTR` 寄存器的信息，相关代码如下：

`StratoVirt-mini_stratovirt_edu/src/boot_loader/gdt.rs`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202312051801107.png" alt="image-20231202183236883" style="zoom:33%;" />

根据 GDT 表的内容生成 `GDTR` 寄存器中代码段和数据段的内容，这些寄存器段信息包装好后将传递到CPU模块中，完成相关寄存器的设置。除了`GDTR` 寄存器外，针对内存布局，还有一些额外的信息需要传递到CPU模块中，需要将这些信息封装起来，作为 `boot_loader` 整个模块的输出，供别的模块（如CPU模块）使用。相关代码如下：

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202312051801145.png" alt="image-20231202183315814" style="zoom:33%;" />

`StratoVirt-mini_stratovirt_edu/src/boot_loader/loader.rs`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202312051802333.png" alt="image-20231202183405294" style="zoom:33%;" />

BootLoader 结构体将记录 `boot_loader` 完成的部分内存布局信息和需要的寄存器设置，并传递给 CPU 模块来完成相关寄存器的设置。

---

和普通汇编程序一样，这里也需要设置地址寄存器来定义程序的入口地址，同时将零页的存放地址告诉 `RSI` 寄存器，代码如下：

`StratoVirt-mini_stratovirt_edu/src/cpu/mod.rs`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202312051802734.png" alt="image-20231202183701139" style="zoom:33%;" />

为了成功进入保护模式，需要设置段寄存器的相关值，将GDT表的信息传递给 `GDTR` 寄存器和代码段寄存器，让保护模式下的 vCPU 能正确寻址，相关代码如下：

`StratoVirt-mini_stratovirt_edu/src/cpu/mod.rs`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202312051802699.png" alt="image-20231202183736533" style="zoom:33%;" />

---

此时 CPU 已经可以正常进入保护模式了，并设置好了低地址段的内存布局来引导内核正常运行。在正式开始运行内核前，还需要做两件事：

* **初始化更多的 vCPU 寄存器信息：**除了通用寄存器和段寄存器外，为了成功运行Linux内核，还需要初始化其他CPU寄存器，如 `fpu`、`mp_state`、`lapic`、`msr` 寄存器以及用来呈现虚拟机CPU特性的 `cpuid`，这部分代码为 CPU 本身硬件信息的初始化被封装在`src/cpu/register.rs` 中，可以直接使用。

* **通过 0x3f8 端口获取内核的输出：**在内核启动参数中设置了 `console=ttyS0`，此时内核在启动后，会通过 `ISA_SERIAL` 标准串口进行输出，该标准串口会在虚拟机退出时通过 0x3f8 串口输出字符信息，可以通过修改 IO_OUT 退出事件的处理函数来捕获这些信息，相关代码如下：

`StratoVirt-mini_stratovirt_edu/src/cpu/mod.rs`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202312051802431.png" alt="image-20231202183909778" style="zoom:33%;" />

将PE格式内核镜像 `vmlinux.bin` 和 `initrd` 文件放在代码中指定的地址，运行程序启动内核镜像，输出如下：

`Terminal`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202312051803124.png" alt="image-20231202183937355" style="zoom:33%;" />

可以发现内核成功启动，顺利进入 initrd 中 `init` 的步骤，但是却无法输入任何命令。这是因为对于内核来说，虚拟机只是捕获了 `0x3f8` 端口的输出，却没有真正实现 `ISA_SERIAL` 标准串口这一设备，虚拟机中的用户态程序无法识别到这个设备，也没有办法真正输入命令。

---

以上就是一个简单的虚拟机 BootLoader 的全部内容。本小节介绍了Linux标准内核引导阶段的流程以及 BootLoader 模块的设计思路和简单实现，成功用简易的 Hypervisor 启动了一个标准Linux内核虚拟机。下面将更进一步，**完成标准输入输出设备的实现，**让虚拟机真正可用。

## 4.6 串口实现

上文已经实现了BootLoader功能，内核与文件系统已经能够正常启动，但是发现启动信息打印非常慢。原因是当前仅将串口I/O端口 `0x3f8` 的内容使用 `println` 函数输出，而没有真正实现串口设备功能。这一小节将会详细介绍串口设备的实现。

---

首先介绍串口设备使用的所有寄存器，如下表所示。

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202312051803497.png" alt="image-20231202184108678" style="zoom:33%;" />

串口设备的寄存器主要用于初始化协商和串口设备收发数据处理，下面介绍**串口设备的初始化过程：**

1. 首先，创建虚拟机内部通知后端的事件，用于通知虚拟机将内部数据输出到标准输出接口上；
2. 其次向 KVM 注册串口设备中断号，中断用于通知虚拟机内部已有接收数据需要处理；
3. 然后初始化串口寄存器，例如 `LCR` 寄存器设置数据长度为8位，`LSR` 寄存器设置发送数据寄存器为空且线路空闲，`MSR` 寄存器设置发送处理就绪、检测电话响起以及检测已连通，波特率设置为9600；
4. 最后创建线程，监听标准输入接口（终端输入）是否有数据处理，将数据写入缓存中，通过寄存器 `RBR` 读操作，通知到虚拟机内部。

`StratoVirt-mini_stratovirt_edu/src/device/serial.rs`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202312051805404.png" alt="image-20231202184151213" style="zoom:33%;" />

---

接下来介绍**寄存器的读操作实现。**偏移量0~7的值分别对应着 `0x3f8~0x3ff` 寄存器，各寄存器介绍如下表所示：

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/image-20231202184214885.png" alt="image-20231202184214885" style="zoom:50%;" />

`StratoVirt-mini_stratovirt_edu/src/device/serial.rs`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202312051805596.png" alt="image-20231202184324815" style="zoom: 50%;" />

---

最后介绍寄存器的写操作实现。偏移量0~7的值分别对应着 `0x3f8~0x3ff` 寄存器，寄存器介绍如下表所示。

![image-20231205180701982](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202312051807047.png)

`StratoVirt-mini_stratovirt_edu/src/device/serial.rs`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202312051807994.png" alt="image-20231202184516059" style="zoom: 50%;" />

---

详细代码可切换至StratoVirt主页 `mini_stratovirt_edu` 分支中的串口实现。使用此 commit 节点进行编译，执行编译出来的stratovirt二进制，结果如下：

`Terminal`

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202312051808890.png" alt="image-20231205180757063" style="zoom:33%;" />

此时仍有一个问题，只有在输入字符的时候，串口的输出信息才会显示出来，该问题将在下一小节中进行详细分析。

## 4.7 Epoll实现

上一小节实现了串口设备，但是运行的结果却还是不如人意，只有在人工输入字符的时候，串口的输出信息才会显示出来。

通过代码逻辑分析，创建 Serial 结构体时使用了互斥 (Mutex) 锁，当前**有两类线程会同时访问 Serial 数据结构：**

* 一个是进行虚拟机内部串口数据的输入和输出处理的 vCPU 线程；
* 另一个是串行标准输入（终端输入）处理线程；

在串行标准输入处理线程中，首先会持有 Serial 数据结构的互斥锁，然后使用 `std::io::stdin().lock().read_raw()` 阻塞等待标准输入，此时 vCPU 线程无法获取互斥锁进行虚拟机内的输入和输出处理；当标准输入处理完数据后，才会释放互斥锁，vCPU 线程才能够获取互斥锁进行虚拟机内的输入和输出处理，这就导致只有在人工输入字符才有串口信息输出问题。

> **解决这个问题核心思路是 Serial 标准输入处理线程不要处于长期阻塞等待处理的状态，导致 Serial 数据结构的互斥锁无法释放给其他线程访问。这就需要引入 Epoll 机制，只有在产生标准输入事件的情况下，才需要进行标准输入的读取操作，这样就能及时让出互斥锁资源。**

下面就介绍如何简易实现 Epoll 处理，当前使用的是 Crates.io 中 `vmm_sys_util` 封装箱，它已经提供了 Epoll 安全API接口 ( `new/ctl,/wait` )。

---

#### Epoll框架简单封装

首先，创建Epoll管理结构体内容的对象，用于保存Epoll对象、监听事件和事件发生时的闭包处理（事件回调函数处理）。

> **Epoll管理结构**

`StratoVirt-mini_stratovirt_edu/src/helper/epoll.rs`

![image-20231205181018569](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202312051810634.png)

> **事件回调函数**

`StratoVirt-mini_stratovirt_edu/src/helper/epoll.rs`

![image-20231205181044240](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202312051810318.png)

> **创建Epoll管理结构体内容的对象**

`StratoVirt-mini_stratovirt_edu/src/helper/epoll.rs`

![image-20231205181105201](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202312051811241.png)

---

> **将监听的文件描述符加入Epoll**

其次，需要将监听的文件描述符加入Epoll中，传入的参数是上述代码块定义中的 `EventNotifier` 结构体，将其设置为监听事件的数据指针。代码如下：

![image-20231202185153329](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/image-20231202185153329.png)

---

> **事件监听循环**

最后，创建一个线程进行事件监听循环处理，当监听事件发生时，获取监听事件的数据指针，调用其存储的闭包函数，进行事件函数处理，以下代码块为事件监听处理函数：

`StratoVirt-mini_stratovirt_edu/src/helper/epoll.rs`

![image-20231205181138599](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202312051811690.png)

---

#### 串行设备处理集成Epoll机制

以串行设备处理为例，使用Epoll机制的如下示例代码：

`StratoVirt-mini_stratovirt_edu/src/device/serial.rs`

![image-20231202185350929](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/image-20231202185350929.png)

> 详细代码可切换至StratoVirt主页的 `mini_stratovirt_edu` 分支中的Epoll实现查看。

重新编译运行，就顺利地解决了只有在输入字符的时候，串口的输出信息才会显示出来的问题。修改完成后，启动信息很快就输出到终端上，运行结果如下：

`Terminal`

![image-20231205181227568](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202312051812654.png)

## //TODO: 4.8 鲲鹏平台支持: AArch64

6.4.1~6.4.7节实现了一个精简的VMM，并且可以在x86_64平台上成功启动客户机。本小节将通过扩展现有的模块实现，在鲲鹏服务器上启动客户机。本节扩展内容主要包括中断控制器、BootLoader、CPU模型和设备树(Device Tree)的实现。

### 中断控制器

KVM提供了AArch64平台中断控制器GIC的模拟能力，因此可以直接创建VGIC（Virtual General Interrupt Controller，虚拟通用中断控制器），并配置该KVM设备的属性。因为在持续迭代过程中，StratoVirt会逐步增加新特性，MSI不可或缺，因此中断控制器模块选择GICv3版本，并添加GICv3 ITS设备。

以下代码描述了VGIC的结构体，包含设置KVM VGIC设备属性的DeviceFd、VGIC中断分发器、VGIC中断再分发器的地址区间等。

StratoVirt-mini_stratovirt_edu/src/device/gicv3.rs

![image-20231205181318732](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202312051813786.png)

(1)创建GICv3中断控制器和ITS。通过第三方kvm-ioctls库，创建VGIC以及ITS对应的KVM设备。创建设备时，可以直接调用虚拟机句柄VmFd的成员方法，在传入的参数中指定需要创建的设备类型。代码如下。

StratoVirt-mini_stratovirt_edu/src/device/gicv3.rs

![image-20231205181339823](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202312051813896.png)

(2)属性设置与初始化。设置KVM设备属性，包含VGIC中断再分发器、ITS的地址信息以及中断数目等。中断控制器模块使用到的Group属性和Group下需要设置的属性值如下，读者可以参考VGIC v3的Linux内核文档[插图]和VGIC ITS的内核文档[插图]。

①KVM_DEV_ARM_VGIC_GRP_ADDR（Group属性）。它包括：

· KVM_VGIC_V3_ADDR_TYPE_DIST：VGIC中断分发器的基地址属性（该基地址为虚拟机内部物理地址空间的地址），必须64KB对齐。VGIC中断分发器的长度固定为64KB。· KVM_VGIC_V3_ADDR_TYPE_DIST：VGIC中断再分发器的基地址属性（该基地址为虚拟机内部物理地址空间的地址），必须64KB对齐。VGIC中断再分发器的长度固定为128KB。· KVM_DEV_ARM_VGIC_GRP_NR_IRQS：中断数目属性。该属性设置VGIC设备实例管理的总中断数目，最大值为1024。· KVM_VGIC_ITS_ADDR_TYPE：ITS的基地址属性，必须64KB对齐。ITS的长度固定为128KB。

②KVM_DEV_ARM_VGIC_GRP_CTRL（Group属性）有。它：

KVM_DEV_ARM_VGIC_CTRL_INIT：初始化VGIC设备和ITS需要设置该寄存器，请求KVM初始化VGIC设备和ITS设备。初始化设备请求，在设备属性设置完成后进行。完成初始化之后，StratoVirt中断控制器模块的初始化就全部完成。代码如下。

StratoVirt-mini_stratovirt_edu/src/device/gicv3.rs

![image-20231205181405968](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202312051814028.png)

### BootLoader

与x86_64不同，AArch64平台上的低端地址区间提供给设备使用，客户机物理内存起始位置为1GB，如图6-13所示。AArch64平台的地址空间布局同样定义在内存子模块中。

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202312051816393.png" style="zoom:50%;" />

AArch64平台的BootLoader实现较为简单，主要包括：①将虚拟机内核和initrd镜像保存到内存中；②在虚拟机内存中为设备树预留空间，大小为64KB。虚拟机内核镜像、initrd镜像和设备树在内存中的布局如6-13所示。其中，设备树和initrd镜像存放在内存的结束位置处，内核镜像存放在内存起始位置处。

将虚拟机内核镜像和initrd镜像保存到内存中后，将内核起始地址、initrd地址和设备树存放地址保存在AArch64BootLoader结构体中，相关代码如下。

StratoVirt-mini_stratovirt_edu/src/boot_loader/aarch64/mod.rs

![image-20231202194723819](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/image-20231202194723819.png)

### CPU模型

AArch64平台的CPU基本结构可以复用6.4.4节中的内容，唯一的不同是寄存器数据结构的不同，这里采用了Rust中编译宏的编译技巧，将x86_64和AArch64的寄存器信息封装在不同的结构中，并添加到CPU数据结构中的同一成员处中，通过编译宏隔开。使用#[cfg(target_arch="x86_64")]，表示仅在x86_64的编译环境中编译下一代码块的代码，#[cfg(target_arch="aarch64")]则表示在AArch64平台需要编译的代码块，这样就可以通过一套代码来支持两个平台的CPU模型。

和x86_64相同，AArch64运行内核代码前也需要设置相关vCPU寄存器的信息，其作用主要是让vCPU可以获取内核和设备树的起始地址信息，这两个值将被设置到USER_PT_REG_PC寄存器和USER_PT_REG_REGS寄存器的首位。和x86_64平台不同的是，Rust中的第三方kvm-ioctls库对AArch64平台的支持较差，只提供了最基本的获取寄存器信息和设置寄存器信息的函数。因此CPU模块对AArch64 vCPU寄存器内容进行了封装，代码如下。

StratoVirt-mini_stratovirt_edu/src/cpu/aarch64/mod.rs

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202312051815781.png" style="zoom:50%;" />

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202312051816856.png" style="zoom: 33%;" />

通过封装好的内核寄存器结构，就可以直接调用VcpuFd的set_one_reg接口获取和设置指定寄存器的值。

除了通过BootLoader得到的内核起始位置信息和设备树位置以外，CPU还需要初始化一些其他内容，用来完成CPU基本硬件状态的设置，包括MPIDR（多处理器亲和寄存器）的初始化等。读者可以通过查看StratoVirt项目mini_stratovirt_edu分支获取详细信息。

### 构建设备树

6.4.5节中提到，在x86_64平台上BootLoader通过e820表将配置的虚拟机内存信息传递给客户机内核，通过MP表将处理器和中断控制器信息传递给虚拟机内核，通过命令行传递其他一些必要的配置项。与x86_64平台不同的是，AArch64平台通过设备树将硬件信息（例如内存信息、CPU信息等）传递给客户机内核。

设备树是一种能够描述硬件信息的数据结构[插图]，该结构可以转换成字节流传递给操作系统，操作系统可以解析并获得硬件信息，进而执行一系列的初始化动作。设备树，顾名思义，为树状结构，其中存在一系列的节点，每个节点有一系列属性，并且可以存在若干子节点。

在StratoVirt项目中，设备树的构建直接通过调用lfdt C接口完成，因此在编译StratoVirt Cargo项目时，需要将用到的库加入rustc的链接选项链表中，对应的编译命令为：cargo rustc-link-args=“-lfdt”。如果仍希望通过执行cargo build命令来编译项目，可以将链接选项写到项目默认编译配置。cargo/config文件中，设置方法可参照StratoVirt项目mini_stratovirt_edu分支。

构建设备树使用到的C接口定义在src/helper/device_tree.rs中，代码如下。可以看到，这些函数为创建节点的最基本的函数，调用这些函数需要加Rust语言中的unsafe标志，为了保证设备树模块基本功能函数的安全性和封装性，需要对C接口函数进行封装，并达到以下目标：①C风格的参数类型，封装后应为Rust的数据结构；②保证封装函数的功能完整，错误处理严谨，确保在调用C接口的作用域内不会发生内存泄漏、越界访问等问题。

为构建设备树，需要用到的主要接口包括：①创建空设备树；②添加子节点；③设置节点属性值。读者可按照这些需求自行实现，可参考StratoVirt项目mini_stratovirt_edu分支中的src/helper/device_tree.rs文件。

StratoVirt-mini_stratovirt_edu/src/helper/device_tree.rs

![image-20231202195133807](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/image-20231202195133807.png)

通常，构建好的设备树数据称为DTB（Device Tree Blob，设备树容器）。与DTB对应的是DTS（Device Tree Source，设备树源），DTS是文本格式的设备树描述，更加可视化，符合阅读习惯。利用DTC（device tree compiler，设备树编译器）工具[插图]，可以方便地在DTS、DTB两种格式之间转换。

在src/helper/device_tree.rs文件中，提供了接口函数dump_dtb，用于将DTB保存到指定文件中。在src/main.rs中，将构建好的DTB数据保存在指定文件中。执行命令dtc-I dtb-O dts input.dtb-o output.dts，可以得到DTS文件。

下面代码展现了DTB中的部分节点示例，其中“/”为根节点（有且仅有一个）。除根节点之外，其他节点有且只有一个父节点。节点的命名规则为“node_name@addr”，其中“node_name”为节点名字，addr为节点reg属性对应的值。节点内部包含了若干节点属性和对应的值。作为内存节点的父节点，根节点中定义了#address-cells和#size-cells，分别代表在子节点的reg属性中地址、地址范围长度所占用的下标数目。以内存节点为例，reg属性描述了节点的地址范围，在<0x000x40000000 0x00 0x20000000>中，前两个数字代表地址，后两个数字代表地址范围长度。

StratoVirt-mini_stratovirt_edu stratovirt的部分DTS：

![image-20231202195252999](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/image-20231202195252999.png)

在src/main.rs中添加设备树的构建之后，在宿主机上将PE格式内核镜像vmlinux.bin和initrd文件放在指定目录下（src/main.rs中指定的目录为/tmp，可修改），运行程序启动内核镜像可以得到如下输出，读者可以从日志中看到内存、中断控制器、串口等的内核日志。

Terminal

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202312051816239.png" alt="image-20231202195529135" style="zoom: 50%;" />



# 5 mini_stratovirt_edu简介

## 5.1 架构设计

`mini_stratovirt_edu` 的目的在于使用 Rust 语言学习从零开始开发出尽可能精简的虚拟机软件，充分利用 Linux 内核中的 KVM 模块所提供的硬件辅助虚拟化能力，实现虚拟机在多平台（ `x86_64/aarch64` ）的运行。预期结果为用户可以通过串口登录虚拟机内部，并且可以执行基本的 Linux 命令。

`mini_stratovirt_edu` 的主要流程如下所示：

>1. **利用 KVM 模块接口创建虚拟机句柄；**
>2. **初始化虚拟机句柄对应的内存区域；**
>
>3. **初始化 vCPU；**
>
>4. **使用 BootLoader 加载内核镜像和加载 initrd 镜像，并且实例化 vCPU；**
>
>5. **创建相关的模拟设备，包括串口、GICv3；**
>
>6. **启动虚拟机的 vCPU。**

`mini_stratovirt_edu` 的实现包括了: **KVM 模型实现、内存模型实现、CPU 模型实现、BootLoader 模型实现、串口设备实现以及对 aarch64 平台的额外支持。**

### KVM模块

Linux 内核中的 KVM 模块提供了硬件辅助虚拟化的能力，并且提供了 CPU、内存和中断的虚拟化支持，其对外提供的接口为 ioctl 函数，具体细节可以参考内核社区的文档：[kernel.org/doc/Documentation/virtual/kvm/api.txt](https://www.kernel.org/doc/Documentation/virtual/kvm/api.txt)

Rust 提供了 KVM 模块的封装库，分别是 `kvm-bindings` 和 `kvm-ioctls` 。其中 `kvm-bindings` 对 KVM 模块使用的结构体进行了封装，`kvm-ioctls` 对 KVM 模块提供的 API 进行了封装。

src/main.rs 的 `main` 函数中展示了其基本用法：

1. 首先打开 `/dev/kvm` 的设备描述符 kvm ，随后利用该描述符创建虚拟机句柄， `kvm.create_vm()` ，该接口本质上是封装了 `ioctl(fd, KVM_CREATE_VM, param)` ；
2. 然后固定分配 64KB 的内存给虚拟机使用，将 HVA 与 GPA 的映射关系通过虚拟机句柄 vm_fd 的 `set_user_memory_region` 成员方法通知 KVM；
3. 接着使用虚拟机句柄 vm_fd 的 `create_vcpu` 成员函数创建 vCPU，设置 vCPU 寄存器；
4. 最后使 vCPU 执行一段汇编代码并处理 vCPU 退出事件，这样借助 `KVM API` 构建出一个最小化且可运行的虚拟机。

### 内存模块

内存模块代码位于 `src/memory` ，其提供了内存管理、内存访问等功能。该模块的关键在于**维护 HVA 与 GPA 之间的映射关系。**

`mini_stratovirt_edu` 分支中内存模块通过 `HostMemMapping` 结构体来实现该功能，该结构体利用 `mmap` 系统调用分配 Host 的虚拟内存， 并且将其与虚拟机的物理地址空间的映射关系保存下来。在 `HostMemMapping` 的析构函数中会调用 `unmap` 来释放宿主机的虚拟内存资源。

此外，内存模块通过 `GuestMemory` 结构体向其他模块提供简单易用的内存访问接口，例如 `read` 函数和 `write` 函数。

### CPU模块

CPU 模块代码位于 `src/cpu` ， CPU 虚拟化的核心在于完成**对基本计算机指令的模拟和处理寄存器数据，**主要工作如下：

* 指令模拟这一部分主要交由 KVM 模块完成，CPU 模块只需要处理相关的 VM_EXIT 陷出事件；
* vCPU 的初始化过程即调用 CPU 模块中的 `new` 函数，就是调用 KVM 模块的 `create_vcpu` 接口函数，函数返回 vCPU 的句柄，该接口实际底层封装的是 `ioctl(fd, KVM_CREATE_VCPU)` ；
* vCPU 的实例化会调用 `realize` 函数，该函数的主要内容就是根据 BootLoader 模块所给的数据信息设置相关寄存器，具体步骤分为三步：
  * 获取 vCPU 句柄中的寄存器值；
  * 修改 vCPU 句柄中的寄存器值；
  * 回写 vCPU 句柄中的寄存器值。

* 启动 CPU，即调用 CPU 模块中的 `start` 函数。该函数的主要内容就是创建一个用户态线程，线程内包含一个循环 loop ，循环之中运行 `cpu.kvm_vcpu_exec` 函数用于处理 KVM 的陷出事件。

### BootLoader模块

BootLoader 模块代码位于 `src/boot_loader` ，该模块负责**操作系统内核的启动引导，**主要需要实现三件事：

* 按照 Linux Boot Procotol 设计 `1` MB低地址虚拟机物理内存布局，根据启动所需的信息配置填充相应的数据结构；
* 将内核镜像文件和 `initrd` 写入虚拟机内存；
* 将写入内存的数据信息传递给 CPU 模块用于寄存器设置。

BootLoader 模块的工作流程如下：

1. 首先在 src/main.rs 的 `main` 函数按照不同的处理器架构选择对应的 `load_boot_source` 函数实现；
2. 随后按照所对应的启动协议进行相关配置，调用 BootLoader 模块的 `load_kernel` 函数；
3. 该函数将所需的数据结构和内核镜像文件写入虚拟机内存当中；
4. 最后将之前配置的相关数据信息返回并且作为参数传递给 vCPU 实例化函数。

### 设备模块

设备模块代码位于 `src/device` 路径下，设备对外暴露的接口一般包括 `new` 函数、`realize` 函数、`read` 函数和 `write` 函数。

BootLoader 模块可以让虚拟机完成内核的引导启动，但是想要和虚拟机交互并且让其执行基本 Linux 命令，还需要串口设备。串口设备用于输入输出。

串口设备有众多的寄存器，这些寄存器主要用于初始化协商和收发数据处理。初始化过程主要包括：

* 创建虚拟机向后端发送通知的事件
* 向 KVM 模块注册中断号用于后端向虚拟机发送中断
* 最后初始化相关寄存器

数据收发则是通过调用 `read` 函数和 `write` 函数实现。

### //TODO: AArch64架构额外支持

由于 aarch64 架构和 x86_64 架构的差异，需要对 aarch64 架构做一些额外支持，主要包括中断控制器、 BootLoader 模块、CPU 模型以及设备树 ( Device Tree )。

KVM 提供了 aarch64 平台中断控制器 GIC 的模拟能力，因此可以直接利用 KVM 接口创建 vGIC v3设备，并且配置该设备的相关属性。

aarch64 的 BootLoader 设计相对 x86_64更简单，只需要将 Device Tree 、内核镜像和 initrd 文件写入内存，并且将三者的地址信息返回并提供给 CPU 模块和 Device Tree 使用。

aarch64 平台通过 Device Tree 来向内核传递硬件信息。 Device Tree 是一种能够描述硬件信息的数据结构，该结构可以转换成字节流传递给内核。内核可以解析该字节流，获得必要的硬件信息并执行一系列的初始化动作。

对于 CPU 模块，aarch64 架构与 x86_64 架构相同，在运行内核代码前也需要设置一定的 vCPU 寄存器信息，其作用主要是让 vCPU 知道内核的起始地址和 Device Tree 的存放地址。由于 Rust kvm-ioctls 库对 aarch64 平台的支持较差，大部分 CPU 寄存器的信息在 CPU 模块中进行了手动封装。

### 总结

在以上模块的支持下，`mini_stratovirt_edu` 实现了一个极其精简的虚拟机，并且可以实现与用户的简单交互，可以执行基础的 Linux 命令，达到了预期结果。



## 5.2 使用指南

### 软硬件要求

> 最低硬件要求

* 处理器架构：目前仅支持 aarch64 和 x86_64 处理器架构。aarch64 需要 ARMv8 及更高版本且支持虚拟化扩展；x86_64 支持 VT-x；
* 2 核 CPU；
* 4 GiB 内存；
* 16 GiB 可用磁盘空间。

> 软件要求

* 操作系统: openEuler-20.03-LTS 及 openEuler 更高版本；

* rust 语言支持：安装 rust 语言以及 cargo 包管理器。

### 具体安装过程

下载 mini_stratovirt_edu 源代码，命令如下：

```shell
git clone https://gitee.com/openeuler/stratovirt.git
cd stratovirt
git checkout -b mini_stratovirt_edu remotes/origin/mini_stratovirt_edu
```

编译 mini_stratovirt_edu 源代码，具体命令如下所示：

```shell
cargo build --release
```

编译成功之后，可以在 target/release/stratovirt 路径下找到生成的二进制文件。

---

为了成功运行 mini_stratovirt_edu， 需要准备：

* PE 格式的 Linux 内核镜像

* ext4 文件系统，initrd 的文件系统镜像

> Linux 内核镜像下载地址：https://repo.openeuler.org/openEuler-21.03/stratovirt_img/
>
> initrd 文件系统镜像制作方法：[docs/mk_initrd.ch.md · openEuler/stratovirt - 码云 - 开源中国 (gitee.com)](https://gitee.com/openeuler/stratovirt/blob/master/docs/mk_initrd.ch.md)
>
> [用 QEMU/Spike+KVM 运行 RISC-V Host/Guest Linux - 泰晓科技 (tinylab.org)](https://tinylab.org/riscv-kvm-qemu-spike-linux-usage/)

可以根据处理器架构选择对应的内核镜像下载链接和使用制作方法生成的 initrd 镜像 ( `initrd.img `)，需要将两个镜像文件放置到指定路径 `/tmp/` ，整个过程的具体命令编写为脚本，如下所示：

```shell
arch=`uname -m`

if [ ${arch} = "x86_64" ]; then
	wget https://repo.openeuler.org/openEuler-21.03/stratovirt_img/x86_64/vmlinux.bin 
	mv vmlinux.bin /tmp/vmlinux.bin
elif [ ${arch} = "aarch64" ]; then
	wget https://repo.openeuler.org/openEuler-21.03/stratovirt_img/aarch64/vmlinux.bin 
	mv vmlinux.bin /tmp/vmlinux.bin
fi
```

运行该脚本之后， `/tmp/` 就多了 `vmlinux.bin` 文件；同时将制作的 `initrd.img` 也放入 `/tmp/` ，如下所示：

```shell
ls /tmp
initrd.img
vmlinux.bin
# ...
```

---

将下载下来的两个镜像文件放到指定位置之后，就可以运行 `mini_stratovirt_edu` 的二进制程序了，具体命令如下所示：

```shell
cd stratovirt
./target/release/stratovirt
```

至此，就完成了虚拟机的启动，并进入虚拟机环境中。



# 6 总结

本章主要介绍了从零开始构建虚拟化平台StratoVirt的完整流程。首先，简要介绍了StratoVirt的使用场景、技术优势以及发展背景；然后介绍了StratoVirt的架构设计，主要分为CPU子系统、内存子系统、I/O子系统三部分进行阐述；最后基于硬件虚拟化技术，使用Rust语言实现了精简版的StratoVirt虚拟化软件，通过KVM模型、内存模型、CPU模型、BootLoader实现以及串口设备实现了组成虚拟化软件的最小集，可以支持x86和鲲鹏双平台的运行。



---



# 7 //TODO：开发流程

## 7.1 在QEMU ARM系统上运行

由于 `mini_stratovirt_edu` 针对ARM系统的整体设计比较精简，先跑一下ARM架构的，构建流程如下：

1. 安装并编译 qemu-system-arm64；
2. 安装交叉编译工具链 toolchain；
3. 构建 qemu 所需的资源文件：
   * 内核镜像
   * 根文件系统
4. 在qemu arm系统中构建并运行 `mini_stratovirt_edu`。

---

[第一期自动搭建openEuler虚拟机运行QEMU运行环境](https://www.openeuler.org/zh/blog/luoyuzhe/001Auto-build-vm-enviroment/index.html)

[QEMU启动ARM64 Linux内核_qemu-system-aarch64-CSDN博客](https://blog.csdn.net/benkaoya/article/details/129509269)





## 7.2 依次测试下各commit

> 将各 commit 做成PATCH，依次测试功能

```shell
./configure --prefix=/opt/riscv64_gdb --with-arch=rv64imc --with-abi=xxx --enable-tui
make 
```







