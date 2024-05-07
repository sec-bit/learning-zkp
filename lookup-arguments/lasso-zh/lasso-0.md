# 理解 Lasso（零）：带索引的查询证明

假设我们有一个公开的表格向量 $\vec{t}$，长度为 $N$，和一个查询向量 $\vec{f}$，长度为 $m$，我们可以利用 Lookup Argument 来证明下面的 lookup 关系：

$$
\forall i\in [0,m), f_i \in \vec{t}
$$

上面这个 Lookup Argument 定义被 Lasso 论文称为 Unindexed Lookup Argument。因为这个定义只保证了 $f_i$ 在 $\vec{t}$ 中，但是并不保证 $f_i$ 出现在 $\vec{t}$ 中某个特定的位置。

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

对于这个表格，我们可以看到表格中的任意两行可以交换，而并不影响表格所要表达的语义。因为表格的每一行同时包含了 XOR 运算的输入和输出。

那么我们问，能不能采用一个单列表格来表达这个 XOR 运算？

## Indexed Lookup Arguments

其中一个思路是这样的，我们只采用一列表格来表示「XOR 运算的输出」，即 $A\oplus B$，而用表格的行索引来代替两个运算的输入（Oprands）。比如，我们在第 0 行放上 $00$，因为 $00 = 00\oplus 00$，等号右边为行数的 4bit 编码， $0000$，其高位 $00$ 表示 $A$，低位 $00$ 表示 $B$。

又比如，表格的第 5 行（记住行数从零计数）为 $00$，因为 $01\oplus 01=00$，而行索引 $5$ 的二进制表示可以按位拆分为两个二进制数 $01$ 和 $01$，即 $(01 01)_{(2)}=5_{(10)}$。 可见，这个单列表格的大小仍然是 16 行。但是与上面的 XOR 表格不同，这个表格的各行是不允许打乱顺序的，下面是单列的 XOR 表格：

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

单列有序表格的好处是，我们只需要用一个多项式对其编码。此外，Lasso 还进一步探索了单列表格可能具有的内部结构，探索如何把单列的大表格拆分成多个小表格，从而提高证明效率。Jolt 展示了如何利用表格的内部结构，来编码 RISC-V 的完整指令运算。

基于单列有序的表格，Lasso 论文定义了一类新的 Lookup Argument，称之为 Indexed Lookup Argument：

$$
\forall i\in [0,m), f_i = t_{a_i}
$$

其中 $\vec{a}=(a_0,a_1,\ldots, a_{m-1})$ 为一组索引值，表示每一个查询 $f_i$ 在表格 $\vec{t}$ 中出现的位置。

对于一个 Indexed Lookup Argument，公共输入为三个向量的承诺：

- 表格向量的承诺 $\mathsf{cm}(\vec{t})$
- 查询向量的承诺 $\mathsf{cm}(\vec{f})$
- 索引向量的承诺 $\mathsf{cm}(\vec{a})$

证明的关系为：

$$
\mathcal{R}_{indexed-lkup}=\{(\mathsf{cm}(\vec{t}), \mathsf{cm}(\vec{f}),\mathsf{cm}(\vec{a});\vec{t},\vec{f},\vec{a})\mid \forall i\in [0, m), f_i = t_{a_i} \}
$$

出现在Plonk 协议中的 Plookup，以及后续的 Caulk/Caulk+，FLookup, Logup，cq 都属于 Unindexed Lookup Arguments。基于 Unindexed Lookup Arguments，我们同样可以构建 Indexed Lookup Argument。常见的有两个方案。

首先，如果 Unindexed Lookup Arguments 支持表格列的加法同态，那么我们可以为表格增加一列，作为 Index 列，然后通过 Verifier 给出一个随机数 $\eta$，然后将 Index 列和 原表格列（或多列）做一个 Random Linear Combination 合并为一列。比如一个单列表格为 $\vec{t}=(t_0,t_1,\ldots, t_{N-1})$，那么我们可以通过 $\eta$ 构造一个混有 Index 的新表格列：

$$
\vec{t}_I' = (t_0, t_1+\eta,t_2+2\eta,\ldots,t_{N-1}+\eta\cdot(N-1))
$$

比如 Plookup，Caulk/Caulk+，Baloo，与 cq 都支持表格承诺的加法同态。但是对于 fLookup 等不支持加法同态的 Lookup Arguments，我们可以找到一个值，$\kappa>\mathsf{max}\{t_i, i\in[0,N)\}$ ，然后通过 $\kappa$ 把索引列合并到原表格列：

$$
\vec{t}_I'' = (t_0, t_1+\kappa,t_2+2\kappa,\ldots,t_{N-1}+\kappa\cdot(N-1))
$$

不过，Prover 还要额外证明表格列中的每一项 $t_i<\kappa$，这需要 $N$ 个 Range Arguments。

从一个 Indexed Lookup Argument 可以更容易地得到 Unindexed Lookup Argument，只需要把索引向量的承诺从公共输入中移除即可，在协议的开头，Prover 补充发送这个索引承诺即可。
 
## Various Lookup Arguments From the Lasso Paper

本系列文章后续将描述总共四个不同的 Indexed Lookup Arguments 协议：

- Lookup Arguments based on Offline Memory Checking
- Lookup Arguments based on Spark
- Lookup Arguments based on Surge
- Lookup Arguments based on Sparse-dense Sumcheck

第一个协议基于经典的 Offline Memory Checking，改进自 Spartan 论文中的 Memory Checking 协议。支持 Indexed Lookup 与 UnIndexed Lookup。通过 Offline Memory Checking，我们将 Lookup 关系归结到一个（只读）内存读取的虚拟机执行关系的合法性。

第二个协议 Spark 源于 Spartan 论文。为了处理查询向量中可能出现的重复表格项，我们引入一个矩阵 $M$ 来作为表格选择器，采用 Matrix-vector Multiplication 公式来证明 Lookup 关系：

$$
M\vec{t} = \vec{f}
$$

其中 $\vec{t}$ 为表格，长度为 $n$，$\vec{f}$ 为 lookup 记录，长度为 $m$。这个核心公式来自 [Baloo] 论文。

矩阵 $M\in\mathbb{F}^{m \times N}$ 充当了选择器的角色。它的每一行都是一个 Unit Vector，即每一个行向量中，只有一个元素为 $1$，其余元素均为零。显然矩阵 $M$ 中包含大量的零，如果我们直接用多项式对 $M$ 中的全部元素进行粗暴地编码，那么这相当于对于一个长度为 $O(m\cdot N)$的稀疏向量编码，浪费严重。

举个例子，比如 $n=8, m=4$，查询向量 $\vec{f}$ 定义为：

$$
\vec{f} = (t_2, t_7, t_{4}, t_{2})
$$

那么 $\vec{M}$ 矩阵满足下面的等式：

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

如果我们可以利用 $M$ 矩阵的稀疏性，即 $M$ 中仅包含有 $m$ 个非零元素，那么我们可以构造更有效率的 Lookup Argument 方案。[Spartan] 论文提出了针对稀疏矩阵的多项式承诺方案，使得其 Evaluation Argument 的证明时间仅与 $m$ 有关。

Spark 协议的另一个特点是利用了 $\tilde{eq}(\vec{X},\vec{Y})$ 的 Tensor 结构。如果表格也具有类似的结构，那么意味着被查询的表格可以拆解成多个维度上的短向量，那么也就意味着 Prover 和 Verifier 不再需要处理一个很大的表格（如果表格长度 $N>2^{64}$），而只需要承诺和证明多个短向量（作为子表格）即可。第三个协议 Surge 正是这样一个可以证明某一类支持子表格拆解的 Lookup Argument。

支持巨大的表格，比如 $N=2^{128}$，并不是只有拆解子表格这一种办法，如果表格满足另外一种特性 MLE-Structured，即表格多项式 $\tilde{t}(\vec{X})$ 的求值运算时间复杂度为 $O(\log{N})$，那么我们可以不需要拆分表格，也不需要让 Prover 承诺表格（表格太大，承诺的计算也无法完成），而是在协议中以「惰性计算」的方式（Lazy On-demand）来临时计算表格的每一项（以表格项的 Index 作为输入，计算表格项 $\tilde{t}(i)$）。这是最后一个 Lookup Argument 的核心思想，被称为 Generalized Lasso。

Generalized Lasso 利用一个所谓的 Sparse-dense Sumcheck 协议来利用查询向量（关于表格向量）的稀疏性，使 Prover 在证明过程中「惰性计算」 $\vec{t}$ 中那些仅被查询到的表格项，这样就做到了证明时间复杂度只与查询的数量有关，而与表格长度无关。并且与 cq 等协议相比，Generalized Lasso 并不需要昂贵的预处理。当然，Generalized Lasso 仅能处理满足 MLE-Structured 的一类表格，而非是一个通用的 Lookup Argument。

总结下，Lasso 论文把 Lasso 可以处理的表格分为三类。

- Unstructured but small
- Decomposable
- Non-decomposable but MLE-structured

##  References

- [Lasso] [Unlocking the lookup singularity with Lasso](https://eprint.iacr.org/2023/1216) by Srinath Setty, Justin Thaler and Riad Wahby.
- [Jolt] [Jolt: SNARKs for Virtual Machines via Lookups](https://eprint.iacr.org/2023/1217) by Arasu Arun, Srinath Setty and Justin Thaler.
- [PLONK] [PLONK: Permutations over Lagrange-bases for Oecumenical Noninteractive arguments of Knowledge](https://eprint.iacr.org/2019/953.pdf) by Ariel Gabizon, Zachary J. Williamson and Oana Ciobotaru.
- [Plookup] [plookup: A simplified polynomial protocol for lookup tables](https://eprint.iacr.org/2020/315) by Ariel Gabizon and Zachary J. Williamson.
- [Caulk] [Caulk: Lookup Arguments in Sublinear Time](https://eprint.iacr.org/2022/621) by Arantxa Zapico, Vitalik Buterin,Dmitry Khovratovich, Mary Maller, Anca Nitulescu and Mark Simkin
- [Caulk+] [Caulk+: Table-independent lookup arguments](https://eprint.iacr.org/2022/957) by Jim Posen and Assimakis A. Kattis.
- [Baloo] [Baloo: Nearly Optimal Lookup Arguments](https://eprint.iacr.org/2022/1565) by Arantxa Zapico, Ariel Gabizon, Dmitry Khovratovich, Mary Maller and Carla Ràfols.
- [CQ] [cq: Cached quotients for fast lookups](https://eprint.iacr.org/2022/1763) by Liam Eagen, Dario Fiore and Ariel Gabizon.