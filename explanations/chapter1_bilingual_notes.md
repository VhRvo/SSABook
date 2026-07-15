# 《SSA-based Compiler Design》第 1 章：对照翻译与解释

> 原文：Chapter 1, *Introduction*, Jeremy Singer
> 整理方式：英文原文 → 中文翻译 → 要点解释。页码指所提供 PDF 的页码。

## 0. 本章主线

SSA（Static Single Assignment，静态单赋值）最核心的规则是：**程序文本中，每个变量名只被定义一次**。如果源程序多次给同一个变量赋值，转成 SSA 时就给每次定义换一个新名字；如果不同控制流路径上的定义在某处汇合，则用 φ 函数合并。

可以先把本章压缩成下面三步：

```text
多次赋值                    分支汇合                    稀疏分析
x = 1                      y1 = ...                    定义直接连到使用
x = 2       --重命名-->    y2 = ...     --φ合并-->     无须在每个程序点
                            y3 = φ(y1,y2)                保存所有变量状态
```

---

## 1. Introduction（引言）— PDF 第 1 页

### English

In computer programming, as in real life, names are useful handles for concrete entities. The key message of this book is that having unique names for distinct entities reduces uncertainty and imprecision.

For example, consider overhearing a conversation about “Homer.” Without any more contextual clues, you cannot disambiguate between Homer Simpson and Homer the classical Greek poet; or indeed, any other people called Homer that you may know. As soon as the conversation mentions Springfield (rather than Smyrna), you are fairly sure that the Simpsons television series (rather than Greek poetry) is the subject. On the other hand, if everyone had a unique name, then there would be no possibility of confusing twentieth century American cartoon characters with ancient Greek literary figures.

### 中文翻译

在计算机程序设计中，名字和现实生活中一样，是指代具体实体的便利把手。本书的核心观点是：**让不同实体拥有互不相同的名字，可以减少不确定性和不精确性。**

例如，假设你偶然听见别人谈论 “Homer”。如果没有更多上下文线索，你无法判断他们说的是动画人物 Homer Simpson，还是古希腊诗人 Homer（荷马），也可能是你认识的其他同名者。一旦谈话中出现 Springfield，而不是 Smyrna，你就基本能确定话题是《辛普森一家》，而不是古希腊诗歌。反过来，如果每个人都有唯一的名字，20 世纪的美国动画人物和古希腊文学家就根本不会被混淆。

### 解释

这个类比直接对应程序变量。普通程序里的 `x` 可以在不同位置被反复赋值，因此仅看到 `x` 并不能确定它代表哪一次赋值得到的值；还必须知道当前执行到了哪里。SSA 用 `x1`、`x2`、`x3` 等唯一名字区分不同定义，使“这个值来自哪里”直接体现在名字里。

### English

This book is about the Static Single Assignment form (SSA), which is a naming convention for storage locations (variables) in low-level representations of computer programs. The term static indicates that SSA relates to properties and analysis of program text (code). The term single refers to the uniqueness property of variable names that SSA imposes. As illustrated above, this enables a greater degree of precision. The term assignment means variable definitions.

For instance, in the code

```text
x = y + 1;
```

the variable `x` is being assigned the value of expression `(y + 1)`. This is a definition, or assignment statement, for `x`. A compiler engineer would interpret the above assignment statement to mean that the lvalue of `x` (i.e., the memory location labeled as `x`) should be modified to store the value `(y + 1)`.

### 中文翻译

本书讨论静态单赋值形式（SSA）。SSA 是计算机程序低层表示中，对存储位置（变量）采用的一种命名约定。

- **Static（静态）**：SSA 讨论的是程序文本（代码）的性质及其分析，而不是某一次具体运行。
- **Single（单一）**：SSA 要求变量名具有唯一性，从而提高分析精度。
- **Assignment（赋值）**：这里指变量定义。

例如：

```text
x = y + 1;
```

变量 `x` 被赋予表达式 `y + 1` 的值。这是一条对 `x` 的定义（赋值）语句。编译器工程师通常把它理解为：修改 `x` 的左值，也就是标签为 `x` 的存储位置，使其中保存 `y + 1` 的值。

### 解释

“静态单赋值”容易被误解成“一个存储位置在运行时只能写一次”。实际上，SSA 约束的是**静态程序文本中的变量名字**。循环体里的某条 SSA 定义可以在运行时执行很多次，这一点在 1.2 节会再次强调。

---

## 1.1 Definition of SSA（SSA 的定义）— PDF 第 2–3 页

### English

The simplest, least constrained, definition of SSA can be given using the following informal prose:

> A program is defined to be in SSA form if each variable is a target of exactly one assignment statement in the program text.

However, there are various, more specialized, varieties of SSA, which impose further constraints on programs. Such constraints may relate to graph-theoretic properties of variable definitions and uses, or the encapsulation of specific control-flow or data-flow information. Each distinct SSA variety has specific characteristics.

### 中文翻译

最简单、约束最少的 SSA 定义可以非形式化地表述为：

> 如果程序文本中的每一个变量都恰好只作为一条赋值语句的目标，那么该程序就是 SSA 形式。

不过，SSA 还有多种更专门的变体，它们会对程序施加更多限制。这些限制可能涉及变量定义与使用之间的图论性质，也可能要求表示特定的控制流或数据流信息。每一种 SSA 变体都有自己的特征。

### 解释

这里的关键词是：

- **program text**：看静态代码，而不是看运行次数；
- **each variable**：准确说是每个“变量名”；
- **exactly one assignment target**：每个名字只有一个定义点，但可以有任意多个使用点。

例如：

```text
x1 = 1       // x1 的唯一定义
y1 = x1 + 1  // 使用 x1
z1 = x1 + 2  // 再次使用 x1，合法
```

### English

One important property that holds for all varieties of SSA, including the simplest definition above, is referential transparency: i.e., since there is only a single definition for each variable in the program text, a variable’s value is independent of its position in the program.

We may refine our knowledge about a particular variable based on branching conditions, e.g., we know the value of `x` in the conditionally executed block following an `if` statement that begins with

```text
if (x == 0)
```

However, the underlying value of `x` does not change at this `if` statement.

### 中文翻译

所有 SSA 变体都具有一个重要性质：**引用透明性**。由于程序文本中的每个变量只有一个定义，变量的值不依赖于它在程序中的位置。

我们可以根据分支条件进一步缩小对某个变量取值的认识。例如，在下面条件成立后执行的代码块中：

```text
if (x == 0)
```

我们知道 `x` 等于 0。但 `if` 语句本身并没有改变 `x` 的底层值；它只是给了我们更多关于该值的信息。

### 解释

这里要区分两件事：

1. **值发生改变**：对变量重新定义；
2. **我们对值的认识变精确**：由控制流条件推断得出。

SSA 中，同一个名字不会被重新定义，所以 `x1` 无论出现在哪里，都指向同一个定义产生的值。分支条件可以让分析器知道 “在这个分支中 `x1 == 0`”，但不会让 `x1` 变成另一个值。

### English

Programs written in pure functional languages are referentially transparent. Such referentially transparent programs are more amenable to formal methods and mathematical reasoning, since the meaning of an expression depends only on the meaning of its subexpressions and not on the order of evaluation or side effects of other expressions.

For a referentially opaque program, consider the following code fragment.

```text
x = 1;
y = x + 1;
x = 2;
z = x + 1;
```

A naive (and incorrect) analysis may assume that the values of `y` and `z` are equal, since they have identical definitions of `(x + 1)`. However, the value of variable `x` depends on whether the current code position is before or after the second definition of `x`, i.e., variable values depend on their context.

When a compiler transforms this program fragment to SSA code, it becomes referentially transparent. The translation process involves renaming to eliminate multiple assignment statements for the same variable. Now it is apparent that `y` and `z` are equal if and only if `x1` and `x2` are equal.

```text
x1 = 1;
y  = x1 + 1;
x2 = 2;
z  = x2 + 1;
```

### 中文翻译

纯函数式语言编写的程序具有引用透明性。这类程序更适合形式化方法和数学推理，因为一个表达式的意义只取决于其子表达式的意义，而不依赖求值顺序或其他表达式的副作用。

下面这段代码则不是引用透明的：

```text
x = 1;
y = x + 1;
x = 2;
z = x + 1;
```

一种幼稚而错误的分析可能认为 `y` 和 `z` 相等，因为它们的定义看起来都是 `x + 1`。但 `x` 的值取决于当前代码位置是在第二次定义 `x` 之前还是之后。也就是说，变量的值依赖上下文。

编译器把它转换成 SSA 后，通过重命名消除了对同一个变量名的多次赋值：

```text
x1 = 1;
y  = x1 + 1;
x2 = 2;
z  = x2 + 1;
```

此时一眼就能看出：`y` 和 `z` 相等，当且仅当 `x1` 和 `x2` 相等。

### 解释

普通形式中，两个文本相同的表达式 `x + 1` 未必有相同值，因为两个 `x` 可能来自不同定义。SSA 形式中，表达式写成 `x1 + 1` 或 `x2 + 1`，其数据来源不再含糊。

在本例中：

```text
x1 = 1  => y = 2
x2 = 2  => z = 3
```

这种“定义来源显式化”是很多优化的基础，例如常量传播、公共子表达式消除和死代码删除。

---

## 1.2 Informal Semantics of SSA（SSA 的非形式化语义）— PDF 第 3–6 页

### 1.2.1 重命名

### English

In the previous section, we saw how straight-line sequences of code can be transformed to SSA by simple renaming of variable definitions. The target of the definition is the variable being defined, on the left-hand side of the assignment statement. In SSA, each definition target must be a unique variable name. Conversely variable names can be used multiple times on the right-hand side of any assignment statements, as source variables for definitions.

Throughout this book, renaming is generally performed by adding integer subscripts to original variable names. In general this is an unimportant implementation feature, although it can prove useful for compiler debugging purposes.

### 中文翻译

上一节说明了：对于没有分支的直线型代码，只需重命名各个变量定义，就能转换成 SSA。定义的目标是赋值语句左侧被定义的变量。在 SSA 中，每一个定义目标都必须使用唯一的变量名。反过来，变量名可以在赋值语句右侧作为源操作数被使用多次。

本书通常通过给原变量名添加整数下标来完成重命名。这一般只是一个不重要的实现细节，不过对调试编译器很有帮助。

### 解释

SSA 的限制是“一个名字一个定义”，不是“一个名字只能出现一次”。因此：

```text
x1 = 10           // 定义一次
y1 = x1 + 1       // 使用一次
z1 = x1 * 2       // 又使用一次
```

完全合法。下标本身没有数学意义；`x7` 不代表 `x` 的第 7 个数组元素，只表示这是编译器给某次定义生成的新版本名。

### 1.2.2 φ 函数与分支汇合

### English

The φ-function is the most important SSA concept to grasp. It is a special statement, known as a pseudo-assignment function. Some call it a “notational fiction.” The purpose of a φ-function is to merge values from different incoming paths, at control-flow merge points.

Consider the following code example and its corresponding control-flow graph (CFG) representation:

```text
x = input()
if (x == 42)
then
    y = 1          // block A
else
    y = x + 2      // block B
end
print(y)
```

### 中文翻译

φ（phi）函数是 SSA 中最重要的概念。它是一种特殊语句，称为“伪赋值函数”，也有人称它为一种“记号上的虚构”。φ 函数的用途是在控制流汇合点，把不同进入路径传来的值合并起来。

考虑上面的程序：若 `x == 42`，就在基本块 A 中令 `y = 1`；否则在基本块 B 中令 `y = x + 2`。两条路径汇合后执行 `print(y)`。

### English

There is a distinct definition of `y` in each branch of the `if` statement. So multiple definitions of `y` reach the `print` statement at the control-flow merge point. When a compiler transforms this program to SSA, the multiple definitions of `y` are renamed as `y1` and `y2`. However, the `print` statement could use either variable, dependent on the outcome of the `if` conditional test.

A φ-function introduces a new variable `y3`, which takes the value of either `y1` or `y2`. Thus the SSA version of the program is:

```text
x1 = input()
if (x1 == 42)
then
A:  y1 = 1
else
B:  y2 = x1 + 2
end
y3 = φ(A: y1, B: y2)
print(y3)
```

### 中文翻译

`if` 的两个分支各有一个不同的 `y` 定义。因此在控制流汇合处，到达 `print` 语句的 `y` 定义不止一个。转成 SSA 时，这两个定义分别被重命名为 `y1` 和 `y2`。但 `print` 到底应该使用哪一个，取决于条件判断选择了哪条路径。

φ 函数因此引入一个新变量 `y3`，让它取得 `y1` 或 `y2` 的值。SSA 版本如上所示。

### 解释

φ 不是普通的“二选一函数”，也不是：

```text
y3 = randomChoice(y1, y2)   // 错误理解
```

它的选择依据是**实际到达当前基本块的前驱控制流边**：

```text
从 A 到达汇合块 => y3 = y1
从 B 到达汇合块 => y3 = y2
```

因此，带前驱标签的写法 `φ(A: y1, B: y2)` 比简写 `φ(y1, y2)` 更明确。φ 的操作数个数通常等于该基本块的直接前驱数。

### English

In terms of their position, φ-functions are generally placed at control-flow merge points, i.e., at the heads of basic blocks that have multiple direct predecessors in control-flow graphs. A φ-function at block `b` has `n` parameters if there are `n` incoming control-flow paths to `b`.

The behaviour of the φ-function is to select dynamically the value of the parameter associated with the actually executed control-flow path into `b`. This parameter value is assigned to the fresh variable name on the left-hand side of the φ-function. Such pseudo-functions are required to maintain the SSA property of unique variable definitions in the presence of branching control flow.

Throughout this book, basic block labels will be omitted from φ-function operands when the omission does not cause ambiguity.

### 中文翻译

从位置上说，φ 函数通常放在控制流汇合点，也就是控制流图中拥有多个直接前驱的基本块开头。如果基本块 `b` 有 `n` 条进入路径，那么 `b` 中的 φ 函数就有 `n` 个参数。

φ 函数在动态行为上会选择与实际执行的进入路径相对应的参数值，再把该值赋给 φ 函数左侧的新变量名。在存在分支控制流时，需要这种伪函数来维持 SSA 的“定义唯一”性质。

本书在不会造成歧义时，会省略 φ 操作数中的基本块标签。

### 1.2.3 多个 φ 必须并行执行

### English

It is important to note that, if there are multiple φ-functions at the head of a basic block, then these are executed in parallel, i.e., simultaneously not sequentially. This distinction becomes important if the target of a φ-function is the same as the source of another φ-function, perhaps after optimizations such as copy propagation.

When φ-functions are eliminated in the SSA destruction phase, they are sequentialized using conventional copy operations. This subtlety is particularly important in the context of register allocated code.

### 中文翻译

必须注意：如果一个基本块开头有多个 φ 函数，它们在语义上是**并行执行**的，也就是同时执行，而不是依次执行。如果一个 φ 的目标变量恰好又是另一个 φ 的源变量——例如经过复制传播优化之后——这一区别就很重要。

在销毁 SSA、消除 φ 函数时，编译器会用普通复制操作把这些并行赋值顺序化。在已经分配寄存器的代码中，这个细节尤其重要。

### 解释

可以把多个 φ 理解成“并行赋值”。例如交换值：

```text
a_new = φ(..., b_old)
b_new = φ(..., a_old)
```

如果错误地顺序执行 `a = b; b = a`，第二条读到的已经是新 `a`，两个值就相同了。正确的顺序化通常需要临时变量：

```text
tmp = a
a = b
b = tmp
```

这就是 SSA destruction 中“并行复制解析”问题的直观来源。

### 1.2.4 φ 不是普通的可执行函数

### English

Strictly speaking, φ-functions are not directly executable in software, since the dynamic control-flow path leading to the φ-function is not explicitly encoded as an input to φ-function. This is tolerable, since φ-functions are generally only used during static analysis of the program. They are removed before any program interpretation or execution takes place.

However, there are various executable extensions of φ-functions, such as φ-if or γ functions, which take an extra parameter to encode the implicit control dependence that dictates the argument the corresponding φ-function should select. Such extensions are useful for program interpretation, if conversion, or hardware synthesis.

### 中文翻译

严格来说，φ 函数不能直接作为普通软件函数执行，因为“通过哪条动态控制流路径到达 φ”并没有被显式编码成 φ 的输入。这通常不是问题，因为 φ 主要用于程序的静态分析，并会在程序真正解释或执行之前被删除。

不过，φ 也存在可执行扩展，例如 φ-if 或 γ 函数。它们增加一个参数，显式编码原本隐含的控制依赖，用来决定选择哪个实参。这些扩展可用于程序解释、if-conversion（分支谓词化）或硬件综合。

### 解释

普通函数 `f(a,b)` 只看显式参数就应当能决定结果；而 `φ(a,b)` 还暗中依赖“控制流从哪条边过来”。所以 φ 更像控制流图上的语义记号，而不是机器指令。退出 SSA 时，编译器通常把相应复制放到前驱边或前驱块中。

### 1.2.5 循环中的 φ

### English

We present one further example in this section, to illustrate how a loop control-flow structure appears in SSA. Here is the non-SSA version of the program:

```text
x = 0;
y = 0;

while (x < 10) {
    y = y + x;
    x = x + 1;
}

print(y);
```

The SSA code features two φ-functions in the loop header; these merge incoming definitions from before the loop for the first iteration and from the loop body for subsequent iterations.

An equivalent textual form of the figure is:

```text
B0:
    x1 = 0
    y1 = 0

B1:                         // loop header
    y2 = φ(B0: y1, B2: y3)
    x2 = φ(B0: x1, B2: x3)
    if (x2 < 10) goto B2 else goto B3

B2:                         // loop body
    y3 = y2 + x2
    x3 = x2 + 1
    goto B1

B3:
    print(y2)
```

### 中文翻译

这个例子展示循环控制流的 SSA 形式。SSA 代码在循环头部放置了两个 φ 函数。第一次迭代时，它们合并的是循环开始前的初始定义；后续迭代时，它们合并的是循环体上一轮产生的定义。

### 解释

循环头有两条进入边：

```text
B0 -> B1   第一次进入循环：x2 取 x1，y2 取 y1
B2 -> B1   回边再次进入：    x2 取 x3，y2 取 y3
```

因此循环中的 φ 可理解为对“循环携带状态”的建模。`x2` 和 `y2` 表示当前迭代开始时的值；`x3` 和 `y3` 表示本轮循环体计算出的、将传给下一轮的值。

退出循环时打印 `y2`，因为条件 `x2 < 10` 不成立时，当前循环头上的 `y2` 就是最终累加结果。

### 1.2.6 SSA 不等于动态单赋值

### English

It is important to outline that SSA should not be confused with (dynamic) single assignment (DSA or simply SA) form used in automatic parallelization. Static Single Assignment does not prevent multiple assignments to a variable during program execution. For instance, in the SSA code fragment above, variables `y3` and `x3` in the loop body are redefined dynamically with fresh values at each loop iteration.

Full details of the SSA construction algorithm are given in Chapter 3. For now, it is sufficient to see that:

1. A φ-function has been inserted at the appropriate control-flow merge point where multiple reaching definitions of the same variable converged in the original program.
2. Integer subscripts have been used to rename variables `x` and `y` from the original program.

### 中文翻译

SSA 不应与自动并行化中使用的（动态）单赋值形式 DSA/SA 混淆。静态单赋值并不禁止程序运行时对某个变量进行多次赋值。例如，在上面的 SSA 循环中，循环体内的 `y3` 和 `x3` 每次迭代都会动态地产生新值。

SSA 构造算法的完整细节在第 3 章介绍。目前只需掌握：

1. 如果原程序中同一变量的多个到达定义在某个控制流汇合点相遇，就在适当位置插入 φ 函数；
2. 用整数下标对原程序中的 `x`、`y` 等变量进行重命名。

### 解释

“每个 SSA 名字只有一个静态定义点”不等于“这个定义点只执行一次”。在循环里，同一条指令可能执行很多次：

```text
x3 = x2 + 1
```

这条定义在代码里只出现一次，所以满足 SSA；但运行时每轮都可能执行，所以会产生一串动态值。区分**静态定义点**与**动态执行实例**，是理解 SSA 名称的关键。

---

## 1.3 Comparison with Classical Data-Flow Analysis（与经典数据流分析的比较）— PDF 第 6–7 页

### English

One of the major advantages of SSA form concerns data-flow analysis. Data-flow analysis collects information about programs at compile time in order to make optimizing code transformations. During actual program execution, information flows between variables. Static analysis captures this behaviour by propagating abstract information, or data-flow facts, using an operational representation of the program such as the control-flow graph (CFG). This is the approach used in classical data-flow analysis.

### 中文翻译

SSA 的主要优点之一体现在数据流分析上。数据流分析在编译时收集程序信息，以便进行优化变换。程序实际运行时，信息会在变量之间流动；静态分析则借助控制流图等程序表示，传播抽象信息（也叫数据流事实）来刻画这种行为。经典数据流分析采用的就是这种方法。

### 解释

“数据流事实”不是程序运行时的具体数值，而是编译器关心的抽象性质，例如：

- 变量是否为常量；
- 是否可能为 0 / null；
- 某个定义能否到达某个位置；
- 某变量在某点是否活跃。

经典的稠密分析通常为 CFG 的每个程序点维护一整组状态，并沿所有边迭代传播，直到不再变化。

### English

Often, data-flow information can be propagated more efficiently using a functional, or sparse, representation of the program such as SSA. When a program is translated into SSA form, variables are renamed at definition points. For certain data-flow problems (e.g., constant propagation) this is exactly the set of program points where data-flow facts may change.

Thus it is possible to associate data-flow facts directly with variable names, rather than maintaining a vector of data-flow facts indexed over all variables, at each program point.

### 中文翻译

使用 SSA 这种函数式或稀疏的程序表示，往往可以更高效地传播数据流信息。程序转换成 SSA 时，变量会在定义点被重命名。对某些数据流问题（例如常量传播）来说，定义点恰好就是数据流事实可能发生变化的全部位置。

因此，可以把数据流事实直接关联到变量名，而不用在每个程序点都维护一个按所有变量索引的数据流事实向量。

### 解释

假设程序有 1000 个 CFG 节点和 100 个变量。稠密方案在概念上可能需要处理许多“节点 × 变量”的状态，即使大部分状态没有变化。SSA 则让分析沿 def-use（定义—使用）关系走：

```text
x1 的定义 -> 使用 x1 的指令 -> 产生 y2 -> 使用 y2 的指令
```

对于常量传播，只有定义新 SSA 值时事实才可能改变，所以不必穿过大量无关程序点。这就是“稀疏”的含义：分析工作集中在真正相关的位置。

### English

Figure 1.1 illustrates this point through an example of non-zero value analysis. For each variable in a program, the aim is to determine statically whether that variable can contain a zero integer value (i.e., null) at runtime. Here `0` represents the fact that the variable is null, `≠0` the fact that it is non-null, and `⊤` the fact that it is maybe-null.

With classical dense data-flow analysis on the CFG in Fig. 1.1a, we would compute information about variables `x` and `y` for each of the entry and exit points of the six basic blocks in the CFG, using suitable data-flow equations. Using sparse SSA-based data-flow analysis on Fig. 1.1b, we compute information about each variable based on a simple analysis of its definition statement. This gives us seven data-flow facts, one for each SSA version of variables `x` and `y`.

### 中文翻译

图 1.1 用“非零值分析”说明这一点。分析目标是静态判断程序中的每个变量在运行时能否包含整数 0（即空值）。其中，`0` 表示变量确定为空，`≠0` 表示确定非空，`⊤` 表示可能为空。

在图 1.1a 的经典稠密 CFG 分析中，需要利用数据流方程，在 6 个基本块的每个入口和出口处分别计算 `x`、`y` 的信息。使用图 1.1b 的稀疏 SSA 分析时，只需根据每个变量的定义语句分析该变量。最终得到 7 个数据流事实，分别对应 `x`、`y` 的 7 个 SSA 版本。

### 解释

抽象值可以看成一个很小的格：

```text
       ⊤（可能为 0，也可能非 0）
      / \
     0  ≠0
```

当不同路径汇合时，如果一条路径说 `y1 = 0`，另一条说 `y2 ≠ 0`，那么 `y3 = φ(y1,y2)` 的结果就是 `⊤`。φ 不仅合并具体运行值，也对应静态分析事实的合并点。

### English

For other data-flow problems, properties may change at points that are not variable definitions. These problems can be accommodated in a sparse analysis framework by inserting additional pseudo-definition functions at appropriate points to induce additional variable renaming.

However, this example illustrates some key advantages of the SSA-based analysis.

1. Data-flow information propagates directly from definition statements to uses, via the def-use links implicit in the SSA naming scheme. In contrast, the classical data-flow framework propagates information throughout the program, including points where the information does not change or is not relevant.
2. The results of the SSA data-flow analysis are more succinct. In the example, there are fewer data-flow facts associated with the sparse (SSA) analysis than with the dense (classical) analysis.

### 中文翻译

对另一些数据流问题，性质可能在变量定义以外的位置发生变化。稀疏分析框架可以在适当位置插入额外的伪定义函数，触发进一步的变量重命名，以容纳这些问题。

这个例子展示了 SSA 分析的两个关键优点：

1. 数据流信息通过 SSA 命名中隐含的 def-use 链，直接从定义语句传播到使用处；经典数据流框架则会把信息传播到整个程序，包括事实不变化或根本无关的位置。
2. SSA 数据流分析的结果更简洁；在该例中，稀疏 SSA 分析需要关联的数据流事实比稠密经典分析更少。

### 稠密与稀疏的对照

| 方面 | 经典稠密分析 | SSA 稀疏分析 |
|---|---|---|
| 状态放在哪里 | CFG 的许多程序点 | SSA 定义/名字上 |
| 如何传播 | 沿 CFG 边传播 | 沿 def-use 链传播 |
| 无关位置 | 往往也要经过 | 通常可以跳过 |
| 结果规模 | 常较大 | 常较简洁 |
| 是否总能直接使用 | 通用 | 某些问题需插入额外伪定义 |

---

## 1.4 SSA in Context（SSA 的背景）— PDF 第 7–8 页

### 1.4.1 历史背景

### English

Throughout the 1980s, as optimizing compiler technology became more mature, various intermediate representations (IRs) were proposed to encapsulate data dependence in a way that enabled fast and accurate data-flow analysis. The motivation behind the design of such IRs was the exposure of direct links between variable definitions and uses, known as def-use chains, enabling efficient propagation of data-flow information. Example IRs include the program dependence graph and program dependence web.

Static Single Assignment form was one such IR, which was developed at IBM Research and announced publicly in several research papers in the late 1980s. SSA rapidly acquired popularity due to its intuitive nature and straightforward construction algorithm. The SSA property gives a standardized shape for variable def-use chains, which simplifies data-flow analysis techniques.

### 中文翻译

20 世纪 80 年代，随着优化编译器技术逐渐成熟，人们提出了多种中间表示（IR），希望以有利于快速、准确数据流分析的方式封装数据依赖。这些 IR 的设计动机是把变量定义与使用之间的直接联系——即 def-use 链——显式暴露出来，从而高效传播数据流信息。程序依赖图和程序依赖网都是这类 IR 的例子。

SSA 也是其中之一。它由 IBM Research 开发，并在 20 世纪 80 年代末的多篇论文中公开。SSA 由于直观、构造算法直接，很快流行起来。SSA 性质为变量的 def-use 链提供了标准化结构，因而简化了数据流分析技术。

### 解释

编译器中“IR 好不好”的重要标准之一，是它是否让后续分析和变换更容易。SSA 的价值不只是变量名更整齐，而是：一个使用 `x7` 的地方，可以直接追到 `x7` 唯一的定义。定义和使用之间的关系不再需要反复通过“到达定义分析”猜测。

### 1.4.2 当前应用

### English

The majority of current commercial and open source compilers, including GCC, LLVM, the HotSpot Java virtual machine, and the V8 JavaScript engine, use SSA as a key intermediate representation for program analysis. As optimizations in SSA are fast and powerful, SSA is increasingly used in just-in-time (JIT) compilers that operate on a high-level target-independent program representation such as Java byte-code, CLI byte-code (.NET MSIL), or LLVM bitcode.

Initially created to facilitate the development of high-level program transformations, SSA form has gained much interest due to its favourable properties that often enable the simplification of algorithms and reduction of computational complexity. Today, SSA form is even adopted for the final code generation phase, i.e., the back-end.

Many compilers perform SSA elimination before register allocation. Research on register allocation even allows the retention of SSA form until the very end of the code generation process.

### 中文翻译

当代大多数商业和开源编译器，包括 GCC、LLVM、HotSpot Java 虚拟机和 V8 JavaScript 引擎，都把 SSA 用作程序分析的关键中间表示。由于 SSA 上的优化快速而强大，SSA 也越来越多地用于 JIT 编译器。这类编译器常处理 Java bytecode、CLI bytecode（.NET MSIL）或 LLVM bitcode 等高级、目标无关的程序表示。

SSA 最初是为了便于开发高级程序变换，但它的良好性质经常能简化算法、降低计算复杂度，因此获得广泛关注。如今，SSA 甚至被用于最终代码生成阶段，也就是编译器后端。

许多编译器会在寄存器分配之前消除 SSA；而较新的寄存器分配研究甚至允许把 SSA 一直保留到代码生成过程的最后阶段。

### 解释

典型编译流水线可以粗略理解为：

```text
源代码
  -> 前端语义分析
  -> SSA IR
  -> SSA 上的分析与优化
  -> 消除 φ / 退出 SSA（有些编译器会更晚）
  -> 寄存器分配与机器码生成
```

不同编译器的具体位置不同。例如 LLVM IR 本身要求 SSA 形式；后端还会使用适合机器代码的 SSA 风格表示，直到某个阶段再处理 PHI 指令。

### 1.4.3 高级语言中的 SSA 性质

### English

Some high-level languages enforce the SSA property. The SISAL language is defined in such a way that programs automatically have referential transparency, since multiple assignments are not permitted to variables. Other languages allow the SSA property to be applied on a per-variable basis, using special annotations like `final` in Java, or `const` and `readonly` in C#.

The main motivation for allowing the programmer to enforce SSA in an explicit manner in high-level programs is that immutability simplifies concurrent programming. Read-only data can be shared freely between multiple threads, without any data dependence problems. This is becoming an increasingly important issue with the shift to multi- and many-core processors.

High-level functional languages claim referential transparency as one of the cornerstones of their programming paradigm. Thus functional programming supports the SSA property implicitly.

### 中文翻译

一些高级语言会强制 SSA 性质。SISAL 不允许对变量多次赋值，因此其程序天然具有引用透明性。另一些语言允许逐变量应用类似性质，例如 Java 的 `final`，以及 C# 的 `const` 和 `readonly`。

允许程序员在高级程序中显式实施这种性质，主要是因为不可变性可以简化并发编程。只读数据能够在多个线程之间自由共享，而不会产生数据依赖问题。随着处理器转向多核和众核，这一点日益重要。

高级函数式语言把引用透明性作为编程范式的基石之一，因此函数式编程隐式支持 SSA 性质。

### 解释

高级语言的不可变变量和编译器 IR 的 SSA 很相似，但不能完全等同：

- 高级语言不可变性是程序员可见的语言语义；
- SSA 主要是编译器内部表示的命名纪律；
- SSA 中循环体的一个静态定义仍可执行多次；
- 对内存、数组元素、对象字段等可变状态，还需要 Memory SSA 等进一步建模。

---

## 1.5 About the Rest of This Book（本书其余内容）— PDF 第 9–10 页

### English

In this chapter, we have introduced the notion of SSA. The rest of this book presents various aspects of SSA, from the pragmatic perspective of compiler engineers and code analysts. The ultimate goals of this book are:

1. To demonstrate clearly the benefits of SSA-based analysis.
2. To dispel the fallacies that prevent people from using SSA.

### 中文翻译

本章介绍了 SSA 的基本概念。本书其余部分将从编译器工程师和代码分析人员的实践视角，讨论 SSA 的不同方面。本书的最终目标是：

1. 清楚展示基于 SSA 的分析有什么好处；
2. 消除那些妨碍人们采用 SSA 的错误认识。

### 1.5.1 Benefits of SSA（SSA 的益处）

### English

SSA imposes a strict discipline on variable naming in programs, so that each variable has a unique definition. Fresh variable names are introduced at assignment statements and control-flow merge points. This serves to simplify the structure of variable def-use relationships and live ranges, which underpin data-flow analysis. There are three major advantages to SSA.

**Compile time benefit.** Certain compiler optimizations can be more efficient when operating on SSA programs, since referential transparency means that data-flow information can be associated directly with variables, rather than with variables at each program point.

**Compiler development benefit.** Program analyses and transformations can be easier to express in SSA. This means that compiler engineers can be more productive in writing new compiler passes and debugging existing passes. For example, the dead code elimination pass in GCC 4.x, which relies on an underlying SSA-based intermediate representation, takes only 40% as many lines of code as the equivalent pass in GCC 3.x, which does not use SSA. The SSA version is simpler since it relies on a general-purpose, factored-out data-flow propagation engine.

**Program runtime benefit.** Conceptually, any analysis and optimization that can be done under SSA form can also be done identically out of SSA form. Because of the compiler development benefit mentioned above, several compiler optimizations are shown to be more effective when operating on programs in SSA form.

### 中文翻译

SSA 对程序变量命名施加严格纪律，使每个变量都有唯一定义。在赋值语句和控制流汇合点引入新变量名，可以简化变量的 def-use 关系和活跃区间结构，而这些正是数据流分析的基础。SSA 有三类主要优势：

**编译时间方面。** 某些编译器优化在 SSA 程序上运行得更高效。由于引用透明性，数据流信息可以直接关联到变量，而不是关联到“每个程序点上的每个变量”。

**编译器开发方面。** 程序分析与变换在 SSA 中更容易表达，所以编译器工程师编写新 pass、调试现有 pass 时更高效。例如，基于 SSA IR 的 GCC 4.x 死代码消除 pass，其代码行数只有不使用 SSA 的 GCC 3.x 对应 pass 的 40%。SSA 版本更简单，是因为它复用了一个通用、独立的数据流传播引擎。

**程序运行时间方面。** 从概念上说，能在 SSA 中完成的分析和优化，在非 SSA 表示中也都能完成。SSA 并没有凭空创造新的优化能力；但由于开发与算法表达更简单，实践中一些优化在 SSA 上能实现得更有效，最终生成更好的程序。

### 解释

三种收益不要混在一起：

| 收益 | 直接受益者 | 原因 |
|---|---|---|
| 编译时间收益 | 编译器运行速度 | 稀疏传播，少处理无关程序点 |
| 开发收益 | 编译器工程师 | 数据依赖清晰，pass 更短、更易调试 |
| 运行时间收益 | 最终生成的程序 | 更容易实现强而可靠的优化 |

SSA 的最大现实价值往往是第二项：它让正确实现优化更容易。理论上“非 SSA 也能做”不等于工程上成本相同。

### 1.5.2 Fallacies About SSA（关于 SSA 的误解）

### English

Some people believe that SSA is too cumbersome to be an effective program representation. This book aims to convince the reader that such a concern is unnecessary, given the application of suitable techniques. The table below presents some common myths about SSA.

| Myth | Reference |
|---|---|
| SSA greatly increases the number of variables. | Chapter 2 reviews the main varieties of SSA, some of which introduce far fewer variables than the original SSA formulation. |
| SSA property is difficult to maintain. | Chapters 3 and 5 discuss simple techniques for the repair of SSA invariants that have been broken by optimization rewrites. |
| SSA destruction generates many copy operations. | Chapters 3 and 21 present efficient and effective SSA destruction algorithms. |

### 中文翻译

一些人认为 SSA 太笨重，无法成为有效的程序表示。本书希望说明：只要采用合适技术，这种担忧没有必要。常见误解包括：

| 误解 | 本书的回应 |
|---|---|
| SSA 会大幅增加变量数量。 | 第 2 章介绍主要 SSA 变体，其中一些比最初的 SSA 方案引入少得多的变量。 |
| SSA 性质很难维护。 | 第 3、5 章介绍优化重写破坏 SSA 不变量后，如何用简单技术修复。 |
| 销毁 SSA 会产生大量复制操作。 | 第 3、21 章介绍高效且有效的 SSA 销毁算法。 |

### 解释

SSA 确实会制造更多“名字”，但更多名字不等于更多运行时存储：

- 多个 SSA 名字的活跃区间可能互不重叠，因此可分配到同一个寄存器；
- φ 引入的复制常可通过合并（coalescing）消除；
- 编译器可以只为需要的变量或区域构造适当 SSA 变体；
- 优化修改局部代码后，可以增量修复 SSA，而不必每次从头重建。

---

## 2. 本章术语表

| 英文 | 推荐译法 | 本章中的含义 |
|---|---|---|
| Static Single Assignment (SSA) | 静态单赋值 | 每个变量名在程序文本中只有一个定义 |
| assignment / definition | 赋值 / 定义 | 产生一个变量值的语句 |
| use | 使用 | 在右侧或其他位置读取变量值 |
| lvalue | 左值 | 表示可写存储位置的表达式 |
| referential transparency | 引用透明性 | 同一名字始终指向同一定义的值，表达式意义不依赖隐藏状态变化 |
| control-flow graph (CFG) | 控制流图 | 以基本块为节点、控制转移为边的程序表示 |
| basic block | 基本块 | 单入口、通常单出口的直线指令序列 |
| control-flow merge point | 控制流汇合点 | 多条控制流路径进入同一基本块的位置 |
| φ-function | φ 函数 / phi 函数 | 按实际前驱边选择值的伪赋值 |
| predecessor | 前驱 | CFG 中有边进入当前块的基本块 |
| def-use chain | 定义—使用链 | 从某个定义到其各个使用点的联系 |
| live range | 活跃区间 | 一个值从定义到最后使用之间保持活跃的范围 |
| dense analysis | 稠密分析 | 在许多/全部程序点维护并传播状态 |
| sparse analysis | 稀疏分析 | 只在事实可能变化或相关的位置处理状态 |
| reaching definition | 到达定义 | 能沿某条路径到达给定位置且未被覆盖的定义 |
| SSA destruction | SSA 销毁 / 退出 SSA | 消除 φ 和 SSA 名字，转换为适合后续代码生成的形式 |

---

## 3. 一页式理解

### 3.1 为什么要 SSA？

普通变量名会被反复复用，导致每次使用可能对应多个定义。SSA 给每次定义一个唯一名字，使数据来源直接可见。

### 3.2 遇到直线代码怎么办？

重命名：

```text
x = 1; x = x + 1

=>

x1 = 1; x2 = x1 + 1
```

### 3.3 遇到分支怎么办？

在汇合点用 φ：

```text
if (...) y1 = 1 else y2 = 2
y3 = φ(y1, y2)
```

`y3` 取实际执行路径对应的值。

### 3.4 遇到循环怎么办？

循环头的 φ 在“初始值”和“上一轮产生的值”之间选择：

```text
i2 = φ(i_initial, i_next)
```

### 3.5 SSA 为什么帮助优化？

每个使用都能直接找到唯一定义，分析可以沿 def-use 链稀疏传播，不必在 CFG 的每个位置维护所有变量的状态。

### 3.6 最容易犯的三个错误

1. 把 SSA 理解成“运行时只能赋值一次”——错，约束的是静态定义点。
2. 把 φ 理解成普通函数——错，它隐式依赖实际进入当前块的控制流边。
3. 把多个 φ 当作顺序赋值——错，同一块开头的 φ 在语义上并行执行。

