---
layout: post
title: "FISCO-BCOS源码阅读——开始篇"
description: ""
categories: ["FISCO BCOS"]
tags: [C++,Blockchain]
redirect_from:
  - /2020/12/01/
typora-root-url: ..
---



* Kramdown table of contents
{:toc .toc}
### 前言

从我之前的博客也可以看出，最近在学习区块链，目前我的研究方向也是在物联网中区块链+联邦学习。但是目前网上的博客，书籍，绝大部分都是对基本概念的介绍，很多细节性的东西我都很不理解，比如区块链在各个节点之间到底是怎么进行数据同步的？这种非常细节性的问题只能从源代码中进行学习了。所以我产生了阅读源码的念头。

但是目前最火的区块链平台，以太坊，Hyperledger Fabric, 以及国内的FISCO BCOS。相关比较可以参考[Hyperledger Fabric和FISCO BCOS技术对比_极速蜗蜗的博客-CSDN博客](https://blog.csdn.net/wxudong1991/article/details/109311501)。我最终毫无疑问的选择了FISCO BCOS进行源码阅读。理由如下

**优点：**

- 它是国产的，完全支持国产加密算法，国产操作系统。
- 它的共识算法支持pbft/raft/rpbft，而Hyperledger Fabric目前只支持raft。目前我的科研使是用的PBFT
- 它是用C++写的，暂时不用先学习Go语言才能看源代码（虽然我的C++也不怎么样）

**缺点：**

- 它是国产的，所以生态还没搭建起来。用户群较小
- 目前发展不够完善，没有手把手那种的教程资料，只有官方手册，不过我觉得这也就够了

### 可以参考的资料

- [FISCO BCOS 技术文档 — FISCO BCOS v2.7.0 文档 (fisco-bcos-documentation.readthedocs.io)](https://fisco-bcos-documentation.readthedocs.io/zh_CN/release-2.7.0/index.html)
- [炼就纯熟区块链开发技能，看这一篇就够了 (qq.com)](https://mp.weixin.qq.com/s?__biz=MzA3MTI5Njg4Mw==&mid=100003088&idx=1&sn=f36770aaecc081baf2fd8167af43d563&chksm=1f2eff0c2859761ab6f03f305f4b3139fb5df4d468c9f9f177a591a956d119d9f78b6299ce96&mpshare=1&scene=23&srcid=1201TC4G92bTgcxqIkEtEfqx&sharer_sharetime=1606810892120&sharer_shareid=89970315c44f2820655652f22c5827c2#rd)
- [带你读源码：四大视角多维走读区块链源码 (qq.com)](https://mp.weixin.qq.com/s?__biz=MzA3MTI5Njg4Mw==&mid=2247486327&idx=1&sn=1f84f80e2614eff5e01b99e9fa8ba95a&chksm=9f2ef96ba859707d6e0ab19c18c3edeed264322ae32aaebe0ef6adbaa50143f63ff324f875fb&scene=21#wechat_redirect)

### 规划

- 重要阅读笔记记录在博客
- 同时我会在源代码里注释，所有注释上传到[zpyang/FISCO-BCOS_code_reading (gitee.com)](https://gitee.com/zpyang/fisco-bcos_code_reading)，这里面我会根据文件夹写更加详细的阅读笔记。

### 初探

在啥都不知道的情况下，我先看了`/fisco-bcos/main/main.c`文件。发现只是用来启动各个服务的进程的。

然后通过在FISCO BCOS微信公众号的查找，成功的找到了[带你读源码：四大视角多维走读区块链源码 (qq.com)](https://mp.weixin.qq.com/s?__biz=MzA3MTI5Njg4Mw==&mid=2247486327&idx=1&sn=1f84f80e2614eff5e01b99e9fa8ba95a&chksm=9f2ef96ba859707d6e0ab19c18c3edeed264322ae32aaebe0ef6adbaa50143f63ff324f875fb&scene=21#wechat_redirect)，里面讲述了各个文件夹定义的类的作用。这不就完美啦，根据这个慢慢看就好了。

我将会慢慢慢慢的按照下面的顺序阅读源代码。。。这是个极其庞大的工程，我也不知道我能坚持多久。加油吧！

**基础层**：

<img src="/images/posts/2020-12-01/jichuceng.png" alt="jichuceng" style="zoom:50%;" />

**核心层**：

<img src="/images/posts/2020-12-01/hexinceng.png" alt="hexinceng" style="zoom:50%;" />

**借口层**：

<img src="/images/posts/2020-12-01/jiekouceng.png" alt="jiekouceng" style="zoom:50%;" />