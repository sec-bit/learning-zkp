# Baloo, lookup  (Part 3)


> TODO: Prover 性能分析

> TODO: Proof size 与 Verifier 性能分析

### 9. Baloo 协议与优化

现在我们把 Subtable, Dot product, CP-expansion 三个子协议整合成一个完整的 Baloo 协议。

**公共输入**

1. 结构化的公共参考字符串 $SRS=\{[x^s]_1, [x^s]_2\}_{i=0}^{D}$
2. 主表格向量的多项式承诺 $T=[t(x)]_1$
3. 查询向量的多项式承诺 $A=[a(x)]_1$

**预计算**

1. $\{[q_i(X)]_1 \}_{i\in[N]}$，其中 $q_i(X)\cdot (X-\omega_i) = t(X) - t(\omega_i)$
2. $\{[w_i(X)]_1\}_{i\in[N]}$，其中 $w_i(X)\cdot (X-\omega_i) = z_\mathbb{H}(X)$

**第一步**

1. Prover 根据查询记录，计算子表格向量多项式 $t_I(X)$，并发送承诺 $[t_I(x)]_1$，

2. Prover 计算查询记录索引向量的插值多项式 $z_I(X)$，并发送承诺 $[z_I(x)]_1$ 与 $[z_I(x)]_2$

3. Prover 计算代表矩阵 $M$ 的向量 $\vec{v}$ 的多项式承诺 $[v(x)]_1$

**第二步** Verifier 发送随机数 $\alpha\in\mathbb{F}$

**第三步** Prover 发送 $a(\alpha)$，并附上 $[q_a(x)]_1$ 

$$
\mathsf{EQ0}: q_a(X)\cdot (X-\alpha) = a(X) - a(\alpha)
$$

**第四步** Prover 和 Verifier 通过 Subtable 子协议，证明 $\vec{t}_I\subset\vec{t}$

**第五步** Prover 计算矩阵 $M$ 的行空间抽样 $\vec{d}$，并发送多项式承诺 $[d(x)]_1$


**第六步** Prover 和 Verifier 通过 Dot-Product 子协议，证明 $\vec{d}\cdot \vec{t}_I=a(\alpha)$

1. Prover 计算 $q_1(X)$

$$
\mathsf{EQ1}: d(X)\cdot t_I(X) - a(\alpha) - r(X) = q_1(X)\cdot z_I(X)
$$


**第七步** Prover 和 Verifier 通过 CP-expansion 子协议，证明 $\vec{d}=\sum_{i=0}^{m-1}\mu_i(\alpha)\cdot \vec{M}_i$

1. Prover 计算 $e(X)$
2. Prover 计算 $q_2(X)$

$$
\mathsf{EQ2}: e(\alpha) = d(\beta)
$$

$$
\mathsf{EQ3}: e(X)(\beta - v(X)) + z_I(\beta)z_I(0)^{-1}\cdot v(X) = z_\mathbb{V}(X) \cdot q_2(X)
$$


**第八步** 

1. Verifier 验证 $[z_I(x)]_1$ 与 $[z_I(x)]_2$ 的一致性

$$
[z_I(x)]_1 \ast [1]_2 \overset{?}{=}[1]_1 \ast [z_I(x)]_2
$$

2. Verifier 验证 $a(\alpha)$ 的正确性：

$$
\big(A - a(\alpha)\cdot[1]_1\big)\ast [1]_2 \overset{?}{=}  [q_a(x)]_1 \ast \big([x]_2 - \alpha\cdot[1]_2\big)
$$

如果把三个子协议串联执行，那么证明尺寸会比较大。我们可以利用各种证明聚合的手段，把多个多项式 Evaluation 证明和 Degree-bound 证明合并起来，以减少证明尺寸。

### 9.1 多项式约束的线性化

当有一个二次以上的多项式约束等式时，我们可以充分利用多项式承诺的「加法同态性」，减少多项式求值的数量。比如在 CP-expansion 中有下面的等式关系：

$$
\mathsf{EQ3}:  e(X)(\beta - v(X)) + z_I(\beta)z_I(0)^{-1}\cdot v(X) = z_\mathbb{V}(X) \cdot q_2(X)
$$

为了验证上面等式成立，最直接粗暴的方式是把所有的多项式在一个随机点 $X=\zeta$ 处打开，然后验证多项式的取值满足上面的等式。但是这样的话，Prover 需要发送 $e(\zeta)$，$v(\zeta)$，$q_1(\zeta)$ 这三个多项式的值，这样的话证明尺寸会比较大。

我们并不需要打开等式中所有的多项式，事实上，我们只需要打开少量的多项式即可，然后把上述等式变成多项式的线性组合。例如，上述等式只需要打开 $e(\zeta)$，保留 $v(X)$ 与 $q_2(X)$，然后等式可以被线性化为：

$$
e(\zeta)\cdot(\beta - v(X)) + z_I(\beta)z_I(0)^{-1}\cdot v(X) = z_\mathbb{V}(\zeta) \cdot q_1(X)
$$

这里 $z_\mathbb{V}(\zeta)$ 由 Verifier 计算。这样，我们得到了一个新的多项式，记为 $p_1(X)$：

$$
p_1(X) = e(\zeta)\cdot(\beta - v(X))+ z_I(\beta)z_I(0)^{-1}\cdot v(X) - z_\mathbb{V}(\zeta) \cdot q_1(X)
$$

如果 $\mathsf{EQ3}$ 成立，那么 $p_1(X)$ 在 $X=\zeta$ 处等于 $0$。这样我们只需要再证明 $e(\zeta)$，$p_1(\zeta)=0$ 即可。这样原本需要打开三个多项式才能证明的等式，现在只需要打开两个多项式即可，并且由于 $p_1(\zeta)$ 为零，因此这个值并不需要 Prover 发送。

第二个可以线性化的多项式约束为 

$$
\mathsf{EQ1}: d(X)\cdot t_I(X) - a(\alpha) - r(X) = q_2(X)\cdot z_I(X)
$$

这里我们可以复用 $X=\beta$ 作为挑战点检验，因为我们可以让 Prover 在 Verifier 发送 $\beta$ 之前发送 $[d(x)]_1$，$[r(x)]_1$，与 $[q_2(x)]_1$。Prover 只需要发送 $d(\beta)$ 与 $z_I(\beta)$，这样线性化之后的多项式记为 $p_2(X)$。Prover 只需要证明 $d(\beta)$, $z_I(\beta)$，$p_2(\beta)=0$ 即可。

$$
p_2(X) = d(\beta)\cdot t_I(X) - a(\alpha) - r(X) - z_I(\beta) \cdot q_2(X) 
$$


### 9.2 多项式求值的聚合

在约束等式 $\mathsf{EQ0}$ 和 $\mathsf{EQ2}$ 中都需要打开 $a(\alpha)$ 处的求值和 $e(\alpha)$ 处的求值，因此我们可以把 $a(X)$ 和 $e(X)$ 在 $X=\alpha $ 处的求值证明聚合在一起。

在 CP-expansion 子协议中，在证明 Generalized Sumcheck 等式时（$\mathsf{EQ1}$），我们可以利用随机数 $\beta$ 作为求值点，而非是 $\zeta$，这样可以避免 $z_I(X)$ 在 $X=\beta$ 和 $X=\zeta$ 两处同时打开。

### 9.3 Degree-bound 证明的合并

第一个 degree bound 证明是关于等式 $\mathsf{EQ1}$ 中的余数多项式 $r(X)=Xg(X)$，其中 $g(X)$ 的 degree 应该严格小于 $m-1$。这个证明其实可以转化为两个证明子目标：

1. $r(0)=0$
2. $\deg(r(X))<m$

注意这里并不是 Univariate Sumcheck 通常的做法。通常的做法是证明 $\deg(g(X)) < m-1$。这里采用 $\deg(X\cdot g(X)) < m$ 的方案是这样可以把 $r(0)=0$ 这个约束和其它在 $X=0$ 处打开的多项式求值约束（如 $z_I(X)$）聚合在一起。同时 $r(X)$ 和 $z_I(X)$ 都需要 degree bound 证明，因此也正好可以聚合在一个约束中。
 
第二个子目标可以和第二个 degree-bound 证明 $\deg(z_I(X) - X^m)<m$ 合并在一起。

第三个 degree-bound 证明是 $\deg(e(X)) < m$

### 9.4 优化后的协议细节

下面是优化后的完整协议以及注解：

**公共输入**：

1. 大表格的承诺 $C_T=[t(x)]_1$， 
2. 查询向量的承诺 $C_a=[a(x)]_1$
3. $\mathbb{H}$ 上的 Vanishing 多项式的承诺 $C_{z_\mathbb{H}}=[z_\mathbb{H}(x)]_1$

**第一步**：Prover 计算并发送下面三个多项式的承诺：$\big(C_v = [v(x)]_1, C_{z_I}=[z_I(x)]_2, C_t=[t_I(x)]_1\big)$

1. 计算并承诺 $\mathbb{H}_I$ 上的 Vanishing 多项式 $z_I(X)$ 
2. 计算并承诺 $v(X)=\sum_{i=0}^{m-1}\omega_{\mathsf{col}(i)}\cdot \mu_i(X)$
3. 计算并承诺子表格向量的多项式 $t_I(X)$

**第二步**：Verifier 发送一个随机挑战数 $\alpha$ ，用于矩阵 $M$ 的行空间随机抽样

**第三步**：Prover 计算并发送 $\big(C_d = [d(X)]_1, C_r=[r(X)]_1, C_{q_1} = [q_1(X)]_1 \big)$

1. 计算并承诺 $d(X)=M(\alpha, X)=\sum_{i=0}^{m-1}\mu_i(\alpha)\cdot \hat\tau_{\mathrm{col}(i)}(X)$
2. 计算并承诺 $r(X)$ 与 $q_1(X)$，满足下面的关系：

$$
\mathsf{EQ1}: d(X)\cdot t_I(X) - a(\alpha) - r(X) = q_1(X)\cdot z_I(X)
$$

3. TODO: 计算 $r(X)$ 还要满足 $\deg(r(X)) < m$ 并且 $r(0)=0$

**第四步**：Verifier 发送一个随机挑战数 $\beta$ 给 Prover，用于矩阵 $M$ 的列空间随机抽样

**第五步**：Prover 计算并发送 $\big(C_e = [e(x)]_1, C_{q_2}=[q_2(x)]_1 \big)$

1. $e(X)=M(X, \beta)=\sum_{i=0}^{m-1}\mu_i(X)\cdot \hat\tau_{\mathrm{col}(i)}(\beta)$，满足下面的等式关系：

$$
\mathsf{EQ2}: e(\alpha) = d(\beta)
$$

2. $q_2(X)$ 满足下面的关系：

$$
\mathsf{EQ3}: e(X)(\beta - v(X)) + z_I(\beta)z_I(0)^{-1}\cdot v(X) = z_\mathbb{V}(X) \cdot q_2(X)
$$

**第六步**：Verifier 发送一个随机挑战数 $\zeta$ 给 Prover，用于挑战 $\mathsf{EQ3}$ 等式的正确性

**第七步**：

1. Prover 计算并发送 $\big(v_1=e(\alpha),v_2=a(\alpha),v_3=z_I(0),v_4=z_I(\beta),v_5=e(\zeta))$

2. Prover 计算线性化多项式 $p_1(X)$, 它在 $X=\beta$ 处等于零

$$
p_1(X) = d(\beta)\cdot t_I(X) - a(\alpha) - r(X) - z_I(\beta)\cdot q_1(X) 
$$

3. Prover 计算线性化多项式 $p_2(X)$，它在 $X=\zeta$ 处等于零

$$
p_2(X) = e(\zeta)(\beta -v(X)) + z_I(\beta)z_I(0)^{-1}\cdot v(X) - z_\mathbb{V}(\zeta) \cdot q_2(X)
$$

**第八步**：Verifier 发送一个随机挑战数 $\gamma$ 给 Prover，用于聚合证明。

**第九步**：Prover 完成下列的计算并发送证明

1. 聚合 $v_1=e(\alpha)$，$v_2=a(\alpha)$ 两个在 $X=\alpha$ 处打开的 Evaluation 证明，还巧妙地聚合了 $\deg(e(X))<m$ 这个 Degree bound 证明。这里当然也约束了 $a(X)$ 的 degree，不过这并不是协议需要的约束。

$$
w_1(X) = X^{D-m+1}\cdot \left(\frac{e(X) - v_1}{X-\alpha} + \gamma \cdot \frac{a(X)-v_2}{X-\alpha}\right)
$$

2. 聚合 $v_3=z_I(0)$, $r(0)=0$ 两个 evaluation proofs，还有 $\deg(r(X))<m$ 和 $\deg(z_I(X) - X^m)<m$ 这两个 degree bound 证明。后者约束了子表格的大小严格等于 $m$。

$$
w_2(X) = \frac{z_I(X) - v_3}{X} + \gamma \cdot \frac{r(X)-0}{X} + X^{D-m+1}\cdot \left(\gamma^2\cdot (z_I(X) - X^m) + \gamma^3 \cdot r(X)\right)
$$

3. 聚合 $v_1 = d(\beta)$ , $v_4=z_I(\beta)$ 和 $p_1(\beta)=0$ 三个 Evaluation 证明

$$
w_3(X) = \frac{d(X) - v_1}{X-\beta} + \gamma\cdot \frac{z_I(X) - v_4}{X-\beta} + \gamma^2 \cdot \frac{p_1(X)-0}{X-\beta}
$$

4. 聚合 $e(\zeta)=v_5$, $p_2(\zeta)=0$ 两个 evaluation 证明

$$
w_4(X) = \frac{e(X) - v_5}{X-\zeta} + \gamma \cdot \frac{p_2(X)-0}{X-\zeta}
$$

5. Prover 发送 $\big(C_{w_1}=[w_1(x)]_1, C_{w_2}=[w_2(x)]_1, C_{w_3}=[w_3(x)]_1, C_{w_4}=[w_4(x)]_1 \big)$

**验证**：Verifier 收到的证明如下：

$$
\pi = \left(
\begin{array}{l}
C_v = [v(x)]_1, {\color{red}C_{z_I}=[z_I(x)]_2}, C_t=[t_I(x)]_1, \\[1ex]
C_d = [d(X)]_1, C_r=[r(X)]_1, C_{q_1} = [q_1(X)]_1, \\[1ex]
C_e = [e(x)]_1, C_{q_2}=[q_2(x)]_1  \\[1ex]
 C_{w_1}=[w_1(x)]_1, C_{w_2}=[w_2(x)]_1, C_{w_3}=[w_3(x)]_1, C_{w_4}=[w_4(x)]_1 \\[1ex]
 v_1=e(\alpha),v_2=a(\alpha),v_3=z_I(0),v_4=z_I(\beta),v_5=e(\zeta)\\
 \end{array}
 \right)
$$

1. 计算线性化多项式承诺 $C_{p_1} = [p_1(x)]_1$

$$
C_{p_1} = v_1\cdot C_t - v_2\cdot [1]_1 - C_r - v_4\cdot C_{q_1}
$$

2. 计算线性化多项式承诺 $C_{p_2} = [p_2(x)]_1$

$$
C_{p_2} = v_5\cdot(\beta\cdot[1]_1 - C_v) + v_4v_3^{-1}\cdot C_v - z_\mathbb{V}(\zeta)\cdot C_{q_2}
$$

Verifier 验证下面的 Pairing 等式：

1. 验证 Subtable 等式

$$
\Big(C_T - C_t + \gamma\cdot C_{z_\mathbb{H}}\Big)\ast [1]_2\overset{?}{=}
\Big(C_{w_5} + \gamma\cdot C_{w_6}\Big) \ast {\color{red}C_{z_I}}
$$

2. 验证打开点在 $X=\alpha$ 处的证明 $\blue{C_{w_1}}$ ，根据化简的多项式关系：

$$
\begin{split}
w_1(X)\cdot X-\alpha\cdot w_1(X) &= X^{D-m+1}\cdot \left(e(X) + \gamma a(X)- (v_1 + \gamma v_2\right))\\
\end{split}
$$

验证等式为：

$$
\Big(\blue{C_{w_1}} \ast [X]_2\Big) \overset{?}{=} \Big([e(X)]_1 + \gamma\cdot[a(X)]_1 - (v_1 + \gamma v_2)\cdot[1]_1\Big)\ast [X^{D-m+1}]_2 + \Big(\alpha\cdot\blue{C_{w_1}}\ast [1]_2 \Big)\ast [1]_2
$$

3. 验证打开点在 $X=0$ 处的证明 $\blue{C_{w_2}}$ ，根据

$$
\begin{split}
w_2(X)\cdot X &= (z_I(X) - v_3) + \gamma \cdot r(X) + X^{D-m+2}\cdot \left(\gamma^2\cdot (z_I(X) - X^m) + \gamma^3 \cdot r(X)\right) \\
&= z_I(X) + X^{D-m+2}\cdot\gamma^2 z_I(X) + \gamma r(X) + X^{D-m+2}\cdot \gamma^3 \cdot r(X) - v_3 - X^{D-m+2}\cdot \gamma^2 \cdot X^m \\
&= (1+\gamma^2\cdot X^{D-m+2})\cdot z_I(X) + (\gamma^3\cdot r(X) - \gamma^2X^m)\cdot X^{D-m+2} + (\gamma\cdot r(X) - v_3) \\
\end{split}
$$

得到验证等式：

$$
\Big(\blue{C_{w_2}} \ast [X]_2\Big) \overset{?}{=} 
\Big([1]_1 +\gamma^2\cdot [X^{D-m+2}]_1\Big)\ast \red{C_{z_I}} + \Big(\gamma^3\cdot C_r - \gamma^2\cdot [X^m]_1\Big)\ast [X^{D-m+2}]_2 + \Big(\gamma\cdot C_r - [v_3]_1\Big)\ast[1]_2
$$

4. 验证打开点为 $X=\beta$ 的证明 $\blue{C_{w_3}}$ ，根据：

$$
\begin{split}
w_3(X)\cdot X - \beta\cdot w_3(X) & = (d(X) - v_1) + \gamma\cdot (z_I(X) - v_4) + \gamma^2 \cdot p_1(X) \\
w_3(X)\cdot X & = \Big(d(X) + \beta\cdot w_3(X) - \gamma v_4 - v_1 + \gamma^2 \cdot p_1(X)\Big)  + \gamma \cdot z_I(X) \\
\end{split}
$$

得到验证等式：

$$
\Big(\blue{C_{w_3}} \ast [X]_2\Big) \overset{?}{=} 
\Big(C_d + \beta\cdot \blue{C_{w_3}} - \gamma\cdot [v_4]_1 - [v_1]_1 + \gamma^2 \cdot C_{p_1}\Big) \ast [1]_2 + \Big(\gamma\cdot [1]_1\Big) \ast \red{C_{z_I}}
$$

请注意这里 $[z_I(X)]_2$ 承诺在 $\mathbb{G}_2$ 上，所以要单独拎出来计算 Pairing。

5. 验证打开点在 $X=\zeta$ 的聚合证明 $\blue{C_{w_4}}$，根据下面公式推导：

$$
\begin{split}
X\cdot w_4(X)-\zeta\cdot w_4(X) &= (e(X) - v_5) + \gamma \cdot p_2(X)\\
w_4(X)\cdot X & = e(X) + \zeta\cdot w_4(X) + \gamma \cdot p_2(X) - v_5 \\
\end{split}
$$

得到验证等式：

$$
\Big(\blue{C_{w_4}}\ast [X]_2\Big) \overset{?}{=} 
\Big(C_e + \zeta\cdot \blue{C_{w_4}} + \gamma \cdot C_{p_2} - v_5\cdot [1]_1\Big)\ast [1]_2 \\
$$

### 9.5 合并 Pairing 验证

### 9.6 性能分析

证明尺寸：$11\mathbb{G}_1 + 1\mathbb{G}_2 + 5\mathbb{F}$ 


TODO
