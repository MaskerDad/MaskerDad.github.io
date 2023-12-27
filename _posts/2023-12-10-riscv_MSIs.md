---
title: riscv_MSIs

date: 2023-12-10 17:00:00 +0800

categories: [AIA]

tags: [riscv, aia]

description: 
---

# 0 参考

[RISC-V Is Getting MSIs! – Stephen Marz](https://blog.stephenmarz.com/2022/06/30/msi/)

[sgmarz/riscv_msi: Message Signaled Interrupts for RISC-V (github.com)](https://github.com/sgmarz/riscv_msi)

AIA草案手册是由约翰·豪泽（John Hauser）编写的，可以在此处找到：https://github.com/riscv/riscv-aia。



# 1 概述

消息信号中断是一种在没有专用中断请求引脚（IRQ）的情况下发出中断信号的方法。MSI在PCI总线上是最常见的用途之一，PCI规范定义了MSI和MSI-X标准。其优势包括：

* 减少设备与CPU或中断控制器之间的直接线路数量；
* 通过设计强制使用带内信号来提高信号传输性能；
* 改善虚拟化环境中的宿主/客户端信号传输。

该代码是用Rust编写的RV32I版本。最初我是为RV64GC编写的，但我写的其他所有内容也都是为RV64GC编写的，所以我认为我应该扩展和拓宽自己的视野。



# 2 MSI中断

一个MSI是由一个 “消息” 触发的，这个术语是指 “内存写入”。实际上，我们可以通过简单地解引用一个MMIO指针来触发一个消息。

```rust
// Note that 0xdeadbeef is not really a valid message.
// The AIA specifies messages 1 through 2047 are valid due to the number of registers
// available. But, the following is just an example. 
let message = 0xdeadbeef; 
// QEMU's 'virt' machine attaches the M-mode IMSIC for HART 0 to 0x2400_0000
// The AIA specifies that this must be a word, a double word will cause a trap. 
write_volatile(0x2400_0000 as *mut u32, message);
```



## 2.1 中断文件的MMIO地址

上面的代码将写入到 MMIO 地址 `0x2400_0000` ，这是QEMU的 `virt` 机器连接到 HART 0 的M模式IMSIC的位置。HART 0 的S模式IMSIC将连接到 `0x2800_0000` 。每个HART相隔一页，意味着HART 1的M模式IMSIC位于 `0x2400_1000`，而S模式IMSIC位于 `0x2800_1000`。

对于许多嵌入式系统来说，这些值可以来自规范或包含扁平设备树（FDT）的开放固件（OF）包。下面是QEMU的 `virt FDT` 的纯文本示例。对于这个代码仓，我硬编码了MMIO地址，而不是读取设备树。

```rust
imsics@24000000 {
   phandle = <0x09>;
   riscv,ipi-id = <0x01>;
   riscv,num-ids = <0xff>;
   reg = <0x00 0x24000000 0x00 0x4000>;
   interrupts-extended = <0x08 0x0b 0x06 0x0b 0x04 0x0b 0x02 0x0b>;
   msi-controller;
   interrupt-controller;
   compatible = "riscv,imsics";
};
```

标准化设备树格式的组织可以在此处找到：https://www.devicetree.org/ 

关于Linux中的设备树的更多信息可以在此处找到：https://www.kernel.org/doc/html/latest/devicetree/usage-model.html 

IMSICs最近被添加到QEMU的virt机器中，因此您可能需要克隆和构建自己的QEMU。QEMU的存储库可以在此处找到：https://github.com/qemu。



## 2.2 通过发送消息来触发一个中断

设备将一个字写入特定的MMIO地址后，中断被触发。这意味着设备不需要连接到IRQ控制器（如RISC-V的平台级中断控制器或PLIC）的线路。只要设备可以进行内存写入，它就可以触发中断。

尽管触发消息如此简单，但我们需要一种机制来启用和优先处理这些消息。在某些情况下，我们可能不想获取某些消息。这就是IMSIC的作用所在。



# 3 IMSIC

为了支持MSIs，某些设备需要能够接收内存写入并将其转换为中断。此外，该设备还需要提供一种机制来启用/禁用和优先处理中断，就像常规的中断控制器一样。这是通过传入的MSI控制器（IMSIC）设备来实现的。

> 高级中断架构（AIA）手册仍在进行中，已经进行了一些重大更改，删除或添加了CSR或其他相关信息。因此，一些代码和表可能已经过时。

IMSIC的寄存器机制由几个控制和状态寄存器（CSR）以及通过选择机制可访问的内部寄存器组成。

## 3.1 新增的寄存器

AIA 在M模式和S模式之间定义了几个新的CSR（控制和状态寄存器）。以下是AIA所定义的新寄存器：

| Register Name | Register Number | Description                                              |
| :------------ | :-------------- | :------------------------------------------------------- |
| MISELECT      | 0x350           | Machine register select                                  |
| SISELECT      | 0x150           | Supervisor register select                               |
| MIREG         | 0x351           | A R/W view of the selected register in MISELECT          |
| SIREG         | 0x151           | A R/W view of the selected register in SISELECT          |
| MTOPI         | 0xFB0           | Machine top-level interrupt                              |
| STOPI         | 0xDB0           | Supervisor top-level interrupt                           |
| MTOPEI        | 0x35C           | Machine top-level external interrupt (requires IMSIC)    |
| STOPEI        | 0x15C           | Supervisor top-level external interrupt (requires IMSIC) |

`MISELECT` 和 `MIREG` 允许我们通过将其编号写入 `MISELECT` 寄存器来选择寄存器。然后，`MIREG` 将表示所选寄存器。例如，如果我们从 `MIREG` 读取，我们将从所选寄存器读取，如果我们向 `MIREG` 写入，我们将向所选寄存器写入。

有四个可选择的寄存器。这些寄存器有M/S模式版本。例如，如果我们写入 `SISELECT` ，我们将访问相应寄存器的S模式版本。

| Register Name      | MISELECT/SISELECT | Description                           |
| :----------------- | :---------------- | :------------------------------------ |
| EIDELIVERY         | 0x70              | External Interrupt Delivery Register  |
| EITHRESHOLD        | 0x72              | External Interrupt Threshold Register |
| EIP0 through EIP63 | 0x80 through 0xBF | External Interrupt Pending Registers  |
| EIE0 through EIE63 | 0xC0 through 0xFF | External Interrupt Enable Registers   |

`MISELECT/SISELECT` 可选择的寄存器，并可通过 `MIREG/SIREG` 进行读写。

---

首先，我们需要做的第一件事是使能IMSIC本身。这是通过一个名为 “使能中断传递” 的寄存器 `EIDELIVERY` 完成的。该寄存器可以包含三个值之一：

| Value       | Description                                     |
| :---------- | :---------------------------------------------- |
| 0           | Interrupt delivery is disabled                  |
| 1           | Interrupt delivery is enabled                   |
| 0x4000_0000 | Optional interrupt delivery via a PLIC or APLIC |

## 3.2 使能IMSIC

因此，我们需要将 1（使中断传递可用）写入 `EIDELIVERY` 寄存器中：

```rust
// First, enable the interrupt file 
// 0 = disabled 
// 1 = enabled 
// 0x4000_0000 = use PLIC instead 
imsic_write(MISELECT, EIDELIVERY); 
imsic_write(MIREG, 1);
```

## 3.3 中断优先级

`EITHRESHOLD` 寄存器保存了一个中断优先级阈值，只优先级更高的中断才能被响应。例如，如果一个中断的优先级值低于 `EITHRESHOLD` 中的值（值低这说明优先级更高），它将被“响应”或解屏蔽。否则，它将被屏蔽，无法被响应。例如，`EITHRESHOLD` 为 5 只允许消息1、2、3和4被响应。注意，消息 0 被保留为 “无消息” 的意义。

> 由于较高的阈值打开了更多的消息，因此具有较低编号的消息具有较高的优先级。

```rust
// Set the interrupt threshold.
// 0 = enable all interrupts
// P = enable < P only
imsic_write(MISELECT, EITHRESHOLD);
// Only hear 1, 2, 3, and 4
imsic_write(MIREG, 5);
```

AIA规范中使用消息本身作为优先级。因此，消息1的优先级为1，而消息1、2、3、4的优先级为1、2、3、4。这更方便，因为我们可以直接控制消息。然而，由于每个消息号都有关联的启用和挂起位，因此最高编号的中断存在限制。规范的最大总消息数为 `32×64-1=2,047`（我们减去1以去除0作为有效消息）。

## 3.4 使能信号

`EIE` 寄存器将控制消息的启用或禁用。对于RV64，这些寄存器的宽度为64位，但仍占用两个相邻的寄存器编号。因此，对于RV64，只能选择偶数编号的寄存器（例如，EIE0、EIE2、EIE4，...，EIE62）。如果尝试选择奇数编号的EIE，将会触发无效指令陷阱。尽管文档中明确说明这是期望的行为，但我花了很多时间才弄清楚这一点。对于RV32，`EIE` 寄存器仅为32位，EIE0到EIE63都是可选的。

`EIE` 寄存器是一个位集。如果相应消息的位为 1，则未屏蔽且已启用，否则被屏蔽且已禁用。对于RV64，消息0到63都位于 `EIE0[63:0]` 中。位表示消息。我们可以使用以下公式确定RV64要选择的寄存器：

```rust
// Enable a message number for machine mode (RV64)
fn imsic_m_enable(which: usize) {
    let eiebyte = EIE0 + 2 * which / 64;
    let bit = which % 64;

    imsic_write(MISELECT, eiebyte);
    let reg = imsic_read(MIREG);
    imsic_write(MIREG, reg | 1 << bit);
}
```

RV32的行为基本相同，唯一不同的是我们不需要将其乘以2来进行缩放。

```rust
// Enable a message number for machine mode (RV32)
fn imsic_m_enable(which: usize) {
    let eiebyte = EIE0 + which / 32;
    let bit = which % 32;

    imsic_write(MISELECT, eiebyte);
    let reg = imsic_read(MIREG);
    imsic_write(MIREG, reg | 1 << bit);
}
```

使用上述代码，我们现在可以启用我们想要响应的消息。以下示例启用了2、4和10号消息：

```rust
imsic_m_enable(2);
imsic_m_enable(4);
imsic_m_enable(10);
```

## 3.5 挂起信号

`EIP` 寄存器的行为与 `EIE` 寄存器完全相同，只是其中的某一位为1表示特定的消息处于挂起状态，即已发送带有该消息编号的写操作到IMSIC。

`EIP` 寄存器可读写。通过从中读取，我们可以确定哪些消息处于挂起状态。通过向其写入，我们可以通过将 1 写入相应的消息位来**手动触发中断消息。**

```rust
// Trigger a message by writing to EIP for Machine mode in RV64
fn imsic_m_trigger(which: usize) {
    let eipbyte = EIP0 + 2 * which / 64;
    let bit = which % 64;

    imsic_write(MISELECT, eipbyte);
    let reg = imsic_read(MIREG);
    imsic_write(MIREG, reg | 1 << bit);
}
```

## 3.6 测试

既然我们可以启用传递以及单独的消息，我们可以通过以下两种方式触发它们：

* 将消息直接写入MMIO地址；
* 将相应消息的中断挂起位设置为1。

```rust
unsafe {
    // We are required to write only 32 bits.
    // Write the message directly to MMIO to trigger
    write_volatile(0x2400_0000 as *mut u32, 2);
}
// Set the EIP bit to trigger
imsic_m_trigger(2);
```



## 3.7 信号Traps

每当一个未屏蔽的消息发送到一个启用的IMSIC时，它将作为外部中断传递到指定的HART。对于M模式的IMSIC，这将作为异步原因 `11` 传递，而对于S模式的IMSIC，这将作为异步原因 `9` 传递。

当我们接收到由于消息传递而引发的中断时，我们需要通过从 `MTOPEI` 或 `STOPEI` 寄存器中读取来 “弹出” 顶级待处理中断，具体取决于特权模式。这将为我们提供一个值，其中位 `26:16` 包含消息编号和位 `10:0` 包含中断优先级。

> 是的，消息编号和消息优先级是相同的数字，因此我们可以选择任意一个。

```rust
// Pop the top pending message
fn imsic_m_pop() -> u32 {
    let ret: u32;
    unsafe {
        // 0x35C is the MTOPEI CSR.
        asm!("csrrw {retval}, 0x35C, zero", retval = out(reg) ret),
    }
    // Message number starts at bit 16
    ret >> 16
}
```

我的编译器不支持此规范中的CSR名称，因此我使用了CSR编号。这就是为什么你看到的是 `0x35C` 而不是 `mtopei`，但它们表示的是同样的意思。

当我们从 `MTOPEI` 寄存器（0x35C）读取时，它将给出最高优先级消息的消息编号。上面代码片段中的 `csrrw` 指令将原子地将CSR的值读入返回寄存器，然后将值零存储到CSR中。

当我们将零写入 `MTOPEI` 寄存器（0x35C）时，我们告诉IMSIC我们正在 “声明” 我们正在处理最顶层的消息，这将清除相应消息编号的 `EIP` 位。

```rust
/// Handle an IMSIC trap. Called from `trap::rust_trap`
pub fn imsic_m_handle() {
    let msgnum = imsic_m_pop();
    match msgnum {
        0 => println!("Spurious message (MTOPEI returned 0)"),
        2 => println!("First test triggered by MMIO write successful!"),
        4 => println!("Second test triggered by EIP successful!"),
        _ => println!("Unknown msi #{}", v),
    }
}
```

第 0 条消息无效，因为当我们从 `MTOPEI` 中弹出时，0表示 “无中断”。

# 4 测例输出

如果您使用新的QEMU运行仓库，成功测试后应该会看到以下内容。

![image-20231211100658059](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202312111006215.png)

# 5 结论

在本篇文章中，我们看到了为支持 RISC-V 的新寄存器添加了对即将到来的MSI控制器（IMSIC）的支持。我们还启用了IMSIC传送、单个消息，并处理了发送消息的两种方式：**通过MMIO直接发送或通过设置相应的EIP位。**最后，我们处理了来自陷阱的中断。

AIA手册的第二部分包括新的高级平台级中断控制器（APLIC）。我们将研究这个系统，并编写驱动程序来开始研究这个新的APLIC如何使用电线或消息进行信号传递。

在APLIC之后，我们将为PCI设备编写驱动程序，并使用它来发送MSI-X消息。
