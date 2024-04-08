# Baloo, lookup  (Part 2)

在前文简化的 Baloo 证明框架上，本文介绍一个「完整但不优化」的Baloo 协议，这需要加入「子表格抽取」子协议，从而支持对大表格查询，并需要保持证明开销只与查询数量相关这个关键特性。
加入子表格抽取协议会增加 Baloo 协议的复杂度，这是由于子表格被编码在一个 $\mathbb{H}$ 的一个任意子集 $\mathbb{H}_I$ 上。Caulk 论文给出了一个相当繁琐的方案，而随后的工作 Caulk+ 巧妙地利用了 Cached Quotients，从而大大优化了子表格抽取协议。因此 Baloo 协议也直接使用了 Caulk+ 中的子表格抽取协议。

完整的 Baloo 证明框架如下：

**公共输入**：

1. 主表格向量 $\vec{t}$，主表格承诺 $[t(X)]_1$， 表长为 $N$；
2. 子表格向量 $\vec{t_I}$，子表格承诺 $[t_I(X)]_1$，表长为 $k$；
3. 查询记录向量 $\vec{a}$，查询承诺 $[a(X)]_1$，查询长度为 $m$，考虑到查询记录的重复，$k\leq m$。

**证明目标**：

$$
\exists M\in \mathbb{F}^{m\times k}, \quad M \vec{t}_I = \vec{a}
$$

**子协议一**：Subtable 协议

Prover 向 Verifier 证明 $\vec{t}_I \subset \vec{t}$，提供一个子向量索引的向量 $I$ 的承诺 $z_I(X)$ 并证明其正确性。这等价于证明了子表格的每一项都在主表格中，并且子表格每项的索引信息也对应与 $z_I(X)$ 的根集合。这也隐含了一个关系：子表格中没有重复的主表元素（除非主表格中也有重复项）。

**子协议二**：CP-expansion 协议

Prover 向 Verifier 提供矩阵的行空间抽样 $M(\alpha, X)$ 的承诺 $[d(X)]_1$，并证明其正确性。换句话说，这等价于证明了存在一个由「行单位向量」构成的矩阵 $M$，并且它的行空间抽样的向量 $\vec{d} $的多项式承诺为 $[d(X)]_1$。

**子协议三**：Dot Product 协议

Prover 向 Verifier 证明 $d(X)$ 所编码的向量 $\vec{d}$ 与子表格向量 $\vec{t}_I$ 的 Dot product 等于 $a(\alpha)$，查询多项式的在 $X=\alpha$ 处的抽样。 这等价于证明了证明目标所要求的 Matrix-vector 乘法的正确性。

下面我们先介绍新增的 Subtable 子协议，然后给出一个增强的 CP-expansion 协议，最后给出一个一般化的 Univariate Sumcheck 协议，基于它构造 Dot Product 协议。

## 5. Subtable 子协议

接下来，我们补齐 Baloo 协议要求的 Subtable 子协议部分。这部分的证明直接沿用了 Caulk/Caulk+ 中的做法。最初的想法来源于 [TAB+20] 中「子向量承诺协议」，即证明一个子向量是另一个向量的一部分。其中的关键思想为下面的扩展多项式余数定理，即存在一个商多项式 $q(X)$，使得：

$$
t(X) - t_I(X) = z_I(X) \cdot q(X)
$$

这里我们沿用 $\mathbb{H}$ 作为 $t(X)$ 的 Domain，用记号 $\mathbb{H}_I$ 表示 Subtable 向量 $\vec{t}_I$ 所在的 Domain, 用 $z_I(X)$ 表示 $\mathbb{H}_I$ 上的 Vanishing Polynomial，用 $\{\tau_i(X)\}_{i=0}^{k-1}$ 表示 $\mathbb{H}_I$ 上的 Lagrange Polynomials。其中 $t_I(X)$ 为子向量 $\vec{t}_I$ 在 $\mathbb{H}_I$ 上的插值多项式，

$$
t_I(X) = \sum_{i=0}^{k-1}t(\omega_{h_i})\cdot \tau_i(X)
$$

而 $I$ 表示子向量 $\vec{t}_I$ 中的每一个元素在 $\mathbb{H}$ 中的索引。

$$
I=(h_0,h_1,\ldots,h_{k-1})
$$

举个例子说明，假如 $\mathbb{H}=(1, \omega, \omega^2, \ldots, \omega^7)$，表格向量为 $\vec{t}=(t_0,t_1,\ldots,t_7)$，而查询向量 $\vec{a}=(t_2, t_2, t_3, t_6)$，那么子表格向量为 $\vec{t}_I=(t_2, t_3, t_6)$ ，而索引向量为 $I=(2, 3, 6)$， 索引向量也可以用 Domain 的子集来表达：$\mathbb{H}_{I}=(\omega^2, \omega^3, \omega^6)$。

这里需要注意，$\mathbb{H}$ 是一个规则的乘法子群，但 $\mathbb{H}_I$ 在一般情况下构成群。它是每次协议运行时，Prover 根据子表格的索引而动态计算的。因此在 $\mathbb{H}_I$ 上进行 Lagrange 插值计算需要 $O(k\log^2{k})$ 的计算量。

回忆下 Caulk+ 中的 Subtable 协议的核心思想是：如果 $\vec{t}_I \subset \vec{t}$，那么就必然存在一个商多项式 $q_I(X)$，使得下面的等式成立：

$$
t(X) - t_I(X) = z_I(X) \cdot q_I(X)
$$

但这里由于子表格 $t_I$ 为 Witness，Prover 需要同时发送 $[q_I(X)]_1$ 与 $[z_I(X)]_2$ 给 Verifier。于是上面的等式能保证 $t_I(X)$ 正确性的前提是 $z_I(X)$ 必须是正确的。而 Caulk+ 论文进一步保证 $z_I(X)$ 的方法非常聪明。如果子表格关系成立，那么它们的索引集合也满足子集关系，于是我们可以证明 $z_I(X)$ 整除 $z_\mathbb{H}(X)$。那么 Prover 可以提供一个商多项式 $z_{\mathbb{H}\backslash I}(X)$ （的承诺 $[z_{\mathbb{H}\backslash I}(X)]_1$），使得：

$$
z_\mathbb{H}(X) = z_I(X)\cdot z_{\mathbb{H}\backslash I}(X)
$$

由于 $\mathbb{H}$ 的 FFT 平滑性质，Verifier 可以在 $O(\log{N})$ 时间内快速计算得到 $[z_\mathbb{H}(X)]_1$，并检查 Prover 提供的两个多项式（承诺）是否满足上面这个等式关系。

### 5.1 Cached Quotients 预计算

如果 Prover 直接计算 $q_I(X)$，那么 Prover 需要 $O(N\log{N})$ 的计算量来完成 degree 为 $N$ 的多项式除法。这显然不符合我们的要求，我们希望 Prover 的性能只与 $m$ （或 $k$）相关。这需要用到 [TAB20] 和 Caulk 引入的 Cached Quotients 技术（篇幅所限，下面只给出结论，相关原理与推导请参考[....]）。

除了直接通过多项式除法，商多项式 $q_I(X)$ 还可以通过一系列（预计算的）商多项式的线性组合来计算得到，这是因为 $q_I(X)$ 具有以下内部结构：

$$
q_I(X) = \frac{1}{z_I'(\omega_{h_0})}\cdot q_0(X) + \frac{1}{z_I'(\omega_{h_1})}\cdot q_1(X) + \cdots + \frac{1}{z_I'(\omega_{h_{k-1}})}\cdot q_{k-1}(X)
$$

其中每一个 $q_i(X)$ 可以是预计算的商多项式（如果主表格 $\vec{t}$ 固定），而 $z_I'(\omega_{h_i})$ 是 $z_I(X)$ 在 $X=\omega_{h_i}$ 处的导数。

$$
q_i(X) = \frac{t(X)-t(\omega_i)}{X-\omega_i}
$$

$$
z_I'(\omega_{h_i}) = \prod_{j\in I, j\neq i}(\omega_{h_i}-\omega_{h_j})
$$

这样一来，Verifier 或者任何第三方可以提前预计算所有商多项式的承诺，即 $\{[q_i(X)]_1\}_{i\in[N]}$。而 Prover 在证明过程中自行计算 $\{z_I'(\omega_{h})\}_{h\in I} $ 并计算得到 $[q_I(X)]_1$ 发送给 Verifier。这样一来，Prover 的计算量就只与 $k$ 相关，而不再与 $N$ 相关，计算量仅为 $O(k\log^2k)$。

此外，$z_{\mathbb{H}\backslash I}(X)$ 如果直接计算也需要 $O(N\log{N})$ 的工作量。Caulk+ 论文发现 $z_{\mathbb{H}\backslash I}(X)$ 也可以通过 Cached Quotients 来预计算，极大优化了 Caulk 原论文的繁琐做法。

我们现在推导一个连乘式的倒数分解等式。对于一个常数多项式 $f(X)=1$，可得
$$
1 = f(X) = \sum_{h\in I}f_i\cdot \frac{1}{z'_I(\omega_{h})}\frac{z_I(X)}{X-\omega_{h}} = \sum_{h\in I}\frac{1}{z'_I(\omega_{h})}\frac{z_I(X)}{X-\omega_{h}}
$$

把等式两边除以 $z_I(X)$，可得：

$$
\frac{1}{z_I(X)}= \sum_{h\in I}\frac{1}{z'_I(\omega_{h})}\cdot \frac{1}{X-\omega_{h}}
$$

然后观察 $z_{\mathbb{H}\backslash I}(X)$ 的定义，我们可以得到下面的等式：

$$
\begin{split}
z_{\mathbb{H}\backslash I}(X) &= \prod_{i\not\in I}(X-\omega_i) \\
&= z_\mathbb{H}(X) \cdot \frac{1}{\prod_{h\in I}(X-\omega_h)} \\
&= z_\mathbb{H}(X) \cdot \frac{1}{z_I(X)} \\
&= z_\mathbb{H}(X) \cdot\Big( \sum_{h\in I} \frac{1}{z'_I(\omega_h)}\cdot \frac{1}{X-\omega_h}\Big)\\
&= \sum_{h\in I} \frac{1}{z'_I(\omega_h)} \cdot \frac{z_\mathbb{H}(X)}{X-\omega_h}
\end{split}
$$

因此，$z_{\mathbb{H}\backslash I}(X)$ 可以看成是对于 $\left\{w_i(X)=\frac{z_\mathbb{H}(X)}{X-\omega_i}\right\}_{i\in[N]}$ 的线性组合，而 $\left\{c_i=\frac{1}{z'_I(\omega_{h_i})}\right\}_{h_i\in I}$ 为所谓的 Bary Centric Weights，证明时需要的计算量也只与 $k$ 相关。我们可以让协议提前预计算 $\left\{\frac{z_\mathbb{H}(X)}{X-\omega_i}\right\}_{i\in[N]}$，这样 Prover 只需要计算（时间复杂度为 $O(k\log^2k)$） Bary Centric Weights 和 $k$ 个乘法的线性组合运算。

### 5.2 协议细节

**公共输入**：

1. 大表格的承诺 $[t(X)]_1$， 
2. 子表格的承诺 $[t_I(X)]_1$，
3. 子表格元素索引向量在 $\mathbb{G}_2$ 上的承诺 $[z_I(X)]_2$

**证明目标**：

$$
t(X) - t_I(X) = z_I(X) \cdot q_I(X)
$$

$$
z_\mathbb{H}(X) = z_I(X)\cdot z_{\mathbb{H}\backslash I}(X)
$$

**预计算**

1. $\{[q_i(X)]_1 \}_{i\in[N]}$，所承诺的每个多项式满足下面的等式：

$$
q_i(X)\cdot (X-\omega_i) = t(X) - t(\omega_i)
$$

2. $\{[w_i(X)]_1\}_{i\in[N]}$，所承诺的每个多项式满足下面的等式：

$$
w_i(X)\cdot (X-\omega_i) = z_\mathbb{H}(X)
$$

**第一步**：

1. Prover 计算 Bary Centric Weights $\{c_i\}_{i\in[k]}$

$$
c_i = \frac{1}{z'_I(\omega_{h_i})}, \qquad \forall i\in [0,k)
$$

2. Prover 计算并发送 $Q=[q(X)]_1$, 与 $W=[w(X)]_1$ 
$$
[q(X)]_1 = \sum_{i\in [k]} c_i\cdot [q_i(X)]_1
$$

$$
[w(X)]_1 = \sum_{i\in[k]} c_i \cdot [w_i(X)]_1
$$

**验证**：Verifier 验证下面的两个 Pairing 等式：

$$
\begin{split}
\Big([t(X)]_1 - [t_I(X)]_1\Big) \ast [1]_2 &\overset{?}{=} Q \ast [z_I(X)]_2 \\
\Big([z_\mathbb{H}(X)]_1\Big) \ast [1]_2 &\overset{?}{=} W \ast [z_I(X)]_2 \\
\end{split}
$$

## 6. Generalized Univariate Sumcheck 

如果我们考虑 $\vec{t}_I$ 是某个主表格 $\vec{t}$ 的子表格，并且 $t_I(X)$ 不再是一个规则的乘法子群上的 Lagrange 插值多项式，那么前文介绍的简化版 CP-expansion 协议则无法适用。我们在增强 CP-expansion 协议之前，先介绍一个更一般化的 Univariate Sumcheck 定理。新的 Univariate Sumcheck 定理不再要求 $t_I(X)$ 恰好编码在一个乘法子群上。

首先回忆一下简化版的 Sumcheck 定理：

$$
a(X) = \frac{\sigma}{n} + X\cdot g(X) + z_\mathbb{H}(X)\cdot q(X), \qquad \deg(r(X))<n-1
$$

这里 $\sigma=\sum_{i\in[n]}a(\omega_i)$ 为 $a(X)$ 在一个乘法子群 $\mathbb{H}$ 上的取值之和。

### 6.1 Generalization

我们临时假设 $H=(h_0, h_1, \ldots, h_{k-1})\subset\mathbb{F}$ 是有限域上的任意子集，多项式 $z_H$ 为 $H$ 上的 Vanishing 多项式，其对应的 Lagrange 多项式为 $\{\tau_i(X)\}_{i\in [k]}$。

现在我们要证明一个长度为 $k$ 的向量 $\vec{a}$ 中的所有元素之和等于 $\sigma$，即 $\sum_i a_i = \sigma$，那么我们先尝试按往常的方式构造相应的多项式 $a(X)=\sum_{i\in[k]} a_i\cdot \tau_i(X)$，并代入 $X=0$ 计算多项式的常数项，那么我们可以得到下面的等式：

$$
a(0) =  \sum_{i=0}^{k-1}a_i\cdot \tau_i(0)
$$

虽然等式右边接近 $\sigma = \sum_{i=0}^{n-1} a(\omega^i)$, 但求和的每一项都带了一个讨厌的因子 $\tau_i(0)$。如何消除这个因子呢？

我们其实可以更改 $a(X)$ 的定义，使其不再是通常意义下的关于 $\vec{a}$ 的 Lagrange 插值多项式，而是插值的过程中提前消去一个因子。

$$
a(X) = \sum_{i=0}^{k-1}a_i\cdot \frac{\tau_i(X)}{\tau_i(0)}
$$

这样一来，$a(0) = \sum_i a_i=\sigma$ 就可以成立。于是我们还能得到下面的关系：

$$
a(X) - a(0) = X\cdot r(X), \qquad \deg(r(X))<k-1
$$

容易理解，因为 $a(0)$ 为 $a(X)$ 的常数项，当减去常数项后，$a(X)$ 可以被 $X$ 整除，因此 $a(X) - a(0) = X\cdot r(X)$。

可以进一步扩展，如果我们要证明两个长度为 $k$ 的向量的点积，比如 $\vec{a}\cdot \vec{b} = \sigma$，那么我们可以构造两个基于 $H$ 的插值多项式 $a(X)$ 和 $b(X)$，

$$
a(X) = \sum_{i=0}^{k-1}a_i\cdot \frac{\tau_i(X)}{\tau_i(0)} \qquad
b(X) = \sum_{i=0}^{k-1}b_i\cdot \tau_i(X)
$$

值得注意的是，这里我们只对 $a(X)$ 采用了 Nomalized Lagrange 插值，而 $b(X)$ 仍然采用了普通的 Lagrange 插值。这是因为我们只需要证明 $a(X)\cdot b(X)$ 的常数项为 $\sigma$ 即可，所以只需要单边引入一个 Nomialized Lagrange 插值就能消去 $\tau_i(0)$ 因子：

$$
\begin{split}
a(X)\cdot b(X) &= \Big(\sum_{i=0}^{k-1}a_i\cdot \frac{\tau_i(X)}{\tau_i(0)}\Big)\cdot \Big(\sum_{i=0}^{k-1}b_i\cdot \tau_i(X)\Big) \\
& \equiv \Big(\sum_{i=0}^{k-1}a_i\cdot b_i \cdot \frac{\tau_i(X)}{\tau_i(0)}\Big) \pmod{z_H(X)} \\
& \equiv \sigma + X\cdot r(X) \pmod{z_H(X)}, \qquad \deg(r(X))<k-1
\end{split}
$$


这个推导过程利用了 Lagrange 多项式之间「相互正交」的性质，并且他们在 $H$ 上的取值为 Unit Vector，即 $\tau_i(X)|_{X=\omega_i}=1$，而 $\tau_i(X)|_{X=\omega_j, i\not =j}=0$ ，这个单位正交性可以表示为下面两个等式关系：

$$
\begin{split}
\tau_i(X)\cdot \tau_i(X) &\equiv  \tau_i(X) \pmod{z_H(X)}\\
\tau_i(X)\cdot \tau_j(X) &\equiv  0 \pmod{z_H(X)}, \quad i\neq j
\end{split}
$$

可以检验，这两个等式的两边在任意 $h_i\in H$ 上都成立，因此上面的 modulo 关系也成立。这个性质保证了我们可以在 $z_H(X)$ 的模意义下，对多项式进行「模乘」运算。

这种消除 $\tau_i(0)$ 因子的插值方式，在 Baloo 论文中被称为 Nomalized Lagrange Interpolation。我们干脆按照论文的方式，把 $\{\tau_i(X)/\tau_i(0)\}$ 命名为 Normalized Lagrange Polynomials，符号记为 $\{\hat\tau_i(X)\}$。

## 7. 增强的 CP-expansion 协议

本小节描述一个增强版 CP-expansion 协议，证明一个子向量 $\vec{t}_I$ 经过矩阵 $M$ 所代表的线性变换之后，得到一个向量 $\vec{a}$，其中矩阵 $M$ 为一个行向量为单位向量的矩阵。

$$
M \vec{t}_I = \vec{a}
$$

其中 $\vec{t}_I$ 是 $\vec{t}$ 的一个子向量，长度为 $k$，后者编码为 $t(X)$。 这里我们用符号 $\mathbb{H}$ 表示 $t(X)$ 的 Domain，而 $I$ 表示子向量 $\vec{t}_I$ 中的每一个元素在 $\mathbb{H}$ 中的索引。

重复强调下，这里 $t_I(X)$ 表示子向量 $\vec{t}_I$ 的插值多项式。 我们把用记号 $\mathbb{H}_I$ 表示  $\vec{t}_I$ 所在的 Domain, 用 $z_I(X)$ 表示 $\mathbb{H}_I$ 上的 Vanishing 多项式，用 $\{\tau_i(X)\}_{i=0}^{k-1}$ 表示 $\mathbb{H}_I$ 上的 Lagrange 多项式。对应的 Normalized Lagrange 多项式记为 $\{\hat\tau_i(X)\}_{i=0}^{k-1}$。请务必注意，$\{\hat\tau_i(X)\}_{i=0}^{k-1}$ 多项式的定义域为 $\mathbb{H}_I$，而不是 $\mathbb{H}$。

把上述向量编码成多项式之后，向量的乘法运算关系可以转化为多项式的乘法运算关系：

$$
\sum_{Y\in \mathbb{H}_I}M(X, Y) \cdot t_I(Y) = a(X)
$$

这里 $M(X, Y)$ 定义如下：

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
\hat\tau_0(Y) \\
\hat\tau_1(Y) \\
\hat\tau_2(Y) \\
\vdots \\
\hat\tau_{k-1}(Y) \\
\end{bmatrix}
$$

通过 Verifier 提供的一个随机挑战数 $\alpha$，我们可以把多项式乘法的运算关系归约到一个新的点积求和关系：

$$
\sum_{X\in \mathbb{H}_I} M(\alpha, X) \cdot t_I(X) = a(\alpha)
$$

其中

$$
M(\alpha, X) = \begin{bmatrix}
\mu_0(\alpha), \mu_1(\alpha), \cdots, \mu_{m-1}(\alpha) \\
\end{bmatrix}
\begin{bmatrix}
M_{0,0} & M_{0,1} & \cdots & M_{0,n-1} \\
M_{1,0} & M_{1,1} & \cdots & M_{1,n-1} \\
M_{2,0} & M_{2,1} & \cdots & M_{2,n-1} \\
\vdots & \vdots & \ddots & \vdots \\
M_{m-1,0} & M_{m-1,1} & \cdots & M_{m-1,n-1} \\
\end{bmatrix}
\begin{bmatrix}
\hat\tau_0(X) \\
\hat\tau_1(X) \\
\hat\tau_2(X) \\
\vdots \\
\hat\tau_{k-1}(X) \\
\end{bmatrix}
$$

我们进一步分析 $M(\alpha, X)$，展开它的定义，我们可得：

$$
M(\alpha, X) = \sum_{i=0}^{m-1}\sum_{j=0}^{k-1}\mu_i(\alpha)\cdot M_{i,j}\cdot \hat\tau_{j}(X) 
$$

考虑到 $M$ 每一行有且仅有一个 $1$，其余元素为 $0$，因此 $\sum_{j=0}^{k-1} M_{i,j}\cdot \hat\tau_{j}(X) = \hat\tau_{\mathsf{col}(i)}(X)$，其中 $\mathsf{col}: [k] \to I$，表示第 $i$ 行中不为 $0$ 的元素的列索引函数。


因此 $M(\alpha, X)$ 可以进一步化简，并记为 $d(X)$：

$$
d(X) = M(\alpha, X) = \sum_{i=0}^{m-1}\sum_{j=0}^{k-1}\mu_i(\alpha)\cdot M_{i,j}\cdot \hat\tau_{j}(X) =\sum_{i=0}^{m-1}\mu_i(\alpha)\cdot \hat\tau_{\mathsf{col}(i)}(X)
$$


接下来的问题是，如何证明 $d(X)$ 即 $M(\alpha, X)$ 的正确性？由于 $M(\alpha, X)$ 是一个二元多项式的部分运算，因此我们可以通过 $M(X,Y)$ 的另一个部分运算 $M(X, \beta)$ 进行关联，证明两者都是从同一个 $M(X, Y)$ 部分运算得到，只要我们能证明 $M(\alpha, X)$ 与 $M(X, \beta)$ 中的任意一个正确编码了 $M$，那么我们就能证明 $M(\alpha, X)$ 的正确性（为何我们需要通过这种方式来证明 $d(X)$ 的正确性，请参考 Part 1 中 2.1 节）。

证明 $M(\alpha, X)$ 与 $M(X, \beta)$ 都是从同一个 $M(X, Y)$ 部分运算得到的结果并不难，我们只需要通过 Oracle 证明下面关系即可：

$$
M(\alpha, X=\beta) = M(X=\alpha, \beta)
$$

接下来，我们仍然需要证明 $M(X, \beta)$ 正确编码了 $M$，我们暂且用一个新的记号 $e(X)=M(X, \beta)$。

$$
e(X) = \sum_{i=0}^{m-1}\hat\tau_{\mathsf{col}(i)}(\beta)\cdot \mu_i(X)
$$

### 7.1 $e(X)$ 的正确性

实际上，$e(X)$ 编码了矩阵 $M$ 的列空间抽样向量 $\vec{e}=(e_0, e_1, \ldots, e_{m-1})$，其长度为 $m$。我们可以通过下面的等式来定义 $e_i$：
 
$$
e_i = \hat\tau_{\mathsf{col}(i)}(\beta) = \frac{\tau_{\mathsf{col}(i)}(\beta)}{\tau_{\mathsf{col}(i)}(0)} 
= \frac{\prod_{j\neq \mathsf{col}(i)}(\beta - \omega_j)}{\prod_{j\neq \mathsf{col}(i)}(-\omega_j)}
= \frac{z_I(\beta)}{z_I(0)}\cdot \frac{-\omega_{\mathsf{col}(i)}}{\beta - \omega_{\mathsf{col}(i)}}
$$

我们把 $\vec{v}$ 定义为 $v_i = \omega_{\mathsf{col}(i)}$，那么 $e_i$ 可以表示为：

$$
e_i = \frac{z_I(\beta)}{z_I(0)}\cdot \frac{-\omega_{\mathsf{col}(i)}}{\beta - \omega_{\mathsf{col}(i)}} = \frac{z_I(\beta)}{z_I(0)}\cdot \frac{-v_i}{\beta - v_i}
$$

化简下等式，可得：

$$
e_i \cdot (\beta - v_i) + \frac{z_I(\beta)}{z_I(0)}\cdot {v_i} = 0
$$

辅助向量 $\vec{v}$ 在 $\mathbb{V}$ 上的插值多项式 $v(X)$定义如下：

$$
v(X)=\sum_{i=0}^{m-1}v_i\cdot \mu_i(X) = \sum_{i=0}^{m-1}\omega_{\mathsf{col}(i)}\cdot \mu_i(X)
$$

把上面的关系转换成多项式之间的关系，可得：

$$
e(X)\cdot (\beta - v(X)) + \frac{z_I(\beta)}{z_I(0)}\cdot v(X) = z_\mathbb{V}(X) \cdot q(X), \qquad \deg(e(X))<m
$$

至此，只要 Prover 能证明上面等式成立，并且证明 $e(X)$ 的 degree 不超过 $m$，那么就能证明 $e(X)$ 具有以下形式：

$$
e(X)
= \sum_{i=0}^{m-1} \frac{z_I(\beta)}{z_I(0)}\cdot \frac{-v_i}{\beta - v_i}\cdot \mu_i(X)
$$

假如 $\vec{v}$ 是正确的，那么我们就可以得出结论，$e(X)$ 是正确的。如何确保 $\vec{v}$ 的正确性呢？

### 7.2 $v(X)$ 的正确性

向量 $\vec{v}$ 的正确性依靠 $d(\beta)=e(\alpha)$ 这个约束来保证。由于

$$
e(X) = \sum_{i=0}^{m-1}\hat\tau_{\mathsf{col}(i)}(\beta)\cdot \mu_i(X)
= \sum_{i=0}^{m-1}\frac{-z_I(\beta)\cdot v_i}{z_I(0)(\beta - v_i)}\cdot \mu_i(X)
$$

如果 $d(\beta)=e(\alpha)$ 成立，那么根据 $d(X)$ 与 $e(X)$ 的对称性，我们把上面等式右边中，$X$ 替换成 $\alpha$，而 $\beta$ 替换成未知数 $X$，可得：

$$
d(X) = \sum_{i=0}^{m-1}\frac{-z_I(X)\cdot v_i}{z_I(0)(X - v_i)}\cdot \mu_i(\alpha)
$$

假如某个 $v_j\not\in\mathbb{H}_I$，那么 $d(X)$ 定义中的第 $j$ 项中，$(X-v_j)$ 将不能整除 $z_I(X)$。而且，由于 $\mu_i(\alpha)$ 这个随机因子（这里要求随机数 $\alpha$ 必须在 $v(X)$ 和 $d(X)$ 承诺产生之后由 Verifier 提供），导致这一项也无法和其他项相抵消，从而导致 $d(X)$ 不再是一个 Polynomial，而是一个 Rational Function，即 $d(X)\in\mathbb{F}(X)$，但 $d(X)\not\in\mathbb{F}[X]$。如果这样， Prover 就不可能计算得到 $d(X)$ 的多项式承诺。

故我们只要 Prover 提供一个多项式承诺 $[d(X)]_1$，并且证明 $d(\beta)=e(\alpha)$，那么也就证明了 $v_i\in\mathbb{H}_I$，进而保证了 $v(X)$ 的正确性。

### 7.3 $d(X)$ 的正确性

最后剩下的 $d(X)$ 的正确性可以由 $e(X)$ 和 $v(X)$ 的正确性，加上 $d(\beta)=e(\alpha)$ 这个约束来证明。


### 7.4 CP-expansion 协议细节

**公共输入**：

1. 子表格的承诺 $[t_I(X)]_1$，
2. 行空间抽样向量的承诺 $[d(X)]_1$，
3. Verifier 提供的挑战数 $\alpha$
4. 子表格索引向量在 $\mathbb{G}_1$ 上的承诺

**证明目标**: 存在一个由 Unit Vector 为行构成的矩阵 $M$，使得 $\vec{d}$ 是这个矩阵的行空间抽样向量。

$$
d_i = \mu_i(\alpha)\cdot \vec{M}_i
$$

$$
d(X) = \sum_{i=0}^{m-1}\mu_i(\alpha)\cdot \vec{M}_i(X)
$$


**第一步**：Prover 计算矩阵 $M$ 的编码 $v(X)$ 并发送 $V=[v(X)]_1$

$$
v(X) = \sum_{i=0}^{m-1}\omega_{\mathsf{col}(i)}\cdot \mu_i(X)
$$

**第二步**：Verifier 发送随机挑战数 $\beta$

**第三步**：

1. Prover 计算 $e(X)$ 与 $q_1(X)$，并发送 $E=[e(X)]_1$, $Q_1=[q_1(X)]_1$,

$$
e(X)\cdot (\beta - v(X)) + \frac{z_I(\beta)}{z_I(0)}\cdot v(X) = z_\mathbb{V}(X) \cdot q_1(X)
$$

2. Prover 计算 $z_I(0)$ 的商多项式 $q_2(X)$，并发送 $z_I(0)$ 与 $Q_2=[q_2(X)]_1$,

$$
q_2(X) = \frac{z_I(X)-z_I(0)}{X}
$$

3. Prover 计算并发送 $d(\beta)$ 与 $z_I(\beta)$

4. Prover 计算并发送 $e(\alpha)$，以及商多项式 $q_3(X)$，并发送 $Q_3=[q_3(X)]_1$

$$
q_3(X) = \frac{e(X)-e(\alpha)}{X-\alpha}
$$

5. Prover 发送 $e(X)$ 的 degree-bound 证明 $[X^{D-m}\cdot e(X)]_1$

**第四步**：Verifier 发送挑战点 $X=\zeta$

**第五步**：Prover 发送 $e(\zeta)$, $v(\zeta)$，$q_1(\zeta)$

**第六步**：Verifier 发送随机挑战数 $\gamma$，用于聚合多项式求值证明

**第七步**：

1. Prover 计算并发送 $W_1=[w_1(X)]_1$，

$$
w_1(X) = \frac{z_I(X) - z_I(\beta)}{X-\beta} + \gamma\cdot \frac{d(X) - d(\beta)}{X-\beta}
$$

2. Prover 计算合并的商多项式 $w_2(X)$ ，并发送其承诺 $W_2=[w_2(X)]_1$

$$
w_2(X) = \frac{e(X)-e(\zeta)}{X-\zeta} + \gamma\cdot\frac{v(X)-v(\zeta)}{X-\zeta} + \gamma^2\cdot \frac{q_1(X)-q_1(\zeta)}{X-\zeta} 
$$

**验证**：Verifier 验证下面的步骤：

1. 计算 $z_{\mathbb{V}}(\zeta)$ 并 验证 $e(X)$ 的正确性

$$
e(\zeta)\cdot(\beta - v(\zeta))+z_I(\beta)z_I(0)^{-1}\cdot v(\zeta) \overset{?}{=} z_{\mathbb{V}}(\zeta)\cdot q_1(\zeta)
$$

2. 验证 $Q_2$ 的正确性：

$$
\Big([z_I(X)]_1 - z_I(0)\cdot[1]_1 \Big) \ast [1]_2 \overset{?}{=} Q_2 \ast [X]_2
$$

3. 验证 $Q_3$ 的正确性：

$$
\Big(E - e(\alpha)\cdot[1]_1 \Big) \ast [1]_2 \overset{?}{=} Q_3 \ast [X]_2
$$

4. 计算合并的多项式承诺 $P_1$ 与 $y_1$ 并验证 $W_1$ 的正确性

$$
\begin{split}
P_1 & = [z_I(X)]_1 + \gamma\cdot [d(X)]_1 \\
y_1 & = z_I(\beta) + \gamma\cdot d(\beta) \\
\end{split}
$$

$$
\Big(P_1 - y_1\cdot[1]_1 \Big) \ast [1]_2 \overset{?}{=} W_1 \ast \Big([X]_2 - \beta
\cdot[1]_2 \Big)
$$

5. 计算合并的多项式承诺 $P_2$ 与 $y_2$ 并验证 $W_2$ 的正确性


$$
\begin{split}
P_2 & = E + \gamma\cdot V + \gamma^2\cdot G + \gamma^3\cdot Q + \gamma^4\cdot [z_I(X)]_1 \\
y_2 & = e(\zeta) + \gamma\cdot v(\zeta) + \gamma^2\cdot q(\zeta) \\
\end{split}
$$

$$
\Big(P_2 - y_2\cdot[1]_1 \Big) \ast [1]_2 \overset{?}{=} W_2 \ast \Big([X]_2 - \zeta
\cdot[1]_2 \Big)
$$

6. 验证 $\deg(e(X)) <m $

$$
\Big([X^{D-m}\cdot e(X)]_1 \Big) \ast [1]_2 \overset{?}{=} E \ast [X^{D-m}]_2
$$


## 8. Dot Product 协议

接下来是要证明剩下的求和关系：

$$
\sum_{X\in \mathbb{H}_I} M(\alpha, X) \cdot t_I(X) = a(\alpha)
$$

利用前文提到的 Generalized Univariate Sumcheck 定理，对于两个长度为 $k$ 的向量，如果 $\vec{A}\cdot\vec{B}=s$，那么两个向量的多项式满足下面的关系（具体推导过程请参考前文 4.1 节）：

$$
A(X)\cdot B(X) \equiv s + X\cdot R(X) \pmod{z_I(X)}, \qquad \deg(R(X))<k-1
$$

其中

$$
A(X) = \sum_{i=0}^{k-1}A_i\cdot \frac{\tau_i(X)}{\tau_i(0)} \qquad
B(X) = \sum_{i=0}^{k-1}B_i\cdot \tau_i(X)
$$

我们把 $A(X) = M(\alpha, X)$ 与 $B(X) = t_I(X)$ 代入上式，可以得到下面的等式：

$$
M(\alpha, X)\cdot t_I(X) \equiv \sigma + X\cdot r(X) + z_I(X) \cdot q_1(X), \qquad \deg(r(X))<k-1
$$

和前文不同的是，$t_I(X)$ 的 Domain 不再是一个 FFT 平滑的乘法子群，而是一个任意的子集，因此 $M(X, Y)$ 矩阵编码时采用的是 Normalized Lagrange Polynomials, $\hat\tau_i(X)$。

**公共输入**：

1. 子表格的承诺 $[t_I(X)]_1$，
2. 行空间抽样向量的承诺 $[d(X)]_1$，
3. 查询记录向量的插值多项式 $a(X)$ 在 $X=\alpha$ 处的取值 $a(\alpha)$，
4. 子表格索引向量在 $\mathbb{G}_1$ 上的承诺 $[z_I(X)]_1$。

**证明目标**

$$
\vec{d}\cdot \vec{t}_I = a(\alpha) 
$$

**第一步**：Prover 计算 $q(X)$ 与  $g(X)$，满足下面关系：

$$
d(X) \cdot t_I(X) = a(\alpha) + X\cdot g(X) + q(X) \cdot z_I(X)
$$

发送 $Q=[q(X)]_1$, $G=[g(X)]_1$

**第二步**：Verifier 发送随机挑战数 $\zeta$

**第三步**：Prover 发送 $d(\zeta)$, $t_I(\zeta)$，$g(\zeta)$，$q(\zeta)$，$z_I(\zeta)$

**第四步**：Verifier 发送随机挑战数 $\gamma$

**第五步**：Prover 计算合并的商多项式 $w(X)$ ，并发送其承诺 $W=[w(X)]_1$

$$
w(X) = \frac{d(X)-d(\zeta)}{X-\zeta} + \gamma\cdot\frac{t_I(X)-t_I(\zeta)}{X-\zeta} + \gamma^2\cdot \frac{g(X)-g(\zeta)}{X-\zeta} + \gamma^3\cdot \frac{q(X)-q(\zeta)}{X-\zeta} + \gamma^4\cdot \frac{z_I(X)-z_I(\zeta)}{X-\zeta}
$$

**验证**：Verifier 验证下面的步骤：

1. 验证 Generalized Sumcheck 等式

$$
d(\zeta)\cdot t_I(\zeta) \overset{?}{=} a(\alpha) + \zeta\cdot g(\zeta) + q(\zeta) \cdot z_I(\zeta)
$$

2. 计算合并的多项式承诺 $P$ 与 $Z$

$$
\begin{split}
P & = [d(X)]_1 + \gamma\cdot [t_I(X)]_1 + \gamma^2\cdot G + \gamma^3\cdot Q + \gamma^4\cdot [z_I(X)]_1 \\
z & = d(\zeta) + \gamma\cdot t_I(\zeta) + \gamma^2\cdot g(\zeta) + \gamma^3\cdot q(\zeta) + \gamma^4\cdot z_I(\zeta) \\
\end{split}
$$

3. 验证 $W$ 的正确性

$$
\Big(P - z\cdot[1]_1 \Big) \ast [1]_2 \overset{?}{=} W \ast \Big([X]_2 - \zeta
\cdot[1]_2 \Big)
$$


## References
- 