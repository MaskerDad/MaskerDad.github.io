---
title: Makefile常识

date: 2024-1-2 17:00:00 +0800

categories: [Makefile]

tags: [Makefile]

description: 
---

# 0 参考



# 1 常用

* Makefile中 `:=/?=/+=` 的区别？

  > = 是最基本的赋值。make会将整个makefile展开后，再决定变量的值。
  > := 是覆盖之前的值。表示变量的值决定于它在makefile中的位置，而不是整个makefile展开后的最终值。
  > ?= 是如果没有被赋值过就赋予等号后面的值
  > += 是添加等号后面的值

* `@` 符号的使用：通常makefile会将其执行的命令行在执行前输出到屏幕上。如果将 `@` 添加到命令行前，这个命令将不被make回显出来。
