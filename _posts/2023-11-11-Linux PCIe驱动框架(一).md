---
title: Linux PCIe框架(一)

date: 2023-11-11 17:00:00 +0800

categories: [kernel, PCI/PCIe]

tags: [PCI/PCIe, MSI, arm64]

description: 


---

# 0 前言

>1. **Kernel版本：4.14**
>2. **ARM64处理器：Contex-A53，双核**

从系列，将会针对PCI/PCIe专题来展开，涉及的内容包括：

1. PCI/PCIe总线硬件；
2. Linux PCI驱动核心框架；
3. Linux PCI Host控制器驱动；
4. PCI/PCIe MSI 中断机制；

# 1 Softirq/tasklet





