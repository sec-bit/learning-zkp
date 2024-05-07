# 理解 Lasso (五)：表格的 MLE 结构

本文介绍 Generalized Lasso，也是 [Lasso] 论文的关键部分之一。与 Lasso 相比，Generalized Lasso 不再对大表格进行拆分，而是把表格作为整体进行证明。为了处理超大尺寸表格，Generalized Lasso 需要要求表格中的每一项是可以通过其 Index 的二进制表示进行计算得到。对于尺寸为 $N$ 的超大表格而言，其 Index 的二进制位数量为 $\log{N}$，因此表格的表项的计算复杂度一定为 $O(\log{N})$。

这样做的一个优势是，Prover 可以不必要对表格进行承诺计算，当 Verifier 挑战表格编码的多项式时，Verifier 可以自行计算挑战点的多项式求值，因为这个运算复杂度仅为 $O(\log{N})$。这样 Prover 可以节省大量的计算时间。


## 1. 什么是 MLE-Structured

按照 [Lasso] 论文的定义，MLE-structured 是指任何 MLE 多项式 $\tilde{t}(\vec{X})$ 可以在 $O(\log{N})$ 时间的计算复杂度内完成求值运算。这里 $N$ 为 $2^s$，$s$ 为多项式未知数的个数。

哪些表格具有这种 MLE-structured 的性质呢？下面给出一些常用的例子：

- Range check 表格，$t_i\in[0, N)$。
- 连续偶数或者奇数构成的表格，$t_i = 2k, t_i\in[0, N)$
- Spread 表格。一种在数字二进制表示的相邻两位插入 0 的表格，例如，对于 $i=0110$，$t_i=00101000$。这种表格用来加速实现位运算。

## 2. Generalized Lasso

Generalized Lasso 可以构造针对 MLE-Structured 表格的 Indexed Lookup Argument。其核心是证明下面的等式：

$$
\tilde{f}(\vec{X}) = \sum_{\vec{y}\in\{0,1\}^{\log{N}}} \tilde{M}(\vec{X}, \vec{y})\cdot \tilde{t}(\vec{y})
$$

这里 $\vec{f}$ 为查询向量，长度为 $m$；$\vec{t}$ 为表格向量，长度为 $N$；$M\in\mathbb{F}^{m\times N}$ 为表格选择矩阵，其中每一行是一个 Unit Vector。而 MLE 多项式 $\tilde{f}(X_0,X_1,\ldots,X_{\log{m}-1})$ 编码了 $\vec{f}$，$\tilde{t}(Y_0,Y_1,\ldots, Y_{\log{N}-1})$ 编码了 $\vec{t}$，而 $\tilde{M}(X_0,X_1,\ldots,X_{\log{m}-1}, Y_0,Y_1,\ldots, Y_{\log{N}-1})$ 编码了 $M$ 矩阵。

Prover 和 Verifier 需要证明每一个 $f_i$ 等于某个表项 $t_j$。他们共同拥有的 Public Inputs 为表格向量与查询向量的多项式承诺，因为我们现在只关注 Indexed Lookup Argument，因此他们还共同拥有 $M$ 的多项式承诺。

协议的第一步是 Verifier 发送一个挑战向量 $\vec{r}\in\mathbb{F}^{m}$，使得上面的约束转化为：

$$
\tilde{f}(\vec{r}) = \sum_{\vec{y}\in\{0,1\}^{\log{N}}} \tilde{M}(\vec{r}, \vec{y})\cdot \tilde{t}(\vec{y})
$$

Verifier 可以通过查询 Oracle $\tilde{f}$ 来得到 $\tilde{f}(\vec{r})$ 的值，我们记为 $v$。于是上面的等式就归约到了一个求和式：

$$
v = \sum_{\vec{y}\in\{0,1\}^{\log{N}}} \tilde{M}(\vec{r}, \vec{y})\cdot \tilde{t}(\vec{y})
$$

此刻，Prover 和 Verifier 可以调用 Sumcheck 协议来完成求和式的证明。但是 Prover 需要计算 $N$ 个 $\tilde{M}(\vec{r}, \vec{y})\cdot \tilde{t}(\vec{y})$ 的值。

在 Sumcheck 协议的结尾，Prover 和 Verifier 可以利用多项式承诺的求值证明，证明 $\tilde{M}(\vec{r}, \vec{Y})$ 和 $\tilde{t}(\vec{Y})$ 在 $\vec{Y}=\vec{\rho}$ 处的求值。尽管我们可以使用 Spark 稀疏多项式承诺来降低最后的求值证明的开销，但是 Prover 在 Sumcheck 协议过程中的计算量至少是 $O(m\cdot N)$。

下一节我们介绍 Generalized Lasso 如何再次利用 $M$ 矩阵的稀疏性，减少 Prover 在 Sumcheck 协议中的计算量。在此之前，我们先列出 Generalized Lasso 的协议框架：

### 协议框架

公共输入：

1. 表格向量 $\vec{t}$ 的承诺：$C_t=\mathsf{PCS.Commit}(\vec{t})$
2. 查询向量 $\vec{f}$ 的承诺：$C_f=\mathsf{PCS.Commit}(\vec{f})$
3. 表格选择矩阵 $M$ 的承诺：$C^{\mathsf{spark}}_M=\mathsf{Spark.Commit}(M)$

第一轮：Verifier 发送挑战向量 $\vec{r}\in\mathbb{F}^{\log{m}}$

第二轮：Prover 计算 $\tilde{f}(\vec{r})$ 的值，并且连同求值证明 $\pi_f$ 一起发送给 Verifier

第三轮：Prover 和 Verifier 进行 Sparse-dense Sumcheck 协议，证明下面的等式：

$$
v = \sum_{\vec{y}\in\{0,1\}^{\log{N}}} \tilde{M}(\vec{r}, \vec{y})\cdot \tilde{t}(\vec{y})
$$

经过 Sumcheck 协议，上面的约束等式被归约到：

$$
v'= \tilde{M}(\vec{r}, \vec{\rho})\cdot \tilde{t}(\vec{\rho})
$$

第四轮：Prover 发送 $v_M, v_t$ 与 求值证明 $\pi_M, \pi_t$ 给 Verifier 

1. $v_M = \tilde{M}(\vec{r}, \vec{\rho})$
2. $v_t = \tilde{t}(\vec{\rho})$

第五轮：Verifier 验证下面的等式：

1. $v'\overset{?}{=}v_M\cdot v_t$
2. $\mathsf{PCS.Verify}(C_t, v_t, \pi_t)\overset{?}{=}1$
2. $\mathsf{PCS.Verify}(C_f, v, \pi_f)\overset{?}{=}1$
3. $\mathsf{Spark.Verify}(C^{\mathsf{spark}}_M, v_M, \pi_M)\overset{?}{=}1$

## 3. Simplified Sparse-dense Sumcheck

这一节，我们分析下 Prover 在 Sumcheck 协议中的开销，以及如何利用 $M$ 的稀疏性质来减少 Prover 的计算量。

再重复下 Sumcheck 要证明的求和等式：

$$
v = \sum_{\vec{b}\in\{0,1\}^{\log{N}}} \tilde{u}(\vec{b})\cdot \tilde{t}(\vec{b})
$$

这里我们用 $\tilde{u}(\vec{b})$ 代替 $\tilde{M}(\vec{r},\vec{b})$，它是一个稀疏的多项式，只有 $m$ 个非零项。而 $\tilde{t}(\vec{b})$ 是一个 MLE-structured 的多项式，它的计算复杂度为 $O(\log{N})$。

Sumcheck 协议总共 $\log{N}$ 轮，在每一轮，Prover 主要的计算量为计算一个一元多项式并发送给 Verifier：

$$
h^{(j)}(X) = \sum_{(b_{j+1}, b_{j+2},\ldots, b_{\log{N}})\in\{0,1\}^{\log{N}-j-1}} \tilde{u}(r_0,r_1,\ldots, r_{j-1}, X, b_{j+1}, b_{j+2},\ldots, b_{\log{N}-1})\cdot \tilde{t}(r_0,r_1,\ldots, r_{j-1}, X, b_{j+1}, b_{j+2},\ldots, b_{\log{N}-1})
$$

注意到这个一元多项式 $h^{(j)}(X)$ 是 $O(N)$ 个项的求和，但是 $\tilde{u}(\vec{b})$ 是一个稀疏的 MLE 多项式。如果 $\tilde{u}(\vec{X})$ 在 $\vec{X}=\vec{y}$ 处的取值为零，那么 Prover 也就可以省去计算 $\tilde{t}(\vec{y})$ 的开销。因此，Prover 实际上只需要计算 $O(m)$ 个项的求和，而不是 $O(N)$ 个项。

进一步展开 $\tilde{u}(\vec{b})$ 的定义，我们可以得到：

$$
\tilde{u}(b_0, b_1, \ldots, b_{\log{N}-1}) = \sum_{i\in S_u} u_i\cdot \tilde{eq}_i(b_0, b_1, \ldots, b_{\log{N}-1})
$$

其中 $S_u$ 定义为 $\vec{u}$ 中非零项的索引集合。因此 $h^{(j)}(X)$ 求和式可以进一步简化为 $m$ 项的求和：

$$
\small
\begin{split}
h^{(j)}(X) &= \sum_{(b_{j+1}, b_{j+2},\ldots, b_{\log{N}})\in\{0,1\}^{\log{N}-j-1}} \tilde{u}(r_0,r_1,\ldots, r_{j-1}, X, b_{j+1}, b_{j+2},\ldots, b_{\log{N}-1})\cdot \tilde{t}(r_0,r_1,\ldots, r_{j-1}, X, b_{j+1}, b_{j+2},\ldots, b_{\log{N}-1}) \\
 &= \sum_{i\in S_u} u_i\cdot \sum_{(b_{j+1}, b_{j+2},\ldots, b_{\log{N}})\in\{0,1\}^{\log{N}-j-1}} \tilde{eq}_i(r_0,r_1,\ldots, r_{j-1}, X, b_{j+1}, b_{j+2},\ldots, b_{\log{N}-1})\cdot \tilde{t}(r_0,r_1,\ldots, r_{j-1}, X, b_{j+1}, b_{j+2},\ldots, b_{\log{N}-1}) \\
 & = \sum_{i\in S_u} u_i \cdot \tilde{eq}_i(r_0,r_1,\ldots, r_{j-1}, X, i_{j+1}, i_{j+2},\ldots, i_{\log{N}-1})\cdot \tilde{t}(r_0,r_1,\ldots, r_{j-1}, X, i_{j+1}, i_{j+2},\ldots, i_{\log{N}-1})
\end{split}
$$

每一轮，假设当前我们在第 $j$ 轮，Prover 要计算 $m$ 个项的求和，每一项包含两个乘法和两个 MLE 多项式的求值，分别为 $\tilde{eq}_i$ 和 $\tilde{t}$ 。接着 Verifier 都会提供一个随机数 $r_j$ 来求值 $h^{(j)}(X)$，然后 Sumcheck 进入下一轮，即第 $j+1$ 轮。

在第 $j$ 轮， Prover 的策略是根据上一轮（第 $j-1$ 轮）的 $\tilde{eq}_i(\ldots,r_{j-1},i_{j},\ldots)$ 和 $\tilde{t}(\ldots,r_{j-1},i_{j},\ldots)$ 的求值来增量式的递推计算  $\tilde{eq}_i(\ldots,r_{j-1},r_{j},\ldots)$ 和 $\tilde{t}(\ldots,r_{j-1},r_{j},\ldots)$，即用 $r_j$ 来代替 $i_j$。

然后我们观察下 $\tilde{eq}_i(r_0,r_1,\ldots, r_{j-1}, X, i_{j+1}, i_{j+2},\ldots, i_{\log{N}-1})$ 的定义，

$$
\tilde{eq}_i(r_0,r_1,\ldots, r_{j-1}, X, i_{j+1}, i_{j+2},\ldots, i_{\log{N}-1})
= \Big(\prod_{k=0}^{j-1}(1-i_k)\cdot(1-r_k)+i_k\cdot r_k\Big) \cdot \Big((1-X)\cdot (1-i_{j}) + X\cdot i_{j}\Big) \cdot \Big(\prod_{k=j+1}^{\log{N}-1}(1-i_k)(1-i_k)+i_k\cdot i_k\Big)
$$

注意到等式右边的三个乘积因子中的最右边一个恒等于 1。如果 $X=i_j$，

$$
\tilde{eq}_i(r_0,r_1,\ldots, r_{j-1}, {\color{blue}i_j}, i_{j+1}, i_{j+2},\ldots, i_{\log{N}-1})
= \prod_{k=0}^{j-1}(1-i_k)\cdot(1-r_k)+i_k\cdot r_k
$$

那么当我们用 ${\color{red}r_j}$ 来代替 ${\color{blue}i_j}$ 时，

$$
\begin{split}
\tilde{eq}_i(r_0,r_1,\ldots, r_{j-1}, {\color{red}r_j}, i_{j+1}, i_{j+2},\ldots, i_{\log{N}-1})
&= \Big(\prod_{k=0}^{j-1}(1-i_k)\cdot(1-r_k)+i_k\cdot r_k\Big)\cdot\Big((1-{\color{red}r_j})\cdot(1-i_j)+{\color{red}r_j}\cdot i_j\Big) \\
&= \tilde{eq}_i(r_0,r_1,\ldots, r_{j-1}, {\color{blue}i_j}, i_{j+1}, i_{j+2},\ldots, i_{\log{N}-1})
\cdot\Big((1-{\color{red}r_j})\cdot(1-i_j)+{\color{red}r_j}\cdot i_j\Big)
\end{split}
$$

因此，根据 $i_j$ 是 $0$ 还是 $1$，Prover 可以仅用一个乘法即可递推地计算出第 $j+1$ 轮所需要的 $\tilde{eq}_i$。又因为总共有 $m$ 个 $\tilde{eq}_i$ 需要计算，所以 Prover 要付出 $O(m)$ 的计算量。

Prover 可以维护一个长度为 $m$ 的数组，里面保存 $\tilde{eq}_i$ 的值，每一轮过后就更新这个数组：

$$
\tilde{eq}_i(r_0,r_1,\ldots, r_{j-1}, {\color{red}r_j}, i_{j+1}, i_{j+2},\ldots, i_{\log{N}-1})
= \begin{cases}
\tilde{eq}_i(r_0,r_1,\ldots, r_{j-1}, {\color{blue}i_j}, i_{j+1}, i_{j+2},\ldots, i_{\log{N}-1})\cdot(1-{\color{red}r_j}) & \text{if } i_j=0 \\
\tilde{eq}_i(r_0,r_1,\ldots, r_{j-1}, {\color{blue}i_j}, i_{j+1}, i_{j+2},\ldots, i_{\log{N}-1})\cdot {\color{red}r_j} & \text{if } i_j=1 \\
\end{cases}
$$

但是对于 $\tilde{t}(r_0,r_1,\ldots, r_{j-1}, X, i_{j+1}, i_{j+2},\ldots, i_{\log{N}-1})$ 这个求值运算，如果 $\tilde{t}$ 没有内部结构，那么 Prover 需要老老实实进行求值运算。这样每一轮中 Prover 仍然需要执行 $m$ 次 MLE 运算求值过程。在 $\log{N}$ 轮的 Sumcheck 协议过程中，Prover 总共的计算量至少为

$$
O(m\cdot\log{N}\cdot \mathsf{evaltime}(\tilde{t}))
$$

如果 $\vec{t}$ 恰好具有 MLE-Structured 性质，那么 $\mathsf{evaltime}(\tilde{t})$ 为 $O(\log{N})$，那么 Prover 的计算量为 $O(m\cdot\log^2{N})$。

进一步，如果 $\vec{t}$ 具有「局部 bit 相关」的特性，即我们可以采用计算 $\tilde{eq}_i$ 的方法给出 $\tilde{t}$ 的递推计算式：

$$
\tilde{t}(r_0, \ldots, r_{j-1}, {\color{red}X}, b_{j+1}, \ldots, b_{\log{N}-1}) = \mathsf{m}({\color{red}X}, j, {\color{blue}b_j})\cdot \tilde{t}(r_0, \ldots, r_{j-1}, {\color{blue}b_j}, b_{j+1}, \ldots, b_{\log{N}-1}) + \mathsf{a}({\color{red}X}, j, {\color{blue}b_j})
$$

这里 $\mathsf{m}_l({\color{red}X}, j, {\color{blue}b_j})$ 和 $\mathsf{a}_l({\color{red}X}, j, {\color{blue}b_j})$ 是两个多项式，他们的计算复杂度为 $O(1)$。如果 $\tilde{t}$ 的递推计算能满足上面的等式，那么 Prover 就可以同样维护一个长度为 $m$ 的数组，保存 $i\in S_u$ 所对应的 $\tilde{t}(r_0, \ldots, r_{j-1}, r_j, {\color{blue}i_{j+1}, \ldots, i_{\log{N}-1}})$ 的值。这样 Prover
可以在每一轮中只需要总共 $O(m)$ 的计算量来计算更新所有需要用到的 $\tilde{t}$ 的值。

这样一来， Prover 的计算量可以进一步降低为 $O(m\cdot\log{N})$。

下面举一个简单例子，来演示下整个过程。假设 $N=8$，一个稀疏的向量 $\vec{u}=(0, u_1, u_2, 0, u_4, 0, 0, 0)$ ，表格向量 $\vec{t}=(t_0,t_1,t_2,t_3,t_4,t_5, t_6, t_7)$。稀疏向量的非零值数量为 $m=3$，$S_u=(1,2,4)$。

Sumcheck 协议要证明的求和式为：

$$
v = \sum_{\vec{b}\in\{0,1\}^3}^{} \tilde{u}(\vec{b})\cdot \tilde{t}(\vec{b})=u_1\cdot t_1 + u_2\cdot t_2 + u_4\cdot t_4
$$

Prover 预计算 $E^{(0)}(i)=\tilde{eq}_i(i_0,\ldots,i_{\log{N}-1})$ 的值为 1， $T^{(0)}(i)=\tilde{t}(i_0,\ldots,i_{\log{N}-1})=t_i$

在第 0 轮中，Prover 计算 $h^{(1)}(X)$：

$$
\begin{split}
h^{(1)}(X) &= \sum_{b_2,b_3\in\{0,1\}} \tilde{u}(X, b_2, b_3)\cdot \tilde{t}(X, b_2, b_3) \\
&= u_1 \cdot E^{(0)}(1) \cdot \tilde{t}(X,0,1)\cdot (1-X)\\
&\  + u_2 \cdot E^{(0)}(2)\cdot \tilde{t}(X,1,0) \cdot (1-X) \\
&\  + u_4 \cdot E^{(0)}(4)\cdot \tilde{t}(X,0,0) \cdot X\\
&= u_1 \cdot E^{(0)}(1) \cdot \Big(\mathsf{m}(X, j=0, b_0=0)\cdot \tilde{t}(b_0=0, b_1=0, b_2=1) + \mathsf{a}(X, j=0, b_0=0)\Big)\cdot (1-X)\\
&\  + u_2 \cdot E^{(0)}(2)\cdot \Big(\mathsf{m}(X, j=0, b_0=0)\cdot \tilde{t}(b_0=0, b_1=1, b_2=0) + \mathsf{a}(X, j=0, b_0=0)\Big) \cdot (1-X) \\
&\  + u_4 \cdot E^{(0)}(4)\cdot \Big(\mathsf{m}(X, j=0, b_0=1)\cdot \tilde{t}(b_0=1, b_1=0, b_2=0) + \mathsf{a}(X, j=0, b_0=1)\Big) \cdot X\\
&= u_1 \cdot E^{(0)}(1) \cdot \Big(\mathsf{m}(X, j=0, b_0=0)\cdot T^{(0)}(1) + \mathsf{a}(X, j=0, b_0=0)\Big)\cdot (1-X)\\
&\  + u_2 \cdot E^{(0)}(2)\cdot \Big(\mathsf{m}(X, j=0, b_0=0)\cdot T^{(0)}(2) + \mathsf{a}(X, j=0, b_0=0)\Big) \cdot (1-X) \\
&\  + u_4 \cdot E^{(0)}(4)\cdot \Big(\mathsf{m}(X, j=0, b_0=1)\cdot T^{(0)}(4) + \mathsf{a}(X, j=0, b_0=1)\Big) \cdot X\\
\end{split}
$$

可以看出，Prover 只需要计算 $\mathsf{m}(X, 0, 0)$，$\mathsf{m}(X, 0, 1)$ ，$\mathsf{a}(X, 0, 0)$，$\mathsf{a}(X, 0, 1)$ 这四个多项式的值。而这四个多项式的计算量为 $O(1)$。Prover 发送 $\big(h^{(1)}(0)), h^{(1)}(1), h^{(1)}(2)\big)$
作为 $h^{(1)}(X)$ 的点值形式发送。

Verifier 发送挑战数 $r_0$，Prover 和 Verifier 检查 

$$
v\overset{?}{=}h^{(1)}(0) + h^{(1)}(1)
$$

然后共同计算 $h^{(1)}(r_0)$ 作为新的求和。

Prover 更新 $E(i)$ 与 $T(i)$ 到 $E^{(1)}$ 与 $T^{(1)}$，

$$
\begin{split}
E^{(1)}(1) &= E^{(0)}(1)\cdot (1-r_0) = (1-r_0)\\
E^{(1)}(2) &= E^{(0)}(2)\cdot (1-r_0) = (1-r_0)\\
E^{(1)}(4) &= E^{(0)}(4)\cdot r_0 = r_0        \\
\end{split}
$$

$$
\begin{split}
T^{(1)}(1) &= \mathsf{m}(r_0, 0, 0)\cdot T^{(0)}(1) + \mathsf{a}(r_0, 0, 0)\\
T^{(1)}(2) &= \mathsf{m}(r_0, 0, 0)\cdot T^{(0)}(2) + \mathsf{a}(r_0, 0, 0)\\
T^{(1)}(4) &= \mathsf{m}(r_0, 0, 1)\cdot T^{(0)}(4) + \mathsf{a}(r_0, 0, 1)\\
\end{split}
$$

下面是第二轮，Prover 计算 $h^{(2)}(X)$：

$$
\begin{split}
h^{(2)}(X) &= \sum_{b_3\in\{0,1\}} \tilde{u}(r_0, X, b_3)\cdot \tilde{t}(r_0, X, b_3) \\
&= u_1 \cdot E^{(1)}(1) \cdot \tilde{t}(r_0,X,1)\cdot (1-X)\\
&\quad  + u_2 \cdot E^{(1)}(2)\cdot \tilde{t}(r_0,X,0) \cdot X \\
&\quad + u_4 \cdot E^{(1)}(4)\cdot \tilde{t}(r_0,X,0) \cdot (1-X)\\
&= u_1 \cdot E^{(1)}(1) \cdot \Big(\mathsf{m}(X, j=1, b_1=0)\cdot \tilde{t}(r_0, b_1=0, b_2=1) + \mathsf{a}(X, j=1, b_1=0)\Big)\cdot (1-X) \\
&\quad + u_2 \cdot E^{(1)}(2)\cdot \Big(\mathsf{m}(X, j=1, b_1=1)\cdot \tilde{t}(r_0, b_1=1, b_2=0) + \mathsf{a}(X, j=1, b_1=1)\Big) \cdot X \\
&\quad  + u_4 \cdot E^{(1)}(4)\cdot \Big(\mathsf{m}(X, j=1, b_1=0)\cdot \tilde{t}(r_0, b_1=0, b_2=0) + \mathsf{a}(X, j=1, b_1=0)\Big) \cdot (1-X) \\
&= u_1 \cdot E^{(1)}(1) \cdot \Big(\mathsf{m}(X, j=1, b_0=0)\cdot T^{(1)}(1) + \mathsf{a}(X, j=1, b_1=0)\Big)\cdot (1-X)\\
&\quad  + u_2 \cdot E^{(1)}(2)\cdot \Big(\mathsf{m}(X, j=1, b_0=1)\cdot T^{(1)}(2) + \mathsf{a}(X, j=1, b_1=1)\Big) \cdot  X \\
&\quad  + u_4 \cdot E^{(1)}(4)\cdot \Big(\mathsf{m}(X, j=1, b_0=0)\cdot T^{(1)}(4) + \mathsf{a}(X, j=1, b_1=0)\Big) \cdot (1-X)\\
\end{split}
$$

Prover 只需要计算 $\mathsf{m}(X, 1, 0)$，$\mathsf{m}(X, 1, 1)$ ，$\mathsf{a}(X, 1, 0)$，$\mathsf{a}(X, 1, 1)$ 这四个多项式的值。而这四个多项式的计算量为 $O(1)$。Prover 发送 $\big(h^{(2)}(0)), h^{(2)}(1), h^{(2)}(2)\big)$
作为 $h^{(2)}(X)$ 的点值形式发送。

Verifier 发送挑战数 $r_1$，Prover 和 Verifier 检查 

$$
h^{(1)}(r_1)\overset{?}{=}h^{(2)}(0) + h^{(2)}(1)
$$

Prover 维护 $E$ 与 $T$ 数组，更新到 $E^{(2)}$ 与 $T^{(2)}$，

$$
\begin{array}{lll}
E^{(2)}(1) &= E^{(1)}(1)\cdot (1-r_1) &= (1-r_0)\cdot(1-r_1)\\
E^{(2)}(2) &= E^{(1)}(2)\cdot r_1     &= (1-r_0)\cdot r_1\\
E^{(2)}(4) &= E^{(1)}(4)\cdot (1-r_1) &= r_0\cdot (1-r_1)       \\
\end{array}
$$

$$
\begin{split}
T^{(2)}(1) &= \mathsf{m}(r_1, 1, 0)\cdot T^{(1)}(1) + \mathsf{a}(r_1, 1, 0)\\
T^{(2)}(2) &= \mathsf{m}(r_1, 1, 1)\cdot T^{(1)}(2) + \mathsf{a}(r_1, 1, 1)\\
T^{(2)}(4) &= \mathsf{m}(r_1, 1, 0)\cdot T^{(1)}(4) + \mathsf{a}(r_1, 1, 0)\\
\end{split}
$$

到了第三轮，Prover 计算 $h^{(3)}(X)$：

$$
\begin{split}
h^{(3)}(X) &= \tilde{u}(r_0, r_1, X)\cdot \tilde{t}(r_0, r_1, X) \\
&= u_1 \cdot E^{(2)}(1) \cdot \tilde{t}(r_0, r_1, X)\cdot X\\
&\quad  + u_2 \cdot E^{(2)}(2)\cdot \tilde{t}(r_0, r_1, X) \cdot (1-X) \\
&\quad + u_4 \cdot E^{(2)}(4)\cdot \tilde{t}(r_0, r_1, X) \cdot (1-X)\\
\end{split}
$$

如果 $\tilde{t}$ 有内部结构，那么

$$
\begin{split}
h^{(3)}(X) &= u_1 \cdot E^{(2)}(1) \cdot \Big(\mathsf{m}(X, j=2, b_2=1)\cdot \tilde{t}(r_0, r_1, b_2=1) + \mathsf{a}(X, j=2, b_2=1)\Big)\cdot (1-X) \\
&\quad + u_2 \cdot E^{(2)}(2)\cdot \Big(\mathsf{m}(X, j=2, b_2=0)\cdot \tilde{t}(r_0, r_1, b_2=0) + \mathsf{a}(X, j=2, b_2=0)\Big) \cdot X \\
&\quad  + u_4 \cdot E^{(2)}(4)\cdot \Big(\mathsf{m}(X, j=2, b_2=0)\cdot \tilde{t}(r_0, r_1, b_2=0) + \mathsf{a}(X, j=2, b_2=0)\Big) \cdot (1-X) \\
&= u_1 \cdot E^{(2)}(1) \cdot \Big(\mathsf{m}(X, j=2, b_2=1)\cdot T^{(2)}(1) + \mathsf{a}(X, j=2, b_2=1)\Big)\cdot (1-X)\\
&\quad  + u_2 \cdot E^{(2)}(2)\cdot \Big(\mathsf{m}(X, j=2, b_2=0)\cdot T^{(2)}(2) + \mathsf{a}(X, j=2, b_2=0)\Big) \cdot  X \\
&\quad  + u_4 \cdot E^{(2)}(4)\cdot \Big(\mathsf{m}(X, j=2, b_2=0)\cdot T^{(2)}(4) + \mathsf{a}(X, j=2, b_2=0)\Big) \cdot (1-X)\\
\end{split}
$$

Prover 发送 $\big(h^{(3)}(0)), h^{(2)}(1), h^{(3)}(2)\big)$
作为 $h^{(3)}(X)$ 的点值形式发送。

Verifier 发送挑战数 $r_2$，Prover 和 Verifier 检查 

$$
h^{(2)}(r_2)\overset{?}{=}h^{(3)}(0) + h^{(3)}(1)
$$

Prover 和 Verifier 最后通过 PCS 来验证下面的 Evaluation 等式：

$$
h^{(3)}(r_3) \overset{?}{=} \tilde{u}(r_0, r_1, r_2)\cdot \tilde{t}(r_0, r_1, r_2) 
$$

Prover 更新 $E$ 到 $E^{(3)}$，则得到 $v_u=\tilde{u}(r_0, r_1, r_2)$：

$$
\begin{array}{lll}
E^{(3)}(1) &= E^{(2)}(1)\cdot r_2 &= (1-r_0)\cdot(1-r_1)\cdot r_2\\
E^{(3)}(2) &= E^{(2)}(2)\cdot (1-r_2)     &= (1-r_0)\cdot r_1 \cdot (1-r_2)\\
E^{(3)}(4) &= E^{(2)}(4)\cdot (1-r_2) &= r_0\cdot (1-r_1)\cdot(1-r_2)       \\
\end{array}
$$

Prover 可以通过 $E^{(3)}$ 来计算得到 $v_u=\tilde{u}(r_0, r_1, r_2)$，计算时间复杂度为 $O(m)$：

$$
\tilde{u}(r_0, r_1, r_2) = u_1 \cdot E^{(3)}(1) + u_2 \cdot E^{(3)}(2) + u_4 \cdot E^{(3)}(4)
$$

Prover 并不需要发送 $\tilde{t}(r_0, r_1, r_2)$，因为这个值可以由 Verifier 直接计算得到，计算时间复杂度为 $O(m)$。

综上，Prover 的计算量为 $O(m\cdot\log{N})$。

## 4. Standard Sparse-dense Sumcheck

标准的 Sparse-dense Sumcheck 可以把 $\log{N}$ 轮的 Sumcheck 过程拆分成 $c=\frac{\log{N}}{\log{m}}$ 个分段，在每个分段中，Prover 都预计算一些辅助的向量，从而避免在接下来的 Sumcheck 分段中做一些重复的计算。这个分段加预计算的步骤被称为 Condensation。通过这种方法，Prover 的计算量可以从 $O(m\cdot\log{N})$ 降到 $O(c\cdot m)$，其中 $c=\frac{\log{N}}{\log{m}}$，即 $N=m^c$。

### 4.1 理解 Condensation

我们先描述一个 Sparse-dense Sumcheck 简单情况。假设查询表格 $\vec{t}$ 中的每一个表项 $t_i$ 都可以用它的 index 的二进制位来计算，例如 $t_i$ 的值可以通过下面的方式计算：

$$
t(i_0, i_1, \ldots, i_{s-1}) = d_0i_0 + d_1i_1 + \cdots + d_{s-1}i_{s-1}
$$

其中 $i=i_0 + i_1\cdot 2 + \cdots + i_{s-1} \cdot 2^{s-1}$ 为 $t_i$ 的表格索引。 那么如果给定 $t(i_0, \ldots, i_k, \ldots, i_{s-1})$ 的值，我们可以在常数时间内计算 $t(i_0, \ldots, c, \ldots, i_{s-1})$ 的值。

$$
t(i_0, \ldots, c, \ldots, i_{s-1}) = t(i_0, \ldots, i_k, \ldots, i_{s-1}) + {\color{blue}d_k\cdot(c - i_k)}
$$

上面这个等式想要表达的含义是：表格的每一项可以表达为该表项的索引（Index）的线性组合，并且是关于 Index 的二进制位的一次多项式。例如 RangeCheck 表格就满足这个特征。

回忆 Generalized Lasso 协议，其核心是证明下面的等式：

$$
\tilde{f}(\vec{X}) = \sum_{\vec{y}\in\{0,1\}^{\log{N}}} \tilde{M}(\vec{X}, \vec{y})\cdot \tilde{t}(\vec{y})
$$

通过 Verifier 提供的一个随机挑战数，上面的等式可以转化为：

$$
\tilde{f}(\vec{r}) = \sum_{\vec{y}\in\{0,1\}^{\log{N}}} \tilde{M}(\vec{r}, \vec{y})\cdot \tilde{t}(\vec{y})
$$

令 $\vec{u} = \tilde{M}(\vec{r}, \vec{y})$，那么等式转换为一个 Inner Product 的形式：

$$
\tilde{f}(\vec{r}) = \sum_{\vec{b}\in\{0,1\}^{\log{N}}}\tilde{u}(\vec{b})\cdot \tilde{t}(\vec{b})
$$

等式右边是一个 $N$ 项的求和式，如果直接让 Prover 去计算每一项中的 $\tilde{u}(\vec{b})$ 和 $\tilde{t}(\vec{b})$，那么 Prover 的计算量至少为 $2N$ 次 evaluation。但是我们可以利用 $\tilde{u}(\vec{b})$ 和 $\tilde{t}(\vec{b})$ 的内部结构来进行优化。首先  $\tilde{u}(\vec{b})$ 编码了一个长度为 $N$ 的向量，记为 $\vec{u}$，它相当于也编码了矩阵 $M$ 的信息，只有 $m$ 个非零值。因此我们只需要 $O(m)$ 就可以计算出所有的 $\tilde{u}(\vec{X})$ 在所有 $\vec{X}=\vec{b}$ 处的取值。其次 $\tilde{t}(\vec{b})$ 编码了 $\vec{t}$，它是一个 MLE-structured 的表格，其每一项都与 Index 的二进制位有关，因此每一个表项 $t_i$ 都可以在 $O(\log{N})$ 时间内计算得到。最后，考虑求和式 $\sum_{\vec{b}\in\{0,1\}^{\log{N}}}\tilde{u}(\vec{b})\cdot \tilde{t}(\vec{b})$ 中若 $\tilde{u}(\vec{b})=0$，那么我们就不需要再计算 $\tilde{t}(\vec{b})$。因此，这个求和式整体上也只需要 $O(m)$ 次 evaluation 即可。

这个 $N$ 项的求和过程如果使用 Sumcheck 协议来证明，那么需要 $\log{N}$ 轮。在第 $j$ 轮中，由于我们都可以根据上一轮 $\tilde{t}(r_0, r_1, \ldots, r_{j-1}, {\color{red} b_{j}},\ldots, b_{\log{N}-1})$ 来计算 $\tilde{t}(r_0, r_1, \ldots, r_{j-1}, {\color{red} r_{j}},\ldots, b_{\log{N}-1})$，因此只需要常数时间的计算量。

我们引入两个辅助的向量 $\vec{q}$ 与 $\vec{z}$，Prover 可以在 Sumcheck 协议运行前就计算好 $\vec{q}$ 与 $\vec{z}$ 的值，然后 Prover 可以利用这些预计算的向量，在 Sumcheck 协议的前 $\log{m}$ 轮中（记住，Sumcheck 协议总共有 $\log{N}$ 轮，我们假设 $\log{N} > \log{m}$）加速计算求和式。这两个向量的每个元素 $q_k$ 与 $z_k$，$k \in [0, m)$ 定义如下：

$$
q_k = \sum_{y\in\mathsf{extend}(k, \log{m}, \log{N})} \tilde{u}(\vec{y})\cdot \tilde{t}(\vec{y})
$$

$$
z_k = \sum_{y\in\mathsf{extend}(k, \log{m}, \log{N})} \tilde{u}(\vec{y})
$$

这里我们引入了一个新的符号：$\mathsf{extend}(k, \log{m}, \log{N})$，它是一个二进制串的集合 

$$
\mathsf{extend}(k, \log{m}, \log{N}) = \{ y \in \{0,1\}^{\log{N}} \mid \mathsf{prefix}(y) = \mathsf{bits}(k) \}
$$

然后筛选那些二进制串的前 $\log{m}$ 位与 $k$ 的二进制位相等（我们采用 Big-endian 的表示方式，前面的位为高位），而后面的 $\log{N} -\log{m}$ 位可以任意值。例如，$\mathsf{extend}(10_{(2)}, 2, 4) = \{\underline{10}00_{(2)}, \underline{10}01_{(2)}, \underline{10}10_{(2)}, \underline{10}11_{(2)}\}$。这个集合中每一个二进制串都是以 $10$ 打头。

那么 $\vec{q}$ 向量的每一个元素 $q_i$，是筛选出那些高位等于 $\mathsf{bits}(i)$ 的二进制串（长度为 $\log{N}$）$y$，然后通过 $y$ 为索引，计算 $\tilde{u}(\vec{y})\cdot \tilde{t}(\vec{y})$ 的求和。换句话说，我们通过 $\vec{q}$ 把前面 $N$ 项求和式 $\sum_{\vec{b}\in\{0,1\}^{\log{N}}}\tilde{u}(\vec{b})\cdot \tilde{t}(\vec{b})$ 划分为了 $\log{m}$ 个子集，然后分别对其进行求和。由于 $\tilde{u}$ 的 $O(m)$ 稀疏性，$\vec{q}$ 的计算量也是 $O(m)$。

再换一个思路去理解，如果我们把 $N$ 项求和式的计算过程描述成一棵深度为 $\log{N}$ 的二叉树，其中树根（第 0 层）为最后的和。而其中每个叶子节点都是 $\tilde{u}(\vec{b})\cdot \tilde{t}(\vec{b})$，这里 $\vec{b}\in\{0,1\}^{\log{N}}$，因此总共有 $N$ 个叶子。那么向量 $\vec{q}$ 就是这颗二叉树中第 $\log{m}-1$ 层的所有节点。

同样， $\vec{k}$ 是叶子结点为 $\tilde{u}(\vec{b})$ 的求和二叉树中的第 $\log{m}-1$ 层。
接下来，我们看看 Prover 在 Sumcheck 协议中前 $\log{m}$ 轮的计算过程（总共有 $\log{N}$ 轮），并且看下这两个辅助向量的作用。

在第一轮中，Prover 要构造一个一元多项式 $h^{(1)}(X)$

$$
\begin{split}
h^{(1)}(X) &=\sum_{(b_1,b_2,\ldots, b_{\log{N}-1})\in\{0,1\}^{\log{N}-1}}\tilde{u}(X, b_1, b_2, \ldots, b_{\log{N}-1})\cdot \tilde{t}(X, b_1, b_2, \ldots, b_{\log{N}-1}) \\
\end{split}
$$

我们把 $\tilde{u}(X, b_1, b_2, \ldots, b_{\log{N}-1})$ 按照定义展开，可以得到：

$$
\begin{split}
h^{(1)}(X) &= \sum_{(b_1,b_2,\ldots, b_{\log{N}-1})\in\{0,1\}^{\log{N}-1}}\Big(\sum_{i\in S_u}u_i\cdot \tilde{eq}_i(X, b_1, b_2, \ldots, b_{\log{N}-1})\Big) \cdot \tilde{t}(X, b_1, b_2, \ldots, b_{\log{N}-1}) \\
\end{split}
$$

这里的 $S_u$ 表示 $\vec{u}$ 中非零元素的索引（的二进制表示）的集合。然后把所有的求和号都展开，我们发现当 $\mathsf{bits}(i)=(i_0,i_1,\ldots,i_{\log{N}-1})\not\in S_u$ 时，$u_i=0$，因此，我们只要关注上面求和式中 $\vec{b}=\mathsf{bits}(i)$ 对应的那些非零项（这时候 $u_i\not= 0$）：

$$
h^{(1)}(X)= \sum_{i\in S_u}u_i\cdot \tilde{eq}_i(X, i_1, i_2, \ldots, i_{\log{N}-1})\cdot \tilde{t}(X, i_1, i_2, \ldots, i_{\log{N}-1}) \\
$$

上面的等式已经变成了一个 $m$ 项的求和式。接下来我们根据 $\vec{t}$ 的结构性，展开 $\tilde{t}(X, i_1, i_2, \ldots, i_{\log{N}-1})$，并且根据 $\tilde{eq}_i(X, i_1, i_2, \ldots, i_{\log{N}-1})$ 的 Tensor 结构，即 $\tilde{eq}_i(X, i_1, i_2, \ldots, i_{\log{N}-1})= \tilde{eq}_{i_0}(X)\cdot \tilde{eq}_{i}(i_1, \ldots, i_{\log{N}-1})=\tilde{eq}_{i_0}(X)$， 我们可以得到：

$$
h^{(1)}(X)=  \sum_{i\in S_u}u_i\cdot \tilde{eq}_{i_0}(X)\cdot \Big(\tilde{t}(i_0, i_1, i_2, \ldots, i_{\log{N}-1}) + {\color{blue}(X-i_0)\cdot d_0}\Big) \\
$$

我们再展开 $\tilde{eq}_{i_0}(X)=(1-X)\cdot (1-i_0) + X\cdot i_0$，可得：

$$
\begin{split}
h^{(1)}(X) &= \sum_{i\in S_u}u_i\cdot \Big((1-X)\cdot (1-i_0) + X\cdot i_0\Big)\cdot \Big(\tilde{t}(i_0, i_1, i_2, \ldots, i_{\log{N}-1}) + {\color{blue}(X-i_0)\cdot d_0}\Big) \\
&= \sum_{i=({\color{red}i_0=0},i_2,\ldots,i_{\log{N}-1})\in S_u} (1-X)\cdot u_i\cdot \tilde{t}(i_0, i_1, i_2, \ldots, i_{\log{N}-1})  + (1-X)\cdot X \cdot d_0\cdot u_i \\
& \quad + \sum_{i=({\color{red}i_0=1},i_2,\ldots,i_{\log{N}-1})\in S_u} X\cdot u_i\cdot \tilde{t}(i_0, i_1, i_2, \ldots, i_{\log{N}-1}) + X\cdot (X-1)\cdot d_0\cdot u_i \\
\end{split}
$$

这时候，我们可以代入之前计算的辅助向量 $\vec{q}$ 与 $\vec{z}$，其中第 $k$ 项为
$$
\begin{split}
q_k&=q_{(k_0,k_1,\ldots, k_{\log{m}-1})}=\sum_{(j_0,j_1,\ldots,j_{\log{N}-\log{m}-1})\in\{0,1\}^{\log{N}-\log{m}}}\tilde{u}\big(k_0,k_1,\ldots, k_{\log{m}-1},j_0,j_1,\ldots, j_{\log{N}-\log{m}-1}\big)\cdot \tilde{t}\big(k_0,k_1,\ldots, k_{\log{m}-1},j_0,j_1,\ldots, j_{\log{N}-\log{m}-1}\big) \\
z_k&=z_{(k_0,k_1,\ldots, k_{\log{m}-1})}=\sum_{(j_0,j_1,\ldots,j_{\log{N}-\log{m}-1})\in\{0,1\}^{\log{N}-\log{m}}}\tilde{u}\big(k_0,k_1,\ldots, k_{\log{m}-1},j_0,j_1,\ldots, j_{\log{N}-\log{m}-1}\big)\\
\end{split}
$$

注意到，$q_k$ 与 $z_k$ 是按照前 $\log{m}$ 位进行划分后的第 $k$ 个子集的求和。我们可以把 $h^{(1)}(X)$ 重新写成 两个 $\log{m}$ 项的求和式：

$$
\begin{split}
h^{(1)}(X) &= \sum_{({\color{red}{0}}, k_1,k_2,\ldots, k_{\log{m}-1})\in\{0,1\}^{\log{m}}}(1-X)\cdot q_k + d_0\cdot (1-X)X\cdot z_k \\
           &+  \sum_{({\color{red}{1}}, k_1,k_2,\ldots, k_{\log{m}-1})\in\{0,1\}^{\log{m}}}X\cdot q_k + d_0\cdot (1-X)X\cdot z_k \\
\end{split}
$$

这样 Prover 只需要 $O(m)$ 的计算量就可以完成第一轮的计算，这个计算包括计算 $h^{(1)}(X)$ 在 $X=0, 1, 2$ 处的求值运算。

那么第二轮开始到第 $\log{m}$ 轮，每一轮 Prover 都只需要 $O(m)$ 的计算量就可以完成计算 $h^{(j)}(X)$ 在 $X=0, 1, 2$ 处的求值。

但是到了 第 $\log{m}+1$ 轮，情况会发生变化，因为这时候 Prover 要计算 $N - m$ 个项的求和，并且 $\vec{q}$ 和 $\vec{z}$ 两个辅助向量已经失效。因此，对于下面新的 $\log{m}$ 个轮次，Prover 需要重新计算 $\vec{q}$ 与 $\vec{z}$，然后按照上面的方案继续。这样相当于把 $\log{N}$ 轮的 Sumcheck 按照 $\log{m}$ 的数量进行划分，每一个 $\log{m}$ 轮次，Prover 都需要重新计算 $\vec{q}$ 与 $\vec{z}$，然后保证 $h^{(j)}(X)$ 的计算始终是 $2m$ 个项的求和式。每次重新计算 $\vec{q}$ 与 $\vec{z}$ 的操作被称为 Condensation。

### 3.2 一般性 $\vec{t}$ 结构

上一小节的讨论基于一个比较强的表格结构：

$$
t(i_0, i_1, \ldots, i_{s-1}) = d_0i_0 + d_1i_1 + \cdots + d_{s-1}i_{s-1}
$$

对于 AND 表格与 SLT 表格等不满足上面结构要求的表格，我们需要放松表格的结构条件。首先， $\tilde{t}$ 可以是多个多项式之和，并且每一个多项式 $\tilde{t}_l$ 的计算过程基于 index 的二进制表示，当 index 发生变化时，$\tilde{t}$ 的值的变化只需要常数个加法和乘法操作：

$$
\tilde{t} = \sum_{l=0}^{\eta-1}\tilde{t}_l
$$

$$
\tilde{t}_l(r_0, \ldots, r_{j-1}, X, b_{j+1}, \ldots, b_{\log{N}-1}) = \mathsf{m}_l(X, j, b_j)\cdot \tilde{t}_l(r_0, \ldots, r_{j-1}, b_j, b_{j+1}, \ldots, b_{\log{N}-1}) + d_l\cdot (X - r_j) + \mathsf{a}_l(X, j, b_j)
$$

