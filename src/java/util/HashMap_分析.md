---
typora-root-url: ../../../note/doc
---

## 1、整体结构图

## 1、前言

Map 这样的 `Key Value` 在软件开发中是非常经典的结构，常用于在内存中存放数据。

本篇主要想讨论 ConcurrentHashMap 这样一个并发容器，在正式开始之前我觉得有必要谈谈 HashMap，没有它就不会有后面的 ConcurrentHashMap。



## HashMap

众所周知 HashMap 底层是基于 `数组 + 链表` 组成的，不过在 jdk1.7 和 1.8 中具体实现稍有不同。

### Base 1.7

1.7 中的数据结构图：

![java.util.HashMap_hashmap internal structure](https://github.com/muyutingfeng/jdk1.8-source-analysis/blob/master/note/doc/java.util.HashMap_hashmap%20internal%20structure.png?raw=true)





















## 2、核心方法

1. put(K key, V value)
2. putVal(int hash, K key, V value, boolean onlyIfAbsent,boolean evict)
3. resize
4. get
5. remove
6. replace



