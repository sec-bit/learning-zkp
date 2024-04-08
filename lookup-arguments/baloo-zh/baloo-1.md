# Baloo, lookup (Part 1)

Baloo 在 Caulk 和 Caulk+ 基础上的改进协议，支持大表格的查询证明，是一个承上启下的 Lookup Argument 协议。本文是第一部分，介绍 Baloo 协议的基本思想。

## 1. Baloo 概述

假设有一个表格为 $\vec{t}$，其大小为 $N$。同时有 $m$ 次的查询，设为 $\vec{a}$。我们用符号 $\{\vec{a}\}$ 表示向量中所有元素构成的集合，使用符号 $\{\vec{a}\}\subset \{\vec{t}\}$ 表示向量元素构成的集合之间的子集关系，同时使用符号 $\vec{f}\subset\vec{t}$ 表示 $\vec{f}$ 是 $\vec{t}$ 的子向量。

我们要构造一个证明系统，使得：  

$$
\forall a_i \in \{\vec{a}\}, a_i \in \{\vec{t}\}
$$

即 $\{\vec{a}\}\subset \{\vec{t}\}$。 但这里请注意， 我们要考虑到 $\vec{a}$ 中可能出现重复的查询元素。

Baloo 之前的 Lookup 协议，如 Caulk/Caulk+ 支持大表格的查询，并使 Prover 所付出的证明开销只与查询的数量有关，即 Prover Time 是一个关于 $m$ 的函数。Caulk/Caulk+ 协议能做到将 Prover 的证明开销降低到 $O(m^2)$，而随后的 Flookup 协议则进一步改进了证明开销，使其降低到了 $O(m\log^2{m})$。但 Flookup 协议的一个主要缺点是，表格的承诺失去了以往 Lookup Argument （如 Plookup, Halo2-lookup，Caulk/Caulk+） 中表格承诺「加法同态」的性质，从而仅能支持「单列表」的查询证明。而在实践中，多列表格查询非常多见。

本文介绍的 Baloo 协议则是在与 Flookup 保持相似证明开销 $O(m\log^2{m})$的前提下，同时依然保持了 Caulk/Caulk+ 协议中表格承诺的加法同态性质，从而恢复支持多列表格查询。

Baloo 的基本思想是将 Lookup Argument 归约到一个线性关系：Matrix-Vector Multiplication Argument，即「矩阵向量乘积证明」。换句话说，如果查询关系成立 $\{\vec{a}\} \subset \{\vec{t}\}$，那么存在一个 $m\times N$ 的矩阵 $M$，使得下面的乘法成立：

$$
M \vec{t} = \vec{a}
$$

并且这里的矩阵 $M$ 有一个特别的性质，其每一行都是一个「单位向量」（Unit Vector），即行向量中仅有一个元素为 $1$，其余元素为 $0$。这个性质可以保证每一次查询仅仅是对表格中「一个特定元素」的直接拷贝（而不是若干个表格元素的线性组合）。这个性质非常重要，后面的协议将会充分利用这个特征，请大家先谨记在心。

举个简单例子，假设表格为 $t=(1,2,3,4,5,6)^{\top}$，而查询记录为 $a=(2,4,6)^{\top}$，那么我们可以构造一个 $3\times 6$ 的矩阵 $M$，使得：

$$
M \vec{t}= \begin{bmatrix}
0 & 1 & 0 & 0 & 0 & 0 \\
0 & 0 & 0 & 1 & 0 & 0 \\
0 & 0 & 0 & 0 & 0 & 1 \\
\end{bmatrix}  
\begin{bmatrix}
1 \\
 2 \\ 
 3 \\ 4 \\ 5 \\ 6 \\
\end{bmatrix} =
\begin{bmatrix}
 2 \\ 
 4 \\ 
 6 \\
\end{bmatrix} = \vec{a}
$$

为了使 Prover Time 与 $\vec{t}$ 的大小无关，我们需要先用一个子协议将 $t$ 中与 $\vec{a}$ 相关的元素分离出来，记为 $\vec{t}_I$，然后证明这个子表和主表的「子向量」关系，然后再采用 Matrix-Vector multiplication argument 证明 $\vec{t}_I$ 和 $\vec{a}$ 的关系。

$$
M \vec{t}_I = \vec{a} \wedge \vec{t}_I \subset \vec{t}
$$

而 MVMA（Matrix-Vector multiplication argument）在基于 R1CS 的 zkSNARK 构造中经常出现，但这里的 MVMA 又有所不同，关键区别是：在 R1CS zkSNARK 的构造中，矩阵 $M$ 表示电路的拓扑，是一个公开值；而在 Baloo 协议中，$M$ 是一个 Prover 临时构造的中间值，如果要进一步考虑隐私保护， $M$ 还需要混入足够的随机值来确保其信息不被 Verifier 得到。这就引入了比较有挑战性的问题：

**难点一**：在证明 $M\vec{t}_I=\vec{a}$ 时，至少 $M$ 和 $\vec{t}_I$ 都是隐藏值，Verifier 只能得到他们的 Oracle。进一步，如果我们把 Baloo 作为 Plonk 的子协议，替代 Plookup 协议，那么 $\vec{a}$ 也是隐藏值。这要求 Prover 要在 Verifier 仅能得到 Oracle 的情况下，证明这个乘法的正确性。

**难点二**： 这个对 Verifier 不可见的矩阵还具有一个特殊性：其中每一行有且仅能有一个非零元素，并且这个非零元素只能等于 $1$。换句话说，矩阵的每一行是一个 Selector，仅能选择表格 $\vec{t}_I$ 中的一个元素。这要求 Prover 在证明矩阵向量乘积关系的同时，还要能证明这个矩阵 $M$ 具有这个特殊性质。

**难点三**：如果我们对 $M$ 的内部结构不加考虑，而直接采用通用的方法来证明矩阵向量乘积，那么 Prover 需要遍历矩阵的每一个元素，证明开销将至少是 $O(m*n)$。假设子表格的大小为 $n=O(m)$ ，那么证明开销复杂度为 $O(m^2)$，计算量偏大。

## 2. 核心思想：向量空间的随机抽样

为了方便理解 Baloo 协议的核心思想，我们先简化下问题，去除「子表格抽取问题」的干扰，而先假设子表格和原表格完全相同 $\vec{t}=\vec{t}_I$，但保留一个关键特性：表格 $\vec{t}$ 需要对 Verifier 隐藏。

先假设表格 $\vec{t}$ 的大小记为 $n$，所以 $n=|\vec{t}|$，查询记录的数量为 $m=|\vec{a}|$。我们的目标是能证明：存在一个特殊的矩阵 $M\in\mathbb{F}^{m\times n}$，满足下面的矩阵向量乘法关系：

$$
M\vec{t} = \vec{a}
$$

协议中，Verifier 拥有表格和查询两个向量的多项式承诺，即公开输入 $[t(X)]$ 与 $[a(X)]$。

### 2.1 多项式编码

这两个多项式 $t(X)$ 与  $a(X)$ 利用 Lagrange Basis 编码了 $\vec{t}$ 与 $\vec{a}$ 两个向量。我们假设在素数域 $\mathbb{F}_p$ 中有一个 FFT 平滑的乘法子群 $\mathbb{H}\subset \mathbb{F}_p^*$，其大小恰好为 $n$，生成元为 $\omega$。所谓的 FFT 平滑是指，$\mathbb{H}$ 的大小恰好为 $2^k$，因此我们可以得到 $\omega^{2^k}=1$， $\omega$ 为 $n$ 次单位根，在这样的 Domain 上，我们可以实现复杂度为 $O(n\log{n})$ 的 FFT算法。

我们可以定义 $\mathbb{H}$ 上的 Lagrange 多项式：

$$
\tau_i(X) = \frac{z_\mathbb{H}(X)}{z_\mathbb{H}'(\omega^i)\cdot(X - \omega^i)} , \qquad i=0,1,\ldots,n-1
$$

这里 $z_\mathbb{H}(X)$ 为 Vanishing 多项式，定义为：$z_\mathbb{H}(X) = \prod_{i=0}^{n-1}(X-\omega^i)$。而 $z_\mathbb{H}'(X)$ 则为 $z_\mathbb{H}(X)$ 的导数多项式，由于 $\mathbb{H}$ 为乘法子群，因此 

$$
z_\mathbb{H}'(\omega^i)=(X^n-1)'|_{X=\omega^i}=n\cdot (X^{n-1})|_{X=\omega^i} = n\cdot (\omega^i)^{n-1} = n\cdot \omega^{-i}
$$

因此，Lagrange 多项式 $\tau_i(X)$ 的定义可以简化为：

$$
\tau_i(X) = \frac{\omega^i\cdot z_\mathbb{H}(X)}{n\cdot(X - \omega^i)}, \qquad i=0,1,\ldots,n-1
$$

Baloo的关键思路是引入一个新的向量 $\vec{d}$，它是矩阵 $M$ 「行空间」上的「随机抽样」。在引入 $\vec{d}$ 的定义之前，我们先用一个「二元多项式」编码矩阵 $M$ ：

$$
M(X, Y) = \begin{bmatrix}
\mu_0(X), \mu_1(X), \cdots, \mu_{m-1}(X) \\
\end{bmatrix}
\begin{bmatrix}
M_{0,0} & M_{0,1} & \cdots & M_{0,n-1} \\
M_{1,0} & M_{1,1} & \cdots & M_{1,n-1} \\
M_{2,0} & M_{2,1} & \cdots & M_{2,n-1} \\
\vdots & \vdots & \ddots & \vdots \\
M_{m-1,0} & M_{m-1,1} & \cdots & M_{m-1,n-1} \\
\end{bmatrix}
\begin{bmatrix}
\tau_0(Y) \\
\tau_1(Y) \\
\tau_2(Y) \\
\vdots \\
\tau_{n-1}(Y) \\
\end{bmatrix}
$$

为了编码二元多项式，我们还需要另一个 FFT 平滑的乘法子群 $\mathbb{V}\subset \mathbb{F}_p^*$，其大小恰好为 $m=2^l$，生成元为 $\nu$。同样，这个 Domain 上的 Lagrange 多项式为：

$$
\mu_i(X) = \frac{z_\mathbb{V}(X)}{z_\mathbb{V}'(\nu^i)\cdot(X - \nu^i)}=\frac{\nu^i\cdot z_\mathbb{V}(X)}{m\cdot(X - \nu^i)}, \qquad i=0,1,\ldots,m-1
$$

其中  $z_\mathbb{V}(X)$ 为 Domain $\mathbb{V}$ 上的 Vanishing 多项式。

### 2.1 行空间抽样

所谓矩阵的「行空间」随机抽样，是指引入一个随机向量 $\vec{\rho}$ ，其中所有元素线性不相关，然后对矩阵的所有行进行线性组合所计算得到的一个向量。

观察上面 $M(X,Y)$ 的定义，我们可以直接使用一个随机数 $\alpha$，然后通过 Lagrange 多项式在 $X=\alpha$ 处的取值 $\mu_i(\alpha)$ 来作为这个抽样随机向量，然后对矩阵的所有行向量进行线性组合计算，得到 $\vec{d}$，其长度等于矩阵的宽 $|\vec{d}|=n$。这个计算可以看成对矩阵 $M$ 进行「纵向折叠」：

$$
\vec{d} =
\begin{bmatrix}
d_0, d_1,\ldots, d_{n-1}
\end{bmatrix} = \begin{bmatrix} 
\sum_{i=0}^{m-1} \mu_{i}(\alpha)M_{i,0}, & \sum_{i=0}^{m-1} \mu_{i}(\alpha)M_{i,1}, & \cdots, & \sum_{i=0}^{m-1} \mu_{i}(\alpha)M_{i,n-1} \\
\end{bmatrix}
$$

我们可以引入一个辅助的多项式 $d(X)$ 在 $\mathbb{H}$ 上对这个向量进行编码，

$$
d(X) = \sum_{j=0}^{n-1}\sum_{i=0}^{m-1} \mu_{i}(\alpha)~M_{i,j}\cdot \tau_j(X)
$$

显然 $d(X)$ 和 矩阵二元多项式 $M(X, Y)$ 之间应该满足下面的关系：

$$
d(Y) = M(\alpha, Y)
$$

上面公式的右边等价于：二元多项式 $M(X, Y)$ 在 $X=\alpha$ 处的「部分运算」（Partial Evaluation）。

有了这个辅助的多项式 $d(X)$，我们就可以把 Lookup Argument 证明目标归约到三个待证明的子目标：

- **子目标一** ：证明 $\vec{d}$ 确实为矩阵 $M$ 的行空间随机抽样，即 $d(X) = M(\alpha, X)$

- **子目标二** ：证明矩阵 $M$ 的每一行是一个 Unit Vector，即有且只有一个 $1$

- **子目标三** ：证明向量 $\vec{d}$ 满足查询关系：$\vec{d}\cdot \vec{t} = a(\alpha)$

先看**子目标一**，为了证明 $\vec{d}$ 的合法性，我们要证明 $d(X) = M(\alpha, X)$，根据 Schwartz-Zippel 定理，这等价于证明：

$$
d(\beta) = M(\alpha, \beta)
$$

这里 $\beta$ 为 Verifier 提供的另一个随机挑战值。问题是我们如何避免让 Prover 直接计算 $M(X, Y)$ 这个二元多项式的承诺，因为其计算量为 $O(m\cdot n)$。这就需要我们利用矩阵 $M$ 的稀疏特性，因为其仅有 $m$ 个 $1$。这和上面的 **子目标二** 非常相关。

考虑到 $M$ 矩阵由若干个 Unit Vector 组成，因此我们可以利用这个稀疏性，重新定义下 $M$ 多项式：

$$
M(X, Y) = \sum_{i=0}^{m-1} \mu_{i}(X) \cdot \tau_{\color{red} \mathsf{col}(i)}(Y)
$$

这个定义里中只对 $M$ 中那些值为 $1$ 的矩阵单元进行求和，由于每一行有且仅有一个 $1$，因此我们只需要考虑每一行的 $1$ 出现在哪一列即可。我们引入一个函数 $\mathsf{col}(i)$，表示矩阵中第 $i$ 个 $1$ 所在的列的编号，显然 

$$
\mathsf{col}(i)\in [0,n),\quad \forall i\in[0,m)
$$

这里 $\tau_{\mathsf{col}(i)}(Y)$ 可以解释为：在第 $i$ 行中，只有 $c=\mathsf{col}(i)$ 处为 $1$，其余位置为 $0$，因此仅有 $\tau_c(Y)$ 被选出，其余的 Lagrange 多项式 $\tau_{j, j\neq{}c}(Y)$ 和零相乘，因此可以被消去。

举个简单的例子：假设表格为 $t=(1,2,3,4,5,6)^{\top}$，而查询记录为 $a=(2,4,6)^{\top}$，

$$
M \vec{t}= \begin{bmatrix}
0 & 1 & 0 & 0 & 0 & 0 \\
0 & 0 & 0 & 1 & 0 & 0 \\
0 & 0 & 0 & 0 & 0 & 1 \\
\end{bmatrix}  
\begin{bmatrix}
1 \\
 2 \\ 
 3 \\ 4 \\ 5 \\ 6 \\
\end{bmatrix} =
\begin{bmatrix}
 2 \\ 
 4 \\ 
 6 \\
\end{bmatrix} = \vec{a}
$$

我们看第一行（$i=0$）的 $1$ 所在的列为 $1$，因此 $\mathsf{col}(0)=1$。第二行（$i=1$）的 $1$ 所在的列为 $3$，因此 $\mathsf{col}(1)=3$ 。 同理 $\mathsf{col}(2)=5$，因此矩阵 $M$ 可以用稀疏的方式定义如下：

$$
M(X, Y) = \mu_0(X)\cdot \tau_{\color{red}1}(Y) + \mu_1(X)\cdot \tau_{\color{red}3}(Y) + \mu_2(X)\cdot \tau_{\color{red}5}(Y)
$$

那么，

$$
d(X) = M(\alpha, X) = \mu_0(\alpha)\cdot \tau_1(X) + \mu_1(\alpha)\cdot \tau_3(X) + \mu_2(\alpha)\cdot \tau_5(X)
$$

如果我们能证明 $M(X,Y)$ 的定义为 $\sum \mu_{i}(X) \cdot \tau_j(Y)$ 这种形式，那么我们相当于证明了**子目标二**，即 $M$ 的每一行是一个 Unit Vector。其实，我们可以略微弱化一下子目标二，只要我们能证明 $d(X)$ 的定义满足 $\sum \mu_{i}(\alpha) \cdot \tau_j(X)$ 即可。

那么回到刚才的问题，如何证明矩阵 $M(\alpha, X)$ 等价于 $\sum_i \mu_{i}(\alpha) \cdot \tau_{\mathsf{col}(i)}(X)$? 

注意到 $M(\alpha, X)$ 的定义不能看成是 $\{\mu_i(\alpha)\}_i$ 在 $\mathbb{H}$ 上的插值多项式，这是因为：

1. 矩阵 $M$ 每一行所选择的元素并没有铺满整个 $\vec{t}$，因此 $\{\tau_{\mathsf{col}(i)}(X)\}_i$ 只是在 $\mathbb{H}$ 上的 Lagrange 多项式集合的一部分，

2. 矩阵 $M$ 的两行可能选择同一个 $\vec{t}$ 中的元素，因此 $\{\tau_{\mathsf{col}(i)}(X)\}_i$ 中会有重复元素。

不过，如果允许我们交换一下 $\alpha$ 和 $X$，那么 $M(X, \alpha)=\sum \tau_{\mathsf{col}(i)}(\alpha)\cdot \mu_{i}(X)$ 倒是可以看成 $\tau_{\mathsf{col}(i)}(\alpha)$ 在 $\mathbb{V}$ 上的插值多项式。为了不重复使用 Verifier 的随机数 $\alpha$，我们让 Verifier 再提供一个新鲜的随机数 $\beta$，于是我们就可以利用插值多项式的对称性，证明 $M(X, Y)$ 满足 $M(X, \beta)=\sum \tau_{\mathsf{col}(i)}(\beta)\cdot \mu_{i}(X)$。

这里就需要我们引入另一个辅助向量 $\vec{e}$，其长度为 $m$：

$$
e_i = \tau_{\mathsf{col}(i)}(\beta), \quad i\in[0, m)
$$

这个向量 $\vec{e}$ 恰好是矩阵 $M$ 在列空间上的随机抽样：

$$
\vec{e} = 
\begin{bmatrix}
e_0\\
e_1\\
e_2\\
\vdots \\
e_{m-1}
\end{bmatrix} = 
\begin{bmatrix}
\tau_{\mathsf{col}(0)}(\beta)\\
\tau_{\mathsf{col}(1)}(\beta)\\
\tau_{\mathsf{col}(2)}(\beta)\\
\vdots \\
\tau_{\mathsf{col}({m-1})}(\beta)
\end{bmatrix}
$$ 

辅助向量 $\vec{e}$ 编码成多项式 $e(X)$，我们可以得到：

$$
e(X)= \sum_{i=0}^{m-1} \tau_{\mathsf{col}(i)}(\beta)\cdot \mu_{i}(X) = M(X, \beta)  
$$

如果 $\vec{e}$ 是正确的，那么下面的等式一定成立：

$$
d(\beta) = M(\alpha, \beta) = e(\alpha)
$$

### 2.2 列空间抽样

这里 $e(X)$ 也是 $M(X, Y)$ 的部分运算（Partial Evaluation），只是关于另一个未知数，即 $Y=\beta$。它也可以被看作为矩阵 $M$ 在列空间上的随机抽样 $\vec{e}$，再次通过 Lagrange 插值得到的多项式。而通过约束等式 $d(\beta) = e(\alpha)$ 则可以保证 $M$ 的行空间和列空间的随机抽样是一致的，即如果 $\vec{e}$ 是矩阵正确的列空间抽样，那么 $\vec{d}$ 也是同一个矩阵的正确的行空间抽样。因而我们只要能证明 $e(X)$ 的正确性，加上证明 $d(\beta) = e(\alpha)$， 那么就可以证明 $d(X)$ 的正确性（即上面的**子目标一**）。如果我们还能够证明 $\vec{e}$ 中的元素都形如 $\tau_{\mathsf{col}(i)}$，那么就可以证明矩阵 $M$ 的特殊性（即上面的**子目标二**）。

问题还没被解决，我们只是找到了一条路，如何把 $\vec{d}$ 的正确性证明归约到了 $\vec{e}$ 的正确性证明。接下来是如何利用 Lagrange 插值多项式的对称性来证明 $e(X)$ 的正确性呢？

观察下 $e(X)$ 的定义：

$$
\begin{split}
e_i &= \tau_{\mathsf{col}(i)}(\beta)\\
 &=  \frac{\omega_{\mathsf{col}(i)}}{m}\cdot\frac{z_\mathbb{H}(\beta)}{\beta - \omega_{\mathsf{col}(i)}} \\
 &= \frac{z_\mathbb{H}(\beta)}{m}\cdot \frac{\omega_{\mathsf{col}(i)}}{\beta - \omega_{\mathsf{col}(i)}} \\
&= \frac{z_\mathbb{H}(\beta)}{m}\cdot \Big(\frac{1}{\beta\cdot\omega^{-1}_{\mathsf{col}(i)}-1}\Big) \\
 \end{split}
$$

如果我们考虑引入一个辅助向量 $\vec{v}$， 其中元素 $v_i=\omega^{-1}_{\mathsf{col}(i)}$，

那么我们可以得到下面的等式：

$$
m\cdot e_i\cdot (\beta\cdot v_i-1) - z_\mathbb{H}(\beta) = 0, \quad i\in[0, m)
$$

我们定义 $\vec{v}$ 的插值多项式为 $v(X) = \sum_{i=0}^{m-1}\omega^{-1}_{\mathsf{col}(i)}\cdot \mu_i(X)$，然后我们可以得到 $e(X)$ 和 $v(X)$ 的关系：

$$
 e(X)\cdot(\beta\cdot v(X) -1 ) - \frac{z_\mathbb{H}(\beta)}{m}= q(X)\cdot z_\mathbb{V}(X)
$$

实际上， **$\vec{v}$ 正是矩阵 $M$ 的编码**，并同时表达了矩阵 $M$ 的特殊性。因为矩阵 $M$ 的每一行只有一个 $1$，其余为零，那么所有行中只有 $m$ 个 $1$，这个编码方式是证明**子目标二**的关键，即如果 $M$ 矩阵不满足这个特殊性，$\vec{v}$ 就无法被正确构造。

任意一个矩阵 $M$ 都对应了唯一的 $\vec{v}$。因此上面的等式实际在证明 $\vec{e}$ 是矩阵 $M$ 的列空间随机抽样。

不过我们还需要一个对 $e(X)$ 的重要约束：

$$
\deg(e(X))< m
$$

这个 degree bound 证明实际起的作用是确保 $e(X)=\sum_i e_i\cdot \mu_i(X)$，结合上面的 $e(X)$ 和 $v(X)$ 的关系等式，我们可以得出下面的结论：

$$
e(X)=\sum_i\Big(\frac{z_\mathbb{H}(\beta)}{m}\cdot \frac{1}{\beta\cdot v_i - 1}\Big)\cdot \mu_i(X)
$$ 

如果再加上证明 $e(\alpha)\overset{?}{=}d(\beta)$，我们可以反过来确保 $\vec{d}$ 确实是矩阵 $M$ 的行空间抽样。不过，这个结论为什么成立呢？我们来推导下其中的过程。

### 2.3 行空间与列空间的一致性

上面我们已经推导出了结论：

$$
e(X)=\sum_i\frac{z_\mathbb{H}(\beta)}{m}\cdot \frac{1}{\beta\cdot v_i - 1}\cdot \mu_i(X)
$$ 

那么如果也能证明「行列空间的一致性」： $e(\alpha)=d(\beta)$，那么由上面的结论可以推出：

$$
e(\alpha)=\sum_i\frac{z_\mathbb{H}(\beta)}{m}\cdot \frac{1}{\beta\cdot v_i - 1}\cdot \mu_i(\alpha) = d(\beta)
$$

那么，由 Schwartz-Zippel 定理，我们把上面的结论看成是多项式在 $X=\beta$ 处的检验，那么我们可以到下面的结论：

$$
\begin{split}
d(X) &= \sum_i\frac{1}{m}\cdot \frac{z_\mathbb{H}(X)}{X\cdot v_i - 1}\cdot \mu_i(\alpha) \\
&= \sum_i\frac{v_i^{-1}\cdot \mu_i(\alpha)}{m}\cdot \frac{z_\mathbb{H}(X)}{X - v_i^{-1}} \\
&= \sum_i {c_i}\cdot \frac{z_\mathbb{H}(X)}{\color{red} X - v_i^{-1}}
\end{split}
$$

我们接下来论证：$v_i^{-1}\in\mathbb{H}$。

显然，上面 $d(X)$ 等式的右边是一个关于「有理分式」（Rational Function）的线性组合。如果 $v_i^{-1}\not\in\mathbb{H}$，那么 $(X- v_i^{-1})$ 不能整除 $z_\mathbb{H}(X)$，那么 $d(X)$ 就不是一个多项式，即 $d(X)\not\in \mathbb{F}_p[X]$。于是，我们无法计算 $d(X)$ 的多项式承诺。 

因此，我们只需要要求 Prover 提供 $d(X)$ 的承诺，就能得到结论 $v_i^{-1}\in\mathbb{H}$，从而证明 Prover 提供的 $v(X)$ 多项式（承诺）中的每一项都形如 $v_i = \omega^{-1}_j$。

接下来，我们可以得出 $d(X)$ 满足下面的关系：

$$
d(X) = \sum_{i=0}^{m-1}\Big(\frac{{\color{red}\omega_j}}{m}\cdot \frac{z_\mathbb{H}(X)}{X - {\color{red}\omega_j}}\Big)\cdot \mu_i(\alpha) = \sum_{i=0}^{m-1}\tau_j(X)\cdot\mu_i(\alpha) 
$$

这就证明了 $d(X)$ 所编码的向量 $\vec{d}$ 确实是某个矩阵 $M$ 的行空间随机抽样。



### 2.4 CP-expansion

至此，证明了 **子目标一** 与 **子目标二**。这个证明协议被称为是 CP-expansion 协议。

回顾一下，我们从一个辅助向量 $\vec{e}$ （矩阵的列空间抽样）出发，证明 $\vec{e}$ 和另一个辅助向量，矩阵 $M$ 的稀疏编码向量 $\vec{v}$，满足矩阵的特殊关系；然后通过证明 $d(\beta)=e(\alpha)$，验证 $\vec{v}$ 中元素的合法性，以及 $d(X)$ 所编码向量是一个合法的矩阵 $M$ 的行空间抽样。

从定义上看，$d(X)$ 和 $e(X)$ 具有一定的对称性，CP-expansion 协议正是利用了这个对称性：

$$
e(X) = M(X, \beta) = \sum_{i=0}^{m-1} \tau_{\mathsf{col}(i)}(\beta)\cdot \mu_{i}(X)
$$

$$
d(X) = M(\alpha, X) = \sum_{i=0}^{m-1} \tau_{\mathsf{col}(i)}(X)\cdot \mu_{i}(\alpha)
$$

由于 $M$ 的特殊性（每一行是一个 Unit Vector，即每行有且仅有一个非零元素 $1$），
Lagrange 多项式也恰好编码了一组单位向量，因此 $e_i$ （对应矩阵的第 $i$ 行）正好是一个 Lagrange 多项式 $\tau_{\mathsf{col}(i)}(X)$ 的随机抽样。而 $d_i$ 则是矩阵的第 $i$ 列，它正好是多个（可以重复的） Lagrange 多项式  $\tau_{\mathsf{col}(i)}(X)$ 的随机线性组合。


与一般的 Matrix-vector Multiplication Argument 协议不同的是，这里的矩阵 $M$ 是采用隐藏的方式由 prover 提供承诺，可以被认为是一种 Commit-and-prove 方式。所以我们在协议前面加上 `CP-`。下面是 CP-expansion 协议的交互细节。


**公共输入**：

1. 矩阵 $M$ 的编码 $\vec{v}$ 的多项式承诺：$[v(X)]_1$
2. Verifier 提供的行空间抽样用的随机数 $\alpha\in\mathbb{F}^*_p$
3. 矩阵 $M$ 的行空间抽样向量 $\vec{d}$ 的多项式承诺 $[d(X)]_1$，

**证明目标**：

1. $d(X) = \sum_{i=0}^{m-1}\tau_{\mathsf{col}(i)}(X)\cdot\mu_i(\alpha) $ 
2. $v(X) = \sum_{i=0}^{m-1}\omega^{-1}_{\mathsf{col}(i)}(X)\cdot\mu_i(X)$

**第一步**: Prover 计算 $e(X)$, $q(X)$，并发送 $\big([e(X)]_1, [q(X)]_1, [X^{D-n+1}\cdot e(X)]_1\big)$

$$
e(X)\cdot(\beta\cdot v(X) -1) - \frac{z_\mathbb{H}(\beta)}{m} = z_\mathbb{V}(X)\cdot q(X)
$$

**第二步**: Verifier 发送一个随机挑战值 $\zeta\in\mathbb{F}$

**第三步**: Prover 计算 $P(X)$，并发送 $\Big(v_1=e(\zeta), v_2=v(\zeta), v_3=q(\zeta) \Big)$

**第四步**: Verifier 发送一个随机挑战值 $\gamma\in\mathbb{F}$

**第五步**: Prover 计算并发送 $[w(X)]_1$

$$
w(X) = \frac{e(X) - v_1}{X-\zeta} + \gamma\cdot \frac{v(X) - v_2}{X-\zeta} + \gamma^2\cdot \frac{q(X) - v_3}{X-\zeta}
$$

**验证**: Verifier 验证下面两个等式是否成立：

$$
[w(X)]_1\ast[X]_2 \overset{?}{=} \Big([e(X)]_1 + \gamma[v(X)]_1 + \gamma^2[q(X)]_1  + \zeta[w(X)]_1 - (v_1+\gamma v_2+ \gamma^2 v_3)\cdot[1]_1\Big)\ast [1]_2
$$

$$
[X^{D-n+1}\cdot e(X)]_1 \ast [1]_2 \overset{?}{=} [e(X)]_1 \ast [X^{D-n+1}]_2
$$

## 3. Univariate Sumcheck 

还剩下 **子目标三** 这个 Dot Product 证明，即证明 $\vec{d}\cdot \vec{t} = a(\alpha)$。证明点积的方法其实有不少，比如 Flookup 中采用的是 Grand Sum Argument，而 Baloo 采用的是基于 Univariate Sumcheck 的方案。

所谓 Univariate Sumcheck 是指，对任意的多项式 $f(X)\in\mathbb{F}[X]$，如果存在一个 FFT 平滑的乘法子群 $\mathbb{H}\subset\mathbb{F}_p^*$，并且 $|\mathbb{H}|=n，$那么下面的等式成立：

$$
f(X) = \frac{\sigma}{n} + X\cdot g(X) + z_\mathbb{H}(X)\cdot q(X), \qquad \deg(r(X))<n-1
$$

其中 $\sigma$ 为多项式在 $\mathbb{H}$ 上的运算求值之和，即

$$
\sigma = \sum_{i=0}^{n-1}f(\omega^i) 
$$

我们可以重新陈述一下这个等式：多项式 $f(X)$ 在 $\mathbb{H}$ 上的求和恰好等于它除以 $z_\mathbb{H}(X)$ 之后的余数多项式的「常数项」，再乘以 $\mathbb{H}$ 的大小。

$$
\sigma = \sum_{i=0}^{n-1}f(\omega^i) = n\cdot (f(X) - X\cdot g(X) - z_\mathbb{H}(X)\cdot q(X))
$$

这个等式为什么成立？下面来推导下这个神奇的关系：

假设有一个任意次数的多项式 $f(X)$，它除以 $z_\mathbb{H}(X)$ 后的余数多项式 $r(X)$ 的次数一定小于 $\mathbb{H}$ 的大小。多项式的除法类似整数环上的除法，$13 \equiv 2 \pmod{5}$ 意味着 $13=5*2 + 3$，余数 $3<5$ 。同样，对任何一个多项式 $f(X)$ 都满足 $f(X) = q(X)\cdot z_\mathbb{H}(X) + r(X)$，其中 $\deg(r(X))<\deg(z_\mathbb{H}(X))=n$。

因此：

$$
r(X) = r_0 + r_1\cdot X + r_2 \cdot X^2 + \cdots r_{n-1} \cdot X^{n-1} = \sum_{j=0}^{n-1} r_j\cdot X^j
$$

并且余数多项式 $r(X)$ 与原多项式 $f(X)$ 还满足一个性质：两者在 $\mathbb{H}$ 上的运算结果保持一致：

$$
\forall \omega_i\in \mathbb{H}, f(\omega_i) = r(\omega_i)
$$ 

这个结论也非常容易证明。因为 $f(X) = q(X)\cdot z_\mathbb{H}(X) + r(X)$，对任意的 $\omega_i\in\mathbb{H}$，$f(\omega_i) = q(\omega_i)\cdot z_\mathbb{H}(\omega_i) + r(\omega_i)$ 必然成立，而 $z_\mathbb{H}(\omega_i) = 0$，代入可得 $f(\omega_i) = r(\omega_i)$。

然后我们把 $r(X)$ 的定义代入到求和公式中，验证 Univariate Sumcheck 等式：

$$
\begin{split}
\sigma = \sum_{i=0}^{n-1} f(\omega^i) &= \sum_{i=0}^{n-1} r(\omega^i) \\
 &= \Big(r_0 + r_1\cdot \omega + r_2 \cdot \omega^2 + \cdots r_{n-1} \cdot \omega^{n-1}\Big) \\
 & \ + \Big(r_0 + r_1\cdot \omega^2 + r_2 \cdot \omega^4 + \cdots r_{n-1} \cdot \omega^{2(n-1)}\Big) \\
 & \ + \Big(r_0 + r_1\cdot \omega^3 + r_2 \cdot \omega^6 + \cdots r_{n-1} \cdot \omega^{3(n-1)}\Big) \\
 & \ + \cdots \\
 & \ + \Big(r_0 + r_1\cdot \omega^{n-1} + r_2 \cdot \omega^{2(n-1)} + \cdots r_{n-1} \cdot \omega^{(n-1)(n-1)}\Big) \\
 & \ + \Big(r_0 + r_1\cdot \omega^n + r_2 \cdot \omega^{2n} + \cdots r_{n-1} \cdot \omega^{n(n-1)}\Big) \\
 &= n\cdot r_0 + r_1\cdot (\sum_i\omega^i) + r_2 \cdot (\sum_i(\omega^2)^i) + \cdots + r_{n-1} \cdot (\sum_i(\omega^{n-1})^i) \\
 &= n\cdot r_0 + r_1\cdot 0 + r_2 \cdot 0 + \cdots + r_{n-1} \cdot 0 \\
 &=  n\cdot r_0
\end{split}
$$

在上面的推导过程中，用到了 $\sum_{i=0}^{n-1}\omega^i = 0$ 这个一般性结论。这个结论可以通过下面的思路快速证明，定义一个多项式 $h(X)$ 如下，并满足 $h(X)\cdot (X-1) = X^n-1$，

$$
h(X) = 1 + X + X^2 + X^3 + \cdot + X^{n-1} = \frac{X^n-1}{X-1}
$$

那么，我们可以代入 $X=\omega$ 到 $h(X)$ 的定义中，并利用 $\omega$ 是单位根（Root of Unity）的前提条件，可得：

$$
h(\omega) = \sum_{i=0}^{n-1}\omega^i = \frac{\omega^n-1}{\omega-1} = \frac{1-1}{\omega-1} = 0
$$

进一步，当 $r(X)$ 减去 $r_0$ 后，可以 $r(X)$ 将会被 $X$ 整除，因此，我们可以得到：

$$
r(X) - r_0 = X\cdot g(X), \qquad \deg(g(X))<n-1
$$

将 $r_0=\sigma/n$ 代入，我们就得到了 Univariate Sumcheck 的等式：

$$
 f(X) \equiv r(X) = \frac{\sigma}{n} + X\cdot g(X) \pmod{z_\mathbb{H}(X)}
$$

即

$$
f(X) = \frac{\sigma}{n} + X\cdot g(X) + z_\mathbb{H}(X)\cdot q(X), \qquad \deg(r(X))<n-1
$$

### 3.1 Dot Product 协议

基于 Univariate Sumcheck 定理，我们给出一个简化的 Dot Product 协议，用来证明两个向量元素的点积。

**公共输入**：

1. 向量承诺 $[d(X)]_1$，$[t(X)]_1$，假设两个向量 $\vec{d}$ 与 $\vec{t}$ 的长度均为 $n$
2. 向量点积为 $\sigma$

**证明目标**：

$$
\vec{d}\cdot \vec{t} = \sigma
$$

**第一步**: Prover 计算 $r(X)$, $q(X)$，并发送 $\big([r(X)]_1, [q(X)]_1, [X^{D-n+2}\cdot r(X)]_1\big)$

$$
d(X)\cdot t(X) = \frac{\sigma}{n} + X\cdot r(X) + z_\mathbb{H}(X)\cdot q(X)
$$

**第二步**: Verifier 发送一个随机挑战值 $\zeta\in\mathbb{F}$

**第三步**: Prover 发送 $\Big(v_1=d(\zeta), v_2=t(\zeta), v_3=r(\zeta), v_4=q(\zeta) \Big)$

**第四步**: Verifier 发送一个随机挑战值 $\gamma\in\mathbb{F}$

**第五步**: Prover 计算并发送 $[w(X)]_1$

$$
w(X) = \frac{d(X) - v_1}{X-\zeta} + \gamma\cdot \frac{t(X) - v_2}{X-\zeta} 
+\gamma^2\cdot \frac{r(X) - v_3}{X-\zeta}
+\gamma^3\cdot \frac{q(X) - v_4}{X-\zeta}
$$

**验证**: Verifier 

1. 计算 $C_p = [d(X)]_1 + \gamma[t(X)]_1 + \gamma^2[r(X)]_1 + \gamma^3[q(X)]_1$
2. 计算 $C_v = \big(v_1 + \gamma\cdot v_2 + \gamma^2\cdot v_3 + \gamma^3\cdot v_4\big)\cdot [1]_1$ 
2. 验证聚合的求值证明：

$$
[w(X)]_1\ast[X]_2 \overset{?}{=} \Big(C_p + \zeta[w(X)]_1 - C_v \Big)\ast [1]_2
$$

3. 验证 $r(X)$ 的 degree bound

$$
[X^{D-n+2}\cdot r(X)]_1 \ast [1]_2 \overset{?}{=} [r(X)]_1 \ast [X^{D-n+2}]_2
$$


## 4. 简化的协议框架

到此为止，我们可以得到一个简化的 Baloo 证明框架：

**公共输入**：
1. 表格向量 $\vec{t}$ 的承诺 $[t(X)]_1$， 表长为 $n$
2. 查询向量 $\vec{a}$ 的承诺 $[a(X)]_1$，查询长度为 $m$

**证明目标**：

$$
\exists M\in \mathbb{F}^{m\times n},\quad M\vec{t} = \vec{a}
$$

**子协议一**：CP-expansion 协议

Prover 向 Verifier 提供矩阵行空间抽样 $M(\alpha, X)$ 的承诺 $[d(X)]_1$，并证明其正确性。换句话说，这等价于证明了存在一个由「行单位向量」构成的矩阵 $M$，并且它的行空间抽样的承诺为 $[d(X)]_1$。

证明的思路是，Prover 提供矩阵的编码向量 $\vec{v}$ 与 列空间抽样 $\vec{e}$ 的承诺 $[v(X)]_1$ 与 $[e(X)]_1$，并证明

$$
e(X)\cdot(\beta\cdot v(X) -1) - \frac{z_\mathbb{H}(\beta)}{m} = z_\mathbb{V}(X)\cdot q_1(X)$$

$$
\deg(e(X))< m
$$

$$
d(\beta)=e(\alpha)
$$

**子协议二**：Dot Product 协议

Prover 向 Verifier 证明 $d(X)$ 所编码的向量 $\vec{d}$ 与子表格向量 $\vec{t}$ 的 Dot product 等于 $a(\alpha)$，查询多项式的在 $X=\alpha$ 处的抽样。 这等价于证明了证明目标所要求的 Matrix-vector 乘法的正确性。

$$
 d(X)\cdot t(X) - \left(\frac{a(\alpha)}{n}\right) - X\cdot g(X) = q_2(X)\cdot z_\mathbb{H}(X)
$$

## References

- 