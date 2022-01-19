---
layout: post
title:  "coursera Cryptography I"
date:   2021-07-12 22:49:01 +0800
categories: 密码学
---

课程地址: https://www.coursera.org/learn/crypto/

# Week1

## The one-time pad and stream ciphers

定义cipher是高效率的一组算法$(E, D)$​使得$(k, m, c)$​​ (分别是key, message和ciphertext)满足

* $E: k * m = c$

* $D: k * c = m$​

  

The One Time Pad:  key和message都是由0或1组成的字符串，且长度相同，加密方法是key和message对应的每一位都做$XOR$​运算



secure cipher 在只有密文的情况下

* 攻击者不能推恢复密钥

* 攻击者不能恢复原文

  

cipher有完美安全性(perfect secrecy) 对所有两个长度相同的原文$m_0$和$m_1$，以及所有的密文$c$，如果我们随机选择一个一个$k$，有$Pr[E(k, m_0) = c] = Pr[E(k, m_1) = c]$​成立。也就是是说如果拿到一个密文，那么原文全是0的概率和原文全是1的概率是一样的。注意，有完美安全性的算法也可能遭受其他攻击。



OTP具有完美安全性



如果一个算法有perfect secrecy，那么$len(k) >= len(m)$​



PRG: pseudorandom generator

PRG predictable: 能从生成结果的前i位大概推导出下一位的值



判断是否negligible 

$\varepsilon$是一个从非负整数到非负实数的函数

* $\varepsilon$ non-neg  $\exist d: \varepsilon(\lambda) \ge \frac{1}{\lambda^d}$ 即对很多$\lambda$值，$\varepsilon$都比对应多项式导数大       eg. $\varepsilon(\lambda) = \frac{1}{\lambda^{1000}}$
* $\varepsilon$ negligible $\forall d, \lambda\ge\lambda_d, \varepsilon\le\frac{1}{\lambda^d}$ 即对大数值$\lambda$，$\varepsilon$都比多项式小          eg. $\varepsilon(\lambda) = \frac{1}{2^\lambda}$​



## Attacks and common mistaks

Attack 1:    two time pad是不安全的，如果拿到两个被同一密码加密的密文，这两个密文做异或会得到原文的异或值，足够解密。

Attack 2:    no integrity 攻击者截获密文虽然不能解码，但可以对密文再用自己的密钥对密文加密，使接收者解密后得不到正确信息 
