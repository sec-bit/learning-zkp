$$
\begin{split}
e(X)\cdot(\beta - v(X)) & = \sum_{i=0}^{m-1} \tau_{\mathsf{col}(i)}(\beta)\cdot \mu_{i}(X)\cdot(\beta - v(X)) \\
& = (\sum_i e_i\cdot \mu_i(X))\cdot(\sum_i(\beta - v_i)\cdot \mu_i(X)) \\
&\equiv (\sum_i e_i\cdot (\beta - v_i)\cdot \mu_i(X))  \pmod{z_\mathbb{H}(X)}  \\
&\equiv (\sum_i \tau_{\mathsf{col}(i)}(\beta)\cdot (\beta - v_i)\cdot \mu_i(X))  \pmod{z_\mathbb{H}(X)}  \\
&\equiv (\sum_i \frac{\omega_{\mathsf{col}(i)}}{m}\cdot\frac{z_\mathbb{H}(\beta)}{\beta - \omega_{\mathsf{col}(i)}}\cdot (\beta - \omega_{\mathsf{col}(i)})\cdot \mu_i(X))  \pmod{z_\mathbb{H}(X)}  \\
&\equiv \frac{z_\mathbb{H}(\beta)}{m}(\sum_i \omega_{\mathsf{col}(i)}\cdot \mu_i(X)) \pmod{z_\mathbb{H}(X)}\\
&\equiv \frac{z_\mathbb{H}(\beta)}{m}\cdot v(X)  \pmod{z_\mathbb{H}(X)}
\end{split}
$$




### 6.3* Further Generalization 

本小节内容与 Baloo 无直接关联，但有助于理解 Generalized Univariate Sumcheck 定理的更一般化。初次阅读可以跳过本小节。

对于任意的 
$f(X)\in\mathbb{F}[X]$，$S\subset I$，下面的关系成立：

$$
f(X)N(X) = \sigma + X\cdot r(X) + z_I(X)\cdot q(X), \qquad \deg(r(X))<k-1
$$

这里 $\sigma=\sum_{s\in S}f(s)$，$N(X)$ 定义如下：

$$
N(X)=\sum_{s\in S}\frac{\tau_s(X)}{\tau_s(0)}
$$

下面我们来推导下这个定理。

$$
f(X)\cdot N(X) = (\sum_{i=0}^{k-1}f(h_i)\cdot \tau_i(X))\cdot (\sum_{s\in S}\frac{\tau_s(X)}{\tau_s(0)}) \equiv \sum_{s\in S}\Big(f(s)\cdot\frac{1}{\tau_s(0)}\cdot \tau_s(X)\Big) \pmod{z_I(X)}
$$

因此，$f(X)N(X)$ 的常数项恰好为 $\sigma=\sum_{s\in S}f(s)$。这个定理进一步扩展了 Univariate Sumcheck 可以对一个任意的 Domain $I$ 中的任意子集 $S$ 对应的值进行求和。

读者可能好奇，为何我们要关注 $\tau_i(0)$ 这个零点的因子。根据 [RZ21] 论文中的定理，这个 $X=0$ 点也可以扩展到任意的 $X=v$ 点。即：

$$
f(X)N_v(X) = \sigma + (X-v)\cdot r(X) + z_I(X)\cdot q(X), \qquad \deg(r(X))<k-1
$$

其中 

$$
N_v(X) = \sum_{s\in S}\frac{\tau_s(X)}{\tau_s(v)}
$$

因为

$$
f(X)N_v(X) = (\sum_{h\in I}f(h)\cdot \tau_h(X))\cdot (\sum_{s\in S}\frac{\tau_s(X)}{\tau_s(v)}) \equiv \sum_{s\in S}\Big(f(s)\cdot\frac{1}{\tau_s(v)}\cdot \tau_s(X)\Big) \pmod{z_I(X)}
$$

而多项式 $g(X) = \sum_{s\in S}\Big(f(s)\cdot\frac{1}{\tau_s(v)}\cdot \tau_s(X)\Big) $ 在 $X=v$ 点的取值为 $\sigma$，根据多项式余数定理，我们可得：

$$
g(X) - g(v) = g(X) - \sigma \equiv (X-v)\cdot r(X) \pmod{z_I(X)}
$$

## 7 协议框架

**公共输入**：

1. 子表格的承诺：  $[t_I(X)]_1$
2. 子表格 Domain $\mathbb{H}_I$ 上的 Vanishing Polynomial 承诺： $[z_I(X)]_2$

**第一步**： Prover 发送子表格索引的承诺：$[v(X)]_1$

$$
v(X) = \sum_{i=0}^{m-1}v_i\cdot \mu_i(X)
$$

**第二步**： Verifier 发送一个随机挑战数 $\alpha$ 给 Prover

**第三步**： Prover 发送矩阵的行空间抽样 $[M(\alpha, X)]_1$ 给 Verifier

$$
M(\alpha, X) =\sum_{i=0}^{m-1}\mu_i(\alpha)\cdot \hat\tau_{\mathsf{col}(i)}(X)
$$

**第四步**： Verifier 发送一个随机挑战数 $\beta$ 给 Prover

**第五步**： Prover 发送矩阵的列空间抽样 $[M(X, \beta)]_1$ 给 Verifier

$$
M(X, \beta) =\sum_{i=0}^{m-1}\hat\tau_{\mathsf{col}(i)}(\beta)\cdot \mu_i(X)
$$

**第六步**：Prover 证明 $M(X, \beta)$ 满足下面的关系式，

$$
M(X, \beta)\cdot (\beta - v(X)) + \frac{z_I(\beta)}{z_I(0)}\cdot v(X) = z_V(X) \cdot q_1(X)
$$

并发送下面的元素：

1. $[q_1(X)]_1$
2. $z_I(\beta)$
3. $z_I(0)$

**第七步**：Prover 证明 $M(\alpha, \beta)$ 分别是 $M(\alpha, X)$ 与 $M(X, \beta)$ 运算结果，并发送：

1. $d(\beta)=e(\alpha)$，
2. $\pi_{d(\beta)}$
3. $\pi_{e(\alpha)}$

**第八步**：Prover 证明 $\deg(M(X,\beta)) < m$

**验证**：Verifier 验证下面的等式：






