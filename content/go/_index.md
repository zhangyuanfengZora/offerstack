---
title: Go 语言
weight: 1
next: /go/0-basics/goroutine-vs-thread
cascade:
  type: docs
---

## 📌 章节概览

Go 语言是当前后端面试的核心考点，本章节按照学习层次递进，覆盖从基础语法到底层原理的完整知识体系。

| 模块 | 核心考点 |
|------|---------|
| 零、基础语法 | 协程vs线程、make/new区别、函数传参机制 |
| 一、基本数据结构 | Slice、Map、Interface、Channel 底层实现 |
| 二、GMP调度模型 | P/M/G架构、工作窃取、抢占式调度 |
| 三、Sync包 | sync.Map、Mutex、WaitGroup、Pool |
| 四、内存管理 | mspan/mcache/mcentral、内存逃逸分析 |
| 五、垃圾回收 | 三色标记法、写屏障机制、GC调优策略 |
