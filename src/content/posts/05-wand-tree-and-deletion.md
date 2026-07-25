---
title: '魔杖（Magic Wand）、树与安全释放：形状规约（Shape Specification）的一次完整推导'
published: 2026-07-24
draft: false
description: '用 tree 谓词证明递归释放，并解释魔杖为何适合表示待补全的数据结构。'
tags: ['形式化验证', '分离逻辑', '魔杖', '二叉树']
category: '形式化验证'
lang: 'zh'
series: 'separation-logic-shapes'
seriesOrder: 2
---

二叉树谓词（binary-tree predicate）的自然定义把左右子树用分离合取（separating conjunction）$\ast$ 连接：

$$
\begin{aligned}
\mathit{tree}(\mathit{nil}) &:= \mathit{emp} \\
\mathit{tree}(x) &:= \exists l,r.\ x \mapsto (l,r) \ast \mathit{tree}(l) \ast \mathit{tree}(r)
\end{aligned}
$$

因此左右子树的存储不重叠。这个谓词适合验证释放操作：

```c
void delete_tree(node *root) {
  if (root != NULL) {
    delete_tree(root->left);
    delete_tree(root->right);
    free(root);
  }
}
```

目标规约是 $\{\mathit{tree}(\mathit{root})\}\ \mathit{delete\_tree}(\mathit{root})\ \{\mathit{emp}\}$。在非空分支展开 $\mathit{tree}(\mathit{root})$ 后，左子树调用使用归纳假设；根节点与右子树作为框架资源（frame）保持不变。再对右子树重复同一步，最后释放根节点。$\mathit{emp}$ 是单位元，所以三次释放留下的空堆合并后仍为 $\mathit{emp}$。

这个证明的前提很强：输入必须是一棵由该谓词描述的、独占拥有（exclusive ownership）的树。它不适用于有向无环图（directed acyclic graph，DAG）或共享节点；共享结构需要引用计数（reference counting）、共享所有权（shared ownership）或其他协议，不能沿用这个 `free` 证明。

## 魔杖（Magic Wand）：把“还缺什么”写成资源接口（Resource Interface）

魔杖（magic wand）通常记作 $P \mathbin{-\!\ast} Q$，也称分离蕴含（separating implication）。它表达的不是普通逻辑中的“如果 $P$，那么 $Q$”，而是：如果再提供一块与当前资源不相交、并且满足 $P$ 的堆，那么当前资源就能扩展为满足 $Q$ 的堆。

可以把它理解为一个等待资源的接口（resource interface）：当前手里有一部分结构，但还缺少某个局部资源；一旦调用者补上这部分资源，接口就保证整体结构满足目标断言。

更形式化地说，若当前堆为 $h$，则

$$
h \models P \mathbin{-\!\ast} Q
$$

表示对任意与 $h$ 不相交的堆 $h'$，只要 $h' \models P$，就有

$$
h \mathbin{\uplus} h' \models Q
$$

其中 $\mathbin{\uplus}$ 是不相交堆并（disjoint heap union）。因此，魔杖本身描述的是“如何接收一块资源并完成更大的断言”，而不是一段已经完整展开的数据结构。

它最基本的使用规则是魔杖消去（wand elimination）：

$$
P \ast (P \mathbin{-\!\ast} Q) \vdash Q
$$

左边的 $P$ 是调用者实际提供的资源，右边的 $P \mathbin{-\!\ast} Q$ 是等待这份资源的接口。分离合取（separating conjunction）要求两者占用不相交的堆；满足这一条件后，接口就可以被兑现，整体资源得到 $Q$。

## 用魔杖表示待补全的链表

考虑一个链表段谓词（list-segment predicate） $\mathit{lseg}(x,y)$：它描述从指针 $x$ 开始、在指针 $y$ 处结束的一段链表。与直接描述完整链表相比，链表段允许我们把“尾部还没有交给当前证明”这件事保留下来。

例如，已经拥有从 $x$ 到 $t$ 的前缀时，可以把它看成一个等待尾部资源的接口：

$$
\mathit{list}(t) \mathbin{-\!\ast} \mathit{list}(x)
$$

它的含义是：只要之后补上一块从 $t$ 开始的尾部资源 $\mathit{list}(t)$，整体就能成为从 $x$ 出发的完整链表 $\mathit{list}(x)$。在常见的链表段定义下，可以通过一个拼接引理（concatenation lemma）说明：

$$
\mathit{lseg}(x,t) \vdash \mathit{list}(t) \mathbin{-\!\ast} \mathit{list}(x)
$$

这里的公式描述的是“前缀作为上下文，等待尾部”的关系；具体证明仍需要根据链表段的端点约定，确认指针连接和堆的分离条件。

更常见的模块化用法是把前缀和尾部拆开。假设某个模块持有一个待补全的上下文（context），其资源满足 $P \mathbin{-\!\ast} Q$；另一个模块稍后产生满足 $P$ 的尾部资源。两者合并后就可以应用：

$$
\mathit{list}(t) \ast (\mathit{list}(t) \mathbin{-\!\ast} \mathit{list}(x))
\vdash \mathit{list}(x)
$$

它正是一般消去规则在链表场景中的实例。

这正是魔杖适合描述链表遍历、递归构造和模块拼接的原因：证明可以先留下一个“还缺尾部”的接口，而不必在当前步骤就拥有整棵结构。

需要注意，魔杖不是普通链表段（ordinary list segment）的增强版，也不是一种新的内存布局。普通链表段描述“现在有哪些节点以及它们如何连接”；魔杖描述“补上什么资源后，当前资源可以满足什么目标”。前者是数据结构谓词，后者是高阶断言（higher-order assertion），二者承担的角色不同。
