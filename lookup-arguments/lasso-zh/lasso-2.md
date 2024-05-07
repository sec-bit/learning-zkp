# 理解 Lasso (二)：稀疏向量与 Tensor 结构

本文我们介绍一个基于 Sumcheck 的「稀疏多项式承诺方案」 Spark，这个方案最早出自 [Spartan] 证明系统。Spark 利用了稀疏向量的结构，可以大幅提升 Prover 的效率。Lasso 是在 Spark 的基础上的进一步拓展了对稀疏向量的处理。理解 Spark 是理解 Lasso 的关键。

普通的多项式承诺方案包括两个阶段，一个是承诺（Commitment）阶段，另一个是求值证明（Evaluation Argument）阶段。对于一个 MLE 多项式 $\tilde{g}\in\mathbb{F}[X_0,X_1,\ldots, X_{n-1}]^{\preceq 1}$，求值点 $\vec{u}\in\mathbb{F}^{n}$，以及运算结果 $v=\tilde{g}(\vec{u})$，那么多项式承诺计算如下：

$$
\mathsf{cm}(g) \leftarrow \mathsf{PCS.Commit}(\tilde{g})
$$

在求值证明阶段，Prover 可以向 Verifier 证明多项式 $\tilde{g}$ 在某一个指定点 $\vec{u}$ 的运算结果为 $v$：

$$
\pi_{g,v} \leftarrow \mathsf{PCS.Eval}(\mathsf{cm}(g), \vec{u}, v; \tilde{g})
$$

Verifier 可以验证求值证明 $\pi_{g,v}$ ：

$$
\mathsf{Accept}/\mathsf{Reject} \leftarrow \mathsf{PCS.Verify}(\mathsf{cm}(g), \vec{u}, v, \pi_v)
$$

如果 $\tilde{g}$ 是一个稀疏的多项式，意味着它在 Boolean HyperCube 上的运算结果中多数的值都为零，那么我们能否利用这个特点，来设计一个针对稀疏多项式更高效的多项式承诺方案？

下面我们演示如何构造 Spark 多项式承诺。不过请记住，Spark 仍然需要基于一个普通的多项式承诺方案。换句话说，Spark 协议是将一个稀疏的 MLE 多项式的求值证明「归约」到多个普通的 MLE 多项式的求值证明，但后者这些 MLE 多项式的大小被大幅减少。

## 1. 稀疏向量的编码

我们考虑一个长度为 $N=2^n$ 的稀疏向量 $\vec{g}=(g_0,g_1,\ldots,g_{N-1})$ 是一个 MLE 多项式 $\tilde{g}(\vec{X})$ 在 Boolean HyperCube $\{0,1\}^n$ 上的取值。记住 $\vec{g}$ 是一个稀疏的向量，其中除了 $m$ 个非零元素之外其余值都为零。

先回忆下 MLE 多项式 $\tilde{g}(\vec{X})$ 的定义：

$$
\tilde{g}(\vec{X}) = \sum_{i=0}^{N-1}g_i\cdot \tilde{eq}_i(\vec{X})
$$

其中 $\tilde{eq}_i(\vec{X})=\tilde{eq}(\mathsf{bits}(i), \vec{X})$ 是 MLE Lagrange 多项式。$\tilde{eq}(\vec{X}, \vec{Y})$ 定义如下：

$$
\tilde{eq}(\vec{X}, \vec{Y})=\prod_{i=0}^{n-1}(X_iY_i+(1-X_i)(1-Y_i))
$$

如果直接使用一个普通的 MLE 多项式承诺方案来证明多项式求值， $\tilde{g}(\vec{u})=v$，由于 $\tilde{g}(\vec{X})$ 是一个关于 $N$ 项的求和公式，那么很显然 Prover 要至少花费 $O(N)$ 的计算量来遍历每一个求和项。

如果给定一个求值点 $\vec{X}=\vec{u}=(u_0, u_1, \ldots, u_{n-1})$，那么所有的 $\tilde{eq}_i(\vec{u}), i\in[0, N)$ 就构成了一个长度为 $N$ 的向量，记为 $\vec{\lambda}$：

$$
\vec{\lambda} = \big(\tilde{eq}_0(\vec{u}), \tilde{eq}_1(\vec{u}), \tilde{eq}_2(\vec{u}), \ldots, \tilde{eq}_{N-1}(\vec{u})\big)
$$

别忘记稀疏向量 $\vec{g}$ 中仅有 $m$ 个非零元素。举个例子，比如 $N=16, n=4, m=4$，即 $\vec{g}$ 向量中仅有四个非零值：

$$
\vec{g} = (0,0, g_2, 0, 0, 0, 0, g_7, 0, g_9, 0,0,0,0, g_{14},0)
$$

那么我们可以换用一种稠密的方式来表示 $\vec{g}$：

$$
\mathsf{DenseRepr}(\vec{g}) = \big((2, g_2), (7, g_7), (9, g_9), (14, g_{14})\big)
$$

可以看出，向量 $\vec{g}$ 的稠密表示是一个长度仅为 $m$ 的向量，其每一个元素为非零元素位置和非零元素值的二元组。我们再把上面二元组向量中的位置值单独记为 $\vec{k}=(k_0, k_1, \ldots, k_{m-1})$ 向量，把元组中非零的 $g_i$ 记为 $\vec{h}=(h_0, h_1, \ldots, h_{m-1})$ 向量：

$$
\begin{split}
\vec{h} &= (g_2,g_7,g_9,g_{14}) \\
\vec{k} &=(2,7,9,14) \\
\end{split}
$$

那么 $\vec{g}$ 的稠密表示可以写成：

$$
\mathsf{DenseRepr}(\vec{g}) = \big((k_0, h_0), (k_1, h_1), \ldots, (k_{m-1}, h_{m-1}) \big)
$$

然后 MLE 多项式 $\tilde{g}(\vec{X})$ 在 $\vec{u}$ 点的求值等式可以改写为：

$$
\tilde{g}(\vec{u}) = \sum_{i=0}^{m-1}h_i\cdot
 \tilde{eq}_{k_i}(\vec{u}) = \sum_{i=0}^{m-1}h_i\cdot\lambda_{k_i}
$$

注意上面这个等式中的求和项的个数仅为 $m$。这意味着在给定 $\vec{h}$ 和 $\vec{\lambda}$ 的情况下，我们成功地把 $\tilde{g}(\vec{X})$ 的求值运算从 $O(N)$ 降到了 $O(m)$。接下来的问题是 Prover 如何向 Verifier 证明求值过程用到了正确的 $h_i$ 和 $\lambda_{k_i}$？

对于一个多项式承诺方案，求值证明的公开输入里面包括了 $\vec{g}$ 向量的承诺，但是上面的求和式需要用到辅助向量 $\vec{h}$，$\vec{k}$ 和 $\vec{\lambda}$。其中 $\vec{\lambda}$ 向量可以通过求值点 $\vec{u}$ 计算得到，其中每个元素为 $\lambda_i=\tilde{eq}_i(\vec{u})$，而求值点 $\vec{u}$ 为公开输入，因此 Verifier 可以公开计算 $\vec{\lambda}$ 向量或者公开验证。但 Verifier 并不能由 $\vec{g}$ 向量的承诺 来直接得到 $\vec{h}$ 和 $\vec{k}$ 这两个向量的信息。因此，我们需要把 $\vec{h}$ 和 $\vec{k}$ 的承诺来替代公开输入中的 $\vec{g}$ 向量的承诺。

换句话说，我们采用 $\vec{h}$ 和 $\vec{k}$ 来作为稀疏向量的 $\vec{g}$ 的编码，并利用一个普通的多项式承诺方案来计算 $\mathsf{cm}(\vec{h})$ 和 $\mathsf{cm}(\vec{k})$ ，并把它们作为多项式求值证明的承诺（做为公开输入）。

## 2. 借助 $\vec{e}$ 的 Sumcheck

我们需要引入一个长度为 $m$ 的辅助向量 $\vec{e}=(e_0, e_1, \ldots, e_{m-1})$，它的每一个元素 $e_i=\lambda_{k_i}$：

$$
\vec{e} = \Big(\tilde{eq}_{k_0}(\vec{u}), \tilde{eq}_{k_1}(\vec{u}),\ldots, \tilde{eq}_{k_{m-1}}(\vec{u})\Big)
$$

这样 $\tilde{g}(\vec{X})$ 在 $\vec{u}$ 点的求值等式等价于下面的求和等式:

$$
\tilde{g}(\vec{u}) = \sum_{i=0}^{m-1}\tilde{h}(\mathsf{bits}(i))\cdot \tilde{e}(\mathsf{bits}(i))
$$

其中 $\tilde{e}(\vec{X})$ 是一个编码了 $\vec{e}$ 的 MLE 多项式，$\tilde{h}(\vec{X})$ 是关于 $\vec{h}$ 的 MLE 多项式

$$
\tilde{e}(\vec{X}) = \sum_{i=0}^{m-1}e_i\cdot \tilde{eq}_i(\vec{X})
\qquad 
\tilde{h}(\vec{X}) = \sum_{i=0}^{m-1}h_i\cdot \tilde{eq}_i(\vec{X})
$$

如果 Prover 要证明上面的求和式，首先提供 $\vec{e}$ 的承诺 $\mathsf{cm}(\vec{e})$ 给 Verifier， 然后通过接下来的两部分来完成证明。

第一部分证明是 Prover 利用 Sumcheck 协议，把 $\tilde{g}(\vec{u})$ 的求值证明规约到下面的等式

$$
v' \overset{?}{=} \tilde{h}(\vec{\rho})\cdot \tilde{e}(\vec{\rho})
$$

其中 $v'$ 为 Sumcheck 协议对 $m$ 个求和项进行折叠运算后的结果，而 $\vec{\rho}$ 为 Sumcheck 运行过程中 Verifier 产生的随机折叠因子。因为 Sumcheck 过程需要 $\log{m}$ 轮，所以 $\vec{\rho}$ 的长度为 $\log{m}$。

接下来 Prover 怎么证明上面的等式呢？ 在求值证明之前，Verifier 已经从公开输入中得到了 $\vec{h}$, $\vec{e}$ 两个向量的承诺，分别为 $\mathsf{cm}(\vec{h})$ 与 $\mathsf{cm}(\vec{e})$，那么到这一步，Prover 和 Verifier 可以再利用普通的 MLE 多项式承诺方案来完成两个 Evaluation Argument，即分别证明： $\tilde{h}(\vec{\rho})=v_h$ 与 $\tilde{e}(\vec{\rho})=v_e$ 的正确性，因为这两个向量长度均为 $m$，因此 Prover 产生这两个 Evaluation Argument 的计算量为 $O(m)$。最后 Verifier 验证 $v \overset{?}{=} v_h\cdot v_e$ 完成第一部分的证明。

第二部分证明是 Prover 证明 $\vec{e}$ 向量关于 $\vec{\lambda}$, $\vec{u}$ 与 $\vec{k}$ 的正确性，这就需要用到前文介绍过的 Offline Memory Checking 方法：Prover 只要证明 $\vec{e}$ 向量中的每一个元素都是从 $\vec{\lambda}$ 向量（看成是内存）中读取出来的即可。这样 Prover 总的计算量为 $O(m+N)$。

## 3. 使用 Memory Checking 证明 $\vec{e}$ 的正确性

辅助向量 $\vec{e}$ 的正确性证明正是 Indexed Lookup Argument：

$$
\forall i\in [0, m), e_i=\lambda_{k_i}
$$

借助 Memory Checking 协议，我们把整个 $\vec{\lambda}$ 向量（公开向量）看成是一段内存，Prover 证明 $\vec{e}$ 向量依次读取自内存 $\vec{\lambda}$，读取的位置为 $\vec{k}$。Prover 可以在 $O(m+N)$ 的计算量内完成上面的证明。

$$
\mathsf{MemChecking}(\mathsf{cm}(\vec{e}), \mathsf{cm}(\vec{\lambda}), \mathsf{cm}(\vec{k}); \vec{e}, \vec{\lambda}, \vec{k_i})
$$

结合前文的定义，这里 $\vec{e}$ 为查询向量 $\vec{f}$，$\lambda$ 为表格向量 $\vec{t}$，而 $\vec{k}$ 为位置向量 $\vec{a}$。

但还有一个问题，$\vec{\lambda}$ 的承诺 $\mathsf{cm}(\vec{\lambda})$ 怎么产生？向量元素 $\lambda_{i}=\tilde{eq}_i(\vec{u})$，其定义中含有一个求值阶段才出现的公开输入 $\vec{u}$，因此不能在 $\tilde{g}$ 的承诺阶段中出现，也无法出现在 $\tilde{g}(\vec{X})$ 求值证明的公开输入中，一般情况多项式承诺方案的公开输入为 $(\mathsf{cm}(\vec{g}), \vec{u}, \tilde{g}(\vec{u}))$。如果由 Prover 计算 $\mathsf{cm}(\vec{\lambda})$ 的话，那么 Prover 需要额外证明承诺的正确性。

幸运的是，$\vec{\lambda}$ 向量具有一定内部的结构，虽然它的长度为 $N$，但在给定 $\vec{u}$ 的情况下，它的插值多项式 $\tilde{\lambda}(\vec{X})$ 可以在 $O(\log{N})$ 的时间内进行求值计算，于是这样一来 Prover 可以不需要提供 $\mathsf{cm}(\vec{\lambda})$，而是让 Verifier 在验证过程中自行计算 $\tilde{\lambda}(\vec{X})$ 在某一点的取值。我们观察下 $\tilde{\lambda}(\vec{X})$ 的定义：

$$
\tilde{\lambda}(\vec{X})= \tilde{eq}(\vec{X}, \vec{u})
$$

容易检验，对于任意的 $i\in[0, N)$，

$$
\lambda_i = \tilde{\lambda}(\mathsf{bits}(i)) = \tilde{eq}(\mathsf{bits}(i), \vec{u}) = \prod_{j=0}^{\log{N}-1}(i_ju_j+(1-i_j)(1-u_j))
$$

上面等式最右边是一个 $\log{N}$ 项的乘积，其中每一个因子只需要常数次的加法和乘法。接下来我们稍微修改下前文中的 Offline Memory Checking 协议，把公开输入中的 $\mathsf{cm}(\vec{\lambda})$ 替换为 $\vec{u}$，并且让 Verifier 自行计算 $\tilde{\lambda}(\vec{X})$ 的值。


### Memory Checking 协议描述

公共输入：
1. $C_e = \mathsf{cm}(\vec{e})$， $|\vec{e}| = m$
2. $C_k = \mathsf{cm}(\vec{k})$， $|\vec{k}| = m$
3. $\vec{u}$, $|\vec{u}| = n = \log{N}$

#### 第一轮

Prover 计算 $S_m$, $\{R_{j}\}_{j\in[m]}$，$\{W_{j}\}_{j\in[m]}$

$$
\begin{split}
S_m &=\{(i, \lambda_i, c^\mathsf{final}_i)\}_{i\in[m]} \\
R_{j} &= (k_j, e_j, c_j), \qquad {j\in[m]} \\
W_{j} &= (k_j, e_j, c_j+1), \qquad {j\in[m]} \\
\end{split}
$$

Prover 计算并发送计数器的承诺 $C_c=\mathsf{cm}(\{c_j\})$，$C^\mathsf{final}_c=\mathsf{cm}(\{c^{\mathsf{final}}_i\})$

#### 第二轮

Verifier 发送挑战数 $\beta, \gamma$

Prover 计算 $\{R_j\}$, $\{W_j\}$, 

$$
\begin{split}
R_j &= k_j +\beta\cdot e_j + \beta^2\cdot c_j -\gamma \\
W_j &= k_j +\beta\cdot e_j + \beta^2\cdot (c_j+1) -\gamma \\
\end{split}
$$

Prover 计算  $\{S_i^{\mathsf{init}}\}$ 与 $\{S_i^{\mathsf{final}}\}$

$$
\begin{split}
S^{\mathsf{init}}_i &= i + \beta \cdot \lambda_i + \beta^2\cdot 0 - \gamma \\
S^{\mathsf{final}}_i &= i + \beta \cdot \lambda_i + \beta^2\cdot c^{\mathsf{final}}_i - \gamma \\
\end{split}
$$

Prover 和 Verifier 利用基于 Sumcheck 的 Grand Product Argument 来证明下面的等式：

$$
(\prod_{i=0}^{N-1} S^{\mathsf{init}}_i)\cdot (\prod_{j=0}^{m-1} R_j) = (\prod_{i=0}^{N-1} S^{\mathsf{final}}_i)\cdot (\prod_{j=0}^{m-1} W_j)
$$

Grand Product Argument 证明最后会归约到对多个 MLE 多项式的求值证明，也就是对 $\tilde{S}^{\mathsf{init}}(\vec{X})$，$\tilde{S}^{\mathsf{final}}(\vec{X})$，$\tilde{R}(\vec{X})$，$\tilde{W}(\vec{X})$ 的求值证明。这些证明可以归约到 $\tilde{I}(\vec{X}), \tilde{k}(\vec{X}), \tilde{e}(\vec{X}), \tilde{c}(\vec{X}), \tilde{c}^{\mathsf{final}}(\vec{X})$ 与 $\tilde{\lambda}(\vec{X})$ 的求值证明。注意我们前面提到过， Verifier 不需要 $\tilde{\lambda}(\vec{X})$ 的承诺求值证明，他可以自行计算 $\tilde{\lambda}(\vec{X})$ 在任意点的求值。因为该多项式的求值计算量仅为 $O(\log{N})$，不影响 Verifier 的简洁性（Succinctness）。

进一步，任何计算过程仅为 $O(\log{N})$ 的 MLE 多项式，Prover 也不必要一定计算它们的承诺，只要把计算任务交给 Verifier 就好。这样 Verifier 仍然保持 SNARK 的特性，同时也提高了 Prover 的效率，省去了计算承诺和产生求值证明的工作量。前提是，这一类 MLE 多项式需要具有一种特殊的内部结构，我们后文会把它们归到一个特殊的分类：MLE-Structured Vector。

对于 Prover 而言，仍然需要在证明过程中构造 $\vec\lambda$，通过动态规划算法，这需要 $O(N)$ 的计算量。

$$
\vec{\lambda} = \big(\tilde{eq}_0(\vec{r}), \tilde{eq}_1(\vec{r}), \tilde{eq}_2(\vec{r}), \ldots, \tilde{eq}_{N-1}(\vec{r})\big)
$$

## 4. 求值证明协议细节

#### 1. 承诺阶段：

Prover 要计算下面两个承诺：

1. $\mathsf{cm}(\vec{h})$：稀疏向量 $\vec{g}$ 中的非零元素向量 $\vec{h}$ 的承诺
2. $\mathsf{cm}(\vec{k})$：$\vec{g}$ 中的所有非零元素在 $\vec{g}$ 中的位置向量 $\vec{k}$ 的承诺

#### 2. 求值证明阶段：

公共输入：

1. 多项式的承诺 $(\mathsf{cm}(\vec{h}), \mathsf{cm}(\vec{k}))$
2. 求值点 $\vec{u}$，以及运算结果 $v=\tilde{g}(\vec{u})$

第一轮：

1. Prover 计算 $\vec{\lambda}$，作为内存模拟
2. Prover 计算 $\vec{e}$， 并发送承诺 $\mathsf{cm}(\vec{e})$，作为 memory 顺序读取出的内容

第二轮：Prover 与 Verifier 执行 Offline Memory Checking 协议，证明

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

Prover 在 Memory-checking 协议中的性能开销为 $O(m+N)$，因为内存的大小为 $N$，读取序列长度为 $m$；在 Sumcheck 协议中为 $O(m)$。因此 Prover 总的计算开销为 $O(m+N)$。

这样一个稀疏多项式承诺方案其实并不理想，因为 Prover 的计算量仍然与 $N$ 线性有关。我们希望能够进一步减少 Prover 的计算量，这就需要进一步探索 $\vec{\lambda}$ 的内部结构。

## 5. 向量 $\vec{e}$ 二维分解

为何 $\tilde{\lambda}(\vec{X})$ 的求值计算量仅为 $O(\log{N})$? 因为向量 $\vec{\lambda}$ 具有一种特殊的结构——Tensor Structure，也就是它可以拆分成多个短向量的 Tensor Product。简化起见，我们试着把 $\lambda_i$ 按照下面的方法拆分成两部分的乘积：

$$
\begin{split}
\lambda_i &= \tilde{eq}(\mathsf{bits}(i), \vec{u}) \\
& =\prod_{j=0}^{n-1}\big(\mathsf{bits}(i)_j\cdot u_j+(1-\mathsf{bits}(i)_j)\cdot(1-u_j)\big) \\
& = \prod_{j=0}^{n/2}\big(\mathsf{bits}(i)_j\cdot u_j+(1-\mathsf{bits}(i)_j)\cdot(1-u_j)\big) \cdot \prod_{j=n/2+1}^{n-1}\big(\mathsf{bits}(i)_j\cdot u_j+(1-\mathsf{bits}(i)_j)\cdot(1-u_j)\big) \\
& = \tilde{eq}(\mathsf{bits}^{(\mathsf{high})}(i), (u_0, u_1, \ldots, u_{n/2})) \cdot \tilde{eq}(\mathsf{bits}^{(\mathsf{low})}(i), (u_{n/2+1}, \ldots, u_{n-1}))
\end{split}
$$

这里 $i_0=\mathsf{bits}^{(\mathsf{high})}(i)$ 和 $i_1=\mathsf{bits}^{(\mathsf{low})}(i)$ 是把 $i$ 的二进制位拆分成相等的两段所表示的数值。举个例子，比如 $i=(13)_{10}$ 是一个十进制数，它的二进制表示为
$\mathsf{bits}(i)=(1101)_2$。我们可以把它拆成高二位与低二位，分别为 $i_0=(11)_2$ 和 $i_1=(01)_2$，那么 $i_0=3, i_1=1$。我们引入一个新的「拼接记号」， $i = i_0\parallel i_1$ 表示 $i$ 的二进制位为其高位和低位两个数的二进制位的拼接，按照 Big-endian 的方式。比如 $(1101)_2 = (11)_2 \parallel (01)_2$。不难验证，拼接操作满足性质： $i\parallel j=i+ \sqrt{N}\cdot j$。

按照上面的分解方法，我们可以分解 $\lambda_{13}$ 为两个值的乘积：

$$
\lambda_{13} = \tilde{eq}((11)_2, (u_0, u_1)) \cdot \tilde{eq}((01)_2, (u_2, u_3))
$$

对于长度为 $N$ 的 $\vec{\lambda}$ 向量中的所有元素 $\lambda_i$，我们可以把其中每一个元素都按照相同拆分方式进行分解：

$$
\begin{split}
\lambda_0 &= \tilde{eq}((00)_2, (u_0, u_1)) \cdot \tilde{eq}((00)_2, (u_2, u_3)) \\
\lambda_1 &= \tilde{eq}((00)_2, (u_0, u_1)) \cdot \tilde{eq}((01)_2, (u_2, u_3)) \\
\lambda_2 &= \tilde{eq}((00)_2, (u_0, u_1)) \cdot \tilde{eq}((10)_2, (u_2, u_3)) \\
\lambda_3 &= \tilde{eq}((00)_2, (u_0, u_1)) \cdot \tilde{eq}((11)_2, (u_2, u_3)) \\
\vdots & \\
\lambda_{15} &= \tilde{eq}((11)_2, (u_0, u_1)) \cdot \tilde{eq}((11)_2, (u_2, u_3)) \\
\end{split}
$$

我们进而把这 16 个元素排成一个 $4\times 4$ 的矩阵，每一个单元格的值 $\lambda_i$ 都等于它对应的行向量元素和列向量元素的乘积。

$$
\begin{array}{c|cccc}
& eq_{0}(u_0, u_1) & eq_{1}(u_0,u_1) & eq_{2}(u_0,u_1) & eq_{3}(u_0,u_1)\\
\hline
eq_{0}(u_2,u_3) & eq_{0\parallel 0}(u_0,u_1,u_2,u_3) & eq_{1\parallel 0}(u_0,u_1,u_2,u_3)& eq_{2\parallel 0}(u_0,u_1,u_2,u_3) & eq_{3\parallel 0}(u_0,u_1,u_2,u_3)\\
eq_{1}(u_2,r_3) & eq_{0\parallel 1}(u_0,u_1,u_2,u_3) & eq_{1\parallel 1}(u_0,u_1,u_2,u_3)& eq_{2\parallel 1}(u_0,u_1,u_2,u_3) & eq_{3\parallel 1}(u_0,u_1,u_2,u_3)\\
eq_{2}(u_2,u_3) & eq_{0\parallel 2}(u_0,u_1,u_2,u_3) & eq_{1\parallel 2}(u_0,u_1,u_2,u_3)& eq_{2\parallel 2}(u_0,u_1,u_2,u_3) & eq_{3\parallel 2}(u_0,u_1,u_2,u_3)\\
eq_{3}(u_2,u_3) & eq_{0\parallel 3}(u_0,u_1,u_2,u_3) & eq_{1\parallel 3}(u_0,u_1,u_2,u_3)& eq_{2\parallel 3}(u_0,u_1,u_2,u_3) & eq_{3\parallel 3}(u_0,u_1,u_2,u_3)\\
\end{array}
$$

如果把上面表格的第一行的元素组成向量，记为 $\vec{\lambda}^{(x)}$ ，第一列记为 $\vec{\lambda}^{(y)}$：

$$
\begin{split}
\vec{\lambda}^{(x)} &= \big(eq_{0}(u_0,u_1), eq_{1}(u_0,u_1), eq_{2}(u_0,u_1), eq_{3}(u_0,u_1)\big) \\
\vec{\lambda}^{(y)} &= \big(eq_{0}(u_2,u_3), eq_{1}(u_2,u_3), eq_{2}(u_2,u_3), eq_{3}(u_2,u_3)\big)
\end{split}
$$

那么 $\vec{\lambda}$ 向量看成是两个长度为 $\sqrt{N}$ 的向量的 Tensor Product：

$$
\vec{\lambda} = \vec{\lambda}^{(x)} \otimes \vec{\lambda}^{(y)}
$$

回到我们关注的向量 $\vec{e}$，其中每一个元素 $e_i$ 也就可以看成是两个数值的乘积 $e_i=e_i^{(x)}\cdot e_i^{(y)}$，其中 $e_i^{(x)}$ 来自于 $\vec{\lambda}^{(x)}$，另一个 $e_i^{(y)}$ 来自于 $\vec{\lambda}^{(y)}$。

这相当于我们把整个 $\vec{e}$ 向量分解到了一个二维空间中，它的值等于横坐标和纵坐标值的乘积。那么我们可以继续采用 Offline Memory Checking 的思路来证明 $\vec{e}$ 的正确性，这次我们需要采用二维的 Offline Memory Checking 协议。更直白点说，我们需要采用两次 Offline Memory Checking 协议来证明 $\vec{e}$ 的正确性，每一个 $e_i$ 对应到两个值的乘积，它们分别读取自 $\vec{\lambda}^{(x)}$ 和 $\vec{\lambda}^{(y)}$：

$$
\begin{split}
\mathsf{MemChecking}(\mathsf{cm}(\vec{e}^{(x)}), \vec{\lambda}^{(x)}, \mathsf{cm}(\vec{k}^{(x)}); \vec{e}^{(x)}, \vec{k}^{(x)}) \\
\mathsf{MemChecking}(\mathsf{cm}(\vec{e}^{(y)}), \vec{\lambda}^{(y)}, \mathsf{cm}(\vec{k}^{(y)}); \vec{e}^{(x)},\vec{k}^{(y)}) \\
\end{split}
$$

于是稀疏多项式 $\tilde{g}(\vec{X})$ 的求值等式可以改写为：

$$
\tilde{g}(\vec{u})=\tilde{g}(\vec{u}_0, \vec{u}_1) = \sum_{i=0}^{m-1}\tilde{h}(\mathsf{bits}(i))\cdot
 \tilde{e}^{(x)}(\mathsf{bits}(i)) \cdot  \tilde{e}^{(y)}(\mathsf{bits}(i))
$$

其中

$$
\begin{split}
\vec{e}^{(x)} &= \Big(\tilde{eq}_{k^{(x)}_0}(\vec{u}_0), \tilde{eq}_{k^{(x)}_1}(\vec{u}_0),\ldots, \tilde{eq}_{k^{(x)}_{m-1}}(\vec{u}_0)\Big)\\
\vec{e}^{(y)} &= \Big(\tilde{eq}_{k^{(y)}_0}(\vec{u}_1), \tilde{eq}_{k^{(y)}_1}(\vec{u}_1),\ldots, \tilde{eq}_{k^{(y)}_{m-1}}(\vec{u}_1)\Big)\\
\end{split}
$$

其中 $k^{(x)}_i, k^{(y)}_i\in(0, 1, \ldots, \log{N}/2)$ 为非零元素 $h_i$ 在二维矩阵中的行列坐标。这样我们可以把求值协议中的 Offline Memory Checking 子协议调用两次，但是内存的大小被大幅缩小到了 $\sqrt{N}=2^{n/2}$。看下前面的例子， $N=16, n=4, m=4$，$\vec{g}$ 向量中仅有四个非零值：

$$
\vec{g} = (0,0, g_2, 0, 0, 0, 0, g_7, 0, g_9, 0,0,0,0,g_{14},0)
$$

向量 $\vec{h}$ 为非零向量：

$$
\vec{h}= (g_2, g_7, g_9, g_{14})
$$

这时候，我们可以采用二维坐标 $(k^{(x)}_i, k^{(y)}_i)$ 来标记 $h_i$ 在 $\vec{g}$ 矩阵中的位置，标记矩阵中的行和列：

$$
\begin{split}
(k^{(x)}_0, k^{(y)}_0) &= (2, 0) \\
(k^{(x)}_1, k^{(y)}_1) &= (3, 1) \\
(k^{(x)}_2, k^{(y)}_2) &= (1, 2) \\
(k^{(x)}_3, k^{(y)}_3) &= (2, 3) \\
\end{split}
$$

我们把其中行坐标向量记为 $\vec{k}^{(x)}$，列坐标向量记为 $\vec{k}^{(y)}$，那么 $\tilde{g}(\vec{u})$ 可以表示为

$$
\begin{split}
\tilde{g}(u_0, u_1, u_2, u_3) &= \sum_{0\le i\lt 4} h_i\cdot \tilde{eq}(\mathsf{bits}(k^{(x)}_i)), (u_0,u_1)) \cdot \tilde{eq}(\mathsf{bits}(k^{(y)}_i), (u_2,u_3)) \\ 
&= \sum_{0\le i\lt 4} h_i\cdot e^{(x)}_i \cdot e^{(y)}_i
\end{split}
$$

经过 Sumcheck 协议之后，上述等式可以被归约到：

$$
v' = \tilde{h}(\vec\rho)\cdot \tilde{e}^{(x)}(\vec\rho) \cdot \tilde{e}^{(y)}(\vec\rho)
$$

然后 Prover 再提供三个 MLE 多项式在 $\vec{\rho}$ 点的取值，$(v_h, v_x, v_y)$ 的求值证明。

在这个二维的求值协议中，Prover 的计算开销就从上一节的 $O(m+N)$ 降低到了 $O(m+2\sqrt{N})$。

下面我们给出完整的二维稀疏多项式承诺方案。

## 6. 二维稀疏多项式承诺 Spark

利用上面的思路，我们把稀疏向量 $\vec{g}$ 重新排列成一个 $\sqrt{N}\times \sqrt{N}$ 的二维矩阵 $G$。为了排版清晰，我们引入符号 $l=\sqrt{N}$：

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

### 6.1. 承诺阶段：

Prover 要计算下面两个承诺：

1. $C_h=\mathsf{cm}(\vec{h})$：稀疏向量 $\vec{g}$ 中的非零元素向量 $\vec{h}$ 的承诺
2. $C_x=\mathsf{cm}(\vec{k}^{(x)})$：$\vec{h}$ 中的所有非零元素在矩阵 $G$ 中的行坐标构成的向量 $\vec{k}^{(x)}$ 的承诺
3. $C_y = \mathsf{cm}(\vec{k}^{(y)})$：$\vec{h}$ 中的所有非零元素在矩阵 $G$ 中的列坐标构成的向量 $\vec{k}^{(y)}$ 的承诺

令 $\mathsf{Spark.Commit}(\vec{g})\to\mathsf{cm}_g^{(\mathsf{spark})}=\big(C_h, C_x, C_y\big)$，这个三元组承诺我们用符号 $\mathsf{cm}_{g}^{(\mathsf{spark})}$ 表示。

### 6.2. 求值证明阶段：

$\pi^{\mathsf{(spark)}}_g\leftarrow\mathsf{Spark.Eval}((C_h, C_x, C_y), \vec{u}, v; (\vec{h},\vec{x},\vec{y}))$：

公共输入：

1. 多项式的承诺 $\mathsf{cm}_{g}^{(\mathsf{spark})}=(C_h, C_x, C_y)$
2. 求值点 $\vec{u}$，这个点可以拆分为两个子向量 $\vec{u}=\vec{u}_x \parallel \vec{u}_y$，其中 $|\vec{u}_x|=|\vec{u}_y|=n/2$
3. 以及运算结果 $v=\tilde{g}(\vec{u})$


第一轮：

1. Prover 计算 $\vec{\lambda}^{(x)}= \{eq_i(\vec{u}_x)\}_{i\in[0, l)}$，作为 $\mathsf{mem}_x$ 内存
2. Prover 计算 $\vec{\lambda}^{(y)}= \{eq_i(\vec{u}_y)\}_{i\in[0, l)}$，作为 $\mathsf{mem}_y$ 内存
2. Prover 计算 $\vec{e}^{(x)}$ 与 $\vec{e}^{(y)}$， 作为分别从内存 $\mathsf{mem}_x$ 与 $\mathsf{mem}_y$ 读取出的内容，并发送承诺 $\mathsf{cm}(\vec{e}^{(x)})$ 与 $\mathsf{cm}(\vec{e}^{(y)})$

第二轮：Prover 与 Verifier 执行两次 Offline Memory Checking 协议，证明 $\mathsf{cm}(\vec{e}^{(x)})$ 与 $\mathsf{cm}(\vec{e}^{(y)})$ 的正确性：

$$
\begin{split}
\mathsf{MemChecking}(\mathsf{cm}(\vec{e}^{(x)}), \vec{u}_x, \mathsf{cm}(\vec{k}^{(x)}); \vec{e}^{(x)}, \vec{k}^{(x)}) \\
\mathsf{MemChecking}(\mathsf{cm}(\vec{e}^{(y)}), \vec{u}_y, \mathsf{cm}(\vec{k}^{(y)}); \vec{e}^{(x)}, \vec{k}^{(y)}) \\
\end{split}
$$

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
2. $v_x=\tilde{e}^{(x)}(\vec{\rho})$，求值证明为 $\pi_x=\mathsf{PCS.Eval}(C_x,\vec{\rho}, v_x; \vec{k}^{(x)})$
3. $v_y=\tilde{e}^{(y)}(\vec{\rho})$，求值证明为 $\pi_y=\mathsf{PCS.Eval}(C_y,\vec{\rho}, v_y; \vec{k}^{(y)})$

验证： Verifier 验证 $\pi_h$ ，$\pi_x$  与 $\pi_y$ 的有效性，并验证下面的等式：

$$
v' \overset{?}{=} v_h\cdot v_x\cdot v_y
$$

### 6.3. 性能分析



## 8. Tensor 结构 (TODO)

如果我们可以把 $\vec{e}$ 分解到二维空间，那么能否分解到更高维的空间？比如 $\vec{f}$ 的长度为 $2^{30}$，那么把它排成二维矩阵，比如 $2^{15}\times 2^{15}$，矩阵的长宽还是较大。如果把 $\vec{f}$ 重新排列成一个立方体，然后同样把 $\tilde{eq}_i(\vec{r})$ 拆分成三段，这样我们可以把 Offline Memory Checking 的 Prover 开销进一步降低到 $O(N^{1/3})$，也就是 $2^{10}$。这个分解的灵活性来源于 $\vec{\lambda}$ 的结构特性，即 一个具有 Tensor Structure 的向量可以用不同的 Tensor Product 分解方式。理论上，我们可以把 $\vec{f}$ 分解成 $\log{N}$ 个长度为 $2$ 的短向量的 Tensor Product。不过实践中，我们只需要将其分解到 $N^{1/c}$ 即可处理超长的向量。

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

本文介绍了 Tensor Structure 的概念，利用这个结构，我们可以把稀疏向量映射到一个二维空间中进行编码，然后我们基于这个结构，可以构造一个稀疏向量的多项式承诺方案。

##  References

- [Spartan] [Spartan: Efficient and general-purpose zkSNARKs without trusted setup
](https://eprint.iacr.org/2019/550) by Srinath Setty.
- [Lasso] [Unlocking the lookup singularity with Lasso](https://eprint.iacr.org/2023/1216) by Srinath Setty, Justin Thaler and Riad Wahby.
- [Jolt] [Jolt: SNARKs for Virtual Machines via Lookups](https://eprint.iacr.org/2023/1217) by Arasu Arun, Srinath Setty and Justin Thaler.
- [PLONK] [PLONK: Permutations over Lagrange-bases for Oecumenical Noninteractive arguments of Knowledge](https://eprint.iacr.org/2019/953.pdf) by Ariel Gabizon, Zachary J. Williamson and Oana Ciobotaru.
- [Plookup] [plookup: A simplified polynomial protocol for lookup tables](https://eprint.iacr.org/2020/315) by Ariel Gabizon and Zachary J. Williamson.
- [Caulk] [Caulk: Lookup Arguments in Sublinear Time](https://eprint.iacr.org/2022/621) by Arantxa Zapico, Vitalik Buterin,Dmitry Khovratovich, Mary Maller, Anca Nitulescu and Mark Simkin
- [Caulk+] [Caulk+: Table-independent lookup arguments](https://eprint.iacr.org/2022/957) by Jim Posen and Assimakis A. Kattis.
- [Baloo] [Baloo: Nearly Optimal Lookup Arguments](https://eprint.iacr.org/2022/1565) by Arantxa Zapico, Ariel Gabizon, Dmitry Khovratovich, Mary Maller and Carla Ràfols.
- [CQ] [cq: Cached quotients for fast lookups](https://eprint.iacr.org/2022/1763) by Liam Eagen, Dario Fiore and Ariel Gabizon.