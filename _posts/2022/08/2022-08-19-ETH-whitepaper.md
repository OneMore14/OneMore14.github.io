---
layout: post
title:  "Ethereum Whitepaper notes"
date:   2022-08-19 15:09:01 +0800
tags: "blockchain"
typora-root-url: ../../../
---



whitepaper pdf: https://ethereum.org/669c9e2e2027310b6b3cdce6e1c52962/Ethereum_Whitepaper_-_Buterin_2014.pdf



为了方便，ETH有时候代表以太网Ethereum，有时候代表以太Ether，还是比较好区分的。



## ETH



### Ethereum Accounts

In Ethereum, the state is made up of objects called "accounts", with each account having a 20-byte address and state transitions being direct transfers of value and information between accounts

一个账号包含4个属性

* The nonce, a counter used to make sure each transaction can only be processed once
* The account's current ether balance
* The account's contract code, if present
* The account's storage (empty by default)

账号分两种: externally owned accounts,   contract accounts

in a contract account, every time the contract account receives a message its code activates.



###  Messages and Transactions

message像信封里信的内容，transaction像一个包含有信件，贴好了邮票的信封实体.



Message在以太中指消息本身，比如一条转账消息从一个账号到另一个账号，转移了多少以太，这些都是消息. 消息可以由external entity或者contract制造；消息也可以携带data信息，即带上一串二进制数据；如果消息的接收者是contract account，那么也可能会有一个返回值。



The term "transaction" is used in Ethereum to refer to the signed data package that stores a message to be sent from an externally owned account. Transactions contain the recipient of the message, a signature identifying the sender, the amount of ether and the data to send.  

Transaction还有STARTGAS和GASPRICE两个参数。EVM执行每一条指令都需要消耗固定数量的gas，比如乘法MUL要5个gas，加法ADD要3个gas，STARTGAS规定了tx最多能消耗多少个gas，GASPRICE就是每个gas值多少ETH。

如果STARTGAS不够，报错runs out of gas，所有操作revert，但gas费不会返回；如果程序运行到某个位置结束，那么剩余gas费会返还给sender.



合约在ETH中是一等公民。



### Code Execution

ETH中代码执行也是堆栈模型。ETH的代码有三处可以访问空间

* stack 保存32byte的值
* memory， 可无限扩展
* the contract's long-term storage， kv存储，kv都是32bytes，和stack和memory不同，可永久保存
