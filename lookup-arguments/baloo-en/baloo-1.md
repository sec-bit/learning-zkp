# Baloo, lookup (Part 1)

Baloo is a improved protocol based on Caulk and Caulk+ that supports query proofs for large tables and serves as a pivotal Lookup Argument protocol. This article is part one, introducing the basic ideas of the Baloo protocol.

## 1. Overview of Baloo

Assume there is a table $\vec{t}$ with size $N$. There are $m$ queries, denoted by $\vec{a}$. We use the symbol $\{\vec{a}\}$ to represent the set of all elements in the vector, and the symbol $\{\vec{a}\}\subset \{\vec{t}\}$ to indicate a subset relationship between sets of vector elements, and $\vec{f}\subset\vec{t}$ to indicate that $\vec{f}$ is a subvector of $\vec{t}$.

We aim to construct a proof system such that:

$$
\forall a_i \in \{\vec{a}\}, a_i \in \{\vec{t}\}
$$

meaning $\{\vec{a}\}\subset \{\vec{t}\}$. However, please note that we need to consider potential duplicate query elements in $\vec{a}$.

Previous Lookup protocols like Caulk/Caulk+ supported large table queries and kept the Prover's proof overhead only related to the number of queries, meaning Prover Time is a function of $m$. The Caulk/Caulk+ protocols managed to reduce Prover's proof overhead to $O(m^2)$, while the subsequent Flookup protocol further improved the proof overhead to $O(m\log^2{m})$. However, a major downside of the Flookup protocol is that the table's commitment loses the "additive homomorphic" property seen in previous Lookup Arguments (like Plookup, Halo2-lookup, Caulk/Caulk+), thus only supporting "single-list" query proofs. In practice, multi-list table queries are very common.

The Baloo protocol introduced here maintains similar proof overheads of $O(m\log^2{m})$ while still retaining the additive homomorphic properties of table commitments from the Caulk/Caulk+ protocols, thus restoring support for multi-list table queries.

The basic idea of Baloo is to reduce the Lookup Argument to a linear relationship: Matrix-Vector Multiplication Argument, meaning if the query relationship $\{\vec{a}\} \subset \{\vec{t}\}$ holds, then there exists an $m\times N$ matrix $M$, such that the following multiplication holds:

$$
M \vec{t} = \vec{a}
$$

And the matrix $M$ has a particular property where each row is a "unit vector" (Unit Vector), meaning the row vector has only one element as $1$ and the rest as $0$. This property ensures each query is merely a direct copy of "a specific element" from the table (rather than a linear combination of several table elements). This characteristic is crucial, and the subsequent protocols will fully utilize this feature, so please keep it in mind.

For instance, if the table is $t=(1,2,3,4,5,6)^{\top}$ and the query record is $a=(2,4,6)^{\top}$, then we can construct a $3\times 6$ matrix $M$, such that:

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

To make Prover Time independent of the size of $\vec{t}$, we first need a sub-protocol to separate the elements of $t$ related to $\vec{a}$, denoted as $\vec{t}_I$, and then prove the subvector relationship between this sub-table and the main table, and then use the Matrix-Vector multiplication argument to prove the relationship between $\vec{t}_I$ and $\vec{a}$.

$$
M \vec{t}_I = \vec{a} \wedge \vec{t}_I \subset \vec{t}
$$

MVMA (Matrix-Vector multiplication argument) often appears in R1CS-based zkSNARK constructions, but here the MVMA is different. The key difference is: in R1CS zkSNARK constructions, the matrix $M$ represents the circuit topology and is a public value; however, in the Baloo protocol, $M$ is an intermediate value temporarily constructed by the Prover, and to further consider privacy protection, $M$ must also be mixed with enough random values to ensure its information is not obtained by the Verifier. This introduces several challenging issues:

**Challenge One**: When proving $M\vec{t}_I=\vec{a}$, at least $M$ and $\vec{t}_I$ are hidden values, and the Verifier can only obtain their Oracle. Further, if we use Baloo as a sub-protocol in Plonk, replacing the Plookup protocol, then $\vec{a}$ is also a hidden value. This requires the Prover to prove the correctness of this multiplication when the Verifier can only obtain the Oracle.

**Challenge Two**: This matrix invisible to the Verifier has a special nature: each row can and must have only one non-zero element, and this non-zero element can only be equal to $1$. In other words, each row of the matrix is a Selector, only able to select an element from the table $\vec{t}_I$. This requires the Prover to prove the matrix-vector multiplication relationship while also proving that this matrix $M$ has this particular property.

**Challenge Three**: If we do not consider the internal structure of $M$ and directly use a general method to prove the matrix-vector multiplication, then the Prover needs to traverse each element of the matrix, with the proof overhead at least being $O(m*n)$. Assuming the sub-table size is $n=O(m)$, then the proof overhead complexity is $O(m^2)$, which is considerably large.

## 2. Core Concept: Random Sampling in Vector Space

To facilitate understanding the core idea of the Baloo protocol, let's simplify the problem by temporarily ignoring the "sub-table extraction issue" and assume that the sub-table and the original table are identical, $\vec{t} = \vec{t}_I$, but retaining a key characteristic: the table $\vec{t}$ needs to remain hidden from the Verifier.

Assume the size of table $\vec{t}$ is $n$, so $n=|\vec{t}|$, and the number of query records is $m=|\vec{a}|$. Our goal is to prove that there exists a special matrix $M\in\mathbb{F}^{m\times n}$ that satisfies the following matrix-vector multiplication relation:

$$
M\vec{t} = \vec{a}
$$

In the protocol, the Verifier has polynomial commitments for both the table and query vectors, namely the public inputs $[t(X)]$ and $[a(X)]$.

### 2.1 Polynomial Encoding

These two polynomials $t(X)$ and $a(X)$ use Lagrange Basis to encode vectors $\vec{t}$ and $\vec{a}$. Assume there is an FFT-smooth multiplicative subgroup $\mathbb{H} \subset \mathbb{F}_p^*$ in the prime field $\mathbb{F}_p$, with size exactly $n$ and generator $\omega$. Being FFT-smooth means the size of $\mathbb{H}$ is exactly $2^k$, thus we have $\omega^{2^k}=1$, and $\omega$ is the $n$th root of unity, allowing us to perform FFT algorithms in complexity $O(n\log{n})$.

We can define the Lagrange polynomials over $\mathbb{H}$:

$$
\tau_i(X) = \frac{z_\mathbb{H}(X)}{z_\mathbb{H}'(\omega^i)\cdot(X - \omega^i)} , \quad i=0,1,\ldots,n-1
$$

Here $z_\mathbb{H}(X)$ is the Vanishing polynomial, defined as: $z_\mathbb{H}(X) = \prod_{i=0}^{n-1}(X-\omega^i)$. And $z_\mathbb{H}'(X)$ is the derivative polynomial of $z_\mathbb{H}(X)$, because $\mathbb{H}$ is a multiplicative subgroup,

$$
z_\mathbb{H}'(\omega^i)=(X^n-1)'|_{X=\omega^i}=n\cdot (X^{n-1})|_{X=\omega^i} = n\cdot (\omega^i)^{n-1} = n\cdot \omega^{-i}
$$

Thus, the definition of the Lagrange polynomial $\tau_i(X)$ can be simplified to:

$$
\tau_i(X) = \frac{\omega^i\cdot z_\mathbb{H}(X)}{n\cdot(X - \omega^i)}, \quad i=0,1,\ldots,n-1
$$

A key idea in Baloo is to introduce a new vector $\vec{d}$, which is a "random sampling" from the "row space" of matrix $M$. Before defining $\vec{d}$, we first encode matrix $M$ using a "bivariate polynomial":

$$
M(X, Y) = \begin{bmatrix}
\mu_0(X), \mu_1(X), \ldots, \mu_{m-1}(X) \\
\end{bmatrix}
\begin{bmatrix}
M_{0,0} & M_{0,1} & \ldots & M_{0,n-1} \\
M_{1,0} & M_{1,1} & \ldots & M_{1,n-1} \\
M_{2,0} & M_{2,1} & \ldots & M_{2,n-1} \\
\vdots & \vdots & \ddots & \vdots \\
M_{m-1,0} & M_{m-1,1} & \ldots & M_{m-1,n-1} \\
\end{bmatrix}
\begin{bmatrix}
\tau_0(Y) \\
\tau_1(Y) \\
\tau_2(Y) \\
\vdots \\
\tau_{n-1}(Y) \\
\end{bmatrix}
$$

To encode the bivariate polynomial, we also need another FFT-smooth multiplicative subgroup $\mathbb{V}\subset \mathbb{F}_p^*$, with size exactly $m=2^l$, and generator $\nu$. Similarly, the Lagrange polynomials

 for this domain are:

$$
\mu_i(X) = \frac{z_\mathbb{V}(X)}{z_\mathbb{V}'(\nu^i)\cdot(X - nu^i)} = \frac{\nu^i\cdot z_\mathbb{V}(X)}{m\cdot(X - \nu^i)}, \quad i=0,1,\ldots,m-1
$$

Where $z_\mathbb{V}(X)$ is the Vanishing polynomial for domain $\mathbb{V}$.

### 2.1 Row Space Sampling

The so-called "random sampling of the row space" of a matrix refers to introducing a random vector $\vec{\rho}$, where all elements are linearly independent, and then calculating a vector from the linear combination of all rows of the matrix.

Looking at the definition of $M(X,Y)$ above, we can directly use a random value $\alpha$, and then use the value of the Lagrange polynomial at $X=\alpha$, $\mu_i(\alpha)$, as this random sampling vector, then calculate the linear combination of all row vectors of the matrix to get $\vec{d}$, with length equal to the width of the matrix $|\vec{d}|=n$. This calculation can be seen as a "vertical folding" of matrix $M$:

$$
\vec{d} =
\begin{bmatrix}
d_0, d_1,\ldots, d_{n-1}
\end{bmatrix} = \begin{bmatrix}
\sum_{i=0}^{m-1} \mu_{i}(\alpha)M_{i,0}, & \sum_{i=0}^{m-1} \mu_{i}(\alpha)M_{i,1}, & \ldots, & \sum_{i=0}^{m-1} \mu_{i}(\alpha)M_{i,n-1} \\
\end{bmatrix}
$$

We can introduce an auxiliary polynomial $d(X)$ on $\mathbb{H}$ to encode this vector,

$$
d(X) = \sum_{j=0}^{n-1}\sum_{i=0}^{m-1} \mu_{i}(\alpha)~M_{i,j}\cdot \tau_j(X)
$$

Clearly, $d(X)$ and the bivariate polynomial $M(X, Y)$ should satisfy the following relationship:

$$
d(Y) = M(\alpha, Y)
$$

The right side of the above formula is equivalent to: the "partial operation" (Partial Evaluation) of the bivariate polynomial $M(X, Y)$ at $X=\alpha$.

With this auxiliary polynomial $d(X)$, we can reduce the Lookup Argument proof target to three sub-goals:

- **Sub-goal One**: Prove that $\vec{d}$ is indeed a random sampling of the row space of matrix $M$, i.e., $d(X) = M(\alpha, X)$

- **Sub-goal Two**: Prove that each row of matrix $M$ is a Unit Vector, i.e., has only one $1$

- **Sub-goal Three**: Prove that vector $\vec{d}$ satisfies the query relation: $\vec{d}\cdot \vec{t} = a(\alpha)$

First, look at **Sub-goal One**. To prove the legitimacy of $\vec{d}$, we need to prove $d(X) = M(\alpha, X)$. According to the Schwartz-Zippel Lemma, this is equivalent to proving:

$$
d(\beta) = M(\alpha, \beta)
$$

Here $\beta$ is another random challenge value provided by the Verifier. The problem is how we can avoid letting the Prover directly calculate the commitment of the bivariate polynomial $M(X, Y)$, because its calculation is $O(m\cdot n)$. This requires us to use the sparsity of matrix $M$, because it only has $m$ $1$s. This is very related to the above **Sub-goal Two**.

Considering that the $M$ matrix is composed of several Unit Vectors, therefore we can use this sparsity to redefine the $M$ polynomial:

$$
M(X, Y) = \sum_{i=0}^{m-1} \mu_{i}(X) \cdot \tau_{\color{red} \mathsf{col}(i)}(Y)
$$

This definition only sums up those matrix cells in $M$ that are $1$, since each row has only one $1$, so we only need to consider which column the $1$ in each row appears. We introduce a function $\mathsf{col}(i)$, which indicates the column number where the $i$th $1$ is located in the matrix, obviously

$$
\mathsf{col}(i)\

in [0,n),\quad \forall i\in[0,m)
$$

Here $\tau_{\mathsf{col}(i)}(Y)$ can be explained as: in the $i$th row, only the position $c=\mathsf{col}(i)$ is $1$, and the rest are $0$, so only $\tau_c(Y)$ is selected, and the rest of the Lagrange polynomials $\tau_{j, j\neq{}c}(Y)$ multiplied by zero, therefore can be eliminated.

For a simple example: suppose the table is $t=(1,2,3,4,5,6)^{\top}$, and the query record is $a=(2,4,6)^{\top}$,

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

We see the $1$ in the first row ($i=0$) is in column $1$, so $\mathsf{col}(0)=1$. The $1$ in the second row ($i=1$) is in column $3$, so $\mathsf{col}(1)=3$. Similarly, $\mathsf{col}(2)=5$, therefore the matrix $M$ can be defined sparsely as follows:

$$
M(X, Y) = \mu_0(X)\cdot \tau_{\color{red}1}(Y) + \mu_1(X)\cdot \tau_{\color{red}3}(Y) + \mu_2(X)\cdot \tau_{\color{red}5}(Y)
$$

Then,

$$
d(X) = M(\alpha, X) = \mu_0(\alpha)\cdot \tau_1(X) + \mu_1(\alpha)\cdot \tau_3(X) + \mu_2(\alpha)\cdot \tau_5(X)
$$

If we can prove that the definition of $M(X,Y)$ is $\sum \mu_{i}(X) \cdot \tau_j(Y)$ in this form, then we are essentially proving **Sub-goal Two**, i.e., each row of $M$ is a Unit Vector. Actually, we can slightly weaken Sub-goal Two, as long as we can prove that the definition of $d(X)$ satisfies $\sum \mu_{i}(\alpha) \cdot \tau_j(X)$.

Then back to the question, how do we prove that the matrix $M(\alpha, X)$ is equivalent to $\sum_i \mu_{i}(\alpha) \cdot \tau_{\mathsf{col}(i)}(X)$?

Note that the definition of $M(\alpha, X)$ cannot be seen as an interpolation polynomial of $\{\mu_i(\alpha)\}_i$ over $\mathbb{H}$, because:

1. Each row of matrix $M$ does not cover the entire $\vec{t}$, so $\{\tau_{\mathsf{col}(i)}(X)\}_i$ is only part of the collection of Lagrange polynomials on $\mathbb{H}$,

2. Two rows of matrix $M$ may select the same element in $\vec{t}$, so $\{\tau_{\mathsf{col}(i)}(X)\}_i$ will have duplicate elements.

However, if we allow swapping $\alpha$ and $X$, then $M(X, \alpha)=\sum \tau_{\mathsf{col}(i)}(\alpha)\cdot \mu_{i}(X)$ can indeed be seen as an interpolation polynomial of $\tau_{\mathsf{col}(i)}(\alpha)$ over $\mathbb{V}$. To avoid reusing the Verifier's random number $\alpha$, we let the Verifier provide another fresh random number $\beta$, then we can use the symmetry of the interpolation polynomial to prove $M(X, Y)$ satisfies $M(X, \beta)=\sum \tau_{\mathsf{col}(i)}(\beta)\cdot \mu_{i}(X)$.

Here we need to introduce another auxiliary vector $\vec{e}$, with length $m$:

$$
e_i = \tau_{\mathsf{col}(i)}(\beta), \quad i\in[0, m)
$$

This vector $\vec{e}$ is exactly the random sampling of the column space

 of matrix $M$:

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

Encoding the auxiliary vector $\vec{e}$ into the polynomial $e(X)$, we can obtain:

$$
e(X)= \sum_{i=0}^{m-1} \tau_{\mathsf{col}(i)}(\beta)\cdot \mu_{i}(X) = M(X, \beta)
$$

If $\vec{e}$ is correct, then the following equation must hold:

$$
d(\beta) = M(\alpha, \beta) = e(\alpha)
$$

### 2.2 Column Space Sampling

Here $e(X)$ is also a partial operation (Partial Evaluation) of $M(X, Y)$, only about another variable, namely $Y=\beta$. It can also be seen as the random sampling of the matrix $M$ in the column space $\vec{e}$, obtained again through Lagrange interpolation. And by constraining the equation $d(\beta) = e(\alpha)$, it can ensure that the random sampling of $M$'s row space and column space is consistent, that is, if $\vec{e}$ is the correct column space sampling of the matrix, then $\vec{d}$ is also the correct row space sampling of the same matrix. Therefore, as long as we can prove the correctness of $e(X)$, along with proving $d(\beta) = e(\alpha)$, then we can prove the correctness of $d(X)$ (i.e., the above **Sub-goal One**). If we can also prove that the elements in $\vec{e}$ are all of the form $\tau_{\mathsf{col}(i)}$, then we can prove the special nature of matrix $M$ (i.e., the above **Sub-goal Two**).

The problem has not been solved yet, we just found a path, how to reduce the proof of the correctness of $\vec{d}$ to the proof of the correctness of $\vec{e}$. Next, how do we use the symmetry of Lagrange interpolation polynomials to prove the correctness of $e(X)$?

Observe the definition of $e(X)$:

$$
\begin{split}
e_i &= \tau_{\mathsf{col}(i)}(\beta)\\
 &=  \frac{\omega_{\mathsf{col}(i)}}{m}\cdot\frac{z_\mathbb{H}(\beta)}{\beta - \omega_{\mathsf{col}(i)}} \\
 &= \frac{z_\mathbb{H}(\beta)}{m}\cdot \frac{\omega_{\mathsf{col}(i)}}{\beta - \omega_{\mathsf{col}(i)}} \\
&= \frac{z_\mathbb{H}(\beta)}{m}\cdot \Big(\frac{1}{\beta\cdot\omega^{-1}_{\mathsf{col}(i)}-1}\Big) \\
 \end{split}
$$

If we consider introducing an auxiliary vector $\vec{v}$, where element $v_i=\omega^{-1}_{\mathsf{col}(i)}$,

Then we can obtain the following equation:

$$
m\cdot e_i\cdot (\beta\cdot v_i-1) - z_\mathbb{H}(\beta) = 0, \quad i\in[0, m)
$$

We define the interpolation polynomial of $\vec{v}$ as $v(X) = \sum_{i=0}^{m-1}\omega^{-1}_{\mathsf{col}(i)}\cdot \mu_i(X)$, then we can get the relationship between $e(X)$ and $v(X)$:

$$
 e(X)\cdot(\beta\cdot v(X) -1 ) - \frac{z_\mathbb{H}(\beta)}{m}= q(X)\cdot z_\mathbb{V}(X)
$$

In fact, **$\vec{v}$ is exactly the encoding of matrix $M$**, and also expresses the special nature of matrix $M$. Because each row of matrix $M$ only has one $1$, with the rest being zeros, so there are only $m$ $1$s in all rows, this encoding method is key to proving **Sub-goal Two**, i.e., if the $M$ matrix does not satisfy this special nature, $\vec{v}$ cannot be correctly constructed.

Any matrix $M$ corresponds to a unique $\vec{v}$. Therefore, the above equation is actually proving that $\vec{e}$ is the random sampling of the column space of matrix $M$.

However, we still need an important constraint for $e(X)$:

$$
\deg(e(X))< m
$$

This degree bound proof actually serves to ensure $e(X)=\sum_i e_i\cdot \mu_i(X)$, combined with the above relationship equation between $e(X)$ and $v(X)$, we can draw the following conclusion:

$$
e(X)=\sum_i\Big(\frac{z_\mathbb{H}(\beta)}{m}\cdot \frac{1}{\beta\cdot v_i - 1}\Big)\cdot \mu_i(X)
$$

If we add the proof that $e(\alpha)\overset{?}{=}d(\beta)$, we can conversely ensure that $\vec{d}$ is indeed a random sampling of the row space of matrix $M$. However, why does this conclusion hold? Let's derive the process here.

### 2.3 Consistency of Row Space and Column Space

Above, we have derived the conclusion:

$$
e(X)=\sum_i\frac{z_\mathbb{H}(\beta)}{m}\cdot \frac{1}{\beta\cdot v_i - 1}\cdot \mu_i(X)
$$

Then if we can also prove the "consistency of row and column spaces": $e(\alpha)=d(\beta)$, then from the above conclusion we can derive:

$$
e(\alpha)=\sum_i\frac{z_\mathbb{H}(\beta)}{m}\cdot \frac{1}{\beta\cdot v_i - 1}\cdot \mu_i(\alpha) = d(\beta)
$$

Then, by the Schwartz-Zippel theorem, we regard the above conclusion as a polynomial test at $X=\beta$, then we can reach the following conclusion:

$$
\begin{split}
d(X) &= \sum_i\frac{1}{m}\cdot \frac{z_\mathbb{H}(X)}{X\cdot v_i - 1}\cdot \mu_i(\alpha) \\
&= \sum_i\frac{v_i^{-1}\cdot \mu_i(\alpha)}{m}\cdot \frac{z_\mathbb{H}(X)}{X - v_i^{-1}} \\
&= \sum_i {c_i}\cdot \frac{z_\mathbb{H}(X)}{\color{red} X - v_i^{-1}}
\end{split}
$$

Next, we argue that: $v_i^{-1}\in\mathbb{H}$.

Clearly, the right side of the above $d(X)$ equation is a linear combination of "rational functions" (Rational Function). If $v_i^{-1}\not\in\mathbb{H}$, then $(X- v_i^{-1})$ cannot divide $z_\mathbb{H}(X)$, then $d(X)$ is not a polynomial, i.e., $d(X)\not\in \mathbb{F}_p[X]$. Therefore, we cannot calculate the polynomial commitment of $d(X)$.

Therefore, we only need to require the Prover to provide the commitment of $d(X)$ to get the conclusion $v_i^{-1}\in\mathbb{H}$, thereby proving that every term in the polynomial (commitment) provided by the Prover, $v(X)$, is of the form $v_i = \omega^{-1}_j$.

Next, we can derive that $d(X)$ satisfies the following relationship:

$$
d(X) = \sum_{i=0}^{m-1}\Big(\frac{{\color{red}\omega_j}}{m}\cdot \frac{z_\mathbb{H}(X)}{X - {\color{red}\omega_j}}\Big)\cdot \mu_i(\alpha) = \sum_{i=0}^{m-1}\tau_j(X)\cdot\mu_i(\alpha)
$$

This proves that the vector $\vec{d}$ encoded by $d(X)$ is indeed a random sampling of the row space of some matrix $M$.

### 2.4 CP-expansion

Thus far, **Sub-goal One** and **Sub-goal Two** have been proven. This proof protocol is called the CP-expansion protocol.

To recap, we start from an auxiliary vector $\vec{e}$ (column space sampling of the matrix), proving that $\vec{e}$ and another auxiliary vector, the sparse encoding vector of matrix $M$, $\vec{v}$, satisfy the special relationship of the matrix; then by proving $d(\beta)=e(\alpha)$, verify the legitimacy of the elements in $\vec{v}$, and that the vector encoded by $d(X)$ is a legitimate random sampling of the row space of matrix $M$.

From the definition, $d(X)$ and $e(X)$ have a certain symmetry, and the CP-expansion protocol leverages this symmetry:

$$
e(X) = M(X, \beta) = \sum_{i=0}^{m-1} \tau_{\mathsf{col}(i)}(\beta)\cdot \mu_{i}(X)
$$

$$
d(X) = M(\alpha, X) = \sum_{i=0}^{m-1} \tau_{\mathsf{col}(i)}(X)\cdot \mu_{i}(\alpha)
$$

Due to the special nature of $M$ (each row is a Unit Vector, i.e., each row has only one non-zero element $1$),
Lagrange polynomials also precisely encode a set of unit vectors, so $e_i$ (corresponding to the $i$th row of the matrix) is precisely a random sampling of a Lagrange polynomial $\tau_{\mathsf{col}(i)}(X)$. While $d_i$ is the $i$th column of the matrix, it is precisely a random linear combination of multiple (possibly repeating) Lagrange polynomials $\tau_{\mathsf{col}(i)}(X)$.

Unlike general Matrix-vector Multiplication Argument protocols, here the matrix $M$ is committed in a hidden manner by the prover, which can be considered a Commit-and-prove method. Therefore, we add `CP-` at the beginning of the protocol. Here are the interaction details of the CP-expansion protocol.

**Public Inputs**:

1. Polynomial commitment of the encoding of matrix $M$, $\vec{v}$: $[v(X)]_1$
2. Verifier-provided random number for row space sampling, $\alpha\in\mathbb{F}^*_p$
3. Polynomial commitment of the row space sampling vector of matrix $M$, $\vec{d}$: $[d(X)]_1$,

**Proof Goals**:

1. $d(X) = \sum_{i=0}^{m-1}\tau_{\mathsf{col}(i)}(X)\cdot\mu_i(\alpha)$
2. $v(X) = \sum_{i=0}^{m-1}\omega^{-1}_{\mathsf{col}(i)}(X)\cdot\mu_i(X)$

**Step One**: Prover calculates $e(X)$, $q(X)$, and sends $\big([e(X)]_1, [q(X)]_1, [X^{D-n+1}\cdot e(X)]_1\big)$

$$
e(X)\cdot(\beta\cdot v(X) -1) - \frac{z_\mathbb{H}(\beta)}{m} = z_\mathbb{V(X)}\cdot q(X)
$$

**Step Two**: Verifier sends a random challenge value $\zeta\in\mathbb{F}$

**Step Three**: Prover calculates $P(X)$, and sends $\Big(v_1=e(\zeta), v_2=v(\zeta), v_3=q(\zeta) \Big)$

**Step Four**: Verifier sends a random challenge value $\gamma\in\mathbb{F}$

**Step Five**: Prover calculates and sends $[w(X)]_1$

$$
w(X) = \frac{e(X) - v_1}{X-\zeta} + \gamma\cdot \frac{v(X) - v_2}{X-\zeta} + \gamma^2\cdot \frac{q(X) - v_3}{X-\zeta}
$$

**Verification**: Verifier verifies whether the following two equations hold:

$$
[w(X)]_1\ast[X]_2 \overset{?}{=} \Big([e(X)]_1 + \gamma[v(X)]_1 + \gamma^2[q(X)]_1  + \zeta[w(X)]_1 - (v_1+\gamma v_2+ \gamma^2 v_3)\cdot[1]_1\Big)\ast [1]_2
$$

$$
[X^{D-n+1}\cdot e(X)]_1 \ast [1]_2 \overset{?}{=} [e(X)]_1 \ast [X^{D-n+1}]_2
$$

## 3. Univariate Sumcheck

The final **Sub-goal Three** is a Dot Product proof, specifically proving that $\vec{d} \cdot \vec{t} = a(\alpha)$. There are several methods to prove a dot product, such as the Grand Sum Argument used in Flookup, while Baloo uses a scheme based on the Univariate Sumcheck.

The Univariate Sumcheck implies that for any polynomial $f(X) \in \mathbb{F}[X]$, if there exists an FFT-smooth multiplicative subgroup $\mathbb{H} \subset \mathbb{F}_p^*$ with $|\mathbb{H}| = n$, then the following equation holds:

$$
f(X) = \frac{\sigma}{n} + X \cdot g(X) + z_\mathbb{H}(X) \cdot q(X), \qquad \deg(g(X)) < n-1
$$

where $\sigma$ is the sum of polynomial evaluations over $\mathbb{H}$, defined as:

$$
\sigma = \sum_{i=0}^{n-1} f(\omega^i)
$$

We can restate this equation: the sum of $f(X)$ over $\mathbb{H}$ is exactly equal to the "constant term" of the remainder polynomial of $f(X)$ divided by $z_\mathbb{H}(X)$, multiplied by the size of $\mathbb{H}$.

$$
\sigma = \sum_{i=0}^{n-1} f(\omega^i) = n \cdot (f(X) - X \cdot g(X) - z_\mathbb{H}(X) \cdot q(X))
$$

Why does this equation hold? Let's derive this magical relationship:

Assume there is a polynomial $f(X)$ of any degree. When divided by $z_\mathbb{H}(X)$, the remainder polynomial $r(X)$ will have a degree less than the size of $\mathbb{H}$. Polynomial division is similar to integer division; just as $13 \equiv 3 \pmod{5}$ means $13 = 5*2 + 3$ with remainder $3 < 5$, any polynomial $f(X)$ satisfies $f(X) = q(X) \cdot z_\mathbb{H}(X) + r(X)$, where $\deg(r(X)) < \deg(z_\mathbb{H}(X)) = n$.

Therefore:

$$
r(X) = r_0 + r_1 \cdot X + r_2 \cdot X^2 + \cdots r_{n-1} \cdot X^{n-1} = \sum_{j=0}^{n-1} r_j \cdot X^j
$$

The remainder polynomial $r(X)$ maintains the same evaluation results as $f(X)$ over $\mathbb{H}$:

$$
\forall \omega_i \in \mathbb{H}, f(\omega_i) = r(\omega_i)
$$

This conclusion is easy to prove because $f(X) = q(X) \cdot z_\mathbb{H}(X) + r(X)$, and for any $\omega_i \in \mathbb{H}$, $f(\omega_i) = q(\omega_i) \cdot z_\mathbb{H}(\omega_i) + r(\omega_i)$ necessarily holds, and since $z_\mathbb{H}(\omega_i) = 0$, we have $f(\omega_i) = r(\omega_i)$.

Next, we substitute the definition of $r(X)$ into the summation formula to verify the Univariate Sumcheck equation:

$$
\sigma = \sum_{i=0}^{n-1} f(\omega^i) = n \cdot r_0
$$

In the derivation above, the general conclusion $\sum_{i=0}^{n-1} \omega^i = 0$ is used. This conclusion can be quickly proven by defining a polynomial $h(X)$ as follows, satisfying $h(X) \cdot (X-1) = X^n-1$,

$$
h(X) = 1 + X + X^2 + X^3 + \cdots + X^{n-1} = \frac{X^n-1}{X-1}
$$

Substituting $X = \omega$ into the definition of $h(X)$ and using the fact that $\omega$ is a root of unity, we get:

$$
h(\omega) = \sum_{i=0}^{n-1} \omega^i = \frac{\omega^n-1}{\omega-1} = \frac{1-1}{\omega-1} = 0
$$

Further, when $r(X)$ is subtracted by $r_0$, $r(X)$ can be divided by $X$, therefore:

$$
r(X) - r_0 = X \cdot g(X), \qquad \deg(g(X)) < n-1
$$

Substituting $r_0 = \sigma/n$, we get the Univariate Sumcheck equation:

$$
f(X) \equiv r(X) = \frac{\sigma}{n} + X \cdot g(X) \pmod{z_\mathbb{H}(X)}
$$

This leads to:

$$
f(X) = \frac{\sigma}{n} + X \cdot g(X) + z_\mathbb{H(X)} \cdot q(X), \qquad \deg(r(X)) < n-1
$$

### 3.1 Dot Product Protocol

Based on the Univariate Sumcheck theorem, we present a simplified Dot Product protocol to prove the dot product of two vector elements.

**Public Inputs**:

1. Vector commitments $[d(X)]_1$, $[t(X)]_1$, assuming both vectors $\vec{d}$ and $\vec{t}$ have length $n$
2. Dot product value $\sigma$

**Proof Goal**:

$$
\vec{d} \cdot \vec{t} = \sigma
$$

**Step One**: Prover calculates $r(X)$, $q(X)$, and sends $\big([r(X)]_1, [q(X)]_1, [X^{D-n+2} \cdot r(X)]_1\big)$

$$
d(X) \cdot t(X) = \frac{\sigma}{n} + X \cdot r(X) + z_\mathbb{H(X)} \cdot q(X)
$$

**Step Two**: Verifier sends a random challenge value $\zeta \in \mathbb{F}$

**Step Three**: Prover sends $\Big(v_1 = d(\zeta), v_2 = t(\zeta), v_3 = r(\zeta), v_4 = q(\zeta) \Big)$

**Step Four**: Verifier sends a random challenge value $\gamma \in \mathbb{F}$

**Step Five**: Prover calculates and sends $[w(X)]_1$

$$
w(X) = \frac{d(X) - v_1}{X-\zeta} + \gamma \cdot \frac{t(X) - v_2}{X-\zeta} + \gamma^2 \cdot \frac{r(X) - v_3}{X-\zeta} + \gamma^3 \cdot \frac{q(X) - v_4}{X-\zeta}
$$

**Verification**: Verifier

1. Computes $C_p = [d(X)]_1 + \gamma [t(X)]_1 + \gamma^2 [r(X)]_1 + \gamma^3 [q(X)]_1$
2. Computes $C_v = \big(v_1 + \gamma \cdot v_2 + \gamma^2 \cdot v_3 + \gamma^3 \cdot v_4\big) \cdot [1]_1$
3. Verifies the aggregated evaluation proof:

$$
[w(X)]_1 \ast [X]_2 \overset{?}{=} \Big(C_p + \zeta [w(X)]_1 - C_v \Big) \ast [1]_2
$$

4. Verifies the degree bound of $r(X)$

$$
[X^{D-n+2} \cdot r(X)]_1 \ast [1]_2 \overset{?}{=} [r(X)]_1 \ast [X^{D-n+2}]_2
$$

## 4. Simplified Protocol Framework

Thus far, we have established a simplified Baloo proving framework:

**Public Inputs**:
1. Commitment to the table vector $\vec{t}$, $[t(X)]_1$, table length $n$
2. Commitment to the query vector $\vec{a}$, $[a(X)]_1$, query length $m$

**Proof Goal**:

$$
\exists M \in \mathbb{F}^{m \times n}, \quad M\vec{t} = \vec{a}
$$

**Sub-protocol One**: CP-expansion Protocol

The Prover provides the Verifier with a commitment to the row space sampling of matrix $M(\alpha, X)$, $[d(X)]_1$, and proves its correctness. In essence, this is equivalent to proving the existence of a matrix $M$ composed of "row unit vectors" and that its row space sampling commitment is $[d(X)]_1$.

The proving approach is for the Prover to provide the matrix's encoding vector $\vec{v}$ and the column space sampling $\vec{e}$ commitments $[v(X)]_1$ and $[e(X)]_1$, and to prove:

$$
e(X) \cdot (\beta \cdot v(X) - 1) - \frac{z_\mathbb H(\beta)}{m} = z_\mathbb{V(X)} \cdot q_1(X)
$$

$$
\deg(e(X)) < m
$$

$$
d(\beta) = e(\alpha)
$$

**Sub-protocol Two**: Dot Product Protocol

The Prover proves to the Verifier that the vector $\vec{d}$ encoded by $d(X)$ and the sub-table vector $\vec{t}$ have a dot product equal to $a(\alpha)$, the sampling of the query polynomial at $X=\alpha$. This is equivalent to proving the correctness of the Matrix-vector multiplication required by the proof goal.

$$
d(X) \cdot t(X) - \left(\frac{a(\alpha)}{n}\right) - X \cdot g(X) = q_2(X) \cdot z_\mathbb{H(X)}
$$