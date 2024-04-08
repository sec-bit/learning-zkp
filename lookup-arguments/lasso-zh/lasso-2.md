# 理解 Lasso (二)：稀疏向量与 Tensor 结构

Lasso 这个名字是 **L**ookup **A**rguments via **S**parse-polynomial-commitments and the **S**umcheck-check protocol, including for **O**versized-tables 的缩写。这里面有三个关键词，

- Sparse Polynomial Commitment
- Sumcheck protocol
- Oversized table

其中第一个关键词为「稀疏多项式承诺」。稀疏多项式是指一个稀疏向量的多项式编码。我们利用向量的稀疏性而改进证明效率。

先看 Lookup 关系可以表示成下面的 Matrix-Vector 乘积

$$
M\vec{t}=\vec{f}
$$

其中 $\vec{t}$ 为表格，长度为 $m$，$\vec{f}$ 为 lookup 记录，长度为 $n$；而选择矩阵 $M\in\mathbb{F}^{N\times m}$ 正是一个稀疏矩阵。它的每一行都是一个 Unit Vector，即向量元素中只有一个 $1$，其余元素均为零。如果我们直接多项式对 $M$ 全部元素进行编码，那么相当于对于一个长度为 $O(m\cdot N)$的向量编码。

举个例子，比如 $N=8, m=4$，$\vec{f}$ 向量为：

$$
\vec{f} = (t_2, t_7, t_{4}, t_{2})
$$

那么 $\vec{M}$ 矩阵为：

$$
\begin{bmatrix}
0 & 0 & 1 & 0 & 0 & 0 & 0 & 0\\
0 & 0 & 0 & 0 & 0 & 0 & 0 & 1\\
0 & 0 & 0 & 0 & 1 & 0 & 0 & 0\\
0 & 0 & 1 & 0 & 0 & 0 & 0 & 0\\
\end{bmatrix}
\begin{bmatrix}
t_0 \\
t_1 \\
t_2 \\
t_3 \\
t_4 \\
t_5 \\
t_6 \\
t_7 \\
\end{bmatrix}
= \begin{bmatrix}
f_0:t_2 \\
f_1:t_7 \\
f_2:t_{4} \\
f_3:t_{2} \\
\end{bmatrix}
$$

如果我们可以利用 $M$ 矩阵的稀疏性，即 $M$ 中仅包含有 $m$ 个非零元素，那么我们可以构造更有效率的 Lookup Argument 方案。

在本文，我们介绍一个基于 Sumcheck 的稀疏多项式承诺方案 Spark，这是 Spartan 证明系统的核心子协议，也是 Lasso 协议的核心。然后我们介绍如何利用 Spark 来实现一个简单的 Lookup argument，其 Prover 计算量仅为 $O(m+N)$，而非 $O(m\cdot N)$。

## 1. 稀疏多项式的编码

先把 Lookup argument 放一边，考虑一个长度为 $N=2^n$ 向量 $\vec{g}=(g_0,g_1,\ldots,g_{N-1})$ 是一个 MLE 多项式 $\tilde{g}(\vec{X})$ 在 Boolean HyperCube $\{0,1\}^n$ 上的取值。我们设定 $\vec{g}$ 是一个稀疏的向量，这意味着 $\vec{g}$ 中除了 $m$ 个非零元素之外其余值都为零。

先回忆下 $\tilde{g}(\vec{X})$ 的定义：

$$
\tilde{g}(\vec{X}) = \sum_{i=0}^{N-1}g_i\cdot \tilde{eq}_i(\vec{X})
$$

如果直接使用一个普通的 MLE 多项式承诺来证明一个多项式求值， $\tilde{g}(\vec{r})=v$，那么很显然 Prover 要至少花费 $O(N)$ 的计算量。

其中 $\tilde{eq}_i(\vec{X})=\tilde{eq}(\mathsf{bits}(i), \vec{X})$ 是 MLE 多项式 Lagrange Basis。$\tilde{eq}(\vec{X}, \vec{Y})$ 定义如下：

$$
\tilde{eq}(\vec{X}, \vec{Y})=\prod_{i=0}^{n-1}(X_iY_i+(1-X_i)(1-Y_i))
$$

如果固定 $\vec{X}=\vec{r}=(r_0, r_1, \ldots, r_{n-1})$，那么根据 $i=0,1,\ldots, N-1$ 的不同， $\tilde{eq}(\mathsf{bits}(i), \vec{X})$ 就构成了一个长度为 $N$ 的向量，记为 $\vec{\lambda}$：

$$
\vec{\lambda} = \big(\tilde{eq}_0(\vec{r}), \tilde{eq}_1(\vec{r}), \tilde{eq}_2(\vec{r}), \ldots, \tilde{eq}_{N-1}(\vec{r})\big)
$$

别忘记 $\vec{g}$ 中仅有 $m$ 个非零元素。举个例子，比如 $N=16, n=4, m=4$，$\vec{f}$ 向量中仅有四个非零值：

$$
\vec{g} = (0,0, g_2, 0, 0, 0, 0, g_7, 0, g_9, 0,0,0,0, g_{14},0)
$$

那么我们把 $\vec{g}$ 中所有的非零向量标记出来，用一种稠密的方式来表示 ：

$$
\big((2, g_2), (7, g_7), (9, g_9), (14, g_{14})\big)
$$

我们把上面这个元组向量中的位置数值抽取出来，记为 $\vec{k}$ 向量，把元组中 $g_i$抽取出来，记为 $\vec{h}$ 向量。 

$$
\begin{split}
\vec{h} &= (g_2,g_7,g_9,g_{14}) \\
\vec{k} &=(2,7,9,14) \\
\end{split}
$$

那么 $\tilde{g}(\vec{X})$ 在 $\vec{r}$ 点的求值等式可以改写为：

$$
\tilde{g}(\vec{r}) = \sum_{i=0}^{m-1}h_i\cdot
 \tilde{eq}_{k_i}(\vec{r}) = \sum_{i=0}^{m-1}h_i\cdot\lambda_{k_i}
$$

注意这个求和式中项的个数仅为 $m$。

### 引入辅助向量 $\vec{e}$

我们再引入一个长度为 $m$ 的辅助向量 $\vec{e}$，它的每一个元素 $e_i=\lambda_{k_i}$：

$$
\vec{e} = \Big(\tilde{eq}_{k_0}(\vec{r}), \tilde{eq}_{k_1}(\vec{r}),\ldots, \tilde{eq}_{k_{m-1}}(\vec{r})\Big)
$$

这样$\tilde{g}(\vec{X})$ 在 $\vec{r}$ 点的求值等式等价于下面的 Inner product 等式:

$$
\tilde{g}(\vec{r}) = \sum_{i=0}^{m-1}h_i\cdot e_i
$$

到此为止，上面的求和等式的右边是一个关于 $m$ 项的求和。于是我们可以采用 Sumcheck 协议，把 $\tilde{g}(\vec{r})$ 的求值证明规约到下面的等式

$$
v = \tilde{h}(\vec{\rho})\cdot \tilde{e}(\vec{\rho})
$$

其中 $\vec{\rho}$ 为 Sumcheck 运行过程中 Verifier 产生的挑战向量，长度为 $\log{m}$。

Prover 如何向 Verifier 证明上面（Sumcheck 归约后）的等式呢？如果求值协议前 $\vec{t}$, $\vec{e}$ 两个向量被承诺过，分别记为 $\mathsf{Commit}(\vec{h})$ 与 $\mathsf{Commit}(\vec{e})$。那么到这一步，Prover 可以利用 MLE 多项式承诺的 Evaluation Argument 来向 Verifier 证明： $\tilde{h}(\vec{\rho})$ 与 $\tilde{e}(\vec{\rho})$ 的值是由 $\mathsf{Commit}(\vec{h})$ 与 $\mathsf{Commit}(\vec{e})$ 所承诺的多项式计算而来。因为这两个被承诺的向量长度均为 $m$，因此这两个 Evaluation argument 的 Prover 计算量可以被降到 $O(m)$。

但是这里遗留了一个问题：向量 $\vec{e}$ 与它的承诺 $\mathsf{Commit}(\vec{e})$ 的正确性如何保证？
因为向量 $\vec{e}$ 中含有 $\vec{r}$ 的值，这是 Verifier 在协议中发送的随机挑战值，所以向量 $\vec{e}$ 只能在 Verifier 发送 $\vec{r}$ 之后，才可以被构造（并计算承诺）。进而 Prover 不可能把 $\mathsf{Commit}(\vec{e})$ 作为 $\tilde{g}$ 的 MLE 承诺的一部分，Prover 只能在 Evaluation argument 协议中，在 Sumcheck 协议之前，临时计算这个承诺。那么 Prover 如何额外证明 $\vec{e}$ 向量被 Prover 正确计算并承诺了呢？

这里我们就需要用到上一节介绍的内容：利用 Memory-in-the-head 的思路来证明：$e_i=\lambda_{k_i}$。我们把整个
$\vec{\lambda}$ 向量看成是一段内存，Prover 证明 $\vec{e}$ 向量读取自内存 $\vec{\lambda}$，读取的位置为 $\vec{k}$：

$$
\forall i\in[m], e_i=\lambda[{k_i}]
$$

注意到 $\vec{\lambda}$ 向量的长度仍为


### 证明 $\vec{e}$ 的正确性

向量 $\vec{\lambda}_r$ 是具有一定内部的结构，虽然它的长度为 $N$，但在给定 $\vec{r}$ 的情况下，它可以在 $O(N)$ 的时间内全部计算出：

$$
\vec{\lambda} = \big(\tilde{eq}_0(\vec{r}), \tilde{eq}_1(\vec{r}), \tilde{eq}_2(\vec{r}), \ldots, \tilde{eq}_{N-1}(\vec{r})\big)
$$

这个算法请参考 [Thaler22] 或 [XZSZ19]。

我们把 $\vec{\lambda}$ 视为一段长为 $N$ 的内存，我们可以利用 Memory-in-the-head 来证明每一个 $e_i$ 项都是从内存 `mem[s[i]]` 中读取出来。

不过证明这个 indexed lookup，仍然需要 Prover 付出 $O(N)$ 的计算量。因为 Memory 的大小为 $\vec{\lambda}$ 的长度。

因此，Prover 总共需要花费 $O(m+N)$ 的计算量。下面我们给出这个协议的细节。

## 2. 稀疏多项式承诺

#### 1. 承诺阶段：

Prover 要计算下面两个承诺：

1. $\mathsf{Commit}(\vec{h})$：稀疏向量 $\vec{g}$ 中的非零元素向量 $\vec{h}$ 的承诺
2. $\mathsf{Commit}(\vec{k})$：$\vec{g}$ 中的所有非零元素在 $\vec{g}$ 中的位置向量 $\vec{k}$ 的承诺

#### 2. 求值证明阶段：

公共输入：

1. 多项式的承诺 $(\mathsf{Commit}(\vec{h}), \mathsf{Commit}(\vec{k}))$
2. 求值点 $\vec{r}$，以及运算结果 $v=\tilde{g}(\vec{r})$

第一轮：

1. Prover 计算 $\vec{\lambda}$，作为内存模拟
2. Prover 计算 $\vec{e}$， 并发送承诺 $\mathsf{Commit}(\vec{e})$，作为 memory 顺序读取出的内容

第二轮：Prover 与 Verifier 执行 Memory-in-the-head 协议，证明

$$
e_i = \vec{\lambda}_{k_i}, \quad \forall i\in [m]
$$

第三轮：Prover 与 Verifier 执行 Sumcheck 协议，证明

$$
v=\sum_{i\in[0,m)}h_i\cdot e_i
$$

并把上面的求和等式归约到 

$$
v' = \tilde{h}(\vec{\rho})\cdot \tilde{e}(\vec{\rho})
$$

其中 $\vec{\rho}$ 为 Verifier 在 Sumcheck 过程中发送的挑战向量。

第四轮：Prover 发送 $(v_h, v_e, \pi_h, \pi_e)$

1. $v_h=\tilde{h}(\vec{\rho})$，求值证明为 $\pi_h$
2. $v_e=\tilde{e}(\vec{\rho})$，求值证明为 $\pi_e$

验证： Verifier 验证 $\pi_h$ 与 $\pi_e$ 的有效性，并验证下面的等式：

$$
v' \overset{?}{=} v_h\cdot v_e
$$

### 性能分析

Prover 性能开销在 Memory-checking 协议中为 $O(N)$，因为内存的大小为 $N$；在 Sumcheck 协议中为 $O(m)$。因此 Prover 总的计算开销为 $O(m+N)$。

如果我们用本小节的 Sparse 承诺方案来计算 Lookup Argument 中的「选择矩阵」 $M$ 的承诺，假设矩阵的尺寸为 $l\times t$，那么把矩阵拍成一维向量后的长度为 $l\cdot t$。选择矩阵中非零元素的个数恰好为矩阵的行数 $l$。将这些数值代入本小节协议的 Prover 开销，我们得到 $O(l + l\cdot t)$。这样，prover 计算量和 lookup 数量和表格项数量的乘积成正比，这显然不满足要求。

## 3. 向量 $\vec{e}$ 的二维坐标

这需要我们进一步探索 $\vec{\lambda}$ 的内部结构。

先上结论：向量 $\vec{\lambda}$ 具有一种特殊的结构——Tensor Structure，也就是它可以拆分成多个短向量的 Tensor Product。简单说，每一个$\lambda_i$ 可以按照下面的方法拆分成两部分的乘积：

$$
\lambda_i = \tilde{eq}_i(\vec{X}) = \tilde{eq}_{i_0\parallel i_1}(\vec{X}_0\parallel\vec{X}_1)
 = \tilde{eq}_{i_0} (\vec{X}_0) \cdot \tilde{eq}_{i_1} (\vec{X}_1)
$$

这里 $i\parallel j$ 表示两个十进制自然数按二进制位拼接在一起的计算得到的自然数，比如 $(2)_{10}\parallel (3)_{10}=(10)_{2}\parallel (11)_{2} = (1011)_{2} = (11)_{10}$。

上面的这个拆分等式我们可以通过 $\tilde{eq}$ 的定义展开来确认：

$$
\begin{split}
\tilde{eq}(\mathsf{bits}(i), \vec{r}) & =\prod_{j=0}^{n-1}\big(\mathsf{bits}(i)_j\cdot r_j+(1-\mathsf{bits}(i)_j)\cdot(1-r_j)\big) \\
& = \prod_{j=0}^{n/2}\big(\mathsf{bits}(i)_j\cdot r_j+(1-\mathsf{bits}(i)_j)\cdot(1-r_j)\big) \cdot \prod_{j=n/2+1}^{n-1}\big(\mathsf{bits}(i)_j\cdot r_j+(1-\mathsf{bits}(i)_j)\cdot(1-r_j)\big) \\
& = \tilde{eq}(\mathsf{bits}^{high}(i), (r_0, r_1, \ldots, r_{n/2})) \cdot \tilde{eq}(\mathsf{bits}^{low}(i), (r_{n/2+1}, \ldots, r_{n-1}))
\end{split}
$$

我们利用 $\vec{\lambda}$ 的内部结构，把它分解为两段长度均为 $\sqrt{N}$ 的内存数组的 Tensor Product，如果这里 $N=16$，那么两片内存长度为 $\sqrt{N}=4$，分别记为 $\vec{\lambda}_{(x)}$ 与 $\vec{\lambda}_{(y)}$

$$
\vec{\lambda} = \vec{\lambda}_{(x)} \otimes \vec{\lambda}_{(y)}
$$

这里 $\vec{\lambda}_{(x)}$ 与 $\vec{\lambda}_{(y)}$ 的定义如下：
$$
\begin{split}
\vec{\lambda}_{(x)} &= \big(eq_{0}(r_0,r_1),  eq_{1}(r_0,r_1),  eq_{2}(r_0,r_1),  eq_{3}(r_0,r_1)\big) \\
\vec{\lambda}_{(y)} &= \big(eq_{0}(r_2,r_3),  eq_{1}(r_2,r_3),  eq_{2}(r_2,r_3),  eq_{3}(r_2,r_3)\big)
\end{split}
$$

我们需要从两段内存中分别读出两个值，$eq_{i}(r_0,r_1)$ 与 $eq_{j}(r_0,r_1)$，并计算它们的乘积，这个值恰好等于 $\lambda_{i\parallel j}$，这里 $i\parallel j=i+ \sqrt{N}\cdot j$。换句话说，我们可以把向量 $\vec{\lambda}$ 拆分成一个矩阵，矩阵中的任何一个单元值都等于它对应的行向量 $\vec{\lambda}_{(x)}$ 和列向量 $\vec{\lambda}_{(y)}$ 中读出来的值的「乘积」。下面我们列出这个矩阵和对应的行列向量之间的乘积关系：

$$
\begin{array}{c|cccc}
& eq_{0}(r_0,r_1) & eq_{1}(r_0,r_1) & eq_{2}(r_0,r_1) & eq_{3}(r_0,r_1)\\
\hline
eq_{0}(r_2,r_3) & eq_{0\parallel 0}(\vec{r}) & eq_{1\parallel 0}(\vec{r})& eq_{2\parallel 0}(\vec{r}) & eq_{3\parallel 0}(\vec{r})\\
eq_{1}(r_2,r_3) & eq_{0\parallel 1}(\vec{r}) & eq_{1\parallel 1}(\vec{r})& eq_{2\parallel 1}(\vec{r}) & eq_{3\parallel 1}(\vec{r})\\
eq_{2}(r_2,r_3) & eq_{0\parallel 2}(\vec{r}) & eq_{1\parallel 2}(\vec{r})& eq_{2\parallel 2}(\vec{r}) & eq_{3\parallel 2}(\vec{r})\\
eq_{3}(r_2,r_3) & eq_{0\parallel 3}(\vec{r}) & eq_{1\parallel 3}(\vec{r})& eq_{2\parallel 3}(\vec{r}) & eq_{3\parallel 3}(\vec{r})\\
\end{array}
$$

这样，Lookup Argument 等式可以改写为：

$$
\tilde{g}(\vec{r})=\tilde{g}(\vec{r}_0, \vec{r}_1) = \sum_{i=0}^{m-1}h_i\cdot
 \tilde{eq}_{x_i}(\vec{r}_0) \cdot  \tilde{eq}_{y_i}(\vec{r}_1)
$$

其中 $x_i\in(0, 1, \ldots, n/2)$，$y_i\in(0,1,\ldots, n/2)$ 为非零元素 $h_i$ 在上面矩阵中的行列坐标。

这样我们可以把 lookup argument 协议中的 Memory-checking 子协议调用两次，但是内存的大小被大幅缩小到了 $\sqrt{N}=n/2$。

举个例子，假设 $N=16, n=4, m=4$，$\vec{g}$ 向量中仅有四个非零值：

$$
\vec{g} = (0,0, g_2, 0, 0, 0, 0, g_7, 0, g_9, 0,0,0,0,g_{14},0)
$$

向量 $\vec{h}$ 为非零向量：

$$
\vec{h}= (g_2, g_7, g_9, g_{14})
$$

这时候，我们可以采用二维坐标 $(x_i, y_i)$ 来标记 $h_i$ 在 $\vec{g}$ 矩阵中的位置，标记矩阵中的行和列：

$$
\begin{split}
(x_0, y_0) &= (0, 2) \\
(x_1, y_1) &= (1, 7) \\
(x_2, y_2) &= (3, 9) \\
(x_3, y_3) &= (4, {14}) \\
\end{split}
$$

我们把其中行坐标向量记为 $\vec{x}$，列坐标向量记为 $\vec{y}$

那么 $\tilde{g}(\vec{r})$ 可以表示为

$$
\begin{split}
\tilde{g}((r_0,r_1), (r_2, r_3)) &= \sum_{i\in[0,m)} h_i\cdot \tilde{eq}(\mathsf{bin}(x_i)), (r_0,r_1)) \cdot \tilde{eq}(\mathsf{bin}(y_i), (r_2,r_3)) \\ 
&= \sum_{i\in[0,m)} h_i\cdot e^{(x)}_i \cdot e^{(y)}_i
\end{split}
$$

其中 $\{e^{(x)}_i\}_{i\in[0,n/2)}$ 和 $\{e^{(y)}_i\}_{i\in[0,n/2)}$ 为 Prover 引入并承诺的辅助向量，代表 Prover 从两段内存 $\vec{\lambda}_{(x)}$ 与 $\vec{\lambda}_{(y)}$ 中顺次读取的值。 经过 Sumcheck 协议之后，上述等式可以被归约到：

$$
v' = \tilde{h}(\rho)\cdot \tilde{e}^{(x)}(\rho) \cdot \tilde{e}^{(y)}(\rho)
$$

这样，Prover 的计算开销就从上一节的 $O(m+N)$ 降低到了 $O(m+2\sqrt{N})$。

## 4. 二维稀疏多项式承诺

利用上面的思路，我们把稀疏向量 $\vec{g}$ 重新排列成一个 $\sqrt{N}\times \sqrt{N}$ 的二维矩阵 G。为了排版清晰，我们引入符号 $l=\sqrt{N}$：

$$
G =
\begin{bmatrix}
g_0 & g_1 & g_2 &\ldots & g_{l-1} \\
g_{l} & g_{l+1} & g_{l+2} &\ldots & g_{2l-1} \\
g_{2l} & g_{2l+1} & g_{2l+2} &\ldots & g_{3l-1} \\
\vdots & \vdots & \vdots & \ddots & \vdots \\
g_{(l-1)l} & g_{(l-1)l+1} & g_{(l-1)l+2} & \ldots & g_{l^2-1} \\
\end{bmatrix}
$$

#### 4.1. 承诺阶段：

Prover 要计算下面两个承诺：

1. $C_h=\mathsf{Commit}(\vec{h})$：稀疏向量 $\vec{g}$ 中的非零元素向量 $\vec{h}$ 的承诺
2. $C_x=\mathsf{Commit}(\vec{x})$：$\vec{h}$ 中的所有非零元素在矩阵 $F$ 中的行坐标构成的向量 $\vec{x}$ 的承诺
3. $C_y = \mathsf{Commit}(\vec{y})$：$\vec{h}$ 中的所有非零元素在矩阵 $F$ 中的列坐标构成的向量 $\vec{y}$ 的承诺

令 $\mathsf{Spark.Commit}(\vec{f})\to\big(C_h, C_x, C_y\big)$

#### 4.2. 求值证明阶段：

$\pi^{\mathsf{(spark)}}_h\leftarrow\mathsf{Spark.Eval}((C_h, C_x, C_y), \vec{r}, v; (\vec{h},\vec{x},\vec{y}))$：

公共输入：

1. 多项式的承诺 $(C_h, C_x, C_y)$
2. 求值点 $\vec{r}$，这个点可以拆分为两个子向量 $\vec{r}=\vec{r}_x \parallel \vec{r}_y$
3. 以及运算结果 $v=\tilde{g}(\vec{r})$


第一轮：

1. Prover 计算 $\vec{\lambda}_{(x)}= \{eq_i(\vec{r}_x)\}_{i\in[0, l)}$，作为内存模拟
2. Prover 计算 $\vec{\lambda}_{(y)}= \{eq_i(\vec{r}_y)\}_{i\in[0, l)}$，作为内存模拟
2. Prover 计算 $\vec{e}^{(x)}$ 与 $\vec{e}^{(y)}$， 作为 memory 顺序读取出的内容，并发送承诺 $\mathsf{Commit}(\vec{e}^{(x)})$ 与 $\mathsf{Commit}(\vec{e}^{(y)})$

$$
e^{(x)}_i = \tilde{eq}_{x_i}(\vec{r}_x) \qquad 
e^{(y)}_i = \tilde{eq}_{y_i}(\vec{r}_y)
$$

第二轮：Prover 与 Verifier 执行 Memory-checking 协议，证明 $\mathsf{Commit}(\vec{e}^{(x)})$ 与 $\mathsf{Commit}(\vec{e}^{(y)})$ 的正确性

第三轮：Prover 与 Verifier 执行 Sumcheck 协议，证明下面的等式求和

$$
v=\sum_{i\in[0,m)}h_i\cdot e^{(x)}_i \cdot e^{(y)}_i 
$$

并把求和等式归约到 

$$
v' = \tilde{h}(\vec{\rho})\cdot \tilde{e}^{(x)}(\vec{\rho})\cdot \tilde{e}^{(y)}(\vec{\rho})
$$

其中 $\vec{\rho}$ 为 Verifier 在 Sumcheck 过程中发送的挑战向量，其长度为 $\log{m}$。

第四轮：Prover 发送 $(v_t, v_x, v_y, \pi_t, \pi_x, \pi_y)$

1. $v_h=\tilde{h}(\vec{\rho})$，求值证明为 $\pi_h=\mathsf{PCS.Eval}(C_h,\vec{\rho}, v_h; \vec{h})$
2. $v_x=\tilde{e}^{(x)}(\vec{\rho})$，求值证明为 $\pi_x=\mathsf{PCS.Eval}(C_x,\vec{\rho}, v_x; \vec{x})$
3. $v_y=\tilde{e}^{(y)}(\vec{\rho})$，求值证明为 $\pi_y=\mathsf{PCS.Eval}(C_y,\vec{\rho}, v_y; \vec{y})$

验证： Verifier 验证 $\pi_h$ ，$\pi_x$  与 $\pi_y$ 的有效性，并验证下面的等式：

$$
v' \overset{?}{=} v_h\cdot v_x\cdot v_y
$$

## 5. 构造 Indexed Lookup Argument

上面介绍了 Sparse Polynomial Commitment，现在回到正题 Lookup Argument。下面是本文开头讨论的 Indexed Lookup Argument 等式：

$$
M\vec{t}=\vec{f}
$$

这里 $\vec{t}$ 为表格向量，长度为 $N$，$\vec{f}$ 为查找向量，长度为 $m$，选择矩阵 $M$ 大小为 $m\times N$。

我们引入三个 MLE 多项式 $\tilde{M}(\vec{X},\vec{Y}), \tilde{t}(\vec{Y}), \tilde{f}(\vec{X})$ 来分别编码矩阵 $M$，表格向量 $\vec{t}$，与查找向量 $\vec{f}$， 那么它们满足下面的关系：

$$
\sum_{y\in\{0,1\}^{\log{N}}}\tilde{M}(\vec{X}, \vec{y})\cdot \tilde{t}(\vec{y}) = \tilde{f}(\vec{X})
$$

Verifier 可以发送一个挑战向量 $\vec{r}$，把上面的等式可以归约到：

$$
\sum_{y\in\{0,1\}^{\log{N}}}\tilde{M}(\vec{r}, \vec{y})\cdot \tilde{t}(\vec{y}) = \tilde{f}(\vec{r})
$$

现在， Prover 要向 Verifier 证明：存在一个矩阵 $M$，使得上面的求和等式成立。

我们会想到直接使用 $\log{N}$ 轮的 Sumcheck 协议，把上面的等式归约到一个新的等式：

$$
\tilde{M}(\vec{r}, \vec{\rho})\cdot \tilde{t}(\vec{\rho}) = v'
$$

其中 $v'$ 为求和项折叠之后的值，$\vec{\rho}$ 为 Verifier 在 Sumcheck 协议过程中发送的挑战值。这时 Verifier 要验证上面的等式，就需要 Prover 提供三个 MLE 的求值证明。因为 $M$ 矩阵是一个稀疏矩阵，因此 Prover 可以在协议最开头采用 Spark 来承诺 $\tilde{M}$，然后在 Sumcheck 协议的末尾， Prover 可以花费 $O(m + \sqrt{m\cdot N})$ 的计算量来产生 $\tilde{M}(\vec{r}, \vec{\rho})$ 的 Evaluation 证明。这远好于 Prover 直接采用普通多项式承诺的开销，$O(m\cdot N)$。

### 协议细节

公共输入：

1. $C_t = \mathsf{PCS.Commit}(\vec{t})$ ，$|\vec{t}| = n$
2. $C_f = \mathsf{PCS.Commit}(\vec{f})$， $|\vec{f}| = m$
3. $C^{\mathsf{(spark)}}_M = \mathsf{Spark.Commit}({M})$， $|M| = m\times N$

第一轮：Verifier 发送挑战向量 $\vec{r}$

$$
\sum_{y\in\{0,1\}^{\log{N}}}\tilde{M}(\vec{r}, \vec{y})\cdot \tilde{t}(\vec{y}) = \tilde{f}(\vec{r})
$$

第二轮：Prover 和 Verifier 执行 Sumcheck 协议，把上式归约到

$$
\tilde{M}(\vec{r}, \vec{\rho})\cdot \tilde{t}(\vec{\rho}) = v'
$$

第三轮：Prover 发送 $(v_M, v_t, \pi^{\mathsf{(spark)}}_M, \pi_t)$

1. $(v_M, \pi^{\mathsf{(spark)}}_M)=\mathsf{Spark.Eval}(C_M^{\mathsf{(spark)}}, (\vec{r}, \vec{\rho}), v_M; M)$
2. $(v_t, \pi_t)=\mathsf{PCS.Eval}(C_t, \vec{\rho}, v_t; \vec{t})$

## 6. Tensor 结构

如果 $\vec{f}$ 的长度为 $2^{30}$，那么把它排成二维矩阵，比如 $2^{15}\times 2^{15}$，矩阵的长宽还是较大。我们可以继续改进上面的协议，把 $\vec{f}$ 重新排列成一个立方体，然后同样把 $\tilde{eq}_i(\vec{r})$ 拆分成三段，这样我们可以把 Memory-checking 的 Prover 开销进一步降低到 $O(N^{1/3})$，也就是 $2^{10}$。这个灵活性来源于 $\vec{\lambda}$ 具有一个通用的 「Tensor Structure」，即它可以有一组短向量的 Tensor Product 来产生。理论上，我们可以把 $\vec{f}$ 分解成 $\log{N}$ 个维度的矩阵。不过，通常情况下，我们只需要分解到 $N^{1/c}$ 即可处理超长向量。

例如当 $N=16$ 时，$\vec{\lambda}$ 即可以排列成一个 $4\times 4$ 的二维矩阵，也可以排列成 $2\times 2\times 2\times 2$ 的四维矩阵：

$$
\small
\begin{split}
\vec{\lambda} &= (r_0,1-r_0)\otimes(r_1,1-r_1)\otimes(r_2,1-r_2) \otimes(r_3,1-r_3) \\
&=\Big((r_0,1-r_0)\otimes(r_1,1-r_1)\Big)\otimes \Big((r_2,1-r_2)\otimes(r_3,1-r_3)\Big) \\
&= \Big((r_0r_1,(1-r_0)r_1,r_0(1-r_1),(1-r_0)(1-r_1))\Big) \otimes 
\Big((r_2r_3,(1-r_2)r_3,r_2(1-r_3),(1-r_2)(1-r_3))\Big) \\
\end{split}
$$


我们可以根据 Tensor Product 逐步来推导下：

$$
(r_0, (1-r_0))\otimes(r_1, (1-r_1))= 
\begin{array}{c|cc}
 & r_0 & (1-r_0) \\
 \hline
 r_1 & r_0r_1 & (1-r_0)r_1\\
(1-r_1) & r_0(1-r_1) & (1-r_0)(1-r_1) \\
\end{array}
$$ 

再利用上面的计算结果来计算 $(r_0, (1-r_0))\otimes(r_1, (1-r_1)) \otimes (r_2, (1-r_2))$ 

$$
\small
\begin{array}{c|cc}
 & r_2 & (1-r_2) \\
 \hline
 r_0r_1 & r_0r_1r_2 & r_0r_1(1-r_2)\\
(1-r_0)r_1 & (1-r_0)r_1r_2 & (1-r_0)r_1(1-r_2) \\
r_0(1-r_1) &r_0(1-r_1)r_2& r_0(1-r_1)(1-r_2)\\
(1-r_0)(1-r_1) &(1-r_0)(1-r_1)r_2& (1-r_0)(1-r_1)(1-r_2)\\
\end{array}
$$

其实，许多常见的向量也具备 Tensor Structure，比如 $(1, \alpha, \alpha^2,\ldots, \alpha^{2^n-1})$：

$$
(1, \alpha, \alpha^2,\ldots, \alpha^{2^n-1})= (1,\alpha)\otimes (1,\alpha^2) \otimes (1,\alpha^4)\otimes \cdots\otimes (1, \alpha^{2^{(n-1)}})
$$

## 7. 小结

本文介绍了 Tensor Structure 的概念，利用这个结构，我们可以把稀疏向量映射到一个多维空间中进行编码。然后我们可以构造一个稀疏向量的多项式承诺方案，然后利用这个承诺方案来构造一个 Indexed Lookup Argument。

##  References

- [Lasso] [Unlocking the lookup singularity with Lasso](https://eprint.iacr.org/2023/1216) by Srinath Setty, Justin Thaler and Riad Wahby.
- [Jolt] [Jolt: SNARKs for Virtual Machines via Lookups](https://eprint.iacr.org/2023/1217) by Arasu Arun, Srinath Setty and Justin Thaler.
- [PLONK] [PLONK: Permutations over Lagrange-bases for Oecumenical Noninteractive arguments of Knowledge](https://eprint.iacr.org/2019/953.pdf) by Ariel Gabizon, Zachary J. Williamson and Oana Ciobotaru.
- [Plookup] [plookup: A simplified polynomial protocol for lookup tables](https://eprint.iacr.org/2020/315) by Ariel Gabizon and Zachary J. Williamson.
- [Caulk] [Caulk: Lookup Arguments in Sublinear Time](https://eprint.iacr.org/2022/621) by Arantxa Zapico, Vitalik Buterin,Dmitry Khovratovich, Mary Maller, Anca Nitulescu and Mark Simkin
- [Caulk+] [Caulk+: Table-independent lookup arguments](https://eprint.iacr.org/2022/957) by Jim Posen and Assimakis A. Kattis.
- [Baloo] [Baloo: Nearly Optimal Lookup Arguments](https://eprint.iacr.org/2022/1565) by Arantxa Zapico, Ariel Gabizon, Dmitry Khovratovich, Mary Maller and Carla Ràfols.
- [CQ] [cq: Cached quotients for fast lookups](https://eprint.iacr.org/2022/1763) by Liam Eagen, Dario Fiore and Ariel Gabizon.