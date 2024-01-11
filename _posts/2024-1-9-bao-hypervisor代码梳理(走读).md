---
title: bao-hypervisor代码梳理(走读)

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

先梳理一下主干，从 `arch/riscv/boot.S` 开始：

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

---

### cpu_init

```c
// core/
cpu_init
    +-> cpu_arch_init
    	+-> if (cpuid == CPU_MASTER)
            +-> sbi_init
            	+-> sbi_get_spec_version
            	+-> sbi_probe_extension
            	+-> interrupts_reserve
            +-> for 0..paltform.cpu_num
                	if cur.hartid is minior_hart:
						sbi_hart_start(...)
                        // wake up other harts!
    +-> if (cpu_is_master())
        // don't wake up other hart here!
```

唤醒从核。

---

### mem_init

```c
mem_init
    +-> mem_port_init
    +-> if(cpu_is_master)
       	+-> mem_setup_root_pool
        +-> mem_create_ppools
    +-> cpu_sync_and_clear_msgs(&cpu_glb_sync);
    /* Wait for master core to initialize memory management */
```

---

### console_init

```c
void console_init()
{
    if (cpu_is_master()) {
        if ((platform.console.base & PAGE_OFFSET_MASK) != 0) {
            WARNING("console base must be page aligned");
        }

        uart = (void*)mem_alloc_map_dev(&cpu()->as, SEC_HYP_GLOBAL, INVALID_VA,
            platform.console.base, NUM_PAGES(sizeof(*uart)));

        fence_sync_write();

        uart_init(uart);
        uart_enable(uart);

        console_ready = true;
    }

    cpu_sync_and_clear_msgs(&cpu_glb_sync);
}
```

uart驱动

---

### interrupt_init

```c
interrupts_init
    +-> interrupts_arch_init
    	+-> if(cpu_is_master)
            +-> irqc_init
            	+-> aplic_init
            +-> aclint_init
      	/* Wait for master hart to finish irqc initialization */
        +-> irqc_cpu_init
           	+-> aplic_idc_init
        +-> CSRS(sie, SIE_SEIE)
    +-> interrupts_cpu_enable
    	+-> interrupts_arch_enable
```

外部、本地中断控制器 `aplic/aclint` 初始化，中断使能。

### vmm_init

```c
vmm_init
    +-> vmm_arch_init
    	* Delegate all interrupts and exceptions not meant to be dealt by the hypervisor
    	* enable the stimer interrupt
    +-> vmm_io_init
    	+-> io_init
    		+-> iommu_arch_init
    			+-> rv_iommu_init()
    			/* Init and enable RISC-V IOMMU. */
    +-> ipc_init
    	if(cpu_is_master)
        +-> ipc_alloc_sheme()
    /* vm */
    +-> vmm_alloc_install_vm
        +-> vmm_alloc_vm
        +-> vmm_vm_install
    +-> vm_init
        +-> vm_allocation_init
        +-> vm_master_init
            /* Before anything else, initialize vm structure.c */
        +-> vm_cpu_init
            /* Initialize each core. */
        +-> vm_vcpu_init
            /* Initialize each virtual core. */
        +-> vm_arch_init
            /**
     		 * Perform architecture dependent initializations. This includes, for example, setting the page
             * table pointer and other virtualization extensions specifics.
             */
        +-> if(master)
            /**
     		 * Create the VM's address space according to configuration and where its image was loaded.
             */
            +-> vm_init_mem_regions
            +-> vm_init_dev
        	+-> vm_init_ipc
    +-> vcpu_run(cpu()->vcpu)
        +-> vcpu_arch_run
            +-> vcpu_arch_entry
            	+-> VM_ENTRY

void vmm_vm_install(struct vm_install_info* install_info)
{
    pte_t* pte = pt_get_pte(&cpu()->as.pt, 0, (vaddr_t)install_info->base);
    *pte = install_info->vm_section_pte;
}
```

## 1.3 总结

整体看了一下，bao-hypervisor的hart应该和guest存在一种绑定关系，不会出现guest在多个hart间迁移的情况，因此代码中并没有出现和 `vs_*` 寄存器相关的 `load/restore` 工作。后续准备看看相关论文、资料。









