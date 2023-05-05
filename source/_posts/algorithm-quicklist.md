---
title: 数据结构与算法(17)- QuickList
date: 2023-05-05 09:35:12
tags: algorithm
---

## 什么是QuickList
>Quicklist 是 Redis 列表（List）数据结构的一种内部优化方式。其采用链址法（Chaining）实现，可以将多个列表分成多个节点依次存储，从而减小单个列表占用空间的大小。

Quicklist 内部由以下两部分组成：

- ziplist 节点：使用 ziplist 来压缩小于等于 ziplist-max-entries 个元素的列表。
- linkedlist 节点：使用 linkedlist 特殊数据结构来存储大于 ziplist-max-entries 个元素的列表。

```
ps: ziplist 是 Redis 的另一种数据结构，类似于 listpack，但它是被设计用来存储非常短的列表。相对于 ListPack，它采取更加紧凑、轻量化的存储方式。
```

Quicklist 的优势在于当列表大小较小时，使用 ziplist 可以有效地减小单个列表占用的空间。而在列表大小较大时，使用 linkedlist 可以避免频繁进行扩容操作，提高整体性能。需要注意的是，在写入和删除操作频繁的情况下，可能会导致整个 Quicklist 结构出现内存碎片，导致性能下降。