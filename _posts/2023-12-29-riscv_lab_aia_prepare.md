---
title: riscv_lab_aia_preapre

date: 2023-12-29 17:00:00 +0800

categories: [riscv-lab, aia]

tags: [aia]

description: 
---

# 0 参考

[riscv-msi](https://github.com/sgmarz/riscv_msi)



# 1 流程梳理

## 1.1 从 `run.sh` 开始

对于一个Rust项目而言，打开项目文件夹后最应该关注的并不是 `src/main.rs`，而是配置文件以及编译链接脚本，尤其对于一些在裸机上直接开发的底层项目更是如此。

> Cargo配置项

包括两部分：`.cargo/config` 和 `cargo.toml`。

* `.cargo/config` 用于程序运行前后的一些配置，比如：目标平台、指定链接脚本、指定运行脚本等；

  ```rust
  [build]
  target = "riscv32i-unknown-none-elf"
  rustflags = ['-Clink-arg=-Tlds/virt.lds']
  
  [target.riscv32i-unknown-none-elf]
  runner = "./run.sh"
  ```

  * 指定 `lds/virt.lds` 代替默认的链接脚本；
  * 利用 `cargo runner` 可以在 `cargo run` 时直接启动qemu。当执行 `cargo run` 时，cargo 会首先将这个 `runner` 字段中的命令末尾附上构建出的可执行文件的路径（相当于把路径传入 runner 脚本或命令），然后执行这个命令。

* `cargo.toml` 用于提供一些程序基本信息以及运行过程中的依赖包：

  ```rust
  [package]
  name = "riscv_msi"
  version = "0.1.0"
  edition = "2021"
  
  # See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html
  
  [dependencies]
  ```

> 如何构建并启动项目： `run.sh`

```shell
#!/bin/bash

if [ $# -ne 1 -o ! -x $1 ]; then
    echo "Use cargo run instead of running this directly."
    exit 2
fi

KERNEL=$1

TRACES="pci_nvme*"

PARAMS+=" -nographic"
PARAMS+=" -machine virt,aclint=on,aia=aplic-imsic"
PARAMS+=" -cpu rv32"
PARAMS+=" -d guest_errors,unimp"
PARAMS+=" -smp 1"
PARAMS+=" -m 32M"
PARAMS+=" -serial mon:stdio"

PARAMS+=" -device pcie-root-port,id=bridge1,multifunction=off,chassis=0,slot=1,bus=pcie.0,addr=01.0"

PARAMS+=" -device pcie-root- port,id=bridge2,multifunction=off,chassis=1,slot=2,bus=pcie.0,addr=02.0"

PARAMS+=" -device qemu-xhci,bus=bridge1,id=xhci"
PARAMS+=" -device usb-tablet,id=usbtablet"
PARAMS+=" -drive if=none,format=raw,file=hdd.dsk,id=hdd1"
PARAMS+=" -device nvme-subsys,id=nvmesubsys,nqn=1234"
PARAMS+=" -device nvme,serial=deadbeef,id=nvmehdd,subsys=nvmesubsys,bus=bridge2"
PARAMS+=" -device nvme-ns,drive=hdd1,bus=nvmehdd"

T=""
for t in $TRACES; do
    T+="--trace $t "
done

exec qemu-system-riscv32 \
    ${PARAMS} \
    -bios none \
    $T \
    -kernel $KERNEL
```

* 这里没有指定bios程序，在riscv下就是SBI实现，比如 `OpenSBI/RustSBI` 等，代码后续应该在M-Mode下做了一些必要的初始化工作；

* 你或许对 `-kernel` 后面直接使用了不经任何裁减的完整ELF文件，在真实的裸机平台上这么做就会出错，因为平台定位不到文件的代码段，它只能定位到文件头，而头部是一些程序段的元数据，通常我们需要将这部分内容丢弃掉，但新版QEMU支持直接加载ELF。

  至少在 Qemu 7.0.0 版本后，我们可以直接将内核可执行文件 `os` 提交给 Qemu 而不必进行任何元数据的裁剪工作，这种情况下我们的内核也能正常运行。注意，低版本QEMU可能不支持aia模拟，建议使用最新版QEMU，此实验在QEMU-8.1.0下运行成功。

> 链接脚本：lds/virt.ld

```shell
OUTPUT_ARCH( "riscv" )
ENTRY(_start)
MEMORY
{
  ram  (rwx) : ORIGIN = 0x80000000, LENGTH = 24M
}

#...

SECTIONS
{
  PROVIDE(_memory_start = ORIGIN(ram));
  PROVIDE(_memory_end = _memory_start + LENGTH(ram));

  .text : {
    PROVIDE(_text_start = .);
    *(.text.init) *(.text .text.*)
    PROVIDE(_text_end = .);
  } >ram AT>ram :text

  . = ALIGN(8);
  PROVIDE(__global_pointer$ = .);

  #...
  
  PROVIDE(_stack_start = .);
  PROVIDE(_stack_end = _stack_start + 8K);
  PROVIDE(_heap_start = _stack_end);
  PROVIDE(_heap_end = _memory_end);
}
```

* `__global_pointer` 指出了 `.text` 段与其他段的界限，具体用来做什么目前不清楚；
* 指定输出目标架构 `riscv` 以及程序入口地址为 `_start`；
* 定义了地址空间 `ram`，起始地址为 `0x80000000`，大小为 `24M`；
* SECTIONS中定义了各程序段的布局以及链接规则，包括：`.text/.data/.rodata/.bss`：
  * 定义全局变量 `_memory_start` 和 `_memory_end` 分别表示物理地址空间的边界；
  * `_text_start/_text_end` 表示 `.text` 段的边界；
  * 栈空间的边界用 `_stack_start/_stack_end` 表示，由高地址向低地址延伸，大小为8K；
  * 堆空间的边界用 `_heap_start/_heap_end` 表示，大小为 `_memory_end - _stack_end`；

---

还需要了解一下 `qemu-virt` 的启动流程：

1. 将必要的文件载入到 Qemu 物理内存之后，Qemu CPU 的程序计数器（PC, Program Counter）会被初始化为 `0x1000` ，因此 Qemu 实际执行的第一条指令位于物理地址 `0x1000` ；
2. 接下来它将执行寥寥数条指令并跳转到物理地址 `0x80000000` 对应的指令处并进入第二阶段。从后面的调试过程可以看出，该地址 `0x80000000` 被固化在 Qemu 中，作为 Qemu 的使用者，我们在不触及 Qemu 源代码的情况下无法进行更改。

这也是为什么在链接脚本 `lds/virt.lds` 中指定物理地址空间 `ram` 的起始地址为 `0x80000000` 的原因。

接下来，进入 `_start` 函数并进行一些必要的初始化工作。

## 1.2 进入 `_start` 函数

```assembly
.section .text.init

.global _start
_start:
.option norelax
    la      sp, _stack_end
    la      gp, __global_pointer$

    csrr    a0, mhartid

    # Allocate 2^13 = 8K of stack space
    slli    t0, a0, 13
    sub     sp, sp, t0

    # Set the trap vector to trap (defined in trap.S)
    la      t0, trap
    csrw    mtvec, t0

    # Jump to main after mret
    la      t0, main
    csrw    mepc, t0
    # 3 << 11 is Mode 3 in MPP (Machine Mode)
    # 1 << 7  is MPIE to turn on interrupts
    li      t0, (3 << 11) | (1 << 7)
    csrw    mstatus, t0

    # 1 << 11 is MEIE to enable external interrupts (Machine)
    # 1 << 9  is SEIE to enable external interrupts (Supervisor)
    li      t0, (1 << 11) | (1 << 9)
    csrw    mie, t0

    csrw    mideleg, 0

    # When main returns, we want to park the HART
    la      ra, park
    mret
.type _start, function
.size _start, . - _start

park:
    wfi
    j       park
.type park, function
.size park, . - park
```

`_start` 函数的注释已经非常清楚，其中：

* 将栈空间的最高地址 `_stack_end` 保存至 `sp`，`__global_pinter$` 保存至 `gp` 中，其中这个 `gp` 是全局指针寄存器，属于riscv的32个通用寄存器之一，为了 `gp` 附近正负2KB内全局变量的访问，更多细节：[riscv-gp](https://www.cnblogs.com/wahahahehehe/p/15140813.html) 
*  为每一个hart都分配8KB的栈空间；
* 设置 `mtvec` 的值为 `trap` 函数地址，作为M-Mode的Trap跳转地址；
* 设置 `mepc` 的值为 `main` 函数地址，作为M-Mode的Trap返回地址；
* 设置 `mstatus`，
  * MPP为3表示原特权级为M-Mode，当执行 `mret` 指令时特权级恢复为MPP所记录的模式；
  * MPIE为M-Mode中断使能保存位，当执行 `mret` 指令时MPIE恢复到MIE上；
* 设置 `mie.MEIE/SEIE` 使能M-Mode中断；
* 设置 `mideleg` 为0，表示在M-Mode处理所有类型的中断，并不代理到其它模式处理；
* 设置 `ra` 为 `park` 函数地址，该函数为一个无限循环；
* 最后执行 `mret` 指令，切换到M-Mode (为了后续测试aia方便，没有发生实际的特权级切换)，同时跳转到 `mepc` 中的地址开始执行指令，也就是 `main` 函数；

## 1.3 进入 `main` 函数

在 `_start` 中分配栈空间并初始化 `sp` 后可以进行函数调用了：

```rust
// Entry point from start.S
#[no_mangle]
fn main(hart: usize) {
    // Make sure we have space for this HART
    if hart >= MAX_HARTS {
        // We don't, send it to park
        return;
    }
    // Set the trap frame for this hart into the scratch register.
    csr_write!("mscratch", &TRAP_FRAMES[hart]);
    // Let hart 0 be the bootstrap hart and set up UART
    if hart == 0 {
        console::uart_init();
        // Setup the IMSIC and see what happens!
        println!("Booted on hart {}.", hart);
        imsic::imsic_init();
        aplic::aplic_init();
        page::page_init();
        console::run();
    }
}
```

* `MAX_HARTS` 限制了当前程序允许进入的hart数量，该值和链接脚本中是否为harts配置栈空间有关。这里的逻辑就是只允许hart0进入程序的主执行流，如果启用了其余harts那它们会退出 `main` 函数并跳转至 `park` 进入无限循环；

* 分配TRAP_FRAME空间并写入 `mscratch`，该寄存器用于保存进程的Trap上下文，以保证程序在Trap Return时恢复上下文；

* 后续只允许hart0继续执行程序，分别调用了以下函数： 

  ```rust
  console::uart_init();
  imsic::imsic_init();
  aplic::aplic_init();
  page::page_init();
  console::run();
  ```

### console::uart_init

[关于uart](https://rcore-os.cn/rCore-Tutorial-Book-v3/chapter9/2device-driver-1.html) 可以看一下这篇文章。`uart_init` 代码：

```rust
// Registers for the NS16550A. This is connected to 0x1000_0000
// via virt.c in QEMU.
const UART_BASE: usize = 0x1000_0000;
// THR is used if STORE
const UART_THR: usize = 0;
// RBR is used if LOAD
const UART_RBR: usize = 0;
const UART_ICR: usize = 1;
const UART_FCR: usize = 2;
const UART_LCR: usize = 3;
const UART_LSR: usize = 5;

/// Write to a UART register. There are no safety checks! So,
/// make sure you only use the UART_XXYYZZ registers for reg.
fn uart_write(reg: usize, val: u8) {
    unsafe {
        write_volatile((UART_BASE + reg) as *mut u8, val);
    }
}

/// Initialize the UART system. For virt, this is probably not necessary.
/// However, LCR = 3 sets word size to 8 bits, FCR = 1 enables the FIFO
/// and ICR = 1 enables interrupts to be triggered when the RBR receives
/// data.
pub fn uart_init() {
    uart_write(UART_LCR, 3);
    uart_write(UART_FCR, 1);
    uart_write(UART_ICR, 1);
}
```

[uart手册解读](https://www.jianshu.com/p/14ae99223683#FIFO%20%E6%8E%A7%E5%88%B6%E5%AF%84%E5%AD%98%E5%99%A8%EF%BC%88Fcr%EF%BC%89) 可以看一下。

* `uart_write(UART_LCR, 3)`：设置 LCR（线路控制寄存器）的值为 3，这将设置数据位长度为 8 位；

- `uart_write(UART_FCR, 1)`：设置 FCR（FIFO 控制寄存器）的值为 1，这将启用 FIFO（先进先出）缓冲区；
- `uart_write(UART_ICR, 1)`：设置 ICR（中断控制寄存器）的值为 1，这将启用接收缓冲区寄存器（RBR）接收到数据时触发中断；

---

通过查找 `dtc` （Device Tree Compiler）工具生成的 `riscv64-virt.dts` 文件，我们可以看到串口设备相关的MMIO模式的寄存器信息和中断相关信息。

```c
//...
chosen {
  bootargs = [00];
  stdout-path = "/uart@10000000";
};

uart@10000000 {
  interrupts = <0x0a>;
  interrupt-parent = <0x02>;
  clock-frequency = <0x384000>;
  reg = <0x00 0x10000000 0x00 0x100>;
  compatible = "ns16550a";
};
```

`chosen` 节点的内容表明字符输出会通过串口设备打印出来。`uart@10000000` 节点表明串口设备中寄存器的MMIO起始地址为 `0x10000000` ，范围在 `0x00~0x100` 区间内，中断号为 `0x0a` 。 `clock-frequency` 表示时钟频率，其值为0x38400 ，即3.6864 MHz。 `compatible = "ns16550a"` 表示串口的硬件规范兼容NS16550A。

在如下情况下，串口会产生中断：

- 有新的输入数据进入串口的接收缓存
- 串口完成了缓存中数据的发送
- 串口发送出现错误

在 UART 中，可访问的 I/O寄存器一共有8个。访问I/O寄存器的方法把串口寄存器的MMIO起始地址加上偏移量，就是各个寄存器的MMIO地址了。

---

> `println!` 是如何在我们的终端上显示内容的？

现在可以来回答这个问题了，代码如下：

```rust
impl Write for Uart {
    fn write_str(&mut self, s: &str) -> Result {
        for c in s.bytes() {
            self.write_char(c as char)?;
        }
        Ok(())
    }

    fn write_char(&mut self, c: char) -> Result {
        while uart_read(UART_LSR) & (1 << 6) == 0 {}
        uart_write(UART_THR, c as u8);
        Ok(())
    }
}

#[macro_export]
macro_rules! print {
    ($($args:tt)+) => ({
        use core::fmt::Write;
        let _ = write!(crate::console::Uart, $($args)+);
    });
}
#[macro_export]
macro_rules! println
{
    () => ({
           print!("\r\n")
           });
    ($fmt:expr) => ({
            print!(concat!($fmt, "\r\n"))
            });
    ($fmt:expr, $($args:tt)+) => ({
            print!(concat!($fmt, "\r\n"), $($args)+)
            });
}
```

`println!` 最终会调到 `write_char`，该函数会循环判断 `LSR.THRE` 是否为0以确定FIFO是否为空，为1则表示FIFO为空可以发送数据至 `THR`，qemu将uart的输出重定向到 `std-out`，最终在终端上显示。 

---

以上是输出的底层流程，相对应的从实际输入设备，比如鼠标、键盘等接收输入数据，在使能IMSIC以及相应中断使能位之后将触发MSI中断，代码如下：

```rust
/// This will be called when the IRQ #10 (hard coded in virt.c)
/// is triggered. This function first determines if the RBR has
/// data via the line status register (LSR) before pushing the
/// received data to the console ring buffer (CONSOLE_BUFFER).
pub fn console_irq() {
    if uart_read(UART_LSR) & 1 == 1 {
        unsafe {
            CONSOLE_BUFFER.push(uart_read(UART_RBR));
        }
    }
}
```

### imsic::imsic_init

IMSIC是RISC-V引入的AIA (高级中断架构) 的其中一个重要组件，其任务是接收和处理MSI中断。在AIA的设计中，IMSIC是可选实现的，每一个hart对应一个IMSIC，如图：

![img](https://pic1.zhimg.com/v2-5f356b01a181d1738b99c81350e49774_b.jpg)

更多IMSIC的细节：

* [imsic规范解读](https://zhuanlan.zhihu.com/p/655606159)
* [aia基本逻辑](https://wangzhou.github.io/riscv-AIA%E5%9F%BA%E6%9C%AC%E9%80%BB%E8%BE%91%E5%88%86%E6%9E%90/)
* [riscv-msi分析](https://zq.org.edu.kg/2023/12/10/riscv_MSIs/)

---

回到 `imsic::imsic::init` 函数中：

```rust
pub fn imsic_init() {
    let hartid = csr_read!("mhartid");
    // First, enable the interrupt file
    // 0 = disabled
    // 1 = enabled
    // 0x4000_0000 = use PLIC instead
    imsic_write(MISELECT, EIDELIVERY);
    imsic_write(MIREG, 1);

    imsic_write(SISELECT, EIDELIVERY);
    imsic_write(SIREG, 1);

    // Set the interrupt threshold.
    // 0 = enable all interrupts
    // P = enable < P only
    // Priorities come from the interrupt number directly
    imsic_write(MISELECT, EITHRESHOLD);
    // Only hear 0, 1, 2, 3, and 4
    imsic_write(MIREG, 5);
    
    // Hear message 10
    imsic_write(SISELECT, EITHRESHOLD);
    imsic_write(SIREG, 11);

    // Enable message #10. This will be UART when delegated by the
    // APLIC.
    imsic_enable(PrivMode::Machine, 2);
    imsic_enable(PrivMode::Machine, 4);
    imsic_enable(PrivMode::Supervisor, 10);

    // Trigger interrupt #2
    // SETEIPNUM no longer works
    // This can be done via SETEIPNUM CSR or via MMIO
    // imsic_write!(csr::s::SETEIPNUM, 2);
    unsafe {
        // We are required to write only 32 bits.
        write_volatile(imsic_m(hartid) as *mut u32, 2)
    }
    imsic_trigger(PrivMode::Machine, 4);
}
```

* aia提供了两个寄存器 `*iselect/*ireg` 以访问IMSIC上的所有寄存器，具体通过 `*iselect` 选择某寄存器，然后 `*ireg` 直接映射到相应寄存器上；
* `eidelivery` 置1使能中断投递；
* `eithreshold` 设置中断优先级阈值为5，aia定义中断编号越大优先级越低，因此这里只允许 `0/1/2/3/4` 的中断进入；
* 调用 `imsic_enable` 以使能M-Mode中断响应 `2/3`，使能S-Mode中断响应 `10`；
* 有两种方式可以触发MSI中断：
  * 写 `seteipnum` 触发，中断号为2；
  * 写 `eip` 触发，将 `eip[4]` 置为1，中断号为4；

---

以上是IMSIC的初始化，当触发MSI中断时跳转至 `stvec` 所保存的Trap处理程序入口 `rust_trap`：

```rust
#[no_mangle]
pub fn rust_trap() {
    let mcause = csr_read!("mcause");
    let interrupt = mcause >> 31 & 1 == 1;

    if interrupt {
        // Interrupt (asynchronous)
        match mcause & 0xFF {
            9 => imsic_handle(PrivMode::Supervisor),
            11 => imsic_handle(PrivMode::Machine),
            _ => println!("Unknown interrupt #{}", mcause),
        }
    } else {
        // Exception (synchronous)
        panic!("Unknown exception #{} @ 0x{:08x}: 0x{:08x}", mcause, csr_read!("mepc"), csr_read!("mtval"));
    }
}
```

根据 `mcause` 再进一步路由到指定的Trap处理分支：

```rust
/// Handle an IMSIC trap. Called from `trap::rust_trap`
pub fn imsic_handle(pm: PrivMode) {
    let msgnum = imsic_pop(pm);
    match msgnum {
        0 => println!("Spurious 'no' message."),
        2 => println!("First test triggered by MMIO write successful!"),
        4 => println!("Second test triggered by EIP successful!"),
        10 => console_irq(),
        _ => println!("Unknown msi #{}", msgnum),
    }
}
```

### aplic::aplic_init

APLIC是RISC-V AIA引入的另一个组件，作用类似于PLIC，负责收集、处理并向hart投递外部中断。有两种投递方式：

* 如果hart侧没有实现IMSIC，APLIC会直接通过线连接的方式将外部中断投递给hart；
* 如果hart侧实现了IMSIC，APLIC必须通过msi的方式将外部中断投递给hart；

更多的APLIC细节：

* [aplic分析](https://zq.org.edu.kg/2023/12/10/riscv_aia_aplic/)

* [aplic规范解读](https://zhuanlan.zhihu.com/p/655029162)

---

```rust
pub fn aplic_init() {
    // The root APLIC
    let mplic = Aplic::as_mut(AplicMode::Machine);
    // The delgated child APLIC
    let splic = Aplic::as_mut(AplicMode::Supervisor);

    // Enable both the machine and supervisor PLICS
    mplic.set_domaincfg(false, true, true);
    splic.set_domaincfg(false, true, true);

    // Write messages to IMSIC_S
    mplic.set_msiaddr(AplicMode::Supervisor, crate::imsic::IMSIC_S);

    // Delegate interrupt 10 to child 0, which is APLIC_S
    // Interrupt 10 is the UART. So, whenever the UART receives something
    // into its receiver buffer register, it triggers an IRQ #10 to the APLIC.
    mplic.sourcecfg_delegate(10, 0);

    // The EIID is the value that is written to the MSI address
    // When we read TOPEI in IMSIC, it will give us the EIID if it
    // has been enabled.
    splic.set_target_msi(10, 0, 0, 10);

    // Level high means to trigger the message delivery when the IRQ is
    // asserted (high).
    splic.set_sourcecfg(10, SourceModes::LevelHigh);

    // The order is important. QEMU will not allow enabling of the IRQ
    // unless the source configuration is set properly.
    // mplic.set_irq(10, true);
    splic.set_ie(10, true);
}

```





### page::page_init



### console::run





 
