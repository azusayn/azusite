---
title: 无锁线程池设计
date: 2026/09/12 01:01:00
categories:
    - Design
tags:
    - concurrency
---

## 需求

在最基本的并发模型中，每一个系统线程会绑定在一个任务上。当进行 I/O 密集型任务时（比如等数据从 page cache 复制到内存中），系统线程会进入休眠态（TASK_INTERUPTIBLE/TASK_UNINTERUPTIBLE），等待数据复制的完成，导致并发性下降。

为了解决这一点，我们需要将线程解放出来去执行别的任务。这就需要解决两个问题：

1. 如何将线程解放出来
2. 如何给线程
