---
title: Go数据结构与算法(10)-实现hashmap
date: 2022-07-14 21:11:25
tags: algorithm
---

面试官经常问到的一个问题，hashmap是如何实现的？今天我们就来说下原理

## 什么是hashmap?
>map就是用于存储键值对（<key,value>）的集合类，也可以说是一组键值对的映射。而hashmap就是将key进行hash运算，得到目标元素在哈希表中的位置，然后再进行少量比较即可得到元素，这使得 HashMap 的查找效率极高