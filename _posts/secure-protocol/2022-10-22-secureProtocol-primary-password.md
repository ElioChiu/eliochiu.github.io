---
layout: "post"
title: "「安全协议」—— 基本协议"
subtitle: "基于口令的安全协议"
author: "eliochiu"
date: 2022-10-21

tags: ["安全协议@Tags"]
lang: zh
catalog: true
header-image: ""
header-style: text
katex: true
---

## 直接口令协议
主机$H$初始化用户$U$后，保存所有用户的身份和口令字典：$(ID_U, PW_U)$

协议流程：
- $U\rightarrow H:ID_U$
- $H\rightarrow U:$ "请输入口令"
- $U\rightarrow H:PW_U \prime$
- $H\rightarrow U:$ 验证$PW_U\prime == PW_U$

协议面临两类攻击：
- 数据库攻击：数据库存储所有用户的身份和口令的明文，一旦泄漏，后果很严重。
- 链路攻击：敌手在链路上直接监听用户的明文口令。

## 使用单向函数
主机$H$初始化用户$U$后，不再保存用户的明文口令，而是保存用户口令的单向函数值（*one-way function*），如哈希函数值。字典中存储的信息为：$(ID_U, OWF(PW_U))$：

协议流程：
- $U\rightarrow H:ID_U$
- $H\rightarrow U:$ "请输入口令"
- $U\rightarrow H:Token \leftarrow OWF(PW_U \prime)$
- $H\rightarrow U:$ 查找用户口令的单向函数值$OWF(PW_U)$，验证$ Token== OWF(PW_U)$

协议相比明文口令的改进：数据库中不再存储明文口令，而是口令的单向函数值，一定程度上减少了明文泄漏的危害。

协议面临两类攻击：
- 数据库攻击：敌手通常会建立一个常用攻击字典，爆破所有口令的单向函数值，和数据库的单向函数值一一比对。
- 链路攻击：敌手直接监听获取用户的$Token$，通过系统验证。

## 使用加盐单向函数
主机$H$初始化用户$U$后，生成一个固定位数的随机数，称为盐值。

主机保存所有用户的身份、盐值和加盐的口令字典：$(ID_U, salt, OWF(salt, PW_U))$

协议流程：
- $U\rightarrow H:ID_U$
- $H\rightarrow U:$ "请输入口令"
- $U\rightarrow H:PW_U \prime$
- $H\rightarrow U:$ 验证$OWF(salt, PW_U\prime) == OWF(salt, PW_U)$

协议面临两类攻击：
- 数据库攻击：对于某一个用户，敌手可以用他的$salt$进行爆破，但若想批量获得所有用户的原口令，需要不断地使用不同的$salt$，代价巨大。
- 链路攻击：敌手直接监听获取用户原始口令，一劳永逸。

## 使用加盐单向函数 —— 加强版
主机$H$初始化用户$U$后，生成一个固定位数的随机数，称为盐值。

主机保存所有用户的身份、盐值和加盐的口令字典：$(ID_U, salt, OWF(salt, PW_U))$

协议流程：
- $U\rightarrow H:ID_U$
- $H\rightarrow U:$ "请输入口令"
- $U\rightarrow H:Token \leftarrow OWF(salt, PW_U \prime)$
- $H\rightarrow U:$ 验证$Token == OWF(salt, PW_U)$

协议面临两类攻击：
- 数据库攻击：同上。
- 链路攻击：敌手直接监听获取用户的$Token$，通过系统验证。


**通过加盐的方法，敌手爆破攻击难度陡然提升，加盐可以降低数据库口令数据泄漏的风险。但是上述协议均无法抵御链路监听攻击，主要原因在于：只要在链路上截获了一次$Token$或者口令，敌手就能一劳永逸地通过验证。**

**使每次验证产生的$Token$发生变化，可以抵御链路监听攻击。**

## 使用哈希链

在线的窃听之所以能成功，是因为用户只使用一个固定的$Token$进行认证，一旦泄漏，敌手可以一劳永逸地登录系统。如果用户每次登录使用的$Token$都不一致，则在一定程度上可以规避窃听的攻击。

一次性口令：用户每次登录都使用不同的“口令”，即使本次的口令被窃听获取，下次不再使用，也不会影响下次的认证。

**初始化：**
- 主机$H$给用户$U$口令$PW_U$，并保存该次用户初始记录$(ID_U, n, Hash^n(PW_U))$
- *ID_U*是用户*U*的身份信息
- $n$是一个较大的整数，例如$n=1000$
- $Hash$为哈希函数，$Hash^n$代表使用哈希函数运行$n$次
- 用户只需记住口令$PW_U$，用户每次登录时主机都会更新用户的口令信息。

**协议第一次运行：**
- 用户$U$给主机$H$发送$Hash^{n-1}(PW_U)$，并发送给主机
- 主机收到$Hash^{n-1}(PW_U)$后，计算$Hash(Hash^{n-1}(PW_U))$，和存储的$Hash^{n}(PW_U)$比对
- 最后，更新口令记录为$(ID_U, n-1, Hash^{n-1}(PW_U))$

**协议第$c$次运行：**
- $U\rightarrow H:ID_U$
- $H\rightarrow U: c$ "请输入口令"
- $U\rightarrow H:Token \leftarrow Hash^{c - 1}(PW_U)$
- $H$计算$Hash(Hash^{c - 1}(PW_U))$是否等于$Hash^{c}(PW_U)$，并更新信息为$(ID_U, c-1, Hash^{c-1}(PW_U))$

计算过程中，计数器的值是动态变化的，每登录一次就减一，当计数器变成0时，用户与主机重新初始化。
