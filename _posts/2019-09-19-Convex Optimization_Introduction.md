---
layout: post
title: "Convex Optimization: Introduction"
description: ""
categories: [ConvexOptimization]
tags: [Essential and Professional Course]
redirect_from:
  - /2019/09/19/
typora-root-url: ..
---

## 1. Mathematical optimization 数值优化

### What’s mathematical optimization problem

$$\begin{array}{*{20}{c}}
  {{\text{minimize}}}&{{f_0}\left( x \right)} \\ 
  {{\text{subject to}}}&{{f_i}\left( x \right) \leqslant {b_i},i = 1,...,m} 
\end{array}$$

这里

- $x=(x_1,...,x_n)$: optimization variables  待优化变量
- ${f_0}:{{\mathbf{R}}^n} \to {\mathbf{R}}$: objective function  目标函数
- ${f_i}:{{\mathbf{R}}^n} \to {\mathbf{R}},i=1,...,m$: constraint function  约束函数
- Optimal solution $x^*$ has smallest value of $f_0$ among all vectors that satisfy the constraints.最优解对应着满足约束条件的优化变量

> **A. Everything is an optimization problem**
>
> **B. We can’t realy solve most optimization problem**

#### Examples

- Portfolio optimization 投资组合优化
- Device sizing in electronic circuits  电流环路的设备尺寸优化
- data fitting 数据拟合

### Solving optimization problem

- general optimization problem

  - very difficult to solve

  - methords involve some compromise , e.g. very long compution time, or not always finding the solution. 这些方法牵扯到一些危害

  - exceptions: certain problem classes can be solved efficiently and reliably.  

    - least-squares problems 最小二乘问题 

      > ${\rm{minimize}}\left\| {{\rm{A}}x - {\rm{b}}} \right\|_2^2$
      >
      > analytical solution: ${x^*} = {({A^T}A)^{ - 1}}{A^T}b$

    - linear programming problems  线性规划问题

      > $\begin{array}{l}
      > {\rm{minimize\ }}{{\rm{c}}^T}x\\
      > {\rm{subject\ to\ }}a_i^Tx \le {b_i}{\rm{,\ }}i = 1,...,m
      > \end{array}$
      >
      > - no analytical formula for solution

    - convex optimization problem  凸优化问题。

      > $\begin{array}{l}
      > {\rm{minimize\ }}{f_0}(x)\\
      > {\rm{subject\ to\ }}{f_i}(x) \le {b_i}{\rm{, }}i = 1,...,m
      > \end{array}$

### convex optimization problem

$$\begin{array}{l}
{\rm{minimize\ }}{f_0}(x)\\
{\rm{subject\ to\ }}{f_i}(x) \le {b_i}{\rm{, }}i = 1,...,m
\end{array}$$

#### Convex:

$${f_i}(\alpha x + \beta y) \le \alpha {f_i}(x) + \beta {f_i}(y)$$ , if $\alpha  + \beta  = 1,\ a \ge 0,\ \beta  \ge 0$

includes least-squares problems and liear problems as special cases. 最小二乘问题和线性规划问题都是凸优化问题的特殊情况。

### Couse goals and topics

#### goal

1. recognize/ formulate problems (such as illumination problem) as convex optimization problems.
2. develop code for problems of moderate size(1000 lamps,5000 patches)
3. characterize optimal solution (optimal power distribution ), give limits of performance, etc.

#### topics

1. covex sets, functions, optimization problems.
2. examples and applications
3. algorithms