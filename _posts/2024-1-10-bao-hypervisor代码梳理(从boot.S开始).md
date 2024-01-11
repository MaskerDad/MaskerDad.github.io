---
title: bao-hypervisor代码梳理(从boot.S开始).md

date: 2024-1-10 17:00:00 +0800

categories: [bao-project学习]

tags: [hypervisor]

description: 
---

# 0 参考







# 1 boot.S

```assembly
/*
	以下代码必须位于镜像的底部，因为这是 bao 的入口点。
	因此 .boot 也必须是链接器脚本中的第一个部分。
	不要在此部分中的_reset_handler之前实现任何代码。
*/
 .section ".boot", "ax"
.globl _reset_handler

_reset_handler:
    mv      a2, a1
    la      a1, _image_start
    la      s6, extra_allocated_phys_mem

	/*
	  尽早设置stvec。如果我们在该引导代码中导致异常，我们最终会到达一个已知的位置。
	*/
    la      t0, _hyp_trap_vector
    and     t0, t0, ~STVEC_MODE_MSK
    or      t0, t0, STVEC_MODE_DIRECT
    csrw    stvec, t0

	/*
	  设置hart初始状态，禁用中断和分页
	*/
    csrw   sstatus, zero
    csrw   sie, zero
    csrw   sip, zero
    csrw   satp, zero 
    
####

#define BIT_MASK(OFF, LEN) (((((1UL)<<((LEN)-1))<<1)-1)<<(OFF))

#define STVEC_MODE_OFF (0)
#define STVEC_MODE_LEN (2)
#define STVEC_MODE_MSK BIT_MASK(STVEC_MODE_OFF, STVEC_MODE_LEN)
#define STVEC_MODE_DIRECT (0)
```

* 上一启动阶段应该传递以下参数：
    * a0 -> hart id
    * a1 -> config 二进制加载地址
* 以下寄存器保留为传递给 init 函数作为参数：
    * a0 -> hart id
    * a1 -> 包含镜像基址加载地址
    * a2 -> config 二进制加载地址（最初在 a1 中传递）

* 其余代码必须在主流程中使用 t0-t6 作为暂存器，在辅助例程中使用 s0-s5。s6-s11 用于保存常量，a3-a7 用作参数和返回值（也可以在辅助例程中使用）。

> 疑问：
>
> 1. 上一阶段传递参数 `config_image_load_addr` 的实现？ => OpenSBI

---

```assembly
#if defined(CPU_MASTER_FIXED)
    la      t0, CPU_MASTER
    li      t1, CPU_MASTER_FIXED
    sd      t1, 0(t0)
#else
.pushsection .data
_boot_lock:
    .4byte 0
.popsection
    la      t0, _boot_lock
    li      t1, 1
1:
    lr.w    t2, (t0)
    bnez    t2, 2f
    sc.w    t2, t1, (t0)   
    bnez    t2, 1b 
    la      t0, CPU_MASTER
    sd      a0, 0(t0)
2:
#endif
```

这段代码是根据是否定义了 CPU_MASTER_FIXED 宏来选择不同的代码路径。

如果定义了 CPU_MASTER_FIXED 宏，将执行第一个代码块。在该代码块中，首先将 CPU_MASTER 地址加载到寄存器 t0 中，然后将 CPU_MASTER_FIXED 的值加载到寄存器 t1 中，最后将寄存器 t1 的值存储到寄存器 t0 所指向的地址中。

如果没有定义 CPU_MASTER_FIXED 宏，将执行第二个代码块。在该代码块中，首先定义一个名为 _boot_lock 的 4 字节变量，并将其初始化为 0。然后将 _boot_lock 的地址加载到寄存器 t0 中，将值 1 加载到寄存器 t1 中。

接下来，使用 load-reserved/store-conditional（lr/sc）指令序列来尝试获取锁。首先使用 lr 指令将寄存器 t2 加载为寄存器 t0 所指向的地址的值，然后使用 bnez 指令检查寄存器 t2 的值是否为非零。如果非零，表示锁已被其他 hart 获取，跳转到标签 2: 处。如果为零，表示锁可用，使用 sc 指令尝试将寄存器 t1 的值存储到寄存器 t0 所指向的地址中。如果存储成功，跳转到标签 2: 处。如果存储失败，表示其他 hart 已经获取了锁，使用 bnez 指令跳转到标签 1: 处重新尝试获取锁。

最后，跳转到标签 2: 处，将 a0 的值存储到 CPU_MASTER 地址中。如果未定义 CPU_MASTER_FIXED 宏，则该代码块将为首个获取锁的 hart 标记为 CPU_MASTER。
