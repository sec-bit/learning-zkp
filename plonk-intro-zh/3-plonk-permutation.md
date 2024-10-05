# 理解 PLONK（三）：置换证明

Plonkish 电路编码用两个矩阵 $(Q,\sigma)$ 描述电路的空白结构，其中 $Q$ 为运算开关， $\sigma$ 为置换关系，用来约束 $W$ 矩阵中的某些位置必须被填入相等的值。本文重点讲解置换证明（Permutation Argument）的原理。

## 回顾拷贝关系

回顾一下 Plonkish 的 $W$ 表格，总共有三列，行数按照 $2^2$ 对齐。

$$
\begin{array}{c|c|c|c|}
i & w_{a,i} & w_{b,i} & w_{c,i}  \\
\hline
1 & {\color{red}x_6} & {\color{blue}x_5} & {\color{green}out} \\
2 & x_1 & x_2 & {\color{red}x_6} \\
3 & x_3 & x_4 & {\color{blue}x_5} \\
4 & 0 & 0 & {\color{green}out} \\
\end{array}
$$

我们想约束 Prover 在填写 $W$ 表时，满足下面的拷贝关系： $w_{a,1}=w_{c,2}$   $w_{b,1}=w_{c,3}$ 与 $w_{c,1}=w_{c,4}$，换句话说， $w_{a,1}$ 位置上的值需要被拷贝到 $w_{c,2}$ 处，而 $w_{b,1}$ 位置上的值需要被拷贝到 $w_{c,3}$ 处， $w_{c,1}$ 位置上的值被拷贝到 $w_{c,4}$ 处。

问题的挑战在于，Verifier 要在看不到 $W$ 表格的情况下，仅通过一次随机挑战就能完成 $W$ 表格中多个拷贝关系的证明。

Plonk 的「拷贝约束」是通过「置换证明」（Permutation Argument）来实现，即把表格中需要约束相等的那些值进行「循环换位」，然后证明「循环换位」后的表格和原来的表格完全相等。


<!-- 简化一下问题：如何证明两个等长向量 $\vec{a}$ 和 $\vec{a}'$ 满足一个已知的置换 $\sigma$，并且置换前后的两个 $\vec{a}=\vec{a}'$

$$
a_i=a'_{\sigma(i)}
$$ -->

举一个简单的例子来说明：假设 $\vec{a}=(a_0,a_1,a_2,a_3)$， 然后将 $\vec{a}$ 每一个元素都进行向左移位，我们得到向量 $\vec{a}_{\circlearrowleft}=(a_1,a_2,a_3,a_0)$ ，这里我们用记号 $\circlearrowleft$ 表示向「左循环移位」，循环左移也可以用一个置换函数 $\sigma$ 来表示：

$$
\sigma=\{0\to 1; 1\to 2; 2\to 3; 3\to0\}
$$

如果我们能证明 $\vec{a}=\vec{a}_\circlearrowleft$ ，那么我们就能得到结论，原向量 $\vec{a}$ 中的所有元素都相等，我们可以通过下面的表格来说明：

$$
\begin{array}{c|c|c|c|c|c}
\vec{a} & a_0 & a_1 & a_2 & a_3 \\
\hline
\vec{a}_\circlearrowleft & a_1 & a_2 & a_3 & a_0 \\
\end{array}
$$

观察上面的表格中，如果上下两个向量 $\vec{a}$ 与 $\vec{a}_\circlearrowleft$ 相等，那么意味着 $a_0=a_1$， $a_1=a_2$， $a_2=a_3$， $a_3=a_0$，进一步我们可以得出 $a_0=a_1=a_2=a_3$，即等价于结论： $\vec{a}$ 中的全部元素都相等。

对于电路赋值表格 $W$ ，我们只需要针对那些需要相等的位置进行「局部地」循环换位，然后让 Prover 证明 $W$ 和经过循环换位后的 $W'$ 表格相等，那么可就可以实现拷贝约束。再举个一个例子，有一个向量 $\vec{w} = (x_0,\ldots, x_5)$ ，我们想证明其中前两个元素相等 $x_0=x_1$ ，后三个元素相等 $x_3=x_4=x_5$，那么我们可以演示 $\vec{w}$ 经过一个内部置换后的向量，记为 $\vec{w}_{\sigma}$ ，仍与原向量相等来证明，下面表格中的蓝色部分表示前两个元素的循环左移，而红色部分表示后三个元素的循环左移：

$$
\begin{array}{c|cc}
\vec{w} =& {\color{blue}x_0} & {\color{blue}x_1} & x_2 & {\color{red}x_3} & {\color{red}x_4} & {\color{red} x_5} \\
\vec{w}_{\sigma} =& {\color{blue}x_1} & {\color{blue}x_0} & x_2 & {\color{red} x_4} & {\color{red} x_3} & {\color{red} x_5} \\ 
\end{array}
$$

而证明两个向量 $`\vec{w}=\vec{w}_{\sigma}`$ ，这个可以通过先多项式编码，然后概率检验的方式完成。剩下的工作就是如何让 Prover 证明 $\vec{w}$  确实是（诚实地）按照事先约定的方式进行循环移位，并且移位后得到了 $`\vec{w}_\sigma`$。

那么接下来就是理解如何让 Prover 证明两个向量之间满足某一个「置换关系」。 置换证明（Permutation Argument）是 Plonk 协议中的核心部分，为了解释它的工作原理，我们先从一个基础协议开始——连乘证明（Grand Product Argument）。

## 冷启动：Grand Product 

证明单个乘法 $v_0\cdot v_1=p$ 比较简单，它是 Hadamard 乘积证明的弱化形式。而要证明下面的「连乘关系」 则并不简单：

$$
p = v_0\cdot v_1 \cdot v_2 \cdot \cdots \cdot v_{N-2}
$$

对付连乘的基本思路是：让 Prover 利用一组单乘的证明来实现多个数的连乘证明，然后再通过多项式的编码，交给 Verifier 进行概率检查。

强调下：证明思路的关键点是如何把一个连乘计算转换成多次的单乘计算。

我们需要通过引入一个「辅助向量」，记为 $\vec{z}=(z_0,z_1,\ldots,z_{N-1})$，把「连乘」的计算看成是一步步的单乘计算，然后辅助向量表示每次单乘之后的「中间值」：

$$
\begin{array}{c|c|l}
v_i & z_i & \ \ v_i\cdot z_i \\
\hline
v_0 & z_0=1  & z_1=v_0\\
v_1 & z_1 & z_2=v_0\cdot v_1\\
v_2 & z_2 & z_3=v_0\cdot v_1\cdot v_2\\
\vdots & \vdots & \vdots\\
v_{N-3} & z_{N-3} & z_{N-2} = v_0\cdot v_1\cdot v_2\cdots v_{N-3}\\
v_{N-2} & z_{N-2} & z_{N-1} = p\\
\end{array}
$$

上面表格表述了连乘过程的计算轨迹（Trace），每一行代表一次单乘，顺序从上往下计算，最后一行计算出最终的结果 $p$ ，而 $(z_0, \ldots, z_{N-2})$ 的角色是充当计算的中间结果。

表格的最左列为要进行成绩计算的 $v_i$，中间列 $z_i$ 为引入的辅助中间变量，记录每次「单乘之前」的中间值，最右列表示每次「单乘之后」的中间值。

不难发现，表格 「中间列」变量 $z_{i+1}$ 向上挪一行与「最右列」$`v_i\cdot z_i`$ 几乎一致，除了中间列的第一个元素 $z_0$ 与最右列的最后一个元素 $z_{n-1}$ 。其中 $z_0$ 元素等于常数 $1$ 作为计算初始值，「最右列」最后一个向量 $z_{n-1}$ 元素记录了最终的计算结果。

这里向量 $\vec{z}$ 被称为 Accumulator，即记录连乘计算过程中的每一个中间结果：

$$
z_k = \prod_{i=0}^{k-1}v_i
$$

我们总结处下面的递推式：

$$
z_0 = 1, \qquad z_{k+1}=v_{k}\cdot z_{k}
$$

我们接下来对 $\vec{v}$ 和 $\vec{z}$ 在 $H$ 上进行多项式编码：

$$
\begin{array}{c|c|c}
H & v_i & z_i &  \\
\hline
\omega^0 & v_0 & z_0=1  \\
\omega^1 & v_1 & z_1 \\
\omega^2 &v_2 & z_2 \\
\vdots & \vdots & \vdots\\
\omega^{N-2} & v_{N-2} & z_{N-2} \\
\omega^{N-1} & 0 & z_{N-1} = p \\
\end{array}
$$

我们用多项式 $v(X)$ 和 $z(X)$ 来编码 $\vec{v}$ 和 $\vec{z}$ 。因为表格的第一个行中，累乘变量 $z_0=1$，因此多项式 $z(X)$ 满足下面第一个多项式约束：

$$
L_0(X)\cdot(z(X)-1)=0, \qquad \forall X\in H 
$$

第二个多项式约束要求表格从第一行到倒数第二行，满足 $\vec{z}$ 的递推关系 $v_i\cdot z_{i}=z_{i+1}$， 
 
$$
v(X)\cdot r(X) = r(\omega\cdot X), \qquad \forall X\in H\backslash\\{\omega^{-1}\\}
$$

注意上面等式中，未知数的取值范围为 $`H\setminus\{\omega^{-1}\}`$， 这是因为 $\omega^{-1}=\omega^{N-1}$，对应于表格的最后一行。由于最后一行没有乘法，只保存连乘的结果 $z_{n-1}=p$，因此约束要去除最后一行。而最后一行需要通过下面的第三个多项式约束：

$$
L_{N-1}(X)\cdot(z(X)-p)=0, \qquad \forall X\in H
$$

如何处理上面第二个多项式约束不能覆盖整个 $H$ 的情况（要去除 $\omega^{-1}$ 这一行）？我们可以将其改写为下面的约束等式，从而让多项式约束的范围重新覆盖整个 $H$ ：

$$
\big(v(X)\cdot r(X) - r(\omega\cdot X)\big)\cdot \big(X-\omega^{-1}\big)=0, \qquad \forall X\in H
$$

我们还可以用一个小技巧来将上面的三个多项式约束化简，并合并为一个多项式约束。

先把计算连乘的表格添加一行，令 $v_{n-1}=\frac{1}{p}$（请记住： $p$ 为 $\vec{v}$ 向量的连乘积）

$$
\begin{array}{c|c|c}
v_i & z_i & v_i\cdot z_i \\
\hline
v_0 & z_0=1  & z_1\\
v_1 & z_1 & z_2\\
v_2 & z_2 & z_3\\
\vdots & \vdots & \vdots\\
v_{N-2} & z_{N-2} & z_{N-1}\\
{\color{blue}v_{N-1}=\frac{1}{p}} & z_{N-1}=p & z_0=1 \\
\end{array}
$$

这样一来， $v_{N-1}\cdot z_{N-1}=z_0=1$ ，表格的最后一行也满足单次乘法的关系。而且最右列恰好是 $\vec{z}$ 的循环移位。由于表格的每一行都满足单次乘法关系，因此我们可以用下面的多项式约束来表示递归的连乘：

$$
v(X)\cdot z(X)=z(\omega\cdot X), \qquad \forall X\in H
$$

这样上面的第三个多项式约束也不再需要了，而且第二个多项式约束也得到了简化。再通过 Verifier 提供一个随机挑战数 $\alpha$，我们只需要定义一个多项式约束就可以表达上面的连乘关系：

$$
L_0(X)\cdot(z(X)-1)+\alpha\cdot(v(X)\cdot z(X)-z(\omega\cdot X))=0, \qquad \forall X\in H
$$

接下来，Verifier 可以在整个有限域上挑战下面的多项式：

$$
L_0(X)\cdot(z(X)-1)+\alpha\cdot(v(X)\cdot z(X)-z(\omega\cdot X)) = h(X)\cdot z_H(X)
$$

其中 $h(X)$ 为商多项式，和 $z_H(X)$ 为 $H$ 上的 Vanishing Polynomial，即 $z_H(X)=(X-1)(X-\omega)\cdots(X-\omega^{N-1})$。（温馨提示：这里我们用符号 $z_H$ 表示 Vanishing Polynomial，而同时使用 $z(X)$ 表示累乘向量的多项式，请注意区分。）

接下来，通过 Schwartz-Zippel 定理，Verifier 可以给出挑战数 $\zeta$ 来验证上述多项式等式是否成立（具体过程略）。

到此为止，如果我们已经理解了如何证明一个向量元素的连乘，那么接下来的问题是如何利用「连乘证明」来实现「Multiset 等价证明」（Multiset Equality Argument）。

## 从 Grand Product 到 Multiset 等价

假设有两个向量，其中一个向量是另一个向量的乱序重排，那么如何证明它们在集合意义（注意：集合无序）上的等价呢？我们不能简单地通过证明两个向量所编码的多项式相等来判断，因为向量元素的顺序不同，所得到的多项式也不同。即使两个多项式 $a(X)\neq b(X)$，它们也可能表示同一个集合。

最容易想到的方案是依次枚举其中一个向量中的每个元素，并证明该元素属于另一个向量。但这个方法有个限制，就是无法处理向量中会出现两个相同元素的情况，也即不支持「多重集合」（Multiset）的判等。例如 $\\{1,1,2\\}$ 就是一个多重集合（Multiset），那么它显然不等于 $\\{1, 2, 2\\}$，也不等于 $\\{2,1\\}$。

一个直接处理多重集合的方案是将两个向量中的所有元素都连乘起来，然后判断两个向量的连乘值是否相等。但这个方案同样有一个严重的限制，就是向量元素必须都为素数，很容易给出一个反例： $3\cdot6=9\cdot2$ ，但 $\\{3,6\\}\neq\\{9,2\\}$。

修改下这个方案，我们假设向量 $\\{q_i\\}$  为一个多项式 $q(X)$ 的根集合，即对向量中的任何一个元素 $q_i$，都满足  $q(q_i)=0$。这个多项式可以定义为：

$$
q(X) = (X-q_0)(X-q_1)(X-q_2)\cdots (X-q_{n-1})
$$

如果存在另一个多项式 $p(X)$ 等于 $q(X)$，那么它们一定具有相同的根集合 $\\{q_i\\}$。比如

$$
\prod_{i}(X - q_i) = q(X) = p(X) = \prod_{i}(X - p_i)
$$

那么这两个向量在 Multiset 的意义上等价，即

$$
\\{q_i\\}=_{multiset}\\{p_i\\}
$$

我们可以利用 Schwartz-Zippel 定理来进一步地检验：向 Verifier 索要一个随机数 $\gamma$，那么 Prover 就可以通过下面的等式证明两个向量 $\\{p_i\\}$ 与 $\\{q_i\\}$ 在多重集合意义上等价：

$$
\prod_{{i\in[n]}}(\gamma-p_i)=\prod_{i\in[n]}(\gamma-q_i)
$$

还没结束，我们需要用上一节的连乘证明方案来继续完成验证，即通过构造辅助向量（作为一个累积器），把连乘转换成多个单乘来完成证明。需要注意的是，这里的两个连乘可以合并为一个连乘，即上面的连乘相等可以转换为

$$
\prod_{{i\in[n]}}\frac{(\gamma-p_i)}{(\gamma-q_i)}=1
$$

到这里，我们已经理解如何证明「Multiset 等价」，下一步我们将完成构造「置换证明」（Permutation Argument），用来实现 Plonk 协议所需的「Copy Constraints」。

## 从 Multiset 等价到置换证明

Multiset 等价可以被看作是一类特殊的置换证明。即两个向量 $`\{p_i\}`$ 和 $`\{q_i\}`$存在一个「未知」的置换关系。

而我们需要的是一个支持「已知」的特定置换关系的证明和验证。也就是对一个有序的向量进行一个「公开特定的重新排列」，即对需要证明等价的每个子集分别进行局部循环移位的置换。

先简化下问题，假如我们想让 Prover 证明两个向量满足一个奇偶位互换的置换：

$$
\begin{array}{rcl}
\vec{a} &=& (a_0, a_1, a_2, a_3,\ldots, a_{n-1}, a_n) \\
\vec{b} &=& (a_1, a_0, a_3, a_2, \ldots, a_n, a_{n-1})\\
\end{array}
$$

我们仍然采用「多项式编码」的方式把上面两个向量编码为两个多项式， $a(X)$ 与 $b(X)$。思考一下，我们可以用下面的「位置向量」来表示「奇偶互换」：

$$
\vec{i}=(0, 1, 2, 3, \ldots, n-1, n),\quad \sigma = (1, 0, 3, 2,\ldots, n, n-1)
$$

进一步把这个位置向量和 $\vec{a}$ 与 $\vec{b}$ 并排放在一起：

$$
\begin{array}{|c|c | c|c|}
a_i & {i} & b_i & \sigma({i}) \\
\hline
a_0 & 0 & b_0=a_1 & 1 \\
a_1 & 1 & b_1=a_0 & 0 \\
a_2 & 2 & b_2=a_3 & 3 \\
a_3 & 3 & b_3=a_2 & 2 \\
\vdots & \vdots & \vdots & \vdots \\
a_{n-1} & n-1 & b_{n-1}=a_n & n \\
a_n & n & b_n=a_{n-1} & n-1 \\
\end{array}
$$

接下来，我们要把上表的左边两列，还有右边两列分别「折叠」在一起。换句话说，我们把 $(a_i, i)$ 视为一个元素，把 $(b_i, \sigma(i))$ 视为一个元素，这样上面表格就变成了：

<!-- hack for buggy mathjax on github -->

$$
\begin{array}{|c|c|}
a'_i=(a_i, i) & b'_i=({b}_i, \sigma(i)) \\
\hline
(a_0, 0) & (b_0=a_1, 1) \\
(a_1, 1) & (b_1=a_0, 0) \\
\vdots & \vdots \\
(a\_{n-1}, n-1) & (b\_{n-1}=a\_{n}, n) \\
(a\_n, n) & (b\_n=a\_{n-1}, n-1) \\
\end{array}
$$

容易看出，如果两个向量 $\vec{a}$ 与 $\vec{b}$ 满足 $\sigma$ 置换，那么，合并后的两个向量 $\vec{a}'$ 和 $\vec{b}'$  将满足 Multiset 等价关系。

也就是说，通过把向量值和位置值合并，就能把一个「置换证明」转换成一个「多重集合等价证明」，即不用再针对某个特定的「置换关系」进行证明。

这里又出现一个问题，表格的左右两列中的元素为二元组（Pair），二元组无法作为一个「一元多项式」的根集合。

我们再使用一个技巧：再向 Verifier 索取一个随机数 $\beta$，把一个元组「折叠」成一个值：

$$
\begin{array}{|c|c|}
a'_i=(a_i+\beta\cdot i) & b_i'=(b + \beta\cdot \sigma(i)) \\
\hline
(a_0 + \beta\cdot 0) & (b_0 + \beta\cdot 1) \\
(a_1 + \beta\cdot 1) & (b_1 + \beta\cdot 0) \\
\vdots & \vdots \\
(a\_{n-1} + \beta\cdot n-1) & (b\_{n-1} + \beta\cdot n) \\
(a\_n + \beta\cdot n) & (b\_n + \beta\cdot (n-1))\\
\end{array}
$$

接下来，Prover 可以对 $\vec{a}'$ 与 $\vec{b}'$ 两个向量进行 Multiset 等价证明，从而可以证明它们的置换关系。

## 完整的置换协议

假设素数域 $\mathbb{F}_p$ 有一个乘法子群 $H=(1, \omega, \omega^2, \ldots, \omega^{N-1})$，其中 $\omega$ 为 $H$ 的生成元。

公共输入：置换关系 $\sigma$

秘密输入：两个长度为 $N$ 的向量 $\vec{a}$ 与 $\vec{b}$ 

预处理：Prover 和 Verifier 构造 $[id(X)]$ 与 $[\sigma(X)]$，其中 $id(X)$ 为 $(0, 1, 2, \ldots, N-1)$ 的多项式编码， $\sigma(X)$ 为 $(\sigma(0), \sigma(1), \ldots, \sigma(N-1))$ 置换向量的多项式编码。

第一步：Prover 构造并发送 $[a(X)]$ 与 $[b(X)]$

第二步：Verifier 发送随机挑战数 $\beta\leftarrow\mathbb{F}_p$ 与 $\gamma\leftarrow\mathbb{F}_p$

第三步：Prover 构造辅助的累乘向量 $\vec{z}$，构造多项式 $z(X)$ 并发送 $[z(X)]$

累乘向量 $\vec{z}$ 的构造方式如下：

$$
\begin{split}
z_0 &= 1 \\
z_{i+1} &= \prod_{i=0}^{N-1} \frac{a_i+\beta\cdot i + \gamma}{b_i+\beta\cdot \sigma(i) + \gamma}
\end{split}
$$

多项式 $z(X)$ 满足两个约束等式：

$$
L_0(X)\cdot(z(X)-1)=0, \qquad \forall X\in H 
$$

$$
\frac{z(\omega\cdot X)}{z(X)} = \frac{a(X)+\beta\cdot id(X) + \gamma}{b(X)+\beta\cdot \sigma(X) + \gamma}, \qquad \forall X\in H 
$$

第四步：Verifier 发送随机挑战数 $\alpha\leftarrow\mathbb{F}_p$，用来合并 $z(X)$ 的两个约束等式

第五步：Prover 构造合并后的约束多项式 $f(X)$ 与 商多项式 $h(X)$，并发送 $[h(X)]$

$$
f(X)= L_0(X)(z(X)-1) + \alpha\cdot \big(z(\omega\cdot X)(b(X)+\beta\cdot\sigma(X)+\gamma)-z(X)(a(X)+\beta\cdot id(X)+\gamma)\big) 
$$

商多项式 $h(X)$ 计算如下：

$$
h(X) = \frac{f(X)}{z_H(X)}
$$

第六步：Verifier 完成下面的查询：

- 向 $[a(X)],[b(X)],[h(X)]$ 查询这三个多项式在 $X=\zeta$ 处的取值 ，得到 $a(\zeta)$， $b(\zeta)$， $h(\zeta)$；
- 向 $[z(X)]$ 查询 $X=\zeta, X=\omega\cdot\zeta$ 两个位置处的取值，得到  $z(\zeta)$ 与  $z(\omega\cdot\zeta)$；
- 向  $[\sigma(X)]$ 与 $[id(X)]$ 这两个多项式发送求值查询 $X=\zeta$ ，得到  $id(\zeta)$ 与 $\sigma(\zeta)$；
- Verifier 自行计算 Vanishing Polynomial 在 $X=\zeta$ 处的取值 $z_H(\zeta)$，与 Lagrange Polynomial $L_0(X)$ 在  $X=\zeta$ 处的取值 $L_0(\zeta)$

验证步：Verifier 根据查询的多项式取值，验证下面的约束等式：

$$
L_0(\zeta)(z(\zeta)-1) + \alpha\cdot (z(\omega\cdot \zeta)(b(\zeta)+\beta\cdot\sigma(\zeta)+\gamma)-z(\zeta)(a(\zeta)+\beta\cdot id(\zeta)+\gamma)) \overset{?}{=} h(\zeta)z_H(\zeta)
$$

协议完毕。

## 总结

置换证明的核心是 Multiset 等价性证明，而 Multiset 等价性证明的核心是连乘证明。连乘证明的关键技术是引入一个辅助的累乘向量，把连乘的计算转换成一组单乘的计算，而证明过程可以批量地将多个单乘计算的证明合并为一个证明。

## References:

- [WIP] Copy constraint for arbitrary number of wires. https://hackmd.io/CfFCbA0TTJ6X08vHg0-9_g
- Alin Tomescu. Feist-Khovratovich technique for computing KZG proofs fast. https://alinush.github.io/2021/06/17/Feist-Khovratovich-technique-for-computing-KZG-proofs-fast.html#fn:FK20
- Ariel Gabizon. Multiset checks in PLONK and Plookup. https://hackmd.io/@arielg/ByFgSDA7D
