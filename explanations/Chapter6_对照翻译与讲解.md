# 第 6 章 Functional Representations of SSA（SSA 的函数式表示）

原书：*SSA-based Compiler Design*，Chapter 6，Lennart Beringer。

说明：本文按“英文原文 → 中文翻译 → 理解与补充”的顺序整理。术语首次出现时给出英文；后文主要使用中文术语。当前内容对应 PDF 第 1–2 页，即原书第 63–64 页。

## 本章脉络

本章的核心观点是：控制流图中的**支配关系（dominance）**，对应于函数式语言中的**语法作用域（syntactic scope）**。沿着这种对应关系，可以把 SSA 的变量定义、φ 函数、控制流汇合与循环等结构表示成函数式语言中的变量绑定、函数参数、续延和局部递归函数。

全章包括以下内容：

- **6.1 Low-Level Functional Program Representations（低层函数式程序表示）**
  - 变量赋值与变量绑定；
  - continuation 与 CPS 控制流；
  - 直接风格与尾递归局部函数；
  - let-normal form。
- **6.2 Functional Construction and Destruction of SSA（SSA 的函数式构造与消除）**
  - 基于活跃性分析的初始构造；
  - λ-dropping、块下沉与参数删除；
  - 嵌套结构、支配关系与循环闭合；
  - SSA 消除。
- **6.3 Refined Block Sinking and Loop Nesting Forests（改进的块下沉与循环嵌套森林）**。
- **6.4 Further Reading and Concluding Remarks（扩展阅读与总结）**。

可以先把本章最重要的对应关系写成：

```text
SSA / 命令式表示                         函数式表示
定义支配一次使用          <------>       变量绑定的作用域包含该变量的出现位置
φ 函数                    <------>       汇合函数或 continuation 的形式参数
跳转到基本块              <------>       尾调用局部函数
控制流图中的支配树        <------>       嵌套函数形成的语法作用域树
```

---

## 章节导言 — PDF 第 1–2 页

### English

This chapter discusses alternative representations of SSA using the terminology and structuring mechanisms of functional programming languages. The reading of SSA as a discipline of functional programming arises from a correspondence between dominance and syntactic scope that subsequently extends to numerous aspects of control and data-flow structure.

### 中文翻译

本章使用函数式编程语言的术语和结构化机制，讨论 SSA 的其他表示方式。之所以可以把 SSA 理解为一种函数式编程规范，是因为控制流图中的“支配关系”与函数式语言中的“语法作用域”存在对应关系；这种对应还可以进一步推广到控制流和数据流结构的许多方面。

### 理解与补充

这是全章最核心的观点：

- 在 SSA 中，一个变量的定义必须支配它的使用；
- 在函数式语言中，一个变量的使用必须位于相应绑定的作用域内。

因而可以近似理解为：

```text
定义支配使用  <------>  绑定的作用域包含变量的出现位置
```

例如：

```text
let x = 10 in
    x + 1
end
```

`x` 的绑定包围着表达式 `x + 1`。对应到控制流图，就是 `x` 的定义支配其使用位置。

---

## 为什么研究 SSA 的函数式表示

### English

The development of functional representations of SSA is motivated by the following considerations:

### 中文翻译

发展 SSA 的函数式表示，主要有以下几个动机。

### 1. 从其他研究领域理解 SSA

#### English

Relating the core ideas of SSA to concepts from other areas of compiler and programming language research provides conceptual insights into the SSA discipline and thus contributes to a better understanding of the practical appeal of SSA to compiler writers.

#### 中文翻译

把 SSA 的核心思想与编译器及程序语言研究中其他领域的概念联系起来，可以帮助我们从概念层面理解 SSA，从而更好地说明为什么 SSA 对编译器开发者具有如此强的实用吸引力。

#### 理解与补充

SSA 不只是一种“给变量加编号”的技巧。它与以下概念都有紧密联系：

- 词法作用域与变量绑定；
- 函数参数；
- continuation（续延）；
- 尾调用；
- 函数式程序的等式推理；
- 类型系统和程序分析。

这些联系可以解释为什么 SSA 很适合优化：它把许多原本隐藏在控制流中的数据依赖显式化了。

### 2. 让 SSA 的隐含约束变成语法结构

#### English

Reformulating SSA as a functional program makes explicit some of the syntactic conditions and semantic invariants that are implicit in the definition and use of SSA. Indeed, the introduction of SSA itself was motivated by a similar goal: to represent aspects of program structure—namely the def-use relationships—explicitly in syntax, by enforcing a particular naming discipline.

In a similar way, functional representations directly enforce invariants such as “all φ-functions in a block must be of the same arity,” “the variables assigned to by these φ-functions must be distinct,” “φ-functions are only allowed to occur at the beginning of a basic block,” or “each use of a variable should be dominated by its (unique) definition.”

Constraints such as these would typically have to be validated or (re-)established after each optimization phase of an SSA-based compiler, but are typically enforced by construction if a functional representation is chosen. Consequently, less code is required, improving the robustness, maintainability, and code readability of the compiler.

#### 中文翻译

把 SSA 重新表述成函数式程序，可以显式呈现一些原本隐含在 SSA 定义和使用方式中的语法条件与语义不变量。事实上，SSA 本身的提出也出于类似目的：通过强制采用特定的变量命名规范，把程序结构中的“定义—使用关系”显式编码进语法之中。

类似地，函数式表示可以直接保证以下不变量：

- 同一基本块中的所有 φ 函数必须具有相同的元数；
- 这些 φ 函数所定义的目标变量必须彼此不同；
- φ 函数只能出现在基本块的开头；
- 变量的每次使用，都应当被它唯一的定义所支配。

在传统的 SSA 编译器中，每次优化阶段结束后，通常都必须检查或重新建立这些约束；而采用函数式表示时，这些性质一般可以通过程序的构造方式自动得到保证。因此，编译器所需的辅助代码更少，健壮性、可维护性和代码可读性也会相应提高。

#### 理解与补充

这里的关键词是 **enforced by construction（由构造保证）**。

传统 SSA 数据结构可能允许构造出非法状态，然后再通过验证器发现问题。函数式抽象则可以让非法状态在语法或数据类型层面难以表达。例如，把汇合块表示成带参数的局部函数后，所有“φ 结果变量”自然成为函数的形式参数，不再需要独立维护一组 φ 指令。

这里的**元数（arity）**指参数的数量。例如，一个汇合块有三个前驱，那么该块中的每个 φ 函数通常都需要三个输入。

### 3. 给 φ 指令提供具体执行模型

#### English

The intuitive meaning of “unimplementable” φ-instructions is complemented by a concrete execution model, facilitating the rapid implementation of interpreters. This enables the compiler developers to experimentally validate SSA-based analyses and transformations at their genuine language level, without requiring SSA destruction.

Indeed, functional intermediate code can often be directly emitted as a program in a high-level mainstream functional language, giving the compiler writer access to existing interpreters and compilation frameworks. Thus, rapid prototyping is supported and high-level evaluation of design decisions is enabled.

#### 中文翻译

函数式表示为通常被认为“不可直接执行”的 φ 指令补充了一个具体执行模型，从而便于快速实现解释器。这样，编译器开发者便可以直接在 SSA 所处的语言层次上实验性地验证各种分析与变换，而不必先执行 SSA 消除。

实际上，函数式中间代码通常可以直接输出成某种主流高级函数式语言的程序，使编译器开发者能够复用现有的解释器和编译框架。因此，这种表示有利于快速原型开发，也便于从较高层次评估各种设计决策。

#### 理解与补充

φ 指令不是普通的运行时指令：

```text
x3 = φ(x1, x2)
```

它的含义取决于控制流从哪个前驱到达当前基本块。进入机器代码阶段后，φ 通常需要被消除并转换成控制流边上的复制操作。

函数式表示可以将它改写成函数参数：

```text
function join(x3) =
    use(x3)

if cond then
    join(x1)
else
    join(x2)
```

这样，“选择哪一个 φ 参数”就变成普通的函数调用和参数传递，因而获得了直接、明确的执行语义。

### 4. 复用函数式语言的程序分析理论

#### English

Formal frameworks of program analysis that exist for functional languages become applicable. Type systems provide a particularly attractive formalism due to their declarativeness and compositional nature. As type systems for functional languages typically support higher-order functions, they can be expected to generalize more easily to interprocedural analyses than other static analysis formalisms.

#### 中文翻译

针对函数式语言建立的程序分析形式化框架也可以被应用到 SSA 上。类型系统尤其具有吸引力，因为它具有声明式和组合式的特点。函数式语言的类型系统通常支持高阶函数，因此，与其他静态分析形式相比，它们有望更容易推广到过程间分析。

#### 理解与补充

“组合式”意味着：分析整个程序时，可以从各个子表达式或子函数的分析结果组合出整体结果，而不必每次都从全局控制流重新开始。

高阶函数还能把 continuation、回调和函数参数统一纳入分析，因此这种框架更容易跨越过程边界。

### 5. 建立比较和转换不同 SSA 变体的形式基础

#### English

We obtain a formal basis for comparing variants of SSA—such as the variants discussed elsewhere in this book—for translating between these variants, and for constructing and destructing SSA. Correctness criteria for program analyses and associated transformations can be stated in a uniform manner and can be proved to be satisfied using well-established reasoning principles.

#### 中文翻译

我们由此获得了一个形式化基础，可用于比较不同的 SSA 变体（包括本书其他章节讨论的变体）、在这些变体之间进行转换，以及构造和消除 SSA。程序分析及其相关变换的正确性条件可以采用统一方式表述，并利用成熟的推理原则证明这些条件确实成立。

#### 理解与补充

函数式表示在这里充当一种共同语言。不同 SSA 变体只要能转换成同一种函数式核心表示，就可以在统一框架下比较：

- 它们保存了哪些控制流和数据流信息；
- 构造或消除时需要满足什么条件；
- 某项优化是否保持程序语义；
- 两种 SSA 变体之间的转换是否正确。

---

## 本章范围

### English

Rather than discussing all these considerations in detail, the purpose of the present chapter is to informally highlight particular aspects of the correspondence and then point the reader to some more advanced material. Our exposition is example-driven but leads to the identification of concrete correspondence pairs between the imperative/SSA world and the functional world.

Like the remainder of the book, our discussion is restricted to code occurring in a single procedure.

### 中文翻译

本章并不打算详细讨论上述所有动机，而是以非形式化方式突出这种对应关系的若干重要方面，并为读者指向一些更深入的材料。本章以示例为主要讲解手段，但最终会归纳出命令式语言／SSA 世界与函数式语言世界之间的一些具体对应关系。

与本书其他章节一样，本章的讨论仅限于单个过程内部的代码。

### 理解与补充

本章不是从严格的形式语义和证明开始，而是先借助代码示例建立直觉。讨论范围限定在单个过程内，因此暂时不处理完整的跨过程控制流、调用图和分别编译问题。

---

## 6.1 Low-Level Functional Program Representations（低层函数式程序表示）— PDF 第 2 页

### English

Functional languages represent code using declarations of the form

```text
function f(x₀, ..., xₙ) = e                         (6.1)
```

where the syntactic category of expression `e` conflates the notions of expressions and commands of imperative languages. Typically, `e` may contain further nested or (mutually) recursive function declarations. A declaration of the form (6.1) binds the formal parameters `xᵢ` and the function name `f` within `e`.

### 中文翻译

函数式语言使用如下形式的声明表示代码：

```text
function f(x₀, ..., xₙ) = e                         (6.1)
```

其中，表达式 `e` 这一语法类别同时承担了命令式语言中“表达式”和“命令”两种结构的作用。通常，`e` 中还可以包含嵌套函数声明或相互递归的函数声明。形如式（6.1）的声明，会在 `e` 内绑定形式参数 `xᵢ` 以及函数名 `f`。

### 理解与补充

命令式语言通常区分：

```text
x + 1        // 表达式：产生值
x = x + 1    // 命令：改变状态
```

纯函数式语言更倾向于让所有结构都产生一个值，所以条件、绑定甚至控制转移都可以作为表达式组织起来。

函数名 `f` 在函数体 `e` 中也处于绑定状态，这使递归调用成为可能：

```text
function f(x) =
    if x == 0 then 1
    else x * f(x - 1)
```

这一点对后文非常重要：作者将使用嵌套的局部递归函数表示基本块和控制流边。

---

## 本节术语表

| 英文术语 | 中文译法 | 本章中的含义 |
|---|---|---|
| functional representation | 函数式表示 | 使用绑定、函数与参数表示 SSA 结构 |
| dominance | 支配关系 | 从过程入口到某节点的每条路径都经过支配节点 |
| syntactic scope | 语法作用域 | 某个绑定对变量名有效的程序文本区域 |
| invariant | 不变量 | 构造和优化过程中必须持续成立的性质 |
| arity | 元数 | 函数、φ 函数或基本块参数的数量 |
| enforced by construction | 由构造保证 | 借助语法或数据结构设计，使非法状态无法表达或难以产生 |
| continuation | 续延 | 描述当前计算结束后如何继续执行的函数式对象 |
| interprocedural analysis | 过程间分析 | 跨越函数或过程边界进行的程序分析 |
| SSA construction | SSA 构造 | 把普通中间表示转换成 SSA |
| SSA destruction | SSA 消除 | 消除 φ 函数并转换回非 SSA 表示 |
| recursive function | 递归函数 | 可在函数体中调用自身的函数 |
| mutually recursive functions | 相互递归函数 | 两个或多个函数通过相互调用形成递归 |

## 阶段性小结

本章开头建立了后续推导所需的观察：

1. SSA 的“定义支配使用”可以由函数式语言的作用域规则直接表达；
2. 基本块开头的一组 φ 函数可以整体改写成函数的形式参数；
3. 跳转到某个汇合块可以改写成对相应函数的调用；
4. 这样得到的表示不仅能表达 SSA，还能借用函数式语言的执行模型、类型系统和等式推理方法。

下一部分将从 `let` 绑定入手，具体比较命令式赋值与函数式变量绑定。
