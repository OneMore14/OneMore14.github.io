---
layout: post
title:  "论文阅读 proofs of space"
date:   2021-07-07 19:28:01 +0800
categories: 区块链
---

# proofs of space

非权威解读，仅用作个人备忘，请谨慎参考.

论文原文: https://eprint.iacr.org/2013/796.pdf

## 1. Introduction

## 2 Defining Proofs of Space

PoS需要两台机器，一台作为prover P，另一台作为verifier V.
PoS协议需要运行在两个阶段: PoS初始化和PoS执行.
协议运行需要一些id作为P和V的共同输入，identifier id保证P不能重复使用同一块空间作验证。


**初始化** 初始化需要给P和V一些公共的输入，比如identifier id、存储大小N等
P和V会有各自的输出。当P是欺诈节点时，V可能直接终止

**执行阶段** P和V会根据初始化使的结果再继续执行，此阶段P没有输出，V只会输出两种结果: 拒绝或接受。

再引入一种称为dishonest prover的概念，

**Completeness:** 对所有诚实的P，V都应该输出接受

**Soundness:** 对不诚实的P，应该有安全相关的入参使其被V接受的概率极低

**Efficiency:** V计算是否接受P的时间应该很少，针对容量N是最多logN的多项式时间，针对安全参数应该是多项式时间

## 3 The model

<script type="text/javascript" src="http://cdn.mathjax.org/mathjax/latest/MathJax.js?config=default"></script>
