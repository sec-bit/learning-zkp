# 理解 Lasso（一）：Memory-in-the-head

假设我们有一个公开的 Table 向量 $\vec{t}$，和一个 Lookup 向量 $\vec{f}$，我们可以利用 Lookup argument 来证明下面的 lookup 关系：

$$
\forall i\in [m], f_i \in \vec{t}
$$

上面这个 Lookup Argument 定义被 Lasso 论文称为 Unindexed Lookup Argument。因为这个定义只保证 $f_i$ 在 $\vec{t}$ 中，但是并不保证 $f_i$ 出现在 $\vec{t}$ 中某个特定的位置。

假如我们要表示一个 2bit-XOR 运算，需要用到一个三列表格：

$$
\begin{array}{|c|c|c|}
\hline
A & B & A\oplus B \\
\hline
00 & 00 & 00 \\
00 & 01 & 01 \\
\vdots & \vdots & \vdots \\
11 & 11 & 00 \\
\hline
\end{array}
$$

其中第一列表示第一个运算数 $A$，第二列表示第二个运算数 $B$，第三列表示 XOR 运算结果， $A\oplus B$。显然，表格中的行是可以互换位置的，而不影响表格所表达的 XOR 操作。注意到这个表格共有 16 行。

那么我们问，能不能采用一个单列表格来表达这个 XOR 运算？

## Indexed Lookup Arguments

其中一个思路是这样的，我们只采用一列表格来表示 XOR 运算的结果，即 $A\oplus B$，而用表格的行索引来表示两个运算数。比如，我们在第 0 行放上 $00$，因为 $00 = 00\oplus 00 $，等号右边为行数的 4bit 编码， $0000$，高位 $00$ 表示 $A$，低位 $00$ 表示 $B$。又比如，第 5 行放上 $00$，因为 $01\oplus 01=00$，而 $(01 01)_{(2)}=5_{(10)}$。 可见，这个单列表格的大小仍然是 16 行。但是这个表格的元素是不允许打乱顺序的，必须保持有序。

$$
\begin{array}{c|c}
i & A \parallel B \\
\hline
0 & 00 \parallel 00 \\
1 & 00 \parallel  01 \\
\vdots & \vdots  \\
15 & 11 \parallel  11 \\
\end{array}
\quad
\begin{array}{|c|}
\hline
A\oplus B \\
\hline
 00 \\
 01 \\
\vdots \\
 00 \\
\hline
\end{array}
$$

单列有序表格的好处不仅是只需用一个多项式编码，Lasso 还进一步探索了单列表格的内部结构，探索如何把单列的大表格拆分成多个小表格，从而提高证明效率。Jolt 展示了如何利用表格的内部结构，来编码 RISC-V 的完整指令运算。

基于单列有序的表格，Lasso 论文定义了一类新的 Lookup Argument，称之为 Indexed Lookup Argument：

$$
\forall i\in [m], \exists j\in[n], f_i = t_j
$$

在 Lasso/Jolt 协议中，Offline Memory Checking 正是一个核心的 Indexed Lookup Argument。

## Memory in the head

重复一遍问题：假设我们有一个公开的 Table 向量 $\vec{t}$（长度为 $n$），另有一个隐私的 lookup 向量 $\vec{f}$（长度为 $m$），还有一个位置向量 $\vec{a}$，如何证明下面的 lookup 关系：

$$
\forall i\in [m], f_i = t_{a_i}
$$

虽然我们有 Plookup, Caulk/Caulk+, Baloo, Logup，CQ 等等方案可以直接使用，但 Offline memory checking 提供了一个全新的视角。

设想我们有一段内存 $\mathsf{mem}$，内存里面放着表格向量 $\vec{t}$，即 $\mathsf{mem}[i]=t_i$，这里 $i$ 为内存地址。然后我们为每一个内存单元都附加一个计数器，标记着这个内存单元被读取的次数。所有的计数器初始化为零，被记为 $c[i]=0$。

新视角是我们把 Lookup 的过程看成是内存读取的过程。定义两个概念，内存 State，与内存读取日志 RLog 与 WLog。其中 State 是一个三元组 $(i, v, c)$ 的集合，一个三元组包含内存地址 $i$，内容 $v$ 和计数器 $c$ 三部分。

$$
S = \{(i, v, c)\}
$$

每一次内存的读取都会修改相应地址上的计数器，让计数器加一，并且同时产生两个事件的日志，读取数据 $\mathsf{RLog}$ 和 更新计数器 $\mathsf{WLog}$。两者也同样都是一个三元组 $(a, v, c)$，包含地址，读取的内容，和计数器的值。  

$$
\begin{split}
\mathsf{RLog} &= (a, v, c) \\
\mathsf{WLog} &= (a, v, c)
\end{split}
$$

其中 $a$ 表示读取的地址，$v$ 表示读取到的内容，$\mathsf{RLog}$ 中的 $c$ 为：在读取时刻内存单元 $i$ 中的计数器值；$\mathsf{WLog}$ 中的 $c$ 为更新后的计数器值。换句话说，我们也可以理解 $\mathsf{RLog}$ 发生在一次内存读取之前, $\mathsf{WLog}$ 发生在一次内存读取后，两个事件之前内存单元因为一次「读取」而将计数器的值加一。换个角度，这很像一个非常简单的虚拟机，并且只有一条指令，即读取指定的内存单元。每次指令执行都会带来状态的转移，并且抛出两个事件（$\mathsf{RLog}$ 和 $\mathsf{WLog}$）。

因此，一个 Lookup 关系的证明，就转化为一段「只读内存」读取日志的正确性证明。换句话说，如果一串内存读取过程是正确的（符合虚拟机运行规则，并可以复现），那么就能推出这样的结论：如果每次读取的内容都是合理的，那么读取的值一定存在于原始内存（表格）中。这种证明思路可以形象地被称为「Memory in the head」，Prover 向 Verifier 证明一个头脑中想象出来的内存的读写合法性。

那么如何证明一系列读取日志的正确性？我们先看一个例子，假如内存的长度为 3，存放的内容为 $[a, b, c, d]$，假如我们要依次从内存中读取 $[a, b, a, c]$，那么会产生下面的 Log：

$$
\begin{split}
S_0 &: ([a, b, c, d], [0, 0, 0, 0])\\ 
\mathsf{RLog}_1 &= (0, a, 0)\\
S_1 &: ([a, b, c, d], [1, 0, 0, 0])\\ 
\mathsf{WLog}_1 &= (0, a, 1)\\
\mathsf{RLog}_2 &= (1, b, 0)\\
S_2 &: ([a, b, c, d], [1, 1, 0, 0])\\
\mathsf{WLog}_2 &= (1, b, 1)\\
\mathsf{RLog}_3 &= (0, a, 1)\\
S_3 &: ([a, b, c, d], [2, 1, 0, 0])\\
\mathsf{WLog}_3 &= (1, a, 2)\\
\mathsf{RLog}_4 &= (2, c, 0)\\
S_4 &: ([a, b, c, d], [2, 1, 1, 0])\\
\mathsf{WLog}_4 &= (2, c, 1)\\
\end{split}
$$

现在思考下，一个 Prover 如何向 Verifier 证明虚拟机状态以及读取日志序列的合法性？下面我们给出虚拟机执行合法性的五个条件：

+ **条件(1)**： Log 必须从一个初始状态 $S_0$ 开始，即内存中依次存放着表格内容 $\vec{t}$，并且计数器都置为零；
+ **条件(2)**： 存在一个正确的终状态 $S_n$ ，即内存中的 $\vec{t}$ 没有被修改；
+ **条件(3)**： 对于每一个事件 $\mathsf{WLog}_j=(i, v, c)$ 日志，在「该事件之前」都会有一个成对出现的 $\mathsf{RLog}_j=(i, v', c')$ 日志，他们记录的读取值相等 $v=v'$，但 $\mathsf{WLog}_j$ 的计数器值为 $\mathsf{RLog}_j$ 的计数器值加一, 即 $c=c'+1$；
+ **条件(4)**： 对于每一个事件 $\mathsf{RLog}_j=(i, v, c)$ ，如果它是地址 $i$ 上的第一次读取，读取值应该等于内存初始状态 $v=S_0(i).v$，并且计数器为零 $c=S_0(i).c=0$；
+ **条件(5)**： 对于每一个事件 $\mathsf{RLog}_j=(i, v, c)$ ，如果它是地址 $i$ 上的后续读取，那么在「该事件之前」一定有一个对应的 $\mathsf{WLog}_{j-k}=(i, v', c')$ 日志， $\mathsf{RLog}_j$ 的计数器值「等于」 $\mathsf{WLog}_{j-k}$ 的计数器值，即 $c=c'$，那么 $\mathsf{WLog}_{j-k}=(i, t, c)$。

这个「正确性」是否充分呢？我们可以试着用归谬法推理下：

假如存在有一个 $\mathsf{RLog}_j(a^*,v^*,c^*)$ 中的读取数值 $v^*$ 非法， $v^*\not\in\vec{t}$，并且此刻计数器为 $c^*$。 假如 $c^*=0$，那么根据**条件 (4)**，$t^*=S_0(i^*).v$，又因为**条件 (1)**，$S_0(i^*).v=t_{i^*} \in \vec{t}$，这与假设矛盾。

另一种情况是 $c^*>0$，那么根据**条件 (5)**，一定存在一个 $\mathsf{WLog}_{j-k}(i^*,v^*,c^*)$ 日志，使得 $\mathsf{WLog}_{j-k}$ 的计数器值为 $c^*$，再根据**条件 (3)**，一定存在一个 $\mathsf{RLog}_{j-k}$ 日志，使得 $\mathsf{RLog}_{j-k}(i^*, v^*, c^*-1)$。以此递归地推理下去，每次 $c^*$ 递减一，最后我们一定可以得到某个 $\mathsf{RLog}_{j-k-l}=(i^*, v^*, {\color{red}0})$，于是根据**条件 (1)** 可得， $S_0=(i^*, v^*, 0)$，这与**条件（1）** $S_0$ 的正确性（$v^*\in\vec{t}$）矛盾。

到此推理完毕，假设错误，因此我们得出结论：满足上面五个条件的虚拟机执行序列中，不可能出现读取一个错误的值的情况。

因此，只要 Prover 能够证明，(1) 初始状态 $S_0$ 正确，并且 (2) 每一步读取日志是自洽的，那么我们可以证明读取过程就是不可伪造的。**条件 (1)(2)(3)(4)(5)** 可以编码为下面的等式约束：

$$
  \{S_0(i, t_i, 0)\}_{i\in[n]} \cup \{\mathsf{WLog}_j(a_j, f_j, c_j+1)\}_{j\in[m]} = \{S_n(i, t_i, c'_i)\}_{i\in[n]} \cup \{\mathsf{RLog}_j(a_j, f_j, c_j)\}_{j\in[m]}
$$

这个等式约束是关于四个 Multiset 之间的关系。其中 $S_0$ 是一个公开的起始状态，它需要满足**条件(1)**。此外，**条件(2)** 关于终状态 $S_n$ 的存在性也体现在上面的等式中。接下来我们简单分析下，上面这个等式如何保证了 **条件(3)**，**条件(4)**，**条件(5)**。

先看下**条件(3)**，对于每一个 $\mathsf{WLog}$（出现在等式左侧），那么就有一个成对出现的 $\mathsf{RLog}$（出现在等式右侧），两个日志的差别是右侧 $\mathsf{RLog}$ 的计数器值少一。看下 **条件(4)**，如果某个 $\mathsf{RLog}$ 中的计数器值为零，那么在等式左边一定有一个相同的三元组元素，出现在 $S_0$ 集合中；如果 $\mathsf{RLog}$ 中的某个元素的计数器值大于零，那么这个元素一定出现在等式左边的 $\mathsf{WLog}$ 中，这符合 **条件(5)**。

注意到，等式右边来自于 $S_n$ 中的每一个元素，可能出现在左边的 $S_0$ 中，这意味着该元素所对应的内存单元从未被读取过；$S_n$ 集合元素也可能出现在 $\mathsf{WLog}$ 中，这意味着该元素的计数器值等于最后一次内存单元计数器的更新值。

大家读到这里，可能有一个疑问，为什么我们不关心 $S_n$ 的正确性，而只需要它的存在性？

回顾下上面的例子，我们假设 $S_4$ 中存在 $t^*\not\in \vec{t}$，于是根据上面的等价关系，我们可以得到等式右边必然存在一个读取日志 $\mathsf{RLog}_j=(a^*, t^*, c^*)$，那么根据前面的推理，必然存在一个$\mathsf{RLog}_{j-l}=(a^*, t^*, 0)$，从而可以得出矛盾，然后推翻假设。

最后我们分析下 Prover 和 Verifier 的输入。对于 Verifier 而言，$S_0$ 是 Public input，这样 Verifier 可以验证 **条件(1)**，Verifier 要求 Prover 提供 $\vec{c}'$ 向量，从而构造 $S_n$，验证 **条件(2)**。此外 Public inputs 还要包括承诺 $Commit(\vec{a})$， $Commit(\vec{t})$ 和 $Commit(\vec{f})$，以便 Verifier 同态地验证 Multiset 等价关系。而日志集合 $\{\mathsf{RLog}_j\}, \{\mathsf{WLog}_j\}$ 由 Prover 构造，并给出其中计数器部分的承诺 $Commit(\vec{c})$，从而允许 Verifier 来验证正确性条件 (3)。而 Verifier 也可以根据 $Commit(\vec{a})$， $Commit(\vec{t})$ 和 $Commit(\vec{f})$ 还有 $Commit(\vec{c})$，同态地构造出 $\{\mathsf{RLog}_j\}, \{\mathsf{WLog}_j\}$ 的承诺。

接下来我们利用 Memory-in-the-head 的思路，设计一个 PIOP 协议。

## 构造 Lookup Argument

我们把这三个集合 $S_0, \{R_j\}, \{W_j\}$  看成是三个三列矩阵：

所有的矩阵列向量都编码为多项式。其中 $S_0$ 矩阵的三列记为 $S_i(X)$, $t(X)$ 与 $S_c(X)$，这里注意
在 $S_0$ 中的 val 一列必须等于表格向量 $\vec{t}$。矩阵 $S_n$ 为虚拟机的终止状态，由于虚拟机内存为只读内存，因此 idx 和 val 两列保持不变，但是内存单元计数器被更新到了 $\vec{c}'$，编码为 $S'_c(X)$。

$$
\small
\begin{array}{ll}
S_0=
\begin{array}{|c|c|c|}
\hline
idx & val & cnt\\
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
addr & val & cnt\\
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
idx & val & cnt\\
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
addr & val & cnt\\
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

日志矩阵 $R$ 的第一列为读取的地址序列，必须等于地址向量 $\vec{a}$，第二列为读取的值，等于 $\vec{f}$，而第三列 $\vec{c}$ 为 Prover 维护的计数器向量，编码为 $R_c(X)$。再看下矩阵 $W$，其每一行为一条内存更新日志，其中第三列为更新后的计数器值，这个值编码为 $W_c(X)$，并应该满足下面的约束：

$$
W_c=R_c(X)+1
$$

Offline Memory Checking 的一个关键技巧是，我们可以把上面的五个正确性条件转换成下面的 Multiset 等价约束：

$$
  \{S_0(i, t_i, 0)\}_{i\in[n]} \cup \{W_j(a_j, f_j, c_j+1)\}_{j\in[m]} = \{S_n(i, t_i, c'_i)\}_{i\in[n]} \cup \{R_j(a_j, f_j, c_j)\}_{j\in[m]}
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

我们可以再使用两个 Verifier 提供的随机挑战数 $Y=\beta$ 与 $Z=\gamma$ ，来把上面的多项式等价关系归结到两个 Grand Product 之间的等价关系。而 Grand Product Argument，我们可以采用 Plonk 协议中的 Grand Product 子协议来完成。

### 协议描述

公共输入：
1. $C_t = \mathsf{Commit}(\vec{t})$ ，$|\vec{t}| = n$
2. $C_f = \mathsf{Commit}(\vec{f})$， $|\vec{f}| = m$
3. $C_a = \mathsf{Commit}(\vec{a})$， $|\vec{a}| = m$

#### 第一轮

Prover 模拟内存读取流程得到终状态 $S_m=\{i, t_i, c^\mathsf{final}_i\}_{i\in[m]}$，得到 $\{R_{j}\}_{j\in[m]}$

$$
\begin{split}
R_{j} &= \{a_j, f_j, c_j\}, \qquad {j\in[m]} \\
W_{j} &= \{a_j, f_j, c_j+1\}, \qquad {j\in[m]} \\
\end{split}
$$

Prover 计算 $\{c_j\}_{j\in[m]}$ 的承诺 $C_c=\mathsf{Commit}(\{c_j\})$，
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

Prover 计算  $S^{\mathsf{init}}(X)$ $S^{\mathsf{final}}(X)$

$$
\begin{split}
S^{\mathsf{init}}_i &= i + \beta \cdot t_i + \beta^2\cdot 0 - \gamma \\
S^{\mathsf{final}}_i &= i + \beta \cdot t_i + \beta^2\cdot c^{\mathsf{final}}_i - \gamma \\
\end{split}
$$

Prover 和 Verifier 利用 Grand Product Argument 来证明下面的等式：

$$
(\prod_i S^{\mathsf{init}}_i)\cdot (\prod_j R_j) = (\prod_i S^{\mathsf{final}}_i)\cdot (\prod_j W_j)
$$

#### 验证

Verifier 计算 $C_R, C_W, C_S^{\mathsf{init}}, C_S^{\mathsf{final}}$，

$$
C_S^{\mathsf{init}} = C_I + \beta \cdot C_t - \gamma\cdot [1] \\
C_R = C_a + \beta\cdot C_f + \beta^2\cdot C_c - \gamma\cdot [1] \\
C_W = C_a + \beta\cdot C_f + \beta^2\cdot (C_c+[1]) - \gamma\cdot [1] \\
$$

这里 $C_I = \mathsf{Commit}(0,1,\ldots,n-1)$

## 理解 Offline Memory Checking 

与 Plookup, Caulk/Caulk+, flookup, Baloo, CQ 相比，用 Memory-in-the-head 的思想来证明 Lookup 确实是一个巧妙且直观的想法。不过我们会问，他们有哪些差别？

这一节，我们从 Plookup 的角度出发，换一个角度来理解 Offline memory checking。

我们先假设 $\vec{f}$ 中不存在重复元素，那么我们可以采用 Vanishing Form 的方式来编码 $\vec{f}$ 与 $\vec{t}$ 为多项式，即：

$$
f(X) = (X-f_0)(X-f_1)(X-f_2)\cdots(X-f_{m-1})
$$

$$
t(X) = (X-t_0)(X-t_1)(X-t_2)\cdots(X-t_{n-1})
$$

Prover 只需要证明下面的等式，就可以证明 $\vec{f}\subset\vec{t}$：

$$
\exists q(X), \quad t(X) = f(X)\cdot q(X)
$$

但是如果 $\vec{f}$ 中存在重复元素，那么用 Vanishing Form 编码的多项式就不满足上面的等式约束了。为了修补这个方案，我们要为表格引入一个新的列，即计数器列，当 $\vec{f}$ 中出现重复元素时，我们可以通过计数器列来区分不同的元素。比如 $\vec{f}$ 中有两次对 $t_0$ 的查询，那么我们可以重新定义 $\vec{f}$:

$$
\vec{f}^* = [(t_0, 0), (t_0, 1), (t_1, 0)]
$$

$$
\vec{t}^* = [(t_0, 0), (t_1, 0)]
$$


这样，我们仍然可以用 Vanishing Form 的方式来约束 $\vec{f}^*$ 与 $\vec{t}^*$ 的关系：

$$
(X-t_0-0\beta)(X-t_1-0\beta)\cdot p(X) = (X-t_0-0\beta)(X-t_0-1\beta)(X-t_1-0\beta)\cdot q(X)
$$

其中 $\beta$ 为 Verifier 提供的随机挑战数，用来合并表格 $\vec{t}$ 和 $\vec{f}$ 的两列。

加了计数器列之后，我们发现等式左边的 $\vec{t}$ 也需要补充一些元素（多项式 $p(X)$），这样等式左右两边才能平衡。显然这里，我们需要补上 $p(X)=X-t_0-1\beta$，才能保证上式左右相等（这里 $q(X)=1$）。

换句话说，我们需要在左边补上那些由于计数器累加产生的重复表项，记为 $p(X)$。但是这个 $p(X)$ 不能由 Prover 提供。为防止 Prover 作弊，等式左边的 $p(X)$ 必须由 Verifier 来提供。

那么接下来，我们面临的问题是，Verifier 并不清楚哪些 $f_i$ 重复，并且也不知道重复了几次。这个问题该如何解决？

我们可以把表格 $\vec{t}^*$ 升级成一个可以自动扩展的表格，每次出现一个对 $t_j$ 重复查询的记录， $f^*_i=(t_j, c_i)$，表格 $\vec{t}^*$ 就会自动扩展出一个新的元素，记为 $(t_j, c_i+1)$，供下一次 lookup 使用，这样我们就可以让 Verifier 直接在原始表格上添加 $m$ 个元素，正好对应 $\vec{f}$ 的元素，但是计数器自增一。这样，等式左边的多项式 $p(X)$ 就可以由 Verifier 自行构造：

$$
p_i = (f_i, c_i+1),\qquad 
p(X) = \prod_{i\in[m]} \big(X- f_i - (c_i+1)\cdot\beta\big)
$$

等式右边的多项式 $q(X)$ 会有哪些元素呢？$q(X)$ 恰好包含所有的 $t_i$ 元素，其中曾经没有被查询过的计数器为零，剩下被查询过的项的计数器恰好等于 $t_i$ 被重复 lookup 次数。这样多项式 $q(X)$ 也可以由Prover 提供计数器的值，然后 $\vec{q}$ 的承诺可以由 Verifier 自行构造，$C_q=[t(X)] + \beta\cdot[c(X)]$。

于是我们得到了下面的等式约束：

$$
\{(t_i, 0)\}_{i\in[n]}\cup \{(f_i, c_i+1)\}_{i\in[m]} = \{(f_i, c_i)\}_{i\in[m]}\cup \{(t_i, c_i')\}_{i\in[n]}
$$

下面我们证明上面的等式保证了 $\vec{f}$ 中的每一个元素都是 $\vec{t}$ 中的元素。

我们用反证法，假如存在一个 $f_i\not\in \vec{t}$，那么根据上面的等式，一定存在一个计数器 $(f_i, k)$ 出现在等式的左边。这时候如何 $k=0$，那么 $f_i=t_i$，与假设矛盾。那么这时候可以断定 $k>0$，那么等式右边一定存在一个 $(f_i, k-1)$，才会让左边出现 $(f_i, k)$；同理可推，左边一定存在一个 $(f_i, k-1)$，那么右边一定会出现一个 $(f_i, k-2)$。以此类推，我们一定可以得到：等式左边会出现 $(f_i, 0)$，于是 $f_i=t_i$，这又与初始初始假设矛盾。

这个思路与 Memory-in-the-head 几乎相同，除了我们不考虑表格的索引问题，依次我们可以构造一个 Unindexed Lookup Argument。

### Plookup 思路

回忆下 Plookup 的方案，对于 $\vec{f}$ 和 $\vec{t}$，如果我们要证明 $\vec{f}\subset \vec{t}$，那么 Prover 需要引入一个中间向量 $\vec{s}$，长度为 $n+m$。它是 $\vec{f}\cup\vec{t}$ 的一个重新排序，按照 $\vec{t}$ 中原有项的顺利进行排列。然后 Prover 证明下面的 Multiset 等价关系：

$$
\{(s_i, s_{i+1})\}\overset{?}{=}\{(f_i, f_i)\}\cup\{(t_i, t_{i+1})\}
$$

这个方案和 Memory-checking 相比，Multiset 约束等式两边的集合元素数量都为 $m+n$，但 Plookup 需要多引入一个中间辅助向量 $\vec{s}$，而后者并不需要。

## Sumcheck-bsed Grand Product Argument

Lasso 基于 Sumcheck，这一节我们介绍一个如何利用 Sumcheck 来实现 Grand Product Argument。这个方案出自 Quark Paper[SL20]。

假设我们要计算下面的连乘：

$$
p = \prod_{i=0}^{n-1} v_i
$$

并且我们有向量 $\vec{v}$ 的 MLE 多项式，$\tilde{v}(\vec{X})$ 以及承诺，$C_v=\mathsf{Commit}(\vec{v})$。那么我们可以通过下面的协议来证明 $v$ 的正确性。

我们引入一个辅助向量 $\vec{f}$，它的长度是 $\vec{v}$ 的两倍，它编码了如下的内容：

$$
\begin{split}
\vec{f}^{(lo)} &= (v_0, v_1, \ldots, v_{n-1}) = \vec{v}\\
\vec{f}^{(hi)} &= (v_0v_1, v_2v_3, \ldots, v_0v_1v_2v_3, \ldots, p, 0) = \vec{v}\\
\end{split}
$$

其中 $\vec{f}^{(lo)}$ 是 $\vec{v}$ 的前半部分，$\vec{f}^{(hi)}$ 是 $\vec{v}$ 的后半部分。
后半部分实际上是 $\vec{v}$ 的连乘计算过程的中间结果。
我们可以想象一棵计算连乘 $\vec{v}$ 的二叉树，叶子层是 $\vec{f}^{(lo)}$，第二层是  $\vec{v}$ 中两两相乘的结果，数量为 $n/2$，依次类推，倒数第二层有两个节点，分别是 $\vec{v}$ 中前半部分元素的连乘结果，和 $\vec{v}$ 中后半部分元素的连乘结果，最后一层就是根节点，正是 $\vec{v}$ 的连乘结果 $p$。而向量 $\vec{f}^{(hi)}$ 正好按层数顺序记录这棵树的中层节点的值，最后一个元素 $f_{2n-1}=0$，是为了保证 $\vec{f}^{(hi)}$ 的长度是 $n$。

辅助向量 $\vec{f}$ 的 MLE 多项式满足下面的几个约束关系。首先是 $\vec{f}$ 的前半部分拷贝自 $\vec{v}$，因此：

$$
\tilde{f}(0, \vec{X}) = \tilde{v}(\vec{X})
$$

其次是 $\vec{f}$ 的后半部分的倒数第二个元素是 $p$，因此：

$$
\tilde{f}(1, 1,\ldots, 1, 0) = p
$$

最后是 $\vec{f}$ 的后半部分是 $\vec{v}$ 的连乘中间结果，$\vec{f}$ 还要满足下面的递归关系：

$$
\tilde{f}(1, \vec{X}) = \tilde{f}(\vec{X}, 0) \cdot \tilde{f}(\vec{X}, 1)
$$

我们可以举个简单的例子来理解这个递推关系，假设 $\vec{v}=(0,1,2,3,4,5,6,7)$, 这里 $n=8$。下面的表格列出来了 $\vec{f}$ 的递推关系：

$$
\begin{array}{|c|c|c|c|}
\vec{X} & \tilde{f}(\vec{X}, 0) & \tilde{f}(\vec{X}, 1) & \tilde{f}(1, \vec{X})\\
000: & v_0 & v_1 & v_0v_1\\
001: & v_2 & v_3 & v_2v_3 \\
010: & v_4 & v_5 & v_4v_5\\
011: & v_6 & v_7 & v_6v_7\\
100: & v_0v_1 & v_2v_3 & v_0v_1v_2v_3\\
101: & v_4v_5 & v_6v_7 & v_4v_5v_6v_7\\
110: & v_0v_1v_2v_3 & v_4v_5v_6v_7 & p\\
111: & p & 0 & 0\\
\end{array}
$$

可以看出，最右边一列正好是 $\vec{f}^{(hi)}$，保存了二叉树所有的中间计算结果。并且上面表格的每一行都满足递推关系，包括最后一行，因为 $\vec{f}$ 最后一个元素故意设置为零，因此最后一行也满足递推关系。

我们利用 MLE $\tilde{g}$把上面的连乘关系转化为求和关系：

$$
\tilde{g}(\vec{X}) = \sum_{\vec{x}\in\{0,1\}^{\log{n}}}(\tilde{f}(1, \vec{x})-\tilde{f}(\vec{x}, 0)\cdot \tilde{f}(\vec{x}, 1))\cdot \tilde{eq}(\vec{x}, \vec{X})
$$

如果 $\vec{f}$ 是正确的，那么上面的 MLE 多项式在 $X=\{0,1\}^{\log{n}}$ 处都为 0，即它应该是一个零多项式。那么把随机挑战数 $\vec{\tau}$ 代入上式，可得

$$
\tilde{g}(\vec{\tau})=0=\sum_{\vec{x}\in\{0,1\}^{\log{n}}}(\tilde{f}(1, \vec{x})-\tilde{f}(\vec{x}, 0)\cdot \tilde{f}(\vec{x}, 1))\cdot \tilde{eq}(\vec{x}, \vec{\tau})
$$

由于等式右边是一个求和式，我们可以用 sumcheck 协议来把等式归约到下面的约束关系：

$$
v' = (\tilde{f}(1, \vec{\rho})-\tilde{f}(\vec{\rho}, 0)\cdot \tilde{f}(\vec{\rho}, 1))\cdot \tilde{eq}(\vec{\rho}, \vec{\tau})
$$

其中 $v'$ 是求和项的折叠之和，$\vec{\rho}$ 是 Verifier 提供的随机挑战数。
Prover 要提供 $\tilde{f}$ 在多个点的求值以及证明：

$$
\begin{split}
v_1 & = \tilde{f}(1, \vec{\rho}) \\
v_2 & = \tilde{f}(\vec{\rho}, 0)  \\
v_3 & = \tilde{f}(\vec{\rho}, 1)  \\
v_4 & = \tilde{f}(0, \vec{\rho}) \\
p & = \tilde{f}(\vec{1},0)\\
\end{split}
$$

Verifier 验证下面的公式：

$$
v'\overset{?}{=}(v_1-v_2\cdot v_3)\cdot \tilde{eq}(\vec{\rho}, \vec{\tau})
$$

$$
v_4 \overset{?}{=} \tilde{v}(\rho)
$$


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