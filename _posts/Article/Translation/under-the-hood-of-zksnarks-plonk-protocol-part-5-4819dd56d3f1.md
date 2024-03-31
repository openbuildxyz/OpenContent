---
title: "Under the hood of zkSNARKs — PLONK protocol: part 5"
authorURL: ""
originalURL: https://medium.com/@cryptofairy/under-the-hood-of-zksnarks-plonk-protocol-part-5-4819dd56d3f1
translator: ""
reviewer: ""
---

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*PvK_3Gp8orJ4F1Qn0jd4Ng.png)

<!-- more -->

# Under the hood of zkSNARKs — PLONK protocol: part 5

[

![Crypto Fairy](https://miro.medium.com/v2/resize:fill:88:88/1*nrbTgZM_zY_Isf4qDqpfjA.png)









][1]

[Crypto Fairy][2]

·

[Follow][3]

8 min read

·

Nov 19, 2023

[

][4]

\--

[][5]

Listen

Share

Previous articles in the PLONK series served as preparations before we got our hands dirty and started implementing the protocol. Diving directly into the code would have made it difficult to understand the reasoning and rationale behind the things we are going to implement. The last significant concept we need to explain before proceeding is related to non-interactive proofs.

[

## Under the hood of zkSNARKs — PLONK protocol: Part 1

### PLONK is one of the zero-knowledge proving systems that belong to the SNARK group. Compared to Groth16, which is an…

medium.com



][6]

In Part 1, we delved into the core of the PLONK protocol, the KZG polynomial commitment scheme, which is interactive by design:

-   The prover commits to a polynomial by sending a commitment to a verifier.
-   In return, the prover receives a random value _r_ selected by the verifier.
-   Based on this random value _r_, the prover generates the next commitment and sends it back to the verifier.
-   The verifier then checks if the prover provided correct commitments, based on everything received from the prover and the random value _r_.

Thanks to the Fiat-Shamir heuristic in PLONK, the KZG becomes non-interactive, meaning that the proof is the only piece of data sent to the verifier. Both the prover and verifier are able to agree on the random value _r_ without any interaction. This is achieved using a hash function:

-   The prover generates a commitment to a polynomial, denoted by the letter C, for example.
-   To obtain the random value _r_, the prover uses the hash function _H_, where _r_\=_H_(_C_), and using the value _r_, generates the remaining commitment _Q_, and sends _C_ and _Q_ to the verifier.
-   The verifier, using the same hash function _H_, generates the value _r_\=_H_(_C_) and, having all the necessary data, is able to verify the commitment _C_.

## Preparation

The final implementation of the PLONK protocol can be found here:

[

## zkSNARK-under-the-hood/plonk\_latest.ipynb at main · tarassh/zkSNARK-under-the-hood

### Implementation of zero knowledge proof protocol - Groth16, Plonk. For education purposes. Not a production ready code…

github.com



][7]

We are using Python because this language is exceptionally suitable for educational purposes. We will be relying heavily on the [Galois][8] library, as it provides a very convenient interface for working with polynomials:

import galois  
  
P = galois.Poly(\[1, 4\], field=galois.GF(7))  
print(f"P = {P}")  
print(f"P(1) = {P(1)}")  
\# --- prints ---  
\# P = x + 4  
\# P(1) = 5

Also, by using the monkey patching technique, we can extend the evaluation function so that polynomials can work with elliptic curve points:

def new\_call(self, at, \*\*kwargs):  
    # class SRS - Elliptic curve points from Trusted Setup  
    if isinstance(at, SRS):  
        coeffs = self.coeffs\[::-1\]  
        result = at.tau1\[0\] \* coeffs\[0\]  
        for i in range(1, len(coeffs)):  
            result += at.tau1\[i\] \* coeffs\[i\]  
        return result  
  
    return galois.Poly.original\_call(self, at, \*\*kwargs)  
  
galois.Poly.original\_call = galois.Poly.\_\_call\_\_  
galois.Poly.\_\_call\_\_ = new\_call

This clever trick helps us make the code compatible with both Finite Fields (in unencrypted form) and Elliptic Curves (in encrypted form). The rationale behind this approach is simple: it is easier to implement the protocol in an unencrypted form by selecting a Finite Field with a lower order (p=241 in our case). This makes it much easier to debug, locate, and troubleshoot bugs.

import galois  
import numpy as np  
from utils import generator1, generator2, curve\_order, normalize, validate\_point  
  
G1 = generator1()  
G2 = generator2()  
  
  
class SRS:  
    """Trusted Setup Class aka Structured Reference String"""  
    def \_\_init\_\_(self, tau, n = 2):  
        self.tau = tau  
        self.tau1 = \[G1 \* int(tau)\*\*i for i in range(0, n + 7)\]  
        self.tau2 = G2 \* int(tau)  
  
    def \_\_str\_\_(self):  
        s = f"tau: {self.tau}\\n"  
        s += "".join(\[f"\[tau^{i}\]G1: {str(normalize(point))}\\n" for i, point in enumerate(self.tau1)\])  
        s += f"\[tau\]G2: {str(normalize(self.tau2))}\\n"  
        return s  
  
def new\_call(self, at, \*\*kwargs):  
    if isinstance(at, SRS):  
        coeffs = self.coeffs\[::-1\]  
        result = at.tau1\[0\] \* coeffs\[0\]  
        for i in range(1, len(coeffs)):  
            result += at.tau1\[i\] \* coeffs\[i\]  
        return result  
  
    return galois.Poly.original\_call(self, at, \*\*kwargs)  
  
galois.Poly.original\_call = galois.Poly.\_\_call\_\_  
galois.Poly.\_\_call\_\_ = new\_call

Next, we need to define our Finite Field with order p = 241 and find roots of unity ([as discussed in part 2][9]), which are necessary for polynomial evaluation.

\# to switch between encrypted (ECC) and unencrypted mode (prime field p=241)  
encrypted = False  
  
\# Prime field p  
p = 241 if not encrypted else curve\_order  
Fp = galois.GF(p)  
  
\# We have 7 gates, next power of 2 is 8  
n = 7  
n = 2\*\*int(np.ceil(np.log2(n)))  
assert n & n - 1 == 0, "n must be a power of 2"  
  
\# Find primitive root of unity (generator)  
omega = Fp.primitive\_root\_of\_unity(n)  
assert omega\*\*(n) == 1, f"omega (ω) {omega} is not a root of unity"  
\# ω = 30  
  
roots = Fp(\[omega\*\*i for i in range(n)\])  
print(f"roots = {roots}")  
\# roots = \[  1  30 177   8 240 211  64 233\]

_Note: Before the provers start generating the proof, the circuit wiring and permutation should be known beforehand. Therefore, the following code might give the wrong impression by including witnesses data (a, b, and c vectors) in it. Witness data is used to demonstrate the permutation relation to real values. In a real situation, a compiler would generate a circuit with placeholders for a, b, and c._

The circuit consists of 7 gates. For reasons explained in previous articles, we need to extend our vectors to 8 gates:

def pad\_array(a, n):  
    return a + \[0\]\*(n - len(a))  
  
\# witness vectors  
a = \[2, 2, 3, 4, 4, 8, -28\]  
b = \[2, 2, 3, 0, 9, 36, 3\]  
c = \[4, 4, 9, 8, 36, -28, -25\]  
  
\# gate vectors  
ql = \[0, 0, 0, 2, 0, 1, 1\]  
qr = \[0, 0, 0, 0, 0, -1, 0\]  
qm = \[1, 1, 1, 1, 1, 0, 0\]  
qc = \[0, 0, 0, 0, 0, 0, 3\]  
qo = \[-1, -1, -1, -1, -1, -1, -1\]  
  
\# pad vectors to length n  
a = pad\_array(a, n)  
b = pad\_array(b, n)  
c = pad\_array(c, n)  
ql = pad\_array(ql, n)  
qr = pad\_array(qr, n)  
qm = pad\_array(qm, n)  
qc = pad\_array(qc, n)  
qo = pad\_array(qo, n)  
  
\# --- padded vectors ----  
a = \[2, 2, 3, 4, 4, 8, -28, 0\]  
b = \[2, 2, 3, 0, 9, 36, 3, 0\]  
c = \[4, 4, 9, 8, 36, -28, -25, 0\]  
ql = \[0, 0, 0, 2, 0, 1, 1, 0\]  
qr = \[0, 0, 0, 0, 0, -1, 0, 0\]  
qm = \[1, 1, 1, 1, 1, 0, 0, 0\]  
qc = \[0, 0, 0, 0, 0, 0, 3, 0\]  
qo = \[-1, -1, -1, -1, -1, -1, -1, 0\]

Now, we have to create permutation vectors (sigma) for a, b, and c:

ai = range(0, n)  
bi = range(n, 2\*n)  
ci = range(2\*n, 3\*n)  
  
sigma = {  
    ai\[0\]: ai\[0\], ai\[1\]: ai\[1\], ai\[2\]: ai\[2\], ai\[3\]: ci\[0\], ai\[4\]: ci\[1\], ai\[5\]: ci\[3\], ai\[6\]: ci\[5\], ai\[7\]: ai\[7\],  
    bi\[0\]: bi\[0\], bi\[1\]: bi\[1\], bi\[2\]: bi\[2\], bi\[3\]: bi\[3\], bi\[4\]: ci\[2\], bi\[5\]: ci\[4\], bi\[6\]: bi\[6\], bi\[7\]: bi\[7\],  
    ci\[0\]: ai\[3\], ci\[1\]: ai\[4\], ci\[2\]: bi\[4\], ci\[3\]: ai\[5\], ci\[4\]: bi\[5\], ci\[5\]: ai\[6\], ci\[6\]: ci\[6\], ci\[7\]: ci\[7\],  
}  
  
k1 = 2  
k2 = 4  
c1\_roots = roots  
c2\_roots = roots \* k1  
c3\_roots = roots \* k2  
  
c\_roots = np.concatenate((c1\_roots, c2\_roots, c3\_roots))  
  
check = set()  
for r in c\_roots:  
    assert not int(r) in check, f"Duplicate root {r} in {c\_roots}"  
    check.add(int(r))  
  
sigma1 = Fp(\[c\_roots\[sigma\[i\]\] for i in range(0, n)\])  
sigma2 = Fp(\[c\_roots\[sigma\[i + n\]\] for i in range(0, n)\])  
sigma3 = Fp(\[c\_roots\[sigma\[i + 2 \* n\]\] for i in range(0, n)\])  
  
print\_sigma(sigma, a, b, c, c\_roots)  
\# print\_sigma outputs  
 w | value  | i      | sigma(i)  
a0 |      2 |      1 |      1  
a1 |      2 |     30 |     30  
a2 |      3 |    177 |    177  
a3 |      4 |      8 |      4  
a4 |      4 |    240 |    120  
a5 |      8 |    211 |     32  
a6 |    -28 |     64 |    121  
a7 |      0 |    233 |    233  
\-- | --     | --     | --      
b0 |      2 |      2 |      2  
b1 |      2 |     60 |     60  
b2 |      3 |    113 |    113  
b3 |      0 |     16 |     16  
b4 |      9 |    239 |    226  
b5 |     36 |    181 |    237  
b6 |      3 |    128 |    128  
b7 |      0 |    225 |    225  
\-- | --     | --     | --      
c0 |      4 |      4 |      8  
c1 |      4 |    120 |    240  
c2 |      9 |    226 |    239  
c3 |      8 |     32 |    211  
c4 |     36 |    237 |    181  
c5 |    -28 |    121 |     64  
c6 |    -25 |     15 |     15  
c7 |      0 |    209 |    209  
  
\--- Sigma ---  
sigma1 = \[  1  30 177   4 120  32 121 233\]  
sigma2 = \[  2  60 113  16 226 237 128 225\]  
sigma3 = \[  8 240 239 211 181  64  15 209\]

As explained in the [previous article][10], to have a correct permutation proof, we need to mark each wire with a unique index, and these indexes have to be generated from roots of unity. The output clearly shows how permutations are encoded: for example, the a0 wire (value 2) has index 1 and is not connected with any other gate, so its permutation index sigma is also 1. The a5 wire (value 8), for instance, has index 211 and is permuted with index 32, which belongs to wire c3 (also value 8).

Now that the permutation map is ready, we can start creating polynomials. Let’s begin with gate polynomials:

def to\_galois\_array(vector, field):  
    # normalize to positive values  
    a = \[x % field.order for x in vector\]  
    return field(a)  
  
def to\_poly(x, v, field):  
    assert len(x) == len(v)  
    y = to\_galois\_array(v, field) if type(v) == list else v  
    return galois.lagrange\_poly(x, y)  
  
QL = to\_poly(roots, ql, Fp)  
QR = to\_poly(roots, qr, Fp)  
QM = to\_poly(roots, qm, Fp)  
QC = to\_poly(roots, qc, Fp)  
QO = to\_poly(roots, qo, Fp)  
\# --- Gate Polynomials ---  
\# QL = 187x^7 + 38x^6 + 119x^5 + 60x^4 + 70x^3 + 22x^2 + 106x + 121  
\# QR = 64x^7 + 8x^6 + x^5 + 211x^4 + 177x^3 + 233x^2 + 240x + 30  
\# QM = 57x^7 + 211x^6 + 73x^5 + 211x^4 + 168x^3 + 211x^2 + 184x + 91  
\# QC = 24x^7 + 90x^6 + 217x^5 + 151x^4 + 24x^3 + 90x^2 + 217x + 151  
\# QO = 240x^7 + 8x^6 + 177x^5 + 30x^4 + x^3 + 233x^2 + 64x + 210

_Note: For better performance, we would need to have polynomials in evaluated form and perform polynomial multiplications using FFT transformations. This would introduce more complexity to the code. But to align more closely with the formulas presented in the PLONK paper we will stick to the coefficient form._

We have encoded our q vectors into polynomials, and if I needed to get a value at index 8 (a root of unity) from polynomial QL, the result would be 2:

\#   i = \[  0   1   2   3   4   5  6    7\]  
roots = \[  1  30 177   8 240 211  64 233\]  
ql =    \[  0   0   0   2   0   1   1   0\]  
  
\# QL(8) = 2

Same technique we should use for permutation polynomials:

S1 = to\_poly(roots, sigma1, Fp)  
S2 = to\_poly(roots, sigma2, Fp)  
S3 = to\_poly(roots, sigma3, Fp)  
  
I1 = to\_poly(roots, c1\_roots, Fp)  
I2 = to\_poly(roots, c2\_roots, Fp)  
I3 = to\_poly(roots, c3\_roots, Fp)

Vanishing polynomial:

def to\_vanishing\_poly(roots, field):  
    # Z^n - 1 = (Z - 1)(Z - w)(Z - w^2)...(Z - w^(n-1))  
    return galois.Poly.Degrees(\[len(roots), 0\], coeffs=\[1, -1\], field=field)  
  
Zh = to\_vanishing\_poly(roots, Fp)  
for x in roots:  
    assert Zh(x) == 0  
\# --- Vanishing Polynomial ---  
\# Zh = x^8 + 240 which translates to x^8 - 1 (240 == -1 in Prime Field p=241)

## Trusted Setup

To implement the trusted setup in our code, we don’t need extensive coding. It’s important to highlight that our trusted setup can operate in two modes, encrypted and unencrypted, by effectively utilising an `encrypted` flag.

def generate\_tau(encrypted):  
    """ Generate a random tau in Fp or G1 """  
    return SRS(Fp.Random(), n) if encrypted else Fp.Random()  
  
tau = generate\_tau(encrypted)

Next article will be dedicated to Prover and Verifier implementations.

[

## Under the hood of zkSNARKs — PLONK protocol: part 6

### Final article in PLONK series. Here we will implement Prover and the Verifier. In the previous article we coded trusted…

medium.com



][11]

[1]: /@cryptofairy?source=post_page-----4819dd56d3f1--------------------------------
[2]: /@cryptofairy?source=post_page-----4819dd56d3f1--------------------------------
[3]: /m/signin?actionUrl=https%3A%2F%2Fmedium.com%2F_%2Fsubscribe%2Fuser%2Fb3a405d735c6&operation=register&redirect=https%3A%2F%2Fmedium.com%2F%40cryptofairy%2Funder-the-hood-of-zksnarks-plonk-protocol-part-5-4819dd56d3f1&user=Crypto+Fairy&userId=b3a405d735c6&source=post_page-b3a405d735c6----4819dd56d3f1---------------------post_header-----------
[4]: /m/signin?actionUrl=https%3A%2F%2Fmedium.com%2F_%2Fvote%2Fp%2F4819dd56d3f1&operation=register&redirect=https%3A%2F%2Fmedium.com%2F%40cryptofairy%2Funder-the-hood-of-zksnarks-plonk-protocol-part-5-4819dd56d3f1&user=Crypto+Fairy&userId=b3a405d735c6&source=-----4819dd56d3f1---------------------clap_footer-----------
[5]: /m/signin?actionUrl=https%3A%2F%2Fmedium.com%2F_%2Fbookmark%2Fp%2F4819dd56d3f1&operation=register&redirect=https%3A%2F%2Fmedium.com%2F%40cryptofairy%2Funder-the-hood-of-zksnarks-plonk-protocol-part-5-4819dd56d3f1&source=-----4819dd56d3f1---------------------bookmark_footer-----------
[6]: /coinmonks/under-the-hood-of-zksnarks-plonk-protocol-part-1-34bc406d8303?source=post_page-----4819dd56d3f1--------------------------------
[7]: https://github.com/tarassh/zkSNARK-under-the-hood/blob/main/plonk_latest.ipynb?source=post_page-----4819dd56d3f1--------------------------------
[8]: https://galois.readthedocs.io/en/v0.3.6/
[9]: /coinmonks/under-the-hood-of-zksnarks-plonk-protocol-part-2-ee00d6accb4d
[10]: /@cryptofairy/under-the-hood-of-zksnarks-plonk-protocol-part4-5e74bddebedb
[11]: /@cryptofairy/under-the-hood-of-zksnarks-plonk-protocol-part-6-5a030d15be68?source=post_page-----4819dd56d3f1--------------------------------