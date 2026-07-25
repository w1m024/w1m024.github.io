---
title: '从结构体到数组：分离逻辑如何建模 C 风格内存'
published: 2026-07-24
draft: false
description: '理解单元化堆、字段布局和地址运算为何需要额外的边界与对齐约束。'
tags: ['形式化验证', '分离逻辑', 'C语言']
category: '形式化验证'
lang: 'zh'
series: 'separation-logic-foundations'
seriesOrder: 3
---

## 单元化堆（Cell-Based Heap）与字段展开（Field Expansion）

为了描述 C 的数组与结构体，许多教学模型把堆写成有限映射：地址映射到值。多字段节点可以展开成若干相邻单元：

$$
\begin{aligned}
x &\mapsto (\mathit{value}, \mathit{next}) \\
&\approx x \mapsto \mathit{value} \ast (x + 1) \mapsto \mathit{next}
\end{aligned}
$$

这只是说明性的派生写法；真实 C 的对象布局、字段偏移、字节大小、对齐、指针有效性和整数转换都比它严格得多。验证时应让使用的工具或内存模型决定 $x + 1$ 的含义，不能把教学模型直接当作 C 标准语义。

## 灵活性与边界约束（Boundary Constraints）

单元化模型的好处是可描述数组切片、字段访问与低层数据结构；代价是对象边界不再自动存在。若允许任意整数参与地址运算，两个“结构体”可能错位重叠。实际规约通常需要额外维护对象有效性、长度、对齐或分配块边界。

<example>
**例：跨对象边界的字段访问。**

设两个相邻结构体分别从 $p$ 和 $p + 4$ 开始。若某段代码错误地把 $p + 3$ 当作一个占两个单元的字段起点，它就会同时覆盖第一个结构体的最后一个单元和第二个结构体的第一个单元。

要排除这种访问，规约必须明确记录对象大小或有效地址范围。
</example>

## 封装布局细节（Encapsulating Layout Details）

本文系列后续优先使用抽象的节点谓词，例如 $\mathit{node}(x, v, \mathit{next})$，而把布局细节封装在其定义中。这样既保留局部推理，也不会在每一篇链表文章里重复讨论地址算术。
