---
layout: post
title:  "F-algebra 学习笔记"
date:   2019-5-13 00:00:00 +0000
categories:
  - post
tags:
  - Haskell
  - F-algebra
---

## 主要参考

[University of Illinois - CS522 - Programming Language Semantics (Fall 2018) - Conventional Semantic Approaches](http://fsl.cs.illinois.edu/images/c/ca/CS522-Fall-2018-basic-semantics.pdf)

[Cornell University - CS6110 - Advanced Programming Languages (Spring 2019) - Partial Orders & Continuity](http://www.cs.cornell.edu/courses/cs6110/2019sp/lectures/lec19.pdf)

## 不动点

### 偏序集

一个**偏序集（partial order set）** {$(D, \sqsubseteq)$} 由集合 {$D$} 和一个在 {$D$} 上定义的二元关系 {$\sqsubseteq$} 组成，满足：

* **自反性（reflexivity）**：对于任意 {$x \in D$}，{$x \sqsubseteq x$}
* **传递性（transitivity）**：对于任意 {$x, y , z \in D$}，若 {$x \sqsubseteq y$} 且 {$y \sqsubseteq z$}，则 {$x \sqsubseteq z$}
* **反对称性（antisymmetry）**：对于任意 {$x, y \in D$}，若 {$x \sqsubseteq y$} 且 {$y \sqsubseteq x$}，则 {$x = y$}

一个偏序集 {$(D, \sqsubseteq)$} 如果还满足**完全性**，即对于任意 {$x, y \in D$}，{$x \sqsubseteq y$} 或 {$y \sqsubseteq x$}，那么它就被称为一个**全序集（total order set）**。如果集合 {$D$} 上的关系 {$\prec$} 只满足自反性和传递性，那么 {$(D, \prec)$} 就被称为一个**预序集（preorder set）**。如果集合 {$D$} 上的关系 {$\sim$} 除了满足自反性和传递性，还满足**对称性（symmetry）**，即对于任意 {$x, y \in D$}，若 {$x \sim y$}，则 {$y \sim x$}，那么 {$\sim$} 就被称作一个**等价关系（equivalence relation）**，{$(D, \sim)$} 则被称为一个**Setoid**；对于任意 {$x \in D$}，{$x$} 的**等价类（equivalence class）**定义为 {$\\{a \in D | a \sim x\\}$}，记作 {$[x]\_\sim$}。

例子：

* {$(Nat, \leq)$}：自然数集和“小于或等于”构成一个全序集。

* {$(Nat \cup \\{\infty\\}, \leq)$}：加入无穷大的自然数集和“小于或等于”构成一个全序集。

* {$(S, =)$}：平坦集合 {$S$} 和恒等关系（identity）构成一个偏序集。任何两个不同的元素在这个偏序集中都不可比较。像这样序关系只包含了自反序对 {$(x, x)$} 的偏序被称作**离散偏序（discrete partial order）**。

* {$(S \cup \\{\perp\\}, \leq^S_\perp)$}：扩充了一个**底（bottom）**元素的集合 {$S$} 和二元关系 {$\leq^S_\perp$} 构成一个偏序集。对于任意 {$a, b \in S \cup \\{\perp\\}$}，当且仅当 {$a = b$} 或 {$a = \perp$} 时，{$a \leq^S_\perp b$}。在指称语义中，像这样用 {$\perp$} 扩充集合很常见，其中 {$\perp$} 用于表示**未定义值（undefined）**。{$S \cup \\{\perp\\}$} 可简写为 {$S_\perp$}，称作**基本域（primitive domain）**或**平坦域（flat domain）**。

* {$(A \rightharpoondown B, \preceq)$}：从 {$A$} 到 {$B$} 的偏函数（partial function）构成的集合和信息量关系（informativeness relation）{$\preceq$} 构成一个偏序集。对于偏函数 {$f, g : A \rightharpoondown B$}，我们称 {$f$} 的信息量少于或等于 {$g$}，当且仅当对于任意 {$a \in A$}，要么 {$f(a)$} 未定义，要么 {$f(a)$} 和 {$g(a)$} 均有定义且 {$f(a) = g(a)$}。

偏序集的合成：

* 如果 {$\\{(S_i, \leq_i)\\}\_{i \in I}$} 是一族偏序集，其中对于任意 {$i \neq j \in I$}，{$S_i \cap S_j = \emptyset$}，那么 {$\bigcup_{i \in I}(S_i, \leq_i) = (\bigcup_{i \in I}S_i, \bigcup_{i \in I}\leq_i)$} 也是偏序集，其中 {$\bigcup_{i \in I}\leq_i$} 定义为 {$a (\bigcup_{i \in I}\leq_i) b$} 当且仅当存在 {$i \in I$} 使得 {$a, b \in S_i$} 且 {$a \leq_i b$}。

* 如果 {$\\{(S_i, \leq_i)\\}\_{i \in I}$} 是一族偏序集，那么 {$\prod_{i \in I}(S_i, \leq_i) = (\prod_{i \in I}S_i, \prod_{i \in I}\leq_i)$} 也是偏序集，其中 {$\prod_{i \in I}\leq_i$} 定义为 {$\\{a_i\\}\_{i \in I} (\prod_{i \in I}\leq_i) \\{b_i\\}_{i \in I}$} 当且仅当对于所有 {$i \in I$}，{$a_i \leq_i b_i$}。

对于预序集 {$(D, \prec)$}，可以定义等价关系 {$\sim$} 为对于任意 {$a, b \in D$}，{$a \sim b$} 当且仅当 {$a \prec b$} 且 {$b \prec a$}。利用这个等价关系，我们能在 {$D$} 关于 {$\sim$} 的**商集（quotient set）** {$D / \sim$}，也就是关于 {$\sim$} 的所有等价类的集合，之上构造偏序 {$\prec^\star$}，定义为对于任意 {$a, b \in D$}，{$[a]\_\sim \prec^\star [b]\_\sim$} 当且仅当 {$a \prec b$}。

有时候，偏序集可以用**哈斯图（Hasse diagram）**来图形化描述。在哈斯图中，偏序集的每个元素都被绘制为一个（可能带标签的）点，点之间的连接线的绘制遵循以下规则：

* 若 {$x$} 和 {$y$} 是偏序集的元素且 {$x \sqsubseteq y$}，那么对应 {$x$} 的点画在对应 {$y$} 的点下方。

* 当且仅当 {$x \sqsubseteq y$} 且在偏序集中不存在一个元素 {$z$} 严格处于 {$x$} 和 {$y$} 之间，也就是 {$x$} 和 {$y$} 之间的序关系不是由于传递性时，才能在代表 {$x$} 和 {$y$} 的点之间连线。

对于偏序集 {$(D, \sqsubseteq)$} 和 {$X \subseteq D$}，元素 {$p \in D$} 被称作 {$X$} 的**上界（upper bound）**，当且仅当对于任意 {$x \in X$}，{$x \sqsubseteq p$}。以及，当且仅当 {$p$} 是一个上界，且对于 {$X$} 的其它任意上界 {$q$}，都有 {$p \sqsubseteq q$} 时，{$p$} 被称为**最小上界（least upper bound, LUB）**或**上确界（supremum）**或**并（join）**，写作 {$\sqcup X$}。

上界和最小上界并不一定总是存在。比如，若 {$D = X = \\{x,y\\}$} 且 {$\sqsubseteq$} 是恒等关系，那么 {$X$} 没有上界。即使上界存在，最小上界也不一定存在。比如，若 {$D = \\{a, b, c, d, e\\}$}，{$\sqsubseteq$} 定义为 {$a \sqsubseteq c$}，{$a \sqsubseteq d$}，{$b \sqsubseteq c$}，{$b \sqsubseteq d$}，{$c \sqsubseteq e$}，{$d \sqsubseteq e$}（如下图），那么 {$D$} 的任意子集都存在上界，但集合 {$\\{a, b\\}$} 没有最小上界（因为 {$c$}，{$d$} 和 {$e$} 均为 {$\\{a, b\\}$} 的上界，且 {$c \sqsubseteq e$}, {$d \sqsubseteq e$}，但是 {$c$} 和 {$d$} 之间不存在序关系）。最后，由于反对称性，最小上界存在即唯一。

![HasseDiagram]({{ "/assets/F-algebra/Hasse.svg" | absolute_url }})

我们可以用相似的手段对偶地定义 {$X \subseteq D$} 的**最大下界（greatest lower bound, GLB）**或**下确界（infimum）**或**交（meet）**，写作 {$\sqcap X$}。

对于偏序集 {$(D, \sqsubseteq)$}，如果其中任意一对元素构成的子集都存在最小上界，那么 {$(D, \sqsubseteq)$} 被称为一个**并半格（join-semilattice）**。我们可以对偶地定义 **交半格（meet-semilattice）**。如果 {$(D, \sqsubseteq)$} 同时是并半格和交半格，我们称其为**格（lattice）**。如果一个格的所有子集都存在最小上界和最小下界，我们称其为**完全格（complete lattice）**。

### 完全偏序

对于偏序集 {$(D, \sqsubseteq)$}，{$D$} 中的一条**链（chain）**被定义为 {$D$} 中的元素组成的一个无限序列 {$d_0 \sqsubseteq d_1 \sqsubseteq d_2 \sqsubseteq \dotsb \sqsubseteq d_n \sqsubseteq \dotsb$}，也可使用集合记法写作 {$\\{d_n \| n \in Nat\\}$}。如果存在 {$n \in Nat$} 使得对于所有的 {$m \geq n$} 都有 {$d_m = d_{m+1}$}，那么这条链就被称为是**稳定的（stationary）**。

如果 {$D$} 是有限的，那么任意链都是稳定的。更一般地，如果对于某个已知的 {$x \in D$}，只有有限个 {$y \in D$} 满足 {$x \sqsubseteq y$}，那么任何包含 {$x$} 的链都是稳定的。

如果 {$\\{d_n \| n \in Nat\\}$} 是 {$(D, \sqsubseteq)$} 中的一条链且有最小上界，我们将其最小上界写作 {$\bigsqcup_{n \in Nat} d_n$} 或者简写为 {$\sqcup d_n$}。

当且仅当偏序集 {$(D, \sqsubseteq)$} 中的任意链都有最小上界时，{$(D, \sqsubseteq)$} 被称作一个**完全偏序（complete partial order, CPO）**。

严格地说，此处的完全偏序采用了**链完全偏序（chain-complete partial order)**的定义。_TODO_ 我们就得到了{$\omega$}**-完全偏序**的定义。

当且仅当 {$(D, \sqsubseteq)$} 包含一个最小元素时，它被称作**含底的（with bottom, bottomed）**。这个最小元素一般写作 {$\perp$}，含底偏序集则写作 {$(D, \sqsubseteq, \perp)$}。

例子：

* {$(\mathcal{P}(S), \subseteq, \emptyset)$} 是含底完全偏序。

* {$(Nat, \leq)$} 以 {$0$} 为底，但是不完全（链 {$0 \leq 1 \leq 2 \leq \dotsb \leq n \leq \dotsb$} 没有上界）。

* {$(Nat, \geq)$} 是完全偏序但不含底。

* {$(Nat \cup \\{\infty\\}, \leq, 0)$} 是含底完全偏序。其中的任何链要么稳定，要么以 {$\infty$} 为最小上界。

* {$(S, =)$} 总是完全偏序。如果 {$S$} 仅有一个元素则含底。

* {$(S \cup \\{\perp\\}, \leq^S_\perp, \perp)$} 是含底完全偏序，经常像 {$(S \cup \\{\perp\\}, \leq^S_\perp)$} 一样被简写为 {$S_\perp$}。在指称语义中，有趣而重要的一点是，基本域 {$S_\perp$} 等价于偏函数集 {$\\{\*\\} \rightharpoondown S$}，其中 {$\\{\*\\}$} 表示一个单元素集合。

* {$(A \rightharpoondown B, \preceq, \perp)$} 是含底完全偏序。其底 {$\perp : A \rightharpoondown B$} 为对每个 {$A$} 的元素都未定义的函数。

含底完全偏序的合成：

* 如果 {$\\{(S_i, \leq_i, \perp)\\}\_{i \in I}$} 是一族底相同的含底完全偏序，即对于任何 {$i \neq j \in I$}，{$S_i \cap S_j = \\{\perp\\}$}，那么 {$\bigcup_{i \in I}(S_i, \leq_i, \perp)$} 也是含底完全偏序。

* 如果 {$\\{(S_i, \leq_i, \perp)\\}\_{i \in I}$} 是一族含底完全偏序，那么 {$\prod_{i \in I}(S_i, \leq_i)$} 也是含底完全偏序。

除非特殊说明，下文提及的完全偏序均默认为含底完全偏序。

### 单调和连续函数

如果 {$(D, \sqsubseteq)$} 和 {$(D', \sqsubseteq')$} 均为偏序集且 {$\mathcal{F} : D \to D'$} 是一个函数，那么当且仅当对于任何满足 {$x \sqsubseteq y$} 的 {$x, y \in D$}，都有 {$\mathcal{F}(x) \sqsubseteq' \mathcal{F}(y)$} 时，{$\mathcal{F}$} 被称作是**单调的（monotone）**。如果 {$\mathcal{F}$} 是单调的，我们可以直接写 {$\mathcal{F} : (D, \sqsubseteq) \to (D', \sqsubseteq')$}。{$Mon((D, \sqsubseteq), (D', \sqsubseteq'))$} 表示从 {$(D, \sqsubseteq)$} 到 {$(D', \sqsubseteq')$} 的单调函数的集合。

单调函数**保持**链，也就是说，只要 {$\\{d_n \| n \in Nat\\}$} 是 {$(D, \sqsubseteq)$} 中的链，{$\\{\mathcal{F}(d_n) \| n \in Nat\\}$} 就一定是 {$(D', \sqsubseteq')$} 中的链。

如果 {$(D, \sqsubseteq)$} 和 {$(D', \sqsubseteq')$} 均为完全偏序，对于 {$(D, \sqsubseteq)$} 中的任何链 {$\\{d_n \| n \in Nat\\}$} 都有 {$\sqcup \mathcal{F}(d_n) \sqsubseteq' \mathcal{F}(\sqcup d_n)$}。论证如下：根据最小上界的性质，对于任何 {$n \in Nat$}，{$d_n \sqsubseteq \sqcup d_n$}，又因为 {$\mathcal{F}$} 是单调的，所以对于任何 {$n \in Nat$}，都有 {$\mathcal{F}(d_n) \sqsubseteq' \mathcal{F}(\sqcup d_n)$}，或者说，{$\mathcal{F}(\sqcup d_n)$} 是 {$\\{\mathcal{F}(d_n) \| n \in Nat\\}$} 的一个上界。接着，因为 {$\sqcup \mathcal{F}(d_n)$} 是 {$\\{\mathcal{F}(d_n) \| n \in Nat\\}$} 的最小上界，所以 {$\sqcup \mathcal{F}(d_n) \sqsubseteq' \mathcal{F}(\sqcup d_n)$}。

但是，{$\mathcal{F}(\sqcup d_n) \sqsubseteq' \sqcup \mathcal{F}(d_n)$} 却并不一定成立。

### 不动点理论