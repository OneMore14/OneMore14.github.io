---
layout: post
title:  "Filecoin经济模型(未完)"
date:   2021-06-02 14:41:01 +0800
categories: filecoin
---

对原文[Engineering Filecoinʼs Economy](https://filecoin.io/2020-engineering-filecoins-economy-en.pdf) 的笔记 + 个人理解解读

## 1. The future of the data storage and distribution industry

主要讲了Filecoin网络存储对比现在云存储的好处,总结就是去中心化能提高网络利用率、开放平台能提供更丰富的功能、通过协议可以验证自己的文件是否还存在等等.将Filecoin的作用类比Airbnb，Airbnb将分散的个人房源联合起来可以与酒店巨头竞争,
那么Filecoin也可以将个体的空余磁盘联合起来挑战集中式云盘。

## 2. The Filecoin Economy
### 2.1 A Market for Data
  Filecoin网络提供了存储文件的市场，Filecoin本身作为一种native token存在，用户存储或取出数据都需要支付FIL。
  
### 2.2 An Export Economy
  作者讲述Filecoin网络内只使用FIL作为货币，参与者只能自己将法币兑换为FIL后使用。FIL能有助于Filecoin网络的
  发展，但FIL的供应不宜增长太快，否则对Filecoin经济有害。理想情况下，FIL的供应应该和Filecoin网络创造的价值大致相同。
  
对全体参与者来说，最重要的目标是让系统尽可能地有效且吸引更多人参与。一个更高效率创造更多价值的经济将引领对商品(这里指Filecoin的存储服务)的更多需求，以及对FIL的需求。
现在只一心利用Filecoin投机的人将赶走有真实存储需求的人，破坏Filecoin的生态。Filecoin网络需要有真实的客户需求来让挖矿长期有利可图。

### 2.3 Participants in the Filecoin Economy
Developer/Client/Miner/Token Holder/Ecosystem Partner，最后一个partner更像是有专业背景的社区/成员，但不是Filecoin开发者
  
其次，各成员应该着眼于长远利益，才能促进Filecoin的良性发展。

## 3. Storage Services on Filecoin
### 3.1 What is a sector?
committed capacity: 一个sector中还未使用，可以服务用户的容量。

Committed capacity sectors: 一整块sector全部可用

Sectors是Filecoin网络的基本存储单位，大小固定。不同的交易(deal)不区分client和miner是否相同(self-dealing),
但committed capacity让self-dealing在经济上是不理智的。

(TODO: 关于这一节上面的解释不确定了，暂时不看这篇文章，补一下Filecoin spec)