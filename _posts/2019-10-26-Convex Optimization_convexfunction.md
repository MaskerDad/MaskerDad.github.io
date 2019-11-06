---
layout: post
title: "Convex Optimization: Convex Function"
description: ""
categories: [ConvexOptimization]
tags: [Essential and Professional Course]
redirect_from:
  - /2019/10/26/
typora-root-url: ..
---

* Kramdown table of contents
{:toc .toc}
# Basic properties and examples

- **convex**: $f(\theta x+(1-\theta)y)\le \theta f(x)+(1-\theta)f(y)$  for  all $x,y \in \mathbf{dom} f$ and $0 \le \theta \le 1$
- **strictly convex**: $f(\theta x+(1-\theta)y)< \theta f(x)+(1-\theta)f(y)$  whenever $x \ne y$ and $0 <\theta < 1$
- **concave**: if $-f$ is convex
- **strictly concave**: if $-f$ is strictly convex
- affine functions are both convex and concave
- $f$ is convex $ \Leftrightarrow$ $g(t)=f(x+tv)$ for $\{ t|x+tv\in \mathbf{dom} f\}$ is convex



- **Extend-value extension(扩展值延伸)**
  - $\tilde f(x) = \left\{ \begin{array}{ll}
    f(x) & x \in \mathbf{dom}f\\
    \infty & x \notin \mathbf{dom}f
    \end{array} \right.$
  - concave function is to be $- \infty$ outside its domain
- **First order condition**
  - $f(y)\ge f(x)+\nabla f(x)^T(y-x)$
  - **global underestimator(全局下估计)**
- **Second order condition**
  - $\nabla^2f(x)\succeq 0$
- Examples
  - Every norm on $R^n$ is convex
  - $f(x)=\max\{x_1,...,x_n\}$ is convex on $R^n$
  - $f(x,y=x^2/y)$ with $\mathbf{dom} f =R\times R_{++}=\{(x,y)\in R^2|y >0\}$ is convex
  - $f(x)=\log (e^{x_1}+\cdots +e^{x_n})$ is convex on $R^n$
  - $f(x)={\left( {\prod\nolimits_{i = 1}^n {{x_i}} } \right)^{1/n}}$ is concave on $\mathbf{dom} f=R^n_{++}$(Geometric mean)
  - $f(\mathbf{X})=\log \det\mathbf{X}$ is concave on $\mathbf{dom} f=S^n_{++}$ (Log-determinant)
- **Sublevel sets(下水平集)**
  - $C_\alpha=\{x\in \mathbf{dom} f|f(x)\le \alpha\}$
- **Epigraph(上境图)**
  - $\mathbf{epi}f =\{(x,t)|x\in \mathbf{dom}f,f(x)\le t\}$
  - A function is convex if and only if its epigragh is a convex set.
  - **hypograph** for concave function
    - $\mathbf{hypo}f =\{(x,t)|t\le f(x)\}$
- **Jensen’s inequality and extensions**
  - $f(\theta x+(1-\theta)y)\le\theta f(x)+(1-\theta) f(y)$
  - $f(\theta_1 x_1+\cdots+\theta_kx_k)\le\theta_1 f(x_1)+\cdots+\theta_k f(x_k)$ for $f$ is convex, $x_1,...,x_k\in \mathbf{dom}f$, $\theta_1,...,\theta_k>0$ and $\theta_1+\cdots+\theta_k=1$
  - $f(\mathbf{E}x)\le\mathbf{E}f(x)$
  - $\sqrt{ab}\le(a+b)/2$
  - $a^\theta b^{1-\theta}\le \theta a+(1-\theta)b$
- **Cauchy-Schwarz inequality**: $(a^Ta)(b^Tb)\ge(a^Tb)^2$



# Operations that preserve convexity

- Nonnegative weighted sums
  - $f=w_1f_1+\cdots+w_mf_m$
  - $g(x)=\int_\mathcal{A} {\ w(y)f(x,y){\rm{d}}y}$
  - if $w\ge0$ and $f$ is convex, $\mathbf{epi}(wf)=\left[ {\begin{array}{*{20}{c}}
    {I}&0\\0&w\end{array}} \right] \mathbf{epi}\ f$
- Affine mapping
  - $g(x)=f(Ax+b)$, where$f:R^n \rightarrow R, A\in R^{n\times m},b \in R^n \Rightarrow g:R^m \rightarrow R$
- Pointwise maximum and supremum 
  - $f(x)=\max\{f_1(x),\dots,f_m(x)\}$
  - $g(x)={\rm sup}_{y\in \mathcal{A}}f(x,y)$
  - almost every convex function can be expressed as the pointwise supremum of a family of affine functions. 
    $f(x) = {\rm sup} \{g(x) | g {\rm \ affine}, g(z) ≤ f(z) for all z \}$

> $W=\mathbf{diag} (w)$ 是把向量$w$转换成对角线矩阵$W$

- Composition (复合函数)
  - $f(x)=h \circ g=h(g(x))$
  - $f$ is convex if $h$ is convex, $ \tilde h$ is nondecreasing, and $g$ is convex
  - $f$ is convex if $h$ is convex, $ \tilde h$ is nonincreasing, and $g$ is concave,
  - $f$ is concave if $h$ is concave, $ \tilde h$ is nondecreasing, and $g$ is concave,
  - $f$ is concave if $h$ is concave, $ \tilde h$ is nonincreasing, and $g$ is convex.
- Vector composition(矢量复合)
  - $f(x) = h(g(x)) = h(g_1(x), . . . , g_k(x))$
  - $f$ is convex if $h$ is convex, $h$ is nondecreasing in each argument, and $g_i $are convex,
    $f$ is convex if $h$ is convex, $h$ is nonincreasing in each argument, and $g_i$ are concave,
    $f$ is concave if $h$ is concave, $h$ is nondecreasing in each argument, and $g_i$ are concave.
- Minimization
  - $g(x)\mathop {\inf }\limits_{y\in C} f(x,y)$
- Perspective of a function
  - $g(x,t)=tf(x/t), g:R^{n+1} \rightarrow R$

# The conjugate function( 共轭函数)

$f^*(y)=\mathop{\rm sup}\limits_{x\in \mathbf{dom}f} (y^Tx-f(x))$

- $f^*$ is a convex function

**Basic properties**

- **Fenchel’s inequality: **  $f(x)+f^*(y) \ge x^Ty$
- $f^{**}=f$
- **Differentiable functions**——Legendre transform
  - $f^*(y)=x^{*T} \nabla f(x^*)-f(x^*)$, where $x^*$ maximizes $y^Tx-f(x)$
- **Scaling and composition with affine transformation**
  - $g(x)=af(x)+b$     $g^*(y)=af^*(y/a)-b$
  - $g(x)=f(Ax+b)$    $g^*(y)=f^*(A^{-T}y)-b^TA^{-T}y$
- **Sums of independent functions**
  - $f(u,v)=f_1(u)+f_2(v)$      $f^*(w,z)=f_1^*(w)+f_2^*(z)$

