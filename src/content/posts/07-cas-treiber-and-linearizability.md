---
title: 'CAS（Compare-and-Swap）、Treiber 栈与逻辑原子性（Logical Atomicity）'
published: 2026-07-24
draft: false
description: '从比较交换的重试循环出发，理解线性化点与逻辑原子三元组的作用。'
tags: ['形式化验证', '无锁编程', 'CAS', '线性化']
category: '形式化验证'
lang: 'zh'
series: 'concurrent-separation-logic'
seriesOrder: 2
---

CAS（compare-and-swap）在一次原子操作中比较某地址的当前值与期望值；相等则写入新值并成功，否则不写入并失败。无锁算法常把它放进重试循环：读取当前状态、准备候选更新、CAS 提交；失败后重新读取。

## 先理解“逻辑原子性”（Logical Atomicity）

CAS 的原子性只描述一次硬件或机器指令：比较和写入不会被其他线程插入。**逻辑原子性**描述的则是一个更大的程序操作，例如整个 `push(v)` 函数：即使它包含读取、分配节点、反复重试等多个步骤，对外仍然可以表现得像在某一个瞬间完成了。

这个“某一个瞬间”就是线性化点。逻辑原子性并不要求函数从开始执行到结束都独占栈，也不要求调用者在开始时知道共享栈的完整状态；它只要求存在一个位于调用与返回之间的瞬间，使操作的抽象效果能够一次性发生。对 `push(v)` 而言，抽象规范可以示意为：

$$
\langle\!\langle \mathit{Stack}(vs) \rangle\!\rangle
\;\mathit{push}(v)\;
\langle\!\langle \mathit{Stack}(v::vs) \rangle\!\rangle_{\mathrm{lin}}
$$

这里的 $vs$ 表示线性化点观察到的抽象栈，而不是函数调用开始时被冻结的一份栈快照。后文会看到，Treiber 栈中成功的 CAS 正好提供了这个瞬间：失败的 CAS 只让当前调用重试，成功的 CAS 才把物理头指针和抽象栈同时推进一步。

Treiber 栈是典型例子。`push` 读出栈顶，让新节点指向它，再 CAS 更新栈顶。成功的 CAS 通常是 `push` 的线性化点：虽然整个函数执行跨越一段时间，但抽象栈在该瞬间从 $vs$ 变为 $v :: vs$。

线性化（linearizability）要求每个调用都能安排在调用与返回之间的某一瞬间，并与实时先后关系和顺序规约一致。逻辑原子三元组（logically atomic triple）正是把上面的逻辑原子性承诺写成证明接口：调用时不必固定共享状态，只需在线性化瞬间完成规定的抽象状态变换。

失败的 CAS 不改变抽象栈；成功的 CAS 才在原子步骤中打开相关不变量、更新物理头指针与幽灵历史，再关闭不变量。这是“重试循环仍可给调用者一个原子接口”的证明骨架。

## 一个最小的形式化证明

下面只证明 `push(v)`；记号是逻辑原子三元组的示意写法，不绑定某个具体证明助手。令

$$
I(h,vs) := \mathit{Stack}(h,vs) \ast \mathit{Hist}(vs)
$$

表示物理头指针为 $h$ 的 Treiber 栈对应抽象序列 $vs$，并且幽灵历史已经记录了该序列。我们希望证明：

$$
\langle\!\langle I(h,vs)\rangle\!\rangle\
\;\mathit{push}(v)\
\langle\!\langle I(h',v::vs)\rangle\!\rangle_{\mathrm{lin}}
$$

下标 $\mathrm{lin}$ 表示：前置状态中的 $vs$ 不必在调用开始时确定；只要在调用与返回之间找到一个瞬间，使得状态从 $vs$ 变为 $v::vs$，这个调用就满足逻辑原子性。

证明循环的关键步骤如下：

1. 读取当前头指针 $h_0$，分配新节点 $n$，并写入 $n.next=h_0$。此时只准备了候选更新，抽象状态仍为 $vs$；证明状态可写成 $n\mapsto(v,h_0) \ast I(h_0,vs)$。
2. 若 `CAS(top, h_0, n)` 失败，说明其他线程已经改变了 `top`。这次尝试没有改变物理栈，也没有更新 $\mathit{Hist}$，因此抽象状态仍为某个当前序列；重新读取并重试即可。
3. 若 CAS 成功，在这个 CAS 的原子瞬间打开 $I(h_0,vs)$。CAS 已经把物理头指针改为 $n$，而 $n.next=h_0$，所以物理结构现在表示 $v::vs$。同时执行幽灵更新

   $$
   \mathit{Hist}(vs) \longrightarrow \mathit{Hist}(v::vs)
   $$

   并关闭不变式为 $I(n,v::vs)$。这个成功的 CAS 就是 `push(v)` 的线性化点，随后函数返回。

每次成功的 CAS 都只对应一个这样的抽象更新；失败的 CAS 没有抽象效果。由于原子 CAS 的成功顺序与这些抽象更新的顺序一致，并且线性化点位于对应调用的开始与返回之间，所有并发 `push` 调用都可以按成功 CAS 的顺序排列。因此，该实现满足上面的逻辑原子三元组，也就满足 `push` 的线性化规范。

本文刻意不证明内存回收。Treiber 栈若在 `pop` 后立即释放节点，可能遇到 ABA 或其他线程仍读取旧节点的问题；安全回收需要独立机制，如 hazard pointers、epoch-based reclamation 或垃圾回收。CAS 本身并不解决这一问题，也不保证单个线程不会饥饿。
