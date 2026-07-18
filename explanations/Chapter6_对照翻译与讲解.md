# 第 6 章 Functional Representations of SSA（SSA 的函数式表示）

原书：*SSA-based Compiler Design*，Chapter 6，Lennart Beringer。

说明：本文按“英文原文 → 中文翻译 → 理解与补充”的顺序整理。术语首次出现时给出英文；后文主要使用中文术语。内容覆盖所提供 PDF 全部 26 页，即原书第 63–88 页。

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

can only be carried out if `y` is not a free variable of `e₂`. Moreover, if `e₂` already contains preexisting bindings to `y`, the substitution first renames these bindings suitably. The renaming only affects `e₂`: occurrences of `x` or `y` in `e₁` refer to conceptually different variables. In general, the semantics-preserving renaming of bound variables is called α-renaming.

Typically, program analyses for functional languages are compatible with α-renaming, and program transformations α-rename bound variables whenever necessary.

### 中文翻译

为了避免改变程序含义，新变量名必须避开可能发生的名字混淆。形式上，把

```text
let x = e₁ in e₂ end
```

改写成

```text
let y = e₁ in e₂[y ↔ x] end
```

只能在 `y` 不是 `e₂` 的自由变量时进行。如果 `e₂` 已经包含对 `y` 的其他绑定，替换前还必须适当重命名这些已有绑定。改名只影响 `e₂`；`e₁` 中出现的 `x` 或 `y` 指向概念上不同的变量。

这种保持语义的已绑定变量重命名统称为 **α-renaming（α-重命名）**。函数式语言的程序分析通常应当与 α-重命名兼容，而程序变换也会在必要时主动进行 α-重命名。

### 理解与补充

α-重命名的本质不是简单的文本替换，而是按绑定关系改名。比如：

```text
let x = 1 in x + y end
```

不能直接把 `x` 改成已经自由出现的 `y`，否则会发生**变量捕获（variable capture）**，把原来不同的两个变量错误地合并。

### English

A consequence of referential transparency is compositional equational reasoning: the meaning of a piece of code `e` depends only on its free variables and can be calculated from the meaning of its subexpressions. Hence, languages with referential transparency allow one to replace a subexpression by some semantically equivalent phrase without altering the meaning of the surrounding code. Since semantic preservation is a core requirement of program transformations, the suitability of SSA for formulating and implementing such transformations can be explained by the proximity of SSA to functional languages.

### 中文翻译

引用透明性的一个结果是可以进行**组合式等式推理（compositional equational reasoning）**：一段代码 `e` 的含义只取决于它的自由变量，并且可以由其子表达式的含义计算得到。

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

A simple way to represent this program in let-normalized direct style is to introduce one function `fᵢ` for each basic block `bᵢ`. The body of each `fᵢ` arises by introducing one let-binding for each assignment and converting jumps into function calls.

In order to determine the formal parameters of these functions we perform a liveness analysis. For each basic block `bᵢ`, we choose an arbitrary enumeration of its live-in variables. We then use this enumeration as the list of formal parameters in the declaration of `fᵢ`, and also as the list of actual arguments in calls to `fᵢ`.

We collect all function definitions in a block of mutually tail-recursive functions at the top level:

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

用 let 范式的直接风格表示该程序，一种简单方法是让每个基本块 `bᵢ` 对应一个函数 `fᵢ`。基本块里的每条赋值变成一个 `let` 绑定，跳转则变成函数调用。

为了确定函数的形式参数，需要执行活跃性分析。对每个基本块 `bᵢ`，任选一种顺序枚举其入口活跃变量；这份枚举同时作为 `fᵢ` 声明中的形式参数列表，以及每次调用 `fᵢ` 时的实参列表。

所有函数声明在顶层组成一个相互尾递归的函数块，得到代码（6.15）。

### English

The resulting program has the following properties:

- All function declarations are closed: The free variables of their bodies are contained in their formal parameter lists.
- Variable names are not unique, but the unique association of definitions to uses is satisfied.
- Each subterm `e₂` of a let-binding `let x = e₁ in e₂ end` corresponds to the direct control-flow successor of the assignment to `x`.

### 中文翻译

所得程序具有以下性质：

- 所有函数声明都是**闭合的（closed）**：函数体中的自由变量都包含在形式参数列表中；
- 变量名尚未全局唯一，但定义与使用之间已经存在唯一对应；
- 在绑定 `let x = e₁ in e₂ end` 中，子项 `e₂` 对应命令式程序里对 `x` 赋值后的直接控制流后继。

原文脚注指出：这里说的闭合不计函数标识符 `fᵢ`；函数标识符总能选择成与普通变量不同的名字。

### English

If desired, we may α-rename to make names globally unique. As the function declarations in code (6.15) are closed, all variable renamings are independent from each other. The resulting code corresponds precisely to the SSA program shown in Fig. 6.3:

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

Each formal parameter of `fᵢ` is the target of one φ-function for block `bᵢ`. The arguments of these φ-functions are the arguments in corresponding positions in calls to `fᵢ`. Since every call to `fᵢ` supplies as many arguments as `fᵢ` has formal parameters, all φ-functions in `bᵢ` have the same arity: the number of call sites of `fᵢ`. An arbitrary enumeration of call sites coordinates the relative positions of φ arguments.

### 中文翻译

如果需要，可以继续做 α-重命名，使所有变量名全局唯一。由于代码（6.15）的函数声明都是闭合的，各变量的重命名彼此独立。所得代码（6.16）与图 6.3 的 SSA 程序精确对应。

函数 `fᵢ` 的每个形式参数，对应基本块 `bᵢ` 中一个 φ 函数的目标。φ 函数的各个输入，对应每次调用 `fᵢ` 时同一位置上的实参。

每个调用点提供的实参数量都等于 `fᵢ` 的形式参数数量，所以 `bᵢ` 中所有 φ 函数具有相同元数；这个元数正好等于 `fᵢ` 的调用点数量。为了确定 φ 输入的排列顺序，只需任意选定一种调用点枚举顺序。

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

Under this perspective, construction of parameter lists amounts to equipping each `bᵢ` with φ-functions for all its live-in variables, followed by renaming. Thus, the method constructs pruned, but not minimal, SSA.

While legal, it introduces more φ-functions than necessary. Each superfluous φ-function corresponds to a situation where all call sites to some function `fᵢ` pass identical arguments. Eliminating such arguments is called λ-dropping, the inverse of the better-known λ-lifting transformation.

### 中文翻译

从这个角度看，按入口活跃变量生成参数列表，就等于先为每个 `bᵢ` 的所有入口活跃变量放置 φ 函数，再执行变量重命名。因此，该方法得到的是**剪枝 SSA（pruned SSA）**，但不是**最小 SSA（minimal SSA）**。

它虽然是合法 SSA，却引入了多余 φ 函数。每个多余 φ 函数都对应这样的函数参数：`fᵢ` 的所有调用点传入的都是相同变量。删除这些参数的技术称为 **λ-dropping（λ-删除）**，它是更常见的 **λ-lifting（λ-提升）**的逆变换。

---

## 6.2.2 λ-Dropping（λ-删除）— PDF 第 13–17 页

### English

λ-dropping may be performed before or after variable names are made distinct, but for our purpose, the former option is more instructive. The transformation consists of two phases, block sinking and parameter dropping.

### 中文翻译

λ-删除既可以在变量名全局唯一化之前进行，也可以在之后进行。这里选择在重命名之前处理，因为这样更容易看清其原理。λ-删除包含两个阶段：

1. **block sinking（块下沉）**；
2. **parameter dropping（参数删除）**。

### 6.2.2.1 Block Sinking（块下沉）

#### English

Block sinking analyses the static call structure to identify which function definitions may be moved inside each other. Whenever declarations contain

```text
f(x₁, ..., xₙ) = e_f
g(y₁, ..., yₘ) = e_g
```

where `f ≠ g` and all calls to `f` occur in `e_f` or `e_g`, the declaration of `f` can be moved into that of `g`. Note the similarity to dominance. Applied aggressively, block sinking makes the entire dominator-tree structure explicit in the program representation. Algorithms for computing the dominator tree from a CFG can therefore identify block-sinking opportunities, using the function call graph as the CFG.

#### 中文翻译

块下沉分析静态调用结构，判断哪些函数定义可以移动到其他函数内部。如果 `f ≠ g`，而所有对 `f` 的调用都只出现在 `f` 自身或 `g` 的函数体中，就可以把 `f` 的声明移动到 `g` 内部。

这个条件与支配关系非常相似。积极执行块下沉，相当于把整棵支配树显式编码进函数嵌套结构。因而可以直接把函数调用图视为 CFG，使用支配树算法寻找块下沉机会。

#### English

In example (6.15), `f₃` is only invoked from within `f₂`, and `f₂` is only called in the bodies of `f₂` and `f₁`. We may move `f₃` into `f₂`, and `f₂` into `f₁`.

One option is to place a moved function at the beginning of its host:

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

This does not change semantics because the moved declaration is closed. Moving `f` into the scope of `g`'s formal parameters and `g` itself does not alter the bindings referenced by uses inside `f`.

#### 中文翻译

在（6.15）中，`f₃` 只从 `f₂` 内调用；`f₂` 只从 `f₁` 和 `f₂` 自身内部调用。因此，可以把 `f₃` 移入 `f₂`，再把 `f₂` 移入 `f₁`。

一种策略是把被移动函数放在宿主函数开头，得到（6.17）。因为被移动的函数声明是闭合的，把它移入宿主形式参数和宿主函数名的作用域，不会改变函数体中任何变量使用所对应的绑定，所以程序语义不变。

#### English

An alternative is to insert `f` near the end of host `g`, close to calls to `f`. This additionally brings `f` into the scope of let-bindings in `e_g`. Referential transparency again preserves semantics because `f` is closed:

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

If `g` has one call site for `f`, `f` is generally inserted immediately before it. If `g` has several call sites, tail position means they lie in separate conditional arms, so `f` is inserted immediately before the conditional.

#### 中文翻译

另一种策略是把 `f` 放到宿主 `g` 的末尾、靠近调用 `f` 的位置。这样，`f` 的声明还会进入 `g` 函数体中各 `let` 绑定的作用域。由于 `f` 原先是闭合的，引用透明性保证这种移动仍不改变语义，得到代码（6.18）。

如果 `g` 中只有一个 `f` 调用点，通常把 `f` 插在该调用之前；如果有多个调用点，由于它们都在尾位置，必定分布在条件语句的不同分支中，此时把 `f` 插在条件语句之前。

#### 理解与补充

两种放置策略得到的嵌套结构都反映支配关系：`f₃` 嵌套在 `f₂` 内，`f₂` 又嵌套在 `f₁` 内，与图 6.3 的支配树一致。但靠近调用点放置通常更好，因为函数声明能看到更多宿主 `let` 绑定，下一阶段就可能删除更多参数。

### 6.2.2.2 Parameter Dropping（参数删除）

#### English

The second phase removes superfluous parameters using syntactic scope. Removing parameter `x` from function `f` means uses of `x` in `f` are no longer bound by `f`'s declaration, but by the innermost enclosing binding for `x`. To preserve meaning, that enclosing binding must also contain every call to `f`, because the binding at the call site determines the actual value previously passed as the argument.

For example, both parameters of `f₃` in (6.18) may be removed. We can statically predict that its `y` is the result of `x × z`, bound to `y` in `f₂`, and its `v` is always the value passed through parameter `v` of `f₂`. The bindings at the declaration of `f₃` are identical to those at its call.

#### 中文翻译

λ-删除的第二阶段根据语法作用域删除多余参数。若从函数 `f` 的参数列表中删除 `x`，函数体中对 `x` 的使用就不再由 `f` 的形式参数绑定，而会转而引用包围 `f` 声明的、距离最近的 `x` 绑定。

为了保持程序含义，这个外部绑定还必须包围所有对 `f` 的调用，因为删除前真正传入的值，是由调用点适用的绑定决定的。

例如，（6.18）中 `f₃` 的两个参数都可以删除。`y` 总是实例化为 `f₂` 中 `x × z` 的结果，而 `v` 总是 `f₂` 参数 `v` 所携带的值。`f₃` 声明处与调用处适用的是同一组 `y` 和 `v` 绑定。

#### English

In general, parameter `x` may be dropped from a possibly recursive function `f` if:

1. The tightest scope for `x` enclosing the declaration of `f` coincides with the tightest scope for `x` surrounding every call site to `f` outside its declaration.
2. The tightest scope for `x` enclosing every recursive call to `f` is the one associated with formal parameter `x` in the declaration of `f`.

The same idea applies simultaneously to a mutually recursive block `f₁, ..., fₙ`: the outside binding at the declaration must coincide with that at every external call, while each internal call must use the binding associated with the caller's corresponding formal parameter. Dropping a parameter removes it from both the formal parameter lists and the argument lists of corresponding calls.

#### 中文翻译

一般来说，如果可能递归的函数 `f` 满足以下两个条件，就可以删除参数 `x`：

1. 包围 `f` 声明的最内层 `x` 作用域，与包围 `f` 声明外每个调用点的最内层 `x` 作用域相同；
2. 对 `f` 的每个递归调用，都位于 `f` 形式参数 `x` 所建立的作用域中。

对于一组相互递归函数 `f₁, ..., fₙ`，也可以同时删除共同参数 `x`：声明这组函数时适用的外部绑定，必须与所有外部调用点适用的绑定一致；而组内调用必须使用调用者相应形式参数建立的绑定。

删除参数时，既要从函数声明的形式参数列表中移除它，也要从所有对应调用的实参列表中移除它。

#### English

Applying these rules to (6.18), both parameters of non-recursive `f₃` are removed:

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

For recursive `f₂`, `y` cannot be removed because the recursive call lies in the scope of the let-binding that redefines `y` in `f₂`. In contrast, neither `v` nor `z` has a binding occurrence in `f₂`'s body. Their scopes at the external call coincide with those at the declaration, so both can be removed:

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

#### 中文翻译

把规则用于（6.18），可以删除非递归函数 `f₃` 的两个参数，得到（6.19）。

再看递归函数 `f₂`：递归调用位于函数体中新 `y` 绑定的作用域内，因此不能删除 `y`。反之，`f₂` 的函数体没有重新绑定 `v` 或 `z`；外部调用点和声明点适用的 `v`、`z` 作用域完全相同，所以可以删除这两个参数，得到（6.20）。

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

`y` 在循环中被重新定义，所以无法通过参数删除消去；这也正是其 φ 函数必须保留的原因。

靠近调用点放置函数优于放在宿主开头，因为靠近调用点可能使函数声明进入更多宿主 `let` 绑定的作用域，从而允许删除更多参数。

此外，一个只有单一直接前驱的基本块不需要 φ 函数：该块必然被其直接前驱支配，可以把对应函数嵌套到前驱函数中并紧贴调用点声明，于是所有参数都能删除。

---

## 6.2.3 Nesting, Dominance, Loop-Closure（嵌套、支配与循环闭合）— PDF 第 17–20 页

### English

Analysing whether function definitions may be nested inside one another is tantamount to analysing imperative dominance. Function `fᵢ` may be moved inside `fⱼ` exactly if all non-recursive calls to `fᵢ` come from within `fⱼ`; equivalently, every path from the initial program point to `bᵢ` traverses `bⱼ`; equivalently, `bⱼ` dominates `bᵢ`.

This extends the earlier observation that lexical scope coincides with the dominance region and definitions should dominate uses. Functional languages do not distinguish code from data with respect to binding and use; CPS already demonstrated this by binding code-representing continuations to variables with `let`.

Thus, the dominator tree suggests a function-nesting scheme in which all children of a node are represented as one block of mutually recursive function declarations.

### 中文翻译

判断函数定义能否互相嵌套，本质上就是分析命令式程序的支配结构。函数 `fᵢ` 可以移入 `fⱼ`，当且仅当对 `fᵢ` 的所有非递归调用都来自 `fⱼ` 内部；这又等价于从程序入口到 `bᵢ` 的每条路径都经过 `bⱼ`，也就是 `bⱼ` 支配 `bᵢ`。

这把“词法作用域等于支配区域、定义应当支配使用”的观察从普通变量扩展到函数标识符。就变量绑定与使用而言，函数式语言并不区分代码和数据；CPS 中已经用 `let` 把表示代码的续延绑定到变量。

因此，支配树直接给出一种函数嵌套方案：支配树某个节点的全部孩子，可表示成一组相互递归的函数声明。

### English

The choice of function placement corresponds to SSA variants. In loop-closed SSA form, SSA names defined in a loop must not be used outside that loop. Special unary φ-nodes are inserted at loop exits. As the loop is unrolled, the arity of these initially trivial φ-nodes grows with the number of unrollings, and the continuation always receives the value from the final executed iteration.

In the running example, the only loop-defined value used in `f₃` is `y`. To prevent dropping `y` from `f₃`, place `f₃` at the beginning of loop header `f₂`, before the let-binding for `y`. The placement policy is:

- Functions targeted by loop-exiting calls and having live-in variables defined in the loop are placed at the beginning of loop headers.
- Other functions are placed at the end of their hosts.

### 中文翻译

函数放在哪里，对应不同的 SSA 变体。在**循环闭合 SSA（loop-closed SSA）**中，循环内定义的 SSA 名字不能直接在循环外使用。为此，需要在循环出口插入专用的一元 φ 节点。

循环展开后，这些原本平凡的一元 φ 节点的元数会随展开次数增加；循环后的 continuation 总是接收实际执行的最后一次迭代所产生的值。

在贯穿示例中，循环内定义、又在 `f₃` 中使用的值只有 `y`。为了阻止参数删除阶段从 `f₃` 中删掉 `y`，要把 `f₃` 放到循环头 `f₂` 的开头，也就是重新绑定 `y` 之前。由此得到放置策略：

- 如果某函数是退出循环调用的目标，而且它有在循环内定义的入口活跃变量，就把它放在循环头开头；
- 其他函数仍放在各自宿主的末尾。

### English

Applying this policy to (6.15) gives:

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

We may drop `v`, but not `y`, from `f₃`, and drop `v` and `z` from `f₂`:

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

把上述放置策略应用于（6.15），得到（6.21）。随后可以从 `f₃` 删除 `v`，但不能删除 `y`；还可以从 `f₂` 删除 `v` 和 `z`，最终得到（6.22）。

（6.22）对应的 SSA 在 `b₃` 开头包含所需的循环闭合 φ 节点：

```text
b₃:
    y₃ ← φ(y₄)
    w₁ ← v₁ + y₃
    return w₁
```

（6.21）、（6.22）的函数嵌套结构，都与原命令式程序及其循环闭合 SSA 的支配结构一致。

### 图 6.5 与循环展开

图 6.5a 展示循环闭合形式；图 6.5b 展开一次循环。函数式表示通过复制 `f₂` 的函数体、但不复制 `f₃` 的声明来完成展开：

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

两个对 `f₃` 的调用都处于 `f₃` 声明的作用域内，并携带正确的循环闭合参数。在对应的 SSA 中，`b₃` 开头的一元 φ 节点变成非平凡的二元 φ 节点，其两个输入对应进入 `b₃` 的两条控制流边，也对应代码（6.23）中 `f₃` 的两个调用点。

展开后的函数调用与嵌套结构，仍与展开后 SSA 的控制流和支配结构一致。

---

## 6.2.4 Destruction of SSA（SSA 消除）— PDF 第 20 页

### English

In the examples before globally unique renaming, every call's argument list coincides with the formal parameter list of the invoked function. Functional programs do not generally obey this discipline, and optimization often destroys it. Programs that do obey it can be converted immediately to imperative non-SSA form.

Thus, SSA destruction amounts to converting a functional program with arbitrary argument lists into one where argument and formal-parameter lists coincide for every function. This can be achieved by introducing let-bindings `let x = y in e end`.

For example, if `f` is declared as `function f(x, y, z) = e`, the call

```text
f(v, z, y)
```

may be converted to

```text
let x = v,
    a = z,
    z = y,
    y = a
in f(x, y, z) end
```

corresponding to move instructions introduced by imperative SSA destruction.

### 中文翻译

在全局唯一重命名之前的示例中，每次调用的实参列表都与被调用函数的形式参数列表相同。一般函数式程序不必遵守这种规范，而且优化变换经常会破坏它。不过，只要程序保持这项规范，就可以立即转换成命令式非 SSA 形式。

因此，SSA 消除可以理解为：把一个调用实参任意的函数式程序，变换成对每个函数而言“实参列表与形式参数列表一致”的程序。实现方法是引入 `let x = y in e end` 形式的复制绑定。

例如，`f` 的形式参数顺序为 `(x, y, z)`，而调用传入 `(v, z, y)` 时，需要借助临时变量 `a` 完成 `y` 与 `z` 的交换。这正对应命令式 SSA 消除时插入的 move 指令。

### English

Appropriate transformations can be formulated as manipulations of the functional representation, although the target format is not immune to α-renaming and is therefore only syntactically a functional language.

A local algorithm can process each call site independently and avoid the lost-copy and swap problems. Rather than introducing let-bindings for every parameter position, its work scales with the number and size of cycles spanning identically named arguments and parameters. One additional variable, `a` above, is reused to break the cycles one by one.

### 中文翻译

因此，SSA 消除也可以完全表述成对函数式表示的变换。不过，目标形式不再对 α-重命名保持语义不变，所以它只能在语法表面上算作函数式语言。

可以设计一个逐调用点处理的局部算法，避免 **lost-copy（复制丢失）**和 **swap（交换）**问题。该算法不必为每个参数位置都增加复制，而只需处理同名实参与形参之间形成的置换环；其开销取决于环的数量和大小。一个额外临时变量就足以逐个打破所有环，上例中的 `a` 就承担这一作用。

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

### 中文翻译

如 λ-删除部分所述，块下沉由基本块之间的支配关系控制。因此，如果一棵典型支配树以 `b` 为根，并有以 `b₁, ..., bₙ` 为根的子树，那么最自然的函数式表示就是：把各 `fᵢ` 的声明组成一个函数块，并嵌套在 `f` 的声明内部，如代码（6.24）。

这个直接方案只表达“父节点支配孩子”，但没有充分利用兄弟子树之间更细的控制流关系。利用这些额外信息，可以得到与 SSA 文献中**循环嵌套森林（loop nesting forest）**对应的更精细放置方式。

---

### 可约 CFG 中的精细放置

### English

These refinements arise if we enrich the dominance tree by adding arrows `bᵢ → bⱼ` whenever the CFG contains a directed edge from one of the dominance successors of `bᵢ`—that is, a descendant of `bᵢ` in the dominator tree—to `bⱼ`.

For a reducible CFG, the resulting graph contains only trivial loops. Ignoring self-loops, we perform a post-order DFS, or more generally a reverse topological ordering, among the `bᵢ` and stagger function declarations according to the order.

For the CFG in Fig. 6.6a and enriched dominance tree in Fig. 6.6b, one possible order of `b`'s children is `[b₅, b₁, b₃, b₂, b₄]`, producing:

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

精细化方法首先增强支配树：如果 CFG 中存在一条有向边，从 `bᵢ` 的某个支配后继——即支配树中 `bᵢ` 的后代——指向 `bⱼ`，就在增强图中添加箭头 `bᵢ → bⱼ`。

对于**可约 CFG（reducible CFG）**，所得增强图只包含平凡循环。忽略自环后，可以在这些 `bᵢ` 上执行后序 DFS，更一般地说就是求一个逆拓扑序，然后按该顺序错开函数声明的位置。

图 6.6 的例子中，`b` 的孩子可按 `[b₅, b₁, b₃, b₂, b₄]` 排列，得到代码（6.25）。这个顺序不是唯一的。

### 图 6.6 解读

图 6.6a 是 CFG；图 6.6b 把 CFG 中跨越支配子树的边以虚线叠加到实线支配树上。

代码（6.25）仍像朴素方案一样保持支配关系，但进一步限制了名字可见性：

- `f₁` 在 `e₅` 中不可见；
- `f₃` 在 `f₁` 或 `f₅` 中不可见；
- 只有确实可能沿控制流到达相应块的代码，才能看到这些局部函数名。

### English

The code respects dominance much like the naive placement, but additionally makes `f₁` inaccessible from `e₅`, and `f₃` inaccessible from `f₁` or `f₅`. Since reordering does not move function declarations inside one another—no declaration crosses the scope of another function's formal parameters—it does not affect the later potential for parameter dropping.

### 中文翻译

这种代码与朴素放置一样尊重支配关系，却能进一步缩小函数名的可见范围。因为重新排序没有把一个函数声明移入或移出另一个函数形式参数的作用域，所以不会影响后续参数删除的机会。

---

### 使用 λ 抽象区分递归与非递归结构

### English

Declaring functions using λ-abstraction brings further improvements. It allows loops and non-recursive control-flow structures to be distinguished syntactically using `letrec` and `let`, and further restricts function-name visibility.

In Fig. 6.6, `b₃` is immediately dominated by `b`, but its only control-flow predecessors are `b₂/g` and `b₄`. We would like `f₃` to be local to tuple `(f₂, f₄)`, invisible to `f`. This can be achieved with let/letrec bindings and pattern matching, inserting the shared declaration of `f₃` between declaration of the names `f₂`, `f₄` and the λ-bindings of their formal parameters:

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

使用 λ 抽象声明函数还能进一步改进表示。许多函数式语言用 `let` 声明非递归绑定、用 `letrec` 声明递归绑定；借助这种区别，可以在语法上区分循环与非递归控制流，并进一步限制函数名的可见范围。

在图 6.6 中，`b₃` 的直接支配者是 `b`，但它真正的控制流前驱只有 `b₂/g` 和 `b₄`。因此，希望 `f₃` 只对函数元组 `(f₂, f₄)` 可见，而对外层 `f` 不可见。

代码（6.26）使用 `let`、`letrec` 和模式匹配实现这一目标：把共享的 `f₃` 声明放在 `f₂`、`f₄` 名字的递归声明与二者形式参数的 λ 绑定之间。

### English

The recursiveness of `f₄` is inherited by the function pair `(f₂, f₄)`, while `f₃` remains non-recursive. In general, the role of `f₃` is played by any merge point `bᵢ` that is not directly called from dominator node `b`.

### 中文翻译

`f₄` 的递归性由函数对 `(f₂, f₄)` 整体承担，而 `f₃` 本身仍然是非递归函数。一般而言，任何不是从支配节点 `b` 直接调用的汇合点 `bᵢ`，都可以采用类似 `f₃` 的私有声明方式。

---

### 不可约 CFG 与 Steensgaard 循环

### English

For irreducible CFGs, the enriched dominance tree is no longer acyclic even after self-loops are ignored. The functional representation then depends not only on DFS order, but also on how the enriched graph is partitioned into loops. Each loop forms a strongly connected component (SCC), and different partitionings correspond to different notions of loop nesting forests.

Among the loop nesting forest strategies in SSA literature, Steensgaard's scheme is particularly appealing from a functional perspective.

In Steensgaard's notion, the headers `H` of a loop `L = (B, H)` are precisely the entry nodes of body `B`: nodes in `B` with a direct predecessor outside `B`.

### 中文翻译

对于**不可约 CFG（irreducible CFG）**，即使忽略自环，增强支配树也不再是无环图。此时，函数式表示不仅依赖 DFS 顺序，还依赖如何把增强图划分成循环。

每个循环构成一个**强连通分量（strongly connected component，SCC）**；不同划分方法对应不同的循环嵌套森林定义。SSA 文献提出了多种方案，其中 Steensgaard 的方案从函数式角度尤其自然。

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

Figure 6.8a shows the CFG-enriched dominance tree of `G₀`. The body of `L₀` is the maximal SCC; after back edges `w → u` and `x → v` break cycles `(u, w)` and `(x, v)`, the body of `L₁` is likewise identified. The resulting loop nesting forest is shown in Fig. 6.8b. Loops are ellipses labelled with their headers and nested according to body containment `B₁ ⊂ B₀`.

### 中文翻译

图 6.8a 展示 `G₀` 的 CFG 增强支配树。最大 SCC 很容易识别为外层循环 `L₀` 的循环体；删除回边 `w → u` 和 `x → v`，打破环 `(u, w)` 与 `(x, v)` 后，又能识别内层循环 `L₁`。

图 6.8b 给出最终的循环嵌套森林。椭圆表示循环，并标注相应循环头；由于 `B₁ ⊂ B₀`，内层循环 `{w, x}` 嵌套在外层循环 `{u, v}` 中。

### English

In the functional representation, a loop `L = (B, {h₁, ..., hₙ})` yields a declaration block for functions `h₁, ..., hₙ`, with private declarations for non-headers `B \ H`. In the example, `L₀` provides entry points for headers `u` and `v`, but not for non-headers `w` and `x`. Instead, loop `L₁` formed by `w` and `x` is nested inside `L₀`:

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

在函数式表示中，一个循环 `L = (B, {h₁, ..., hₙ})` 会产生一组循环头函数 `h₁, ..., hₙ` 的声明，而非循环头节点 `B \ H` 则成为私有声明。

在本例中，外层循环 `L₀` 对外提供循环头 `u`、`v` 两个入口，但不公开非循环头 `w`、`x`。由 `w`、`x` 组成的内层循环 `L₁` 嵌套在 `L₀` 内部，退出函数 `exit` 又成为 `L₁` 的私有声明，得到代码（6.27）。

### English

This representation captures the essential information of Steensgaard's construction. The functional reading extends the correspondence between nesting and dominance from individual functions/basic blocks to groups: loop `L₀` dominates `L₁` because every path from `entry` to a node in `L₁` passes through `L₀`; more specifically, every path to a header of `L₁` passes through a header of `L₀`.

Each construction step may identify several loops because a CFG can contain several maximal SCCs. These SCC bodies do not overlap, so the result is a forest of trees like Fig. 6.8b. The relation between trees is acyclic, allowing corresponding function-declaration tuples to be placed according to the loop-extended notion of dominance.

### 中文翻译

代码（6.27）捕获了 Steensgaard 构造的全部关键信息。函数式解释把“单个函数嵌套对应基本块支配”的关系扩展到函数组和基本块组：

- 每条从 `entry` 到 `L₁` 中节点的路径都经过 `L₀`，所以 `L₀` 支配 `L₁`；
- 更具体地说，每条从 `entry` 到 `L₁` 某个循环头的路径，都经过 `L₀` 的某个循环头。

Steensgaard 构造的每一步都可能识别多个循环，因为 CFG 可能同时含有多个最大 SCC。这些 SCC 的循环体不会重叠，所以最终得到的是由多棵树组成的森林。不同树之间的关系必然无环，因此可以按照扩展到循环层面的支配关系放置对应的函数元组声明。

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

SSA 提出后不久，O'Donnell [213] 和 Kelsey [160] 就注意到 `let` 绑定与变量声明点之间的对应关系，并使用 CPS 把这种对应推广到程序结构的其他方面。Appel [10, 11] 在早期续延式编译工作 [9] 的基础上，使用直接风格表示推广了这种对应关系。

### CPS、直接风格与低层函数式语言

### English

Continuations and low-level functional languages have been intensively studied since their inception about four decades ago [174, 294]. Historical retrospectives include Reynolds [246] and Wadsworth [299]. Early studies of CPS and direct style include Reynolds and Plotkin [229, 244, 245]. Prominent CPS-based compilers include those of Sussman et al. [279] and Appel [9].

An active research area concerns the relative merits of functional representations, formal conversion algorithms, and efficient compilation to machine code, especially their integration with program analyses and optimizing transformations [92, 161, 243].

Kennedy's variant [161] explicitly names every control-flow point, taking the naming of merge points to its logical conclusion. The uniform representation enables efficient implementation of optimizing transformations while avoiding administrative overhead found in some alternatives.

### 中文翻译

续延和低层函数式语言自约四十年前诞生以来，一直受到深入研究 [174, 294]。Reynolds [246] 与 Wadsworth [299] 回顾了其历史发展；Reynolds 和 Plotkin [229, 244, 245] 的工作属于 CPS 与直接风格的早期研究。Sussman 等人 [279] 和 Appel [9] 的编译器是两个著名的 CPS 编译器实例。

目前仍很活跃的研究方向包括：比较各种函数式表示的优缺点、设计表示之间的形式化转换算法、把它们高效编译到机器码，以及让这些表示与程序分析和优化变换有效集成 [92, 161, 243]。

Kennedy [161] 的变体尤其有吸引力：它把“显式命名汇合点”的思路贯彻到底，要求所有控制流点都具有显式名字。所得表示高度统一，使优化变换能够高效实现，并避免某些替代方案容易产生的管理性开销。

### ANF、B-form 与 SIL

### English

Occasionally, direct style refers to the combination of tail-recursive functions and let-normal form, with the latter strengthened so only variables may appear as branch conditions. Variations include administrative normal form (A-normal form, ANF) [121], B-form [280], and SIL [285].

### 中文翻译

有时，“直接风格”特指“尾递归函数 + let 范式”的组合，而且 let 范式会进一步加强：分支条件中只能出现变量。这种规范的变体包括：

- **administrative normal form（管理范式）**，又称 **A-normal form（ANF）** [121]；
- **B-form** [280]；
- **SIL** [285]。

### 单子式中间语言

### English

Closely related to continuations and direct-style representations are monadic intermediate languages used by Benton et al. [24] and Peyton-Jones et al. [223]. They partition expressions into values and computations, similarly to isolating primitive terms in let-normal form [229, 245]. Following Moggi [200], this permits uniform treatment of side effects—memory access, IO, exceptions, and so on—and simplifies reasoning about analyses and transformations in the presence of impure features.

### 中文翻译

与续延和直接风格密切相关的，还有 Benton 等人 [24] 与 Peyton-Jones 等人 [223] 使用的**单子式中间语言（monadic intermediate language）**。

这类语言把表达式区分成“值”和“计算”，类似于 let 范式把原始项单独隔离出来 [229, 245]。按照 Moggi [200] 的思路，这样可以统一处理内存访问、I/O、异常等副作用，并简化在非纯语言特性存在时对程序分析及相关变换的推理。

### λ-提升、λ-删除与 SSA 消除

### English

Lambda-lifting and dropping are well-known functional-programming transformations, studied in depth by Johnsson [155] and Danvy et al. [91].

Rideau et al. [247] give an in-depth study of SSA destruction, including a Coq-verified implementation for the “windmills” problem: correctly introducing φ-compensating assignments. Beringer [25] gives the local algorithm avoiding the lost-copy and swap problems identified by Briggs et al. [50]; its cycle-breaking method agrees with results of May [196].

### 中文翻译

λ-提升与 λ-删除是函数式编程领域著名的程序变换，Johnsson [155] 和 Danvy 等人 [91] 对它们进行了深入研究。

Rideau 等人 [247] 详细研究了 SSA 消除，其中包括一个在 Coq 证明助理中验证的 “windmills” 问题实现；该问题要求正确插入用于补偿 φ 函数的赋值。Beringer [25] 提出了一个局部算法，用于避免 Briggs 等人 [50] 指出的复制丢失与交换问题；其中打破置换环的算法与 May [196] 的结果一致。

### 循环嵌套森林

### English

The author is unaware of prior work transferring loop nesting forests to the functional setting, or functional loop analyses corresponding to loop nesting forests. The discussion of Steensgaard's construction uses Ramalingam's classification [236] and its Fig. 6.7 example.

Ramalingam also discusses constructions by Sreedhar, Gao and Lee [266], and by Havlak [142]. A functional reading of Sreedhar, Gao and Lee would essentially yield the nesting described at the beginning of Sect. 6.3. In Havlak's construction, loop entry points are not necessarily classified as headers, making an elegant functional representation challenging.

### 中文翻译

作者尚未发现此前有工作把循环嵌套森林分析转移到函数式环境，也没有发现函数式领域存在与循环嵌套森林直接对应的循环分析。本章对 Steensgaard 构造的讨论基于 Ramalingam [236] 对循环嵌套森林的分类，图 6.7 的例子也来自该文。

Ramalingam 还讨论了 Sreedhar、Gao 和 Lee [266] 的构造以及 Havlak [142] 的构造。作者认为，对 Sreedhar、Gao 和 Lee 方法进行函数式解释，基本会得到 6.3 节开头的嵌套方式。Havlak 的构造不会总把循环入口归类成循环头，因此要给出优雅的函数式表示至少相当困难。

### 数据流分析与类型系统

### English

Extending syntactic correspondences, similarities can be identified between characteristic analysis frameworks: data-flow analyses for SSA and type systems for functional languages. Chakravarty et al. [62] prove correctness of a functional representation of Wegman and Zadeck's SSA-based sparse conditional constant propagation algorithm [303]. Beringer et al. [26] study data-flow equations for liveness and read-once variables and formally translate their solutions into properties of typing derivations. Laud et al. [177] give a formal correspondence between data-flow analyses and type systems, though for a simple imperative language rather than SSA.

### 中文翻译

除了语法结构的对应，还能在两类语言特有的程序分析框架之间找到相似性：SSA 常用数据流分析，函数式语言常用类型系统。

Chakravarty 等人 [62] 证明了 Wegman 与 Zadeck 的 SSA 稀疏条件常量传播算法 [303] 的一种函数式表示是正确的。Beringer 等人 [26] 研究活跃性与“只读一次变量”的数据流方程，并把方程解形式化转换成相应类型推导的性质。Laud 等人 [177] 给出了数据流分析与类型系统之间的形式对应，不过他们研究的是简单命令式语言，而不是 SSA。

### 现实编译器中的循环分析

### English

At present, intermediate languages in functional compilers do not provide syntactic support for expressing nesting forests directly, and most functional compilers do not perform advanced analyses of nested loops. An exception is MLton, which implements Steensgaard's loop-nesting-forest algorithm and subsequently analyses loop unrolling and loop switching transformations [278].

### 中文翻译

目前，函数式编译器的中间语言通常没有直接表达嵌套森林的语法支持，多数函数式编译器也不会对嵌套循环执行高级分析。

MLton 编译器是一个例外：它实现了 Steensgaard 的循环嵌套森林检测算法，并在此基础上分析循环展开和循环切换变换 [278]。

---

## Concluding Remarks（总结性说明）

### 其他 SSA 表示

### English

Besides low-level functional languages, alternative representations of SSA have been proposed but lie outside this chapter's scope.

Glesner [129] uses an encoding in abstract state machines [134] that ignores sequential control flow inside basic blocks while retaining control flow between blocks, in order to prove correctness of a code-generation transformation. Later work by the same author uses a more direct SSA representation in Isabelle/HOL to study further SSA-based analyses.

### 中文翻译

除了低层函数式语言，研究者还提出过其他 SSA 表示，不过详细讨论超出了本章范围。

Glesner [129] 使用抽象状态机 [134] 编码 SSA：忽略基本块内部的顺序控制流，但保留基本块之间的控制流，以证明一项代码生成变换的正确性。该作者后来的工作在定理证明器 Isabelle/HOL 中使用更直接的 SSA 表示，研究其他基于 SSA 的分析。

### 用类型表示定义点

### English

Matsuno and Ohori [195] present a formalism capturing core SSA aspects in non-standard types while leaving program text in unstructured non-SSA form. Rather than classifying values or computations, types represent variable definition points.

Contexts associate variables at each program point with types in the standard manner, but the unusual types make the association model reaching definitions rather than values held at runtime. Types associated with a variable correspond to sets of def-use paths; type variables may be introduced and used in a way corresponding to insertion of φ-nodes.

### 中文翻译

Matsuno 和 Ohori [195] 提出一种形式系统：程序文本仍保持无结构的非 SSA 形式，而 SSA 的核心性质则编码在一种非标准类型概念中。

这些类型不是用于分类运行时的值或计算，而是表示变量的定义点。类型上下文仍按通常方式，把每个程序点的变量与类型关联；但由于“类型”的含义不同，这种关联描述的是**到达定义关系**，而不是变量运行时保存的值。

一个变量所关联的类型与其 def-use 路径集合相对应；类型还可以包含类型变量，而类型变量的引入和使用分别对应 SSA 中 φ 节点的引入和使用。

### 把程序视为方程组

### English

Finally, Pop et al.'s model [233] dispenses with control flow entirely and views programs as sets of equations assigning values to variables, in a style reminiscent of partial recursive functions. This model is discussed further in Chap. 10.

### 中文翻译

最后，Pop 等人 [233] 的模型完全抛弃控制流，把程序视为一组描述变量取值的方程，其风格类似于部分递归函数。本书第 10 章会进一步讨论该模型。

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
