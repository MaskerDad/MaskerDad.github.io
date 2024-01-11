---
title: fpsimd_context_switch

date: 2024-1-10 17:00:00 +0800

categories: [SIMD内核支持]

tags: [vector]

description: 
---

# 0 参考





# 1 背景

> 基于[[v9, 00/10] riscv: support kernel-mode Vector](https://lore.kernel.org/linux-riscv/20240108035209.GA212605@sol.localdomain/T/#mda836061caf7a5db9b6994a58ec8e32721ae5038)，目前最新v6.7主线未合入。
>
> ---
>
> 设置了TIF_RISCV_V_DEFER_RESTORE，但本次恢复还是多余的场景：
>
> 从多hart多task角度出发，arm的解决方案可以做到：返回用户态前，hart上装载的如果不是task的vector_state能够感知到。
>
> * hart视角下：TIF_RISCV_V_DEFER_RESTORE没问题
> * task视角下：如果task还被调度到原hart上时，原hart的vector_state不变，原实现感知不到。

![image-20240111102727997](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202401111027078.png)

# 2 框架

为了减少不必要地保存和恢复vector状态的次数，内核需要跟踪两件事：

> a) 对于每个任务，内核需要记住最后一个将任务的vector状态加载到寄存器上的hart是哪一个；
>
> b) 对于每个hart，内核需要记住最近加载到寄存器上的用户态vector状态属于哪个任务，或者在此期间是否已被用于执行内核模式vector操作。

* 对于a），我们向 `thread_struct` 添加了一个 `vector_cpu` 字段，每当vector状态被加载到hart上时，该字段会更新为当前hart的ID。
* 对于b），我们添加了per-hart变量 `vector_last_state`，其中包含最近加载到hart上的任务的用户空间vector状态的地址，如果在此之后执行了内核模式vector，则为NULL。

---

基于以上，我们在任务切换时就不再需要立即恢复下一个vector状态。相反，我们可以将这个检查推迟到用户空间的恢复阶段，在这个阶段我们验证hart的 `vector_last_state` 和任务的 `vector_cpu` 是否仍然保持同步。如果是这种情况，我们可以省略vector的恢复操作。

为了描述上述的 `task-cpu` 的双向同步，使用统一线程标识 `TIF_FOREIGN_VSTATE` 来指示当前任务的用户态vector状态是否存在于hart中。除非当前hart的vetor寄存器包含当前任务的最新用户态vector状态，否则设置该标志。

对于某个任务，其可能的执行序列如下：

* **任务被调度：**如果任务的vector_cpu字段包含当前hart的ID，且hart的 `vector_last_state` per-cpu变量指向任务的vector_state，`TIF_FOREIGN_VSTATE` 标志位被清除，否则被设置；
* **任务返回到用户空间：**如果设置了 `TIF_FOREIGN_VSTATE` 标志，任务的用户空间vector状态将从内存复制到寄存器中，任务的vector_cpu字段将设置为当前hart的ID，当前CPU的 `vector_last_state` 指针将设置为该任务的vstate，并清除 `TIF_FOREIGN_VSTATE` 标志；
* **该任务执行一个普通的系统调用：**当返回到用户空间时，`TIF_FOREIGN_VSTATE` 标志仍将被清除，因此不会恢复vector状态；
* **该任务执行一个系统调用，该系统调用执行一些vector指令：**在此之前，调用 `kernel_vector_begin()` 函数，将任务的vector寄存器内容复制到内存中，清除vector_last_state变量，并设置 `TIF_FOREIGN_VSTATE` 标志；
* **在调用kernel_vector_end()之后，任务被抢占：**由于我们还没有从第二个系统调用中返回，`TIF_FOREIGN_VSTATE`仍然被设置，因此vector寄存器中的内容不会被保存到内存中，而是被丢弃。

![fpsimd_reduce_switch_times.drawio](https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202312201753572.png)

1. **task0首次被调度：**

   > 判断是否保持同步:
   >
   > TIF_FOREIGN_VSTATE = (task0->vector_cpu != hart0 || vector_last_state != task0)

2. **task0返回用户态：**

   > \* 判断TIF_FOREIGN_VSTATE，这里为TRUE，
   > 那就恢复vector_state到寄存器上；
   > \* task0->vector_cpu = hart0;
   > \* vector_last_state = task0;
   > \* TIF_FOREIGN_VSTATE = false;

3. **task0让出CPU控制权**

4. **task0再次被调度运行，目标CPU仍然为hart0：**

   > 还是判断和1）相同的两个变量，看是否同步，此时：
   > task0->vector_cpu = hart0；
   > vector_last_state = task0;
   > => TIF_FOREIGN_VSTATE = false;

5. **task0再次返回用户态：**

   > task0再次被调度运行，目标CPU仍然为hart0：

---

<font color='red'>***优化点***</font>

- [ ] Trap进入内核，如果不发生调度，再次返回同一hart，一切都没变，这种场景是否常见？





# 3 实现
