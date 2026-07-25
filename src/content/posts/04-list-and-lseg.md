---
title: '从 list 到 lseg：为链表（Linked List）循环写出不变量（Invariant）'
published: 2026-07-24
draft: false
description: '归纳谓词描述完整链表，链表段描述遍历中的前缀，并由拼接引理维持循环不变量。'
tags: ['形式化验证', '分离逻辑', '链表', '循环不变量']
category: '形式化验证'
lang: 'zh'
series: 'separation-logic-shapes'
seriesOrder: 1
---

链表遍历看似只是不断移动游标，但在分离逻辑中，每次移动都必须回答两个问题：已经走过的节点由什么断言描述，尚未访问的节点又归谁所有？完整链表谓词 $\mathit{list}$ 与链表段谓词 $\mathit{lseg}$ 正是为这两个部分服务的。

## 完整链表（Complete List）：$\mathit{list}$

完整链表可以用归纳谓词表示：

$$
\begin{aligned}
\mathit{list}(\mathit{nil}) &:= \mathit{emp} \\
\mathit{list}(x) &:= \exists v,n.\ x \mapsto (v,n) \ast \mathit{list}(n)
\quad (x \ne \mathit{nil})
\end{aligned}
$$

其中，$\mathit{emp}$ 表示空堆；$x \mapsto (v,n)$ 表示节点 $x$ 的值为 $v$、后继指针为 $n$。分离合取 $\ast$ 保证头节点与尾部占用的存储互不重叠，因此也排除了这一定义下的环。

## 展开与折叠：在抽象与节点之间切换

展开（unfold）会把归纳谓词拆开一层，暴露头节点；折叠（fold）则执行反方向的推理，把节点和尾部重新封装为 $\mathit{list}$。

<example>

以链表 $a \to b \to \mathit{nil}$ 为例。对 $\mathit{list}(a)$ 展开一层，可以得到：

$$
\begin{aligned}
\mathit{list}(a)
&\Longrightarrow
a \mapsto (v_a,b) \ast \mathit{list}(b)
\end{aligned}
$$

展开后，抽象谓词中出现了具体断言 $a \mapsto (v_a,b)$，此时才有依据读取或修改 $a$ 的字段。如果还需要访问节点 $b$，可以继续展开：

$$
\begin{aligned}
\mathit{list}(b)
&\Longrightarrow
b \mapsto (v_b,\mathit{nil}) \ast \mathit{list}(\mathit{nil}) \\
&=
b \mapsto (v_b,\mathit{nil}) \ast \mathit{emp}
\end{aligned}
$$

修改完成后，可以沿相反方向折叠。例如，

$$
a \mapsto (v'_a,b) \ast \mathit{list}(b)
\Longrightarrow
\mathit{list}(a)
$$

表示修改后的节点 $a$ 与尾部 $\mathit{list}(b)$ 仍能组成一条完整链表。两种操作的关系可以概括为：

$$
\mathit{list}(a)
\;\underset{\mathrm{fold}}{\overset{\mathrm{unfold}}{\rightleftarrows}}\;
a \mapsto (v_a,b) \ast \mathit{list}(b)
$$

</example>

展开让证明获得访问节点所需的具体堆断言；折叠则证明访问或修改后的节点仍然满足抽象结构。

## 从完整链表到链表段（List Segment）：$\mathit{lseg}$

遍历开始后，只使用 $\mathit{list}(\mathit{head})$ 已经不够：游标走过的前缀与尚未访问的后缀必须同时表示。链表段 $\mathit{lseg}(x,y)$ 描述从 $x$ 到 $y$、但不包含 $y$ 的一段链：

$$
\begin{aligned}
\mathit{lseg}(x,y)
&:= (x=y \land \mathit{emp}) \\
&\quad \lor\ \exists v,n.\ x \mapsto (v,n) \ast \mathit{lseg}(n,y)
\end{aligned}
$$

在空段分支中，$x=y$ 且不占用任何堆资源；在递归分支中，可以从头部剥离一个节点，剩余部分仍是一条终点为 $y$ 的链表段。

## 用链表段表达循环不变量（Loop Invariant）

在循环入口处，可以使用下面的不变量，把堆分为“已遍历前缀”和“未遍历后缀”：

$$
\mathit{lseg}(\mathit{head},\mathit{cur})
\ast
\mathit{list}(\mathit{cur})
$$

循环刚开始时，$\mathit{cur}=\mathit{head}$，前缀退化为 $\mathit{emp}$，所以该不变量与初始的 $\mathit{list}(\mathit{head})$ 一致。

当 $\mathit{cur}\ne\mathit{nil}$ 时，展开后缀的头节点，得到循环体内可直接使用的形式：

$$
\mathit{lseg}(\mathit{head},\mathit{cur})
\ast
\mathit{cur} \mapsto (v,\mathit{next})
\ast
\mathit{list}(\mathit{next})
$$

游标前进到 $\mathit{next}$ 后，原来的当前节点应并入已遍历前缀。要重新建立循环不变量，需要使用链表段的拼接引理。

## 拼接引理（Concatenation Lemma）

两段首尾相接的链表段可以合并：

$$
\mathit{lseg}(x,y)
\ast
\mathit{lseg}(y,z)
\vdash
\mathit{lseg}(x,z)
$$

把当前节点视为从 $\mathit{cur}$ 到 $\mathit{next}$、长度为一的链表段，再与原有前缀拼接，就可以得到新的 $\mathit{lseg}(\mathit{head},\mathit{next})$。结合未访问的 $\mathit{list}(\mathit{next})$，循环不变量便重新成立。

这个引理通常需要归纳证明。在自动验证器中，它常以幽灵函数或辅助引理的形式出现。验证器不能自动发现拼接，并不意味着该蕴含不成立；这只是自动化能力与归纳推理之间的边界。

## 定义的边界

不同文献对 $\mathit{lseg}$ 的定义并不完全相同，尤其会影响 $\mathit{lseg}(x,x)$ 是否只能表示空段。使用具体工具前，应确认它对端点、环以及非接触条件的精确定义，不能把某一种记法直接当作通用标准。
