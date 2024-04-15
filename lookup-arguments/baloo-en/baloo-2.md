# Baloo, lookup (Part 2)

Building upon the simplified Baloo proof framework, this article introduces a "complete but unoptimized" Baloo protocol, which incorporates a "Subtable extraction" sub-protocol to support queries on large tables, maintaining the key feature that the proof overhead is only related to the number of queries. The addition of the subtable extraction protocol increases the complexity of the Baloo protocol due to the subtable being encoded on an arbitrary subset $\mathbb{H}_I$ of $\mathbb{H}$. The Caulk paper provided a rather cumbersome scheme, and subsequent work, Caulk+, cleverly used Cached Quotients, greatly optimizing the subtable extraction protocol. Therefore, the Baloo protocol also directly uses the subtable extraction protocol from Caulk+.

The complete Baloo proof framework is as follows:

**Public Inputs**:

1. Main table vector $\vec{t}$, main table commitment $[t(X)]_1$, table length $N$;
2. Subtable vector $\vec{t}_I$, subtable commitment $[t_I(X)]_1$, table length $k$;
3. Query record vector $\vec{a}$, query commitment $[a(X)]_1$, query length $m$, considering the repetition of query records, $k \leq m$.

**Proving goal**:

$$
\exists M \in \mathbb{F}^{m \times k}, \quad M \vec{t}_I = \vec{a}
$$

**Sub-protocol One**: Subtable Protocol

Prover proves to Verifier that $\vec{t}_I \subset \vec{t}$, provides a commitment to a vector of subvector indices $z_I(X)$ and proves its correctness. This is equivalent to proving that every item in the subtable is in the main table, and the index information of each item in the subtable corresponds to the root set of $z_I(X)$. This also implies a relationship: there are no duplicate main table elements in the subtable (unless there are duplicates in the main table).

**Sub-protocol Two**: CP-expansion Protocol

Prover provides Verifier with a commitment to the row space sampling of the matrix $M(\alpha, X)$ $[d(X)]_1$, and proves its correctness. In other words, this is equivalent to proving the existence of a matrix $M$ composed of "row unit vectors", and that its row space sampling vector $\vec{d}$ polynomial commitment is $[d(X)]_1$.

**Sub-protocol Three**: Dot Product Protocol

Prover proves to Verifier that the vector $\vec{d}$ encoded by $d(X)$ and the subtable vector $\vec{t}_I$ have a Dot product equal to $a(\alpha)$, the sampling of the query polynomial at $X=\alpha$. This is equivalent to proving the correctness of the Matrix-vector multiplication required by the proving goal.

Next, we introduce the newly added Subtable sub-protocol, then present an enhanced CP-expansion protocol, and finally give a generalized Univariate Sumcheck protocol, based on which the Dot Product protocol is constructed.

## 5. Subtable Sub-protocol

We now complete the Subtable sub-protocol part required by the Baloo protocol. This part of the proof directly uses the approach from Caulk/Caulk+. The initial idea comes from the "Subvector Commitment Protocol" in [TAB+20], i.e., proving that a subvector is part of another vector. The key idea involves the extended polynomial remainder theorem, that there exists a quotient polynomial $q(X)$ such that:

$$
t(X) - t_I(X) = z_I(X) \cdot q(X)
$$

Here we use $\mathbb{H}$ as the domain for $t(X)$, denote $\mathbb{H}_I$ as the domain where the Subtable vector $\vec{t}_I$ is located, $z_I(X)$ as the Vanishing Polynomial on $\mathbb{H}_I$, and $\{\tau_i(X)\}_{i=0}^{k-1}$ as the Lagrange Polynomials on $\mathbb{H}_I$. $t_I(X)$ is the interpolation polynomial of the subvector $\vec{t}_I$ on $\mathbb{H}_I$,

$$
t_I(X) = \sum_{i=0}^{k-1} t(\omega_{h_i}) \cdot \tau_i(X)
$$

And $I$ represents the indices of each element in the subvector $\vec{t}_I$ within $\mathbb{H}$.

$$
I = (h_0, h_1, \ldots, h_{k-1})
$$

For example, if $\mathbb{H} = (1, \omega, \omega^2, \ldots, \omega^7)$, the table vector is $\vec{t} = (t_0, t_1, \ldots, t_7)$, and the query vector $\vec{a} = (t_2, t_2, t_3, t_6)$, then the subtable vector is $\vec{t}_I = (t_2, t_3, t_6)$, and the index vector is $I = (2, 3, 6)$, which can also be expressed as a subset of the domain: $\mathbb{H}_I = (\omega^2, \omega^3, \omega^6)$.

It's important to note that $\mathbb{H}$ is a regular multiplicative subgroup, but $\mathbb{H}_I$ does not generally form a group. It is dynamically calculated by the Prover based on the subtable indices during each protocol execution. Therefore, performing Lagrange interpolation on $\mathbb{H}_I$ requires a computational complexity of $O(k\log^2{k})$.

Recall that the core idea in Caulk+ for the Subtable protocol is: if $\vec{t}_I \subset \vec{t}$, then there must exist a quotient polynomial $q_I(X)$ such that the following equation holds:

$$
t(X) - t_I(X) = z_I(X) \cdot q_I(X)
$$

However, since the subtable $t_I$ is a witness, Prover needs to send both $[q_I(X)]_1$ and $[z_I(X)]_2$ to the Verifier. Therefore, the premise that ensures the correctness of $t_I(X)$ is that $z_I(X)$ must be correct. The method in the Caulk+ paper to ensure $z_I(X)$ is quite clever. If the subtable relationship holds, then their index sets also satisfy a subset relationship, and thus we can prove that $z_I(X)$ divides $z_\mathbb{H}(X)$. The Prover can then provide a quotient polynomial $z_{\mathbb{H}\backslash I}(X)$ (the commitment $[z_{\mathbb{H}\backslash I}(X)]_1$), so that:

$$
z_\mathbb{H}(X) = z_I(X) \cdot z_{\mathbb{H}\backslash I}(X)
$$

Due to the FFT smoothness of $\mathbb{H}$, the Verifier can quickly compute $[z_\mathbb{H}(X)]_1$ in $O(\log{N})$ time and check whether the two polynomial commitments provided by the Prover satisfy the above equation.

### 5.1 Cached Quotients Precomputation

If Prover directly calculates $q_I(X)$, then Prover needs a computational complexity of $O(N\log{N})$ to complete the division of a polynomial of degree $N$. This does not meet our requirements, as we hope the performance of the Prover is only related to $m$ (or $k$). This requires the use of Cached Quotients technology introduced by [TAB20] and Caulk (due to space limitations, only conclusions are given below, please refer to original paper for related principles and derivations).

In addition to direct polynomial division, the quotient polynomial $q_I(X)$ can also be calculated by a linear combination of a series of (precomputed) quotient polynomials, because $q_I(X)$ has the following internal structure:

$$
q_I(X) = \frac{1}{z_I'(\omega_{h_0})} \cdot q_0(X) + \frac{1}{z_I'(\omega_{h_1})} \cdot q_1(X) + \cdots + \frac{1}{z_I'(\omega_{h_{k-1}})} \cdot q_{k-1}(X)
$$

Where each $q_i(X)$ can be a precomputed quotient polynomial (if the main table $\vec{t}$ is fixed), and $z_I'(\omega_{h_i})$ is the derivative of $z_I(X)$ at $X=\omega_{h_i}$.

$$
q_i(X) = \frac{t(X) - t(\omega_i)}{X-\omega_i}
$$

$$
z_I'(\omega_{h_i}) = \prod_{j \in I, j \neq i} (\omega_{h_i} - \omega_{h_j})
$$

In this way, Verifier or any third party can precompute the commitments of all quotient polynomials, i.e., $\{[q_i(X)]_1\}_{i\in[N]}$. During the proof process, the Prover independently calculates $\{z_I'(\omega_{h})\}_{h\in I}$ and computes $[q_I(X)]_1$ to send to the Verifier. Thus, the computational load of the Prover is only related to $k$ and no longer related to $N$, with a computational complexity of only $O(k\log^2k)$.

In addition, if $z_{\mathbb{H}\backslash I}(X)$ is directly calculated, it also requires a workload of $O(N\log{N})$. The Caulk+ paper found that $z_{\mathbb{H}\backslash I}(X)$ can also be precomputed using Cached Quotients, greatly optimizing the cumbersome method of the original Caulk paper.

We now derive a reciprocal decomposition equation for a product series. For a constant polynomial $f(X) = 1$, we obtain

$$
1 = f(X) = \sum_{h \in I} f_i \cdot \frac{1}{z'_I(\omega_{h})} \frac{z_I(X)}{X-\omega_{h}} = \sum_{h \in I} \frac{1}{z'_I(\omega_{h})} \frac{z_I(X)}{X-\omega_{h}}
$$

Divide both sides of the equation by $z_I(X)$ to obtain:

$$
\frac{1}{z_I(X)} = \sum_{h \in I} \frac{1}{z'_I(\omega_{h})} \cdot \frac{1}{X-\omega_{h}}
$$

Then observe the definition of $z_{\mathbb{H}\backslash I}(X)$, we can get the following equation:

$$
\begin{split}
z_{\mathbb{H}\backslash I}(X) &= \prod_{i \not\in I} (X-\omega_i) \\
&= z_\mathbb{H(X)} \cdot \frac{1}{\prod_{h \in I} (X-\omega_h)} \\
&= z_\mathbb{H(X)} \cdot \frac{1}{z_I(X)} \\
&= z_\mathbb{H(X)} \cdot \Big(\sum_{h \in I} \frac{1}{z'_I(\omega_h)} \cdot \frac{1}{X-\omega_h} \Big) \\
&= \sum_{h \in I} \frac{1}{z'_I(\omega_h)} \cdot \frac{z_\mathbb{H(X)} }{X-\omega_h}
\end{split}
$$

Thus, $z_{\mathbb{H}\backslash I(X)}$ can be seen as a linear combination of $\{w_i(X) = \frac{z_\mathbb{H(X)} }{X-\omega_i}\}_{i\in[N]}$, and $\{c_i = \frac{1}{z'_I(\omega_{h_i})}\}_{h_i \in I}$ as the so-called Barycentric Weights, which require a computational load that is also only related to $k$ during the proof. We can let the protocol precompute $\{\frac{z_\mathbb{H(X)} }{X-\omega_i}\}_{i\in[N]}$, so that the Prover only needs to calculate (with a computational complexity of $O(k\log^2k)$) Barycentric Weights and a linear combination operation of $k$ multiplications.

### 5.2 Protocol Details

**Public Inputs**:

1. Commitment of the large table $[t(X)]_1$,
2. Commitment of the subtable $[t_I(X)]_1$,
3. Commitment of the subtable element index vector in $\mathbb{G}_2$ $[z_I(X)]_2$

**Proving goal**:

$$
t(X) - t_I(X) = z_I(X) \cdot q_I(X)
$$

$$
z_\mathbb{H(X)} = z_I(X) \cdot z_{\mathbb{H}\backslash I(X)}
$$

**Precomputation**

1. $\{[q_i(X)]_1 \}_{i\in[N]}$, each committed polynomial satisfies the following equation:

$$
q_i(X) \cdot (X-\omega_i) = t(X) - t(\omega_i)
$$

2. $\{[w_i(X)]_1\}_{i\in[N]}$, each committed polynomial satisfies the following equation:

$$
w_i(X) \cdot (X-\omega_i) = z_\mathbb{H(X)}
$$

**Step One**:

1. Prover calculates Barycentric Weights $\{c_i\}_{i\in[k]}$

$$
c_i = \frac{1}{z'_I(\omega_{

h_i})}, \quad \forall i \in [0,k)
$$

2. Prover calculates and sends $Q = [q(X)]_1$, and $W = [w(X)]_1$
$$
[q(X)]_1 = \sum_{i \in [k]} c_i \cdot [q_i(X)]_1
$$

$$
[w(X)]_1 = \sum_{i \in [k]} c_i \cdot [w_i(X)]_1
$$

**Verification**: Verifier verifies the following two pairing equations:

$$
\begin{split}
\Big([t(X)]_1 - [t_I(X)]_1\Big) \ast [1]_2 &\overset{?}{=} Q \ast [z_I(X)]_2 \\
\Big([z_\mathbb{H}(X)]_1\Big) \ast [1]_2 &\overset{?}{=} W \ast [z_I(X)]_2 \\
\end{split}
$$

## 6. Generalized Univariate Sumcheck

If we consider $\vec{t}_I$ as a subtable of some main table $\vec{t}$, and $t_I(X)$ is no longer a Lagrange interpolation polynomial on a regular multiplicative subgroup, then the simplified CP-expansion protocol previously introduced is not applicable. Before enhancing the CP-expansion protocol, let's introduce a more generalized Univariate Sumcheck theorem. The new Univariate Sumcheck theorem no longer requires that $t_I(X)$ be exactly encoded on a multiplicative subgroup.

First, recall the simplified Sumcheck theorem:

$$
a(X) = \frac{\sigma}{n} + X \cdot g(X) + z_\mathbb{H(X)} \cdot q(X), \quad \deg(r(X))<n-1
$$

Here, $\sigma = \sum_{i \in [n]} a(\omega_i)$ is the sum of the evaluations of $a(X)$ on a multiplicative subgroup $\mathbb{H}$.

### 6.1 Generalization

We temporarily assume $H = (h_0, h_1, \ldots, h_{k-1}) \subset \mathbb{F}$ is any subset on a finite field, the polynomial $z_H$ is the Vanishing Polynomial on $H$, and its corresponding Lagrange polynomials are $\{\tau_i(X)\}_{i \in [k]}$.

Now, if we want to prove the sum of all elements in a vector $\vec{a}$ of length $k$ equals $\sigma$, i.e., $\sum_i a_i = \sigma$, then we first try to construct the corresponding polynomial $a(X) = \sum_{i \in [k]} a_i \cdot \tau_i(X)$ in the usual way and substitute $X = 0$ to calculate the constant term of the polynomial. Then we can get the following equation:

$$
a(0) = \sum_{i=0}^{k-1} a_i \cdot \tau_i(0)
$$

Although the right side of the equation is close to $\sigma = \sum_{i=0}^{n-1} a(\omega^i)$, each term in the summation carries an annoying factor $\tau_i(0)$. How do we eliminate this factor?

We can actually change the definition of $a(X)$ so that it is no longer the usual Lagrange interpolation polynomial about $\vec{a}$, but rather an interpolation process that eliminates a factor in advance.

$$
a(X) = \sum_{i=0}^{k-1} a_i \cdot \frac{\tau_i(X)}{\tau_i(0)}
$$

In this way, $a(0) = \sum_i a_i = \sigma$ can hold. Thus, we can also get the following relationship:

$$
a(X) - a(0) = X \cdot r(X), \quad \deg(r(X))<k-1
$$

Understandably, because $a(0)$ is the constant term of $a(X)$, when it is subtracted, $a(X)$ can be divided by $X$, so $a(X) - a(0) = X \cdot r(X)$.

This can be further extended. If we need to prove the dot product of two vectors of length $k$, such as $\vec{a} \cdot \vec{b} = \sigma$, then we can construct two interpolation polynomials based on $H$, $a(X)$ and $b(X)$,

$$
a(X) = \sum_{i=0}^{k-1} a_i \cdot \frac{\tau_i(X)}{\tau_i(0)} \quad
b(X) = \sum_{i=0}^{k-1}

 b_i \cdot \tau_i(X)
$$

It is noteworthy that we only use Normalized Lagrange interpolation for $a(X)$ here, while $b(X)$ still uses ordinary Lagrange interpolation. This is because we only need to prove that the constant term of $a(X) \cdot b(X)$ is $\sigma$, so only one-sided introduction of a Nomialized Lagrange interpolation is needed to eliminate the $\tau_i(0)$ factor:

$$
\begin{split}
a(X) \cdot b(X) &= \Big(\sum_{i=0}^{k-1} a_i \cdot \frac{\tau_i(X)}{\tau_i(0)}\Big) \cdot \Big(\sum_{i=0}^{k-1} b_i \cdot \tau_i(X)\Big) \\
& \equiv \Big(\sum_{i=0}^{k-1} a_i \cdot b_i \cdot \frac{\tau_i(X)}{\tau_i(0)}\Big) \pmod{z_H(X)} \\
& \equiv \sigma + X \cdot r(X) \pmod{z_H(X)}, \quad \deg(r(X))<k-1
\end{split}
$$

This derivation uses the 'mutually orthogonal' property of Lagrange polynomials, and their evaluations on $H$ as a Unit Vector, i.e., $\tau_i(X)|_{X=\omega_i}=1$, while $\tau_i(X)|_{X=\omega_j, i \not =j}=0$, this unit orthogonality can be represented by the following two equation relationships:

$$
\begin{split}
\tau_i(X) \cdot \tau_i(X) &\equiv \tau_i(X) \pmod{z_H(X)}\\
\tau_i(X) \cdot \tau_j(X) &\equiv 0 \pmod{z_H(X)}, \quad i \neq j
\end{split}
$$

You can verify that both sides of these two equations hold at any $h_i \in H$, therefore the modulo relationship also holds. This property ensures that we can perform 'modular multiplication' of polynomials under $z_H(X)$.

This method of eliminating the $\tau_i(0)$ factor in the interpolation is referred to as Nomalized Lagrange Interpolation in the Baloo paper. Let's simply follow the paper's approach and name $\{\tau_i(X)/\tau_i(0)\}$ as Normalized Lagrange Polynomials, denoted as $\{\hat\tau_i(X)\}$.

## 7. Enhanced CP-expansion Protocol

This section describes an enhanced CP-expansion protocol, proving that a subvector $\vec{t}_I$ undergoes a linear transformation represented by matrix $M$, resulting in a vector $\vec{a}$, where matrix $M$ is a matrix with unit vectors as rows.

$$
M \vec{t}_I = \vec{a}
$$

Here, $\vec{t}_I$ is a subvector of $\vec{t}$, length $k$, later encoded as $t(X)$. Here we use the symbol $\mathbb{H}$ to represent the Domain for $t(X)$, and $I$ represents the index of each element in the subvector $\vec{t}_I$ within $\mathbb{H}$.

To reiterate, here $t_I(X)$ represents the interpolation polynomial of the subvector $\vec{t}_I$. We denote $\mathbb{H}_I$ as the Domain where $\vec{t}_I$ is located, $z_I(X)$ as the Vanishing Polynomial on $\mathbb{H}_I$, and $\{\tau_i(X)\}_{i=0}^{k-1}$ as the Lagrange Polynomials on $\mathbb{H}_I$. The corresponding Normalized Lagrange polynomials are denoted as $\{\hat\tau_i(X)\}_{i=0}^{k-1}$. Please note that the domain of the $\{\hat\tau_i(X)\}_{i=0}^{k-1}$ polynomials is $\mathbb{H}_I$, not $\mathbb{H}$.

After encoding these vectors into polynomials, the vector multiplication relationship can be transformed into a polynomial multiplication relationship:

$$
\sum_{Y \in \mathbb{H}_I} M(X, Y) \cdot t_I(Y) = a(X)
$$

Here $M(X, Y)$ is defined as follows:

$$
M(X, Y) = \begin{bmatrix}
\mu_0(X), \mu_1(X), \cdots, \mu_{m-1}(X) \\
\end{bmatrix}
\begin{bmatrix}
M_{0,0} & M_{0,1} & \cdots & M_{0,n

-1} \\
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

Use a random challenge $\alpha$ provided by the Verifier, we can reduce the polynomial multiplication relationship to a new dot product summation relationship:

$$
\sum_{X \in \mathbb{H}_I} M(\alpha, X) \cdot t_I(X) = a(\alpha)
$$

Where

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

Upon further analysis of $M(\alpha, X)$, unfolding its definition, we obtain:

$$
M(\alpha, X) = \sum_{i=0}^{m-1}\sum_{j=0}^{k-1}\mu_i(\alpha)\cdot M_{i,j}\cdot \hat\tau_{j}(X)
$$

Considering that $M$ has only one $1$ in each row and the rest of the elements are $0$, thus $\sum_{j=0}^{k-1} M_{i,j}\cdot \hat\tau_{j}(X) = \hat\tau_{\mathsf{col}(i)}(X)$, where $\mathsf{col}: [k] \to I$, represents the column index function of the non-zero element in row $i$.

Thus $M(\alpha, X)$ can be further simplified and noted as $d(X)$:

$$
d(X) = M(\alpha, X) = \sum_{i=0}^{m-1}\sum_{j=0}^{k-1}\mu_i(\alpha)\cdot M_{i,j}\cdot \hat\tau_{j}(X) =\sum_{i=0}^{m-1}\mu_i(\alpha)\cdot \hat\tau_{\mathsf{col}(i)}(X)
$$

The next question is, how do we prove the correctness of $d(X)$, i.e., $M(\alpha, X)$? Since $M(\alpha, X)$ is a partial operation of a binary polynomial, we can associate it with another partial operation of $M(X, Y)$, $M(X, \beta)$, to prove that both are derived from the same $M(X, Y)$, as long as we can prove that any one of $M(\alpha, X)$ or $M(X, \beta)$ correctly encodes $M$, then we can prove the correctness of $M(\alpha, X)$ (why we need to prove the correctness of $d(X)$ in this way, please refer to Section 2.1 in Part 1).

Proving that $M(\alpha, X)$ and $M(X, \beta)$ are both derived from the same $M(X, Y)$ is not difficult, we only need to prove the following relationship through Oracle:

$$
M(\alpha, X=\beta) = M(X=\alpha, \beta)
$$

Next, we still need to prove the correctness of $M(X, \beta)$, we temporarily use a new symbol $e(X)=M(X, \beta)$.

$$
e(X) = \sum_{i=0}^{m-1}\hat\tau_{\mathsf{col}(i)}(\beta)\cdot \mu_i(X)
$$

### 7.1 Correctness of $e(X)$

In fact, $e(X)$ encodes the column space sampling vector $\vec{e}=(e_0, e_1, \ldots, e_{m-1})$, which is of length $m$. We can define $e_i$ by the following equation:

$$
e_i = \hat\tau_{\mathsf{col}(i)}(\beta) = \frac{\tau_{\mathsf{col}(i)}(\beta)}{\tau_{\mathsf{col}(i)}(0)}
= \frac{\prod_{j \neq \mathsf{col}(i)} (\beta - \omega_j)}{\prod_{j \neq \mathsf{col}(i)} (-\omega_j)}
= \frac{z_I(\beta)}{z_I(0)}\cdot \frac{-\omega_{\mathsf{col}(i)}}{\beta - \omega_{\mathsf{col}(i)}}
$$

We define $\vec{v}$ as $v_i = \omega_{\mathsf{col}(i)}$, then $e_i$ can be expressed as:

$$
e_i = \frac{z_I(\beta)}{z_I(0)}\cdot \frac{-\omega_{\mathsf{col}(i)}}{\beta - \omega_{\mathsf{col}(i)}} = \frac{z_I(\beta)}{z_I(0)}\cdot \frac{-v_i}{\beta - v_i}
$$

Simplify the equation to obtain:

$$
e_i \cdot (\beta - v_i) + \frac{z_I(\beta)}{z_I(0)}\cdot {v_i} = 0
$$

The auxiliary vector $\vec{v}$ on $\mathbb{V}$ is defined as the interpolation polynomial $v(X)$:

$$
v(X)=\sum_{i=0}^{m-1}v_i\cdot \mu_i(X) = \sum_{i=0}^{m-1}\omega_{\mathsf{col}(i)}\cdot \mu_i(X)
$$

Convert the above relationship into a relationship between polynomials to obtain:

$$
e(X)\cdot (\beta - v(X)) + \frac{z_I(\beta)}{z_I(0)}\cdot v(X) = z_\mathbb{V(X)} \cdot q(X), \quad \deg(e(X))<m
$$

So far, as long as the Prover can prove the above equation holds and prove that the degree of $e(X)$ does not exceed $m$, then $e(X)$ can have the following form:

$$
e(X)
= \sum_{i=0}^{m-1} \frac{z_I(\beta)}{z_I(0)}\cdot \frac{-v_i}{\beta - v_i}\cdot \mu_i(X)
$$

If $\vec{v}$ is correct, then we can conclude that $e(X)$ is correct. How do we ensure the correctness of $\vec{v}$?

### 7.2 Correctness of $v(X)$

The correctness of the vector $\vec{v}$ relies on the constraint $d(\beta)=e(\alpha)$. Since

$$
e(X) = \sum_{i=0}^{m-1}\hat\tau_{\mathsf{col}(i)}(\beta)\cdot \mu_i(X)
= \sum_{i=0}^{m-1}\frac{-z_I(\beta)\cdot v_i}{z_I(0)(\beta - v_i)}\cdot \mu_i(X)
$$

If $d(\beta)=e(\alpha)$ holds, according to the symmetry of $d(X)$ and $e(X)$, we replace $X$ with $\alpha$ in the right side of the equation above, and replace $\beta$ with the unknown $X$ to obtain:

$$
d(X) = \sum_{i=0}^{m-1}\frac{-z_I(X)\cdot v_i}{z_I(0)(X - v_i)}\cdot \mu_i(\alpha)
$$

If a certain $v_j \not\in \mathbb{H}_I$, then in the $j$th term of the definition of $d(X)$, $(X-v_j)$ will not divide $z_I(X)$. Moreover, because of the random factor $\mu_i(\alpha)$ (here it is required that the random number $\alpha$ must be provided by the Verifier after the commitments of $v(X)$ and $d(X)$ are generated), this term cannot cancel out with other terms, resulting in $d(X)$ no longer being a Polynomial but a Rational Function, i.e., $d(X)\in\mathbb{F}(X)$, but $d(X)\not\in\mathbb{F}[X]$. If this is the case, the Prover cannot calculate the polynomial commitment of $d(X)$.

Therefore, as long as the Prover provides a polynomial commitment $[d(X)]_1$ and proves $d(\beta)=e(\alpha)$, it also proves that $v_i\in\mathbb{H}_I$, further ensuring the correctness of $v(X)$.

### 7.3 Correctness of $d(X)$

The remaining correctness of $d(X)$ can be proved by the correctness of $e(X)$ and $v(X)$, along with the constraint $d(\beta)=e(\alpha)$.

### 7.4 CP-expansion Protocol Details

**Public Inputs**:

1. Commitment of the subtable $[t_I(X)]_1$,
2. Commitment to the row space sampling vector $[d(X)]_1$,
3. Challenge number $\alpha$ provided by Verifier
4. Commitment to the subtable index vector in $\mathbb{G}_1$

**Proving goal**: There exists a matrix $M$ with Unit Vectors as rows, such that $\vec{d}$ is the row space sampling vector of this matrix.

$$
d_i = \mu_i(\alpha)\cdot \vec{M}_i
$$

$$
d(X) = \sum_{i=0}^{m-1}\mu_i(\alpha)\cdot \vec{M}_i(X)
$$

**Step One**: Prover calculates the encoding of matrix $M$, $v(X)$ and sends $V=[v(X)]_1$

$$
v(X) = \sum_{i=0}^{m-1}\omega_{\mathsf{col}(i)}\cdot \mu_i(X)
$$

**Step Two**: Verifier sends a random challenge number $\beta$

**Step Three**:

1. Prover calculates $e(X)$ and $q_1(X)$, and sends $E=[e(X)]_1$, $Q_1=[q_1(X)]_1$,

$$
e(X)\cdot (\beta - v(X)) + \frac{z_I(\beta)}{z_I(0)}\cdot v(X) = z_\mathbb{V(X)} \cdot q_1(X)
$$

2. Prover calculates the quotient polynomial of $z_I(0)$, $q_2(X)$, and sends $z_I(0)$ and $Q_2=[q_2(X)]_1$,

$$
q_2(X) = \frac{z_I(X)-z_I(0)}{X}
$$

3. Prover calculates and sends $d(\beta)$ and $z_I(\beta)$

4. Prover calculates and sends $e(\alpha)$, and the quotient polynomial $q_3(X)$, and sends $Q_3=[q_3(X)]_1$

$$
q_3(X) = \frac{e(X)-e(\alpha)}{X-\alpha}
$$

5. Prover sends the degree-bound proof of $e(X)$ $[X^{D-m}\cdot e(X)]_1$

**Step Four**: Verifier sends challenge point $X=\zeta$

**Step Five**: Prover sends $e(\zeta)$, $v(\zeta)$, $q_1(\zeta)$

**Step Six**: Verifier sends a random challenge number $\gamma$ for aggregated polynomial evaluation proof

**Step Seven**:

1. Prover calculates and sends $W_1=[w_1(X)]_1$,

$$
w_1(X) = \frac{z_I(X) - z_I(\beta)}{X-\beta} + \gamma\cdot \frac{d(X) - d(\beta)}{X-\beta}
$$

2. Prover calculates the combined quotient polynomial $w_2(X)$ and sends its commitment $W_2=[w_2(X)]_1$

$$
w_2(X) = \frac{e(X)-e(\zeta)}{X-\zeta} + \gamma\cdot\frac{v(X)-v(\zeta)}{X-\zeta} + \gamma^2\cdot \frac{q_1(X)-q_1(\zeta)}{X-\zeta}
$$

**Verification**: Verifier verifies the following steps:

1. Calculate $z_{\mathbb{V}}(\zeta)$ and verify the correctness of $e(X)$

$$
e(\zeta)\cdot(\beta - v(\zeta))+z_I(\beta)z_I(0)^{-1}\cdot v(\zeta) \overset{?}{=} z_{\mathbb{V}}(\zeta)\cdot q_1(\zeta)
$$

2. Verify the correctness of $Q_2$:

$$
\Big([z_I(X)]_1 - z_I(0)\cdot[1]_1 \Big) \ast [1]_2 \overset{?}{=} Q_2 \ast [X]_2
$$

3. Verify the correctness of $Q_3$:

$$
\Big(E - e(\alpha)\cdot[1]_1 \ Big) \ast [1]_2 \overset{?}{=} Q_3 \ast [X]_2
$$

4. Calculate the combined polynomial commitment $P_1$ and $y_1$ and verify the correctness of $W_1$

$$
\begin{split}
P_1 & = [z_I(X)]_1 + \gamma\cdot [d(X)]_1 \\
y_1 & = z_I(\beta) + \gamma\cdot d(\beta) \\
\end{split}
$$

$$
\Big(P_1 - y_1\cdot[1]_1 \ Big) \ast [1]_2 \overset{?}{=} W_1 \ast \Big([X]_2 - \beta
\cdot[1]_2 \ Big)
$$

5. Calculate the combined polynomial commitment $P_2$ and $y_2$ and verify the correctness of $W_2$


$$
\begin{split}
P_2 & = E + \gamma\cdot V + \gamma^2\cdot G + \gamma^3\cdot Q + \gamma^4\cdot [z_I(X)]_1 \\
y_2 & = e(\zeta) + \gamma\cdot v(\zeta) + \gamma^2\cdot q(\zeta) \\
\end{split}
$$

$$
\Big(P_2 - y_2\cdot[1]_1 \ Big) \ast [1]_2 \overset{?}{=} W_2 \ast \Big([X]_2 - \zeta
\cdot[1]_2 \ Big)
$$

6. Verify $\deg(e(X)) <m$

$$
\Big([X^{D-m}\cdot e(X)]_1 \ Big) \ast [1]_2 \overset{?}{=} E \ast [X^{D-m}]_2
$$


## 8. Dot Product Protocol

Finally, we need to prove the remaining summation relationship:

$$
\sum_{X \in \mathbb{H}_I} M(\alpha, X) \cdot t_I(X) = a(\alpha)
$$

Using the Generalized Univariate Sumcheck theorem mentioned earlier, for two vectors of length $k$, if $\vec{A} \cdot \vec{B} = s$, then the polynomials of the two vectors satisfy the following relationship (for the specific derivation process, please refer to Section 4.1 earlier):

$$
A(X) \cdot B(X) \equiv s + X \cdot R(X) \pmod{z_I(X)}, \quad \deg(R(X))<k-1
$$

Where

$$
A(X) = \sum_{i=0}^{k-1} A_i \cdot \frac{\tau_i(X)}{\tau_i(0)} \quad
B(X) = \sum_{i=0}^{k-1} B_i \cdot \tau_i(X)
$$

Substituting $A(X) = M(\alpha, X)$ and $B(X) = t_I(X)$ into the equation above, we can obtain the following equation:

$$
M(\alpha, X) \cdot t_I(X) \equiv \sigma + X \cdot r(X) + z_I(X) \cdot q_1(X), \quad \deg(r(X))<k-1
$$

Unlike before, the domain of $t_I(X)$ is no longer an FFT smooth multiplicative subgroup, but an arbitrary subset, hence $M(X, Y)$ matrix encoding uses Normalized Lagrange Polynomials, $\hat\tau_i(X)$.

**Public Inputs**:

1. Commitment of the subtable $[t_I(X)]_1$,
2. Commitment to the row space sampling vector $[d(X)]_1$,
3. Value of the query record vector interpolation polynomial $a(X)$ at $X=\alpha$ $a(\alpha)$,
4. Commitment to the subtable index vector in $\mathbb{G}_1$ $[z_I(X)]_1$.

**Proving goal**

$$
\vec{d} \cdot \vec{t}_I = a(\alpha)
$$

**Step One**: Prover calculates $q(X)$ and $g(X)$, satisfying the following relationship:

$$
d(X) \cdot t_I(X) = a(\alpha) + X \cdot g(X) + q(X) \cdot z_I(X)
$$

Send $Q=[q(X)]_1$, $G=[g(X)]_1$

**Step Two**: Verifier sends a random challenge number $\zeta$

**Step Three**: Prover sends $d(\zeta)$, $t_I(\zeta)$, $g(\zeta)$, $q(\zeta)$, $z_I(\zeta)$

**Step Four**: Verifier sends a random challenge number $\gamma$

**Step Five**: Prover calculates the combined quotient polynomial $w(X)$ and sends its commitment $W=[w(X)]_1$

$$
w(X) = \frac{d(X)-d(\zeta)}{X-\zeta} + \gamma \cdot \frac{t_I(X)-t_I(\zeta)}{X-\zeta} + \gamma^2 \cdot \frac{g(X)-g(\zeta)}{X-\zeta} + \gamma^3 \cdot \frac{q(X)-q(\zeta)}{X-\zeta} + \gamma^4 \cdot \frac{z_I(X)-z_I(\zeta)}{X-\zeta}
$$

**Verification**: Verifier verifies the following steps:

1. Verify the Generalized Sumcheck equation

$$
d(\zeta) \cdot t_I(\zeta) \overset{?}{=} a(\alpha) + \zeta \cdot g(\zeta) + q(\zeta) \cdot z_I(\zeta)
$$

2. Calculate the combined polynomial commitment $P$ and $Z$

$$
\begin{split}
P & = [d(X)]_1 + \gamma \cdot [t_I(X)]_1 + \gamma^2 \cdot G + \gamma^3 \cdot Q + \gamma^4 \cdot [z_I(X)]_1 \\
z & = d(\zeta) + \gamma \cdot t_I(\zeta) + \gamma^2 \cdot g(\zeta) + \gamma^3 \cdot q(\zeta) + \gamma^4 \cdot z_I(\zeta) \\
\end{split}
$$

3. Verify the correctness of $W$

$$
\Big(P - z \cdot [1]_1 \ Big) \ast [1]_2 \overset{?}{=} W \ast \Big([X]_2 - \zeta
\cdot [1]_2 \ Big)
$$

