# 理解 Lasso (三)：大表格的稀疏查询证明

Lasso 这个名字是 **L**ookup **A**rguments via **S**parse-polynomial-commitments and the **S**umcheck-check protocol, including for **O**versized-tables 的缩写。这里面有三个关键词，

- Sparse Polynomial Commitment
- Sumcheck protocol
- Oversized table

本文继续讨论如何利用 Sparse Polynomial Commitment 来构造 Indexed Lookup Argument。但为了能处理 Oversized Table（比如 $2^{128}$ 这样的表格），需要充分利用表格的内部结构。


## 1. 构造简易 Indexed Lookup Argument

前文介绍了 Sparse Polynomial Commitment，现在回到正题 Lookup Argument。下面是一种 Lookup 关系的表示：

$$
M\vec{t}=\vec{f}
$$

这里 $\vec{t}$ 为表格向量，长度为 $N$，$\vec{f}$ 为查找向量，长度为 $m$，选择矩阵 $M$ 大小为 $m\times N$。

我们引入三个 MLE 多项式 $\tilde{M}(\vec{X},\vec{Y}), \tilde{t}(\vec{Y}), \tilde{f}(\vec{X})$ 来分别编码矩阵 $M$，表格向量 $\vec{t}$，与查找向量 $\vec{f}$， 那么它们满足下面的关系：

$$
\sum_{\vec{y}\in\{0,1\}^{\log{N}}}\tilde{M}(\vec{X}, \vec{y})\cdot \tilde{t}(\vec{y}) = \tilde{f}(\vec{X})
$$

Verifier 可以发送一个挑战向量 $\vec{r}$，把上面的等式可以归约到：

$$
\sum_{\vec{y}\in\{0,1\}^{\log{N}}}\tilde{M}(\vec{r}, \vec{y})\cdot \tilde{t}(\vec{y}) = \tilde{f}(\vec{r})
$$

现在， Prover 要向 Verifier 证明上面的求和等式成立，我们会立即想到使用 $\log{N}$ 轮的 Sumcheck 协议，把上面的等式归约到一个新的等式：

$$
\tilde{M}(\vec{r}, \vec{\rho})\cdot \tilde{t}(\vec{\rho}) = v'
$$

其中 $v'$ 为 $\log{N}$ 个求和项折叠之后的值，$\vec{\rho}$ 为 Verifier 在 Sumcheck 协议过程中发送的挑战值。这时 Verifier 要验证上面的等式，就需要 Prover 提供三个 MLE 的求值证明。因为 $M$ 矩阵是一个稀疏矩阵，因此 Prover 可以在协议最开头采用 Spark 协议来承诺 $\tilde{M}$，然后在 Sumcheck 协议的末尾， Prover 可以花费 $O(m + \sqrt{m\cdot N})$ 的计算量来产生 $\tilde{M}(\vec{r}, \vec{\rho})$ 的 Evaluation 证明。这远好于 Prover 直接采用普通多项式承诺的开销，$O(m\cdot N)$。

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

当然，我们可以通过把 $M$ 整个向量排成一排，得到长度为 $m\times N$ 长的一维向量 $\vec{\mathcal{h}}$，然后把这个向量在一个 $c$ 维的空间中
进行拆分: 

$$
\tilde{M}(\vec{X}^{(D_1)},\ldots, \vec{X}^{(D_c)}) = \sum_{i=0}^{m-1}\mathsf{val}(\mathsf{bits}(i)) \cdot \tilde{eq}(\mathsf{bits}(i), \vec{X}^{(D_1)})\cdot \ldots \cdot \tilde{eq}(\mathsf{bits}(i), \vec{X}^{(D_c)})
$$

然后利用 Spark 协议来达到 $O(m + c\sqrt[c]{m\cdot N})$ 的 Proving Time 复杂度。但是 Prover 在产生 $\pi_t$ 时则需要 $O(N)$ 的计算量。那么总体上，Spark 虽然可以有效降低 Prover 的工作量，但是如果表格尺寸 $N$ 非常大，那么 Prover 仍然需要花费大量的时间来计算表格。那么还能不能更进一步呢？像 Caulk/Caulk+, cq 那样让 Prover 的性能开销变为关于 $N$ 的亚线性复杂度。

Lasso 协议正是朝着这个方向迈出了一大步，它甚至不需要像 cq 那样要实现对完整的大表格做预处理。尽管它不通用，只能针对几类特殊的表格，但不少常见的运算都可以证明。

Lasso 的核心思想是，我们能否把表格向量 $\vec{t}$ 像稀疏向量一样按照多个维度去拆解？如果能像 Tensor Structure 那样，一个巨大的表格可以表示为若干个小表格的运算。这样 Prover 和 Verifier 就可以对多个小表格做 Lookup 证明，那么最终得到的效果就是：看起来我们可以实现一个虚拟的大表格的查询证明。

顺着这个思路往下想，一般情况下表格不可能是稀疏的，不过非稀疏的表格在某些情况下是可以分解的。比如我们在前文提到的异或运算的表格，

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

直觉上，一个2-bit XOR 表格是可以分解为两个 1-bit XOR 表格的运算。因为 XOR 运算是按位进行，操作数的高位和低位的 XOR 运算互不干扰。进而我们可以推广到 AND 运算，OR 运算等等。具体怎么做到呢？接下去，我们深入到表格的内部结构中。

## 2. 分解表格

稀疏的选择矩阵 $M$ 还有一个特点是，其中的所有非零元素都为 $1$。那么我们可以换一种方式来表达 Lookup 等式：

$$
\tilde{f}(\vec{X}) = \sum_{i=0}^{m-1}T[{\mathsf{col}(i)}] \cdot \tilde{eq}_i(\vec{X})
$$

为了排版清晰，这里我们换用大写的 $T$ 表示未被分解的大表格，分解出的子表格用小写字母 $t$ 表示，并且用 $T[i]$ 和 $t[i]$ 符号来表示表格中第 $i$ 个元素 $t_i$。其中 $\mathsf{col}(i)$ 表示第 $i$ 行的 $1$ 所在的列坐标，可以看成是 $M$ 矩阵的一种稠密表示。容易验证对任意的 $i\in[0, m)$，$T[\mathsf{col}(i)] = f_i$，相当于列出表格中的被 $M$ 非零元素筛选出来的元素。因此这个等式可以看成 $\tilde{f}(\vec{X})$ 的另一种定义，等价于

$$
 \tilde{f}(\vec{X}) = \sum_{\vec{y}\in\{0,1\}^{\log{N}}}\tilde{M}(\vec{X}, \vec{y})\cdot \tilde{t}(\vec{y})
$$

我们把 $T[{\mathsf{col}(i)}]$ 单独排成一个向量 $\vec{h}$，然后把向量编码成 MLE 多项式，记为 $\tilde{h}(\vec{X})$。那么通过一个随机挑战向量 $\vec{r}$， Lookup 的关系就归约到下面的等式:

$$
\tilde{f}(\vec{r}) \overset{?}{=}  \sum_{i\in[0,m)}h_i\cdot \tilde{eq}_i(\vec{r}) 
$$

根据 Offline Memory Checking 的思路，我们可以证明 $h_i$ 都读取自表格 $T$。这样相当于原地踏步，我们为了证明一个 Lookup关系，我们归约到了另一个 Lookup 关系。不过我们是否可以 $h_i$ 分解到一个二维（或者多维）的子表格上呢？就像 Spark 协议中的 $\vec{e}$ 向量一样，我们是把 $\vec{e}$ 所读取的内存 $\vec{\lambda}$ 分解成了 $\vec{\lambda}^{(x)}$ 和 $\vec{\lambda}^{(y)}$，然后把 $\vec{e}$ 分解为 $\vec{e}^{(x)}$ 和 $\vec{e}^{(y)}$。然而并不是所有的表格都能像 $\vec{\lambda}$ 一样满足 Tensor Structure 的。事实上，绝大部分的表格不满足这个条件。不过幸运地是，尽管他们不满足 Tensor Structure，但是一大类的有用表格可以按照类似的思路处理。

我们先看一个简单但很实用的表格，RangeCheck 表格。当需要证明 $0\leq x \lt 2^k$，我们可以构造一个表格 $T_\mathsf{{range, k}}=(0,1,\ldots,2^k-1)$，如果 $x\in T_\mathsf{{range, k}}$，那么说明 $x$ 在 $0$ 到 $2^k-1$ 之间。

这个表格 $T_\mathsf{{range, k}}$ 可以被分解成两个 $2^{k/2}$ 的 RangeCheck 表格之间的运算:

$$
T_\mathsf{{range, k}}[i\cdot 2^{k/2}+j] = t_\mathsf{{range, {k/2}}}[i]\cdot 2^{k/2} + t_\mathsf{{range, k/2}}[j]
$$

比如我们假设 $k=4$，$T_\mathsf{{range, 4}}$ 定义如下：

$$
T_\mathsf{{range,4}} = (0,1,2,3, \ldots, 15)
$$

另一个 2-bit RangeCheck 表格 $t_\mathsf{{range, 2}}$ 定义如下：

$$
t_\mathsf{{range, 2}} = (0,1,2,3)
$$

那么我们可以用下面的矩阵来展示 $T_\mathsf{{range,4}}$ 和子表格 $t_\mathsf{{range,2}}$ 之间的关系：

$$
\begin{array}{c|cccc}
 & 0 & 1 & 2 & 3\\ 
\hline
0\cdot 2^2=0 & 0 & 1 & 2 & 3 \\
1\cdot 2^2 = 4 & 4 & 5 & 6 & 7 \\
2\cdot 2^2 = 8 & 8 & 9 & 10 & 11 \\
3\cdot 2^2 = 12 & 12 & 13 & 14 & 15 \\
\end{array}
$$

矩阵的第一行为 $t^{(x)}=t_\mathsf{{range, 2}}$，第一列也为 $t^{(y)}=t_\mathsf{{range, 2}}$，矩阵中的每个单元可以表示为 

$$
T_\mathsf{{range,4}}[i,j]=  2^2 \cdot t^{(x)}[i] + t^{(y)}[j]
$$

于是矩阵的所有单元构成了 $T_\mathsf{{range,4}}$ 的所有元素。

针对 Rangecheck 表格这个特例，我们可以构造一个高效的 Lookup Argument。

## 3. RangeCheck 表格的 Lookup Argument

我们用 $\tilde{t}_{\mathsf{range2}}(\vec{X})$ 和 $\tilde{T}_{\mathsf{range4}}(\vec{X})$ 表示 2bit 和 4bit-RangeCheck 表格的 MLE 多项式，那么它们满足下面的等式：

$$
\tilde{T}_{\mathsf{range4}}(X_0,X_1,X_2,X_3) = 4\cdot \tilde{t}_{\mathsf{range2}}(X_0,X_1) + \tilde{t}_{\mathsf{range2}}(X_2,X_3)
$$

那么 Lookup 关系可以写成下面的形式：

$$
\begin{split}
 \tilde{f}(r_0,r_1,r_2,r_3) &= \sum_{i\in[0,m)}\tilde{T}_{\mathsf{range4}}(\mathsf{bits}({\mathsf{col}(i)})) \cdot \tilde{eq}_i(r_0,r_1,r_2,r_3)  \\
 & = \sum_{i\in[0,m)}\Big(4\cdot \tilde{t}_{\mathsf{range2}}(\mathsf{bits}^{(hi)}({\mathsf{col}(i)})) + \tilde{t}_{\mathsf{range2}}(\mathsf{bits}^{(lo)}({\mathsf{col}(i)}))\Big) \cdot \tilde{eq}_i(r_0,r_1,r_2,r_3)  \\
\end{split}
$$

这里 $\mathsf{bits}^{(hi)}(\mathsf{col}(i))$ 和 $\mathsf{bits}^{(lo)}(\mathsf{col}(i))$ 分别表示 $\mathsf{col}(i)$ 的高 2bits 和低 2bits。

同样，我们需要借助向量 $\vec{e}$，其中向量元素 $e_i$ 表示 $\tilde{T}_{\mathsf{range4}}$ 在 $\mathsf{bits}(\mathsf{col}(i)), i\in[0,m)$ 处的取值，因此上面的等式可以转化为：

$$
 \tilde{f}(r_0,r_1,r_2,r_3) = \sum_{i\in[0,m)} e_i \cdot \tilde{eq}_i(r_0,r_1,r_2,r_3)  \\
$$

由于 $\tilde{T}_{\mathsf{range4}}$ 的可分解性，我们可以把 $e_i$ 的高 2bits 和低 2bits 分别抽取出来，它们构成两个向量 $\vec{e}^{(x)}$ 与 $\vec{e}^{(y)}$，分别对应 $t_{\mathsf{range2}}$ 中的第 $i_0$ 项和 第 $i_1$ 项，满足 $i=4\cdot i_0 + i_1$。

接下来构造 $\tilde{e}^{(x)}$ 与 $\tilde{e}^{(y)}$ 两个 MLE 多项式，分别编码 $\vec{e}^{(x)}$ 与 $\vec{e}^{(y)}$，那么上面的等式可以转化为：

$$
\begin{split}
 \tilde{f}(r_0,r_1,r_2,r_3) & = \sum_{\vec{b}\in\{0,1\}^{\log{m}}} \Big(4\cdot \tilde{e}^{(x)}(\vec{b})+ \tilde{e}^{(y)}(\vec{b})\Big)\cdot\tilde{eq}(\vec{b}, (r_0,r_1,r_2,r_3))
\end{split}
$$

由于这个等式是一个求和式，因此我们可以利用 Sumcheck 协议来把上面的等式归约到：

$$
v' =  \Big(4\cdot \tilde{e}^{(x)}(\vec{\rho})+ \tilde{e}^{(y)}(\vec{\rho})\Big)\cdot\tilde{eq}(\vec{\rho}, (r_0,r_1,r_2,r_3))
$$

其中 $\vec{\rho}$ 为 Verifier 在 Sumcheck 过程中产生的长度为 $\log{m}$ 的挑战向量。辅助向量 $\vec{e}^{(x)}$ 与 $\vec{e}^{(y)}$ 的正确性可以由 Offline Memory Checking 来证明。

类似的，我们可以把 32-bit RangeCheck 表格分解成四个 8-bit RangeCheck 表格， 或者两个 16-bit RangeCheck 表格。

我们用这个可分解表格，构造一个 Lookup Argument，与之前的方案的差异在于，它利用了表格向量的内部结构，可以处理超大的表格。

## 4. Lasso 协议框架

Lasso 的核心协议是一个类似 Spark 的稀疏多项式承诺，被称为 Surge。对于任意一个查找记录 $f_i$，假如 $f_i$ 在主表格 $T$ 中的索引值为 $a_i$。因为主表格 $\vec{T}$ 可被分解，比如 $\vec{T}$ 可以被分解为 $c$ 个子表格。分解维度的数量 $c$ 对应于主表的索引值 $i$ 按照二进制位的拆分。例如：

$$
\begin{split}
\mathsf{bits}(i) &= \mathsf{dim}^{(0)}(i)\parallel \mathsf{dim}^{(1)}(i) \parallel \cdots \parallel\mathsf{dim}^{(c-1)}(i)
\end{split}
$$

即每一个主表元素 $T_i$ 都可以写成关于 $c$ 个子表格 $(t^{(0)}, t^{(1)}, \ldots, t^{(c-1)})$ 中的元素的运算：

$$
T_i = \mathcal{G}\Big(
\ t^{(0)}[\mathsf{dim}^{(0)}(i)],
\ t^{(1)}[\mathsf{dim}^{(1)}(i)],
\ldots,
\ t^{(c-1)}[\mathsf{dim}^{(c-1)}(i)]
\ 
\Big)
$$

我们可以写下对于可分解表格的 Lookup Argument 的等式：

$$
 \tilde{f}(\vec{X}) \overset{?}{=} \sum_{\vec{b}\in\{0,1\}^{\log{m}}}
\mathcal{G}\Big(t^{(0)}[\mathsf{dim}^{(0)}(\vec{b})], t^{(1)}[\mathsf{dim}^{(1)}(\vec{b})], \ldots,  t^{(c-1)}[\mathsf{dim}^{(c-1)}(\vec{b})]\Big)
\cdot \tilde{eq}(\vec{b},\vec{X}) 
$$

以 32-bit 的 Rangcheck 表格为例，假如我们需要把它分解为四个子表格，这四个子表格完全一摸一样，都是一个 8-bit 的 Rangecheck 表格。那么我们可以写下下面的等式：

$$
T_{\mathsf{range, 32}}[(i_0,i_1,i_2,i_3)_{(2)}] = 2^{24} \cdot t_{\mathsf{range,8}}[i_0]
+ 2^{16} \cdot t_{\mathsf{range,8}}[i_1] 
+ 2^{8} \cdot t_{\mathsf{range,8}}[i_2]
+ t_{\mathsf{range,8}}[i_3]
$$

这里 $\mathcal{G}(\cdot)$ 的定义如下

$$
\mathcal{G}(y_0, y_1, y_2, y_3) = 2^{24} \cdot y_0
+ 2^{16} \cdot y_1
+ 2^{8} \cdot y_2
+ y_3
$$

### 协议细节

公共输入：

1. 子表格的承诺：$\{\mathsf{cm}(\tilde{t}^{(i)}(\vec{X}))\}_{i\in[c]}$
2. 查询向量的承诺：$\mathsf{cm}(\tilde{f}(\vec{X}))$

第一轮：Prover 计算并承诺 $\{\tilde{\mathsf{dim}}^{(i)}(\vec{X})\}_{i\in[c]}$

第二轮：Verifier 发送随机向量 $\vec{r}\in\mathbb{F}^{\log{m}}$

第三轮：Prover 计算向量 $\vec{e}^{(0)}, \vec{e}^{(1)}, \ldots, \vec{e}^{(c-1)}$

$$
e^{(i)}_j = t^{(i)}[{\mathsf{dim}}^{(i)}(\mathsf{bits}(j))], \quad \forall j\in[0, m), \forall i\in[0, c)
$$

Prover 计算计数器向量（长度为 $\log{m}$） $\vec{\mathsf{c}}^{(0)}, \vec{\mathsf{c}}^{(1)}, \ldots, \vec{\mathsf{c}}^{(c-1)}$，

Prover 计算终状态中的计数器向量（长度为 $\log{N}/c$） $\vec{\mathsf{s}}^{(0)}, \vec{\mathsf{s}}^{(1)}, \ldots, \vec{\mathsf{s}}^{(c-1)}$

Prover 发送向量的承诺 $\mathsf{cm}(\vec{e}^{(0)}), \mathsf{cm}(\vec{e}^{(1)}), \ldots, \mathsf{cm}(\vec{e}^{(c-1)})$

第四轮：Prover 和 Verifier 运行 Sumcheck 协议，证明下面的等式：

$$
v \overset{?}{=} \sum_{\vec{b}\in\{0,1\}^{\log{m}}} \mathcal{G}\Big(\tilde{e}^{(0)}(\vec{b}), \tilde{e}^{(1)}(\vec{b}),\ldots, \tilde{e}^{(c-1)}(\vec{b})\Big)\cdot \tilde{eq}(\vec{b}, \vec{r})
$$

Prover 和 Verifier 把等式归约到：

$$
v' = \mathcal{G}\Big(\tilde{e}^{(0)}(\vec{\rho}), \tilde{e}^{(1)}(\vec{\rho}),\ldots, \tilde{e}^{(c-1)}(\vec{\rho})\Big)\cdot \tilde{eq}(\vec{\rho}, \vec{r})
$$

第五轮：Prover 发送 $(v_{e,0}, v_{e,1}, \ldots, v_{e,c-1})$，以及 $(\pi_{e,0}, \pi_{e,1}, \ldots, \pi_{e,c-1})$

1. $v_{e,i} = \tilde{e}^{(i)}(\vec{\rho}), \quad \forall i\in [0, c)$
2. $\pi_{e,i} = \mathsf{PCS.Eval}(E^{(i)}, \vec{\rho}, v_{e,i}; \vec{e}^{(i)}), \quad \forall i\in [0, c)$

第六轮：Verifier 验证 

$$
\mathsf{PCS.Verify}(\mathsf{cm}(\vec{e}^{(i)}), \vec{\rho}, v_{e,i}, \pi_{e, i}) = 1, \quad \forall i\in [0, c)
$$

$$
v' \overset{?}{=} \mathcal{G}(v_{e,0}, v_{e,1}, \ldots, v_{e,c-1})\cdot \tilde{eq}(\vec{\rho}, \vec{r})
$$

第七轮：Prover 和 Verifier 调用 Offline Memory Checking 证明每个 $\vec{e}^{(i)}$ 的正确性，即每个向量元素 $e_j\in \vec{e}^{(i)}$ 都是从表格 $t^{(i)}$ 中读取，读取的位置为 $\mathsf{dim}^{(i)}(\mathsf{bits}(j))$：

$$
\vec{e}^{(i)} = (t^{(i)}[\mathsf{dim}^{(i)}(\mathsf{bits}(0))], t^{(i)}[\mathsf{dim}^{(i)}(\mathsf{bits}(1))], \ldots, t^{(i)}[\mathsf{dim}^{(i)}(\mathsf{bits}(m-1))]), \quad \forall i\in[0, c)
$$


## 5. 二元操作表格的分解

除了 RangeCheck 表格之外，还有 AND, OR,  XOR 这类按位计算表格也可以按照同样的思路进行分解。例如下面是一个 1-bit AND 表，记为 $\mathsf{AND}^{(1)}$：

$$
\begin{array}{c|c}
i & A \parallel B \\
\hline
0 & 0 \parallel 0 \\
1 & 0 \parallel  1 \\
2 & 1 \parallel  0 \\
3 & 1 \parallel  1 \\
\end{array}
\quad
\begin{array}{|c|}
\hline
A\& B \\
\hline
 0 \\
 0 \\
0 \\
1 \\
\hline
\end{array}
$$

可以看出，这个表格有 4 行，第 $i$ 行的表格元素为 $A\& B$，而表格的索引值的高位为 $A$，低位为 $B$。比如 $i=2$ 这一行，$i=(10)_{(2)}$ 二进制位的高位为 $1$，低位为 $0$，那么这一行的表格元素为 $0$，表示 $1 \& 0 = 0$。假设我们要分解一个 2-bit AND 表格，$\mathsf{AND}^{(2)}$, 那么我们可以用下面的矩阵来表示：

$$
\begin{array}{c|cccc}
 & 0 & 0 & 0 & 1 \\
\hline
0 & 00 & 00 & 00 & 01 \\
0 & 00 & 00 & 00 & 01 \\
0 & 00 & 00 & 00 & 01 \\
1 & 10 & 10 & 10 & 11 \\
\end{array}
$$ 

矩阵中的每个单元格表示 $(A_0, A_1)\&(B_0,B_1)$，其中 $A_0\& B_0=\mathsf{T_{and,1}}[(A_0,B_0)]$，$A_1\& B_1=\mathsf{T_{and,1}}[(A_1,B_1)]$
满足下面的等式：

$$
\mathsf{AND}^{(2)}[(A_0,A_1, B_0, B_1)_{(2)}] = 2\cdot \mathsf{AND^{(1)}}[(A_0,B_0)] + \mathsf{AND^{(1)}}[(A_1,B_1)]
$$

因此，我们可以推而广之，对于任意的 $W$-bit AND 表格，我们可以把操作数 $A$ 和 $B$ 按位拆分成 $c$ 段，每一段查子表格 $\mathsf{AND}^{(W/c)}$ 确定 $A_i\& B_i$，然后将 $c$ 个运算结果再按位拼装起来。下面写出这个关系等式：

$$
\mathsf{\widetilde{AND}}^{(W)}(\vec{X}_0\parallel\vec{X}_1\parallel\cdots \parallel \vec{X}_{c-1},\  \vec{Y}_0\parallel\vec{Y}_1\parallel\cdots\parallel \vec{Y}_{c-1}) = 
2^{c-1} \cdot \mathsf{\widetilde{AND}}^{(W/c)}(\vec{X}_0, \vec{Y}_0) 
+ 2^{c-2} \cdot \mathsf{\widetilde{AND}}^{(W/c)}(\vec{X}_1, \vec{Y}_1) 
+ \ldots
+ \mathsf{\widetilde{AND}}^{(W/c)}(\vec{X}_{c-1}, \vec{Y}_{c-1})
$$

代入到 Lookup 关系等式中，我们可以得到：

$$
 \tilde{f}(\vec{X},\vec{Y}) \overset{?}{=} \sum_{\vec{a},\vec{b}\in\{0,1\}^{\log{m}}}
\mathcal{G}_{\mathsf{AND}}({\mathsf{{AND}}^{(W/c)}}[\mathsf{dim}_0(\vec{a}), \mathsf{dim}_0(\vec{b})], \ldots,  {\mathsf{{AND}}^{(W/c)}}[\mathsf{dim}_{c-1}(\vec{a}), \mathsf{dim}_{c-1}(\vec{b})])
\cdot \tilde{eq}((\vec{a},\vec{b}),(\vec{X},\vec{Y})) 
$$

代入上面的 Lasso 协议，我们可以构造出对 $\mathsf{\widetilde{AND}}^{(W)}$ 表格的 Lookup Arugment 方案。

同样我们可以把其它的二元位操作同样按照这样的思路去分解，如 $\mathsf{OR}$ 与 $\mathsf{XOR}$。把主表格拆分成 $c$ 段，假设主表格表示两个长度为 $W$ 的二进制数的位运算，那么第 $i$ 个子表格对应主表索引的第 $i\cdot (W/c)$ 位到 $(i+1)\cdot (W/c)$ 之间的位运算。$(\mathsf{dim}_i(\vec{a}), \mathsf{dim}_i(\vec{b}))$ 表示两个操作数 $\vec{X}$ 和 $\vec{Y}$ 的二进制位在主表格第 $i$ 个维度上的位置索引。

##  References

- [Spartan] [Spartan: Efficient and general-purpose zkSNARKs without trusted setup
](https://eprint.iacr.org/2019/550) by Srinath Setty.
- [Lasso] [Unlocking the lookup singularity with Lasso](https://eprint.iacr.org/2023/1216) by Srinath Setty, Justin Thaler and Riad Wahby.
- [Jolt] [Jolt: SNARKs for Virtual Machines via Lookups](https://eprint.iacr.org/2023/1217) by Arasu Arun, Srinath Setty and Justin Thaler.
- [Baloo] [Baloo: Nearly Optimal Lookup Arguments](https://eprint.iacr.org/2022/1565) by Arantxa Zapico, Ariel Gabizon, Dmitry Khovratovich, Mary Maller and Carla Ràfols.
