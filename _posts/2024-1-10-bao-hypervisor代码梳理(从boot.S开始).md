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

[riscv原子指令分析 | Sherlock's blog (wangzhou.github.io)](https://wangzhou.github.io/riscv原子指令分析/)

以上代码片段用于 `master_hart` 的设置，其中 `lr/sc` 配合可以实现汇编级的锁操作：

* 如果定义了 `CPU_MASTER_FIXED`，即预先定义了主hart的ID，直接将 `CPU_MASTER_FIXED` 值写入 `CPU_MASTER` 变量中；

* 如果没定义 `CPU_MASTER_FIXED`，则写了一段汇编限制 “首先获取锁的hart” 作为主hart，具体实现：

  * 定义了 `_boot_lock` 变量并初始化为0，该变量表示主hart是否已经设置成功，为0表示未成功，为1表示成功；

    * `bnez t2, 2f`，如果不是0就直接向后跳转到标签2处，无需执行主hart设置逻辑；

    > 当然 `_boot_lock` 只能限制那些执行该指令前的harts，对于几乎同步执行 sc.w 的harts来说起不到限制。

  * `lr.w/sc.w` 配合使用，`lr.w` 执行时会给加载地址打上一个flag，`sc.w` 首先会看这个flag是否被清除：

    * 没有被清除就执行后续的保存动作，这里就是将t1的值保存到 `_boot_lock` 中，也就是说首个执行这条指令的hart被选中做了主hart，主hart设置成功，最后将t2设置为0表示指令执行成功；
    * 如果flag被清除了， 则直接设置t2为一个非零值表示指令执行失败，其它harts执行该指令时都会失败；

  * 然后，对于主hart的后续逻辑，将 `a0(hart_id)` 保存到 `CPU_MASTER` 中；对于其它harts则向前跳转至标签1；

  * 对于那些重新回到标签1的harts，又会重新加载一遍 `_boot_lock` 的值，但此时该值已经被更新为1了，直接向后跳转至标签2；

实际上，对于多hart执行流来说，其上代码限制了两种情况：

1. 当主hart设置成功后，仍未执行 `lr.w` 的harts；
2. 多个hart都执行了 `lr.w`，但还未执行 `sc.w`，它们读到的 `_boot_lock` 都是未修改的0，因此 `bnez t2, 2f`并不会跳转到标签2执行；

至此，`CPU_MASTER` 被设置了唯一的主hart id，后续我们可以根据 `CPU_MASTER` 和 `a0` 寄存器的差异来执行不同的逻辑。

---

```assembly
2:
#endif
   	/* Setup bootstrap page tables. Assuming sv39 support. */ 
 	/* Skip initialy global page tables setup if not hart */
    LD_SYM  t0, CPU_MASTER
	bne     a0, t0, wait_for_bsp   

 	la	    a3, _page_tables_start	
	la	    a4, _page_tables_end	
    add     a3, a3, s6
    add     a4, a4, s6
	call	clear		 

    la          t0, root_l1_pt
    add         t0, t0, s6
    la          t1, root_l2_pt
    add         t1, t1, s6
    PTE_FILL    t1, t1, PTE_TABLE
    li          t2, BAO_VAS_BASE
    PTE_PTR     t2, t0, 1, t2
    STORE       t1, 0(t2)


    la          t0, root_l2_pt
    add         t0, t0, s6
    LD_SYM      t1, _image_start_sym
    PTE_PTR     t1, t0, 2, t1
    LD_SYM      t2, _image_load_end_sym
    PTE_PTR     t2, t0, 2, t2

    la          t0, _image_start
    PTE_FILL    t0, t0, PTE_HYP_FLAGS | PTE_PAGE
1:
    bge     t1, t2, 2f
    STORE   t0, 0(t1)
    add     t1, t1, 8
    add     t0, t0, 0x400
    j       1b
2:
    la          t0, root_l2_pt
    add         t0, t0, s6
    LD_SYM      t2, _image_end_sym
    PTE_PTR     t2, t0, 2, t2
    bge         t1, t2, 3f
    la          t0, _image_noload_start
    PTE_FILL    t0, t0, PTE_HYP_FLAGS | PTE_PAGE
    j 1b
3:
    fence   w, w
    la      t0, _barrier
    li      t1, 1
    STORE   t1, 0(t0)
    j       map_cpu
```

