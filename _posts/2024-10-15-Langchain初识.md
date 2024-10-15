---
title: Langchain初识

date: 2024-10-15 17:00:00 +0800

categories: [Langchain]

tags: [langchain]

description: 
---



# 前⾔ 

过去半年，随着ChatGPT的⽕爆，直接带⽕了整个LLM这个⽅向，然LLM毕竟更多是基于过去的经验数据 预训练 ⽽来，没法获取最新的知识，以及各企业私有的知识：

* 为了获取最新的知识，ChatGPT plus版集成了bing搜索的功能，有的模型则会调⽤⼀个定位于 “链接各种AI模型、⼯具的langchain”的bing功能 
* 为了处理企业私有的知识，要么基于开源模型微调，要么也可以基于langchain的思想调取⼀个外挂的向量知识库(类似存在本地的数据库⼀样) 

所以越来越多的⼈开始关注langchain，并把它与LLM结合起来应⽤，更直接推动了 数据库 、知识图谱与LLM的结合应⽤

---

本⽂侧重点：

* LLM与langchain/数据库/知识图谱的结合应⽤ 
  * ⽐如，虽说基于知识图谱的问答 早在2019年之前就有很多研究了，但谁会想到今年KBQA因为LLM如此突⻜猛进
  * ⽐如，还会解读langchain-ChatGLM项⽬的关键源码，不只是把它当做⼀个⼯具使⽤，因为对⼯具的原理更了解，则对⼯具的使⽤更顺畅
* 梳理 langchain-ChatGLM 代码

# 1 什么是Langchain?

## 1.1 整体架构





## 1.2 Langchain示例









