# 第 2 章 Properties and Flavours（性质与变体）

原书：*SSA-based Compiler Design*，Chapter 2，Philip Brisk 与 Fabrice Rastello。

说明：本文按“英文原文 → 中文翻译 → 理解与补充”的顺序整理。术语首次出现时给出英文；后文主要使用中文术语。图中的空白框表示与当前讨论无关的基本块内容。

## 本章脉络

本章讨论的不是“什么是 SSA”，而是：满足“每个变量只定义一次、每次使用唯一对应一个定义”之后，还能形成哪些不同的 SSA 形态。

- **minimal SSA（最小 SSA）**：φ 函数数量达到理论最少。
- **strict SSA（严格 SSA）**：每个使用都被其定义支配。
- **pruned SSA（剪枝 SSA）**：不为死变量放置无用 φ 函数。
- **conventional SSA, C-SSA（常规 SSA）**：每个 φ-web 内部无冲突。
- **transformed SSA, T-SSA（变换后 SSA）**：优化后，同一 φ-web 中可能出现冲突。

---

## 章节导言

### English

Recall from the previous chapter that a procedure is in SSA form if every variable is defined only once, and every use of a variable refers to exactly one definition. Many variations, or flavours, of SSA form that satisfy these criteria can be defined, each offering its own considerations. For example, different flavours vary in terms of the number of φ-functions, which affects the size of the intermediate representation; some variations are more difficult to construct, maintain, and destruct than others. This chapter explores these SSA flavours and provides insights into their relative merits in certain contexts.

### 中文翻译

回顾上一章：如果一个过程中的每个变量都只定义一次，并且变量的每次使用都准确地指向唯一一个定义，那么这个过程就是 SSA 形式。可以定义出许多满足这些条件的 SSA 变体（flavours），而每种变体都有各自需要权衡的因素。例如，不同变体所需的 φ 函数数量不同，这会影响中间表示的大小；某些变体也比其他变体更难构造、维护和消除。本章将考察这些 SSA 变体，并说明它们在不同场景下各自的优缺点。

### 理解与补充

SSA 的两个基本条件只规定了“定义与使用如何对应”，没有规定：

1. φ 函数必须放多少个；
2. 未初始化变量如何表示；
3. 无用 φ 函数是否保留；
4. 优化以后，φ 连接起来的变量能否直接合并回一个变量。

这些额外性质正是本章各节的主题。

---

## 2.1 Def-Use and Use-Def Chains（定义—使用链与使用—定义链）

### English

Under SSA form, each variable is defined once. Def-use chains are data structures that provide, for the single definition of a variable, the set of all its uses. In turn, a use-def chain, which under SSA consists of a single name, uniquely specifies the definition that reaches the use. As we will illustrate further in the book (see Chap. 8), def-use chains are useful for forward data-flow analysis as they provide direct connections that shorten the propagation distance between nodes that generate and use data-flow information.

### 中文翻译

在 SSA 形式中，每个变量只定义一次。**定义—使用链（def-use chain）**是一种数据结构：对于变量的这个唯一定义，它记录该定义的全部使用位置。反过来，**使用—定义链（use-def chain）**在 SSA 中只包含一个名字，它唯一确定能够到达该使用位置的定义。正如本书后面还会进一步说明的那样（见第 8 章），def-use 链很适合正向数据流分析，因为它在产生数据流信息的节点与使用这些信息的节点之间建立了直接连接，从而缩短了传播距离。

### 理解与补充

可把两种链理解为两个方向的索引：

- `definition → all uses`：def-use 链，一对多；
- `use → its definition`：use-def 链，在 SSA 中是一对一。

在非 SSA 代码中，同一个名字 `x` 可能有多个定义，因此看到一次 `x` 的使用时，还要做 reaching-definitions（到达定义）分析才能知道它来自哪里。SSA 通过重命名把这个关系直接编码进变量名，例如 `x3` 的使用总是指向 `x3` 的唯一定义。

### English

Because of its single definition per variable property, SSA form simplifies def-use and use-def chains in several ways. First, SSA form simplifies def-use chains as it combines the information as early as possible. This is illustrated by Fig. 2.1 where the def-use chain in the non-SSA program requires as many merges as there are uses of x, whereas the corresponding SSA form allows early and more efficient combination.

### 中文翻译

由于每个变量只有一个定义，SSA 在多个方面简化了 def-use 链和 use-def 链。首先，SSA 会尽早合并信息，因此简化了 def-use 链。如图 2.1 所示，在非 SSA 程序中，对 `x` 的每个使用都需要分别合并可能到达它的定义；而对应的 SSA 程序通过 φ 函数提前完成合并，效率更高。

### 图 2.1 解读

左侧非 SSA 代码中，两条分支分别执行 `x ← 1` 和 `x ← 2`。后面 `y ← x + 1` 与 `z ← x + 2` 都使用 `x`，所以每个使用点都必须考虑两个可能的定义。

右侧 SSA 代码把两个定义分别命名为 `x1`、`x2`，并在汇合点执行：

```text
x3 ← φ(x1, x2)
```

后面的两个表达式只使用 `x3`。因此，控制流合并和数据流合并都集中在 φ 函数处完成。图中的虚线是 def-use 边。

### English

Second, as it is easy to associate each variable with its single defining operation, use-def chains can be represented and maintained almost for free. As this constitutes the skeleton of the so-called SSA graph (see Chap. 14), when considering a program under SSA form, use-def chains are implicitly considered as a given. The explicit representation of use-def chains simplifies backward propagation, which favours algorithms such as dead-code elimination.

### 中文翻译

其次，因为很容易把每个变量与其唯一的定义操作关联起来，所以 use-def 链几乎可以零成本地表示和维护。use-def 链构成所谓 SSA 图（见第 14 章）的骨架，因此在讨论 SSA 程序时，通常默认这种链已经存在。显式表示 use-def 链可以简化反向传播，因而有利于死代码消除等算法。

### 理解与补充

死代码消除从“这个结果是否被使用”向定义处回溯。use-def 链让算法不用扫描整个基本块或重新求 reaching definitions，而是可以从一次使用直接跳到它的定义。

### English

For forward propagation, since def-use chains are precisely the reverse of use-def chains, computing them is also easy; maintaining them requires minimal effort. However, even without def-use chains, some lightweight forward propagation algorithms such as copy folding¹ are possible: Using a single pass that processes operations along a topological order traversal of a forward CFG,² most definitions are processed prior to their uses. When processing an operation, the use-def chain provides immediate access to the prior computed value of an argument. Conservative merging is performed when, at some loop headers, a φ-function encounters an unprocessed argument. Such a lightweight propagation engine proves to be fairly efficient.

### 中文翻译

对于正向传播，def-use 链正好是 use-def 链的反向关系，因此也很容易计算，维护成本很低。不过，即使不显式维护 def-use 链，也可以实现某些轻量级正向传播算法，例如复制折叠¹：按照正向 CFG² 的拓扑序，用一次遍历处理各个操作，大多数定义都会先于其使用被处理。处理一个操作时，use-def 链可以立即给出某个实参先前算出的值。如果在某个循环头部，φ 函数遇到尚未处理的实参，就执行保守合并。这种轻量级传播引擎在实践中相当高效。

### 理解与补充

这里的关键是：删掉 CFG 的回边后，图变成 DAG，就可以按拓扑序处理。普通路径上通常是“先定义、后使用”；循环回边带来的 φ 实参可能尚未计算，此时算法不给出过度乐观的结论，而是采用保守值。

¹ **Copy folding（复制折叠）**：对复制 `a = b`，把后续由该定义支配的 `a` 的使用重命名为 `b`，再删除复制。它通常与 SSA 重命名阶段结合。

² **Forward CFG（正向控制流图）**：从 CFG 中删除回边后得到的无环简化图。

---

## 2.2 Minimality（最小性）

### English

SSA construction is a two-phase process: placement of φ-functions, followed by renaming. The goal of the first phase is to generate code that fulfils the single reaching-definition property, as already outlined. Minimality is an additional property relating to code that has φ-functions inserted, but prior to renaming; Chap. 3 describes the classical SSA construction algorithm in detail, while this section focuses primarily on describing the minimality property.

### 中文翻译

SSA 构造分为两个阶段：先放置 φ 函数，再进行重命名。第一阶段的目标，是生成满足**单一到达定义性质（single reaching-definition property）**的代码。最小性是 φ 函数已经插入、但变量尚未重命名时可以具有的一项附加性质。第 3 章会详细介绍经典 SSA 构造算法，本节主要说明“最小性”本身。

### English

A definition D of variable v reaches a point p in the CFG if there exists a path from D to p that does not pass through another definition of v. We say that a code has the single reaching-definition property iff no program point can be reached by two definitions of the same variable. Under the assumption that the single reaching-definition property is fulfilled, the minimality property states the minimality of the number of inserted φ-functions.

### 中文翻译

如果 CFG 中存在一条从变量 `v` 的定义 `D` 到程序点 `p` 的路径，并且这条路径不经过 `v` 的其他定义，那么称定义 `D` 能够**到达（reach）** `p`。当且仅当不存在任何程序点能同时被同一变量的两个定义到达时，代码满足单一到达定义性质。在已经满足该性质的前提下，最小性要求插入的 φ 函数数量最少。

### 理解与补充

“最小”不是指 SSA 名字最少，也不是指总代码量最少，而是：在保证每个程序点至多看见同一变量的一个到达定义时，φ 函数不能再少。

### English

This property can be characterized using the following notion of join sets. Let n₁ and n₂ be distinct basic blocks in a CFG. A basic block n₃, which may or may not be distinct from n₁ or n₂, is a join node of n₁ and n₂ if there exist at least two non-empty paths, i.e., paths containing at least one CFG edge, from n₁ to n₃ and from n₂ to n₃, respectively, such that n₃ is the only basic block that occurs on both of the paths. In other words, the two paths converge at n₃ and no other CFG node. Given a set S of basic blocks, n₃ is a join node of S if it is the join node of at least two basic blocks in S. The set of join nodes of set S is denoted by J(S).

### 中文翻译

可以用**汇合节点集合（join set）**刻画最小性。设 `n₁`、`n₂` 是 CFG 中两个不同的基本块。若存在两条非空路径（至少包含一条 CFG 边），分别从 `n₁`、`n₂` 到达基本块 `n₃`，并且 `n₃` 是两条路径中唯一共同出现的基本块，那么 `n₃` 就是 `n₁` 与 `n₂` 的汇合节点；`n₃` 可以与 `n₁` 或 `n₂` 相同，也可以不同。换句话说，两条路径在 `n₃` 汇合，此前没有共同的 CFG 节点。给定基本块集合 `S`，如果 `n₃` 是 `S` 中至少两个基本块的汇合节点，就称 `n₃` 是 `S` 的汇合节点。`S` 的所有汇合节点组成的集合记为 `J(S)`。

### English

Intuitively, a join set corresponds to the placement of φ-functions. In other words, if n₁ and n₂ are basic blocks that both contain a definition of variable v, then we ought to instantiate φ-functions for v at every basic block in J({n₁, n₂}). Generalizing this statement, if Dᵥ is the set of basic blocks containing definitions of v, then φ-functions should be instantiated in every basic block in J(Dᵥ). As inserted φ-functions are themselves definition points, some new φ-functions should be inserted at J(Dᵥ ∪ J(Dᵥ)). Actually, it turns out that J(S ∪ J(S)) = J(S), so the join set of the set of definition points of a variable in the original program characterizes exactly the minimum set of program points where φ-functions should be inserted.

### 中文翻译

直观上，汇合集合对应于 φ 函数的放置位置。如果基本块 `n₁` 和 `n₂` 都包含变量 `v` 的定义，那么应该在 `J({n₁, n₂})` 的每个基本块中为 `v` 放置 φ 函数。推广开来，若 `Dᵥ` 是所有包含 `v` 定义的基本块构成的集合，就应当在 `J(Dᵥ)` 中的每个基本块放置 φ 函数。由于新插入的 φ 函数本身也是定义点，看起来还可能需要在 `J(Dᵥ ∪ J(Dᵥ))` 中继续插入 φ 函数。实际上有 `J(S ∪ J(S)) = J(S)`，因此，原程序中某变量的定义点集合的汇合集合，恰好刻画了必须插入 φ 函数的最小程序点集合。

### 理解与补充

一句话概括：**两个定义的控制流第一次可能相遇之处，就是需要 φ 的地方。** φ 的结果又是一个定义，但 join 运算的闭包性质保证这里描述的集合已经足够。

### English

We are not aware of any optimizations that require a strict enforcement of minimality property. However, placing φ-functions only at the join set can be done easily using a simple topological traversal of the CFG as described in Chap. 4, Sect. 4.4. Classical techniques place φ-functions of a variable v at J(Dᵥ ∪ {r}), with r the entry node of the CFG. There are good reasons for that, as we will explain further. Finally, as explained in Chap. 3, Sect. 3.3 for reducible flow graphs, some copy-propagation engines can easily turn a non-minimal SSA code into a minimal one.

### 中文翻译

据作者所知，没有哪一种优化必须严格保证最小性。不过，如第 4 章 4.4 节所述，只需对 CFG 做一次简单的拓扑遍历，就能只在汇合集合中放置 φ 函数。经典方法会在 `J(Dᵥ ∪ {r})` 中为变量 `v` 放置 φ 函数，其中 `r` 是 CFG 的入口节点。这样做有充分理由，后文将进一步解释。最后，对于可规约流图，第 3 章 3.3 节会说明：某些复制传播引擎可以轻易把非最小 SSA 变成最小 SSA。

### 理解与补充

把入口 `r` 当成每个变量都有一个“未定义的伪定义”，可同时获得下一节的严格性/支配性质。由此可见，“φ 最少”不一定是唯一目标；编译器往往愿意为更强的结构性质多放少量 φ。

---

## 2.3 Strict SSA Form and Dominance Property（严格 SSA 与支配性质）

### English

A procedure is defined as strict if every variable is defined before it is used along every path from the entry to the exit point; otherwise, it is non-strict. Some languages, such as Java, impose strictness as part of the language definition; others, such as C/C++, impose no such restrictions. The code in Fig. 2.2a is non-strict as there exists a path from the entry to the use of a that does not go through the definition. If this path is taken through the CFG during the execution, then a will be used without ever being assigned a value. Although this may be permissible in some cases, it is usually indicative of a programmer error or poor software design.

### 中文翻译

如果从入口到出口的每一条路径上，每个变量都先定义、后使用，就称该过程是**严格的（strict）**；否则是非严格的。Java 等语言把严格性作为语言定义的一部分，而 C/C++ 等语言不施加这种限制。图 2.2a 的代码是非严格的，因为存在一条从入口到 `a` 的使用点、却不经过 `a` 定义的路径。如果执行时沿 CFG 走了这条路径，`a` 会在从未赋值的情况下被使用。某些情形下这也许被允许，但通常说明程序存在错误或设计不佳。

### English

Under SSA, because there is only a single (static) definition per variable, strictness is equivalent to the dominance property: Each use of a variable is dominated by its definition. In a CFG, basic block n₁ dominates basic block n₂ if every path in the CFG from the entry point to n₂ includes n₁. By convention, every basic block in a CFG dominates itself. Basic block n₁ strictly dominates n₂ if n₁ dominates n₂ and n₁ ≠ n₂. We use the symbols n₁ dom n₂ and n₁ sdom n₂ to denote dominance and strict dominance, respectively.

### 中文翻译

在 SSA 中，由于每个变量只有一个静态定义，所以严格性等价于**支配性质（dominance property）**：变量的每次使用都被它的定义支配。在 CFG 中，如果从入口到基本块 `n₂` 的每一条路径都经过基本块 `n₁`，就称 `n₁` **支配（dominate）** `n₂`。按约定，每个基本块都支配自身。如果 `n₁` 支配 `n₂` 且 `n₁ ≠ n₂`，就称 `n₁` **严格支配（strictly dominate）** `n₂`。分别记为 `n₁ dom n₂` 和 `n₁ sdom n₂`。

### 理解与补充

“定义支配使用”意味着：无论控制流如何选择，只要执行到了这个使用点，就一定已经执行过它的定义。因此，SSA 名字不会在某条路径上“凭空出现”。

### English

Adding a (undefined) pseudo-definition of each variable to the entry point (root) of the procedure ensures strictness. The single reaching-definition property discussed previously mandates that each program point be reachable by exactly one definition (or pseudo-definition) of each variable. If a program point U is a use of variable v, then the reaching definition D of v will dominate U; otherwise, there would be a path from the CFG entry node to U that does not include D. If such a path existed, then the program would not be in strict SSA form, and a φ-function would need to be inserted somewhere in J(r, D), as in our example of Fig. 2.2b where ⊥ represents the undefined pseudo-definition.

### 中文翻译

在过程入口（根节点）为每个变量添加一个“未定义”的伪定义，就能保证严格性。前述单一到达定义性质要求：对于每个变量，每个程序点都只能由一个定义（或伪定义）到达。若程序点 `U` 使用变量 `v`，那么 `v` 的到达定义 `D` 必须支配 `U`；否则，就会存在一条从 CFG 入口到 `U`、但不经过 `D` 的路径。若存在这种路径，程序便不是严格 SSA，需要在 `J(r, D)` 的某个位置插入 φ 函数。图 2.2b 就是一个例子，其中 `⊥` 表示未定义伪定义。

### 图 2.2 解读

原程序的两个独立条件分支形成“双菱形”：一条路径定义 `a`，另一条路径可能绕过该定义，最后却使用 `a`。严格 SSA 在入口引入 `a₀ ← ⊥`，并在汇合处放置：

```text
a1 ← φ(a0, ⊥)
```

于是使用 `a1` 的路径在结构上总有定义到达；但若运行时选中了 `⊥`，语义上仍然是在使用未初始化值。SSA 只是把这个事实显式表示出来，并没有修复源程序错误。

### English

The so-called minimal SSA form is a variant of SSA form that satisfies both the minimality and dominance properties. As shall be seen in Chap. 3, minimal SSA form is obtained by placing the φ-functions of variable v at J(Dᵥ, r) using the formalism of dominance frontier. If the original procedure is non-strict, conversion to minimal SSA will create a strict SSA-based representation. Here, strictness refers solely to the SSA representation; if the input program is non-strict, conversion to and from strict SSA form cannot address errors due to uninitialized variables. To finish with, the use of an implicit pseudo-definition in the CFG entry node to enforce strictness does not change the semantics of the program by any means.

### 中文翻译

所谓**最小 SSA 形式（minimal SSA form）**，是同时满足最小性与支配性质的 SSA 变体。第 3 章将说明：利用支配边界的形式化方法，在 `J(Dᵥ, r)` 中放置变量 `v` 的 φ 函数，就能得到最小 SSA。如果原过程是非严格的，转换到最小 SSA 后会得到一种严格的 SSA 表示。这里的“严格”只针对 SSA 表示本身；如果输入程序非严格，那么转换进、转换出严格 SSA 都不能解决未初始化变量造成的错误。最后，在 CFG 入口使用隐式伪定义来强制严格性，绝不会改变程序语义。

### English

SSA with dominance property is useful for many reasons that directly originate from the structural properties of the variable live ranges. The immediate dominator or “idom” of a node N is the unique node that strictly dominates N but does not strictly dominate any other node that strictly dominates N. All nodes but the entry node have immediate dominators. A dominator tree is a tree where the children of each node are those nodes it immediately dominates. Because the immediate dominator is unique, it is a tree with the entry node as root.

### 中文翻译

满足支配性质的 SSA 有许多用途，这些用途直接来自变量活跃区间的结构性质。节点 `N` 的**直接支配者（immediate dominator，idom）**是这样一个唯一节点：它严格支配 `N`，但不严格支配其他同样严格支配 `N` 的节点。除入口外，每个节点都有直接支配者。**支配树（dominator tree）**是一棵树，每个节点的孩子是它直接支配的节点。由于直接支配者唯一，因此它确实构成一棵以入口节点为根的树。

### English

For each variable, its live range, i.e., the set of program points where it is live, is a sub-tree of the dominator tree. Among other consequences of this property, we can cite the ability to design a fast and efficient method to query whether a variable is live at point q or an iteration-free algorithm to compute liveness sets (see Chap. 9). This property also allows efficient algorithms to test whether two variables interfere (see Chap. 21).

### 中文翻译

对于每个变量，它的**活跃区间（live range）**，也就是它处于活跃状态的程序点集合，是支配树的一棵子树。借助这一性质，可以设计快速方法来查询变量在程序点 `q` 是否活跃，也可以用无迭代算法计算活跃集合（见第 9 章）。这一性质还允许高效判断两个变量是否冲突（见第 21 章）。

### 理解与补充

一般数据流分析常需反复迭代直至不动点。严格 SSA 把活跃区间组织成支配树中的连通子树，许多问题因此可转化为树上的祖先、包含或相交查询。

### English

Usually, we suppose that two variables interfere if their live ranges intersect (see Sect. 2.6 for further discussions about this hypothesis). Note that in the general case, a variable is considered to be live at a program point if there exists a definition of that variable that can reach this point (reaching-definition analysis), and if there exists a definition-free path to a use (upward-exposed use analysis). For strict programs, any program point from which you can reach a use without going through a definition is necessarily reachable from a definition.

### 中文翻译

通常，如果两个变量的活跃区间相交，就认为它们发生冲突（这一假设会在 2.6 节进一步讨论）。一般来说，若变量的某个定义能到达某程序点（到达定义分析），并且从该点存在一条不经过其他定义而到达某次使用的路径（向上暴露使用分析），则认为该变量在此程序点活跃。对于严格程序，只要从某程序点出发可以不经过定义而到达一次使用，该点就必然也能由某个定义到达。

### English

Another elegant consequence is that the intersection graph of live ranges belongs to a special class of graphs called chordal graphs. Chordal graphs are significant because several problems that are NP-complete on general graphs have efficient linear-time solutions on chordal graphs, including graph colouring. Graph colouring plays an important role in register allocation, as the register assignment problem can be expressed as a colouring problem of the interference graph. In this graph, two variables are linked with an edge if they interfere, meaning they cannot be assigned the same physical location (usually, a machine register, or “colour”).

### 中文翻译

另一个很优美的结果是：活跃区间的相交图属于一种特殊图类——**弦图（chordal graph）**。弦图之所以重要，是因为包括图着色在内，一些在一般图上为 NP 完全的问题，在弦图上存在高效的线性时间解法。图着色对寄存器分配非常重要，因为寄存器指派可以表述为冲突图的着色问题：如果两个变量冲突，就在图中用一条边连接它们，表示二者不能分配到同一个物理位置——通常是同一个机器寄存器，也就是同一种“颜色”。

### English

The underlying chordal property highly simplifies the assignment problem otherwise considered NP-complete. In particular, a traversal of the dominator tree, i.e., a “tree scan,” can colour all of the variables in the program, without requiring the explicit construction of an interference graph. The tree scan algorithm can be used for register allocation, which is discussed in greater detail in Chap. 22.

### 中文翻译

底层的弦图性质大幅简化了原本被认为是 NP 完全的分配问题。特别是，只需遍历支配树，即进行一次“树扫描（tree scan）”，就可以给程序中的所有变量着色，而无需显式构造冲突图。树扫描算法可用于寄存器分配，第 22 章会详细讨论。

### English

As we have already mentioned, most φ-function placement algorithms are based on the notion of dominance frontier (see Chaps. 3 and 4) and consequently do provide the dominance property. As we will see in Chap. 3, this property can be broken by copy propagation: In our example of Fig. 2.2b, the argument a₁ of the copy represented by a₂ = φ(a₁, ⊥) can be propagated and every occurrence of a₂ can be safely replaced by a₁; the now identity φ-function can then be removed obtaining the initial code, that is, still SSA but not strict anymore.

### 中文翻译

如前所述，大多数 φ 函数放置算法都以支配边界为基础（见第 3、4 章），所以能够保证支配性质。但第 3 章会说明，复制传播可能破坏这一性质：在图 2.2b 的例子中，可以传播由 `a₂ = φ(a₁, ⊥)` 表示的复制中的实参 `a₁`，并安全地把每个 `a₂` 都替换为 `a₁`；随后可以删除退化为恒等复制的 φ 函数，得到最初的代码。此时代码仍是 SSA，却不再严格。

### English

Making a non-strict SSA code strict is about the same complexity as SSA construction (actually we need a pruned version as described below). Still, the “strictification” usually concerns only a few variables and a restricted region of the CFG: The incremental update described in Chap. 5 will do the work with less effort.

### 中文翻译

把非严格 SSA 重新变成严格 SSA，其复杂度大致与重新构造 SSA 相当（实际上需要下文所述的剪枝版本）。不过，“严格化”通常只涉及少数变量和 CFG 的有限区域，因此第 5 章所述的增量更新可以用更低成本完成。

---

## 2.4 Pruned SSA Form（剪枝 SSA）

### English

One drawback of minimal SSA form is that it may place φ-functions for a variable at a point in the control-flow graph where the variable was not actually live prior to SSA. Many program analyses and optimizations, including register allocation, are only concerned with the region of a program where a given variable is live. The primary advantage of eliminating those dead φ-functions over minimal SSA form is that it has far fewer φ-functions in most cases. It is possible to construct such a form while still maintaining the minimality and dominance properties otherwise. The new constraint is that every use point for a given variable must be reached by exactly one definition, as opposed to all program points. Pruned SSA form satisfies these properties.

### 中文翻译

最小 SSA 的一个缺点是：它可能会在 CFG 中某个位置为变量放置 φ 函数，尽管该变量在转换为 SSA 之前在那里根本不活跃。包括寄存器分配在内的许多程序分析和优化，只关心变量实际活跃的程序区域。与最小 SSA 相比，删除这些死 φ 函数的主要优点是：大多数情况下 φ 函数会少得多。同时仍然可以保留最小性和支配性质，只需把约束从“变量的每个程序点都恰好由一个定义到达”，改为“变量的每个**使用点**都恰好由一个定义到达”。**剪枝 SSA（pruned SSA）**满足这些性质。

### 理解与补充

最小 SSA 的“最小”是相对于“所有程序点都有单一到达定义”这个较强约束而言。剪枝 SSA 放松为只保障真正会使用该变量的点，因此还能进一步删除对程序结果毫无贡献的 φ。

### English

Under minimal SSA, φ-functions for variable v are placed at the entry points of basic blocks belonging to the set J(S, r). Under pruned SSA, we suppress the instantiation of a φ-function at the beginning of a basic block if v is not live at the entry point of that block. One possible way to do this is to perform liveness analysis prior to SSA construction, and then use the liveness information to suppress the placement of φ-functions as described above; another approach is to construct minimal SSA and then remove the dead φ-functions using dead-code elimination; details can be found in Chap. 3.

### 中文翻译

在最小 SSA 中，变量 `v` 的 φ 函数被放置在属于集合 `J(S, r)` 的基本块入口。剪枝 SSA 则规定：如果 `v` 在某基本块入口不活跃，就不在该入口实例化 φ 函数。一种做法是在构造 SSA 前先进行活跃性分析，再用所得信息抑制上述 φ 放置；另一种做法是先构造最小 SSA，再用死代码消除删除死 φ 函数。细节见第 3 章。

### English

Figure 2.3a shows an example of minimal non-pruned SSA. The corresponding pruned SSA form would remove the dead φ-function that defines Y₃ since Y₁ and Y₂ are only used in their respective definition blocks.

### 中文翻译

图 2.3a 展示了一个最小但未剪枝的 SSA。相应的剪枝 SSA 会删除定义 `Y₃` 的死 φ 函数，因为 `Y₁`、`Y₂` 只在各自的定义块中被使用。

### 图 2.3 解读

两个分支分别得到 `Y₁` 与 `Y₂`，汇合处构造 `Y₃ = φ(Y₁, Y₂)`；但后续真正参与计算的是 `Z₁`、`Z₂` 以及 `Z₃`。因此 `Y₃` 没有普通使用，是死定义。剪枝 SSA 会连同这个 φ 及其仅为该 φ 服务的相关活跃性一起消去。

图中还说明一个细节：未剪枝形式可能给全局值编号提供额外的“值相等”线索，使其发现 `Y₃` 和 `Z₃` 具有相同值。不过这种潜在分析便利通常不足以抵消大量死 φ 的成本。

---

## 2.5 Conventional and Transformed SSA Form（常规 SSA 与变换后 SSA）

### English

In many non-SSA and graph colouring based register allocation schemes, register assignment is done at the granularity of webs. In this context, a web is the maximum union of def-use chains that have either a use or a definition in common. As an example, the code in Fig. 2.4a leads to two separate webs for variable a. The conversion to minimal SSA form replaces each web of a variable v in the pre-SSA program with some variable names vᵢ. In pruned SSA, these variable names partition the live range of the web: At every point in the procedure where the web is live, exactly one variable vᵢ is also live; and none of the vᵢ is live at any point where the web is not.

### 中文翻译

在许多非 SSA、基于图着色的寄存器分配方案中，寄存器以 **web** 为粒度分配。这里的 web 是这样一种最大 def-use 链并集：其中的链共享某个使用或某个定义。图 2.4a 的代码会为变量 `a` 形成两个彼此分离的 web。转换到最小 SSA 时，SSA 之前变量 `v` 的每个 web 会被若干变量名 `vᵢ` 取代。在剪枝 SSA 中，这些名字会划分原 web 的活跃区间：在过程内原 web 活跃的每个位置，恰好有一个 `vᵢ` 活跃；原 web 不活跃之处，任何 `vᵢ` 也都不活跃。

### 理解与补充

web 可以粗略理解为“通过定义—使用关系连成的一整团同名值”。SSA 重命名把这团值拆成多个版本，但这些版本合起来仍对应转换前的一个 web。

### English

Based on this observation, we can partition the variables in a program that has been converted to SSA form into φ-equivalence classes that we will refer as φ-webs. We say that x and y are φ-related to one another if they are referenced by the same φ-function, i.e., if x and y are either parameters or defined by the φ-function. The transitive closure of this relation defines an equivalence relation that partitions the variables defined locally in the procedure into equivalence classes, the φ-webs. Intuitively, the φ-equivalence class of a resource represents a set of resources “connected” via φ-functions. For any freshly constructed SSA code, the φ-webs exactly correspond to the register webs of the original non-SSA code.

### 中文翻译

基于这一观察，可以把 SSA 程序中的变量划分为若干 **φ 等价类**，本文称之为 **φ-web**。如果 `x` 和 `y` 被同一个 φ 函数引用——也就是二者是该 φ 的参数或结果——就说 `x` 与 `y` 具有 φ 关系。对这种关系取传递闭包，会得到一种等价关系，把过程中局部定义的变量划分为多个等价类，即 φ-web。直观上，一个资源的 φ 等价类，就是经由 φ 函数“连接”起来的一组资源。对于刚刚构造完成、尚未经过优化的 SSA 代码，φ-web 与原非 SSA 代码的寄存器 web 精确对应。

### 示例

若有：

```text
a4 ← φ(a2, a3)
```

则 `a2`、`a3`、`a4` 两两经由同一个 φ 相连，属于同一 φ-web。若另一个 φ 又连接 `a4` 与 `a5`，传递闭包会把 `a5` 也放入同一类。

### English

Conventional SSA form (C-SSA) is defined as SSA form for which each φ-web is interference-free. Many program optimizations such as copy propagation may transform a procedure from conventional to a non-conventional (T-SSA for Transformed SSA) form, in which some variables belonging to the same φ-web interfere with one another. Figure 2.4c shows the corresponding transformed SSA form of our previous example: Here variable a₁ interferes with variables a₂, a₃, and a₄, since it is defined at the top and used last.

### 中文翻译

**常规 SSA（Conventional SSA，C-SSA）**是指每个 φ-web 内都不存在变量冲突的 SSA。复制传播等许多程序优化可能会把过程从常规形式变成非常规形式，即 **变换后 SSA（Transformed SSA，T-SSA）**；在 T-SSA 中，同一 φ-web 的某些变量会彼此冲突。图 2.4c 展示了前例对应的 T-SSA：变量 `a₁` 在顶部定义、最后才使用，所以它与 `a₂`、`a₃`、`a₄` 都发生冲突。

### 图 2.4 解读

- 图 (a)：源程序中名为 `a` 的定义—使用关系形成两个 register web。
- 图 (b)：转换为 C-SSA 后，对应两个 φ-web：`{a₁}` 与 `{a₂, a₃, a₄}`。后一组由 `a₄ = φ(a₂, a₃)` 连起来，且组内活跃区间互不重叠。
- 图 (c)：传播 `a₁` 后，`a₁` 的活跃区间从顶端一直延伸到末端；它与 `a₂/a₃/a₄` 重叠。虽然仍满足“每个 SSA 名字只定义一次”，却不再满足 C-SSA 的 φ-web 内无冲突条件。

### English

Bringing back the conventional property of a T-SSA code is as “difficult” as translating out of SSA (also known as SSA “destruction,” see Chap. 3). Indeed, the destruction of conventional SSA form is straightforward: Each φ-web can be replaced with a single variable; all definitions and uses are renamed to use the new variable, and all φ-functions involving this equivalence class are removed.

### 中文翻译

让 T-SSA 重新满足常规性质，与把代码转换出 SSA 一样“困难”；后一过程也称 SSA **消除（destruction）**，见第 3 章。事实上，消除 C-SSA 非常直接：把每个 φ-web 替换成一个变量，把所有定义和使用都重命名为这个新变量，再删除涉及该等价类的全部 φ 函数即可。

### 理解与补充

之所以可以直接合并，是因为 C-SSA 保证同一 φ-web 中的变量不冲突。它们的活跃期不需要同时占据存储位置，所以可以安全地共享一个名字/位置。

### English

SSA destruction starting from non-conventional SSA form can be performed through a conversion to conventional SSA form as an intermediate step. This conversion is achieved by inserting copy operations that dissociate interfering variables from the connecting φ-functions. As those copy instructions will have to be inserted at some points to get rid of φ-functions, for machine-level transformations such as register allocation or scheduling, T-SSA provides an inaccurate view of the resource usage.

### 中文翻译

从非常规 SSA 开始做 SSA 消除时，可以先把它转换为常规 SSA。做法是插入复制操作，把发生冲突的变量与连接它们的 φ 函数分离。由于为了消除 φ 函数，最终必须在某些位置插入这些复制指令，所以对于寄存器分配、指令调度等机器级变换而言，T-SSA 对资源使用情况给出的视图并不准确。

### English

Another motivation for sticking to C-SSA is that the names used in the original program might help capture some properties otherwise difficult to discover. Lexical partial redundancy elimination (PRE) as described in Chap. 11 illustrates this point. Apart from those specific examples most current compilers choose not to maintain the conventional property. Still, we should outline that, as later described in Chap. 21, checking if a given φ-web is (and if necessary turning it back to) interference-free can be done in linear time (instead of the naive quadratic time algorithm) in the size of the φ-web.

### 中文翻译

坚持使用 C-SSA 的另一个理由是：原程序使用的变量名可能有助于捕捉某些原本很难发现的性质。第 11 章的词法部分冗余消除（lexical PRE）就说明了这一点。除这些特定例子外，当代多数编译器并不会持续维护常规性质。不过需要指出的是，如第 21 章所述，判断给定 φ-web 是否无冲突，并在必要时恢复无冲突性质，可以按 φ-web 大小在线性时间内完成，而不需要朴素的二次时间算法。

---

## 2.6 A Stronger Definition of Interference（更强/更精细的冲突定义）

### English

Throughout this chapter, two variables have been said to interfere if their live ranges intersect. Intuitively, two variables with overlapping lifetimes will require two distinct storage locations; otherwise, a write to one variable will overwrite the value of the other. In particular, this definition has applied to the discussion of interference graphs and the definition of conventional SSA form, as described above.

### 中文翻译

本章此前一直把冲突定义为：两个变量的活跃区间相交。直观上，生存期重叠的两个变量需要两个不同的存储位置；否则，写入其中一个变量会覆盖另一个变量的值。上述冲突图和常规 SSA 的定义都采用了这个标准。

### English

Although it suffices for correctness, this is a fairly restrictive definition of interference, based on static considerations. Consider for instance the case when two simultaneously live variables in fact contain the same value, then it would not be a problem to put both of them in the same register. The ultimate notion of interference, which is obviously undecidable because of a reduction to the halting problem, should decide for two distinct variables whether there exists an execution for which they simultaneously hold two different values. Several “static” extensions to our simple definition are still possible, in which, under very specific conditions, variables whose live ranges overlap one another may not interfere. We present two examples.

### 中文翻译

这种基于静态因素的冲突定义足以保证正确性，但相当保守。例如，两个变量虽然同时活跃，实际却含有相同的值，那么把它们放在同一个寄存器中并没有问题。最理想的冲突判定应该回答：对于两个不同变量，是否存在某次执行，使它们在同一时刻持有不同值。由于可以把停机问题归约到该问题，这种终极判定显然不可判定。不过，我们仍可以对简单定义做若干静态扩展：在非常具体的条件下，活跃区间重叠的变量也可能不冲突。下面给出两个例子。

### 第一种放宽：路径互斥

### English

Firstly, consider the double-diamond graph of Fig. 2.2a again, which, although non-strict, is correct as soon as the two if conditions are the same. Even if a and b are unique variables with overlapping live ranges, the paths along which a and b are respectively used and defined are mutually exclusive with one another. In this case, the program will either pass through the definition of a and the use of a, or the definition of b and the use of b, since all statements involved are controlled by the same condition, albeit at different conditional statements in the program. Since only one of the two paths will ever execute, it suffices to allocate a single storage location that can be used for a or b. Thus, a and b do not actually interfere with one another.

### 中文翻译

首先，再看图 2.2a 的双菱形图。虽然它并不严格，但只要两个 `if` 的条件相同，程序就是正确的。即使 `a`、`b` 是活跃区间重叠的不同变量，分别定义和使用它们的路径也彼此互斥。由于相关语句都受同一条件控制——尽管位于程序中两个不同的条件语句处——程序要么经过 `a` 的定义与使用，要么经过 `b` 的定义与使用。两条路径中只会执行一条，所以只分配一个供 `a` 或 `b` 使用的存储位置就够了。因此，`a` 与 `b` 实际上并不冲突。

### English

A simple way to refine the interference test is to check if one of the variables is live at the definition point of the other. This relaxed but correct notion of interference would not make a and b in Fig. 2.2a interfere while variables a₁ and b₁ of Fig. 2.2b would still interfere. This example illustrates the fact that live range splitting required here to make the code fulfil the dominance property may lead to less accurate analysis results. As far as the interference is concerned, for a SSA code with dominance property, the two notions are strictly equivalent: Two live ranges intersect iff one contains the definition of the other.

### 中文翻译

一种简单的精细化方法是：检查其中一个变量在另一个变量的定义点是否活跃。这种放宽后的冲突定义仍然正确；它不会认为图 2.2a 中的 `a`、`b` 冲突，但仍会认为图 2.2b 中的 `a₁`、`b₁` 冲突。这个例子说明，为使代码满足支配性质而进行的活跃区间分裂，可能导致分析结果不够精确。就冲突而言，对于满足支配性质的 SSA 代码，两种定义严格等价：两个活跃区间相交，当且仅当其中一个包含另一个变量的定义点。

### 第二种放宽：值相等

### English

Secondly, consider two variables u and v, whose live ranges overlap. If we can prove that u and v will always hold the same value at every place where both are live, then they do not actually interfere with one another. Since they always have the same value, a single storage location can be allocated for both variables, because there is only one unique value between them. Of course, this new criterion is in general undecidable. Still, a technique such as global value numbering that is straightforward to implement under SSA (see Chap. 11.5.1) can do a fairly good job, especially in the presence of a code with many variable-to-variable copies, such as one obtained after a naive SSA destruction pass (see Chap. 3). In that case (see Chap. 21), the difference between the refined notion of interference and the non-value-based one is significant.

### 中文翻译

其次，考虑活跃区间重叠的变量 `u`、`v`。如果能够证明：在两者同时活跃的每个位置，它们始终持有相同的值，那么它们实际并不冲突。因为两者之间只有一个唯一值，所以可共享一个存储位置。当然，这个新判据一般也是不可判定的。不过，全局值编号等技术在 SSA 中很容易实现（见 11.5.1 节），而且效果相当好；当代码中存在大量变量到变量的复制时尤其如此，例如朴素 SSA 消除后产生的代码（见第 3 章）。在这种情况下（见第 21 章），精细冲突定义与不考虑值的定义会产生显著差异。

### English

This refined notion of interference has significant implications if applied to SSA form. In particular, the interference graph of a procedure is no longer chordal, as any edge between two variables whose lifetimes overlap could be eliminated by this property.

### 中文翻译

把这种精细冲突定义用于 SSA 会产生重大影响。尤其是，过程的冲突图将不再保证为弦图，因为两个生存期重叠的变量之间原本存在的边，都可能依据“值始终相同”等性质被删除。

### 理解与补充

这里存在一个重要取舍：

- 保守定义“活跃区间相交即冲突”更容易分析，并保留弦图结构，可使用线性时间着色；
- 精细定义能让更多变量共享寄存器，但删边后可能破坏弦图，失去原有的算法优势。

更精确不必然意味着整体更便宜。

---

## 2.7 Further Reading（延伸阅读）

### English

The advantages of def-use and use-def chains provided almost for free under SSA are well illustrated in Chaps. 8 and 13.

### 中文翻译

第 8 章和第 13 章充分展示了 SSA 几乎零成本提供 def-use 链和 use-def 链所带来的优势。

### English

The notion of minimal SSA and a corresponding efficient algorithm to compute it were introduced by Cytron et al. [90]. For this purpose they extensively develop the notion of dominance frontier of a node n, DF(n) = J(n, r). The fact that J⁺(S) = J(S) was actually discovered later, with a simple proof by Wolfe [307]. More details about the theory on (iterated) dominance frontier can be found in Chaps. 3 and 4. The post-dominance frontier, which is its symmetric notion, also known as the control dependence graph, finds many applications. Further discussions on control dependence graph can be found in Chap. 14.

### 中文翻译

最小 SSA 的概念及其高效构造算法由 Cytron 等人 [90] 提出。为此，他们系统发展了节点 `n` 的支配边界概念：`DF(n) = J(n, r)`。`J⁺(S) = J(S)` 这一事实则由 Wolfe [307] 稍后发现，并给出了一个简单证明。关于（迭代）支配边界的更多理论见第 3、4 章。与其对称的概念是后支配边界，也称控制依赖图，具有许多用途；更多讨论见第 14 章。

### English

Most SSA papers implicitly consider the SSA form to fulfil the dominance property. The first technique that really exploits the structural properties of the strictness is the fast SSA destruction algorithm developed by Budimlić et al. [53] and revisited in Chap. 21.

### 中文翻译

大多数 SSA 论文都默认 SSA 形式满足支配性质。第一个真正利用严格性结构特征的技术，是 Budimlić 等人 [53] 提出的快速 SSA 消除算法；第 21 章会重新讨论它。

### English

The notion of pruned SSA has been introduced by Choi, Cytron and Ferrante [67]. The example of Fig. 2.3 to illustrate the difference between pruned and non-pruned SSA has been borrowed from Cytron et al. [90]. The notions of conventional and transformed SSA were introduced by Sreedhar et al. in their seminal paper [267] for destructing SSA form. The description of the existing techniques to turn a general SSA into either a minimal, a pruned, a conventional, or a strict SSA is provided in Chap. 3.

### 中文翻译

剪枝 SSA 的概念由 Choi、Cytron 和 Ferrante [67] 提出。用于说明剪枝与未剪枝 SSA 差异的图 2.3，取自 Cytron 等人 [90]。常规 SSA 与变换后 SSA 的概念，由 Sreedhar 等人在关于 SSA 消除的奠基性论文 [267] 中提出。把一般 SSA 转换为最小、剪枝、常规或严格 SSA 的现有技术，会在第 3 章介绍。

### English

The ultimate notion of interference was first discussed by Chaitin in his seminal paper [60] that presents the graph colouring approach for register allocation. His interference test is similar to the refined test presented in this chapter. In the context of SSA destruction, Chap. 21 addresses the issue of taking advantage of the dominance property with this refined notion of interference.

### 中文翻译

Chaitin 在提出寄存器分配图着色方法的奠基性论文 [60] 中，首次讨论了终极意义上的冲突概念。他的冲突测试与本章提出的精细测试相似。在 SSA 消除的背景下，第 21 章会研究如何结合这种精细冲突定义来利用支配性质。

---

## 全章总结：五个性质之间的关系

| 形式/性质 | 核心要求 | 主要收益 | 主要代价或风险 |
|---|---|---|---|
| 基本 SSA | 每个名字定义一次；每次使用唯一对应定义 | use-def 关系直接、数据流分析方便 | 并不自动保证严格、精简或易消除 |
| Minimal SSA | 在既定约束下 φ 数量最少 | IR 较小 | 仍可能含对不活跃变量无用的 φ |
| Strict SSA | 定义支配每次使用 | 活跃区间成为支配树子树；利于分析和分配 | 优化可能破坏；严格化有维护成本 |
| Pruned SSA | 只在变量活跃处保留所需 φ | 通常显著减少死 φ | 构造前需活跃性信息，或构造后做 DCE |
| C-SSA | 每个 φ-web 内无冲突 | SSA 消除简单；资源关系准确 | 优化时持续维护较麻烦 |
| T-SSA | 允许 φ-web 内冲突 | 优化自由度高，多数现代编译器采用 | 消除 SSA 前可能要插入复制并恢复常规性 |

可以把最常用的工程组合记成：

> **严格 + 剪枝 SSA**：定义支配使用，同时不保留无用 φ；这是许多优化阶段喜欢的形态。经过复制传播等优化后，它可能变成 T-SSA；在寄存器分配或退出 SSA 前，再解决 φ-web 冲突与并行复制问题。

## 三个容易混淆的点

1. **最小 SSA 不等于剪枝 SSA。** 最小性是在较强的“所有程序点”约束下减少 φ；剪枝只保障使用点，因此还能删除死 φ。
2. **SSA 不等于严格 SSA。** “每个名字定义一次”并不保证定义支配使用；优化也可能把严格 SSA 变回非严格 SSA。
3. **活跃区间相交不一定真的冲突。** 路径互斥或值恒等时可以共享位置；但采用更精细的定义会破坏弦图等便利结构。
