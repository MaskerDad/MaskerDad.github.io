---
title: 2024-1-9-bao-hypervisor代码分析

date: 2024-1-9 17:00:00 +0800

categories: [bao-project学习]

tags: [hypervisor]

description: 
---

# 0 参考



# 1 框架

## 1.1 目录结构

> **ls src/**

```shell
arch  core  lib  linker.ld  platform
```

> **tree src/arch/riscv**

```shell
./arch/riscv
├── aclint.c
├── arch.mk
├── asm_defs.c
├── boot.S
├── cache.c
├── cpu.c
├── exceptions.S
├── inc
│   └── arch
│       ├── aclint.h
│       ├── bao.h
│       ├── cache.h
│       ├── cpu.h
│       ├── csrs.h
│       ├── encoding.h
│       ├── fences.h
│       ├── hypercall.h
│       ├── instructions.h
│       ├── interrupts.h
│       ├── iommu.h
│       ├── mem.h
│       ├── page_table.h
│       ├── platform.h
│       ├── sbi.h
│       ├── spinlock.h
│       ├── tlb.h
│       ├── vm.h
│       └── vmm.h
├── interrupts.c
├── iommu.c
├── irqc
│   ├── aia
│   │   ├── aplic.c
│   │   ├── inc
│   │   │   ├── aplic.h
│   │   │   ├── irqc.h
│   │   │   └── vaplic.h
│   │   ├── objects.mk
│   │   └── vaplic.c
│   └── plic
│       ├── inc
│       │   ├── irqc.h
│       │   ├── plic.h
│       │   └── vplic.h
│       ├── objects.mk
│       ├── plic.c
│       └── vplic.c
├── mem.c
├── objects.mk
├── page_table.c
├── relocate.S
├── root_pt.S
├── sbi.c
├── sync_exceptions.c
├── vm.c
└── vmm.c
```

> **tree src/core**

```shell
./core/
├── cache.c
├── config.c
├── console.c
├── cpu.c
├── hypercall.c
├── inc
│   ├── bao.h
│   ├── cache.h
│   ├── config_defs.h
│   ├── config.h
│   ├── console.h
│   ├── cpu.h
│   ├── emul.h
│   ├── fences.h
│   ├── hypercall.h
│   ├── interrupts.h
│   ├── io.h
│   ├── ipc.h
│   ├── mem.h
│   ├── objpool.h
│   ├── page_table.h
│   ├── platform_defs.h
│   ├── platform.h
│   ├── spinlock.h
|   ├── tlb.h
│   ├── types.h
│   ├── vm.h
│   └── vmm.h
├── init.c
├── interrupts.c
├── ipc.c
├── mem.c
├── mmu
│   ├── inc
│   │   └── mem_prot
│   │       ├── io.h
│   │       ├── mem.h
│   │       └── vmm.h
│   ├── io.c
│   ├── mem.c
│   ├── objects.mk
│   ├── vm.c
│   └── vmm.c
├── mpu
│   ├── config.c
│   ├── inc
│   │   └── mem_prot
│   │       ├── io.h
│   │       ├── mem.h
│   │       └── vmm.h
│   ├── io.c
│   ├── mem.c
│   ├── objects.mk
│   ├── vm.c
│   └── vmm.c
├── objects.mk
├── objpool.c
├── vm.c
└── vmm.c
```

> **tree src/lib**

```shell
./lib
├── bitmap.c
├── inc
│   ├── bit.h
│   ├── bitmap.h
│   ├── list.h
│   ├── printk.h
│   ├── string.h
│   └── util.h
├── objects.mk
├── printk.c
└── string.c
```

> **tree src/platform**

```shell
./platform/
├── drivers
│   ├── 8250_uart
│   │   ├── 8250_uart.c
│   │   ├── inc
│   │   │   └── drivers
│   │   │       └── 8250_uart.h
│   │   └── objects.mk
│   ├── sbi_uart
│       ├── inc
│       │   └── drivers
│       │       └── sbi_uart.h
│       ├── objects.mk
│       └── sbi_uart.c
├── qemu-riscv64-virt
│   ├── inc
│   │   └── plat
│   │       └── platform.h
│   ├── objects.mk
│   ├── platform.mk
│   └── virt_desc.c
```

## 1.2 简单梳理

先梳理一下主干吧，从 `arch/riscv/boot.S` 开始：

```assembly
_reset_handler
	# 设置trap处理入口
    la      t0, _hyp_trap_vector
    and     t0, t0, ~STVEC_MODE_MSK
    or      t0, t0, STVEC_MODE_DIRECT
    csrw    stvec, t0
    
    # 禁用中断和MMU
    csrw   sstatus, zero
    csrw   sie, zero
    csrw   sip, zero
    csrw   satp, zero
    
    #...
    # setup_cpu
    # _enter_vas
    j init
```

然后进入 `core/init.c: init`：

```c
void init(cpuid_t cpu_id, paddr_t load_addr)
{
    /**
     * These initializations must be executed first and in fixed order.
     */
    cpu_init(cpu_id, load_addr);
    mem_init(load_addr);
    /* -------------------------------------------------------------- */
    console_init();
    if (cpu_is_master()) {
        console_printk("Bao Hypervisor\n\r");
    }
    interrupts_init();
    vmm_init();
    /* Should never reach here */
    while (1) { }
}
```

初始化按固定次序执行：

```c
init
    +-> cpu_init
    +-> mem_init
    +-> console_init
    +-> interrupt_init
    +-> vmm_init
```











