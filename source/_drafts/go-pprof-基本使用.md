---
title: go pprof 基本使用
date: 2021-11-23 17:20:28
index_img:
banner_img:
categories:
tags:
---

## 说明

大家好 我是李二狗

go自带的pprof工具包可以说是问题分析的利器了 

那么什么是pprof呢简单来讲pprof就是一个分析profile文件的工具，

那么什么是profile呢？

我们可以理解为一种标准格式的样本数据的集合，
不同的指标有不同的样本集合，我们在日常开发中主要集中在cup的指标和内存的指标上，
所以我们使用pprof分析go时的也就主要是cup profile 和 heap profile，
本文就是主要演示这两种数据的pprof分析使用

> 注： cpu指标的采集要手动指定采集的时间间隔

## 调试pprof的方式

1. web页面
    - 调用图
    - 火焰图
    - top列表

2. 命令行
    - 调用图
    - top列表
  
## 收集profile文件的方法

1. net/http/pprof：低侵入
2. runtime/pprof：部分侵入
3. test bench

## 生产环境使用

## 总结