---
title: '为什么指针程序需要分离逻辑（Separation Logic）'
published: 2026-07-24
draft: false
description: '用一个别名反例说明普通霍尔逻辑的 Frame 问题，以及分离逻辑如何恢复局部推理。'
tags: ['形式化验证', '分离逻辑', '程序逻辑']
category: '形式化验证'
lang: 'zh'
series: 'separation-logic-foundations'
seriesOrder: 1
---

指针程序（pointer program）难验证，不是因为赋值本身复杂，而是因为我们通常不知道两个名字会不会指向同一块内存。

<details>

<summary>【例】指针别名会破坏“未改动”断言</summary>

```c
void set_four(int *p) { *p = 4; }
```

若只知道 `*p == 1`，执行后可得到 `*p == 4`。但若还知道 `*q == 3`，不能仅因代码未出现 `q` 就断言它仍为 3：`p` 与 `q` 可能发生指针别名（pointer aliasing），指向同一地址。

</details>

## 局部操作为何难以证明

这就是框架问题（Frame Problem）：每次证明一个局部操作，都要说明所有“没有被改动”的内存为什么真的没有被改动。对链表、树等结构逐一列出不相等关系，证明会迅速失去可维护性。

<details>

<summary>【例】链表首结点更新的别名负担</summary>

例如，考虑一个只给链表首结点打标记的函数：

```c
struct Node {
  int value;
  int marked;
  struct Node *next;
};

void mark_head(struct Node *head) { head->marked = 1; }
```

若希望在调用后仍断言 `head->next->marked == 0`，普通断言必须额外说明 `head != head->next`；若还要保留第二个、第三个结点的性质，又要分别说明 `head != head->next->next`、`head != head->next->next->next`，依此类推。函数实际只写了一个字段，但为了证明整条链其余部分未变，前置条件会随着链表长度不断堆积别名排除条件。

</details>

分离逻辑（Separation Logic）把程序状态写成栈（stack）与堆（heap），并引入分离合取（separating conjunction）$P \ast Q$。它不仅表示 $P$ 和 $Q$ 同时成立，还要求它们使用的堆资源可以拆成两块不相交的部分。因此：

$$
x \mapsto 1 \ast y \mapsto 2
$$

同时表达了两个单元的内容和 $x \ne y$。把操作需要的最小资源写在规约（specification）中，其他不相交资源就可以作为框架资源（frame）保留：

$$
\frac{\{P\}\ c\ \{Q\}}
     {\{P \ast F\}\ c\ \{Q \ast F\}}
\quad \text{Frame}
$$

这里的关键不是“程序没有提到 $F$”，而是前置条件（precondition）所赋予的资源权限，不足以支持程序安全地访问或修改 $F$。

故障避免语义（fault-avoiding semantics）假设程序只能访问自己拥有权限的内存。若程序读写一个未拥有的地址，执行就会发生 `fault`，即非法内存访问（invalid memory access），程序不能算作正常、安全地执行。

所以，若我们已经证明：

$$
\{P\}\ c\ \{Q\}
$$

意味着：只给程序 \(P\) 所描述的堆资源时，程序 \(c\) 仍能安全运行，并得到 \(Q\)。现在再额外放入一块与 \(P\) 不相交的资源 \(F\)，程序没有获得操作 \(F\) 的权限；一旦碰它就会 fault。因此在正常执行中，\(F\) 必须保持不变，便得到：

$$
\{P \ast F\}\ c\ \{Q \ast F\}
$$

这里“紧凑规约”指只写出程序实际需要的最小资源的规约。它不仅描述程序会如何改变 \(P\)，也隐含保证：程序不会越权破坏任何额外且不相交的背景资源 \(F\)。
