---
title: "Under the hood of zkSNARKs — PLONK protocol: part 6"
authorURL: ""
originalURL: https://medium.com/@cryptofairy/under-the-hood-of-zksnarks-plonk-protocol-part-6-5a030d15be68
translator: ""
reviewer: ""
---

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*1EEFOLYL-EE3bGj0mTeMtA.png)

<!-- more -->

# Under the hood of zkSNARKs — PLONK protocol: part 6

[

![Crypto Fairy](https://miro.medium.com/v2/resize:fill:88:88/1*nrbTgZM_zY_Isf4qDqpfjA.png)









][1]

[Crypto Fairy][2]

·

[Follow][3]

10 min read

·

Nov 22, 2023

[

][4]

\--

1

[][5]

Listen

Share

Final article in PLONK series. Here we will implement Prover and the Verifier. In the previous article we coded trusted setup, gate and permutations polynomials, now we are ready to start generating proof.

[

## Under the hood of zkSNARKs — PLONK protocol: part 5

### Previous articles in the PLONK series served as preparations before we got our hands dirty and started implementing the…

medium.com



][6]

## The Prover

In the PLONK protocol, the prover is required to complete five rounds in a specified order. This process results in the creation of a proof comprising 9 Elliptic Curve Points and 6 Finite Field values.

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*AKhVZ2zppnI93rPzC0ymsw.png)

Final implementations also can be found here:

[

## zkSNARK-under-the-hood/plonk\_latest.ipynb at main · tarassh/zkSNARK-under-the-hood

### Implementation of zero knowledge proof protocol - Groth16, Plonk. For education purposes. Not a production ready code…

github.com



][7]

## Round 1

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*Ov5oGoFQeYYfcqMs4UQjAw.png)

In Round 1, we need to compute wire polynomials, using the same technique employed for gate polynomials in the previous article. This technique involves converting vectors into polynomials via Lagrange interpolation. Additionally, it’s necessary to ‘blind’ the wire polynomials. For each of a, b, and c, we must generate blinding polynomials (such as b1x + b2, etc.). The use of blinding polynomials is based on a straightforward rationale. We utilize the KZG commitment scheme at the core, where each polynomial’s evaluation at a random value `zeta` reveals some information about the polynomial, contradicting the principles of zero-knowledge proofs. Hence, for our newly generated polynomials, we incorporate these blinding polynomials. Blinding polynomials are multiplied by the Vanishing polynomial _Zh_ to generate correct polynomial commitments \[a\], \[b\], and \[c\]. Since evaluations occur over the roots of unity, _Zh_ yields 0, making the blinding factor negligible. However, when the polynomials are evaluated at the random value `zeta` in round 4, the blinding polynomials will significantly affect the results.

random\_b = \[Fp.Random() for i in range(0, 9)\]  
  
\# blinding polynomials  
bA = galois.Poly(random\_b\[:2\], field=Fp)  
bB = galois.Poly(random\_b\[2:4\], field=Fp)  
bC = galois.Poly(random\_b\[4:6\], field=Fp)  
  
\# wire polynomials  
\_A = to\_poly(roots, a, Fp)  
\_B = to\_poly(roots, b, Fp)  
\_C = to\_poly(roots, c, Fp)  
  
\# Wire polynomials with blinding factor  
A = \_A + bA\*Zh  
B = \_B + bB\*Zh  
C = \_C + bC\*Zh  
  
\# gate constraints polynomial  
\# g(x) = a(x)\*ql(x) + b(x)\*qr(x) + a(x)\*b(x)\*qm(x) + qc(x) + c(x)\*qo(x)  
G = A\*QL + B\*QR + A\*B\*QM + QC + C\*QO  
  
for i in range(0, len(roots)):  
    assert G(roots\[i\]) == 0, f"G({roots\[i\]}) != 0"  
  
assert G % Zh == 0, f"G(x) % Zh(x) != 0"  
  
\# commitments \[a\], \[b\], \[c\]  
round1 = \[A(tau), B(tau), C(tau)\]

[In Part 3][8], we discussed how to constrain gate polynomials, so the introduction of polynomial G should not be surprising to the reader. Instead of sending a commitment to G, we transmit three commitments because the Q polynomials are also known to the verifier, enabling them to easily reconstruct the commitment to G. Also, G polynomial used to verify if it is constrained correctly — it should equal zero at all roots of unity, indicating that it can be divided by Zh without a remainder. If this assertion fails, it implies that there was an error in the construction of the polynomial.

## Round 2

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*YGorRZIhiCq1dyFZgZX6WA.png)

Beta (β) and gamma (γ) are the first random values generated from a transcript in Round 1 (\[a\], \[b\], \[c\]). [In Part 5][9], I mentioned the Fiat-Shamir heuristic as a mechanism for obtaining random values. [Part 4][10] is dedicated entirely to the permutation polynomial z(x). The blinding polynomials in this context serve the same purpose as they did in Round 1.

  
beta = numbers\_to\_hash(round1 + \[0\], Fp)  
gamma = numbers\_to\_hash(round1 + \[1\], Fp)  
  
\_F = (A + I1 \* beta + gamma) \* (B + I2 \* beta + gamma) \* (C + I3 \* beta + gamma)  
\_G = (A + S1 \* beta + gamma) \* (B + S2 \* beta + gamma) \* (C + S3 \* beta + gamma)  
  
acc\_eval = \[Fp(1)\]  
for i in range(0, n):  
    acc\_eval.append(  
        acc\_eval\[-1\] \* (\_F(roots\[i\]) / \_G(roots\[i\]))  
    )  
assert acc\_eval.pop() == Fp(1)  
ACC = galois.lagrange\_poly(roots, Fp(acc\_eval))  
  
bZ = galois.Poly(random\_b\[6:9\], field=Fp)  
Z = bZ \* Zh + ACC  
  
assert Z(roots\[0\]) == 1  
assert Z(roots\[-1\]) == 1  
  
round2 = \[Z(tau)\]

## Round 3

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*kQeXSIQtQe_tSBxjKX7xeA.png)

In Round 3, we encounter a substantial computational phase. The primary task involves calculating the polynomial t(x), pivotal for proving the entire statement. Let’s break this down:

-   The first line involves the polynomial G from Round 1, divided by Zh(x) — the vanishing polynomials. This division aims to demonstrate that G equals 0 at every evaluation point (roots of unity). Therefore, if G(x)/Zh(x) divides without remainder, it indicates that the prover can provide a correct polynomial commitment for G.
-   Line #4, featuring L1(x), equals 1 for the first element in the roots of unity and 0 for the others. The permutation check formula, detailed in [Part 4][11], begins from `acc0=1`. This necessitates addressing it in the t(x) polynomial.
-   The two middle polynomials can be simplified to f’(X)z(X) — g’(X)z(ωX) = 0 or, equivalently, f’(X)z(X) = g’(X)z(ωX). This simplification is to validate that the permutations are correct.
-   The role of the α value is crucial. Instead of proving that multiple different polynomials are zero at the roots of unity, the protocol combines them as linearly independent terms. This allows their validation together for all these polynomials, as elaborated in [Part 1][12].
-   Splitting the t(X) polynomial into three parts is a protocol trade-off. To calculate the commitment to t(X), the prover would typically require more time compared to other polynomials due to its higher degree. Additionally, this also necessitates a larger size for the trusted setup.

def shift\_poly(poly: galois.Poly, omega: Fp):  
    coeffs = poly.coeffs\[::-1\]  
    coeffs = \[c \* omega\*\*i for i, c in enumerate(coeffs)\]  
    return galois.Poly(coeffs\[::-1\], field=poly.field)  
  
alpha = numbers\_to\_hash(round1 + round2, Fp)  
  
Zomega = shift\_poly(Z, omega=omega)  
  
L1 = galois.lagrange\_poly(roots, Fp(\[1\] + \[Fp(0)\] \* (n - 1)))  
  
for i, r in enumerate(roots):  
    # make sure L1 equal 1 at root\[0\] and 0 for the rest  
    assert L1(r) == (Fp(1) if i == 0 else Fp(0))  
  
T0 = G  
assert T0 % Zh == 0, f"T0(x) % Zh(x) != 0"  
  
T1 = (\_F \* Z - \_G \* Zomega) \* alpha  
assert T1 % Zh == 0, f"T1(x) % Zh(x) != 0"  
  
T2 = (Z - galois.Poly(\[1\], field=Fp)) \* L1 \* alpha\*\*2  
assert T2 % Zh == 0, f"T2(x) % Zh(x) != 0"  
  
T = (T0 + T1 + T2)  
assert T % Zh == 0, f"T(x) % Zh(x) != 0"  
  
for r in roots:  
    assert T(r) == 0, f"T({r}) != 0"  
  
T = T // Zh  
  
t\_coeffs = T.coeffs\[::-1\]  
  
Tl = galois.Poly(t\_coeffs\[:n\]\[::-1\], field=Fp)  
Tm = galois.Poly(t\_coeffs\[n:2\*(n)\]\[::-1\], field=Fp)  
Th = galois.Poly(t\_coeffs\[2\*(n):\]\[::-1\], field=Fp)  
  
X\_n = galois.Poly.Degrees(\[n, 0\], coeffs=\[1, 0\], field=Fp)  
X\_2n = galois.Poly.Degrees(\[2\*(n), 0\], coeffs=\[1, 0\], field=Fp)  
\# make sure that T was split correctly  
\# T = TL + X^n \* TM + X^2n \* TH  
assert T == (Tl + X\_n \* Tm + X\_2n \* Th)  
assert T.degree == 3 \* n + 5  
  
b10 = Fp.Random()  
b11 = Fp.Random()  
  
Tl = Tl + b10 \* X\_n  
Tm = Tm - b10 + b11 \* X\_n  
Th = Th - b11  
assert T == (Tl + X\_n \* Tm + X\_2n \* Th)  
  
round3 = \[Tl(tau), Tm(tau), Th(tau)\]

## Round 4

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*Z5PqGHwMQqk1TWJnYwXrVw.png)

This round is the simplest The prover now needs to evaluate polynomials at a specific random point, zeta.

zeta = numbers\_to\_hash(round1 + round2 + round3, Fp)  
  
a\_zeta = A(zeta)  
b\_zeta = B(zeta)  
c\_zeta = C(zeta)  
s1\_zeta = S1(zeta)  
s2\_zeta = S2(zeta)  
z\_omega\_zeta = Zomega(zeta)  
  
round4 = \[a\_zeta, b\_zeta, c\_zeta, s1\_zeta, s2\_zeta, z\_omega\_zeta\]

**Round 5**

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*bhPHVuhl3gjmsZ-1alUqIg.png)

In Round 5 of the PLONK protocol, we focus on constructing the **linearization polynomial** r(X). The PLONK paper, on [page 18][13], introduces an optimization technique titled ‘Reducing the number of field elements in the proof.’ This technique is applied to the r(X) polynomials. Let’s consider an example:

-   The verifier aims to check the identity h1(X) · h2(X) − h3(X) ≡ 0. Typically, the prover would send the values of h1, h2, and h3 evaluated at a random point r, and the verifier would then check if h1(r) · h2(r) − h3(r) equals 0. In this scenario, the prover sends three field elements.
-   However, the prover could simply send a single value c = h1(r), and the verifier can then verify if a new polynomial L(X) = c · h2(X) − h3(X) equals zero at the same point. Here, L(X) is referred to as the **linearization polynomial**.
-   In the context of KZG, we can express this as computing \[L\] = c · \[h2\] − \[h3\]. This approach effectively reduces the number of field elements required in the proof.

The next two W polynomials function as openings in the KZG polynomial commitment scheme. They are composed with other polynomials that we aim to prove. To ensure their linear independence, we use the value ‘v’, which is employed for batching multiple polynomials into one, as detailed in [Part 1][14].

v = numbers\_to\_hash(round1 + round2 + round3 + round4, Fp)  
  
R = (QM \* a\_zeta \* b\_zeta +  
    QL \* a\_zeta +  
    QR \* b\_zeta +  
    QO \* c\_zeta +  
    QC)  
R += (Z \*   
    (a\_zeta + beta \* zeta + gamma) \*  
    (b\_zeta + beta \* zeta \* k1 + gamma) \*  
    (c\_zeta + beta \* zeta \* k2 + gamma) \* alpha)  
R -= (z\_omega\_zeta \*  
    (a\_zeta + beta \* s1\_zeta + gamma) \*  
    (b\_zeta + beta \* s2\_zeta + gamma) \*   
    (c\_zeta + beta \* S3 + gamma) \* alpha)  
R += (Z - Fp(1)) \* L1(zeta) \* alpha\*\*2  
R -= Zh(zeta) \* (Tl + zeta\*\*n \* Tm + zeta\*\*(2\*n) \* Th)  
  
X\_minus\_zeta = galois.Poly(\[1, -zeta\], field=Fp)  
  
Wzeta = R + \\  
        (A - a\_zeta) \* v + \\  
        (B - b\_zeta) \* v\*\*2 + \\  
        (C - c\_zeta) \* v\*\*3 + \\  
        (S1 - s1\_zeta) \* v\*\*4 + \\  
        (S2 - s2\_zeta) \* v\*\*5  
  
assert Wzeta % X\_minus\_zeta == 0, f"Wzeta(x) % X - zeta != 0"  
Wzeta = Wzeta // X\_minus\_zeta  
  
X\_minus\_omega\_zeta = galois.Poly(\[1, -(omega\*zeta)\], field=Fp)  
  
Womega\_zeta = (Z - z\_omega\_zeta)  
assert Womega\_zeta % X\_minus\_omega\_zeta == 0, f"Womega\_zeta(x) % X - ω\*zeta != 0"  
Womega\_zeta = Womega\_zeta // X\_minus\_omega\_zeta  
  
round5 = \[Wzeta(tau), Womega\_zeta(tau)\]  
  
u = numbers\_to\_hash(round1 + round2 + round3 + round4 + round5, Fp)

Finally the proof consists of 9 points and 6 field elements:

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*3ZtmfR7xQQrrIYEgBbJEsA.png)

## **Verifier**

Firstly, what the verifier needs to do with the proof is to check whether all 9 points are indeed on the elliptic curve, and whether the 6 elements are valid field elements. While I am not entirely certain of the reason for this, it might be related to mitigating the possibility of a weak curve attack. A weak curve attack in cryptography refers to a method of compromising elliptic curve cryptography (ECC) by exploiting weaknesses in the chosen elliptic curve. Therefore, an attacker might attempt to use points from a weaker curve to breach the protocol.

\# Values provided by the prover (round 1 to 5) is a proof.  
a\_exp = round1\[0\]  
b\_exp = round1\[1\]  
c\_exp = round1\[2\]  
  
z\_exp = round2\[0\]  
  
tl\_exp = round3\[0\]  
tm\_exp = round3\[1\]  
th\_exp = round3\[2\]  
  
\# Note: verifier has to verify that the following values are in the correct Fp field  
a\_zeta, b\_zeta, c\_zeta, s1\_zeta, s2\_zeta, z\_omega\_zeta = round4  
  
w\_zeta\_exp = round5\[0\]  
w\_omega\_zeta\_exp = round5\[1\]  
  
\# Note: verifier has to verify that the following values are on the curve  
if encrypted:  
    validate\_point(qm\_exp)  
    validate\_point(ql\_exp)  
    validate\_point(qr\_exp)  
    validate\_point(qo\_exp)  
    validate\_point(qc\_exp)  
    validate\_point(qpi\_exp)  
    validate\_point(z\_exp)  
    validate\_point(s1\_exp)  
    validate\_point(s2\_exp)  
    validate\_point(s3\_exp)  
    validate\_point(tl\_exp)  
    validate\_point(tm\_exp)  
    validate\_point(th\_exp)  
    validate\_point(a\_exp)  
    validate\_point(b\_exp)  
    validate\_point(c\_exp)  
    validate\_point(w\_zeta\_exp)  
    validate\_point(w\_omega\_zeta\_exp)

Next, the verifier must calculate the challenges β, γ, α, zeta, v, and u using the elements provided in the proof.

beta = numbers\_to\_hash(round1 + \[0\], Fp)  
gamma = numbers\_to\_hash(round1 + \[1\], Fp)  
alpha = numbers\_to\_hash(round1 + round2, Fp)  
zeta = numbers\_to\_hash(round1 + round2 + round3, Fp)  
v = numbers\_to\_hash(round1 + round2 + round3 + round4, Fp)  
u = numbers\_to\_hash(round1 + round2 + round3 + round4 + round5, Fp)

All following steps are done according to paper:

\# evaluate vanishing and L1 polynomial ar zeta  
Zh\_z = Zh(zeta)  
L1\_z = L1(zeta)  
\# PI\_z = PI(z) - for public input  
  
\# Compute constant part of r polynomials,  
\# r0 = PI\_z - L1\_z ...  
r0 = (- L1\_z \* alpha\*\*2 -  
    (a\_zeta + beta \* s1\_zeta + gamma) \*  
    (b\_zeta + beta \* s2\_zeta + gamma) \*  
    (c\_zeta + gamma) \* z\_omega\_zeta \* alpha)  
  
\# Compute first part of batched polynomial commitment   
D\_exp = (qm\_exp \* a\_zeta \* b\_zeta +  
        ql\_exp \* a\_zeta +  
        qr\_exp \* b\_zeta +  
        qo\_exp \* c\_zeta +  
        qc\_exp)  
  
D\_exp += (z\_exp \* (  
        (a\_zeta + beta \* zeta + gamma) \*  
        (b\_zeta + beta \* zeta \* k1 + gamma) \*  
        (c\_zeta + beta \* zeta \* k2 + gamma) \* alpha  
        + L1\_z \* alpha\*\*2 + u))  
  
D\_exp -= (s3\_exp \*  
        (a\_zeta + beta \* s1\_zeta + gamma) \*  
        (b\_zeta + beta \* s2\_zeta + gamma) \*   
        alpha \* beta \* z\_omega\_zeta)  
  
D\_exp -= ((tl\_exp +   
        tm\_exp \* zeta\*\*n  +  
        th\_exp \* zeta\*\*(2\*n)) \*  
        Zh\_z)  
  
  
F\_exp = (D\_exp +   
        a\_exp \* v +  
        b\_exp \* v\*\*2 +  
        c\_exp \* v\*\*3 +  
        s1\_exp \* v\*\*4 +  
        s2\_exp \* v\*\*5)  
  
E\_exp = (-r0 +  
        v \* a\_zeta +  
        v\*\*2 \* b\_zeta +  
        v\*\*3 \* c\_zeta +  
        v\*\*4 \* s1\_zeta +  
        v\*\*5 \* s2\_zeta +  
        u \* z\_omega\_zeta)  
  
if encrypted:  
        E\_exp = G1 \* E\_exp  
  
e1 = w\_zeta\_exp + w\_omega\_zeta\_exp \* u  
e2 = (w\_zeta\_exp \* zeta + w\_omega\_zeta\_exp \* (u \* zeta \* omega) +  
    F\_exp + (E\_exp \* Fp(p-1)))  
  
if encrypted:  
    pairing1 = tau.tau2.pair(e1)  
    pairing2 = G2.pair(e2)  
  
    print(f"pairing1 = {pairing1}")  
    print(f"pairing2 = {pairing2}")  
  
    assert pairing1 == pairing2, f"pairing1 != pairing2"  
else:  
    print("\\n\\n--- e1, e2 ---")  
    print(f"e1 = {e1 \* tau}")  
    print(f"e2 = {e2}")  
    assert e1 \* tau == e2

_UPD: The article was originally written without considering public input. The final script has now been updated to include one parameter based on public feedback_.

Final implementation:

[

## zkSNARK-under-the-hood/plonk\_latest.ipynb at main · tarassh/zkSNARK-under-the-hood

### Implementation of zero knowledge proof protocol - Groth16, Plonk. For education purposes. Not a production ready code…

github.com



][15]

[1]: /@cryptofairy?source=post_page-----5a030d15be68--------------------------------
[2]: /@cryptofairy?source=post_page-----5a030d15be68--------------------------------
[3]: /m/signin?actionUrl=https%3A%2F%2Fmedium.com%2F_%2Fsubscribe%2Fuser%2Fb3a405d735c6&operation=register&redirect=https%3A%2F%2Fmedium.com%2F%40cryptofairy%2Funder-the-hood-of-zksnarks-plonk-protocol-part-6-5a030d15be68&user=Crypto+Fairy&userId=b3a405d735c6&source=post_page-b3a405d735c6----5a030d15be68---------------------post_header-----------
[4]: /m/signin?actionUrl=https%3A%2F%2Fmedium.com%2F_%2Fvote%2Fp%2F5a030d15be68&operation=register&redirect=https%3A%2F%2Fmedium.com%2F%40cryptofairy%2Funder-the-hood-of-zksnarks-plonk-protocol-part-6-5a030d15be68&user=Crypto+Fairy&userId=b3a405d735c6&source=-----5a030d15be68---------------------clap_footer-----------
[5]: /m/signin?actionUrl=https%3A%2F%2Fmedium.com%2F_%2Fbookmark%2Fp%2F5a030d15be68&operation=register&redirect=https%3A%2F%2Fmedium.com%2F%40cryptofairy%2Funder-the-hood-of-zksnarks-plonk-protocol-part-6-5a030d15be68&source=-----5a030d15be68---------------------bookmark_footer-----------
[6]: /@cryptofairy/under-the-hood-of-zksnarks-plonk-protocol-part-5-4819dd56d3f1?source=post_page-----5a030d15be68--------------------------------
[7]: https://github.com/tarassh/zkSNARK-under-the-hood/blob/main/plonk_latest.ipynb?source=post_page-----5a030d15be68--------------------------------
[8]: /@cryptofairy/under-the-hood-of-zksnarks-plonk-protocol-part3-821855e49ce6
[9]: /@cryptofairy/under-the-hood-of-zksnarks-plonk-protocol-part-5-4819dd56d3f1
[10]: /@cryptofairy/under-the-hood-of-zksnarks-plonk-protocol-part4-5e74bddebedb
[11]: /@cryptofairy/under-the-hood-of-zksnarks-plonk-protocol-part4-5e74bddebedb
[12]: /coinmonks/under-the-hood-of-zksnarks-plonk-protocol-part-1-34bc406d8303
[13]: https://eprint.iacr.org/2019/953.pdf
[14]: /coinmonks/under-the-hood-of-zksnarks-plonk-protocol-part-1-34bc406d8303
[15]: https://github.com/tarassh/zkSNARK-under-the-hood/blob/main/plonk_latest.ipynb?source=post_page-----5a030d15be68--------------------------------