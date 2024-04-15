# Baloo, lookup (Part 3)

### 9. Baloo Protocol and Optimizations

Now, we integrate the Subtable, Dot Product, and CP-expansion sub-protocols into a complete Baloo protocol.

**Public Inputs**

1. Structured public reference string $SRS=\{[x^s]_1, [x^s]_2\}_{i=0}^{D}$
2. Polynomial commitment of the main table vector $T=[t(x)]_1$
3. Polynomial commitment of the query vector $A=[a(x)]_1$

**Precomputations**

1. $\{[q_i(X)]_1 \}_{i\in[N]}$, where $q_i(X)\cdot (X-\omega_i) = t(X) - t(\omega_i)$
2. $\{[w_i(X)]_1\}_{i\in[N]}$, where $w_i(X)\cdot (X-\omega_i) = z_\mathbb{H}(X)$

**Step One**

1. Based on query records, the Prover calculates the polynomial of the subtable vector $t_I(X)$ and sends its commitment $[t_I(x)]_1$.

2. The Prover calculates the interpolation polynomial of the query record index vector $z_I(X)$ and sends the commitments $[z_I(x)]_1$ and $[z_I(x)]_2$.

3. The Prover calculates and commits to the polynomial of the matrix $M$'s vector $\vec{v}$ as $[v(x)]_1$.

**Step Two** The Verifier sends a random number $\alpha\in\mathbb{F}$.

**Step Three** The Prover sends $a(\alpha)$ along with $[q_a(x)]_1$

$$
\mathsf{EQ0}: q_a(X)\cdot (X-\alpha) = a(X) - a(\alpha)
$$

**Step Four** Through the Subtable sub-protocol, the Prover and Verifier prove that $\vec{t}_I\subset\vec{t}$.

**Step Five** The Prover calculates the row space sampling of matrix $M$ as $\vec{d}$ and sends the polynomial commitment $[d(x)]_1$.

**Step Six** Through the Dot-Product sub-protocol, the Prover and Verifier prove that $\vec{d}\cdot \vec{t}_I=a(\alpha)$

1. The Prover calculates $q_1(X)$

$$
\mathsf{EQ1}: d(X)\cdot t_I(X) - a(\alpha) - r(X) = q_1(X)\cdot z_I(X)
$$

**Step Seven** Through the CP-expansion sub-protocol, the Prover and Verifier prove that $\vec{d}=\sum_{i=0}^{m-1}\mu_i(\alpha)\cdot \vec{M}_i$

1. The Prover calculates $e(X)$.
2. The Prover calculates $q_2(X)$.

$$
\mathsf{EQ2}: e(\alpha) = d(\beta)
$$

$$
\mathsf{EQ3}: e(X)(\beta - v(X)) + z_I(\beta)z_I(0)^{-1}\cdot v(X) = z_\mathbb{V}(X) \cdot q_2(X)
$$

**Step Eight**

1. The Verifier verifies the consistency between $[z_I(x)]_1$ and $[z_I(x)]_2$.

$$
[z_I(x)]_1 \ast [1]_2 \overset{?}{=}[1]_1 \ast [z_I(x)]_2
$$

2. The Verifier verifies the correctness of $a(\alpha)$:

$$
\big(A - a(\alpha)\cdot[1]_1\big)\ast [1]_2 \overset{?}{=}  [q_a(x)]_1 \ast \big([x]_2 - \alpha\cdot[1]_2\big)
$$

Executing the three sub-protocols in series can result in a large proof size. We can utilize various proof aggregation techniques to combine multiple polynomial evaluation proofs and degree-bound proofs to reduce the proof size.

### 9.1 Polynomial Constraint Linearization

When dealing with polynomial constraints of quadratic or higher degree, we can fully utilize the "additive homomorphism" of polynomial commitments to reduce the number of polynomial evaluations. For example, in the CP-expansion, we have the following equation:

$$
\mathsf{EQ3}:  e(X)(\beta - v(X)) + z_I(\beta)z_I(0)^{-1}\cdot v(X) = z_\mathbb{V(X)} \cdot q_2(X)
$$

To verify the above equation, the most straightforward and brutal method would be to open all polynomials at a random point $X=\zeta$, and then verify that the polynomial values satisfy the equation. However, this would require the Prover to send the values $e(\zeta)$, $v(\zeta)$, and $q_1(\zeta)$, leading to a large proof size.

We do not need to open all polynomials in the equation; in fact, we only need to open a few. We can then transform the equation into a linear combination of polynomials. For example, the equation only needs to open $e(\zeta)$, keep $v(X)$ and $q_2(X)$, and the equation can be linearized as:

$$
e(\zeta)\cdot(\beta - v(X)) + z_I(\beta)z_I(0)^{-1}\cdot v(X) = z_\mathbb{V(\zeta)} \cdot q_1(X)
$$

Here, $z_\mathbb{V(\zeta)}$ is computed by the Verifier. Thus, we obtain a new polynomial, denoted as $p_1(X)$:

$$
p_1(X) = e(\zeta)\cdot(\beta - v(X)) + z_I(\beta)z_I(0)^{-1}\cdot v(X) - z_\mathbb{V(\zeta)} \cdot q_1(X)
$$

If $\mathsf{EQ3}$ holds, then $p_1(X)$ equals $0$ at $X=\zeta$. Thus, we only need to prove $e(\zeta)$ and $p_1(\zeta)=0$. Originally, proving the equation required opening three polynomials, but now only two need to be opened, and since $p_1(\zeta)$ is zero, this value does not need to be sent by the Prover.

The second polynomial constraint that can be linearized is:

$$
\mathsf{EQ1}: d(X)\cdot t_I(X) - a(\alpha) - r(X) = q_2(X)\cdot z_I(X)
$$

Here we can reuse $X=\beta$ as the challenge point, allowing the Prover to send $[d(x)]_1$, $[r(x)]_1$, and $[q_2(x)]_1$ before the Verifier sends $\beta$. The Prover only needs to send $d(\beta)$ and $z_I(\beta)$, thus linearizing the equation into polynomial $p_2(X)$. The Prover only needs to prove $d(\beta)$, $z_I(\beta)$, and $p_2(\beta)=0$.

$$
p_2(X) = d(\beta)\cdot t_I(X) - a(\alpha) - r(X) - z_I(\beta) \cdot q_2(X)
$$

### 9.2 Aggregation of Polynomial Evaluations

In the constraint equations $\mathsf{EQ0}$ and $\mathsf{EQ2}$, evaluations at $a(\alpha)$ and $e(\alpha)$ are required, so we can aggregate the evaluations of $a(X)$ and $e(X)$ at $X=\alpha$.

In the CP-expansion sub-protocol, when proving the Generalized Sumcheck equation ($\mathsf{EQ1}$), we can use the random number $\beta$ as the evaluation point instead of $\zeta$, which avoids opening $z_I(X)$ at both $X=\beta$ and $X=\zeta$.

### 9.3 Merging of Degree-bound Proofs

The first degree-bound proof relates to the equation $\mathsf{EQ1}$ and the remainder polynomial $r(X)=Xg(X)$, where the degree of $g(X)$ should be strictly less than $m-1$. This proof can actually be broken down into two proof sub-goals:

1. $r(0)=0$
2. $\deg(r(X))<m$

This approach is not the usual practice in Univariate Sumcheck. Normally, the practice is to prove $\deg(g(X)) < m-1$. We use the approach $\deg(X\cdot g(X)) < m$ to aggregate the constraint $r(0)=0$ with other polynomial evaluations opened at $X=0$ (such as $z_I(X)$). Additionally, both $r(X)$ and $z_I(X)$ require degree-bound proofs, so they can ideally be aggregated in a single constraint.

The second sub-goal can be merged with the second degree-bound proof $\deg(z_I(X) - X^m)<m$.

The third degree-bound proof is $\deg(e(X)) < m$

### 9.4 Optimized Protocol Details

Here are the detailed steps of the optimized protocol along with notes:

**Public Inputs**:

1. The commitment of the main table vector $C_T=[t(x)]_1$,
2. The commitment of the query vector $C_a=[a(x)]_1$,
3. The commitment of the vanishing polynomial on $\mathbb{H}$, $C_{z_\mathbb{H}}=[z_\mathbb{H}(x)]_1$.

**Step 1**: The Prover computes and sends the commitments of three polynomials: $\big(C_v = [v(x)]_1, C_{z_I}=[z_I(x)]_2, C_t=[t_I(x)]_1\big)$

1. Compute and commit the vanishing polynomial $z_I(X)$ on $\mathbb{H}_I$.
2. Compute and commit $v(X)=\sum_{i=0}^{m-1}\omega_{\mathsf{col}(i)}\cdot \mu_i(X)$.
3. Compute and commit the polynomial of the sub-table vector $t_I(X)$.

**Step 2**: The Verifier sends a random challenge number $\alpha$, used for the random sampling of the matrix $M$'s row space.

**Step 3**: The Prover computes and sends $\big(C_d = [d(X)]_1, C_r=[r(X)]_1, C_{q_1} = [q_1(X)]_1 \big)$

1. Compute and commit $d(X)=M(\alpha, X)=\sum_{i=0}^{m-1}\mu_i(\alpha)\cdot \hat\tau_{\mathrm{col}(i)}(X)$.
2. Compute and commit $r(X)$ and $q_1(X)$, satisfying the following relation:

$$
\mathsf{EQ1}: d(X)\cdot t_I(X) - a(\alpha) - r(X) = q_1(X)\cdot z_I(X)
$$

3. Ensure $r(X)$ also satisfies $\deg(r(X)) < m$ and $r(0)=0$.

**Step 4**: The Verifier sends another random challenge number $\beta$ to the Prover, used for the random sampling of the matrix $M$'s column space.

**Step 5**: The Prover computes and sends $\big(C_e = [e(x)]_1, C_{q_2}=[q_2(x)]_1 \big)$

1. $e(X)=M(X, \beta)=\sum_{i=0}^{m-1}\mu_i(X)\cdot \hat\tau_{\mathrm{col}(i)}(\beta)$, satisfying the following equation relation:

$$
\mathsf{EQ2}: e(\alpha) = d(\beta)
$$

2. $q_2(X)$ satisfies the following relation:

$$
\mathsf{EQ3}: e(X)(\beta - v(X)) + z_I(\beta)z_I(0)^{-1}\cdot v(X) = z_\mathbb{V(X)} \cdot q_2(X)
$$

**Step 6**: The Verifier sends another random challenge number $\zeta$ to the Prover, used to challenge the correctness of the $\mathsf{EQ3}$ equation.

**Step 7**:

1. The Prover computes and sends $\big(v_1=e(\alpha),v_2=a(\alpha),v_3=z_I(0),v_4=z_I(\beta),v_5=e(\zeta))$
2. The Prover calculates the linearized polynomial $p_1(X)$, which equals zero at $X=\beta$:

$$
p_1(X) = d(\beta)\cdot t_I(X) - a(\alpha) - r(X) - z_I(\beta)\cdot q_1(X)
$$

3. The Prover calculates the linearized polynomial $p_2(X)$, which equals zero at $X=\zeta$:

$$
p_2(X) = e(\zeta)(\beta -v(X)) + z_I(\beta)z_I(0)^{-1}\cdot v(X) - z_\mathbb{V(\zeta)} \cdot q_2(X)
$$

**Step 8**: The Verifier sends another random challenge number $\gamma$ to the Prover, used for aggregated proof.

**Step 9**: The Prover completes the following calculations and sends the proofs

1. Aggregate $v_1=e(\alpha)$ and $v_2=a(\alpha)$, two evaluations opened at $X=\alpha$, and cleverly aggregate the $\deg(e(X))<m$ degree bound proof. This also constrains the degree of $a(X)$, though not required by the protocol.

$$
w_1(X) = X^{D-m+1}\cdot \left(\frac{e(X) - v_1}{X-\alpha} + \gamma \cdot \frac{a(X)-v_2}{X-\alpha}\right)
$$

2. Aggregate $v_3=z_I(0)$, $r(0)=0$ two evaluation proofs, and $\deg(r(X))<m$ and $\deg(z_I(X) - X^m)<m$ two degree bound proofs. The latter constrains the sub-table size to strictly equal $m$.

$$
w_2(X) = \frac{z_I(X) - v_3}{X} + \gamma \cdot \frac{r(X)-0}{X} + X^{D-m+1}\cdot \left(\gamma^2\cdot (z_I(X) - X^m) + \gamma^3 \cdot r(X)\right)
$$

3. Aggregate $v_1 = d(\beta)$, $v_4=z_I(\beta)$, and $p_1(\beta)=0$ three evaluation proofs

$$
w_3(X) = \frac{d(X) - v_1}{X-\beta} + \gamma\cdot \frac{z_I(X) - v_4}{X-\beta} + \gamma^2 \cdot \frac{p_1(X)-0}{X-\beta}
$$

4. Aggregate $e(\zeta)=v_5$, $p_2(\zeta)=0$ two evaluation proofs

$$
w_4(X) = \frac{e(X) - v_5}{X-\zeta} + \gamma \cdot \frac{p_2(X)-0}{X-\zeta}
$$

5. The Prover sends $\big(C_{w_1}=[w_1(x)]_1, C_{w_2}=[w_2(x)]_1, C_{w_3}=[w_3(x)]_1, C_{w_4}=[w_4(x)]_1 \big)$

**Verification**: The Verifier receives the following proof:

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

1. Compute the commitment of the linearized polynomial $C_{p_1} = [p_1(x)]_1$:

$$
C_{p_1} = v_1\cdot C_t - v_2\cdot [1]_1 - C_r - v_4\cdot C_{q_1}
$$

2. Compute the commitment of the linearized polynomial $C_{p_2} = [p_2(x)]_1$:

$$
C_{p_2} = v_5\cdot(\beta\cdot[1]_1 - C_v) + v_4v_3^{-1}\cdot C_v - z_\mathbb{V}(\zeta)\cdot C_{q_2}
$$

The Verifier checks the following Pairing equations:

1. Verify the Subtable equation:

$$
\Big(C_T - C_t + \gamma\cdot C_{z_\mathbb{H}}\Big)\ast [1]_2\overset{?}{=}
\Big(C_{w_5} + \gamma\cdot C_{w_6}\Big) \ast {\color{red}C_{z_I}}
$$

2. Verify the proof opened at $X=\alpha$, $\blue{C_{w_1}}$, according to the simplified polynomial relation:

$$
\begin{split}
w_1(X)\cdot X-\alpha\cdot w_1(X) &= X^{D-m+1

}\cdot \left(e(X) + \gamma a(X)- (v_1 + \gamma v_2\right))\\
\end{split}
$$

The verification equation is:

$$
\Big(\blue{C_{w_1}} \ast [X]_2\Big) \overset{?}{=} \Big([e(X)]_1 + \gamma\cdot[a(X)]_1 - (v_1 + \gamma v_2)\cdot[1]_1\Big)\ast [X^{D-m+1}]_2 + \Big(\alpha\cdot\blue{C_{w_1}}\ast [1]_2 \Big)\ast [1]_2
$$

3. Verify the proof opened at $X=0$, $\blue{C_{w_2}}$, according to:

$$
\begin{split}
w_2(X)\cdot X &= (z_I(X) - v_3) + \gamma \cdot r(X) + X^{D-m+2}\cdot \left(\gamma^2\cdot (z_I(X) - X^m) + \gamma^3 \cdot r(X)\right) \\
&= z_I(X) + X^{D-m+2}\cdot\gamma^2 z_I(X) + \gamma r(X) + X^{D-m+2}\cdot \gamma^3 \cdot r(X) - v_3 - X^{D-m+2}\cdot \gamma^2 \cdot X^m \\
&= (1+\gamma^2\cdot X^{D-m+2})\cdot z_I(X) + (\gamma^3\cdot r(X) - \gamma^2X^m)\cdot X^{D-m+2} + (\gamma\cdot r(X) - v_3) \\
\end{split}
$$

The verification equation is:

$$
\Big(\blue{C_{w_2}} \ast [X]_2\Big) \overset{?}{=}
\Big([1]_1 +\gamma^2\cdot [X^{D-m+2}]_1\Big)\ast \red{C_{z_I}} + \Big(\gamma^3\cdot C_r - \gamma^2\cdot [X^m]_1\Big)\ast [X^{D-m+2}]_2 + \Big(\gamma\cdot C_r - [v_3]_1\Big)\ast[1]_2
$$

4. Verify the proof opened at $X=\beta$, $\blue{C_{w_3}}$, according to:

$$
\begin{split}
w_3(X)\cdot X - \beta\cdot w_3(X) & = (d(X) - v_1) + \gamma\cdot (z_I(X) - v_4) + \gamma^2 \cdot p_1(X) \\
w_3(X)\cdot X & = \Big(d(X) + \beta\cdot w_3(X) - \gamma v_4 - v_1 + \gamma^2 \cdot p_1(X)\Big)  + \gamma \cdot z_I(X) \\
\end{split}
$$

The verification equation is:

$$
\Big(\blue{C_{w_3}} \ast [X]_2\Big) \overset{?}{=}
\Big(C_d + \beta\cdot \blue{C_{w_3}} - \gamma\cdot [v_4]_1 - [v_1]_1 + \gamma^2 \cdot C_{p_1}\Big) \ast [1]_2 + \Big(\gamma\cdot [1]_1\Big) \ast \red{C_{z_I}}
$$

Please note that $[z_I(X)]_2$ is committed in $\mathbb{G}_2$, so it must be separately calculated for Pairing.

5. Verify the aggregated proof at $X=\zeta$, $\blue{C_{w_4}}$, according to the derived formula:

$$
\begin{split}
X\cdot w_4(X)-\zeta\cdot w_4(X) &= (e(X) - v_5) + \gamma \cdot p_2(X)\\
w_4(X)\cdot X & = e(X) + \zeta\cdot w_4(X) + \gamma \cdot p_2(X) - v_5 \\
\end{split}
$$

The verification equation is:

$$
\Big(\blue{C_{w_4}}\ast [X]_2\Big) \overset{?}{=}
\Big(C_e + \zeta\cdot \blue{C_{w_4}} + \gamma \

cdot C_{p_2} - v_5\cdot [1]_1\Big)\ast [1]_2 \\
$$
