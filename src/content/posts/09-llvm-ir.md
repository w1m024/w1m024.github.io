---
title: 'LLVM IR：从 C 代码到控制流'
published: 2026-08-21
description: '从 LLVM 架构、数据表示和类型系统出发，理解 LLVM IR 中的控制流与函数。'
image: ''
tags: ['LLVM', '编译原理', '编译器']
category: '编译原理'
draft: false
lang: 'zh'
---

# LLVM IR

这篇笔记整理自 个人学习 [llvm-ir-tutorial](https://github.com/Evian-Zhang/llvm-ir-tutorial) 过程中的学习笔记。保留部分原文总结性表述，删减部分示例并加入少量个人理解。仅为记录本人学习过程，内容比较简单，无法代替原教程，不建议作为初学资料。

**AI 辅助说明：本文过程中使用 AI 辅助进行内容归纳、结构调整、术语润色和格式整理。**

## 目录

1. [LLVM 架构简介](#1-llvm-架构简介)
2. [Hello world：读取第一份 IR](#2-hello-world读取第一份-ir)
3. [数据表示](#3-数据表示)
   - [3.1 数据区与符号表](#31-数据区与符号表)
   - [3.2 寄存器和栈](#32-寄存器和栈)
   - [3.3 数据的使用](#33-数据的使用)
4. [类型系统](#4-类型系统)
5. [控制流](#5-控制流)
   - [5.1 控制语句](#51-控制语句)
   - [5.2 函数](#52-函数)

---

## 1. LLVM 架构简介

LLVM 可以看成一套编译器基础设施。语言前端把源代码转换成 LLVM IR，LLVM 的优化器处理 IR，目标后端再把 IR 转换成汇编或目标文件。不同语言可以共用同一套中间表示和后端。

以 `clang test.c -o test` 为例，可以把过程简化为：

```text
C 源代码
  ↓ 预处理、语法分析、语义分析
抽象语法树（AST）
  ↓ 前端代码生成
LLVM IR
  ↓ opt 优化
优化后的 LLVM IR
  ↓ llc
汇编代码
  ↓ 汇编器和链接器
可执行文件
```

可以用下面的命令观察各个阶段：

```bash
# 查看 AST
clang -Xclang -ast-dump -fsyntax-only test.c

# 生成可读的 LLVM IR（.ll）
clang -S -emit-llvm test.c

# 用 opt 优化 IR
opt test.ll -S --O3 -o test.opt.ll

# 生成汇编代码
llc test.ll -o test.s
```

这里的 `opt` 是 LLVM 优化工具，`llc` 是把 LLVM IR 转成汇编的后端工具。`llvm-as` 和 `llvm-dis` 分别负责在可读 IR 与比特码之间进行汇编和反汇编。

### LLVM IR 的三种形态

“LLVM IR”在不同语境下可能指三种形态：

- **内存中的 IR**：编译器通过 LLVM API（编程接口）创建和操作的对象，是编译器实现最直接接触的形态。
- **可读形式的 IR**：通常保存为 `.ll` 文件，适合阅读、调试和手工实验。
- **比特码形式的 IR**：通常保存为 `.bc` 文件，是适合工具链处理的二进制形式。

可读 IR 与比特码可以互相转换：

```bash
# 生成比特码
clang -c -emit-llvm test.c -o test.bc

# .ll → .bc
llvm-as test.ll -o test.bc

# .bc → .ll
llvm-dis test.bc -o test.ll
```

LLVM IR 的完整语法和语义以 [LLVM Language Reference](https://llvm.org/docs/LangRef.html) 为准。

## 2. Hello world：读取第一份 IR

这里的“Hello world”不是打印字符串，而是用一个只包含局部整数的 C 程序作为最小示例，观察变量如何被表示：

```c
int main() {
    int a = 10;
    int b = a;
    b = 20;
    return b;
}
```

这段示例使用现代 LLVM IR 的写法：`i32` 表示 32 位整数，`ptr` 表示指针，`dso_local` 表示符号属于当前动态共享对象（DSO），通常不需要按可抢占的外部符号处理。

用 `clang -S -emit-llvm scalar.c -o scalar.ll` 生成 IR 后，函数主体可以简化为：

```llvm
define dso_local i32 @main() {
entry:
  %retval = alloca i32, align 4
  %a = alloca i32, align 4
  %b = alloca i32, align 4
  store i32 0, ptr %retval, align 4
  store i32 10, ptr %a, align 4
  %0 = load i32, ptr %a, align 4
  store i32 %0, ptr %b, align 4
  store i32 20, ptr %b, align 4
  %1 = load i32, ptr %b, align 4
  ret i32 %1
}
```

读这段 IR 时，可以先抓住四个关键词：

- `define dso_local i32 @main()`：定义一个返回 `i32` 的函数。函数名和全局符号以 `@` 开头。
- `entry:`：一个基本块的标签。
- `alloca`：在当前函数的栈帧中分配内存。`%a` 不是整数 `a` 的值，而是存放 `a` 的那块内存的地址。
- `load` / `store`：分别从地址读取值、向地址写入值。

例如，`%0 = load i32, ptr %a` 表示从 `%a` 指向的内存中读取一个 `i32`，并把读取结果命名为 `%0`；`store i32 10, ptr %a` 则把整数 `10` 写入这块内存。

因此，C 语言中的 `return b` 在这份 IR 中需要先读取 `b` 的值，再返回读取结果：

```llvm
%1 = load i32, ptr %b, align 4
ret i32 %1
```

不能直接写成 `ret i32 %b`，因为 `%b` 保存的是地址，不是 `b` 的整数值。

这份示例还展示了 SSA（Static Single Assignment，静态单赋值）值：`%0`、`%1` 等名字代表指令产生的值。一个 SSA 名称只能被定义一次；如果需要表示“同一个变量后来有了新值”，就要使用新的 SSA 名称，或在控制流汇合处使用后文介绍的 `phi` 指令表达这种变化。

## 3. 数据表示

### 3.1 数据区与符号表

从汇编和进程内存的角度，可以先使用一个简化模型：

- **寄存器**保存当前计算中的值；
- **栈**保存函数调用期间的局部数据和临时地址；
- **堆**保存动态分配的数据；
- **数据区**保存全局数据以及程序需要的静态内容。

堆上的对象通常需要通过某个指针引用；全局数据则在程序装载时进入进程的数据区。LLVM IR 中，全局变量和全局常量的名字都以 `@` 开头：

```llvm
@global_variable = global i32 0
@global_constant = constant i32 0
```

`global` 表示可写的全局对象，`constant` 表示只读的全局对象。

#### 符号、链接与可见性

编译器通常按编译单元生成目标文件。一个文件可能引用了另一个文件中定义的函数或变量；链接器会收集这些符号，并把引用与定义匹配起来。动态链接时，动态链接器还可能参与符号解析。

LLVM IR 用链接类型和可见性控制符号如何参与这个过程。常见的链接类型包括：

- 默认的外部链接：符号可以在链接时被其他编译单元看到；
- `private`：名字不进入符号表，适合只在当前模块内部使用的对象；
- `internal`：作为当前模块的局部符号存在，不参与跨模块的符号解析，语义上接近 C 的文件内 `static`。

可见性常见取值为 `default`、`hidden` 和 `protected`，用于进一步控制动态链接时的可见范围和符号是否可以被替换。`dso_local` 表示符号属于当前动态共享对象（DSO）。对这类符号，生成的代码通常不需要通过过程链接表（PLT）进行可抢占式解析。

下面的 C 代码覆盖了几种常见的符号：

```c
int a;
extern int b;
static int c;
void d(void);
void e(void) {}
static void f(void) {}
```

其含义可以概括为：

- `a`：当前文件定义、其他文件可以使用的全局变量；
- `b`：当前文件引用、其他文件负责定义的全局变量；
- `c`：当前文件定义、其他文件不可直接使用的变量；
- `d`：当前文件引用、其他文件负责定义的函数；
- `e`：当前文件定义、其他文件可以使用的函数；
- `f`：当前文件定义、其他文件不可直接使用的函数。

Clang 生成的 IR 通常会用 `internal` 表示 C 的 `static`，用 `external` 链接属性表示外部符号；如果函数只有声明、定义在其他模块中，则用 `declare` 形式写出函数签名。可被其他模块使用的符号还可能带有 `dso_local` 等信息。

### 3.2 寄存器和栈

寄存器访问通常比内存访问更高效，但寄存器数量有限。LLVM IR 因此使用虚拟寄存器来表达计算结果，虚拟寄存器名字以 `%` 开头：

```llvm
%local_value = add i32 1, 2
```

虚拟寄存器最终会由后端分配到目标架构的物理寄存器，或在寄存器不足时溢出到栈上。这里要区分 LLVM IR 的虚拟寄存器和具体 ABI（Application Binary Interface，应用程序二进制接口）的寄存器保存约定：

- **caller-saved register（调用者保存寄存器）**：函数调用后不保证保留。调用者如果还需要其中的值，就必须在调用前保存。
- **callee-saved register（被调用者保存寄存器）**：函数返回时需要恢复到调用前的值。以 System V AMD64 ABI 为例，`rbp`、`rbx`、`r12`–`r15` 属于这类寄存器。

这些约定由 ABI 决定，不是 LLVM IR 类型系统的一部分。它们的作用是让调用者和被调用者能够协作保存跨函数调用仍然有效的值。

当需要一个地址，或需要表达可变的内存对象时，就要使用栈。栈帧可以理解为一次函数调用在栈上占用的那部分空间。LLVM IR 用 `alloca` 在当前函数的栈帧中分配空间：

```llvm
%local_variable = alloca i32
```

### 3.3 数据的使用

#### 全局变量和栈变量都是地址

全局变量和栈变量虽然存储位置不同，但在 LLVM IR 中都可以先看作指向内存区域的指针：

```llvm
@global_variable = global i32 0
%local_variable = alloca i32
```

上面两个名字都不是 `i32` 值本身，所以不能直接把 `@global_variable` 作为 `add` 的操作数。需要用 `load` 读取，用 `store` 写入：

```llvm
%value = load i32, ptr @global_variable
store i32 42, ptr %local_variable
```

#### SSA

LLVM IR 严格遵守 SSA（Static Single Assignment，静态单赋值）形式：每个 SSA 名称只能定义一次。下面的写法不合法：

```llvm
%1 = add i32 1, 2
%1 = add i32 3, 4
```

如果一个值在不同控制流路径上有不同来源，需要在汇合处使用 `phi`；如果值本身存放在内存中，则可以通过 `load` 和 `store` 表达多次更新。

## 4. 类型系统

汇编通常被视为弱类型形式，而 LLVM IR 是强类型的。每条指令都会明确写出操作数和结果的类型。

### 基本类型

常见的基本类型包括：

- `void`：表示没有返回值，常用作函数返回类型；
- `iN`：宽度为 `N` 的整数，例如 `i1`、`i8`、`i32`、`i64`；
- `float`、`double` 等浮点类型。

`i1` 只有 `true` 和 `false` 两个值：

```llvm
%flag = alloca i1
store i1 true, ptr %flag
```

LLVM IR 的整数类型本身不携带“有符号”或“无符号”属性。符号语义由使用它的指令决定。例如：

```llvm
%unsigned_result = udiv i8 -6, 2
%signed_result = sdiv i8 -6, 2
```

整数通常按补码表示；`udiv` 和 `sdiv` 分别按无符号和有符号规则解释操作数。

### 类型转换

整数转换常见有三种：

- `trunc`：截去高位，把较宽整数转换为较窄整数；
- `zext`：在高位补零，进行零扩展；
- `sext`：复制符号位，进行符号扩展。

```llvm
%short_value = trunc i32 257 to i8
%zero_extended = zext i8 -1 to i32
%sign_extended = sext i8 -1 to i32
```

浮点数与整数之间可以使用 `fptoui`、`fptosi`、`uitofp` 和 `sitofp`。转换到更窄类型时需要注意取值范围；某些超出范围的浮点到整数转换可能产生未定义行为，也就是 LLVM 不再保证程序结果或后续行为。

### 指针类型

现代 LLVM IR 使用不透明指针类型 `ptr`。它只表示一个地址，不在指针类型本身中记录所指向的元素类型；元素类型会在 `load` 或 `getelementptr` 等具体指令的上下文中给出。

不同 LLVM 版本生成的文本 IR 可能还会出现 `i32*` 这样的带元素类型指针写法。本篇统一使用较新的 `ptr` 语法；两者在这里表达的核心关系都是“地址”和“地址中的值”需要区分。

需要把地址当作整数处理时，可以用 `ptrtoint`；需要把整数解释为地址时，可以用 `inttoptr`。这类转换应该谨慎使用，因为它们与目标平台的地址表示有关。

### 聚合类型

数组和结构体是常见的聚合类型：

```llvm
%MyStruct = type {
    i32,
    i8
}

@global_array = global [4 x i32] [i32 0, i32 1, i32 2, i32 3]
@global_string = global [12 x i8] c"Hello world\00"
```

无论是数组还是结构体，只要它作为全局变量或栈变量存在，名字通常代表一块内存的地址；如果它被 `load` 到虚拟寄存器中，则寄存器里保存的是聚合值本身。两种形态的访问指令不同。

#### `getelementptr`：从指针计算元素地址

对于指针形式的数组或结构体，可以使用 `getelementptr` 计算字段地址，再用 `load` 读取：

```llvm
%MyStruct = type { i32, i32 }

define void @foo(ptr %my_structs_ptr) {
    %my_y_ptr = getelementptr %MyStruct, ptr %my_structs_ptr, i64 2, i32 1
    %my_y_val = load i32, ptr %my_y_ptr
    ret void
}
```

这里的索引依次表示：第 3 个 `MyStruct`（索引 `2`），以及该结构体的第 2 个字段（索引 `1`）。如果指针直接指向一个结构体而不是结构体数组，通常要先使用一个 `i64 0` 的索引，再选择字段：

```llvm
%my_y_ptr = getelementptr %MyStruct, ptr %my_structs_ptr, i64 0, i32 1
```

多级聚合可以继续追加索引，例如 `my_structs[2].y[3]`：

```llvm
%MyStruct = type { i32, [5 x i32] }
%my_structs = alloca [4 x %MyStruct]
%element_ptr = getelementptr [4 x %MyStruct], ptr %my_structs, i64 0, i64 2, i32 1, i64 3
```

这里的第一个 `i64 0` 先从“指向整个数组的指针”进入数组，第二个 `i64 2` 再选择第 3 个结构体，后面的索引依次选择 `y` 字段和 `y[3]`。数组作为指针基址时，这个首个零索引不能省略。

`getelementptr` 只计算地址，不负责读取或写入内存。更完整的语义可以参考 [The Often Misunderstood GEP Instruction](https://llvm.org/docs/GetElementPtr.html)。

#### `extractvalue` 和 `insertvalue`：操作寄存器中的聚合值

如果结构体已经被加载到虚拟寄存器中，它是一个值而不是指针，此时不能用 `getelementptr`：

```llvm
%MyStruct = type { i32, i32 }
@my_struct = global %MyStruct { i32 1, i32 2 }

define i32 @main() {
    %value = load %MyStruct, ptr @my_struct
    %second = extractvalue %MyStruct %value, 1
    %updated = insertvalue %MyStruct %value, i32 233, 1
    ret i32 0
}
```

`extractvalue` 取出聚合值中的字段，`insertvalue` 返回一个替换了指定字段的新聚合值；它们也可以用于存放在虚拟寄存器中的数组。

### 标签、元数据与属性

标签用于标识基本块，是控制流跳转的目标。以 `!` 开头的标识符通常表示元数据，例如调试信息和模块标志；调试元数据记录源代码位置等信息，模块标志则记录供模块和工具链使用的附加约束。可以通过 `clang -S -emit-llvm -g test.c` 让生成的 IR 包含更多调试元数据。

属性不是类型，通常用于描述函数的额外性质，例如 `nounwind` 表示函数不会抛出异常：

```llvm
define void @foo() nounwind {
    ret void
}
```

当多个函数共享一组属性时，可以把属性集中定义为属性组，再通过 `#0` 等编号引用：

```llvm
define void @foo() #0 {
    ret void
}

attributes #0 = { nounwind }
```

## 5. 控制流

数据流关注值如何在内存、寄存器和指令之间移动；控制流关注指令执行的先后顺序。条件分支、循环、`switch` 和函数调用都会改变控制流。LLVM 可以据此进行分支布局和函数内联（把被调用函数的代码嵌入调用点）等优化；静态分析工具也需要恢复和遍历这些控制流关系。

### 5.1 控制语句

在汇编层面，常见的高级控制语句都可以拆解为：

- 标签；
- 无条件跳转；
- 比较指令；
- 条件跳转。

#### 比较和跳转

`icmp` 接收比较方案和两个整数操作数，返回 `i1`：

```llvm
%comparison_result = icmp uge i32 %a, %b
```

比较方案包括：

- `eq`、`ne`：相等、不相等；
- `ugt`、`uge`、`ult`、`ule`：无符号的大于、大于等于、小于、小于等于；
- `sgt`、`sge`、`slt`、`sle`：有符号的大于、大于等于、小于、小于等于。

`br` 既能进行条件跳转，也能进行无条件跳转：

```llvm
br i1 %comparison_result, label %true_block, label %false_block
br label %loop_start
```

#### 基本块与终结指令

一个函数由多个基本块（Basic Block）组成。若基本块 A 的终结指令跳转到基本块 B，则 A 是 B 的前驱，B 是 A 的后继。基本块通常包含：

1. 开头的标签，标签可以省略；
2. 一系列顺序执行的指令；
3. 结尾的一条终结指令，例如 `br` 或 `ret`。

如果一个基本块没有显式标签，LLVM 可以为它自动分配标签。但每个基本块都必须以终结指令结束，否则 IR 无法形成完整的控制流图。

下面是一段循环的基本结构：

```llvm
define i32 @main() {
entry:
    %i = alloca i32
    store i32 0, ptr %i
    br label %start

start:
    %i_value = load i32, ptr %i
    %comparison_result = icmp slt i32 %i_value, 4
    br i1 %comparison_result, label %body, label %end

body:
    %next = add i32 %i_value, 1
    store i32 %next, ptr %i
    br label %start

end:
    ret i32 0
}
```

控制流图可以帮助检查每个基本块的前驱和后继。使用 `opt` 生成 `.dot` 文件，再交给 Graphviz 可以得到图像：

```bash
clang -S -emit-llvm for.c -o for.ll
opt -p dot-cfg for.ll
dot .main.dot -Tpng -o for.png
```

上面的命令假定 `for.c` 已经包含要观察的循环示例；`opt` 生成的 `.dot` 文件名可能会随模块名和 LLVM 版本变化。安装 Graphviz 后，再用 `dot` 将它渲染成 PNG。

![main 函数的控制流图](/llvm-ir/cfg-main.png)

#### `switch`

LLVM 的 `switch` 可以根据一个值跳转到多个目标标签。后端会根据目标平台和分支数量，选择一串条件跳转或跳转表等实现方式。跳转表把分支目标放在数组中，再用索引快速找到目标地址；具体机器码由后端决定。

#### `select`

当一个值只需要在两个结果之间选择时，可以使用 `select`，不必为简单的赋值建立完整的分支：

```llvm
define i32 @choose(i32 %x) {
    %positive = icmp sgt i32 %x, 0
    %y = select i1 %positive, i32 1, i32 2
    ret i32 %y
}
```

`select` 的第一个参数是 `i1` 条件；条件为 `true` 时返回第二个值，否则返回第三个值。在适合的目标平台上，它可能被编译成条件移动指令。

#### `phi`

`phi` 根据控制流来自哪个前驱基本块来选择值，因此可以表达多个分支汇合后的 SSA 值：

```llvm
define i32 @choose(i32 %x) {
    %positive = icmp sgt i32 %x, 0
    br i1 %positive, label %true_block, label %false_block

true_block:
    br label %end

false_block:
    br label %end

end:
    %y = phi i32 [1, %true_block], [2, %false_block]
    ret i32 %y
}
```

`select` 根据条件值选择，`phi` 根据控制流前驱选择；`phi` 可以接收两个以上的 `[值, 基本块]` 对。

### 5.2 函数

#### 定义、声明与调用

函数定义由返回类型、函数名、参数列表和函数体组成：

```llvm
define i32 @foo(i32 %a, i64 %b) {
    ret i32 0
}
```

如果当前模块要调用另一个模块中定义的函数，需要先用 `declare` 声明它：

```llvm
declare i32 @printf(ptr, ...)
```

调用使用 `call` 指令：

```llvm
define i32 @foo(i32 %a) {
    ret i32 %a
}

define i32 @bar() {
    %result = call i32 @foo(i32 1)
    ret i32 %result
}
```

一次调用涉及参数传递、进入被调用函数以及取得返回值。调用约定是调用者与被调用者对参数、返回值和寄存器保存位置达成的规则。具体参数和返回值放在哪些寄存器或栈位置，由调用约定决定。LLVM IR 默认使用 C 调用约定；更多约定可以参考 [Calling Conventions](https://llvm.org/docs/LangRef.html#calling-conventions)。

#### 尾调用与 `fastcc`

如果一个函数在返回前调用另一个函数，且调用结果直接作为返回结果，就可能形成尾调用。普通调用通常可能需要新的栈帧，具体取决于目标平台、调用约定和优化级别；尾调用优化则尝试复用当前栈帧，把调用转换为类似循环的跳转，从而减少栈的增长。

`fastcc` 会使用更激进的参数传递策略，为尾调用优化提供条件。以递归函数为例，优化后的汇编可能在尾部使用 `jmp` 而不是 `call`：

```asm
foo:
    cmpl    $1, %edi
    jne     .LBB0_2
    movl    $1, %eax
    retq
.LBB0_2:
    decl    %edi
    jmp     foo
```

是否进行尾调用优化仍由调用约定、目标平台、函数属性和后端判断共同决定。可以使用 `llc tail_call_test.ll -tailcallopt` 进行实验，并参考 [Sibling Call Optimization](https://llvm.org/docs/CodeGenerator.html#sibling-call-optimization)。

#### 调用图

和控制流图类似，LLVM 工具链也可以生成函数调用图：

```bash
clang -S -emit-llvm cg.c -o cg.ll
opt -p dot-callgraph cg.ll
dot cg.ll.callgraph.dot -Tpng -o cg.png
```

上面的命令假定 `cg.c` 已经包含要观察的函数调用关系；实际生成的 `.dot` 文件名可能随 LLVM 版本变化。

![函数调用图](/llvm-ir/call-graph.png)
