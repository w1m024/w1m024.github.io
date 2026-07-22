---
title: Java 多线程笔记
published: 2025-06-08T15:11:55+08:00
description: Java 并发容器、线程池、Future 与 ThreadLocal 的学习笔记。
tags: [Java, 并发]
category: Java
lang: zh_CN
draft: false
---

这篇笔记整理 Java 并发编程中最常用的几个入口，后续会随学习持续补充。

## 线程安全集合

java.util.concurrent 为 List、Map、Set、Queue 和 Deque 提供了相应的并发实现。选择时要先判断读写比例、容量边界和阻塞需求，而不是只看是否「线程安全」。

## 线程池与 Future

线程池负责复用工作线程，Future 用于获得异步任务的结果。CompletableFuture 进一步提供了组合、回调和异常处理能力，适合编排有依赖关系的异步任务。

## ThreadLocal

ThreadLocal 用来在线程处理链中保存上下文。在线程池里必须在 finally 中调用 remove()，否则线程复用可能导致上下文残留、数据串扰或内存泄漏。
