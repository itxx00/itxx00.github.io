---
layout: post
title: "系统中的随机数和熵值"
description: "简单总结一下系统中的随机数以及相关问题"
categories: [os]
tags: [centos7, random, entropy]
---

> 本文试着总结系统中的随机数前前后后以及管理中需要注意的问题 [先欠着]

* Kramdown table of contents
{:toc .toc}

## 1 什么是随机数？

随机数就是无法预测的数

## 2 随机数有什么用？

随机是为了安全

## 3 如何获得随机数？

/dev/random
/dev/urandom
/proc/sys/kernel/random/entropy_avail

## 4 会有哪些问题？

1 随机数数产生速度慢
2 影响上层应用

## haveged 和 rng-tools
