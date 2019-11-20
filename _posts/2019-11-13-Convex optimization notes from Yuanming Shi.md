---
layout: post
title: "Convex optimization notes from Yuanming Shi"
description: ""
categories: [ConvexOptimization]
tags: [Essential and Professional Course]
redirect_from:
  - /2019/11/13/
typora-root-url: ..
---

> 这篇博客只记录每章我认为需要记忆的一些名词定义和公式。这套课件是我导师自己写的，吸收了各家之长，知识更广泛，更贴近于科研需要的知识。
>
> 注意：这个笔记只用来整理知识架构，备忘，所以里面有些公式因为在Markdown里面不能完美复现，比如向量和矩阵的加粗显示，所以本笔记里很多公式里是向量但是我没加粗。使用的时候一定要注意。可以再在书里查看一下。

* Kramdown table of contents
{:toc .toc}
## Convex set

- Affine set : $x_1,x_2 \in C,\theta \in R\Rightarrow x=\theta x_1+(1-\theta x_2)\in C$ (Line)
- Convex set : $x_1,x_2 \in C,\theta \in [0,1]\Rightarrow x=\theta x_1+(1-\theta x_2)\in C$ (Line segment)
- Hyperplane : $\{x|a^Tx=b\}$
- Polyhedra :  $\{ {\mathbf{x}}| {\mathbf{A}}{\mathbf{x}}\preceq{\mathbf{b}}\},{\mathbf{C}}{\mathbf{x}}={\mathbf{d}}\}$ ($A\in R^{m\times n},C\in R^{p\times n},\preceq$ is componentwise inequality)
- Euclidean Ball :$B(x_c,r)=\{x|{\left\| {x - {x_c}} \right\|_2} \le r\}=\{x_c+ru|{\left\| u\right\|_2} \le 1\}$
- Ellipsoid : $E(x_c,P)=\{x|(x-x_c)^TP^{-1}(x-x_c) \le 1\}=\{x_c+Au|{\left\| u\right\|_2} \le 1\}$, $P\in S^n_{++}$
- Norm: a function $\left\| {\cdot } \right\|$ that satifies
  - $\left\| {x} \right\| \ge 0$; $\left\| {x} \right\| = 0$ if and only if $x=0$
  - $\left\| {tx} \right\| =|t| \left\| {x} \right\|$ for $t\in R$
  - $\left\| {x+y} \right\| \le \left\| {x} \right\|+\left\| {y} \right\|$
  - Norm ball : $\{x|\left\| {x-x_c} \right\|\le r\}$
  - Norm cone: $\{(x,t)\in R^{n+1}|\left\| {x} \right\|\le t\}$ (Euclidean norm cone, second-order cone, ice-cream cone)
- Positive semidefinite
  - $S^n$ is set of symmetric $n\times n$ matrices
  - $S^n_+=\{\mathbf{X}\in S^n|\mathbf{X} \succeq 0\}$: positive semidefinite $n \times n$ matrices。$\mathbf{X}\in S^n_+\Leftrightarrow z^T \mathbf{X} z \ge 0 {\rm \ for\  all\  z }$
  - $S^n_{++}=\{\mathbf{X}\in S^n|\mathbf{X} \succ 0\}$: positive definite $n \times n$ matrices。
- Operation that Preserve Convexity
  -  intersection
  - affine function
  - perspective function
  - linear-fractional function
- Proper cone:
  - $K$ is closed (contains its boundary)
  - $K$ is solid (has nonempty interior)
  - $K$ is pointed (contains no line $x\in K,-x \in K\Rightarrow x=0$)
- Dual Cones $K^*=\{y|y^Tx \ge0 {\rm \ for\  all\ }x \in K\}$
  - self-dual
  - $y \succeq_{K^*}0 \Leftrightarrow y^Tx \ge 0 {\rm\  for\ all\ }x \succeq_K0$

## Convex function

- Convex function : $f(\theta x+(1-\theta)y\le \theta f(x)+(1-\theta)f(y)$ for $0\le\theta\le1$
- Example on $R^n$
  - Affine function $f(\mathbf{x})=\mathbf{a}^T\mathbf{x}+b$ is convex and convave
  - Norms $\left \|\mathbf{x} \right\|$ are convex
  - Quadratic function $f(\mathbf{x})=\mathbf{x}^T\mathbf{P}\mathbf{x}+2\mathbf{q}^T\mathbf{x}+r$ is convex if and only if  $\mathbf{P}\succeq0$
  - Geometric mean $f(\mathbf{x})=(\prod\nolimits_{i = 0}^n x_i )^{1/n}$ is convave
  - log-sum-exp $f(\mathbf{x})=\log \sum_ie^{x_i}$ is convex
  - Quadratic over linear $f(\mathbf{x},y)=\mathbf{x}^T\mathbf{x}/y$ is convex on $R^n \times R_{++}$

- Epigraph ${\rm epi\ } f =\{(x,t)\in R^{n+1}| x\in {\rm dom \ }f, f(x)\le t\} $
- Restriction of a Convex Function to a Line 
  - $f:R^n \rightarrow R$ is convex if and only if the function $g:R \rightarrow R, g(t)=f(\mathbf{x}+t\mathbf{v}), {\rm dom\ }g=\{t|\mathbf{x}+t\mathbf{v}\in {\rm dom\ }f\}$ is conve for any $\mathbf{x}\in {\rm dom\ }f,\mathbf{v} \in R^n$
  - arbitrary line

- First-order condition:$f(\mathbf{y})\ge f(\mathbf{x})+\nabla f(\mathbf{x})^T(\mathbf{y}-\mathbf{x})\ \ \ \forall \mathbf{x},\mathbf{y}\in {\rm dom} f$
- Second-order condition: $\nabla^2f(\mathbf{x})\succeq0 \ \ \ \forall \mathbf{x}\in {\rm dom} f $
- Operations that preserve convexity
  - nonnegative weight sum $\alpha_1f_1+\alpha_2f_2$
  - composition with affine function $f(\mathbf{A}\mathbf{x}+\mathbf{b})$
  - composition with scalar function $f(x)=h(g(\mathbf{x}))$
  - pointwise maximum $f:=\max\{f_1,\cdots,f_n\}$
  - supermum,minimization  $g(x)=\mathop {\rm sup}\limits_{\mathbf{y}\in A}f(x,y) ,g(x)=\mathop {\rm inf}\limits_{\mathbf{y}\in A}f(x,y)  $
  - perspection $g(x,t) =tf(x,t), {\rm dom\ } g= \{(x,t)\in R^{n+1}|x/t \in {\rm  dom\ }f,t >0\}$
- Quasi-convexity : the sublevel sets $S_\alpha=\{\mathbf{x}\in {\rm dom\ }f|f(\mathbf{x})\le \alpha\} $ is all convex
- Log-convexity : $\log f$ is convex

## Convex Optimization Problems

- stationary point  $\nabla f(\mathbf{x})=0$
  - local minimum and gobal minimum
  - Saddle point: if $\mathbf{x}$ is a stationary opint and for any neighborhood $B \subseteq R^n $ exist $\mathbf{y},\mathbf{z} \in B$ such that $f(\mathbf{z})\le f(\mathbf{x})\le f(\mathbf{y})$ and $\lambda_{\min}(\nabla^2f(x))\le0$
- Convex Optimization problem
  - $\begin{array}{*{20}{c}}
    {{\rm{minimize}}}&{{f_0}(x)}&{}\\
    {{\rm{subject\ to}}}&{{f_i}(\mathbf{x}) \le 0}&{i = 1, \cdots ,m}\\
    {}&{{\bf{Ax}} = {\bf{b}}}&{}
    \end{array}$
  - $f_0,f_1,\cdots,f_m$ are convex and equality constraints are affine
- Oprimality Criterion
  - Minimum principle: $\nabla f_0(\mathbf{x^*})^T(\mathbf{y}-\mathbf{x}^*)\ge0 $ for all feasible $\mathbf{y}$
  - Unconstrained problem:  $\nabla f_0(\mathbf{x^*})=0,x^*\in {\rm dom\ }f$
  - Equality constrained problem $\mathbf{A}\mathbf{x^*}=\mathbf{b},\nabla f_0(\mathbf{x^*})+\mathbf{A}\mathbf{v}=0$
  - minimization over nonnegative orthant:$x \succeq0,\left\{ {\begin{array}{*{20}{c}}
    {{\nabla _i}{f_0}(x) \ge 0}&{{x_i} = 0}\\
    {{\nabla _i}{f_0}(x) = 0}&{{x_i} > 0}
    \end{array}} \right.$
- Equivalent Reformulations
  - Introducing slack variables for linear inequalities$\begin{array}{*{20}{c}}
    {\mathop {\rm{minimize}}\limits_{x,s}}&{{f_0}(x)}&{}\\
    {{\rm{subject\ to}}}&{{\mathbf{a}_i}\mathbf{x} +s_i=b_i}&{i = 1, \cdots ,m}\\
    {}&{s_i\ge0}&{}
    \end{array}$
  - epigraph form$\begin{array}{*{20}{c}}
    {\mathop {\rm{minimize}}\limits_{x,t}}&{t}&{}\\
    {{\rm{subject\ to}}}&{f_0(x)-t\le0}&{}\\{}&{{f_i}(\mathbf{x}) \le 0}&{i = 1, \cdots ,m}\\
    {}&{\bf{Ax}} = {\bf{b}}&{}
    \end{array}$
  - minimizing over some variables$\begin{array}{*{20}{c}}
    {\mathop {\rm{minimize}}\limits_{x}}&{{{\tilde f}_0}(\mathbf{x})}&{}\\
    {{\rm{subject\ to}}}&{{f_i}(\mathbf{x}) \le 0}&{i = 1, \cdots ,m}
    \end{array}$ where ${{{\tilde f}_0}(\mathbf{x})}={\rm inf}_yf_0(\mathbf{x},\mathbf{y})$
  - Quasi-convex  optmization: $f_0$ is quasiconvex, and $f_1,\cdots,f_m$ are convex
- Classes of Convex Problem
  - LP(Linear Programming) : objective and constraint functions are affine 
  - QP(Quadratic Programming): convex quadratic objective and affine constraint function
  - QCQP(Quadratically Constrained Quadratic Programming), inequality constraint function is quadratic
  - SOCP(Second-Order Cone Programming): linear objective and second-order cone inequality constrains $\left\| A_ix+b\right\|_2\le c_i^Tx+d_i\ \ \ i=1,\cdots,m$
  - SDP(Semi-Definite Programming)$\begin{array}{*{20}{c}}
    {\mathop {\rm{minimize}}\limits_{x}}&{\mathbf{c}^T\mathbf{x}}&{}\\
    {{\rm{subject\ to}}}&{x_1\mathbf{F}_1+\cdots+x_n\mathbf{F}_n\preceq\mathbf{G}}&{}\\{}&{\mathbf{A}\mathbf{x}=\mathbf{b}}\end{array}$ 

## Lagrange Duality

- Primal problem:  $\begin{array}{*{20}{c}}
  {\mathop {\rm{minimize}}\limits_{x}}&{f_0(\mathbf{x})}&{}\\
  {{\rm{subject\ to}}}&{f_i(x)\le0}&{i = 1, \cdots ,m}\\{}&{{h_i}(\mathbf{x}) == 0}&{i = 1, \cdots ,p}
  \end{array}$

- Lagrangian $L(\mathbf{x},\mathbf{\lambda},\mathbf{v})=f_0(\mathbf{x})+\sum\limits_{i = 1}^m {{{\bf{\lambda }}_i}{f_i}({\bf{x}})}+\sum\limits_{i = 1}^p {{{\bf{v}}_i}{h_i}({\bf{x}})}$

- Dual function: $g(\mathbf{\lambda},\mathbf{v})=\mathop{\rm inf}\limits_{x\in D}L(\mathbf{x},\mathbf{\lambda},\mathbf{v})$, concave

  $f_0(x)\ge L(\mathbf{x},\mathbf{\lambda},\mathbf{v}) \ge g(\mathbf{\lambda},\mathbf{v})$

- Lagrange dual problem: $\begin{array}{*{20}{c}}
  {\mathop {\rm{maximize}}\limits_{\mathbf{\lambda},\mathbf{v}}}&{g(\mathbf{\lambda},\mathbf{v})}&{}\\
  {{\rm{subject\ to}}}&{\mathbf{\lambda}\succeq 0}&{}
  \end{array}$

- Duality

  - week duality:  $d^*\le p^*$(对偶问题的最优解小于原问题的最优解)
  - strong duality: $d^*= p^*$, is very desirable, does not hold in general, usually holds for convex problem
  - duality gap: $p^*-d^*$

- slater’s Constraint Qualification: conditions that guarantee strong duality in convex problem

  - $\exists x \in {\rm int\  }D),f_i(x)<0$

- Complementary slackness $\lambda_i^*f_i(x^*)=0$（互补松弛性）

- KKT condition

  1. primial feasibility: $f_i(x)\le0,i=1,\cdots,m,h_i(x)=0,i=1,\cdots,p$
  2. dual feasibility: $\lambda\succeq0$
  3. complementary slackness: $\lambda_i^*f_i(x^*)=0$ for $i=1,\cdots,m$
  4. zero gradient of Lagrangian with respect to $\mathbf{x}$ : $\nabla f_0(\mathbf{x})+\sum\limits_{i = 1}^m {{{\bf{\lambda }}_i}{\nabla f_i}({\bf{x}})}+\sum\limits_{i = 1}^p {{{\bf{v}}_i}{\nabla h_i}({\bf{x}})}=0$

  - We already known that if strong duality holds and $x,\lambda,v$ are optimal, then they must satisfy the KKT condition.
  - If $x^*,\lambda ^*,v^*$ satisfy the KKT conditions for a convex problem, then they are optimal.

## Constructive Convex Analysis and Disciplined Convex Programming

- Conic program: $\begin{array}{*{20}{c}}
  {{\rm{minimize}}}&{c^Tx}&{}\\
  {{\rm{subject\ to}}}&{Ax=b}&{x\in K}
  \end{array}$,where $K$ is convex cone.

  - the modern canonical form
  - there are well developed solvers for cone programs

  > 如何求解一个凸问题
  >
  > 1. 用一个现存的定制的的求解器去求解特定的问题
  > 2. 利用现存的流行方法研发一个针对你的问题的新的求解器
  > 3. 把你的问题转换为一个cone program (CVX)，然后用一个标准cone program 求解器(SDPT3, MDSEK)

- Basic example

  | Convex                                                       | Concave                                                      |
  | ------------------------------------------------------------ | ------------------------------------------------------------ |
  | $x^p(p\ge0 {\rm \ or\ } p\le0)$<br />$e^x$<br />$x\log x$<br />$a^Tx+b$<br />$x^TPx(P\succeq0)$<br />$\left\|x\right\|$(any norm)<br />$\max\{x_1,\dots,x_n\}$ | $x^p(0\le p\le1)$<br />$\log x$<br />$\sqrt{xy}$<br />$x^TPx(P\preceq0)$<br />$\min\{x_1,\dots,x_n\}$ |
  | $x^2/y(y>0),x^Tx/y(y>0),x^TY^{-1}x(Y\succ0)$<br />$\log(e^{x_1}+\cdots+e^{x_n})$<br />$f(x)=x_{[1]}+\cdots+x_{[k]}$(sum of lagrest $k$ entries)<br />$f(x,y)=x\log(x/y)$   (x,y>0)<br />$\lambda_{\max}(X)(X^T=X)$ | $\log\det X,(\det X)^{1/n}$  $(X\succ 0)$<br /> $\log\Phi(x)$  ($\Phi$is Gaussian CDF)<br />$\lambda_{\min}(X)(X^T=X)$ |

- Calculus rules

  - nonnegative scaling: $\alpha f$
  - sum : $f+g$
  - affine composition : $f(Ax+b)$
  - pointwise maximum : $\max_if_i(x)$
  - composition: $h(f(x))$,  $h$ convex increasing, $f$ convex

- Disciplined convex programing(DCP)

  > 描述凸优化问题的框架；基于Constructive Convex Analysis；凸性是充分而不必要条件；基于 domain specific languages (DSL) and tools

  - zero or one objective, with form
    - minimize {scalar convex expression} or
    - maximize {scalar convex expression}
  - zero or more constraints, with form
    - {convex expression} <={concave expression} or
    - {concave expression} >={convex expression} or
    - {affine expression} >={affine expression}

  > 很容易构建一个DCP分析器；不难将DCP转化为cone problem

- Modeling languages

  - CVX       Matlab       Grant,Boyd            2005
  - CVXPY   Python       Diamond, Boyd    2013

  > slides 中有示例代码，这里不再赘述

## Gradient method

### Gradient methords for unconstrained problem

- Dradient descent (GD) $x^{t+1}=x^{t}-\eta_t\nabla f(x^t)$, a.k.a. steepest descent

- Exact Line Search : $\eta_t=\arg \mathop \min \limits_{\eta \ge 0} f(x^t- \eta \nabla f(x^t))$

- strongly convex and smooth
  - A twice-differentiable function $f$ is said to be $\mu$-strongly  convex and $L$-smooth if $0 \preceq \mu I \preceq\nabla^2f(x)\preceq LI, \forall x$
  - more on strong convexity
    1. $f(y)\ge f(x)+\nabla f(x)^T(x-y)+\frac{\mu}{2}\left \|x-y \right\|^2_2, \forall x,y$
    2. $f(\lambda x+(1-\lambda)y)\le \lambda f(x)+(1-\lambda)f(y)-\frac{\mu}{2}\lambda(1-\lambda)\left \|x-y \right\|^2_2, \forall x,y,0\le\lambda\le1$
    3. $\langle \nabla f(x)-\nabla f(y),x-y\rangle \ge\mu\left \|x-y \right\|^2_2, \forall x,y$
    4. $\nabla^2 f(x)\succeq \mu I, \forall x$
  - more on smoothness
    1. $f(y)\le f(x)+\nabla f(x)^T(x-y)+\frac{L}{2}\left \|x-y \right\|^2_2, \forall x,y$
    2. $f(\lambda x+(1-\lambda)y)\ge \lambda f(x)+(1-\lambda)f(y)-\frac{L}{2}\lambda(1-\lambda)\left \|x-y \right\|^2_2, \forall x,y,0\le\lambda\le1$
    3. $\langle \nabla f(x)-\nabla f(y),x-y\rangle \ge \frac{1}{L}\left \|\nabla f(x)-\nabla f(y)\right\|^2_2, \forall x,y$
    4. $\left \|\nabla f(x)-\nabla f(y)\right\|_2\le L\left \|x-y \right\|_2, \forall x,y$ (L-Lipschitz gradient)
    5. $\left \| \nabla^2 f(x)\right\| \le L, \forall x$
  
- Backtracking line search

  1. Initialize $\eta=1,0<\alpha\le1/2,0<\beta<1$
  2. while $f(x^t-\eta\nabla f(x^t))>f(x^t)-\alpha\eta\left \| \nabla f(x^t)\right\|^2_2$ do:
  3. ​           $\eta \leftarrow\beta\eta$

  -   $f(x^t)-f(x^*) \le \left( 1-\min\{2\mu\alpha,2\beta\alpha\mu/L\}\right)^{t}(f(x^0)-f(x^*))$

- Local strong convexity

  Let $f$ be locally $\mu$-strongly  convex and $L$-smooth such that  $\mu I \preceq\nabla^2f(x)\preceq LI, \forall x\in B_0$, where $B_0:=\{x|\left\|x-x^*\right\|_2\le \left\|x^0-x^*\right\|_2\}$

- Regularity condition: $\langle \nabla f(x),x-x^*\rangle \ge\frac{\mu}{2}\left \|x-x^* \right\|^2_2+\frac{1}{2L}\left \|\nabla f(x)\right\|^2_2, \forall x$

  - $\left\|x^t-x^*\right\|^2_2\le \left( 1- \frac{\mu}{L}\right)^t \left\|x^0-x^*\right\|^2_2$

| problem                                                      | algorithm | stepsize rule                                                | convergence rate                                             |
| ------------------------------------------------------------ | --------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Quadratic minimization<br />${\rm minimize}_x f(x):=\frac{1}{2}(x-x^*)^TQ(x-x^*)$ | GD        | $\eta_t=\frac{2}{\lambda_1(Q)+\lambda_n(Q)}$(constant stepsize) | $\left\|x^t-x^*\right\|_2\le \left( \frac{\lambda_1(Q)-\lambda_n(Q)}{\lambda_1(Q)+\lambda_n(Q)}\right)^t \left\|x^0-x^*\right\|_2$ |
| Quadratic minimization<br />${\rm minimize}_x f(x):=\frac{1}{2}(x-x^*)^TQ(x-x^*)$ | GD        | Exact Line Search                                            | $f(x^t)-f(x^*) \le \left( \frac{\lambda_1(Q)-\lambda_n(Q)}{\lambda_1(Q)+\lambda_n(Q)}\right)^{2t}(f(x^0)-f(x^*))$ |
| Strongly convex and smooth functions                         | GD        | $\eta_t=\frac{2}{\mu+L}$(constant stepsize)                  | $\left\|x^t-x^*\right\|_2\le \left( \frac{\kappa-1}{\kappa+1}\right)^t \left\|x^0-x^*\right\|_2$<br />$\kappa:=L/\mu$ |
| Convex and smooth functions                                  | GD        | $\eta_t=\frac{1}{L}$                                         | $f(x^t)-f(x^*) \le \frac{2L\left\|x^0-x^*\right\|_2^2}{t}$   |

### Gradient methods for constrained problems

- Frank-Wolfe/ conditional gradient algorithm

  1. for $t=0,1,\cdots$ do
  2. ​      $y^t:=\arg\min_{x\in C}\langle \nabla f(x^t),x \rangle$                              (direction finding)
  3. ​      $x^{t+1}=(1-\eta_t)x^t+\eta_ty^t$                                     (line search and update)

  - stepsize $\eta_t$ determined by line search or $\eta_t=\frac{2}{t+2}$
  - Frank-Wolfe can also be applied to nonconvex problems

- Optimizing over Atomic Sets

  - Gauge function $U_D(x):={\rm inf}_{t\ge0}\{t|x\in tD\}$
  - Support function $U^*_D:={\rm sup}_{s\in D}\langle s,y \rangle$

- Projected gradient methods

  1. for $t=0,1,\cdots$ do

  2. ​      $x^{t+1}=\mathcal{P}_{\mathcal{C}}(x^t-\eta_t\nabla f(x^t))$ 

     where $\mathcal{P}_{\mathcal{C}}(x)=\arg\min_{z\in \mathcal{C}}\left\|x-z\right\|_2$ is Euclidean projection onto $\mathcal{C}$

  - Projection theorem
    - Let  $\mathcal{C}$ be closed convex set, Then $x_\mathcal{C}$ is projection of $x$ onto $\mathcal{C}$ iff $(x-x_\mathcal{C})^T(z-x_\mathcal{C})\le0,\forall z\in \mathcal{C}$
    - $-\nabla f(x^t)^T(x^{t+1}-x^t)\ge 0$
    - Nonexpansicness of projection: $\left\|\mathcal{P}_{\mathcal{C}}(x)- \mathcal{P}_{\mathcal{C}}(z)\right\|_2\le \left\|x-z\right\|_2$

| problem                                                      | algorithm    | stepsize rule            | convergence rate                                             | iteration complexity              |
| ------------------------------------------------------------ | ------------ | ------------------------ | ------------------------------------------------------------ | --------------------------------- |
| convex & smooth                                              | Frank-Wolfe  | $\eta_t=\frac{2}{t+2}$   | $f(x^t)-f(x^*) \le \frac{2Ld_C^2}{t+2}$<br />$d_C={\rm sup}_{x,y\in C}\left\|x-y\right\|_2$ | $O(\frac{1}{\epsilon})$           |
| strongly convex & smooth<br />($x\in {\rm int}(\mathcal{C})$) | Projected GD | $\eta_t=\frac{2}{\mu+L}$ | $\left\|x^t-x^*\right\|_2\le \left( \frac{\kappa-1}{\kappa+1}\right)^t \left\|x^0-x^*\right\|_2$<br />$\kappa:=L/\mu$ |                                   |
| strongly convex & smooth                                     | Projected GD | $\eta_t=\frac{1}{L}$     | $\left\|x^t-x^*\right\|_2\le \left( 1- \frac{\mu}{L}\right)^t \left\|x^0-x^*\right\|_2$<br />$O((1-\frac{1}{\kappa})^t$ | $O(\kappa\log\frac{1}{\epsilon})$ |
| convex & smooth                                              | Projected GD | $\eta_t=\frac{1}{L}$     | $f(x^t)-f(x^*) \le \frac{3L\left\|x^0-x^*\right\|_2^2+f(x^0)-f(x^*)}{t+1}$<br />$O(\frac{1}{t})$ | $O(\frac{1}{\epsilon})$           |

## Subgradient methods

- generalizing steepest descent

  1. $d^t\in \arg\min_{\left\|d \right\|_2\le1}f'(x^t,d)$,  where $f'(x^t,d)=\lim_{\alpha \downarrow0} \frac{f(x+\alpha d)-f(x)}{\alpha}$
  2. $x^{t+1}=x^t+\eta_td^t$

  > 问题：1. Finding steepest descent direction involves expensive computation, 2，stepsize rule 不好选择，可能收敛不到最优解，特别是不可导的点。

- (projected) subgradient method

  > 主要就是为了解决不可导的点的问题的

  - $x^{t+1}=\mathcal{P}_{\mathcal{C}}(x^t-\eta_tg^t)$ , where $g^t$ is any subgradient of $f$ at $x^t$

- Subgradient

  - We say $g$ is subgradient of $f$ at point $x$ if $f(z)\ge f(x)+g^T(z-x), \forall z$
    - set of all subgradients of $f$ at $x$ is called subdifferential of $f$ at $x$, denoted by $\partial f(x)$
  - Basic rules
    - scaling: $\partial (\alpha f)=\alpha\partial (f)$
    - summation: $\partial (f_1+f_2)=\partial (f_1)\partial (f_2)$
    - affine transformation : $\partial (f(Ax+b))=A^T\partial f(Ax+b)$
    - chain rule: $\partial (g\circ f)=g'(f(x))\partial f(x)$
    - composition: suppose $f(x)=h(f_1(x),\cdots,f_n(x)),q=\nabla h(y)|_{y=[f_1(x),\cdots,f_n(x)]},g_i\in \partial f_i(x)$. then   $q_1g_1+\cdots+q_ng_n\in \partial f(x)$
    - pointwise maximum: $f(x)=\max_{1\le i\le k}f_i(x)$, then $\partial f(x)={\rm conv}\{ \cup \{\partial f_i(x)|f_i(x)=f(x)\}\}$
    - pointwise supremum :  $f(x)=\sup_{\alpha\in \mathcal{F}}f_\alpha(x)$, then $\partial f(x)={\rm closure}({\rm conv}\{ \cup \{\partial f_\alpha(x)|f_\alpha(x)=f(x)\}\})$
  - negative subgradent is not necessarily descent direction (lack of continuity)
    - $f^{{\rm best},t}:=\mathop \min\limits_{1\le i\le t} f(x^i)$
    - $f^{\rm opt}:=\min_x f(x)$

- Lipschitz function : $|f(x)-f(z)|\le L_f\|x-z\|_2, \forall x,z$

- Projected subgradient update：$x^{t+1}=\mathcal{P}_{\mathcal{C}}(x^t-\eta_tg^t)$

- majorizing function: $\|x^{t+1}-x^*\|_2^2\le\|x^{t}-x^*\|_2^2-2\eta_t(f(x^t)-f^{\rm opt})+\eta^2_t\|g^t\|_2^2$

  - Polyak’s stepsize rule: $\eta_t=\frac{f(x^t)-f^{\rm opt}}{\|g^t\|_2^2}\Rightarrow \|x^{t+1}-x^*\|_2^2\le\|x^{t}-x^*\|_2^2-\frac{(f(x^t)-f^{\rm opt})^2}{\|g^t\|_2^2}$

  - > 必须知道$f^{\rm opt}$, 所以并不实用

| problem                             | algorithm                    | stepsize rule               | convergence rate                                             | iteration complexity      |
| ----------------------------------- | ---------------------------- | --------------------------- | ------------------------------------------------------------ | ------------------------- |
| convex & Lipschitz problem          | projected subgradient method | $\eta_t=\frac{1}{\sqrt{t}}$ | $f^{{\rm best}，t}\mathbin{\lower.3ex\hbox{$\buildrel<\over{\smash{\scriptstyle\sim}\vphantom{_x}}$}} \frac{\|x^0-x^*\|_2^2+L_f\log t}{\sqrt{t}}$<br />$O(\frac{1}{\sqrt{t}})$ | $O(\frac{1}{\epsilon^2})$ |
| strongly convex & Lipschitz problem | projected subgradient method | $\eta_t=\frac{2}{\mu(t+1)}$ | $f^{{\rm best},t}-f^{\rm opt}\le \frac{2L_f^2}{\mu}\cdot\frac{1}{t+1}$<br />$O(\frac{1}{t})$ | $O(\frac{1}{\epsilon})$   |

## Proximal gradient methods

- composite models : $\begin{array}{*{20}{c}}
  {{\rm{minimize}}_x}&{F(x):=f(x)+h(x)}&{}\\
  {{\rm{subject\ to}}}&{x\in R^n}&{}
  \end{array}$, f convex and smooth, h convex$F^{\rm opt}:=\min_x F(x)$

  - $\mathcal{l}_1 $regularized minimization $\begin{array}{*{20}{c}}
    {{\rm{minimize}}_x}&{f(x)+\|x\|_1}
    \end{array}$,use $\mathcal{l}_1 $ regularization to promote sparsity.
  - nuclear norm  regularized minimization $\begin{array}{*{20}{c}}
    {{\rm{minimize}}_x}&{f(x)+\|x\|_*}
    \end{array}$,use nuclear norm regularization to promote low-rank structure.

- Proximal gradient descent

  - $x^{t+1}=\arg \min_x \left \{ f(x^t)+\langle \nabla f(x^t),x-x^t\rangle+\frac{1}{2\eta_t}\|x-x^t\|_2^2\right \}$

- projected proximal gradient  descent

  - $x^{t+1}=\arg \min_x \left \{ f(x^t)+\langle \nabla f(x^t),x-x^t\rangle+\frac{1}{2\eta_t}\|x-x^t\|_2^2\right \}+\mathbb{L}_\mathcal{C}(x)$

    ​         $=\arg \min _x \left\{ \frac{1}{2}\|x-(x^t-\eta_t \nabla f(x^t))\|_2^2+\eta_t c(x) \right\}$

    where $\mathbb{L}_\mathcal{C}(x)=\left\{ {\begin{array}{*{20}{c}}
      0&{{\rm{if\ }}x \in \mathcal{C}} \\ 
      \infty &{{\text{else}}} 
    \end{array}} \right.$

- proximal operator

  - ${\rm prox}_h(x):=\arg\min_z \left\{ \frac{1}{2}\|z-x\|_2^2 +h(z)\right\}$ , for any convex function $h$, may be non-smooth

  - Projected GD update $x^{t+1}={\rm prox}_{\eta_t \mathbb{L}_\mathcal{C} }(x^t-\eta_t \nabla f(x^t))$

  - accommodate more general $h$

    1. for $t=0,1,\cdots$ do
    2. $x^{t+1}={\rm prox}_{\eta_t h}(x^t-\eta_t \nabla f(x^t))$

  - > 在一般环境下定义良好，比如非光滑凸函数；能够被常用的函数有效评估，比如正则化函数；这个概念很简单，覆盖了很多众所周知的优化算法

  - basic rules

    - if $f(x)=ag(x)+b$ with $a>0$, then ${\rm prox}_f(x)={\rm prox}_{ag}(x)$
    - affine addition: $f(x)=g(x)+a^Tx+b$ , then ${\rm prox}_f(x)={\rm prox}_{g}(x-a)$
    - quadratic addition:  $f(x)=g(x)+\frac{\rho}{2}\|x-a\|_2^2$, then ${\rm prox}_f(x)={\rm prox}_{\frac{1}{1+\rho}g}(\frac{1}{1+\rho}x+\frac{\rho}{1+\rho}a)$ 
    - scaling and translation : $f(x)=g(ax+b)$  with $a \ne 0$ , then ${\rm prox}_f(x)=\frac{1}{a}({\rm prox}_{a^2g}(ax+b)-b)$
    - orthogonal mapping: $f(x)=g(Qx)$ with $Q^TQ=QQ^T=I$, then ${\rm prox}_f(x)=Q^T{\rm prox}_{g}(Qx)$
    - orthogonal affine mapping:  $f(x)=g(Qx+b)$ with $QQ^T=\alpha^{-1}I$, then  ${\rm prox}_f(x)=(I=\alpha Q^TQ)x+\alpha Q^T({\rm prox}_{\alpha^{-1}g}(Qx+b)-b)$
    - norm composition: $f(x)=g(\|x\|_2)$, then ${\rm prox}_f(x)={\rm prox}_{g}(\|x\|_2)\frac{x}{\|x\|_2}, \forall x \ne 0$

  - firm nonexpansiveness(非膨胀性) $\langle {\rm prox}_h(x_1)-{\rm prox}_{h}(x_2),x_1-x_2\rangle \ge \|{\rm prox}_h(x_1)-{\rm prox}_{h}(x_2)\|^2_2$

  - nonexpansiveness $\|{\rm prox}_h(x_1)-{\rm prox}_{h}(x_2)\|_2\le \|x_1-x_2\|_2$

  - interpret prox via resolvant of subdifferential operator $z={\rm prox}_f(x) \Leftrightarrow z=(\mathcal{I}+ \partial f)^{-1}(x)$ , where $\mathcal{I} $ is identity mapping  ($\mathcal{I}(z)=z$)

  - **Moreau decomposition**

    - Suppose $f$ is closed convex, and $f^*(x):= \sup_z\{ \langle x,z\rangle -f(z)\}$ is **convex conjugate** of $f$, then $x={\rm prox}_f(x)+{\rm prox}_{f^*}(x)$

    > proximal mapping 和 duality的关键联系
    >
    > generalization of orthogonal decomposition

    - $x=\mathcal{P}_\mathcal{K}(x)+\mathcal{P}_\mathcal{K^{\circ}}(x)$, where $K^{\circ}:=\{x|\lang x,z\rang \le 0, \forall z \in \mathcal{K}\}$

    - extended Moreau decomposition

      Suppose $f$ is closed convex and $\lambda >0 $, then $x={\rm prox}_{\lambda f}(x)+\lambda {\rm prox}_{\frac{1}{\lambda}f^*}(x/\lambda)$

  - > slides 中有很多函数的分解，还没仔细看

- Backtracking line search for proximal gradient methods

  Let $\tau_L(x):={\rm prox}_{\frac{1}{L}h}(x-\frac{1}{L}\nabla f(x))$

  1. Initialize $\eta=1,0<\alpha\le 1/2,0<\beta<1$
  2. while $f(\tau_{L_t}(x^t))>f(x^t)-\lang \nabla f(x^t),x^t-\tau_{L_t}(x^t)\rang+\frac{L_T}{2}\|\tau_{L_t}(x^t)-x^t\|_2^2$ do
  3. ​     $L^t\leftarrow \frac{1}{\beta}L^t ({\rm or\ } \eta_t\leftarrow \beta \eta_t)$ 

  here, $\frac{1}{L_t}$ correspond to $\eta_t$, and $\tau_{L_t}(x^t)$ generalizes $x^{t+1}$

| problem                          | algorithm                    | stepsize rule        | convergence rate                                             | iteration complexity              |
| -------------------------------- | ---------------------------- | -------------------- | ------------------------------------------------------------ | --------------------------------- |
| convex & smooth problem          | proximal GD                  | $\eta_t=\frac{1}{L}$ | $F(x^t)-F^{\rm opt}\le \frac{L\|x^0-x^*\|_2^2}{2t}$<br />$O(\frac{1}{{t}})$ | $O(\frac{1}{\epsilon})$           |
| strongly convex & smooth problem | projected subgradient method | $\eta_t=\frac{1}{L}$ | $\|x^t-x^*\|_2^2\le \left( 1- \frac{\mu}{L}\right)^t \|x^0-x^*\|_2^2$<br />$O((1-\frac{1}{\kappa})^t)$ | $O(\kappa\log\frac{1}{\epsilon})$ |

## Accelerated gradient methods

> 问题：1. GD专注于每次迭代时改善损失，可能会目光短浅。2. GD可能有时候是锯齿形瞬变的
>
> 解决方法：1. 从历史迭代中探索信息 2. 增加缓冲器（比如momentum）产生更平滑的轨迹

- Heavy-ball method

  - $x^{t+1}=x^{t}-\eta_t \nabla f(x^t)+\theta_t(x^t-x^{t-1})$
  - add inertia to the “ball ” (i.e. include momentum term) to mitigate zigzagging
  - System matrix$\left[ \begin{gathered}
      {x^{t + 1}} - {x^*} \hfill \\
      {x^t} - {x^*} \hfill \\ 
    \end{gathered}  \right] = \left[ {\begin{array}{*{20}{c}}
      {(1 + \theta ){\mathbf{I}} - {\eta _t}\int_0^1 {{\nabla ^2}f({x_\tau }){\text{d}}\tau } }&{ - {\theta _t}{\mathbf{I}}} \\ 
      {\mathbf{I}}&0 
    \end{array}} \right]\left[ \begin{gathered}
      {x^t} - {x^*} \hfill \\
      {x^{t - 1}} - {x^*} \hfill \\ 
    \end{gathered}  \right]$

- Nesterov’s accelerated gradient methods

  - $x^{t+1}=y^t-\eta_t\nabla f(y^t)$

    $y^{t+1}=x^{t+1}+\frac{t}{t+3}(x^{t+1}-x^t)$

  - alternates between gradient updates and proper extrapolation (not a descent method(i.e. may not have $f(x^{t+1})\le f(x^t)$ ))

  - one of most beautiful and mysterious results in optimization

  - > 可以用微分方程解释，具体看slides 

- FISTA (Fast iterative shrinkage-thresholding algorithm(快速迭代收缩阈值算法))

  - $x^{t+1}={\rm prox}_{\eta_t h}(y^t-\eta_t\nabla f(y^t))$ 

    $y^{t+1}=x^{t+1}+\frac{\theta_t-1}{\theta_{t+1}}(x^{t+1}-x^t)$

    where $y^0=x^0,\theta_0=1$ and $\theta_{t+1}=\frac{1+\sqrt{1+4\theta_t^2}}{2}$ (momentum coefficient)

  - Rippling behavior: take $y^{t+1}=x^{t+1}+\frac{1-\sqrt{q}}{1+\sqrt{q}}(x^{t+1}-x^t)$ $q^*=1/\kappa$  

    - when $q>q^*$ : we underestimate momentum $\rightarrow$ slower convergence

    - when $q<q^*$ : we overestimate momentum ($\frac{1-\sqrt{q}}{1+\sqrt{q}}$ is large) $\rightarrow$ overshooting/ rippling behavior

    - >  $q=q^*$是最好的，但是由于$u,L$很难计算，导致$\kappa,p^*$很难计算，所以实际使用时需要自己调试

  - Adaptive restart

    - When certain criterion is met , restart running FISTA with $\begin{array}{*{20}{c}}
        {{x^0} \leftarrow {x^t}} \\ 
        {{y^0} \leftarrow {x^t}} \\ 
        {{\theta _0} \leftarrow 1} 
      \end{array}$
    - function scheme: restart when $f(x^t)>f(x^{t-1})$
    - gradient scheme: restart when $\lang\nabla f(y^{t-1}),x^t-x^{t-1}>0\rang$

| problem                        | algorithm                               | stepsize rule                                                | convergence rate                                             | iteration complexity                     |
| ------------------------------ | --------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ---------------------------------------- |
| strong convex & smooth problem | Heavy-ball method                       | $\eta_t=\frac{4}{(\sqrt{L}+\sqrt{\mu})^2}$<br />$\theta_t=\max\{|1-\sqrt{\eta_tL}|,|1-\sqrt{\eta_t\mu}|\}^2$ | $\left\| \left[ \begin{gathered}  {x^{t + 1}} - {x^*} \hfill \\  {x^t} - {x^*} \hfill \\ \end{gathered}  \right] \right \|_2 \leqslant {\left( {\frac{{\sqrt \kappa   - 1}}{{\sqrt \kappa   + 1}}} \right)^t}\left \| \left[ \begin{gathered}{x^1} - {x^*} \hfill \\  {x^0} - {x^*} \hfill \\ \end{gathered}  \right]\right \|_2$ | $O(\sqrt{\kappa}\log\frac{1}{\epsilon})$ |
| convex & smooth problem        | Nesterov’s accelerated gradient methods | $\eta_t=\frac{1}{L}$                                         | $f(x^t)-f^{\rm opt}\le  \frac{2L\|x^0-x^*\|_2^2}{(t+1)^2}$<br />$O(\frac{1}{t^2})$ | $O(\frac{1}{\sqrt{\epsilon}})$           |
| convex & smooth problem        | FISTA                                   | $\eta_t=\frac{1}{L}$                                         | $F(x^t)-F^{\rm opt}\le  \frac{2L\|x^0-x^*\|_2^2}{(t+1)^2}$<br />$O(\frac{1}{t^2})$ | $O(\frac{1}{\sqrt{\epsilon}})$           |
| strong convex & smooth problem | FISTA                                   | $\eta_t=\frac{1}{L}$                                         | $F(x^t)-F^{\rm opt}\le\left( 1-\frac{1}{\sqrt{\kappa}}\right )^t\left( F(x^0)-F^{\rm opt}+\frac{\mu \|x^0-x^*\|_2^2}{2}\right ) $<br />$O((1-\frac{1}{\sqrt{\kappa}})^t)$ | $O(\sqrt{\kappa}\log\frac{1}{\epsilon})$ |

## Smoothing for nonsmooth optimization

> 非光滑导致输出subgradient成为一个first order oracle (black box model), 所以所以不能提升收敛速度
>
> 解决方法： approximate a nonsmooth objective  function by a smooth function

- Smooth approximation

  A convex function $f$ is called $(\alpha,\beta)$-smoothable if , for any $\mu>0,\exist$ convex function$f_\mu$ s.t.

  - $f_\mu(x)\le f(x)\le f_\mu(x)+\beta\mu, \forall x$
  - $f_\mu$ is $\frac{\alpha}{\mu}$-smooth

  here, $f_\mu$ is called $\frac{1}{\mu}$-smooth approximation of f with parameter $(\alpha,\beta)$, $\mu$ is tradeoff between approximation accuracy and smoothness 

  - Examples
    - $\|x\|_1 \Rightarrow f_\mu(x):=\sum\nolimits_{i = 1}^n {{h_\mu }({x_i})} ,{h_\mu }(z) = \left\{ {\begin{array}{*{20}{c}}
        {{z^2}/2\mu }&{{\text{if }}|z|\le \mu} \\ 
        {|z| - \mu /2}&{{\text{else}}} 
      \end{array}} \right.$(Huber function)
    - $\|x\|_2 \Rightarrow f_\mu(x):=\sqrt{\|x\|_2^2+\mu^2}-\mu$
    - $\max_i{x_i}\Rightarrow f_\mu(x):=\mu \log (\sum\nolimits_{i = 1}^ne^{x^i/\mu})-\mu \log n$
  - basic rules
    - addition $\lambda_1f_1+\lambda_2 f_2 \Rightarrow \lambda_1f_{\mu,1}+\lambda_2 f_{\mu,2}$ with parameters $(\lambda_1\alpha_1+\lambda_2 \alpha_2,\lambda_1\beta_1+\lambda_2 \beta_2)$
    - affine transformation  $h(Ax+b) \Rightarrow h_\mu(Ax+b)$ with parameters $(\alpha_1+\lambda_2 \alpha\|A\|^2,\beta)$

- Smoothing via Moreau envelope

  Moreau envelope (or Moreau-Yosida regularization) of convex function $f$ with parameter $\mu>0$ is defined as

  $M_{\mu f}(x):=\inf_z\left\{f(z)+\frac{1}{2\mu}\|x-z\|_2^2\right\}$

  - $M_{\mu f}$ is smoothed or regularized form of $f$
  - minimizers of $f$=minimizers of $M_f$ $\Rightarrow$ minimizing $f$ and $M_f$ are equivalent
  - Connection with proximal operater
    - $M_{\mu f}=f({\rm prox_{\mu f}(x)})+\frac{1}{2\mu}\|x-{\rm prox_{\mu f}(x)}\|$
    - ${\rm prox_{\mu f}(x)}=x-\mu\nabla M_{\mu f}(x)$
  - Properties of Moreau envelope
    - $M_{\mu f}$ is convex
    - $M_{\mu f}$ is $\frac{1}{\mu}$-smooth
    - if $f$ is $L_f$-Lipschitz, then $M_{\mu f}$ is $\frac{1}{\mu}$-smooth approximation of $f$ with parameters $(1,L_f^2/2)$

- Smoothing via conjugation

  Suppose $f=g^*$, namely, $f(x)=\mathop \sup \limits_{z}\{\lang z,x\rang-g(z)\}:=g^*(x)$

  $f_\mu(x)=\mathop \sup \limits_{z}\{\lang z,x\rang-g(z)-\mu d(z)\}=(g+\mu d)^*(x)$

  for some 1-strong convex and continuous function $d$ , called proximity function(近邻函数)

  - Properties
    - $g+\mu d$ is $\mu$-strongly convex $\Rightarrow$ $f_\mu$ is$\frac{1}{\mu}$-smooth
    - $f_\mu(x)\le f(x)\le f_\mu(x)+\mu D$ with $D:=\sup_xd(x)$ $\Rightarrow$ $f_\mu$ is$\frac{1}{\mu}$-smooth approximation of $f$ with parameters $(1,D)$
| problem                                               | algorithm | parameter                     | iteration complexity    |
| ----------------------------------------------------- | --------- | ----------------------------- | ----------------------- |
| $\frac{1}{\mu}$-smooth with $(\alpha, \beta)$ problem | FISTA     | $\mu=\frac{\epsilon}{2\beta}$ | $O(\frac{1}{\epsilon})$ |

> iteration complexity 从subgradient的$O(\frac{1}{\epsilon^2})$提升到了$O(\frac{1}{\epsilon})$

## Dual and primal-dual methods

- Dual formulation $\begin{array}{*{20}{c}}
  {\mathop {\rm{minimize}}\limits_{x}}&{f(x)+h(Ax)}&{}\\
  \end{array} \Rightarrow \begin{array}{*{20}{c}}
  {\mathop {\rm{minimize}}\limits_{x,z}}&{f(x)+h(z)}&{}\\
  {{\rm{subject\ to}}}&{Ax=z}&{}\\
  \end{array} \Rightarrow \begin{array}{*{20}{c}}
  {{\rm{maximize}}_\lambda \mathop \min \limits_{x,z}}&{f(x)+h(z)+\lang \lambda, Ax-z\rang}&{}
  \end{array}$

  $\Rightarrow 
  {{\rm{maximize}}_\lambda \mathop \min \limits_{x,z}}\{\lang A^T\lambda, x\rang+f(x)\}+{\mathop \min \limits_{z}}\{h(z)-\lang \lambda, z\rang\} $

  $\Rightarrow \begin{array}{*{20}{c}}
  {{\rm{minimize}}_\lambda}&{f^*(-A^T\lambda)-h^*(\lambda)}
  \end{array}$

  - primal : $\begin{array}{*{20}{c}}
    {\mathop {\rm{minimize}}\limits_{x}}&{f(x)+h(Ax)}&{}\\
    \end{array}$
  - dual: $\Rightarrow \begin{array}{*{20}{c}}
    {{\rm{minimize}}_\lambda}&{f^*(-A^T\lambda)-h^*(\lambda)}
    \end{array}$
    - if $f^*$ is smooth or strongly convex
    - proximal operator w.r.t. h is cheap

- Dual proximal gradient algorithm

  1. for $t=0,1,\cdots$ do
  2. ​      $\lambda^{t+1}={\rm prox}_{\eta_th^*}(\lambda^t+\eta_tA\nabla f^*(-A^T\lambda^t))$

- Primal representation of dual proximal gradient algorithm

  1. for $t=0,1,\cdots$ do
  2. ​      $x^t=\arg \min_x\{f(x)+\lang A^T\lambda^t, x\rang\}$
  3. ​       $\lambda^{t+1}=\lambda^t+\eta_tAx^t-\eta_t{\rm prox}_{\eta_t^{-1}h^*}(\eta_t^{-1}\lambda^t+Ax^t)$

- Accelerated dual proximal gradient algorithm

  1. for $t=0,1,\cdots$ do
  2. ​      $\lambda^{t+1}={\rm prox}_{\eta_th^*}(w^t+\eta_tA\nabla f^*(-A^Tw^t))$
  3. ​      $\theta_{t+1}=\frac{1+\sqrt{1+4\theta^3_t}}{2}$
  4. ​       $w^{t+1}=\lambda^{t+1}+\frac{\theta_t-1}{\theta_{t+1}}(\lambda^{t+1}-\lambda^{t})$

- Primal representation of accelerated  dual proximal gradient algorithm

  1. for $t=0,1,\cdots$ do
  2. ​    $x^t=\arg \min_x\{f(x)+\lang A^T\lambda^t, x\rang\}$
  3. ​    $\lambda^{t+1}=w^t+\eta_tAx^t-\eta_t{\rm prox}_{\eta_t^{-1}h^*}(\eta_t^{-1}w^t+Ax^t)$
  4. ​    $\theta_{t+1}=\frac{1+\sqrt{1+4\theta^3_t}}{2}$
  5. ​     $w^{t+1}=\lambda^{t+1}+\frac{\theta_t-1}{\theta_{t+1}}(\lambda^{t+1}-\lambda^{t})$

- Primal-dual proximal gradient method

  $\begin{array}{*{20}{c}}
  {\mathop {\rm{minimize}}\limits_{x}}&{f(x)+h(Ax)}&{}\\
  \end{array}$where $f$ and $h$ are closed and convex, both $f$ and $h$ might be non-smooth, both $f$ and $h$ admit inexpensive proximal operators

  > 我们仅仅已经讨论了proximal method (resp. dual proximal method) ,但是他们只能更新primal(resp. dual )变量。我们能利用${\rm prox}_f$ 和${\rm prox}_h$从而同时更新primal和dual变量吗？

  - Saddle-point problem ${\rm minimize}_x \max_\lambda f(x)+\lang \lambda ,Ax\rang-h^*(\lambda)$

    > saddle points : $\forall x\in X,y\in Y, f(x^*,y)\le f(x^*,y^*)\le f(x,y^*)$

    - optimality condition $0 \in \left[ {\begin{array}{*{20}{c}}
        {}&{{A^T}} \\ 
        { - A}&{} 
      \end{array}} \right]\left[ \begin{gathered}
        x \hfill \\
        \lambda  \hfill \\ 
      \end{gathered}  \right] + \left[ \begin{gathered}
        \partial f(x) \hfill \\
        \partial {h^*}(\lambda ) \hfill \\ 
      \end{gathered}  \right]: =\mathcal{F} (x,\lambda )$

    - fixed-point iteration/resolvent iteration $x^{t+1}={(\mathcal{I}+\eta\mathcal{F})^{-1}(x^t)}$

      > issue: 计算$(\mathcal{I}+\eta\mathcal{F})^{-1}$太难了

    - $\mathcal{A}(x,\lambda):=\left[ {\begin{array}{*{20}{c}}
        {}&{{A^T}} \\ 
        { - A}&{} 
      \end{array}} \right]\left[ \begin{gathered}
        x \hfill \\
        \lambda  \hfill \\ 
      \end{gathered}  \right]$, $\mathcal{B}(x,\lambda):=\left[ \begin{gathered}
        \partial f(x) \hfill \\
        \partial {h^*}(\lambda ) \hfill \\ 
      \end{gathered}  \right]$

      > solution: 将$(\mathcal{I}+\eta\mathcal{F})^{-1}$拆分成$(\mathcal{I}+\eta\mathcal{A})^{-1}$和$(\mathcal{I}+\eta\mathcal{B})^{-1}$分别解线性问题和$\rm prox$

    - operator splitting via Cayley operators

      Let $\mathcal{R}_\mathcal{A}:=(\mathcal{I}+\eta\mathcal{A})^{-1}$, $\mathcal{R}_\mathcal{B}:=(\mathcal{I}+\eta\mathcal{B})^{-1}$ be resolvents and $\mathcal{C}_\mathcal{A}:=(2\mathcal{R}_\mathcal{A}-\mathcal{I})$, $\mathcal{C}_\mathcal{B}:=(2\mathcal{R}_\mathcal{B}-\mathcal{I})$ be Cayley operators

      then 

      $0\in \mathcal{A}(x)+\mathcal{B}(x) \Leftrightarrow \mathcal{C}_\mathcal{A}\mathcal{C}_\mathcal{B}(z)=z$ with $x=\mathcal{R}_\mathcal{B}(z)$

      > 这个的证明。。。没看懂
  >
      > issue： 怎么求解$\mathcal{C}_\mathcal{A}\mathcal{C}_\mathcal{B}(z)$呢 直接用$z^{t+1}=\mathcal{C}_\mathcal{A}\mathcal{C}_\mathcal{B}(z^t)$可能不会收敛
  
    - Douglas-Rachford splitting（damped fixted-point iteration）
    
        $z^{t+1}=\frac{1}{2}(\mathcal{I}+\mathcal{C}_\mathcal{A}\mathcal{C}_\mathcal{B})(z^t)$
    
        > expilcit expression (更清晰的阐释)
        >
        > $x^{t+\frac{1}{2}}=\mathcal{R}_\mathcal{B}(z^t)$
        >
        > $z^{t+\frac{1}{2}}=2x^{t+\frac{1}{2}}-z^t$
        >
        > $x^{t+1}=\mathcal{R}_\mathcal{A}(z^{t+\frac{1}{2}})$
        >
        > $z^{t+1}=\frac{1}{2}(z^t+2x^{t+1}-z^{t+\frac{1}{2}})=z^t+x^{t+1}-x^{t+\frac{1}{2}}$
        >
        > 其中$x^{t+\frac{1}{2}},z^{t+\frac{1}{2}}$都是辅助变量
    
        applying Douglas-Rachford splitting to optimality condition
    
        $x^{t+\frac{1}{2}}={\rm prox}_{\eta f}(p^t)$
    
        $\lambda^{t+\frac{1}{2}}={\rm prox}_{\eta h^*}(q^t)$
    
        $\left[ \begin{gathered}
          {x^{t + 1}} \hfill \\
          {\lambda ^{t + 1}} \hfill \\ 
        \end{gathered}  \right] = {\left[ {\begin{array}{*{20}{c}}
          I&{\mu {A^T}} \\ 
          { - \eta A}&I 
        \end{array}} \right]^{ - 1}}  \left[ \begin{gathered}
          2{x^{t + \frac{1}{2}}} - {p^t} \hfill \\
          2{\lambda ^{t + \frac{1}{2}}} - {q^t} \hfill \\ 
        \end{gathered}  \right]$
    
        $p^{t+1}=p^t+x^{t+1}-x^{t+\frac{1}{2}}$
    
        $q^{t+1}=q^t+\lambda^{t+1}-\lambda^{t+\frac{1}{2}}$

## Alternating direction method of multipliers (ADMM)

- Two-block problem

   $\begin{array}{*{20}{c}}
  {\mathop {\rm{minimize}}\limits_{x,z}}&{F(x,z):=f_1(x)+f_2(z)}&{}\\
  {{\rm{subject\ to}}}&{Ax+Bz=b}&{}\\
  \end{array}$

  > 这也可以用Douglas-Rachford splitting求解

- Augmented Lagrangian method

  - Dual problem : ${\rm minimize}_\lambda f_1^*(-A^T\lambda)+f_2^*(-B^T\lambda)+\lang\lambda,b\rang$

  - Proximal point method $\lambda^{t+1}=\arg \min_\lambda \left\{ f_1^*(-A^T\lambda)+f_2^*(-B^T\lambda)+\lang\lambda,b\rang +\frac{1}{2\rho}\|\lambda -\lambda^t\|_2^2\right\}$

  - Augmented Lagrangian method (or method for multipliers)

    $ (x^{t+1},z^{t+1})=\arg \min_{x,z}\left\{ f_1(x)+f_2(z)+\frac{\rho}{2}\|Ax+Bz-b+\frac{1}{\rho}\lambda^t\|_2^2\right\} $ (primal step)

    $\lambda^{t+1}=\lambda^{t}+\rho(Ax^{t+1}+Bz^{t+1}-b)$   (dual step)

    > primal step通常是expensive，就像解决原问题一样
    >
    > $x,z$的最小化不能分开进行

- Alternating direction method of multipliers 

  $ x^{t+1}=\arg \min_{x}\left\{ f_1(x)+\frac{\rho}{2}\|Ax+Bz^t-b+\frac{1}{\rho}\lambda^t\|_2^2\right\} $

  $ z^{t+1}=\arg \min_{z}\left\{f_2(z)+\frac{\rho}{2}\|Ax^{t+1}+Bz-b+\frac{1}{\rho}\lambda^t\|_2^2\right\} $

  $\lambda^{t+1}=\lambda^{t}+\rho(Ax^{t+1}+Bz^{t+1}-b)$

  > 混合了对偶分解(dual decomposition )和增强拉格朗日( Augmented Lagrangian method)方法的好处
  >
  > $x,z$是接近对称的但是不是一个整体

- robust PCA

  $M=L$ (low-rank) $+S$(sparse)

  > 怎么将一个矩阵$M$分解成一个低秩矩阵和系数矩阵的叠加(superposition)
  
  - convex programing
  
    $\begin{array}{*{20}{c}}
    {\mathop {\rm{minimize}}\limits_{L,S}}&{\|L\|_*+\lambda\|S\|_1}&{}\\
    {{\rm{subject\ to}}}&{L+S=M}&{}\\
    \end{array}$
  
    where $\|L\|_*:=\sum\nolimits_{i = 1}^n {{\sigma _i}(L)} $ is nuclear norm, and $\|S\|_1:=\sum\nolimits_{i,j} {|S_{i,j}|} $ is enteywise $l_1$ norm
  
  - ADMM for solving
  
    $ L^{t+1}=\arg \min_{L}\left\{ \|L\|_*+\frac{\rho}{2}\|L+S^t-M+\frac{1}{\rho}\Lambda^t\|_F^2\right\} $
  
    $ S^{t+1}=\arg \min_{S}\left\{\lambda\|S\|_1+\frac{\rho}{2}\|L^{t+1}+S-M+\frac{1}{\rho}\Lambda^t\|_F^2\right\} $
  
    $\Lambda^{t+1}=\Lambda^{t}+\rho(L^{t+1}+S^{t+1}-M)$
  
    - is equivalent to 
  
      $ L^{t+1}={\rm SVT}_{\rho^{-1}}\left(M-S^t-\frac{1}{\rho}\Lambda^t\right) $    (singular value thresholding)
  
      $ S^{t+1}={\rm ST}_{\lambda \rho^{-1}}\left(M-L^{t+1}-\frac{1}{\rho}\Lambda^t\right) $   (soft thresholding)
  
      $\Lambda^{t+1}=\Lambda^{t}+\rho(L^{t+1}+S^{t+1}-M)$
  
      where for any $X$ with ${\rm SVD} X=U \Sigma  V^T (\Sigma={\rm diag}(\{\sigma_i\}))$, one has
  
      ${\rm SVT}_{\tau}(X)=U{\rm diag}(\{(\sigma_i-\tau)_+\})V^T$ and
  
      ${({\text{S}}{{\text{T}}_\tau }(X))_{i,j}} = \left\{ {\begin{array}{*{20}{c}}
        {{X_{i,j}} - \tau }&{{\text{if }}{X_{i,j}} > \tau } \\ 
        0&{{\text{if |}}{X_{i,j}}| \leqslant \tau } \\ 
        {{X_{i,j}} + \tau }&{{\text{if }}{X_{i,j}} <  - \tau } 
      \end{array}} \right.$
  
  - other examples 
  
    - graphical lasso $\begin{array}{*{20}{c}}
      {\mathop {\rm{minimize}}\limits_{\Theta}}&{-\log \det\Theta+\lang\Theta ,S\rang+\lambda\|\Theta\|_1}&{}\\
      {{\rm{subject\ to}}}&{\Theta \succeq 0}&{}\\
      \end{array}$(估计稀疏高斯图形模型)
    - consensus optimization（共识优化）
  
  - conbergence rate: $O(1/t)$
  
    iteration complexity $O(1/\epsilon)$
  
    > 收敛性证明部分抽空一步一步推导一下

## Quasi-Newton methods

- Newton’s method 
  - $x^{t+1}=x^t-(\nabla^2f(x^t))^{-1}\nabla f(x^t)$
  - Quadratic convergence $O(\log\log\frac{1}{\epsilon})$

- Quasi-Newton method $x^{t+1}=x^t-\eta_tH_t\nabla f(x^t)$

- BFGS(Broyden-Fletcher-Goldfarb-Shanno) method

  1. for $t=0,1,\cdots$ do 

  2. ​      $x^{t+1}=x^t-\eta_tH_t\nabla f(x^t)$ (line search to determin $\eta_t$)

  3. ​      $H_{t+1}=(I-\rho_ts_ty_t^T)H_t(I-\rho_ty_ts_t^T)+\rho_ts_ts_t^T$

     where $s_t=x^{t+1}-x^t,y_t=\nabla f(x^{t+1})-\nabla f(x^{t}),\rho_t=\frac{1}{y_t^T s_t}$

  - iteration cost:  gradient method <$O(n^2)$<newton method $O(n^3)$
  - no magic formula for initization; possinle choices: approximate inverse Hessian at $x^0$, or identity matrix.

- Computational complexity = iteration cost $\times$ iteration complexity

  iteration complexity : gradient method $O(\log(\frac{1}{\epsilon}))$ >BFGS>Newton method $O(\log\log(\frac{1}{\epsilon}))$

- L-BFGS (Limited-memory BFGS)

  $H_t^L=V_{t-1}^T\cdots V_{t-m}^TH^L_{t,0}V_{t-m}\cdots V_{t-1}$

  ​            $+\rho_{t-m}V_{t-1}^T\cdots V_{t-m+1}^Ts_{t-m}s_{t-m}^TV_{t-m+1}\cdots V_{t-1}$

  ​            $+\rho_{t-m+1}V_{t-1}^T\cdots V_{t-m+2}^Ts_{t-m+1}s_{t-m+1}^TV_{t-m+2}\cdots V_{t-1}$

  ​             $+\cdots+\rho_{t-1}s_{t-1}s_{t-1}^T$  

  ​    where $V_t=(I-\rho_t y_ts_t^T)$

  - can be computed recursively (递归的)
  - intialization $H^L_{t,0}$ may vary from iteration to iteration.
  - only needs to store $\{(s_i,y_i)\}_{t-m\le i<t}$

## Stochastic gradient methods 

- stochastic approximation / stochastic gradient descent (SGD)

  $x^{t+1}=x^t-\eta_t g(x^t;\xi^t)$

  where $g(x^t;\xi^t)$ is unbiased estimate of $\nabla F(x^t)$, i.e. $\mathbb{E}[g(x^t;\xi^t)]=\nabla F(x^t)$ 

  - $\nabla F(x^t)=0 \Rightarrow$ finding roots of $G(x):=\mathbb{E}[g(x^t;\xi^t)]$ 
  - Examples
    - SGD for empirical risk minimization $x^{t+1}=x^t-\eta_t \nabla_xf_{i_t}(x^t;\{a_i,y_i\})$ (choose $i_t$ uniformly at random)
    - temporal difference (TD) learning: Reinforcement learning studies Markov decision process (MDP) with unknown model.
    - Q-learning: solve Bellman equation
  - $\mathbb{E}[\| g(x^t;\xi^t)\|_2^2]\le \sigma_g^2+c_g \|\nabla F (x)|_2^2$

- iterate averaging

  > 没看懂它推导出来的结果是用来干什么的，slides中说是减轻震荡和减少方差，但是我没看懂怎么减少的，下一个sildes应该会讲。

| problem                        | algorithm | stepsize rule                                            | convergence rate                                             | iteration complexity                     |
| ------------------------------ | --------- | -------------------------------------------------------- | ------------------------------------------------------------ | ---------------------------------------- |
| strong convex & smooth problem | SGD       | $\eta_t<\frac{1}{Lc_g}$                                  | $\mathbb{E}[F(x^t)-F(x^*)] \le \frac{\eta L \sigma_g^2}{2\mu}+(1-\eta \mu)^t(F(x^0)-F(x^*))$ | $O(\sqrt{\kappa}\log\frac{1}{\epsilon})$ |
| strongly convex problem        | SGD       | $\eta_t=\frac{\theta}{t+1}$ for $\theta >\frac{1}{2\mu}$ | $\mathbb{E}[\|x^t-x^*\|_2^2] \le \frac{c_\theta}{t+1}$<br />where $c_\theta=\max\{\frac{2\theta^2 \sigma_g^2}{2\mu\theta-1},\|x^0-x^*\|_2^2\}$ | $O(\frac{1}{\sqrt{\epsilon}})$           |



需要整理

1. norm
2. 线性收敛，log收敛速度,算法复杂度
3. 

$\max x_i \le f(\mathbf{x})\le \max x_i+\log n, f(\mathbf{x})=(\prod\nolimits_{i = 0}^n x_i )^{1/n}$

Gradient: $\nabla f(\mathbf{x})=[\frac{{\delta f(\mathbf{x})}}{{\delta x_1}} \cdots \frac{{\delta f(\mathbf{x})}}{{\delta x_n}}]^T\in R^n$

Hessian: $\nabla^2f(\mathbf{x})=\left (\frac{{\delta^2 f(\mathbf{x})}}{{\delta x_i\delta x_j}}\right )_{ij}\in R^{n\times n}$

Taylor series $f(\mathbf{x}+ \mathbf{\delta})=f(x)+\nabla f(\mathbf{x})^T\mathbf{\delta}+\frac{1}{2}\mathbf{\delta}^T\nabla^2f(\mathbf{x})\mathbf{\delta}+o(\left\|\mathbf{\delta}\right\|^2)$



$\left\| I-\eta Q\right\|=max\{|1-\eta\lambda_1(Q)|,|1-\eta\lambda_n(Q)|\}$

w.r.t 关于，iff 当且仅当, a.k.a. 又名 resp. 分别的，分开的