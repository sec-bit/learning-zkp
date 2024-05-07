# 理解 Lasso (四)：更多的可分解表格

Jolt 论文给出了更多的可分解表格，用于表达 RISC-V 指令的计算过程。


## 更多的表格拆分案例

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
\mathsf{EQ}^{(W/c)}(\vec{x}, \vec{y}) = \prod_{i=0}^{(W/c)-1} \Big(x_i\cdot y_i + (1-x_i)\cdot (1-y_i)\Big)
$$

这个定义是说，如果 $x_i$ 和 $y_i$ 相等，那么 $x_i\cdot y_i + (1-x_i)\cdot (1-y_i)=1$，否则为 $0$。那么 ${\mathsf{EQ}}^{(W/c)}$ 表格的定义就是把 $W$ 位的表格拆分成 $c$ 个 $W/c$ 位的子表格，然后把这些子表格的结果相乘：

$$
{\mathsf{EQ}}^{(W)}(\vec{X}, \vec{Y})=\prod_{i=0}^{c-1}{\mathsf{EQ}}^{(W/c)}(\vec{X}_i, \vec{Y}_i)
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

如果要采用 Lasso 框架来证明关于 $\mathsf{LTU}^{(W)}(\vec{x}, \vec{y})$ 表格的 Lookup，我们需要定义下给出参数化的多项式 $\mathcal{G}(\cdot)$：

$$
\mathcal{G}\Big(
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
 
应用 Lasso 框架来实现移位运算，参数化的多项式 $\mathcal{G}(\cdot)$ 可以依据上面的等式来定义。

