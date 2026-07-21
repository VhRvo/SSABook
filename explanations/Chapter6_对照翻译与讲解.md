# 第 6 章 Functional Representations of SSA（SSA 的函数式表示）

原书：*SSA-based Compiler Design*，Chapter 6，Lennart Beringer。

说明：本文按“英文原文 → 中文翻译 → 理解与补充”的顺序整理。英文部分保留原段落的完整论述，中文部分逐句翻译其中的条件、限定和因果关系；额外阐释只放在“理解与补充”或图解中。术语首次出现时给出英文，后文主要使用中文术语。内容覆盖所提供 PDF 全部 26 页，即原书第 63–88 页。

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
φ 函数                  <------>       汇合函数或 continuation 的形式参数
跳转到基本块             <------>       尾调用局部函数
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

## 6.1.1 Variable Assignment Versus Binding（变量赋值与变量绑定）— PDF 第 3–4 页

### English

A language construct provided by almost all functional languages is the let-binding:

```text
let x = e₁ in e₂ end
```

The effect of this expression is to evaluate `e₁` and bind the resulting value to variable `x` for the duration of the evaluation of `e₂`. The code affected by this binding, `e₂`, is called the static scope of `x` and is easily syntactically identifiable. In the following, we occasionally indicate scopes by code-enclosing boxes and list the variables that are in scope using subscripts.

### 中文翻译

几乎所有函数式语言都提供 `let` 绑定：

```text
let x = e₁ in e₂ end
```

这个表达式首先对 `e₁` 求值，然后在对 `e₂` 求值期间，把所得结果绑定到变量 `x`。受该绑定影响的代码 `e₂` 称为 `x` 的**静态作用域（static scope）**，可以直接从语法结构中识别。后文有时会用包围代码的方框表示作用域，并用下标列出当前处于作用域中的变量。

### English

For example, the scope associated with the top-most binding of `v` to `3` in code

```text
let v = 3 in
    let y = (let v = 2 × v in 4 × v end)
    in y × v end
end                                                     (6.2)
```

spans both inner let-bindings, the scopes of which are themselves not nested inside one other as the inner binding of `v` occurs in the `e₁` position of the let-binding for `y`.

### 中文翻译

例如，在代码（6.2）中，最外层把 `v` 绑定为 `3`，它的作用域覆盖里面的两个 `let` 绑定。不过，里面两个绑定的作用域并不是互相嵌套的，因为内层对 `v` 的绑定出现在 `y` 的 `let` 绑定的 `e₁` 位置，也就是用于计算 `y` 初值的表达式中。

### English

In contrast to an assignment in an imperative language, a let-binding for variable `x` hides any previous value bound to `x` for the duration of evaluating `e₂` but does not permanently overwrite it. Bindings are treated in a stack-like fashion, resulting in a tree-shaped nesting structure of boxes in our code excerpts.

For example, in the above code, the inner binding of `v` to value `2 × 3 = 6` shadows the outer binding of `v` to value `3` precisely for the duration of the evaluation of the expression `4 × v`. Once this evaluation has terminated (resulting in the binding of `y` to `24`), the binding of `v` to `3` becomes visible again, yielding the overall result of `72`.

### 中文翻译

命令式赋值会覆盖变量原来的值；与之不同，`let` 对变量 `x` 的绑定只在计算 `e₂` 的期间隐藏 `x` 之前的绑定，并不会永久覆盖它。绑定按照类似栈的方式管理，因此作用域形成树状嵌套结构。

在上面的例子中，内层 `v` 的值是 `2 × 3 = 6`。这个绑定只在计算 `4 × v` 时遮蔽外层值 `3`，因此 `y = 4 × 6 = 24`。该计算结束后，外层 `v = 3` 再次可见，最终结果为 `y × v = 24 × 3 = 72`。

### 理解与补充

这里要区分两个概念：

- **赋值（assignment）**改变一个存储位置当前保存的值；
- **绑定（binding）**在一个明确的语法区域内，让名字指向某个值。

同名绑定的嵌套称为**遮蔽（shadowing）**。它不会改变外层绑定，只是使外层名字暂时不可见。

### English

The concepts of binding and static scope ensure that functional programs enjoy the characteristic feature of SSA, namely the fact that each use of a variable is uniquely associated with a point of definition. Indeed, the point of definition for a use of `x` is given by the nearest enclosing binding of `x`. Occurrences of variables in an expression that are not enclosed by a binding are called free. A well-formed procedure declaration contains all free variables of its body among its formal parameters. Thus, the notion of scope makes explicit the invariant that each use of a variable should be dominated by its (unique) definition.

### 中文翻译

绑定和静态作用域保证函数式程序具有 SSA 的典型性质：变量的每次使用都唯一对应一个定义点。具体来说，一次 `x` 的使用所对应的定义，就是包围它的、距离最近的 `x` 绑定。

表达式中没有被任何绑定包围的变量出现称为**自由出现（free occurrence）**。一个良构的过程声明，会把函数体中的所有自由变量都列入形式参数。因此，作用域把“变量的每次使用都应被其唯一的定义支配”这一不变量显式表达出来。

### English

In contrast to SSA, functional languages achieve the association of definitions to uses without imposing the global uniqueness of variables, as witnessed by the duplicate binding occurrences for `v` in the above code. As a consequence of this decoupling, functional languages enjoy a strong notion of referential transparency: The choice of `x` as the variable holding the result of `e₁` depends only on the free variables of `e₂`.

For example, we may rename the inner `v` in code (6.2) to `z` without altering the meaning of the code:

```text
let v = 3 in
    let y = (let z = 2 × v in 4 × z end)
    in y × v end
end                                                     (6.3)
```

Note that this conversion formally makes the outer `v` visible for the expression `4 × z`.

### 中文翻译

函数式语言与 SSA 不同：它不要求变量名在整个程序中全局唯一，也能建立定义与使用之间的唯一对应关系，上例中两个同名的 `v` 绑定就说明了这一点。

这种解耦带来了很强的**引用透明性（referential transparency）**：用于保存 `e₁` 结果的变量名如何选择，只取决于 `e₂` 中的自由变量。比如，可以把代码（6.2）内层的 `v` 改名为 `z`，得到代码（6.3），程序含义不变。形式上，改名后外层的 `v` 在表达式 `4 × z` 中重新可见，但该表达式并没有使用它。

### English

In order to avoid altering the meaning of the program, the choice of the newly introduced variable has to be such that confusion with other variables is avoided. Formally, this means that a renaming

```text
let x = e₁ in e₂ end
```

to

```text
let y = e₁ in e₂[y ↔ x] end
```

can only be carried out if `y` is not a free variable of `e₂`. Moreover, in the event that `e₂` already contains some preexisting bindings to `y`, the substitution of `x` by `y` in `e₂` (denoted by `e₂[y ↔ x]` above) first renames these preexisting bindings in a suitable manner. Also note that the renaming only affects `e₂`—any occurrences of `x` or `y` in `e₁` refer to conceptually different but identically named variables, but the static scoping discipline ensures these will never be confused with the variables involved in the renaming. In general, the semantics-preserving renaming of bound variables is called α-renaming.

Typically, program analyses for functional languages are compatible with α-renaming in that they behave equivalently for fragments that differ only in their choice of bound variables, and program transformations α-rename bound variables whenever necessary.

### 中文翻译

为了避免改变程序含义，新变量名必须避开可能发生的名字混淆。形式上，把

```text
let x = e₁ in e₂ end
```

改写成

```text
let y = e₁ in e₂[y ↔ x] end
```

只能在 `y` 不是 `e₂` 的自由变量时进行。如果 `e₂` 中已经存在一些对 `y` 的绑定，那么在用 `y` 替换 `e₂` 中由外层绑定引入的 `x` 之前，必须先以适当方式重命名这些已有的 `y` 绑定；上式用 `e₂[y ↔ x]` 表示这一避免捕获的替换过程。

还要注意，重命名只影响 `e₂`，不会修改 `e₁`。`e₁` 中即使也出现名字 `x` 或 `y`，它们也表示概念上不同、只是恰好同名的变量；静态作用域规则保证这些变量不会与当前正在重命名的绑定混淆。

这种保持语义的已绑定变量重命名统称为 **α-renaming（α-重命名）**。函数式语言的程序分析通常必须与 α-重命名兼容：如果两段程序只是在已绑定变量的名字选择上不同，分析结果应当等价。程序变换也会在需要避免变量冲突或捕获时主动执行 α-重命名。

### 理解与补充

α-重命名的本质不是简单的文本替换，而是按绑定关系改名。比如：

```text
let x = 1 in x + y end
```

不能直接把 `x` 改成已经自由出现的 `y`，否则会发生**变量捕获（variable capture）**，把原来不同的两个变量错误地合并。

### English

A consequence of referential transparency, and thus a property typically enjoyed by functional languages, is compositional equational reasoning: the meaning of a piece of code `e` is only dependent on its free variables and can be calculated from the meaning of its subexpressions. For example, the meaning of a phrase `let x = e₁ in e₂ end` only depends on the free variables of `e₁` and on the free variables of `e₂` other than `x`. Hence, languages with referential transparency allow one to replace a subexpression by some semantically equivalent phrase without altering the meaning of the surrounding code. Since semantic preservation is a core requirement of program transformations, the suitability of SSA for formulating and implementing such transformations can be explained by the proximity of SSA to functional languages.

### 中文翻译

引用透明性的一个结果——也是函数式语言通常具有的性质——是可以进行**组合式等式推理（compositional equational reasoning）**。一段代码 `e` 的含义只取决于它的自由变量，并且能够由各个子表达式的含义组合计算出来。

例如，短语 `let x = e₁ in e₂ end` 的含义，只取决于 `e₁` 的所有自由变量，以及 `e₂` 中除 `x` 之外的自由变量。之所以排除 `x`，是因为 `e₂` 中相应的 `x` 已经由当前 `let` 绑定，其值来自 `e₁`，不再依赖外部环境中的同名变量。

因此，在具有引用透明性的语言中，可以用语义等价的表达式替换某个子表达式，而不改变周围代码的含义。保持语义又是程序变换的核心要求，所以 SSA 之所以特别适合描述和实现优化，可以用它与函数式语言的接近程度来解释。

---

## 6.1.2 Control Flow: Continuations（控制流：续延）— PDF 第 5–7 页

### English

The correspondence between let-bindings and points of variable definition in assignments extends to other aspects of program structure, in particular to code in continuation-passing style (CPS), a program representation routinely used in compilers for functional languages.

Satisfying a roughly similar purpose as return addresses or function pointers in imperative languages, a continuation specifies how the execution should proceed once the evaluation of the current code fragment has terminated. Syntactically, continuations are expressions that may occur in functional position, as is the case for the variable `k` in the following modification of code (6.2):

```text
let v = 3 in
    let y = (let v = 2 × v in 4 × v end)
    in k(y × v) end
end                                                     (6.4)
```

In effect, `k` represents any function that may be applied to the result of expression (6.2).

### 中文翻译

`let` 绑定与赋值定义点之间的对应关系，还可以扩展到其他程序结构，特别是**续延传递风格（continuation-passing style，CPS）**。CPS 是函数式语言编译器经常使用的一种程序表示。

续延的作用大致类似于命令式语言中的返回地址或函数指针：它规定当前代码片段计算结束后，程序应当怎样继续执行。在语法上，续延是可以出现在函数位置、接受参数的表达式。代码（6.4）中的变量 `k` 就表示任意一个可以作用于代码（6.2）计算结果的函数。

### English

Surrounding code may specify the concrete continuation by binding `k` to a suitable expression. It is common practice to write these continuation-defining expressions in λ-notation, in the form `λx.e`. The formal parameter `x` represents the placeholder for the argument to which the continuation is applied. Note that `x` is α-renameable, as λ acts as a binder.

For example, a client wishing to multiply the result by `2` may bind `k` to `λx.2 × x`:

```text
let k = λx.2 × x
in let v = 3 in
       let y = (let z = 2 × v in 4 × z end)
       in k(y × v) end
   end
end                                                     (6.5)
```

When `k` is applied to `y × v`, the dynamic value `72` is substituted for `x` in `2 × x`, just like in an ordinary function application.

### 中文翻译

外围代码可以把 `k` 绑定到一个适当的表达式，从而给出具体续延。续延通常使用 λ 表示法书写，即 `λx.e`。形式参数 `x` 是续延实参的占位符；由于 λ 是绑定结构，`x` 可以进行 α-重命名。

例如，如果调用方希望把原计算结果乘以 `2`，就可以令 `k = λx.2 × x`，如代码（6.5）所示。调用 `k(y × v)` 时，运行时的值 `72` 被代入表达式 `2 × x` 中，过程与普通函数调用完全相同，最终得到 `144`。

### English

Alternatively, the client may wrap fragment (6.4) in a function definition with formal argument `k` and construct the continuation in the calling code:

```text
function f(k) =
    let v = 3 in
        let y = (let z = 2 × v in 4 × z end)
        in k(y × v) end
    end
in let k = λx.2 × x in f(k) end
end                                                     (6.6)
```

This makes CPS a discipline of programming using higher-order functions, as continuations are constructed “on the fly” and communicated as arguments of other function calls.

### 中文翻译

另一种做法是把代码（6.4）包装成以 `k` 为形式参数的函数，并在调用处构造续延，如代码（6.6）所示。

由此可见，CPS 是一种使用**高阶函数**的编程规范：续延在运行过程中动态构造，并作为其他函数调用的参数传递。

### English

Typically, the caller of `f` is itself parametric in its continuation, as in

```text
function g(k) =
    let k′ = λx.k(x + 7) in f(k′) end                  (6.7)
```

where `f` is invoked with a newly constructed continuation `k′` that adds `7` to its formal argument before passing the resulting value to the outer continuation `k`.

### 中文翻译

通常，`f` 的调用者自身也把续延作为参数，如代码（6.7）。函数 `g` 构造新续延 `k′`：先给 `f` 的结果加 `7`，再把所得值传给外层续延 `k`。

换句话说，`k′` 把“加 7”这一后续步骤插入当前计算与外层后续计算之间。

### English

In a similar way, the function

```text
function h(y, k) =
    let x = 4 in
        let k′ = λz.k(z × x)
        in if y > 0
           then let z = y × 2 in k′(z) end
           else let z = 3 in k′(z) end
           end
    end                                                 (6.8)
```

constructs from `k` a continuation `k′` that is invoked with different arguments in each branch of the conditional. In effect, the sharing of `k′` amounts to the definition of a control-flow merge point, as indicated by the CFG corresponding to `h` in Fig. 6.1a. Contrary to the functional representation, the top-level continuation parameter `k` is not explicitly visible in the CFG; it roughly corresponds to the frame slot that holds the return address in an imperative procedure call.

### 中文翻译

类似地，函数（6.8）根据 `k` 构造续延 `k′`，并在条件语句的两个分支中用不同参数调用它。两个分支共享同一个 `k′`，实际上就定义了一个控制流汇合点，这与图 6.1a 中 `h` 的 CFG 相对应。

顶层续延参数 `k` 在 CFG 中没有显式出现；它大致对应命令式过程调用栈帧中保存返回地址的槽位。

### 图 6.1 解读：续延参数就是 φ 结果

图 6.1a 的普通 CFG 为：

```text
             y
             |
      x ← 4; if (y > 0)
          /             \
     z ← y × 2         z ← 3
          \             /
            return z × x
```

重命名成 SSA 后得到图 6.1b：

```text
          z₁ ← y × 2       z₂ ← 3
                 \         /
                 z ← φ(z₁, z₂)
                 return z × x
```

对应的函数式代码只需对两个分支的局部绑定做 α-重命名：

```text
function h(y, k) =
    let x = 4 in
        let k′ = λz.k(z × x)
        in if y > 0
           then let z₁ = y × 2 in k′(z₁) end
           else let z₂ = 3 in k′(z₂) end
           end
    end                                                 (6.9)
```

续延 `k′` 的形式参数 `z` 与 φ 函数的结果完全对应：来自多个调用点的实参 `z₁`、`z₂` 被统一绑定到公共名字 `z`，供汇合点之后的代码使用。

`z₁` 和 `z₂` 的作用域分别在调用 `k′` 时结束，这也对应命令式变量的支配区域在跳转到汇合点时结束。从（6.8）到（6.9）只需要保持引用透明性的 α-重命名，说明（6.8）本来就包含 SSA 从命令式程序中提炼出的关键结构。

### English

Programs in CPS equip all function declarations with continuation arguments. By interspersing ordinary code with continuation-forming expressions, they model the flow of control exclusively by communicating, constructing, and invoking continuations.

### 中文翻译

CPS 程序会为所有函数声明增加续延参数。普通计算与续延构造表达式交错出现，程序完全通过传递、构造和调用续延来表达控制流。

---

## 6.1.3 Control Flow: Direct Style（控制流：直接风格）— PDF 第 8–9 页

### English

An alternative to the explicit passing of continuation terms via additional function arguments is the direct style, in which we represent code as a set of locally named tail-recursive functions, for which the last operation is a call to a function, eliminating the need to save the return address.

In direct style, no continuation terms are constructed dynamically and then passed as function arguments, and we hence exclusively employ λ-free function definitions in our representation.

### 中文翻译

除了通过额外参数显式传递续延，还可以采用**直接风格（direct style）**：把代码表示成一组有局部名字的尾递归函数，每个函数的最后一个操作是调用另一个函数，因此不必保存返回地址。

直接风格不会动态构造续延并把它作为函数参数传递，所以这里仅使用不含 λ 表达式的函数定义。

### English

For example, code (6.8) may be represented as

```text
function h(y) =
    let x = 4 in
        function h′(z) = z × x
        in if y > 0
           then let z = y × 2 in h′(z) end
           else let z = 3 in h′(z) end
           end
    end                                                 (6.10)
```

where the local function `h′` plays a similar role to the continuation `k′` and is jointly called from both branches. In contrast to the CPS representation, however, the body of `h′` returns its result directly rather than passing it on as an argument to some continuation. Neither `h` nor `h′` contains additional continuation parameters.

Thus, rather than handing its result directly over to some caller-specified receiver, `h` simply returns control to the caller, who is then responsible for any further execution. Roughly speaking, the effect is similar to always setting the return address of a procedure call to the instruction pointer immediately following the call instruction.

### 中文翻译

代码（6.8）可以改写成（6.10）。局部函数 `h′` 与之前的续延 `k′` 作用相似，并由条件的两个分支共同调用。

不过，与 CPS 不同，`h′` 的函数体直接返回结果，而不是把结果作为参数传给另一个续延；`h` 和 `h′` 也都不包含额外的续延参数。因此，`h` 不会把结果直接交给调用者指定的接收者，而只是把控制权返回调用者，由调用者决定后续执行。粗略地说，这类似于把过程调用的返回地址固定为调用指令之后的下一条指令。

### English

A stricter format is obtained if the granularity of local functions is required to be that of basic blocks:

```text
function h(y) =
    let x = 4 in
        function h′(z) = z × x
        in if y > 0
           then function h₁() =
                    let z = y × 2 in h′(z) end
                in h₁() end
           else function h₂() =
                    let z = 3 in h′(z) end
                in h₂() end
           end
    end                                                 (6.11)
```

Now, function invocations correspond precisely to jumps, reflecting more directly the CFG from Fig. 6.1.

### 中文翻译

如果要求每个局部函数的粒度严格对应一个基本块，就得到代码（6.11）。此时：

- `h₁` 对应条件真分支的基本块；
- `h₂` 对应条件假分支的基本块；
- `h′` 对应控制流汇合块；
- 函数调用精确对应 CFG 中的跳转。

因此，这种形式比（6.10）更直接地反映图 6.1 的控制流图。

### English

The choice between CPS and direct style is orthogonal to the granularity level of functions: both are compatible with the strict notion of basic blocks and with more relaxed formats such as extended basic blocks. In the extreme case, all control-flow points are explicitly named, with one local function or continuation per instruction. Exploration of this design space has not yet led to a clear consensus in the literature.

In the discussion below, we employ the arguably easier-to-read direct style, but the gist applies equally well to CPS.

### 中文翻译

CPS 与直接风格的选择，和函数采用什么粒度彼此独立。两者既可以严格地让一个函数对应一个基本块，也可以采用扩展基本块等更宽松的粒度。极端情况下，可以给每个控制流点显式命名，即每条指令都对应一个局部函数或续延。学术界对这一设计空间仍没有明确共识。

后文将采用相对容易阅读的直接风格，不过讨论的核心同样适用于 CPS。

### English

Independent of the granularity level, moving from the CFG to SSA is again captured by suitably α-renaming the bindings of `z` in `h₁` and `h₂`:

```text
function h(y) =
    let x = 4 in
        function h′(z) = z × x
        in if y > 0
           then function h₁() =
                    let z₁ = y × 2 in h′(z₁) end
                in h₁() end
           else function h₂() =
                    let z₂ = 3 in h′(z₂) end
                in h₂() end
           end
    end                                                 (6.12)
```

Again, the formal parameter `z` of merge-point function `h′` is identical in role to a φ-function. Since the blocks representing the arms of the conditional do not contain φ-functions, `h₁` and `h₂` have empty parameter lists. The free occurrence of `y` in `h₁` is bound at the top level by the formal argument of `h`.

### 中文翻译

无论局部函数采用哪种粒度，从 CFG 转换到 SSA 仍然可以通过适当的 α-重命名表达。代码（6.12）把两个分支中的 `z` 分别改名为 `z₁` 和 `z₂`。

汇合函数 `h′` 的形式参数 `z` 再次扮演 φ 函数的角色。条件两个分支对应的基本块没有 φ 函数，所以 `h₁` 和 `h₂` 的参数列表为空。`h₁` 函数体中自由出现的 `y`，由顶层函数 `h` 的形式参数绑定。

---

## 6.1.4 Let-Normal Form（Let 范式）— PDF 第 10–11 页

### English

For both direct style and CPS the correspondence to SSA is most pronounced for code in let-normal form: Each intermediate result must be explicitly named by a variable, and function arguments must be names or constants.

Syntactically, let-normal form isolates basic instructions in a separate category of primitive terms `a` and requires let-bindings to have the form `let x = a in e end`. In particular, neither jumps, conditional or unconditional, nor let-bindings are primitive.

### 中文翻译

无论直接风格还是 CPS，当代码处于 **let-normal form（let 范式）**时，它与 SSA 的对应最明显：每个中间结果都必须显式地由变量命名，而函数实参只能是变量名或常量。

在语法上，let 范式把基本指令单独划入“原始项 `a`”这一类别，并要求 `let` 绑定具有 `let x = a in e end` 的形式。跳转——无论条件跳转还是无条件跳转——以及其他 `let` 绑定，都不属于原始项。

### English

Let-normalized form is obtained by repeatedly rewriting nested bindings as follows, subject to the side condition that `y` is not free in `e″`:

```text
let x = (let y = eᵧ in e′ end)
in e″ end

        ==>

let y = eᵧ
in let x = e′
   in e″ end
end
```

For example, let-normalizing code (6.3) pulls the binding for `z` outside the binding for `y`:

```text
let v = 3 in
    let z = 2 × v in
        let y = 4 × z in y × v end
    end
end                                                     (6.13)
```

### 中文翻译

反复应用上述改写，就能得到 let 范式，但必须满足附加条件：`y` 不能在 `e″` 中自由出现。这个条件用于防止移动绑定时发生变量捕获。

例如，对代码（6.3）进行 let 规范化，会把 `z` 的绑定从 `y` 的初始化表达式内部提到外面，得到代码（6.13）。

### English

Programs in let-normal form do not contain let-bindings in the `e₁` position of outer let-expressions. The stack discipline is simplified as scopes are nested inside each other. While still enjoying referential transparency, let-normal code is in closer correspondence to imperative code because the chain of nested let-bindings directly reflects the sequence of statements in a basic block, occasionally interspersed by definitions of continuations or local functions.

The remainder of this chapter omits the scope-identifying boxes and abbreviates chains of let-normal bindings by single comma-separated, ordered let-blocks. Under this convention, code (6.13) becomes

```text
let v = 3,
    z = 2 × v,
    y = 4 × z
in y × v end                                           (6.14)
```

### 中文翻译

let 范式程序不会在外层 `let` 的 `e₁` 位置再嵌入一个 `let`。所有绑定的作用域按顺序互相嵌套，使栈式绑定管理更简单。

let 范式仍然保持引用透明性，但比未规范化的函数式代码更接近命令式代码：一串嵌套的 `let` 绑定直接对应基本块中的顺序指令，其间偶尔穿插续延或局部函数的定义。

本章余下部分将省略标识作用域的方框，把一串有顺序的 let 范式绑定缩写成一个用逗号分隔的 `let` 块。代码（6.13）因此缩写成（6.14）。

### 理解与补充

let 范式与 SSA 的对应非常直接：

```text
let x = primitive-operation in ...
```

几乎就是命令式中间表示中的：

```text
x ← primitive-operation
```

差别在于，函数式形式用词法作用域而不是可变存储解释名字。

### 表 6.1：函数式形式与 SSA 的基础对应

| 函数式概念 | 命令式／SSA 概念 |
|---|---|
| `let` 中的变量绑定 | 赋值，即定义点 |
| α-重命名 | 变量重命名 |
| 绑定出现与变量使用之间的唯一对应 | 定义与使用之间的唯一对应 |
| 续延或局部函数的形式参数 | φ 函数，即定义点 |
| 已绑定变量的词法作用域 | 支配区域 |

### 6.1 小结

到这里可以得到一条完整的翻译链：

```text
命令式赋值       -> let 绑定
变量重命名       -> α-重命名
控制流跳转       -> 尾调用局部函数或调用续延
控制流汇合       -> 多处调用同一个函数
φ 函数           -> 汇合函数的形式参数
基本块指令序列   -> let 范式绑定链
```

---

## 6.2 Functional Construction and Destruction of SSA（SSA 的函数式构造与消除）— PDF 第 11–20 页

### 表 6.2：程序结构层面的对应关系

| 函数式概念 | 命令式／SSA 概念 |
|---|---|
| 直接子项关系 | 直接控制流后继关系 |
| 函数 `fᵢ` 的元数 | 基本块 `bᵢ` 开头的 φ 函数数量 |
| `fᵢ` 形式参数彼此不同 | `bᵢ` 的 φ 块中左值变量彼此不同 |
| 函数 `fᵢ` 的调用点数量 | `bᵢ` 中 φ 函数的元数 |
| 参数提升／参数删除 | 添加／删除 φ 函数 |
| 块上浮／块下沉 | 按支配树结构重新排序 |
| 潜在的嵌套结构 | 支配树 |

### English

The relationship between SSA and functional languages is extended by the correspondences shown in Table 6.2. We discuss some of these aspects by considering the translation into SSA, using the program in Fig. 6.2 as a running example.

### 中文翻译

表 6.2 把 SSA 与函数式语言的对应关系进一步扩展到程序结构层面。下面将研究如何把程序转换成 SSA，并始终使用图 6.2 中的程序作为贯穿示例。

### 图 6.2：贯穿示例

原始 CFG 包含三个基本块：

```text
b₁:                         b₂:
    v ← 1                       x ← 5 + y
    z ← 8                       y ← x × z
    y ← 4                       x ← x - 1
       |                        if (x = 0)
       +-----------------------> /       \
                              false      true
                                |          \
                                +--> b₂     b₃:
                                               w ← v + y
                                               return w
```

`b₂` 是循环头和循环体，`b₃` 是退出块。这个例子将展示：最初按活跃变量生成的参数怎样对应 φ 函数，以及 λ-dropping 如何删掉多余参数。

---

## 6.2.1 Initial Construction Using Liveness Analysis（使用活跃性分析进行初始构造）— PDF 第 11–13 页

### English

A simple way to represent this program in let-normalized direct style is to introduce one function `fᵢ` for each basic block `bᵢ`. The body of each `fᵢ` arises by introducing one let-binding for each assignment and converting jumps into function calls. In order to determine the formal parameters of these functions we perform a liveness analysis. For each basic block `bᵢ`, we choose an arbitrary enumeration of its live-in variables. We then use this enumeration as the list of formal parameters in the declaration of the function `fᵢ`, and also as the list of actual arguments in calls to `fᵢ`. We collect all function definitions in a block of mutually tail-recursive functions at the top level:

```text
function f₁() =
    let v = 1, z = 8, y = 4
    in f₂(v, z, y) end

and f₂(v, z, y) =
    let x = 5 + y, y = x × z, x = x - 1
    in if x = 0 then f₃(y, v) else f₂(v, z, y) end

and f₃(y, v) =
    let w = y + v in w end

in f₁() end                                             (6.15)
```

### 中文翻译

用 let 范式的直接风格表示这个程序，一种简单方法是：为每个基本块 `bᵢ` 引入一个函数 `fᵢ`。构造每个 `fᵢ` 的函数体时，把基本块中的每条赋值分别转换成一个 `let` 绑定，并把控制流跳转转换成函数调用。

为了确定这些函数的形式参数，需要先执行活跃性分析。对于每个基本块 `bᵢ`，任意选择一种顺序，枚举它的所有入口活跃变量（live-in variables）。然后，一方面把这份有序枚举作为函数 `fᵢ` 声明中的形式参数列表，另一方面也把同样的顺序用作所有 `fᵢ` 调用的实际参数列表。

最后，在程序顶层把所有函数定义收集到同一个相互尾递归函数块中，得到代码（6.15）。这里“相互尾递归”表示基本块之间的跳转全部位于函数体尾部，因此调用另一个基本块函数之后不需要再返回当前函数继续执行。

### English

The resulting program has the following properties:

- All function declarations are closed: The free variables of their bodies are contained in their formal parameter lists;¹
- Variable names are not unique, but the unique association of definitions to uses is satisfied;
- Each subterm `e₂` of a let-binding `let x = e₁ in e₂ end` corresponds to the direct control-flow successor of the assignment to `x`.

### 中文翻译

所得程序具有以下性质：

- 所有函数声明都是**闭合的（closed）**：函数体中的每个自由变量都包含在该函数的形式参数列表中；¹
- 变量名还不是全局唯一的，但每个变量使用都已经唯一对应一个定义；
- 对于任意 `let x = e₁ in e₂ end`，其中的子项 `e₂` 都对应命令式程序中“对 `x` 的赋值”之后的直接控制流后继。

¹ 原文脚注：这里不计函数标识符 `fᵢ`；这些标识符总可以选择成与普通变量不同的名字。

### English

If desired, we may α-rename to make names globally unique. As the function declarations in code (6.15) are closed, all variable renamings are independent from each other. The resulting code (6.16) corresponds precisely to the SSA program shown in Fig. 6.3 (see also Table 6.2): Each formal parameter of a function `fᵢ` is the target of one φ-function for the corresponding block `bᵢ`. The arguments of these φ-functions are the arguments in the corresponding positions in the calls to `fᵢ`. As the number of arguments in each call to `fᵢ` coincides with the number of formal parameters of `fᵢ`, the φ-functions in `bᵢ` are all of the same arity, namely the number of call sites to `fᵢ`. In order to coordinate the relative positioning of the arguments of the φ-functions, we choose an arbitrary enumeration of these call sites.

```text
function f₁() =
    let v₁ = 1, z₁ = 8, y₁ = 4
    in f₂(v₁, z₁, y₁) end

and f₂(v₂, z₂, y₂) =
    let x₁ = 5 + y₂, y₃ = x₁ × z₂, x₂ = x₁ - 1
    in if x₂ = 0 then f₃(y₃, v₂) else f₂(v₂, z₂, y₃) end

and f₃(y₄, v₃) =
    let w₁ = y₄ + v₃ in w₁ end

in f₁() end                                             (6.16)
```

### 中文翻译

如果需要，还可以执行 α-重命名，使变量名在整个程序中全局唯一。由于代码（6.15）中的每个函数声明都是闭合的，一个函数内部的变量重命名不会影响其他函数，因此所有重命名可以彼此独立地完成。得到的代码（6.16）与图 6.3 所示的 SSA 程序精确对应；表 6.2 中的对应关系也可以在这里直接观察到。

具体来说，函数 `fᵢ` 的每个形式参数，都对应基本块 `bᵢ` 中一个 φ 函数的赋值目标。这个 φ 函数的各个输入参数，则来自每个 `fᵢ` 调用中与该形式参数处于相同位置的实际参数。

每次调用 `fᵢ` 时，实际参数的数量都等于 `fᵢ` 的形式参数数量。因此，`bᵢ` 开头的所有 φ 函数必然具有相同的元数；这个元数并不是 `fᵢ` 的形式参数个数，而是 `fᵢ` 的调用点个数，因为每个调用点为每个 φ 函数贡献一个输入。为了让不同 φ 函数中的输入位置具有一致含义，需要任意选定一种调用点枚举顺序，并让所有 φ 函数都按照这个顺序排列输入。

### 图 6.3 解读：剪枝但非最小的 SSA

按（6.16）解释成 SSA 后：

```text
b₁:
    v₁ ← 1
    z₁ ← 8
    y₁ ← 4

b₂:
    v₂ ← φ(v₁, v₂)
    z₂ ← φ(z₁, z₂)
    y₂ ← φ(y₁, y₃)
    x₁ ← 5 + y₂
    y₃ ← x₁ × z₂
    x₂ ← x₁ - 1
    if x₂ = 0 then b₃ else b₂

b₃:
    y₄ ← φ(y₃)
    v₃ ← φ(v₂)
    w₁ ← v₃ + y₄
    return w₁
```

图中的支配树是一条链：`b₁` 支配 `b₂`，`b₂` 支配 `b₃`。

### English

Under this perspective, the above construction of parameter lists amounts to equipping each `bᵢ` with φ-functions for all its live-in variables, with subsequent renaming of the variables. Thus, the above method corresponds to the construction of pruned (but not minimal) SSA—see Chap. 2.

While resulting in a legal SSA program, the construction clearly introduces more φ-functions than necessary. Each superfluous φ-function corresponds to the situation where all call sites to some function `fᵢ` pass identical arguments. The technique for eliminating such arguments is called λ-dropping and is the inverse of the more widely known transformation λ-lifting.

### 中文翻译

从这个角度看，上述参数列表的构造过程，等价于为每个基本块 `bᵢ` 的每个入口活跃变量分别放置一个 φ 函数，然后再对变量执行重命名。因此，这种方法构造的是**剪枝 SSA（pruned SSA）**，但不是**最小 SSA（minimal SSA）**；关于二者的定义可参见第 2 章。

虽然结果是一个合法的 SSA 程序，但这种构造显然会引入超过实际需要数量的 φ 函数。每一个多余 φ 函数都对应如下情形：对于函数 `fᵢ` 的某一个参数位置，`fᵢ` 的所有调用点传入的实际参数完全相同。消除这类参数的技术称为 **λ-dropping（λ-删除）**；它是函数式编程中更广为人知的 **λ-lifting（λ-提升）**变换的逆过程。

---

## 6.2.2 λ-Dropping（λ-删除）— PDF 第 13–17 页

### English

λ-dropping may be performed before or after variable names are made distinct, but for our purpose, the former option is more instructive. The transformation consists of two phases, block sinking and parameter dropping.

### 中文翻译

λ-删除既可以在把变量名改成互不相同之前进行，也可以在完成这种重命名之后进行。不过，为了说明本章关心的对应关系，在重命名之前进行 λ-删除更有启发性。整个变换由两个阶段组成：

1. **block sinking（块下沉）**；
2. **parameter dropping（参数删除）**。

### 6.2.2.1 Block Sinking（块下沉）

#### English

Block sinking analyses the static call structure to identify which function definitions may be moved inside each other. For example, whenever our set of function declarations contains definitions

```text
f(x₁, ..., xₙ) = e_f
g(y₁, ..., yₘ) = e_g
```

where `f ≠ g` and such that all calls to `f` occur in `e_f` or `e_g`, we can move the declaration for `f` into that of `g`—note the similarity to the notion of dominance. If applied aggressively, block sinking indeed amounts to making the entire dominance tree structure explicit in the program representation. In particular, algorithms for computing the dominator tree from a CFG discussed elsewhere in this book can be applied to identify block sinking opportunities, where the CFG is given by the call graph of functions.

#### 中文翻译

块下沉会分析程序的静态调用结构，以判断哪些函数定义能够移动到另一个函数定义内部。假设当前函数声明集合中包含 `f(x₁, ..., xₙ) = e_f` 和 `g(y₁, ..., yₘ) = e_g`，其中 `f` 与 `g` 不是同一个函数；如果对 `f` 的所有调用都只出现在 `f` 自己的函数体 `e_f` 或 `g` 的函数体 `e_g` 中，就可以把 `f` 的声明移动到 `g` 的声明内部。

这个判定条件与支配关系非常相似：除了递归调用外，所有到达 `f` 的调用路径都要经过 `g`。如果尽可能积极地执行块下沉，最终效果就是把整棵支配树的结构显式编码到程序的函数嵌套结构中。更具体地说，本书其他章节介绍的“从 CFG 计算支配树”的算法可以直接用于识别块下沉机会，只需把函数之间的调用图当作这里的 CFG。

#### English

In our example (6.15), `f₃` is only invoked from within `f₂`, and `f₂` is only called in the bodies of `f₂` and `f₁` (see the dominator tree in Fig. 6.3 (right)). We may thus move the definition of `f₃` into that of `f₂`, and the latter one into `f₁`.

Several options exist as to where `f` should be placed in its host function. The first option is to place `f` at the beginning of `g`, by rewriting to

```text
function g(y₁, ..., yₘ) =
    function f(x₁, ..., xₙ) = e_f
    in e_g end
```

This transformation does not alter the semantics of the code, as the declaration of `f` is closed: moving `f` into the scope of the formal parameters `y₁, ..., yₘ` (and also into the scope of `g` itself) does not alter the bindings to which variable uses inside `e_f` refer. Applying this transformation to example (6.15) yields the following code:

```text
function f₁() =
    function f₂(v, z, y) =
        function f₃(y, v) =
            let w = y + v in w end
        in let x = 5 + y, y = x × z, x = x - 1
           in if x = 0 then f₃(y, v) else f₂(v, z, y) end
        end
    in let v = 1, z = 8, y = 4
       in f₂(v, z, y) end
    end
in f₁() end                                             (6.17)
```

#### 中文翻译

在示例（6.15）中，只有 `f₂` 的函数体会调用 `f₃`；而 `f₂` 只在 `f₁` 的函数体和 `f₂` 自己的函数体中被调用。这与图 6.3 右侧的支配树完全一致。因此，可以先把 `f₃` 的定义移动到 `f₂` 内部，再把已经包含 `f₃` 的 `f₂` 定义移动到 `f₁` 内部。

被移动的 `f` 在宿主函数中可以放在多个位置。第一种选择是把 `f` 放在宿主 `g` 的开头，也就是先声明局部函数 `f`，再执行原来的 `g` 函数体 `e_g`。

这种变换不会改变代码语义，因为 `f` 的声明原本是闭合的。移动之后，`f` 虽然进入了 `g` 的形式参数 `y₁, ..., yₘ` 以及函数名 `g` 本身的作用域，但 `e_f` 内的变量使用原本就不依赖这些新进入的绑定，所以它们所指向的绑定不会改变。把这一变换应用于（6.15），就得到代码（6.17）。

#### English

An alternative strategy is to insert `f` near the end of its host function `g`, in the vicinity of the calls to `f`. This brings the declaration of `f` additionally into the scope of all let-bindings in `e_g`. Again, referential transparency and preservation of semantics are respected as the declaration of `f` is closed. In our case, the alternative strategy yields the following code:

```text
function f₁() =
    let v = 1, z = 8, y = 4
    in function f₂(v, z, y) =
           let x = 5 + y, y = x × z, x = x - 1
           in if x = 0
              then function f₃(y, v) =
                       let w = y + v in w end
                   in f₃(y, v) end
              else f₂(v, z, y)
              end
       in f₂(v, z, y) end
    end
in f₁() end                                             (6.18)
```

In general, one would insert `f` directly prior to its call if `g` contains only a single call site for `f`. In the event that `g` contains multiple call sites for `f`, these are (due to their tail-recursive positioning) in different arms of a conditional, and we would insert `f` directly prior to this conditional.

Both outlined placement strategies result in code whose nesting structure reflects the dominance relationship of the imperative code. In our example, code (6.17) and (6.18) both nest `f₃` inside `f₂` inside `f₁`, in accordance with the dominator tree of the imperative program shown in Fig. 6.3.

#### 中文翻译

另一种策略是把 `f` 插入宿主函数 `g` 的末端，尽可能靠近对 `f` 的调用。与放在开头相比，这种放置会让 `f` 的声明额外进入 `e_g` 中所有 `let` 绑定的作用域。由于 `f` 的声明仍然是闭合的，引用透明性保证这些新增的外部绑定不会改变 `f` 原有自由变量的含义，因此语义仍然保持不变。在贯穿示例中，这种策略得到代码（6.18）。

一般而言，如果 `g` 中只有一个 `f` 调用点，就把 `f` 的声明直接插在该调用之前。如果 `g` 中有多个 `f` 调用点，由于这些调用处于尾递归位置，它们会分布在某个条件语句的不同分支中；此时，应当把 `f` 的声明直接插在这个条件语句之前，使所有分支中的调用都位于其作用域内。

上述两种放置策略产生的代码，其嵌套结构都反映命令式代码中的支配关系。在本例里，（6.17）和（6.18）都把 `f₃` 嵌套在 `f₂` 中，再把 `f₂` 嵌套在 `f₁` 中，与图 6.3 所示命令式程序的支配树一致。

#### 理解与补充

两种放置策略得到的嵌套结构都反映支配关系：`f₃` 嵌套在 `f₂` 内，`f₂` 又嵌套在 `f₁` 内，与图 6.3 的支配树一致。但靠近调用点放置通常更好，因为函数声明能看到更多宿主 `let` 绑定，下一阶段就可能删除更多参数。

### 6.2.2.2 Parameter Dropping（参数删除）

#### English

The second phase of λ-dropping, parameter dropping, removes superfluous parameters based on the syntactic scope structure. Removing a parameter `x` from the declaration of some function `f` has the effect that any use of `x` inside the body of `f` will not be bound by the declaration of `f` any longer, but by the (innermost) binding for `x` that contains the declaration of `f`. In order to ensure that removing the parameter does not alter the meaning of the program, we thus have to ensure that this `f`-containing binding for `x` also contains any call to `f`, since the binding applicable at the call site determines the value that is passed as the actual argument (before `x` is deleted from the parameter list).

For example, the two parameters of `f₃` in (6.18) can be removed without altering the meaning of the code, as we can statically predict the values they will be instantiated with: Parameter `y` will be instantiated with the result of `x × z` (which is bound to `y` in the first line in the body of `f₂`), and `v` will always be bound to the value passed via the parameter `v` of `f₂`. In particular, the bindings for `y` and `v` at the declaration site of `f₃` are identical to those applicable at the call to `f₃`.

#### 中文翻译

λ-删除的第二个阶段称为参数删除。它依据程序的语法作用域结构，移除实际上不需要通过调用传递的形式参数。

如果从函数 `f` 的声明中删除参数 `x`，那么 `f` 函数体中所有对 `x` 的使用都不再由 `f` 的这个形式参数绑定，而会向外查找，改由“包围 `f` 声明的最内层 `x` 绑定”来解释。为了保证删除参数不改变程序含义，必须确认这个包围 `f` 声明的 `x` 绑定也同时包围每一个对 `f` 的调用。原因是：在删除参数以前，调用点处生效的 `x` 绑定决定了作为实际参数传入 `f` 的值；删除后，函数体直接捕获外层 `x`，二者必须是同一个绑定。

例如，（6.18）中 `f₃` 的两个参数都可以删除，而且不会改变代码含义，因为它们在每次调用时会得到什么值可以静态确定：参数 `y` 总会被实例化为 `x × z` 的结果，也就是 `f₂` 函数体第一行重新绑定给 `y` 的值；参数 `v` 总会得到通过 `f₂` 的形式参数 `v` 传进来的值。关键在于，`f₃` 声明位置适用的 `y`、`v` 绑定，与调用 `f₃` 时适用的 `y`、`v` 绑定完全相同。

#### English

In general, we may drop a parameter `x` from the declaration of a possibly recursive function `f` if the following two conditions are met:

1. The tightest scope for `x` enclosing the declaration of `f` coincides with the tightest scope for `x` surrounding every call site to `f` outside its declaration.
2. The tightest scope for `x` enclosing every recursive call to `f` is the one associated with formal parameter `x` in the declaration of `f`.

The rationale for these clauses is that removing `x` from `f`'s parameter list means that any free occurrence of `x` in `f`'s body is now bound outside of `f`'s declaration. Therefore, for each call to `f` to be correct, one needs to ensure that this outside binding coincides with the one containing the call.

Similarly, we may simultaneously drop a parameter `x` occurring in all declarations of a block of mutually recursive functions `f₁, ..., fₙ`, if the scope for `x` at the point of declaration of the block coincides with the tightest scope in force at any call site to some `fᵢ` outside the block, and if in each call to some `fᵢ` inside some `fⱼ`, the tightest scope for `x` is the one associated with the formal parameter `x` of `fⱼ`. In both cases, dropping a parameter means removing it from the list of formal parameter lists of the function declarations concerned, and also from the argument lists of the corresponding function calls.

#### 中文翻译

一般来说，对于一个可能递归的函数 `f`，只有同时满足以下两个条件，才可以从它的声明中删除参数 `x`：

1. 包围 `f` 声明的最内层 `x` 作用域，必须与包围 `f` 声明之外每一个 `f` 调用点的最内层 `x` 作用域完全相同；
2. 对 `f` 的每个递归调用，也就是 `f` 函数体内部对自身的调用，包围该调用的最内层 `x` 作用域必须正是 `f` 的形式参数 `x` 所建立的作用域。

这两个条件背后的理由是：删除 `f` 的参数 `x` 后，`f` 函数体里原先由形式参数绑定的 `x` 会变成自由出现，并改由 `f` 声明之外的绑定来解释。为了保证每个调用仍然正确，这个声明外的绑定必须与包围相应调用点的绑定是同一个绑定。

同样的原则可以推广到一组相互递归的函数 `f₁, ..., fₙ`。如果参数 `x` 出现在这组函数的每个声明中，可以同时从所有声明删除它，但需满足：声明整个递归函数块时生效的 `x` 作用域，必须与函数块外每个 `fᵢ` 调用点处最内层的 `x` 作用域相同；同时，在某个 `fⱼ` 内调用某个 `fᵢ` 时，调用点最内层的 `x` 作用域必须是 `fⱼ` 的形式参数 `x` 建立的作用域。

无论处理单个递归函数还是相互递归函数块，“删除参数”都包含两项同步修改：从有关函数声明的形式参数列表中移除该参数，同时从所有对应函数调用的实际参数列表中移除相同位置的实参。

#### English

In code (6.18), these conditions sanction the removal of both parameters from the non-recursive function `f₃`. The scope applicable for `v` at the site of declaration of `f₃` and also at its call site is the one rooted at the formal parameter `v` of `f₂`. In case of `y`, the common scope is the one rooted at the let-binding for `y` in the body of `f₂`. We thus obtain the following code:

```text
function f₁() =
    let v = 1, z = 8, y = 4
    in function f₂(v, z, y) =
           let x = 5 + y, y = x × z, x = x - 1
           in if x = 0
              then function f₃() =
                       let w = y + v in w end
                   in f₃() end
              else f₂(v, z, y)
              end
       in f₂(v, z, y) end
    end
in f₁() end                                             (6.19)
```

Considering the recursive function `f₂` next we observe that the recursive call is in the scope of the let-binding for `y` in the body of `f₂`, preventing us from removing `y`. In contrast, neither `v` nor `z` has binding occurrences in the body of `f₂`. The scopes applicable at the external call site to `f₂` coincide with those applicable at its site of declaration and are given by the scopes rooted in the let-bindings for `v` and `z`. Thus, parameters `v` and `z` may be removed from `f₂`:

```text
function f₁() =
    let v = 1, z = 8, y = 4
    in function f₂(y) =
           let x = 5 + y, y = x × z, x = x - 1
           in if x = 0
              then function f₃() =
                       let w = y + v in w end
                   in f₃() end
              else f₂(y)
              end
       in f₂(y) end
    end
in f₁() end                                             (6.20)
```

Interpreting the uniquely renamed variant of (6.20) back in SSA yields the desired code with a single φ-function, for variable `y` at the beginning of block `b₂`, see Fig. 6.4. The reason that this φ-function cannot be eliminated (the redefinition of `y` in the loop) is precisely the reason why `y` survives parameter dropping.

Given this understanding of parameter dropping we can also see why inserting functions near the end of their hosts during block sinking (as in code (6.18)) is in general preferable to inserting them at the beginning of their hosts (as in code (6.17)): The placement of function declarations in the vicinity of their calls potentially enables the dropping of more parameters, namely those that are let-bound in the body of the host function.

#### 中文翻译

在代码（6.18）中，上述条件允许从非递归函数 `f₃` 删除两个参数。对于 `v`，`f₃` 声明处和调用处适用的作用域，都是由 `f₂` 的形式参数 `v` 建立的作用域；对于 `y`，两个位置共同适用的作用域，则是由 `f₂` 函数体中 `y` 的 `let` 绑定建立的作用域。因此，删除两个参数后得到代码（6.19）。

接着考察递归函数 `f₂`。对 `f₂` 的递归调用位于 `f₂` 函数体中那个新 `y` 的 `let` 绑定作用域内，因此调用点的 `y` 与声明 `f₂` 时外部的 `y` 不是同一个绑定；这使我们不能删除参数 `y`。与之相反，`f₂` 的函数体中没有对 `v` 或 `z` 的新绑定。`f₂` 的外部调用点和 `f₂` 的声明位置所适用的 `v`、`z` 作用域完全相同，分别来自外层关于 `v`、`z` 的 `let` 绑定。因此，可以从 `f₂` 中删除参数 `v` 和 `z`，得到代码（6.20）。

### 图 6.4 解读：λ-删除后的 SSA

把（6.20）做全局唯一重命名并重新解释成 SSA 后，只剩循环头 `b₂` 中关于 `y` 的一个 φ 函数：

```text
b₁:
    v₁ ← 1
    z₁ ← 8
    y₁ ← 4

b₂:
    y₂ ← φ(y₁, y₃)
    x₁ ← 5 + y₂
    y₃ ← x₁ × z₁
    x₂ ← x₁ - 1
    if x₂ = 0 then b₃ else b₂

b₃:
    w₁ ← v₁ + y₃
    return w₁
```

把（6.20）中的变量全部唯一重命名，再重新解释成 SSA，就会得到所需的结果：在基本块 `b₂` 的开头只保留一个关于变量 `y` 的 φ 函数，如图 6.4 所示。这个 φ 函数不能消除的原因，是循环体中会重新定义 `y`；而这恰好也是 `y` 无法通过参数删除消去的原因。换言之，“参数必须保留”和“φ 函数必须保留”是同一个结构事实的两种表达。

理解参数删除后，也就能看出为什么在块下沉阶段，把函数插在宿主末端、靠近调用点的位置——如（6.18）——通常优于把它插在宿主开头——如（6.17）。靠近调用点放置函数声明，会让它进入宿主函数体中更多 `let` 绑定的作用域；这些由宿主函数体建立的绑定随后可能直接被嵌套函数捕获，从而允许删除更多形式参数。

### English

An immediate consequence of this strategy is that blocks with a single direct predecessor indeed do not contain φ-functions. Such a block `b_f` is necessarily dominated by its direct predecessor `b_g`, hence we can always nest `f` inside `g`. Inserting `f` in `e_g` directly prior to its call site implies that condition (1) is necessarily satisfied for all parameters `x` of `f`. Thus, all parameters of `f` can be dropped and no φ-function is generated.

### 中文翻译

这一策略有一个直接结论：只有一个直接前驱的基本块确实不应包含 φ 函数。设这样的基本块为 `b_f`，它必然被自己的直接前驱 `b_g` 支配，所以总可以把对应函数 `f` 嵌套到函数 `g` 内。再把 `f` 插入 `g` 的函数体 `e_g`，并放在唯一调用点之前，那么对于 `f` 的每个参数 `x`，参数删除条件（1）都会自动成立：`f` 的声明和调用处于同一个最内层 `x` 作用域。因此可以删除 `f` 的全部参数，也就不会生成任何 φ 函数。

---

## 6.2.3 Nesting, Dominance, Loop-Closure（嵌套、支配与循环闭合）— PDF 第 17–20 页

### English

As we observed above, analysing whether function definitions may be nested inside one another is tantamount to analysing the imperative dominance structure: function `fᵢ` may be moved inside `fⱼ` exactly if all non-recursive calls to `fᵢ` come from within `fⱼ`, i.e., exactly if all paths from the initial program point to block `bᵢ` traverse `bⱼ`, i.e., exactly if `bⱼ` dominates `bᵢ`. This observation is merely the extension to function identifiers of our earlier remark that lexical scope coincides with the dominance region, and that points of definition/binding occurrences should dominate the uses.

Indeed, functional languages do not distinguish between code and data when aspects of binding and use of variables are concerned, as witnessed by our use of the let-binding construct for binding code-representing expressions to the variables `k` in our syntax for CPS.

Thus, the dominator tree immediately suggests a function nesting scheme, where all children of a node are represented as a single block of mutually recursive function declarations.²

### 中文翻译

如前面所见，分析函数定义能否互相嵌套，等同于分析命令式程序中的支配结构。函数 `fᵢ` 能够移动到 `fⱼ` 内部，当且仅当对 `fᵢ` 的所有非递归调用都来自 `fⱼ` 的函数体；这又恰好等价于从初始程序点到基本块 `bᵢ` 的每条路径都经过 `bⱼ`，也就是 `bⱼ` 支配 `bᵢ`。

这个观察只是把前面关于普通变量的结论扩展到了函数标识符：词法作用域与支配区域相对应，变量或函数的定义点、绑定出现应当支配它们的使用。

事实上，就名字的绑定与使用而言，函数式语言并不区分代码和数据。前面的 CPS 语法已经展示了这一点：表示后续代码的续延表达式，同样可以使用 `let` 绑定到变量 `k` 上，再像普通值一样传递和使用。

因此，支配树立即给出一种函数嵌套方案：支配树中同一个节点的全部孩子，可以表示成单个相互递归函数声明块。²

² 原文脚注：6.3 节将概述这种表示方式的进一步改进。

### English

The choice as to where functions are placed corresponds to variants of SSA. For example, in loop-closed SSA form (see Chaps. 14 and 10), SSA names that are defined in a loop must not be used outside the loop. To this end, special-purpose unary φ-nodes are inserted for these variables at the loop exit points. As the loop is unrolled, the arity of these trivial φ-nodes grows with the number of unrollings, and the program continuation is always supplied with the value the variable obtained in the final iteration of the loop.

In our example, the only loop-defined variable used in `f₃` is `y`—and we already observed in code (6.17) how we can prevent the dropping of `y` from the parameter list of `f₃`: we insert `f₃` at the beginning of `f₂`, preceding the let-binding for `y`. Of course, we would still like to drop as many parameters from `f₂` as possible, hence we apply the following placement policy during block sinking:

Functions that are targets of loop-exiting function calls and have live-in variables that are defined in the loop are placed at the beginning of the loop headers. Other functions are placed at the end of their hosts. Applying this policy to our original program (6.15) yields (6.21).

### 中文翻译

函数声明选择放在宿主的什么位置，对应不同的 SSA 变体。例如，在**循环闭合 SSA（loop-closed SSA）**中——参见第 14 章和第 10 章——凡是在循环内部定义的 SSA 名字，都不允许直接在循环外部使用。为了满足这个约束，需要在循环的各个出口点为这些变量插入专门的一元 φ 节点，由这个新定义把循环最终产生的值传递到循环外。

当循环被展开时，这些最初只有一个输入、因而看似平凡的 φ 节点，会随着展开次数增加而提高元数。无论实际从哪一次展开出来的迭代退出，程序的后续部分都会通过该 φ 节点得到变量在最后一次实际执行迭代中取得的值。

在贯穿示例中，`f₃` 使用的变量里，只有 `y` 是在循环内部定义的。代码（6.17）已经展示了如何阻止参数删除阶段从 `f₃` 的参数列表中删掉 `y`：把 `f₃` 的声明插在 `f₂` 的开头，放在循环体中重新绑定 `y` 的 `let` 之前。这样，`f₃` 声明处看到的是循环入口的 `y`，而调用处看到的是循环迭代后产生的 `y`，二者作用域不再相同，因此 `y` 必须继续作为参数传递。

与此同时，我们仍希望尽量多地删除 `f₂` 的参数。因此，块下沉采用以下组合放置策略：

- 如果某个函数是“退出循环的函数调用”的目标，并且它的入口活跃变量中存在循环内部定义的变量，就把这个函数放在循环头函数的开头；
- 其他函数仍放在各自宿主函数的末端。

把这项策略应用于原始程序（6.15），得到代码（6.21）。

### English

```text
function f₁() =
    let v = 1, z = 8, y = 4
    in function f₂(v, z, y) =
           function f₃(y, v) =
               let w = y + v in w end
           in let x = 5 + y, y = x × z, x = x - 1
              in if x = 0 then f₃(y, v) else f₂(v, z, y) end
           end
       in f₂(v, z, y) end
    end
in f₁() end                                             (6.21)
```

We may now drop `v` (but not `y`) from the parameter list of `f₃`, and `v` and `z` from `f₂`, to obtain code (6.22).

```text
function f₁() =
    let v = 1, z = 8, y = 4
    in function f₂(y) =
           function f₃(y) =
               let w = y + v in w end
           in let x = 5 + y, y = x × z, x = x - 1
              in if x = 0 then f₃(y) else f₂(y) end
           end
       in f₂(y) end
    end
in f₁() end                                             (6.22)
```

### 中文翻译

在（6.21）中，现在可以从 `f₃` 的参数列表删除 `v`，因为声明处与调用处看到的是同一个外层 `v`；但不能删除 `y`，因为函数声明位于循环头开头，而调用位于循环体重新定义 `y` 之后。对于 `f₂`，仍然可以删除 `v` 和 `z`。这些删除最终得到代码（6.22）。

### English

The SSA form corresponding to (6.22) contains the desired loop-closing φ-node for `y` at the beginning of `b₃`, as shown in Fig. 6.5a. The nesting structure of both (6.21) and (6.22) coincides with the dominance structure of the original imperative code and its loop-closed SSA form.

### 中文翻译

与（6.22）对应的 SSA 形式，在 `b₃` 开头包含所需的 `y` 循环闭合 φ 节点，如图 6.5a 所示：

```text
b₃:
    y₃ ← φ(y₄)
    w₁ ← v₁ + y₃
    return w₁
```

代码（6.21）和（6.22）的函数嵌套结构，都与原命令式代码以及它的循环闭合 SSA 形式中的支配结构一致。

### 图 6.5 与循环展开

### English

We unroll the loop by duplicating the body of `f₂`, without duplicating the declaration of `f₃`:

### 中文翻译

图 6.5a 展示循环闭合形式，图 6.5b 展示把循环展开一次后的形式。函数式表示通过复制 `f₂` 的函数体完成循环展开，但刻意不复制 `f₃` 的声明：

```text
function f₁() =
    let v = 1, z = 8, y = 4
    in function f₂(y) =
           function f₃(y) =
               let w = y + v in w end
           in let x = 5 + y, y = x × z, x = x - 1
              in if x = 0 then f₃(y)
                 else function f₂′(y) =
                          let x = 5 + y, y = x × z, x = x - 1
                          in if x = 0 then f₃(y) else f₂(y) end
                      in f₂′(y) end
                 end
       in f₂(y) end
    end
in f₁() end                                             (6.23)
```

### English

Both calls to `f₃` are in the scope of the declaration of `f₃` and contain the appropriate loop-closing arguments. In the SSA reading of this code—shown in Fig. 6.5b—the first instruction in `b₃` has turned into a non-trivial φ-node. As expected, the parameters of this φ-node correspond to the two control-flow arcs leading into `b₃`, one for each call site to `f₃` in code (6.23). Moreover, the call and nesting structure of (6.23) is indeed in agreement with the control flow and dominance structure of the loop-unrolled SSA representation.

### 中文翻译

代码（6.23）中两个对 `f₃` 的调用都位于 `f₃` 声明的作用域内，并分别携带正确的循环闭合实参。按照 SSA 方式读取这段代码时——结果如图 6.5b——`b₃` 中的第一条指令已经从平凡的一元 φ 节点变成非平凡的二元 φ 节点。

正如预期，这个 φ 节点的两个输入参数对应两条进入 `b₃` 的控制流边；在函数式表示中，它们又分别对应代码（6.23）里 `f₃` 的两个调用点。此外，（6.23）的调用结构与函数嵌套结构，也确实分别符合循环展开后 SSA 表示的控制流结构和支配结构。


---

## 6.2.4 Destruction of SSA（SSA 消除）— PDF 第 20 页

### English

The above example code excerpts where variables are not made distinct exhibit a further pattern: The argument list of any call coincides with the list of formal parameters of the invoked function. This discipline is not enjoyed by functional programs in general, and is often destroyed by optimizing program transformations. However, programs that do obey this discipline can be immediately converted to imperative non-SSA form.

Thus, the task of SSA destruction amounts to converting a functional program with arbitrary argument lists into one where argument lists and formal parameter lists coincide for each function. This can be achieved by introducing additional let-bindings of the form `let x = y in e end`.

For example, a call `f(v, z, y)` where `f` is declared as `function f(x, y, z) = e` may be converted to

```text
let x = v,
    a = z,
    z = y,
    y = a
in f(x, y, z) end
```

in correspondence to the move instructions introduced in imperative formulations of SSA destruction (see Chaps. 3 and 21).

### 中文翻译

在前面的示例代码中，变量名还没有改成全局互不相同时，可以观察到另一个规律：任意函数调用的实际参数列表，都与被调用函数声明中的形式参数列表一致。这里的“一致”不仅是数量相同，还表示经过当前作用域解释后，调用形如 `f(x, y, z)`，与声明使用相同名字和顺序。

一般的函数式程序并不遵守这项规范，各种优化变换也经常会破坏它。不过，如果某个程序确实满足这项规范，就可以立刻把它转换成命令式的非 SSA 形式。

因此，SSA 消除任务可以重新表述为：把一个函数调用可以使用任意实参列表的函数式程序，转换成这样一种程序——对每个函数而言，所有调用的实参列表都与其形式参数列表一致。实现这种规范化的方法，是在调用前增加 `let x = y in e end` 形式的绑定；这些绑定在命令式解释中就是复制指令。

例如，函数声明为 `function f(x, y, z) = e`，但某个调用是 `f(v, z, y)`。为了把调用整理成 `f(x, y, z)`，先把 `v` 复制给 `x`。原来的 `z` 要传给形式参数 `y`，而原来的 `y` 又要传给形式参数 `z`，二者形成交换环，所以必须先用临时变量 `a` 保存原来的 `z`，再顺序完成 `z = y` 和 `y = a`。这些 `let` 绑定正对应命令式 SSA 消除方案插入的 move 指令；相关内容见第 3 章和第 21 章。

### English

Appropriate transformations can be formulated as manipulations of the functional representation, although the target format is not immune to α-renaming and thus only syntactically a functional language. For example, we can give a local algorithm that considers each call site individually and avoids the “lost-copy” and “swap” problems (cf. Chap. 21): Instead of introducing let-bindings for all parameter positions of a call, the algorithm scales with the number and size of cycles that span identically named arguments and parameters (like the cycle between `y` and `z` above), and employs a single additional variable (called `a` in the above code) to break all these cycles one by one.

### 中文翻译

这些适当的转换都可以表述成对函数式表示本身的操作。不过，目标格式已经不再“免疫于”α-重命名：随意重命名绑定变量可能破坏调用实参与形式参数必须同名的规范。因此，它只在语法外形上还是函数式语言，而不再具有普通函数式语言完整的 α-等价性质。

例如，可以设计一个局部算法，分别处理每一个调用点，并避免第 21 章讨论的 **lost-copy（复制丢失）**和 **swap（交换）**问题。这个算法不需要无条件地为调用的每个参数位置都插入一个 `let` 复制；它的工作量只随“同名实参与形式参数之间形成的置换环”的数量和大小增长。上例中 `y` 与 `z` 就形成一个环。算法只需要一个额外变量——上例命名为 `a`——就能依次打破所有这样的环，而不必为每个环永久保留不同的临时变量。

### 6.2 小结

函数式视角把 SSA 的完整生命周期统一成几个普通变换：

```text
活跃性分析                  -> 为基本块函数选择参数
全局唯一 α-重命名           -> 得到合法 SSA 名字
块下沉                      -> 让函数嵌套反映支配树
参数删除                    -> 删除多余 φ 函数
循环出口函数的特殊放置      -> 构造 loop-closed SSA
统一调用实参与形式参数名字  -> 消除 SSA
```

---

## 6.3 Refined Block Sinking and Loop Nesting Forests（精细块下沉与循环嵌套森林）— PDF 第 20–24 页

### English

As discussed when outlining λ-dropping, block sinking is governed by the dominance relation between basic blocks. Thus, a typical dominance tree with root `b` and subtrees rooted at `b₁, ..., bₙ` is most naturally represented as a block of function declarations for the `fᵢ`, nested inside the declaration of `f`:

```text
function f(...) =
    let ... <body of b> ...
    in function f₁(...) = e₁    <body of b₁, with calls to b, bᵢ>
       ...
       and fₙ(...) = eₙ         <body of bₙ, with calls to b, bᵢ>
       in ... <calls to b, bᵢ from b> ... end
                                                     (6.24)
```

By exploiting additional control-flow structure between the `bᵢ`, it is possible to obtain refined placements, namely placements that correspond to notions of loop nesting forests that have been identified in the SSA literature.

### 中文翻译

如 λ-删除部分所述，块下沉由基本块之间的支配关系控制。因此，如果一棵典型支配树以 `b` 为根，并有以 `b₁, ..., bₙ` 为根的子树，那么最自然的函数式表示就是：把各 `fᵢ` 的声明组成一个函数块，并嵌套在 `f` 的声明内部，如代码（6.24）。

如果进一步利用各个 `bᵢ` 之间的控制流结构，就可以得到更精细的函数放置方案。具体而言，这些方案对应 SSA 研究文献中已经提出的各种**循环嵌套森林（loop nesting forest）**概念。也就是说，目标不再只是让函数嵌套反映支配树，还要让函数名的可见性和递归分组反映支配树兄弟节点之间的真实控制流。

---

### 可约 CFG 中的精细放置

### English

These refinements arise if we enrich the above dominance tree by adding arrows `bᵢ → bⱼ` whenever the CFG contains a directed edge from one of the dominance successors of `bᵢ` (i.e., the descendants of `bᵢ` in the dominance tree) to `bⱼ`.

In the case of a reducible CFG, the resulting graph contains only trivial loops. Ignoring these self-loops, we perform a post-order DFS (or more generally a reverse topological ordering) among the `bᵢ` and stagger the function declarations according to the resulting order. As an example, consider the CFG in Fig. 6.6a and its enriched dominance tree shown in Fig. 6.6b. A possible (but not unique) ordering of the children of `b` is `[b₅, b₁, b₃, b₂, b₄]`, resulting in the nesting shown in code (6.25).

```text
function f(...) =
    let ... <body of f> ...
    in function f₅(...) = e₅
       in function f₁(...) = e₁       <calls f and f₅>
          in function f₃(...) = e₃
             in function f₂(...) =
                    let ... <body of f₂> ...
                    in function g(...) = e_g   <calls f₃>
                       in ... <calls f₅ and g> ... end
                in function f₄(...) = e₄       <calls f₃ and f₄>
                   in ... <calls f₁, f₂, f₄, f₅> ... end
                                                     (6.25)
```

### 中文翻译

这种精细化首先要增强前面的支配树。对于任意两个支配树节点 `bᵢ` 和 `bⱼ`，如果原 CFG 中存在一条有向边，它从 `bᵢ` 的某个支配后继指向 `bⱼ`，就在增强支配树中添加一条箭头 `bᵢ → bⱼ`。这里 `bᵢ` 的支配后继，就是支配树中 `bᵢ` 的后代节点。注意，箭头的起点记为整棵以 `bᵢ` 为根的支配子树，而不一定是 CFG 边的直接源基本块。

如果 CFG 是**可约的（reducible）**，所得增强图只会包含平凡的循环，也就是自环。忽略这些自环后，在各 `bᵢ` 之间执行后序深度优先遍历；更一般地说，可以计算一个逆拓扑顺序。然后按照这个顺序把各函数声明交错、嵌套地排列，使某个函数只在确实可能需要调用它的后续范围内可见。

以图 6.6a 的 CFG 和图 6.6b 的增强支配树为例，根节点 `b` 的孩子可以按 `[b₅, b₁, b₃, b₂, b₄]` 排列。这个顺序只是可行顺序之一，并不唯一；按照它交错函数声明，就得到代码（6.25）所示的嵌套结构。

### 图 6.6 解读

图 6.6a 是 CFG；图 6.6b 把 CFG 中跨越支配子树的边以虚线叠加到实线支配树上。

代码（6.25）仍像朴素方案一样保持支配关系，但进一步限制了名字可见性：

- `f₁` 在 `e₅` 中不可见；
- `f₃` 在 `f₁` 或 `f₅` 中不可见；
- 只有确实可能沿控制流到达相应块的代码，才能看到这些局部函数名。

### English

The code respects the dominance relationship in much the same way as the naive placement, but additionally makes `f₁` inaccessible from within `e₅`, and makes `f₃` inaccessible from within `f₁` or `f₅`. As the reordering does not move function declarations inside each other (in particular: no function declaration is brought into or moved out of the scope of the formal parameters of any other function) the reordering does not affect the potential to subsequently perform parameter dropping.

### 中文翻译

这段代码与朴素放置方案基本以同样方式保持支配关系，但它还提供了额外的可见性约束：在 `e₅` 内无法访问 `f₁`，在 `f₁` 或 `f₅` 内也无法访问 `f₃`。这些限制更准确地反映实际控制流中“不可能从这里跳转到那里”的关系。

这种重新排序并没有把某个函数声明真正移动到另一个函数声明的内部或外部。特别是，没有任何函数声明被带入或移出另一个函数形式参数的作用域。因此，变量在函数声明处可见的绑定集合没有变化，后续能够执行哪些参数删除也不会受到影响。

---

### 使用 λ 抽象区分递归与非递归结构

### English

Declaring functions using λ-abstraction brings further improvements. This enables us not only to syntactically distinguish between loops and non-recursive control-flow structures using the distinction between `let` and `letrec` present in many functional languages, but also to further restrict the visibility of function names.

Indeed, while `b₃` is immediately dominated by `b` in the above example, its only control-flow predecessors are `b₂/g` and `b₄`. We would hence like to make the declaration of `f₃` local to the tuple `(f₂, f₄)`, i.e., invisible to `f`. This can be achieved by combining let/letrec bindings with pattern matching, if we insert the shared declaration of `f₃` between the declaration of the names `f₂` and `f₄` and the λ-bindings of their formal parameters `pᵢ`:

```text
letrec f = λp.
    let ... <body of f> ...
    in let f₅ = λp₅.e₅
       in let f₁ = λp₁.e₁                 <calls f and f₅>
          in letrec (f₂, f₄) =
                 let f₃ = λp₃.e₃
                 in (
                    λp₂. let ... <body of f₂> ...
                         in let g = λp_g.e_g            <calls f₃>
                            in ... <calls f₅ and g> ... end,
                    λp₄.e₄                              <calls f₃ and f₄>
                    )
                 end
             in ... <calls f₁, f₂, f₄, f₅> ... end
                                                     (6.26)
```

### 中文翻译

改用 λ 抽象来声明函数，还能进一步改善表示。许多函数式语言区分 `let` 和 `letrec`：前者建立非递归绑定，后者建立递归或相互递归绑定。利用这种语法区别，不仅可以直接区分循环控制流与非递归控制流，还可以继续缩小函数名的可见范围。

在前面的例子中，虽然 `b₃` 的直接支配者是根节点 `b`，但 `b₃` 实际只有两个控制流前驱：`b₂` 内部的块 `g`，以及 `b₄`。因此，更准确的作用域设计是让 `f₃` 的声明只属于函数元组 `(f₂, f₄)`，而不是让外层函数 `f` 也能访问 `f₃`。

代码（6.26）通过组合 `let`、`letrec` 与模式匹配实现这一点。关键做法是：先用 `letrec (f₂, f₄)` 声明递归函数元组的两个名字，再在这个元组的右侧内部声明二者共享的 `f₃`，最后才用 λ 抽象分别绑定 `f₂`、`f₄` 的形式参数 `p₂`、`p₄`。于是 `f₃` 对两个函数体都可见，却不会泄漏到外层 `f`。

### English

The recursiveness of `f₄` is inherited by the function pair `(f₂, f₄)` but `f₃` remains non-recursive. In general, the role of `f₃` is played by any merge point `bᵢ` that is not directly called from the dominator node `b`.

### 中文翻译

原 CFG 中 `f₄` 的递归性现在由递归函数对 `(f₂, f₄)` 整体继承；共享的 `f₃` 自身仍是普通的非递归函数。一般来说，凡是某个汇合点 `bᵢ` 虽然被支配节点 `b` 支配、却没有从 `b` 直接调用，都可以扮演这里 `f₃` 的角色：把它声明在真正需要共享它的递归函数元组内部。

---

### 不可约 CFG 与 Steensgaard 循环

### English

In the case of irreducible CFGs, the enriched dominance tree is no longer acyclic (even when ignoring self-loops). In this case, the functional representation not only depends on the chosen DFS order but additionally on the partitioning of the enriched graph into loops. As each loop forms a strongly connected component (SCC), different partitionings are possible, corresponding to different notions of loop nesting forests. Of the various loop nesting forest strategies proposed in the SSA literature [236], the scheme introduced by Steensgaard [271] is particularly appealing from a functional perspective.

In Steensgaard's notion of loops, the headers `H` of a loop `L = (B, H)` are precisely the entry nodes of its body `B`, i.e., those nodes in `B` that have a direct predecessor outside of `B`. For example, `G₀` shown in Fig. 6.7 contains the outer loop `L₀ = ({u, v, w, x}, {u, v})`, whose constituents `B₀` are determined as the maximal SCC of `G₀`. Removing the back edges of `L₀` from `G₀` (i.e., edges from `B₀` to `H₀`) yields `G₁`, whose (only) SCC determines a further inner loop `L₁ = ({w, x}, {w, x})`. Removing the back edges of `L₁` from `G₁` results in the acyclic `G₂`, terminating the process.

### 中文翻译

对于**不可约 CFG（irreducible CFG）**，即使忽略所有自环，增强后的支配树仍不再是无环图。此时，函数式表示不仅取决于所选择的 DFS 遍历顺序，还额外取决于如何把增强图划分成若干循环。

因为每个循环都形成一个**强连通分量（strongly connected component，SCC）**，增强图可能存在多种循环划分方式；不同划分对应 SSA 文献中不同的循环嵌套森林概念。在已有的多种策略 [236] 中，Steensgaard [271] 提出的方案从函数式表示角度看尤其有吸引力。

在 Steensgaard 的定义中，一个循环写成 `L = (B, H)`：

- `B` 是循环体节点集合；
- `H` 是循环头集合；
- `H` 精确等于 `B` 的入口节点，即在 `B` 外有直接前驱的那些 `B` 中节点。

### 图 6.7：逐层删除回边

图 6.7 的原始图 `G₀` 包含节点 `entry, u, v, w, x, exit`。

外层循环为：

```text
L₀ = ({u, v, w, x}, {u, v})
```

其循环体 `B₀ = {u, v, w, x}` 是 `G₀` 的最大 SCC，循环头为 `H₀ = {u, v}`。删除从 `B₀` 指回 `H₀` 的回边后得到 `G₁`。

`G₁` 中唯一的 SCC 又确定内层循环：

```text
L₁ = ({w, x}, {w, x})
```

再从 `G₁` 删除 `L₁` 的回边，得到无环的 `G₂`，构造过程终止。

### English

Figure 6.8a shows the CFG-enriched dominance tree of `G₀`. The body of loop `L₀` is easily identified as the maximal SCC, and likewise the body of `L₁` once the cycles `(u, w)` and `(x, v)` are broken by the removal of the back edges `w → u` and `x → v`. The loop nesting forest resulting from Steensgaard's construction is shown in Fig. 6.8b. Loops are drawn as ellipses decorated with the appropriate header nodes and nested in accordance with the containment relation `B₁ ⊂ B₀` between the bodies.

### 中文翻译

图 6.8a 展示 `G₀` 的 CFG 增强支配树。外层循环 `L₀` 的循环体可以直接识别为其中的最大 SCC。删除回边 `w → u` 和 `x → v`，分别打破 `(u, w)` 与 `(x, v)` 两个环后，剩余图中的最大 SCC 同样直接给出内层循环 `L₁` 的循环体。

图 6.8b 展示 Steensgaard 构造最终得到的循环嵌套森林。每个循环画成一个椭圆，并在椭圆上标出相应的循环头节点。循环之间按照循环体的包含关系嵌套：因为 `B₁ ⊂ B₀`，表示 `L₁` 的椭圆嵌套在表示 `L₀` 的椭圆内部。

### English

In the functional representation, a loop `L = (B, {h₁, ..., hₙ})` yields a function declaration block for functions `h₁, ..., hₙ`, with private declarations for the non-headers from `B \ H`. In our example, loop `L₀` provides entry points for the headers `u` and `v` but not for its non-headers `w` and `x`. Instead, the loop comprised of the latter nodes, `L₁`, is nested inside the definition of `L₀`, in accordance with the loop nesting forest.

```text
function entry(...) =
    let ... <body of entry> ...
    in letrec (u, v) =                         <outer loop L₀>
           letrec (w, x) =                    <inner loop L₁>
               let exit = λp_exit. ... <body of exit>
               in (
                  λp_w. ... <body of w; calls u, x, exit> ...,
                  λp_x. ... <body of x; calls w, v, exit> ...
                  )
               end
           in (
              λp_u. ... <body of u; calls w> ...,
              λp_v. ... <body of v; calls x> ...
              )
           end
       in ... <calls from entry to u and v> ...
                                                     (6.27)
```

### 中文翻译

在函数式表示中，一个循环 `L = (B, {h₁, ..., hₙ})` 会产生一个函数声明块，对外提供循环头函数 `h₁, ..., hₙ`。循环体中不属于循环头的节点，即 `B \ H`，则在这个函数声明块内部以私有函数方式声明，外部代码不能把它们当作循环入口调用。

在本例中，外层循环 `L₀` 对外提供循环头 `u`、`v` 两个入口，却不为非循环头节点 `w`、`x` 提供外部入口。后两个节点本身组成内层循环 `L₁`，所以按照循环嵌套森林，把 `L₁` 的整个定义嵌套在 `L₀` 内部。进一步按代码（6.26）的方案放置 `L₁`，并让退出函数 `exit` 成为 `L₁` 的私有函数，就得到代码（6.27）。

### English

By placing `L₁` inside `L₀` according to the scheme from code (6.26) and making `exit` private to `L₁`, we obtain the representation (6.27), which captures all the essential information of Steensgaard's construction. Effectively, the functional reading of the loop nesting forest extends the earlier correspondence between the nesting of individual functions and the dominance relationship to groups of functions and basic blocks: loop `L₀` dominates `L₁` in the sense that any path from `entry` to a node in `L₁` passes through `L₀`; more specifically, any path from `entry` to a header of `L₁` passes through a header of `L₀`.

In general, each step of Steensgaard's construction may identify several loops, as a CFG may contain several maximal SCCs. As the bodies of these SCCs are necessarily non-overlapping, the construction yields a forest comprised of trees shaped like the loop nesting forest in Fig. 6.8b. As the relationship between the trees is necessarily acyclic, the declarations of the function declaration tuples corresponding to the trees can be placed according to the loop-extended notion of dominance.

### 中文翻译

按照代码（6.26）的方案把 `L₁` 放到 `L₀` 内部，并让 `exit` 成为 `L₁` 的私有函数后，得到的表示（6.27）保存了 Steensgaard 构造的全部关键信息。函数式解释把此前“单个函数的嵌套对应单个基本块之间的支配关系”推广到了函数组和基本块组：

- 每条从 `entry` 到 `L₁` 中节点的路径都经过 `L₀`，所以 `L₀` 支配 `L₁`；
- 更具体地说，每条从 `entry` 到 `L₁` 某个循环头的路径，都经过 `L₀` 的某个循环头。

一般情况下，Steensgaard 构造的每一步都可能同时识别出多个循环，因为一个 CFG 可以含有多个最大 SCC。不同最大 SCC 的节点集合必然互不重叠，所以递归构造最终得到的不是单棵循环嵌套树，而是一片由若干这类树组成的森林。不同树之间的关系必然无环，因此可以按照扩展到循环层面的支配关系，为每棵树放置对应的递归函数声明元组。

### 6.3 小结

6.3 节把函数嵌套从“表示一棵支配树”推进为“表示控制流可见性和循环层次”：

```text
支配树父子关系        -> 函数声明的基本嵌套
兄弟子树间控制流      -> 更严格的函数名可见范围
非递归控制流          -> let
循环／递归控制流      -> letrec
多入口不可约循环      -> 一组循环头函数的递归元组
循环体包含关系        -> 函数元组的嵌套关系
```

---

## 6.4 Further Reading and Concluding Remarks（扩展阅读与总结）— PDF 第 24–26 页

## Further Reading（扩展阅读）

### SSA 与函数式表示的早期研究

### English

Shortly after the introduction of SSA, O'Donnell [213] and Kelsey [160] noted the correspondence between let-bindings and points of variable declaration and its extension to other aspects of program structure using continuation-passing style. Appel [10, 11] popularized the correspondence using a direct-style representation, building on his earlier experience with continuation-based compilation [9].

### 中文翻译

SSA 提出后不久，O'Donnell [213] 和 Kelsey [160] 就指出：函数式语言中的 `let` 绑定对应程序中的变量声明点；而且借助续延传递风格，这种对应还可以扩展到程序结构的其他方面。Appel [10, 11] 随后使用直接风格表示普及了这种观点；这项工作建立在他更早的续延式编译经验 [9] 之上。

### CPS、直接风格与低层函数式语言

### English

Continuations and low-level functional languages have been an object of intensive study since their inception about four decades ago [174, 294]. For retrospective accounts of the historical development, see Reynolds [246] and Wadsworth [299]. Early studies of CPS and direct style include work by Reynolds and Plotkin [229, 244, 245]. Two prominent examples of CPS-based compilers are those by Sussman et al. [279] and Appel [9]. An active area of research concerns the relative merit of the various functional representations, algorithms for formal conversion between these formats, and their efficient compilation to machine code, in particular with respect to their integration with program analyses and optimizing transformations [92, 161, 243].

A particularly appealing variant is that of Kennedy [161], where the approach to explicitly name control-flow points such as merge points is taken to its logical conclusion. By mandating all control-flow points to be explicitly named, a uniform representation is obtained that allows optimizing transformations to be implemented efficiently, avoiding the administrative overhead to which some of the alternative approaches are susceptible.

### 中文翻译

续延与低层函数式语言从大约四十年前诞生起，就一直是密集研究的对象 [174, 294]。关于其历史发展的回顾性叙述，可以参见 Reynolds [246] 和 Wadsworth [299]。Reynolds 与 Plotkin 的工作 [229, 244, 245] 属于 CPS 和直接风格的早期研究；Sussman 等人 [279] 与 Appel [9] 的编译器，则是两个著名的 CPS 编译器实例。

目前仍然活跃的研究领域包括：比较各种函数式表示各自的相对优势；研究在这些格式之间进行形式化转换的算法；以及把这些表示高效编译成机器代码。尤其重要的问题是，如何把表示形式与程序分析、优化变换结合起来，而不引入过高的维护成本 [92, 161, 243]。

Kennedy [161] 提出的变体尤其有吸引力。它把“显式命名汇合点等控制流点”的方法贯彻到底，强制所有控制流点都拥有显式名字。这样得到一种统一的中间表示，优化变换可以在其上高效实现，同时避免其他一些方案容易产生的管理性、维护性开销。

### ANF、B-form 与 SIL

### English

Occasionally, the term direct style refers to the combination of tail-recursive functions and let-normal form, and the conditions on the latter notion are strengthened so that only variables may appear as branch conditions. Variations of this discipline include administrative normal form (A-normal form, ANF) [121], B-form [280], and SIL [285].

### 中文翻译

在一些文献中，“直接风格”这个术语特指尾递归函数与 let 范式的组合。同时，let 范式的条件还会被进一步加强，要求分支条件位置只能出现变量，不能直接出现任意复杂表达式。这种编程规范的变体包括：

- **administrative normal form（管理范式）**，又称 **A-normal form（ANF）** [121]；
- **B-form** [280]；
- **SIL** [285]。

### 单子式中间语言

### English

Closely related to continuations and direct-style representation are monadic intermediate languages as used by Benton et al. [24] and Peyton-Jones et al. [223]. These partition expressions into categories of values and computations, similar to the isolation of primitive terms in let-normal form [229, 245]. This allows one to treat side-effects (memory access, IO, exceptions, etc.) in a uniform way, following Moggi [200], and thus simplifies reasoning about program analyses and the associated transformations in the presence of impure language features.

### 中文翻译

与续延和直接风格表示密切相关的，还有 Benton 等人 [24] 与 Peyton-Jones 等人 [223] 使用的**单子式中间语言（monadic intermediate language）**。

这类语言把表达式划分成“值”和“计算”两个类别，类似于 let 范式把原始项从一般表达式中单独分离出来 [229, 245]。按照 Moggi [200] 的思路，这种划分允许编译器以统一方式处理内存访问、输入输出、异常等副作用。因而，即使源语言包含非纯特性，对程序分析及其相关变换进行推理也会更简单。

### λ-提升、λ-删除与 SSA 消除

### English

Lambda-lifting and dropping are well-known transformations in the functional programming community, and are studied in-depth by Johnsson [155] and Danvy et al. [91].

Rideau et al. [247] present an in-depth study of SSA destruction, including a verified implementation in the proof assistant Coq for the “windmills” problem, i.e., the task of correctly introducing φ-compensating assignments. The local algorithm to avoid the lost-copy problem and swap problem identified by Briggs et al. [50] was given by Beringer [25]. In this solution, the algorithm to break the cycles is in line with the results of May [196].

### 中文翻译

λ-提升与 λ-删除是函数式编程领域众所周知的程序变换；Johnsson [155] 和 Danvy 等人 [91] 对它们进行了深入研究。

Rideau 等人 [247] 对 SSA 消除进行了深入研究，其中包括一个使用 Coq 证明助理验证过的 “windmills” 问题实现。所谓 windmills 问题，就是如何正确引入补偿 φ 函数语义所需的赋值。Briggs 等人 [50] 指出了复制丢失与交换问题，而 Beringer [25] 给出了避免这两个问题的局部算法；该方案中用于打破复制环的算法与 May [196] 的研究结果一致。

### 循环嵌套森林

### English

We are not aware of previous work that transfers the analysis of loop nesting forests to the functional setting, or of loop analyses in the functional world that correspond to loop nesting forests. Our discussion of Steensgaard's construction was based on a classification of loop nesting forests by Ramalingam [236], which also served as the source of the example in Fig. 6.7.

Two alternative constructions discussed by Ramalingam are those by Sreedhar, Gao and Lee [266], and Havlak [142]. To us, it appears that a functional reading of Sreedhar, Gao, and Lee's construction would essentially yield the nesting mentioned at the beginning of Sect. 6.3. Regarding Havlak's construction, the fact that entry points of loops are not necessarily classified as headers appears to make an elegant representation in functional form at least challenging.

### 中文翻译

作者没有发现此前已有研究把循环嵌套森林分析迁移到函数式环境，也没有发现函数式语言领域存在与循环嵌套森林相对应的循环分析。本章对 Steensgaard 构造的讨论，建立在 Ramalingam [236] 对各种循环嵌套森林方案所做的分类之上；图 6.7 使用的示例同样来自 Ramalingam 的工作。

Ramalingam 讨论的另外两种构造，分别由 Sreedhar、Gao 和 Lee [266] 以及 Havlak [142] 提出。作者认为，如果从函数式角度解释 Sreedhar、Gao 和 Lee 的构造，得到的基本就是 6.3 节开头介绍的那种按支配树孩子组织函数的嵌套。对于 Havlak 的构造，困难在于循环入口点不一定都被归类为循环头；而函数式表示通常希望把公开的循环入口直接表示成递归函数组的名字，因此要得到优雅表示至少非常具有挑战性。

### 数据流分析与类型系统

### English

Extending the syntactic correspondences between SSA and functional languages, similarities may be identified between their characteristic program analysis frameworks, data-flow analyses and type systems. Chakravarty et al. [62] prove the correctness of a functional representation of Wegmann and Zadeck's SSA-based sparse conditional constant propagation algorithm [303]. Beringer et al. [26] consider data-flow equations for liveness and read-once variables, and formally translate their solutions to properties of corresponding typing derivations. Laud et al. [177] present a formal correspondence between data-flow analyses and type systems but consider a simple imperative language rather than SSA.

### 中文翻译

把 SSA 与函数式语言之间的语法对应继续向前推广，还能在二者各自典型的程序分析框架之间发现相似性：SSA 领域主要使用数据流分析，而函数式语言领域经常使用类型系统。

Chakravarty 等人 [62] 证明了 Wegmann 与 Zadeck 基于 SSA 的稀疏条件常量传播算法 [303] 的一种函数式表示是正确的。Beringer 等人 [26] 研究关于活跃性和“只读一次变量”的数据流方程，并把这些方程的解形式化地转换为相应类型推导所具有的性质。Laud 等人 [177] 则给出了数据流分析与类型系统之间的形式对应，不过他们处理的是一种简单命令式语言，而不是 SSA。

### 现实编译器中的循环分析

### English

At present, intermediate languages in functional compilers do not provide syntactic support for expressing nesting forest directly. Indeed, most functional compilers do not perform advanced analyses of nested loops. As an exception to this rule, the MLton compiler (http://mlton.org) implements Steensgaard's algorithm for detecting loop nesting forests, leading to a subsequent analysis of the loop unrolling and loop switching transformations [278].

### 中文翻译

目前，函数式编译器的中间语言并不提供直接表达循环嵌套森林的语法支持。事实上，大多数函数式编译器也不会对嵌套循环执行高级分析。

MLton 编译器（http://mlton.org）是这条规律的一个例外。它实现了 Steensgaard 的循环嵌套森林检测算法，并以检测结果为基础，继续分析循环展开和循环切换两种变换 [278]。

---

## Concluding Remarks（总结性说明）

### 其他 SSA 表示

### English

In addition to low-level functional languages, alternative representations for SSA have been proposed, but their discussion is beyond the scope of this chapter.

Glesner [129] employs an encoding in terms of abstract state machines [134] that disregards the sequential control flow inside basic blocks but retains control flow between basic blocks to prove the correctness of a code generation transformation. Later work by the same author uses a more direct representation of SSA in the theorem prover Isabelle/HOL for studying further SSA-based analyses.

### 中文翻译

除低层函数式语言外，研究者还提出了其他 SSA 表示方式，不过对这些方案的讨论超出了本章范围。

Glesner [129] 使用抽象状态机 [134] 编码 SSA。该编码忽略基本块内部的顺序控制流，却保留基本块之间的控制流，用于证明一项代码生成变换的正确性。同一作者后来的工作在定理证明器 Isabelle/HOL 中采用更直接的 SSA 表示，以研究其他基于 SSA 的程序分析。

### 用类型表示定义点

### English

Matsuno and Ohori [195] present a formalism that captures core aspects of SSA in a notion of (non-standard) types while leaving the program text in unstructured non-SSA form. Instead of classifying values or computations, types represent the definition points of variables.

Contexts associate program variables at each program point with types in the standard fashion, but the non-standard notion of types means that this association models the reaching-definitions relationship rather than characterizing the values held in the variables at runtime. Noting a correspondence between the types associated with a variable and the sets of def-use paths, the authors admit types to be formulated over type variables whose introduction and use corresponds to the introduction of φ-nodes in SSA.

### 中文翻译

Matsuno 和 Ohori [195] 提出一种形式系统：程序文本本身仍保持无结构的非 SSA 形式，而 SSA 的核心性质被编码进一种非标准的“类型”概念中。与普通类型用于分类运行时值或计算不同，这里的类型表示变量的定义点。

类型上下文仍然按照标准方式，在每个程序点把程序变量与类型关联起来。但是，由于这里采用非标准的类型概念，这种关联并不是在描述变量运行时保存了什么样的值，而是在建模哪些定义能够到达当前程序点，也就是**到达定义关系**。

作者观察到，某个变量关联的类型与该变量的 def-use 路径集合之间存在对应关系。为此，他们允许类型表达式包含类型变量；这些类型变量的引入与使用，对应 SSA 形式中 φ 节点的引入与使用。

### 把程序视为方程组

### English

Finally, Pop et al.'s model [233] dispenses with control flow entirely and instead views programs as sets of equations that model the assignment of values to variables in a style reminiscent of partial recursive functions. This model is discussed in more detail in Chap. 10.

### 中文翻译

最后，Pop 等人 [233] 的模型完全取消了控制流这一表示维度，转而把程序看成一组方程；这些方程描述如何给变量确定取值，其形式令人联想到部分递归函数。本书第 10 章会更详细地讨论这种模型。

---

## 全章术语表

| 英文术语 | 中文译法 | 本章中的含义 |
|---|---|---|
| functional representation | 函数式表示 | 使用绑定、函数与参数表示 SSA 结构 |
| dominance | 支配关系 | 从过程入口到某节点的每条路径都经过支配节点 |
| dominance region | 支配区域 | 由某定义或基本块支配的程序区域 |
| syntactic / lexical scope | 语法／词法作用域 | 某个绑定对变量名有效的程序文本区域 |
| binding | 绑定 | 在一个作用域内让名字指向某个值或函数 |
| shadowing | 遮蔽 | 内层同名绑定暂时隐藏外层绑定 |
| free variable | 自由变量 | 在当前表达式内没有相应绑定的变量 |
| α-renaming | α-重命名 | 保持绑定结构和语义的变量改名 |
| variable capture | 变量捕获 | 错误改名使原自由变量被某个绑定纳入作用域 |
| referential transparency | 引用透明性 | 表达式含义只依赖自由变量，可用等价表达式替换 |
| continuation | 续延 | 描述当前计算结束后如何继续执行的函数式对象 |
| continuation-passing style, CPS | 续延传递风格 | 通过构造、传递和调用续延表达全部控制流 |
| direct style | 直接风格 | 使用局部尾递归函数而不显式传递续延 |
| tail call | 尾调用 | 函数返回前执行的最后一个操作是另一次调用 |
| let-normal form | let 范式 | 所有中间结果显式命名，调用实参为变量或常量 |
| arity | 元数 | 函数、φ 函数或基本块参数的数量 |
| live-in variable | 入口活跃变量 | 进入基本块时仍可能在之后被使用的变量 |
| closed function | 闭合函数 | 函数体的自由变量全部由形式参数等结构绑定 |
| pruned SSA | 剪枝 SSA | 不为入口不活跃变量插入 φ，但不一定达到最少数量 |
| minimal SSA | 最小 SSA | φ 函数数量满足最小性要求的 SSA |
| λ-lifting | λ-提升 | 把自由变量变成参数并把局部函数提升到外层 |
| λ-dropping | λ-删除 | 通过块下沉与参数删除消除多余函数参数／φ 函数 |
| block sinking | 块下沉 | 把函数声明移动到支配它所有调用的位置内部 |
| parameter dropping | 参数删除 | 利用共同外部绑定删除形式参数和相应实参 |
| loop-closed SSA | 循环闭合 SSA | 禁止循环内定义的 SSA 名字直接用于循环外 |
| lost-copy problem | 复制丢失问题 | 顺序执行并行复制时，源值被过早覆盖 |
| swap problem | 交换问题 | 两个或多个值形成置换环，无法直接顺序复制 |
| reducible CFG | 可约控制流图 | 循环具有结构化入口，可通过自然循环等方式分解 |
| irreducible CFG | 不可约控制流图 | 存在多入口等非结构化循环的控制流图 |
| strongly connected component, SCC | 强连通分量 | 任意两个节点之间都相互可达的最大节点集合 |
| loop nesting forest | 循环嵌套森林 | 通过循环体包含关系组织多个循环形成的森林 |
| loop header | 循环头 | 循环体中可由循环外直接进入的入口节点 |
| letrec | 递归绑定 | 允许一组函数在各自定义中互相引用的绑定结构 |
| monadic intermediate language | 单子式中间语言 | 区分值与计算并统一表示副作用的中间语言 |

## 全章总结

本章并不是简单地说“SSA 看起来像函数式语言”，而是给出一套可用于构造、优化和消除 SSA 的结构性对应：

1. **名字层面**：SSA 的唯一定义对应函数式绑定；α-重命名对应 SSA 重命名。
2. **数据流层面**：φ 函数对应续延或局部函数的形式参数；调用实参对应 φ 输入。
3. **控制流层面**：基本块对应局部尾递归函数；跳转对应尾调用；汇合点对应被多个位置调用的函数。
4. **支配层面**：变量作用域对应定义的支配区域；函数嵌套对应基本块支配树。
5. **构造层面**：按入口活跃变量生成函数参数可以得到剪枝 SSA；全局 α-重命名得到唯一 SSA 名字。
6. **最小化层面**：块下沉使作用域结构显式化，参数删除则消除多余参数，也就是删除多余 φ 函数。
7. **循环层面**：函数的特殊放置可以构造 loop-closed SSA；递归函数元组及其嵌套可表示不可约循环和循环嵌套森林。
8. **消除层面**：把调用实参规范化为与形式参数一致，相当于插入 φ 补偿复制；置换环用一个临时变量打破。

最浓缩的结论是：

```text
SSA 中的支配、φ 函数和控制流图
可以分别读取为
函数式语言中的作用域、函数参数和尾调用结构。
```

这种读取方式让许多 SSA 不变量由语法构造自动保证，也为解释器、形式化证明、程序分析和优化变换提供了统一基础。
