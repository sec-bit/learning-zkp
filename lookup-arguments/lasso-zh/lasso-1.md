# 理解 Lasso（一）：Offline Memory Checking

假设我们有一个公开的 Table 向量 $\vec{t}$（长度为 $n$），和一个 Lookup 向量 $\vec{f}$（长度为 $m$），此外还有一个索引向量 $\vec{a}$（长度为 $m$），如何证明下面的 Indexed Lookup 关系？

$$
\forall i\in [0, m), f_i = t_{a_i}
$$

虽然我们有 Plookup, Caulk/Caulk+, Baloo, Logup，cq 等等方案可以直接使用，但 Offline memory checking 提供了一个更直观的新视角来看待 Lookup Argument。

## 1. Memory-in-the-head

我们把 Lookup 的过程看成是一个虚拟机读取内存的过程。如果一个 Lookup 关系成立，那么我们一定可以构造出一个合法的虚拟机执行序列，这个序列中的每一步都是合法的内存读取操作，从而证明每一个执行步读取的值 $f_i$ 都是出自只读内存 $\mathsf{mem}$，即证明了 Lookup 关系。如果我们关心内存读取的地址的话，那么我们就实现了一个 Indexed Lookup Argument。对于一个 Prover，她可以在本地构造虚拟机的执行序列 $T$，并向 Verifier 证明 $T$ 的合法性。而对于 Verifier 而言，我们比较关心 Verifier 如何在不需要遍历 $T$的情况下，验证这个长度为 $m$ 的执行序列。
因此，一个 Lookup 关系的证明，就转化为一段「只读内存」读取日志的正确性证明。换句话说，如果一串内存读取过程是正确的（符合虚拟机运行规则，并可以复现），那么就能推出这样的结论：如果每次读取的内容都是合理的，那么读取的值一定存在于原始内存（表格）中。这种证明思路可以形象地被称为「Memory in the head」，Prover 向 Verifier 证明一个头脑中想象出来的内存的读写合法性。

下面是一个虚拟机执行序列的例子，也是一个确定性状态转移关系：

$$
S_0 \overset{T_0}{\to} S_1 \overset{T_1}{\to} S_2 \overset{T_2}{\to} \cdots \overset{T_{m-1}}{\to} S_m
$$

其中 $(T_0, T_1, \ldots, T_{m-1})$ 代表内存读取操作，按顺序读出来 $(f_0, f_1, \ldots, f_{m-1})$。虚拟机内存状态 $S$ 是一个三元组 $(i, v, c)$ 的集合，一个三元组包含内存地址 $i$，内容 $v$ 和计数器 $c$ 三部分。注意到我们为每一个内存单元都附加一个计数器，标记着这个内存单元被读取的次数，这个计数器的作用是确保只读内存在读取过程中仍然会发生实质性的状态更新，从而提供了执行步 $T_i$ 的验证信息。


内存初始状态 $S_0$ 中的元素如下：

$$
S_0 = \begin{array}{|c|c|c|}
\text{addr.} & \text{value} & \text{counter} \\
\hline
0 & t_0 & 0 \\
1 & t_1 & 0 \\
2 & t_2 & 0 \\
\vdots & \vdots & \vdots \\
n-1 & t_{n-1} & 0 \\
\end{array}
$$

由于每一次内存的读取（虚拟机的执行）都会修改相应地址上的计数器，让计数器加一，因此我们规定虚拟机在每一次读写内存的前后，必须抛出两个日志（或者理解为事件），内存读取日志 $R$ 与内存更新日志 $W$。两者也同样都是一个三元组 $(i, v, c)$，包含内存读取地址 $i$，读取内容 $v$，和计数器的值 $c$。

$$
\begin{split}
R &= (i, v, c) \\
W &= (i, v, c)
\end{split}
$$

对于内存读取日志 $R$ 中的 $c$ 为读取时刻内存单元 $i$ 中的计数器值；$W$ 中的 $c$ 为更新后的计数器值。换句话说，我们也可以理解 $R$ 发生在一次内存读取之前, $W$ 发生在一次内存读取后，两个事件之前内存单元因为一次「读取」而将计数器的值加一。这一前一后两个日志的作用是约束每一次内存读取的合法性。怎么做到的呢？我们先看一个例子，假如内存的长度为 4，存放的内容为 $[t_0, t_1, t_2, t_3]$，假如我们要依次从内存中读取 $[t_1, t_3, t_1]$，那么会产生下面的日志序列

$$
\begin{split}
R_1 &= (1, t_1, 0)\\ 
W_1 &= (1, t_1, 1)\\
R_2 &= (3, t_3, 0)\\
W_2 &= (3, t_3, 1)\\
R_3 &= (1, t_1, 1)\\
W_3 &= (1, t_1, 2)\\
\end{split}
$$

三次读取产生的状态转移如下：

$$
S_0 : \begin{bmatrix}
0, & t_0, & 0 \\
1, & t_1, & 0 \\
2, & t_2, & 0 \\
3, & t_3, & 0 \\
\end{bmatrix} \longmapsto S_1: \begin{bmatrix}
0, & t_0, & 0 \\
1, & t_1, & {\color{red}1} \\
2, & t_2, & 0 \\
3, & t_3, & 0 \\
\end{bmatrix} \longmapsto S_2: \begin{bmatrix}
0, & t_0, & 0 \\
1, & t_1, & 1 \\
2, & t_2, & 0 \\
3, & t_3, & {\color{red}1} \\
\end{bmatrix} \longmapsto S_3 : \begin{bmatrix}
0, & t_0, & 0 \\
1, & t_1, & {\color{red}2} \\
2, & t_2, & 0 \\
3, & t_3, & 1 \\
\end{bmatrix}
$$

## 2. 内存读取的验证

现在思考下，一个 Prover 如何向 Verifier 证明虚拟机执行序列的合法性？下面我们给出虚拟机执行合法性的四个条件：

$$
\begin{array}{ll}
\text{Cond1:} & S_0.\vec{v} = \vec{t} \text{ and } S_0.\vec{c}=\vec{0}\\[1ex]
\text{Cond2:} & \exists S_n, S_0\longmapsto^* S_n \text{ and } S_0.\vec{v} = S_n.\vec{v}\\[1ex]
\text{Cond3:} & \forall W_j=(i, v, c), \exists R_j=(i, v', c'), v=v' \text{ and } c=c'+1\\[1ex]
\text{Cond4:} & \forall R_j=(i, v, c), \\[1ex]
& \qquad \text{ if } c=0, R_j=S_0[i] \\[1ex]
& \qquad \text{ if } c>0, \exists k>0, \exists W_{j-k}, W_{j-k} =R_j\\[1ex]
\end{array}
$$

解读如下：

+ **条件(1)**： 虚拟机执行必须从一个初始状态 $S_0$ 开始，即内存中依次存放着表格内容 $\vec{t}$，并且计数器都置为零；
+ **条件(2)**： 存在一个正确的终状态 $S_n$ ，并且 $S_n$ 中的内存数据 $\vec{t}$ 没有被修改；
+ **条件(3)**： 对于每一个 $W_j$ 日志，在「该事件之前」都会有一个成对出现的 $R_j$ 日志，他们记录的读取值相等 $W_j.v=R_j.v$，但 $W_j.c=R_j.c+1$；
+ **条件(4)**： 对于每一个 $R_j$ ，如果它是地址 $i$ 上的第一次读取，读取值应该等于内存初始状态 $R_j.v=S_0[i].v$；如果它是地址 $i$ 上的第二次或后续读取，那么在「该事件之前」一定有一个对应的 $W_{j-k}$ 日志，使得 $R_j=W_{j-k}$。

这四个条件是否完备充分呢？我们可以试着用归谬法推理下：

假如存在有一个 $R^*_j=(a^*,v^*,c^*)$ 中的读取数值 $v^*$ 非法， 即 $v^*\not\in\vec{t}$，并且此刻计数器为 $c^*$。 

假如 $c^*=0$，那么根据**条件 (4)**，$t^*=S_0[i^*].v$，又因为**条件 (1)**，$S_0(i^*).v=t_{i^*} \in \vec{t}$，这与假设矛盾。

另一种情况是 $c^*>0$，那么根据**条件 (4)**，一定存在一个 $W_{j-k}(i^*,v^*,c^*)$ 日志，使得 $W_{j-k}$ 的计数器值为 $c^*$，再根据**条件 (3)**，一定存在一个 $R_{j-k}$ 日志，使得 $R_{j-k}(i^*, v^*, c^*-1)$。以此递归地推理下去，每次 $c^*$ 递减一，最后我们一定可以得到某个 $R_{j-k-l}=(i^*, v^*, {\color{red}0})$，于是根据**条件 (1)** 可得， $S_0=(i^*, v^*, 0)$，这与**条件（1）** $S_0$ 的正确性（$v^*\in\vec{t}$）矛盾。

到此推理完毕，存在有一个非法读取日志 $R^*_j$ 的假设不正确，因此我们得出结论：满足上面四个条件的虚拟机执行序列中，不可能出现读取一个错误的值的情况。

因此，只要 Prover 能够证明，(1) 初始状态 $S_0$ 正确，并且 (2) 每一步读取日志是自洽的，那么我们可以证明读取过程就是不可伪造的。Offline Memory Checking 提供了一个漂亮的约束等式，同时满足上面四个条件：

$$
S_0 \cup \{W_j\}_{j=0}^{m-1} = S_n \cup \{R_j\}_{j=0}^{m-1}
$$

我们进一步分析下这个约束等式，先展开下等式左右两边的定义：

$$
  S_0=\{(i, t_i, 0)\}_{i\in[n]} \cup W=\{(a_j, f_j, c_j+1)\}_{j\in[m]} = S_n=\{(i, t_i, c'_i)\}_{i\in[n]} \cup R=\{(a_j, f_j, c_j)\}_{j\in[m]}
$$

这个等式约束是关于四个多重集合（Multiset）之间的关系。容易看出，初始状态约束 **条件(1)** 和 终状态约束 **条件(2)** 已体现在上面的等式中。接下来我们简单分析下，上面这个等式如何保证了 **条件(3)** 和 **条件(4)**。

先看下**条件(3)**，对于每一个 $W_j$（出现在等式左侧），那么就有一个成对出现的 $R_j$（出现在等式右侧），两个日志的差别是右侧 $R_j$ 的计数器值少一。看下 **条件(4)**，如果某个 $R_j$ 中的计数器值为零，那么在等式左边一定有一个相同的三元组元素，出现在 $S_0$ 集合中；如果 $R_j$ 中的某个元素的计数器值大于零，那么这个元素一定出现在等式左边的 $R_j$ 中。

注意到，等式右边来自于 $S_n$ 中的每一个元素，可能出现在左边的 $S_0$ 中，这意味着该元素所对应的内存单元从未被读取过；$S_n$ 集合元素也可能出现在 $W$ 中，这意味着该元素的计数器值等于最后一次内存单元计数器的更新值。

最后我们分析下 Prover 和 Verifier 的输入。对于 Verifier 而言，$S_0$ 属于 Public inputs，这样 Verifier 可以验证 **条件(1)**，Verifier 要求 Prover 提供 $\vec{c}'$ 向量，从而构造 $S_n$，验证 **条件(2)**。此外 Public inputs 还要包括承诺 $\mathsf{cm}(\vec{a})$， $\mathsf{cm}(\vec{t})$ 和 $\mathsf{cm}(\vec{f})$，以便 Verifier 同态地验证 Multiset 等价关系。而日志集合 $\{R_j\}, \{W_j\}$ 由 Prover 构造，并给出其中计数器部分的承诺 $\mathsf{cm}(\vec{c})$，从而允许 Verifier 来验证正确性条件 (3)。而 Verifier 也可以根据 $\mathsf{cm}(\vec{a})$， $\mathsf{cm}(\vec{t})$ 和 $\mathsf{cm}(\vec{f})$ 还有 $\mathsf{cm}(\vec{c})$，同态地构造出 $\{R_j\}, \{W_j\}$ 的承诺。

接下来我们利用 Memory-in-the-head 的思路，设计一个 PIOP 协议，实现 Indexed Lookup Argument。

## 3. 构造 Lookup Argument 协议

我们把这四个集合 $S_0, \{R_j\}, \{W_j\}, S_n$  看成是三列矩阵，并且所有的矩阵列向量都编码为多项式。其中 $S_0$ 矩阵的三列记为 $S_i(X)$, $t(X)$ 与 $S_c(X)$，这里注意
在 $S_0$ 中的 value 一列必须等于表格向量 $\vec{t}$。矩阵 $S_n$ 为虚拟机的终止状态，由于虚拟机内存为只读内存，因此 $addr.$ 和 $value$ 两列保持不变，但是内存单元计数器被更新到了 $\vec{c}'$，编码为 $S'_c(X)$，多项式编码的 Domain 记为 $H\subset \mathbb{F}$。

$$
\small
\begin{array}{ll}
S_0=
\begin{array}{|c|c|c|}
\hline
\text{addr.} & \text{value} & \text{counter}\\
\hline
0 & t_0 & 0\\
1 & t_1 & 0\\
2 & t_2 & 0\\
\vdots & \vdots & \vdots \\
n-1 & t_{n-1} & 0 \\
\hline
S_i(X) & t(X) & S_c(X) \\
\hline
\end{array}
& 
R=
\begin{array}{|c|c|c|}
\hline
\text{addr.} & \text{value} & \text{counter}\\
\hline
a_0 & f_0 & c_0\\
a_1 & f_1 & c_1\\
a_2 & f_2 & c_2\\
\vdots & \vdots & \vdots \\
a-1 & f_{m-1} & c_{m-1}\\
\hline
a(X) & f(X) & R_c(X)\\
\hline
\end{array}
\qquad \\[10ex]
S_n=
\begin{array}{|c|c|c|}
\hline
\text{addr.} & \text{value} & \text{counter}\\
\hline
0 & t_0 & c'_0\\
1 & t_1 & c'_1\\
2 & t_2 & c'_2\\
\vdots & \vdots & \vdots \\
n-1 & t_{n-1} & c'_{n-1} \\
\hline
S_i(X) & t(X) & S'_c(X) \\
\hline
\end{array}
& 
W=
\begin{array}{|c|c|c|}
\hline
\text{addr.} & \text{value} & \text{counter}\\
\hline
a_0 & f_0 & c_0+1\\
a_1 & f_1 & c_1+1\\
a_2 & f_2 & c_2+1\\
\vdots & \vdots & \vdots \\
a-1 & f_{m-1} & c_{m-1}+1 \\
\hline
a(X) & f(X) & W_c(X)\\
\hline
\end{array}
\end{array}
$$

日志矩阵 $R$ 的第一列为读取的地址序列，它必须等于地址向量 $\vec{a}$，第二列为读取的值，等于 $\vec{f}$，而第三列 $\vec{c}$ 为 Prover 维护的计数器向量，编码为 $R_c(X)$。再看下矩阵 $W$，其每一行为一条内存更新日志，其中第三列为更新后的计数器值，这个值编码为 $W_c(X)$，并应该满足下面的约束：

$$
W_c(X)=R_c(X)+1
$$

下面是 Offline Memory Checking 的约束等式：

$$
\begin{array}{ccccccc}
S_0 && W & & S_n & & R \\
  \Big(\{(i, t_i, 0)\}_{i\in[n]}\Big) &\cup& \{(a_j, f_j, c_j+1)\}_{j\in[m]} &=& \{(i, t_i, c'_i)\}_{i\in[n]} &\cup& \{(a_j, f_j, c_j)\}_{j\in[m]}
\end{array}
$$

我们可以用多项式之间的约束关系描述下 Multiset 等价约束：

$$
S_0(Y, Z)\cdot W(Y, Z) =  S_n(Y, Z) \cdot R(Y, Z)
$$

其中四个二元多项式的定义如下：

$$
\begin{split}
  S_0(Y, Z) &= \prod_{X\in H}\Big(S_i(X)+ t(X) \cdot Y + S_c(X)\cdot Y^2 - Z\Big)\\
  S_n(Y, Z) &= \prod_{X\in H}\Big(S_i(X)+ t(X) \cdot Y + S'_c(X)\cdot Y^2 - Z\Big)\\
  R(Y, Z) &= \prod_{X\in H}\Big(a(X) + f(X)\cdot Y + R_c(X)\cdot Y^2 - Z \Big) \\
  W(Y, Z) &= \prod_{X\in H}\Big(a(X) + f(X)\cdot Y + W_c(X)\cdot Y^2 - Z\Big) \\
\end{split}
$$

其中 $W_c(X)=R_c(X) + 1$。

我们可以再使用两个 Verifier 提供的随机挑战数 $Y=\beta$ 与 $Z=\gamma$ ，把上面的多项式等价关系归结到两个 Grand Product 之间的等价关系。而 Grand Product Argument，我们可以有多种方案来完成。例如我们可以采用 Plonk 协议中的 Grand Product 子协议来完成，也可以采用 GKR 协议，或者 论文 [Quarks, SL20] 中 基于 Sumcheck 的协议。

### 协议描述

公共输入：
1. $C_t = \mathsf{cm}(\vec{t})$ ，$|\vec{t}| = n$
2. $C_f = \mathsf{cm}(\vec{f})$， $|\vec{f}| = m$
3. $C_a = \mathsf{cm}(\vec{a})$， $|\vec{a}| = m$

#### 第一轮

Prover 模拟内存读取流程得到终状态 $S_m=\{(i, t_i, c^\mathsf{final}_i)\}_{i\in[m]}$，得到 $\{R_{j}\}_{j\in[m]}$，$\{W_{j}\}_{j\in[m]}$

$$
\begin{split}
R_{j} &= \{(a_j, f_j, c_j)\}, \qquad {j\in[m]} \\
W_{j} &= \{(a_j, f_j, c_j+1)\}, \qquad {j\in[m]} \\
\end{split}
$$

Prover 计算 $\{c_j\}_{j\in[m]}$ 的承诺 $C_c=\mathsf{cm}(\{c_j\})$，
Prover 计算 $\{c^{\mathsf{final}}_i\}_{i\in[n]}$ 的承诺 $C^\mathsf{final}_c$

Prover 发送 $\big(C_c, C^\mathsf{final}_c\big)$

#### 第二轮

Verifier 发送挑战数 $\beta, \gamma$

Prover 计算读取/更新日志向量 $R(X)$, $W(X)$, 

$$
\begin{split}
R_j &= a_j +\beta\cdot f_j + \beta^2\cdot c_j -\gamma \\
W_j &= a_j +\beta\cdot f_j + \beta^2\cdot (c_j+1) -\gamma \\
\end{split}
$$

Prover 计算  $S^{\mathsf{init}}(X)$ 与 $S^{\mathsf{final}}(X)$

$$
\begin{split}
S^{\mathsf{init}}_i &= i + \beta \cdot t_i + \beta^2\cdot 0 - \gamma \\
S^{\mathsf{final}}_i &= i + \beta \cdot t_i + \beta^2\cdot c^{\mathsf{final}}_i - \gamma \\
\end{split}
$$

Prover 和 Verifier 利用 Grand Product Argument 来证明下面的等式：

$$
(\prod_{i=0}^{n-1} S^{\mathsf{init}}_i)\cdot (\prod_{j=0}^{m-1} R_j) = (\prod_{i=0}^{n-1} S^{\mathsf{final}}_i)\cdot (\prod_{j=0}^{m-1} W_j)
$$

#### 验证

Verifier 计算 $C_R, C_W, C_S^{\mathsf{init}}, C_S^{\mathsf{final}}$，并验证 Grand Product Argument

$$
C_S^{\mathsf{init}} = C_I + \beta \cdot C_t - \gamma\cdot [1] \\
C_R = C_a + \beta\cdot C_f + \beta^2\cdot C_c - \gamma\cdot [1] \\
C_W = C_a + \beta\cdot C_f + \beta^2\cdot (C_c+[1]) - \gamma\cdot [1] \\
C_S^{\mathsf{final}} = C_I + \beta \cdot C_t + \beta^2\cdot C^\mathsf{final}_c- \gamma\cdot [1] \\
$$

这里 $C_I = \mathsf{cm}(0,1,\ldots,n-1)$

## 4. 对比理解 Offline Memory Checking 

与 Plookup, Caulk/Caulk+, flookup, Baloo, CQ 相比， Memory-in-the-head 方式证明 Lookup 是一个巧妙且直观的想法。不过我们会想知道他们之间有何差别？

这一节，我们从 Plookup 的角度出发，换一个角度来理解 Offline memory checking。

我们先假设 $\vec{f}$ 中不存在重复元素，那么我们可以采用 Vanishing Form 的方式来编码 $\vec{f}$ 与 $\vec{t}$ 为多项式：

$$
f(X) = (X-f_0)(X-f_1)(X-f_2)\cdots(X-f_{m-1})
$$

$$
t(X) = (X-t_0)(X-t_1)(X-t_2)\cdots(X-t_{n-1})
$$

Prover 可以通过下面的等式来证明 $\vec{f}\subset\vec{t}$：

$$
\exists q(X), \quad t(X) = f(X)\cdot q(X)
$$

但是如果考虑 $\vec{f}$ 中存在重复元素，那么用 Vanishing Form 编码的多项式就不满足上面的等式约束了。处理重复元素是 Lookup Argument 中比较棘手的问题。为了修补这个方案，我们要为表格向量 $\vec{t}$ 和查询向量 $\vec{f}$ 分别扩展一个新的列向量，称为计数器列 $\vec{c}$。每当 $\vec{f}$ 中出现重复读取同一个表格元素时，我们可以通过计数器列来区分这两次不同的读取。比如 $\vec{f}$ 中有两次对 $t_0$ 的查询，那么我们可以定义一个扩展后的查询向量 $\vec{f}^*$:

$$
\vec{f}^* = [(t_0, 0), (t_0, 1), (t_1, 0)]
$$

扩展后的向量中的每一个元素是一个二元组，其中第二部分为计数器值。 扩展查询中的前两个查询 $(t_0, 0), (t_0, 1)$，虽然查询值都为 $t_0$，但是由于计数器会按顺序加一，因此，两个二元组不再相等。

同样，我们也可以定义一个扩展后的表格向量 $\vec{t}^*$:

$$
\vec{t}^* = [(t_0, 0), (t_1, 0)]
$$
 
那么我们会问下面的公式会成立吗？

$$
\{f^*_j\} \overset{?}{\subset}  \{t^*_i\}
$$

很显然，它不成立，因为等式左边有 $(t_0, 1)$，而右边的集合中不包含这个元素。显然我们需要在公式的右边补上 $(t_0, 1)$。换句话说，我们需要在右边补上那些由于计数器累加产生的重复表项，记为 $\vec{p}^*$。

$$
 \{f^*_j\} \overset{?}{\subset} \{t^*_i\}\cup\{p^*_j\} 
$$

但是这个向量 $\vec{p}$ 不能由 Prover 提供。为了防止 Prover 作弊， $\vec{p}$ 必须由 Verifier 来提供。那么接下来，我们面临的问题是，Verifier 并不清楚哪些 $f_i$ 重复，并且也不能知道重复了几次。这个问题该如何解决？

我们可以把查询 $f^*_j$ 看成是一个「消耗」表格元素的机器，每次查询 $f_i$，都会消耗掉一个对应的表格元素 $t_j$。我们把向量 $\vec{p}^*$ 看成一个可以「产生」新元素的机器，每次出现一个对 $t_j$ 重复查询的记录，比如 $f^*_i=(t_j, c_i)$，那么 $\vec{p}^*$ 就会自动产生出一个新的元素，记为 $(t_j, c_i+1)$，供下一次查询「消耗」。这样我们就可以让 Verifier 在等式左边添加 $m$ 个元素，正好对应 $\vec{f}$ 的元素，但是所有元素中的计数器都自增一。这样，等式左边的集合 $\{p^*_j\}$ 就可以由 Verifier 自行构造：

$$
p_j = (f_j, c_j+1)
$$

这个公式成立 $\{f^*_j\}\subset \{t^*_i\}\cup\{p^*_j\}$，我们继续可以用 Vanishing Form 的方式来表达这个子集关系，比如对上面的例子，我们可以得到下面的多项式等式：

$$
(X-t_0)(X-t_1)\cdot p(X) = (X-t_0)(X-t_0-\beta)(X-t_1)\cdot q(X)
$$

其中 $\beta$ 为 Verifier 提供的随机挑战数，用来合并表格 $\vec{t}$ 和 $\vec{f}$ 的二元组为一个单值。这里 $p(X)$ 编码了那个能自动产生新元素的机器，它的每一个因子都是一个 $(t_i, c_i+1)$ 元素。下面是 $p(X)$ 多项式的定义：

$$
p(X) = (X-t_0 - \beta)(X-t_0 - 2\beta)(X-t_1 - \beta)
$$

多项式等式右边的多项式 $q(X)$ 会有哪些元素呢？$q(X)$ 恰好包含有所有的等待被消耗的 $t_i$ 元素，其中包括始终没有被查询过的计数器为零的元素，还包括被查询过的，但是又被 $\vec{p}^*$ 复制产生的元素。于是我们得到了下面的等式约束：

$$
\begin{array}{ccccccc}
t(X) &  & p(X) &  & f(X) & & q(X) \\[1ex]
\{(t_i, 0)\}_{i\in[n]} &\cup& \{(f_i, c_i+1)\}_{i\in[m]} &=& \{(f_i, c_i)\}_{i\in[m]} &\cup& \{(t_i, c_i')\}_{i\in[n]} \\
\end{array}
$$

下面我们证明上面的等式保证了 $\vec{f}$ 中的每一个元素都是 $\vec{t}$ 中的元素。

我们用反证法，假如存在一个 $f_i\not\in \vec{t}$，那么根据上面的等式，一定存在一个计数器 $(f_i, k)$ 出现在等式的左边。这时候如果 $k=0$，那么 $f_i=t_i$，与假设矛盾。那么这时候可以断定 $k>0$，那么等式右边一定存在一个 $(f_i, k-1)$，才会让左边出现 $(f_i, k)$；同理可推，左边一定存在一个 $(f_i, k-1)$，那么右边一定会出现一个 $(f_i, k-2)$。以此类推，我们一定可以得到：等式左边会出现 $(f_i, 0)$，于是 $f_i=t_i$，这又与初始初始假设矛盾。

这个思路与 Memory-in-the-head 几乎一摸一样，除了我们不考虑表格的索引问题。基于这个思路，我们可以构造一个 Unindexed Lookup Argument。

### 对比 Plookup 

回忆下 Plookup 的方案，对于 $\vec{f}$ 和 $\vec{t}$，如果我们要证明 $\vec{f}\subset \vec{t}$，那么 Prover 需要引入一个中间向量 $\vec{s}$，长度为 $n+m$。它是 $\vec{f}\cup\vec{t}$ 的一个重新排序，按照 $\vec{t}$ 中原有项的顺序进行排列。然后 Prover 证明下面的 Multiset 等价关系：

$$
\{(s_i, s_{i+1})\}\overset{?}{=}\{(f_i, f_i)\}\cup\{(t_i, t_{i+1})\}
$$

这个方案和 Memory-checking 相比，Multiset 约束等式两边的集合元素数量都为 $m+n$，但 Plookup 需要多引入一个中间辅助向量 $\vec{s}$，而后者则需要引入一个计数器向量 $\vec{c}$。计数器向量节省了 Prover 在排序上的工作开销，另一方面，向量 $\vec{c}=(0,1,\ldots,m-1)$ 中的值较小且规律，Prover 计算其承诺会更有优势（如 Perdesen 承诺或者 KZG10）。

## 5. 小结

本文介绍了如何采用传统的 Offline Memory Checking 技术构造 Lookup Arguments，其中关于 Memory Checking 的公式 $S_0\cup W = R\cup S_m$ 蕴含着非常巧妙的思想。

## References

- [SL20]  [Quarks: Quarks: Quadruple-efficient transparent zkSNARKs](https://eprint.iacr.org/2020/1275) by Srinath Setty and Jonathan Lee.
- [Lasso] [Unlocking the lookup singularity with Lasso](https://eprint.iacr.org/2023/1216) by Srinath Setty, Justin Thaler and Riad Wahby.
- [Jolt] [Jolt: SNARKs for Virtual Machines via Lookups](https://eprint.iacr.org/2023/1217) by Arasu Arun, Srinath Setty and Justin Thaler.
- [PLONK] [PLONK: Permutations over Lagrange-bases for Oecumenical Noninteractive arguments of Knowledge](https://eprint.iacr.org/2019/953.pdf) by Ariel Gabizon, Zachary J. Williamson and Oana Ciobotaru.
- [Plookup] [plookup: A simplified polynomial protocol for lookup tables](https://eprint.iacr.org/2020/315) by Ariel Gabizon and Zachary J. Williamson.
- [Caulk] [Caulk: Lookup Arguments in Sublinear Time](https://eprint.iacr.org/2022/621) by Arantxa Zapico, Vitalik Buterin,
Dmitry Khovratovich, Mary Maller, Anca Nitulescu and Mark Simkin
- [Caulk+] [Caulk+: Table-independent lookup arguments](https://eprint.iacr.org/2022/957) by Jim Posen and Assimakis A. Kattis.
- [Baloo] [Baloo: Nearly Optimal Lookup Arguments](https://eprint.iacr.org/2022/1565) by Arantxa Zapico, Ariel Gabizon, Dmitry Khovratovich, Mary Maller and Carla Ràfols.
- [CQ] [cq: Cached quotients for fast lookups](https://eprint.iacr.org/2022/1763) by Liam Eagen, Dario Fiore and Ariel Gabizon.
