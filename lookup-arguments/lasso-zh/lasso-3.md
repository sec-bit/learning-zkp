# 理解 Lasso (三)：可分解表格的 Lookup 证明

基于稀疏多项式承诺 Spark 的 Lookup Argument 实际上是在证明一个 Inner product $\langle \vec{u}, \vec{t}\rangle$，其中 $\vec{u}$ 为一个稀疏向量，而 $\vec{t}$ 则是一个稠密的向量，代表被查找的表格。

Lasso 论文把可以处理的表格分为三类。

- Unstructured but small
- Decomposable
- Non-decomposable but structured

前文介绍的证明系统都是针对 Unstructured-but-small 这类表格，并利用了查找向量 $\vec{f}$ 本身具有一定的稀疏性和特殊的内在结构。设想如果表格也具有可利用的内在结构，那么我们可以继续沿用类似的思路，把表格分解成若干个小表格，然后把大表格上的 Lookup 关系降维到对小表格的 Lookup 关系。这就是本文要描述的表格分解，也是 Lasso 可以处理的第二类表格。

先用一个简单的例子来理解下什么是表格的分解。比如这里有一个 1-bit XOR 表格，记为 $\vec{t}_{\mathsf{XOR}^1}$，这个表格的计算定义如下：

$$
\begin{array}{c|c|c}
A & B & A\oplus B \\
\hline
0 & 0 & 0 \\
0 & 1 & 1 \\
1 & 0 & 1 \\
1 & 1 & 0 \\
\end{array}
$$

我们可以利用这个 1-bit XOR 表格，构造出一个 4-bit XOR 表格，基本思路是，我们可以把两个 4bit 数，依次按照每个 bit 进行 XOR 运算（通过查找 1-bit XOR 表格），然后把四个运算结果再拼接在一起，得到一个 4bit 的结果。

$$
(a_4\parallel a_3\parallel  a_2\parallel a_1)_{(2)} \oplus (b_4\parallel b_3\parallel b_2\parallel b_1)_{(2)} = (c_4\parallel c_3\parallel c_2\parallel c_1)_{(2)}
$$

这里每一个 $(a_i,b_i,c_i)\in \vec{t}_{\mathsf{XOR}^1}$。这相当于把一个 4-bit XOR 表格分解成了四个 1-bit XOR 表格，类似还有一些属于 Lasso 支持的第二类「Decomposable」表格。我们下面逐步分析表格的分解，以及 Lasso 如何利用可分解表格从而实现大表格（这些表格的长度可以高达 $2^{128}$ ）的 Lookup 证明。

## 1. 表格分解

回顾下 Indexed Lookup Argument 的基本等式：
$$
M\vec{t} \overset{?}{=} \vec{f}
$$

其中 $M\in\mathbb{F}^{m\times N}$，其中 $m=|\vec{f}|$ 为 Lookup 的数量，$N=|\vec{t}|$ 为表格的长度。

考虑到 $M$ 是一个比较特殊的选择矩阵，其每一行都是一个 Unit Vector，即有且仅有一个 $1$，其余均为 $0$。
那么我们可以用稠密的方法来编码矩阵，只关注那些非零元素：

$$
t_{\mathsf{col}(i)}  \overset{?}{=} f_i, \quad \forall i \in [m]
$$

其中 $\mathsf{col}(i)$ 表示第 $i$ 行的 $1$ 所在的列坐标。那么 $\{\mathsf{col}(i)\}_{i\in[m]}$ 就是 $M$ 的一种稠密表示。

我们把 $t_{\mathsf{col}(i)}$ 编码成 MLE 多项式，记为 $\tilde{h}(X)$，$\vec{f}$ 编码为 $\tilde{f}(X)$。那么通过一个随机挑战向量 $\vec{r}$， Lookup 的关系就归约到下面的等式:

$$
\tilde{h}(\vec{r})=\sum_{i\in[0,m)}t_{\mathsf{col}(i)} \cdot \tilde{eq}_i(\vec{r}) \overset{?}{=} \tilde{f}(\vec{r})
$$

假设表格 $\vec{t}$ 可以被分解为两个子表格，比如我们考虑下 4-bit RangeCheck 表格，包含了所有 4bit 自然数，

$$
\mathsf{T_{range,4}} = (0,1,2,3, \ldots, 15)
$$

而一个分解后的 2-bit RangeCheck 表格为：

$$
\mathsf{T_{range, 2}} = (0,1,2,3)
$$

表格的分解关系定义如下：

<!-- $$
\mathsf{T_{range,4}} = \Big(2^2\mathsf{T^T_{range,2}}\Big) \times (1,1,1,1)+ \mathsf{T_{range,2}}
$$ -->

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


我们用 $\tilde{t}_{\mathsf{range2}}(\vec{X})$ 和 $\tilde{t}_{\mathsf{range4}}(\vec{X})$ 表示 2bit 和 4bit-RangeCheck 表格的 MLE 多项式，那么它们满足下面的等式：

$$
\tilde{t}_{\mathsf{range4}}(X_0,X_1,X_2,X_3) = 4\cdot \tilde{t}_{\mathsf{range2}}(X_0,X_1) + \tilde{t}_{\mathsf{range2}}(X_2,X_3)
$$

那么 Lookup 关系可以写成下面的形式：

$$
\begin{split}
 \tilde{f}(r_0,r_1,r_2,r_3) &= \sum_{i\in[0,m)}\tilde{t}_{\mathsf{range4}}(\mathsf{bits}({\mathsf{col}(i)})) \cdot \tilde{eq}_i(r_0,r_1,r_2,r_3)  \\
\end{split}
$$

类似的，我们可以把 32bit RangeCheck 表格分解成四个 8bit RangeCheck 表格， 或者两个 16bit RangeCheck 表格。

我们引入一个长度为 $m$ 的辅助向量 $\vec{e}$，其中向量元素 $e_i$ 表示 $\tilde{t}_{\mathsf{range4}}$ 在 $\mathsf{bits}(\mathsf{col}(i)), i\in[0,m)$ 处的取值，因此上面的等式可以转化为：

$$
 \tilde{f}(\vec{r}) = \sum_{i\in[0,m)} e_i \cdot \tilde{eq}_i(\vec{r})  \\
$$

由于 $t_{\mathsf{range4}}$ 的可分解性，我们可以把 $e_i$ 的高 2bits 和低 2bits 分别抽取出来，它们构成两个向量 $\vec{e}^{(hi)}$ 与 $\vec{e}^{(lo)}$，分别对应 $t_{\mathsf{range2}}$ 中的第 $i_0$ 项和 第 $i_1$ 项， $i=4\cdot i_0 + i_1$。

构造 $\tilde{e}^{(hi)}$ 与 $\tilde{e}^{(lo)}$ 两个 MLE，分别编码 $\vec{e}$ 中元素的高 2bits 和低 2bits，那么上面的等式可以转化为：

$$
\begin{split}
 \tilde{f}(\vec{r}) & = \sum_{\vec{b}\in\{0,1\}^{\log{m}}} \Big(4\cdot \tilde{e}^{(hi)}(\vec{b})+ \tilde{e}^{(lo)}(\vec{b})\Big)\cdot\tilde{eq}(\vec{b}, \vec{r})
\end{split}
$$

由于这个等式是一个求和，因此我们可以利用 Sumcheck 协议来把上面的等式归约到：

$$
v' =  \Big(4\cdot \tilde{e}^{(hi)}(\vec{\rho})+ \tilde{e}^{(lo)}(\vec{\rho})\Big)\cdot\tilde{eq}(\vec{\rho}, \vec{r})
$$

其中 $\vec{\rho}$ 为 Verifier 在 Sumcheck 过程中产生的挑战数。辅助向量 $\vec{e}^{(hi)}$ 与 $\vec{e}^{(lo)}$ 的正确性可以由 Memory-checking 来证明。

我们用这个思路可以构造一个 Lookup Argument，与之前的 Lookup Argument 不同的地方在于，我们利用了表格向量的内部结构。

除了 RangeCheck 表格之外，还有 AND, OR,  XOR 这类按位计算表格。例如对于 4-bit AND 表格可以分解为两个 2-bit AND 表格的运算：

$$
\begin{array}{c|cccc}
 & 00 & 01 & 10 & 11 \\
\hline
00 & 00 & 00 & 00 & 00 \\
01 & 00 & 01 & 00 & 01 \\
10 & 00 & 00 & 10 & 10 \\
11 & 00 & 01 & 10 & 11 \\
\end{array}
$$

我们把 $t_{\mathsf{AND}4}(\vec{x}, \vec{y})$ 拆分为

$$
t_{\mathsf{AND}4}(\vec{x}_0\parallel\vec{x}_1, \vec{y}_0\parallel\vec{y}_1) = 
2^2 \cdot t_{\mathsf{AND}2}(\vec{x}_0, \vec{y}_0) + t_{\mathsf{AND}2}(\vec{x}_1, \vec{y}_1)
$$

对于上面这类位运算表格，我们可以把 Lookup 关系等式写成下面的**通用形式**：

$$
 \tilde{f}(\vec{r}) \overset{?}{=} \sum_{\vec{b}\in\{0,1\}^{\log{m}}}
G(t_0[\mathsf{dim}_0(\vec{b})], t_1[\mathsf{dim}_1(\vec{b})], \ldots,  t_{c-1}[\mathsf{dim}_{c-1}(\vec{b})])
\cdot \tilde{eq}(\vec{b},\vec{r}) 
$$

这里表格 $\vec{t}$ 被拆分成了 $c$ 个子表格，分别为 $(t_0, t_1,\ldots, t_{c-1})$。而 $G(\cdots)$ 是一个参数化的多项式，可以根据不同的位运算表格来定义。

假设主表格表示两个长度为 $W$ 的二进制数的位运算，那么第 $i$ 个子表格对应主表索引的第 $i\cdot W/c$ 位
到 $i\cdot (W/c+1)$ 之间的位运算。$\mathsf{dim}_i(\vec{b})$ 表示第 $\vec{b}$ 个查询表项，在主表格第 $i$ 个维度上的位置坐标。

## 2. Lasso 协议框架

至此，我们可以定义出完整的 Lasso 协议框架。Lasso 的核心协议是一个类似 Spark 的稀疏多项式承诺。其中的稀疏向量是查找记录在各个维度的子表格中的索引。对于任意一个查找记录 $f_i$，假如 $f_i$ 在主表格 $T$ 中的索引值为 $a_i$。因为主表格可被分解，比如 $T$ 可以被分解为 $k\cdot c$ 个子表格，其中共有 $c$ 个维度，其中每个维度上有 $k$ 个子表格。维度数量 $c$ 等价于我们把索引值 $a_i$ 先按照二进制位拆分为两段，代表两个位运算操作数，然后再把高位低位两段再分解为 $c$ 段：

$$
\begin{split}
a_i &= X_i\parallel Y_i \\
& = \Big(\mathsf{dim}^{(0)}(i), \mathsf{dim}^{(1)}(i), \ldots, \mathsf{dim}^{(c-1)}(i)\Big)
\parallel
\Big(\mathsf{dim}^{(0)}(i), \mathsf{dim}^{(1)}(i), \ldots, \mathsf{dim}^{(c-1)}(i)\Big)
\end{split}
$$

 $t^{(0)}, t^{(1)}, \ldots, t^{(c-1)}$，那么 $f_i$ 在第 $i$

 个维度上的索引为 $b_i$，那么 $f_i$ 在主表格中的索引为 $\mathsf{dim}_i(\vec{b})$。

$$
 \tilde{f}(\vec{r}) \overset{?}{=} \sum_{\vec{b}\in\{0,1\}^{\log{m}}}
G(t^{(0)}[\mathsf{dim}^{(0)}(\vec{b})], t^{(1)}[\mathsf{dim}^{(1)}(\vec{b})], \ldots,  t^{\alpha-1}[\mathsf{dim}^{(c-1)}(\vec{b})])
\cdot \tilde{eq}(\vec{b},\vec{r}) 
$$

请注意，我们这里仅给出 Unindexed 

公共输入：

1. 子表格的承诺：$\{\mathsf{PCS.Commit}(\tilde{t}_i(\vec{X}))\}_{i\in[c]}$
2. 查询向量的承诺：$\mathsf{PCS.Commit}(\tilde{f}(\vec{X}))$

第一轮：Prover 计算并承诺 $\{\tilde{dim}_i(\vec{X})\}_{i\in[c]}$

第二轮：Verifier 发送随机向量 $\vec{r}\in\mathbb{F}^{\log{m}}$

第三轮：Prover 计算向量 $\vec{e}^{(0)}, \vec{e}^{(1)}, \ldots, \vec{e}^{(\alpha-1)}$

$$
e^{(i)}_j = t_i[\mathsf{dim}_i(\vec{r})], \quad \forall j\in[m], \forall i\in[0, \alpha)
$$

Prover 计算向量（长度为 $\log{m}$） $\vec{\mathsf{c}}^{(0)}, \vec{\mathsf{c}}^{(1)}, \ldots, \vec{\mathsf{c}}^{(\alpha-1)}$，

Prover 计算向量（长度为 $\log{N}/c$） $\vec{\mathsf{s}}^{(0)}, \vec{\mathsf{s}}^{(1)}, \ldots, \vec{\mathsf{s}}^{(\alpha-1)}$

Prover 发送向量的承诺 $E^{(0)}, E^{(1)}, \ldots, E^{(\alpha-1)}$

第四轮：Prover 和 Verifier 运行 Sumcheck 协议，证明下面的等式：

$$
v \overset{?}{=} \sum_{\vec{b}\in\{0,1\}^{\log{m}}} G\Big(\tilde{e}^{(0)}(\vec{b}), \tilde{e}^{(1)}(\vec{b}),\ldots, \tilde{e}^{(\alpha-1)}(\vec{b})\Big)\cdot \tilde{eq}(\vec{b}, \vec{r})
$$

最后，Prover 和 Verifier 把等式归约到：

$$
v' = G\Big(\tilde{e}^{(0)}(\vec{\rho}), \tilde{e}^{(1)}(\vec{\rho}),\ldots, \tilde{e}^{(\alpha-1)}(\vec{\rho})\Big)\cdot \tilde{eq}(\vec{\rho}, \vec{r})
$$

第五轮：Prover 发送 $(v_{e,0}, v_{e,1}, \ldots, v_{e,\alpha-1})$，以及 $(\pi_{e,0}, \pi_{e,1}, \ldots, \pi_{e,\alpha-1})$

1. $v_{e,i} = \tilde{e}^{(i)}(\vec{\rho}), \quad \forall i\in [0, \alpha)$
2. $\pi_{e,i} = \mathsf{PCS.Eval}(E^{(i)}, \vec{\rho}, v_{e,i}; \vec{e}^{(i)}), \quad \forall i\in [0, \alpha)$

第六轮：Verifier 验证 

$$
\mathsf{PCS.Verify}(E^{(i)}, \vec{\rho}, v_{e,i}, \pi_{e, i}) = 1, \quad \forall i\in [0, \alpha)
$$

$$
v' \overset{?}{=} G(v_{e,0}, v_{e,1}, \ldots, v_{e,\alpha-1})\cdot \tilde{eq}(\vec{\rho}, \vec{r})
$$

第七轮：Prover 和 Verifier 调用 Offline-memory-checking 证明每个 $\vec{e}^{(i)}$ 的正确性，即每个向量元素 $e_j\in \vec{e}^{(i)}$ 都是从表格 $t^{(i)}$ 中读取，读取的位置为 $\mathsf{dim}^{(i)}(\mathsf{bits}(j))$：

$$
\vec{e}^{(i)} = (t^{(i)}[\mathsf{dim}^{(k)}(\mathsf{bits}(0))], t^{(i)}[\mathsf{dim}^{(k)}(\mathsf{bits}(1))], \ldots, t^{(i)}[\mathsf{dim}^{(k)}(\mathsf{bits}(m-1))]), \quad \forall i\in[0, \alpha), k = \alpha/c
$$

## 更多的表格拆分案例

Jolt 论文给出了更多的可分解表格，用于表达 RISC-V 指令的计算过程。

我们首先看一个简单的可分解表格 $T_{\mathsf{EQ}}$，这个表格用来判断 $A\overset{?}{=} B$，如果 A 和 B 相等，那么这个计算返回 $1$，否则返回 $0$。下面是一个 2-bit 表格示例：

$$
\small
\begin{array}{c|c|c}
A & B & A\overset{?}{=} B \\
\hline
00 & 00 & 1\\
00 & 01 & 0\\
00 & 10 & 0\\
00 & 11 & 0\\
01 & 00 & 0\\
01 & 01 & 1\\
01 & 10 & 0\\
01 & 11 & 0\\
10 & 00 & 0\\
10 & 01 & 0\\
10 & 10 & 1\\
10 & 11 & 0\\
11 & 00 & 0\\
11 & 01 & 0\\
11 & 10 & 0\\
11 & 11 & 1\\
\end{array}
$$

然后我们分析一个 W-bit 长的表格如何分解。对于判等表格来说，我们可以把 $W$ 位的表格拆分成 $c$ 个 $W/c$-bit 子表格，这些子表格完全相等，因为高位低位运算互不干扰。我们先看子表格的定义：

$$
T_{\mathsf{EQ}}^{(W/c)}(\vec{x}, \vec{y}) = \prod_{i=0}^{(W/c)-1} \Big(x_i\cdot y_i + (1-x_i)\cdot (1-y_i)\Big)
$$

这个定义是说，如果 $x_i$ 和 $y_i$ 相等，那么 $x_i\cdot y_i + (1-x_i)\cdot (1-y_i)=1$，否则为 $0$。那么 $T_{\mathsf{EQ}}^{(W/c)}$ 表格的定义就是把 $W$ 位的表格拆分成 $c$ 个 $W/c$ 位的子表格，然后把这些子表格的结果相乘：

$$
T_{\mathsf{EQ}}^{(W)}(\vec{X}, \vec{Y})=\prod_{i=0}^{c-1}T_{\mathsf{EQ}}^{(W/c)}(\vec{X}_i, \vec{Y}_i)
$$

这里，

$$
\quad \vec{X}=\vec{X}_0\parallel\vec{X}_1\parallel\ldots\parallel\vec{X}_{c-1}, \qquad \vec{Y}=\vec{Y}_0\parallel\vec{Y}_1\parallel\ldots\parallel\vec{Y}_{c-1}
$$

### LTU 表格

接下来我们看一个稍微复杂点的表格，用来计算 $X \overset{?}{<} Y$，表格的索引值为 $X\parallel Y$，如果关系成立，那么表格项为 $1$，否则表格项为 $0$。

比较两个整数的一般算法是并行扫描两个整数 $X$, $Y$ 的每一个二进制位。从最高位开始，当遇到第一个不同的位时，停下来比较，并输出结果。

我们引入一个辅助的表格 $\mathsf{LTU}_i$，它表示 $X$ 与 $Y$ 第一个不同的 bit 位于第 $i$ 个位置（按照从低位到高位的顺序）

$$
\mathsf{LTU}^{(W)}_i(\vec{x}, \vec{y}) = (1-x_i)\cdot y_i\cdot \mathsf{EQ}^{(W-i-1)}(\vec{x}_{>i}, \vec{y}_{>i})
$$

例如，$\mathsf{LTU}^{(4)}_1(1101, 1110)=1$，因为 $x_1=0, y_1=1$，并且 $\mathsf{EQ}^{(2)}(11, 11)=1$。而 $\mathsf{LTU}^{(4)}_2(1101, 1110)=0$，因为 $x_2=1$，所以 $(1-x_2)=0$。下面是所有 $\mathsf{LTU}_i^{(4)}(1101,1110)$ 的计算过程：

$$
\begin{array}{c|cccc}
i=   & 3 & 2 & 1 & 0\\
\hline
x_i & 1 & 1 & {\color{red}0} & 1\\
y_i & 1 & 1 & {\color{red}1} & 0\\
\hline
EQ(x_{>i},x_{>i})   & 1 & 1 & 1 & 0\\
(1-x_i)y_i          & 0 & 0 & {\color{red}1}  & 0  \\
\hline
\mathsf{LTU}_i               & 0 & 0 & {\color{blue}1} & 0  \\
\end{array}
$$

利用多个辅助表格的求和，我们可以定义出 $\mathsf{LTU}^{(W)}$ 表格：

$$
\mathsf{LTU}^{(W)}(\vec{x}, \vec{y})=\sum_{i=0}^{W-1}\mathsf{LTU}^{(W)}_i(\vec{x}, \vec{y})
$$

对上面的例子，我们可以计算 LTU 表格中位于 $11011110$ 这个位置的项，$\mathsf{LTU}^{(4)}(1101,1110)$

$$
\mathsf{LTU}^{(4)}(1101,1110)=\sum_{j\geq i} \mathsf{LTU}_j(1101, 1110)= 0 + 0 + {\color{blue}1} +  0  = 1\\
$$

假如 $W=64$，那么 $\mathsf{LTU}^{(W)}$ 的尺寸将是 $2^{128}$，理论上我们无法存储这么大的表格。我们需要把大表格拆分成若干小表格关于一个多项式 $G(\cdot)$ 的运算。

接下来，我们分析如何拆分这个表格。假设我们把 $W$ 宽的 bits 拆分为 $c$ 段，每一段为 $W/c$。操作数 $\vec{x}$ 和 $\vec{y}$ 按 $c$ 段拆分后，表示如下：

$$
\vec{x}^{(W)} = \vec{x}^{(W/c)}_{c-1}\parallel \vec{x}^{(W/c)}_{c-2}\parallel \ldots \parallel \vec{x}^{(W/c)}_{0}, \quad \vec{y}^{(W)} = \vec{y}^{(W/c)}_{c-1}\parallel \vec{y}^{(W/c)}_{c-2}\parallel \ldots \parallel \vec{y}^{(W/c)}_{0}
$$

一个初步的想法是将这 $c$ 组操作数分别输入到 $c$ 个 $\mathsf{LTU}^{(W/c)}$ 表格中，然后把这些表格的结果相加。但是这样的计算结果并不是我们想要的，因为这样的计算结果会有多个 $1$，而我们只需要其中一组出现 $1$，其余为零，这样我们最后求和的结果才是 $1$。因此，我们可以考虑从高位开始，当某一段的 $\mathsf{LTU}^{(W/c)}$ 计算输出 $1$ 时，我们要求从高位开始的比较过的 $W/c$ 段都应该相等，否则就输出 $0$。

$$
\mathsf{LTU}^{(W)}(\vec{x}, \vec{y})=\sum_{j=0}^{c-1}
\Big(
    \mathsf{LTU}^{(W/c)}(\vec{x}_j, \vec{y}_j)\cdot 
    \prod_{k=j+1}^{c-1}\mathsf{EQ}^{(W/c)}(\vec{x}_k, \vec{y}_k)
\Big)
$$

如果要采用 Lasso 框架来证明关于 $\mathsf{LTU}^{(W)}(\vec{x}, \vec{y})$ 表格的 Lookup，我们需要定义下给出参数化的多项式 $G(\cdot)$：

$$
G\Big(
    \mathsf{LTU}^{(W/c)}[X_0\parallel Y_0], \mathsf{EQ}^{(W/c)}[X_0\parallel Y_0], 
    \mathsf{LTU}^{(W/c)}[X_1\parallel Y_1], \mathsf{EQ}^{(W/c)}[X_1\parallel Y_1], 
    \ldots,
    \mathsf{LTU}^{(W/c)}[X_{c-1}\parallel Y_{c-1}], \mathsf{EQ}^{(W/c)}[X_{c-1}\parallel Y_{c-1}] 
\Big)
$$

总共有 $c$ 段，每一段需要两个表格，$\mathsf{LTU}^{(W/c)}$ 和 $\mathsf{EQ}^{(W/c)}$

### SLL 表格

下面是一个不容易切分 bits 的表格，那就是移位运算，比如向左移位（Shift Left Logical, SLL），`0011 << 02 = 1100`。移位运算会给表格的切分带来些麻烦，因为移位运算会将低位分段的部分 bits 
移动到高位分段。

我们先考虑单个表格的 SLL 运算。下面的 $\mathsf{SLL}_k(\vec{X})$ 表示把长度为 $W$ 的位向量 $\vec{X}$ 向左移 $k$ 位的运算：

$$
\mathsf{SLL}^{(W)}_k(\vec{X}) = \sum_{j=0}^{W-k-1}2^{j+k}\cdot X_j
$$

这个定义比较容易理解，对于 $W$ 个 bits，我们只需要把低 $k$ 位中的每一个 $X_i, i\in[0,k)$，乘上 $2^{k+i}$，然后对这个 $k$ 个数进行求和。然后高位的 $W-k$ 位会溢出，因此被直接抛弃。这个表格比较容易定义是，因为我们将移位的位数作为一个常数。

接下来，我们考虑左移的位数是一个变量 $Y$。这时候，我们可以利用一个选择子，来根据 $Y$ 值来选择不同的 $\mathsf{SLL}_k$。

$$
\mathsf{SLL}^{(W)}(\vec{X}, \vec{Y}) = \sum_{k=0}^{W-1} \mathsf{EQ}^{(W)}(\mathsf{bits}(k), \vec{Y})\cdot \mathsf{SLL}_k(\vec{X})
$$

其中 $|\vec{Y}|=\log{W}$，因为 $Y$ 的取值范围是 $[0, W-1]$（在 RISC-V 的规范中，移位操作的位移为 $\log{W}$）。

例如我们要计算 $\mathsf{SLL}^{(4)}(1101, 10)$，

$$
\begin{array}{|c|c|c|}
 & \mathsf{EQ}^{(4)}(k, \vec{Y}) & \mathsf{SLL}_k(\vec{X})\\
\hline
k=0 & \mathsf{EQ}^{(4)}(00, 10) = 0  & \mathsf{SLL}_0(1101) = 1101  \\
k=1 & \mathsf{EQ}^{(4)}(01, 10) = 0  & \mathsf{SLL}_1(1101) = 1010  \\
k=2 & \mathsf{EQ}^{(4)}(10, 10) = {\color{red}1}  & \mathsf{SLL}_2(1101) = {\color{red}0100}  \\
k=3 & \mathsf{EQ}^{(4)}(11, 10) = 0  &  \mathsf{SLL}_3(1101) = 1010 \\
\end{array}
$$

等式可以展开为：

$$
\begin{split}
\mathsf{SLL}^{(4)}(1101, 10) & = \mathsf{EQ}^{(4)}(00, 10)\cdot \mathsf{SLL}_0(1101)  \\
& + \mathsf{EQ}^{(4)}(01, 10)\cdot \mathsf{SLL}_1(1101)   \\
& + \mathsf{EQ}^{(4)}(10, 10)\cdot \mathsf{SLL}_2(1101)   \\
& + \mathsf{EQ}^{(4)}(11, 10)\cdot \mathsf{SLL}_3(1101)   \\
& =  \mathsf{EQ}^{(4)}(10, 10)\cdot \mathsf{SLL}_2(1101) \\
& = 1\cdot 0100 \\
& = 0100
\end{split}
$$

如果 $W=64$，我们需要把表格拆分为 $c$ 个 $W/c$-bit 子表格。我们用一个例子（$W=16, c=4, \log{W}=4$）来说明，`0101 1100 1001 1010 << 0110`，这个移位操作是向左移 6 位，假如我们需要将这个表格拆分为 $c=4$ 个子表格，每个子表格计算的位数为 $W/c=4$。
这样，每个子表格 $\mathsf{SLL}^{(4)}(\vec{x}, \vec{y})$ 长度为 $2^{W/c+log{W}}=2^{4+4} = 2^8$，每个表格项的位长为 $W$。那么这个移位操作相当于是 $c=4$ 个移位运算结果的**求和**：例如我们给出这个子表格的表项示例：

$$
\begin{array}{|c|c|c|}
\vec{x} & \vec{y} & \mathsf{SLL}^{(4)}(\vec{x}, \vec{y}) \\
\hline
0000 & 0000 & 0000\ 0000\ 0000\ 0000\\
\vdots & \vdots & \vdots\\
0001 & 0001 & 0000\ 0000\ 0000\ 0010\\
\vdots & \vdots & \vdots\\
0001 & 0101 & 0000\ 0000\ 0010\ 0000\\
\vdots & \vdots & \vdots\\
1001 & 0011 & 0000\ 0000\ 0100\ 1000\\
\end{array}
$$

$\mathsf{SLL}^{(16)}({(0101 1100 1001 1010)}_{2}, {0110}_{2})$ 的计算过程看下表：

$$
\begin{array}{llrr|rr}
 0101 << 6 &= & {\color{red}01} & {\color{red}0100} & 0000 & &\\
 1100 << 6 &= &    & {\color{red}11} &0000 &0000 \\
 1001 << 6 &= && & 10 &0100 &0000 & & \\
 1010 << 6 &= &&&&10 & 1000 &0000\\
 \mathsf{SLL}^{(16)}(\vec{x}, 0110) & = & {\color{red}01} & {\color{red}0111} & 0010 & 0110 & 1000 & 0000 \\
\end{array}
$$

这里红色标记的部分为溢出 $(W=16)$ 部分。根据 SLL 的语义，我们要抛弃这部分的 bits。剩下的部分，我们把它们错位（乘上 2 的次幂 ）相加。由于不同的分段溢出的 bits 数量不等，我们需要先给出一个定义，$m_{i, k}$ 来标记每一个分段中溢出的 bits 数量：

$$
m_{i, k} = \mathsf{min}\Big(W/c, \mathsf{max}\big(0, k+(W/c)\cdot (i+1) - W\big)\Big), \qquad \forall i\in[0, c), k\in[0, \log{W})
$$

这里 $k$ 为移位的位数，$i$ 为分段的索引。然后 $\mathsf{SLL}_i^{(W/c)}$ 子表格的定义如下：

$$
\mathsf{SLL}_i^{(W/c)}(\vec{X}, \vec{Y}) = \sum_{k=0}^{W-1}\mathsf{EQ}^{(W)}(\mathsf{bits}(k), \vec{Y})
\cdot \left(\sum_{j=0}^{W/c-m_{i,k}-1}2^{j+k}\cdot X_j\right)
$$

最后我们给出 $\mathsf{SLL}^{(W/c)}$ 的分解定义：

$$
\mathsf{SLL}^{(W/c)}(\vec{X}_{c-1}\parallel\cdots\parallel\vec{X}_0,\ \vec{Y}) = \sum_{i=0}^{c-1}2^{i\cdot W/c}\cdot\mathsf{SLL}^{W/c}_i(\vec{X}_i,\vec{Y})
$$
 
应用 Lasso 框架来实现移位运算，参数化的多项式 $G(\cdot)$ 可以依据上面的等式来定义。

----


### 表格的拆分

Lasso 协议之所以可以支持一个巨大表格的 Lookup，核心原理是将巨大表格拆解成若干个小表格，例如把一个 $2^{128}$ 大小的 XOR 表格可以拆分为 8 个并列的长度为 $2^{16}$ 的 XOR 表格。因为两个 64bit 整数的 XOR运算可以拆解成 8对 8bit 整数的 XOR 运算。

Lasso 支持两类巨大的表格，一是可以拆分的，另一类表格被称为  MLE-structured 表格。我们将在后续文章里面讨论后者这类表格的证明。

假如一个巨大表格可以拆分成 $k$ 个子表格：$T_1, T_2, \ldots, T_k$

$$
T(\vec{X}_0, \vec{X}_1, \ldots, \vec{X}_{c-1}) = g\left(
    \begin{array}{lll}
    T_0[\vec{X}_0], &T_1[\vec{X}_0], &\ldots, & T_{k-1}[\vec{X}_{0}], \\ 
    T_0[\vec{X}_1], &T_1[\vec{X}_1], &\ldots, & T_{k-1}[\vec{X}_{1}], \\ 
    \vdots & \vdots & \ddots & \vdots \\
    T_0[\vec{X}_{c-1}], &T_1[\vec{X}_{c-1}], &\ldots, &T_{k-1}[\vec{X}_{c-1}]\\
\end{array}\right)
$$

表格 $T$ 的索引也可以拆分成 $c$ 段，并且表格 $T$ 的每一项都可以通过 $k$ 个子表格来计算。那么
我们把这类表格称为 SOS(Spark-Only-Structured) 结构的表格，或者被称为「可被分解表格」(Decomposable Table)。

于是，Prover 需要计算 $k\cdot c$ 个子表格读取日志，编码成 $\log(m)$-variate-MLE 多项式 $e_0(\vec{X}), e_1(\vec{X}),\ldots, e_{kc}(\vec{X})$，并且产生承诺，$E_0, E_1, \ldots, E_{kc}$。

### 协议

