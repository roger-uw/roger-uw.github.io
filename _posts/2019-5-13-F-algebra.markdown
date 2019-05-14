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

主要参考：

[University of Illinois - CS522 - Programming Language Semantics (Fall 2018) - Conventional Semantic Approaches](http://fsl.cs.illinois.edu/images/c/ca/CS522-Fall-2018-basic-semantics.pdf)

[Cornell University - CS6110 - Advanced Programming Languages (Spring 2019) - Partial Orders & Continuity](http://www.cs.cornell.edu/courses/cs6110/2019sp/lectures/lec19.pdf)

## 不动点

### 偏序集

一个**偏序集（partial order set）** {$(D, \sqsubseteq)$} 由集合 {$D$} 和一个在 {$D$} 上定义的二元关系 {$\sqsubseteq$} 组成，满足：

* **自反性**：对于任意 {$x \in D$}，{$x \sqsubseteq x$}
* **传递性**：对于任意 {$x, y , z \in D$}，若 {$x \sqsubseteq y$} 且 {$y \sqsubseteq z$}，则 {$x \sqsubseteq z$}
* **反对称性**：对于任意 {$x, y \in D$}，如 {$x \sqsubseteq y$} 且 {$y \sqsubseteq x$}，则 {$x = y$}

一个偏序集 {$(D, \sqsubseteq)$} 如果还满足**完全性**，即对于任意 {$x, y \in D$}，{$x \sqsubseteq y$} 或 {$y \sqsubseteq x$}，那么它就被称为一个**全序集（total order set）**。

例子：

* {$(Nat, \leq)$}：自然数集和“小于或等于”构成一个全序集。
* {$(Nat \cup \{\infty\}, \leq)$}：加入无穷大的自然数集和“小于或等于”构成一个全序集。
* {$(S \cup {\perp}, \leq^S_\perp)$}：扩充了一个底（bottom）元素的集合 {$S$} 和二元关系 {$\leq^S_\perp$} 构成一个偏序集。对于任意 {$a, b \in S \cup {\perp}$}，当且仅当 {$a = b$} 或 {$a = \perp$} 时，{$a \leq^S_\perp b$}。在指称语义中，像这样用 {$\perp$} 扩充集合很常见，其中 {$\perp$} 用于表示*未定义值（undefined）*。{$S \cup {\perp}$} 可简写为 {$S_\perp$}，称作*基本域（primitive domain）*或*平坦域（flat domain）*。
* {$(A \rightharpoondown B, \preceq)$}：从 {$A$} 到 {$B$} 的偏函数（partial function）构成的集合和信息量关系（informativeness relation）{$\preceq$} 构成一个偏序集。对于偏函数 {$f, g : A \rightharpoondown B$}，我们称 {$f$} 的信息量少于或等于 {$g$}，当且仅当对于任意 {$a \in A$}，要么 {$f(a)$} 未定义，要么 {$f(a)$} 和 {$g(a)$} 均有定义且 {$f(a) = g(a)$}。

有时候，偏序集可以用哈斯图（Hasse diagram）来图形化描述。在哈斯图中，偏序集的每个元素都被绘制为一个（可能带标签的）点，点之间的连接线的绘制遵循以下规则：

* 若 {$x$} 和 {$y$} 是偏序集的元素且 {$x \sqsubseteq y$}，那么对应 {$x$} 的点画在对应 {$y$} 的点下方。

* 当且仅当 {$x \sqsubseteq y$} 且在偏序集中不存在一个元素 {$z$} 严格处于 {$x$} 和 {$y$} 之间，也就是 {$x$} 和 {$y$} 之间的序关系不是由于传递性时，才能在代表 {$x$} 和 {$y$} 的点之间连线。

对于偏序集 {$(D, \sqsubseteq)$} 和 {$X \subseteq D$}，元素 {$p \in D$} 被称作 {$X$} 的**上界（upper bound）**，当且仅当对于任意 {$x \in X$}，{$x \subseteq p$}。以及，当且仅当 {$p$} 是一个上界，且对于 {$X$} 的其它任意上界 {$q$}，都有 {$p \sqsubseteq q$} 时，{$p$} 被称为**最小上界（least upper bound, LUB）**或上确界，写作 {$\sqcup X$}。

上界和最小上界并不一定总是存在。比如，若 {$D = X = {x,y}$} 且 {$\sqsubseteq$} 是恒等关系，那么 {$X$} 没有上界。即使上界存在，最小上界也不一定存在。比如，若 {$D = {a, b, c, d, e}$}，{$\sqsubseteq$} 定义为 {$a \sqsubseteq c$}，{$a \sqsubseteq d$}，{$b \sqsubseteq c$}，{$b \sqsubseteq d$}，{$c \sqsubseteq e$}，{$d \sqsubseteq e$}（如下图），那么 {$D$} 的任意子集都存在上界，但集合 {${a, b}$} 没有最小上界。由于反对称性，最小上界存在即唯一。

![HasseDiagram]({{ "/assets/F-algebra/Hasse.svg" | absolute_url }})

### 完全偏序

### 单调和连续函数

### 不动点理论