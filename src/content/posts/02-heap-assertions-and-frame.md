---
title: '堆断言（Heap Assertions）、分离合取（Separating Conjunction）'
published: 2026-07-24
draft: false
description: '区分普通合取与分离合取，理解 points-to 断言的足迹，以及局部推理的适用边界。'
tags: ['形式化验证', '分离逻辑', '堆模型']
category: '形式化验证'
lang: 'zh'
series: 'separation-logic-foundations'
seriesOrder: 2
---

分离逻辑（Separation Logic）中，$\land$ 与 $\ast$ 的区别决定了断言（assertion）如何描述堆。

## 同一份堆与相互分离的堆

- $P \land Q$：同一份堆同时满足 $P$ 与 $Q$；二者可以谈论同一地址。
- $P \ast Q$：堆可分成不相交的两份，分别满足 $P$ 与 $Q$。

## 精确指向与模糊指向

堆断言可按对足迹（footprint）的约束强度分为精确（precise）与模糊（imprecise，或 non-precise）两类。

### 精确指向（Precise Points-to）

这里的“精确”约束的是**堆的边界**，而不只是值写得有多具体。$x \mapsto 3$ 断言当前堆恰好有一个单元：地址为 $x$，值为 $3$。$x \mapsto {-}$ 仍然是精确指向；它只是不关心该单元的值，足迹依然恰好是 $\{x\}$。

因此，若真实堆还包含地址 $z$，它本身不满足 $x \mapsto 3$，却可以满足：

$$
x \mapsto 3 \ast z \mapsto w
$$

### 模糊指向（Imprecise Points-to）

相对地，本文用 $x \hookrightarrow v$ 表示模糊指向：堆中至少有地址为 $x$、内容为 $v$ 的单元，但其余堆可以存在。它可理解为 $x \mapsto v \ast \mathit{True}$；因此它保留了关于 $x$ 的局部事实，却不再确定整个堆的足迹。

在本文采用的经典精确堆模型里，$x \mapsto v$ 所占用的堆只有地址 $x$，其内容为 $v$。因此：

$$
x \mapsto 3 \ast y \mapsto 3 \quad\Longrightarrow\quad x \ne y
$$

相反，$x \mapsto 3 \land y \mapsto 3$ 会强制 $x = y$。

## 空堆与断言的足迹

$\mathit{emp}$ 表示空堆，是 $\ast$ 的单位元：$P \ast \mathit{emp}$ 与 $P$ 等价。它看似简单，却让“空链表不额外占用内存”能被精确表达。

<example>
$$
\mathit{list}(\mathit{NULL}) \triangleq \mathit{emp}
$$

当头指针为 $\mathit{NULL}$ 时，空链表不要求、也不占用任何堆单元。
</example>

还可以按足迹的约束强度阅读断言：

1. 纯断言（pure assertion），例如 $n > 0$，不依赖堆；
2. 精确足迹断言，例如 $x \mapsto {-}$，唯一确定占用的地址集合；
3. 递归形状断言，例如 $\mathit{list}(x)$，在给定起点时描述一块可被识别的结构性足迹。

## 记法说明

这些术语和符号在 Iris、VeriFast、Viper 等工具中并不完全同义。本文将 $\hookrightarrow$ 约定为上述模糊指向；在其他逻辑中，它的记法及其是否等同于 $x \mapsto v \ast \mathit{True}$，都必须以该逻辑的定义为准。
