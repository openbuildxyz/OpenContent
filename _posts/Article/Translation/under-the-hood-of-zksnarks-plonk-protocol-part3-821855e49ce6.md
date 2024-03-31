---
title: "Under the hood of zkSNARKs — PLONK protocol: part3"
authorURL: ""
originalURL: https://medium.com/@cryptofairy/under-the-hood-of-zksnarks-plonk-protocol-part3-821855e49ce6
translator: ""
reviewer: ""
---

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*i0lHuI6MK-k5f1k8bm178w.png)

<!-- more -->

# Under the hood of zkSNARKs — PLONK protocol: part3

[

![Crypto Fairy](https://miro.medium.com/v2/resize:fill:88:88/1*nrbTgZM_zY_Isf4qDqpfjA.png)









][1]

[Crypto Fairy][2]

·

[Follow][3]

6 min read

·

Nov 11, 2023

[

][4]

\--

2

[][5]

Listen

Share

Many materials available online explain the basics of the PLONK protocol, often referencing Vitalik’s example _P_(_x_)=_x³_+_x_+5=35 from his article:

[

## Understanding PLONK

### Very recently, Ariel Gabizon, Zac Williamson and Oana Ciobotaru announced a new general-purpose zero-knowledge proof…

vitalik.ca



][6]

To offer a different perspective, this article will explore a modified equation that introduces additional complexities. We will use the equation _f_(_x_,_y_)=2_x²_−_x²y²_+3 and set it to equal -25, with _x_\=2 and _y_\=3. This equation features two input parameters, a subtraction operation, a negative result, the repeated use of an operation (_x²_), and multiplication by a constant (as seen in 2_x²_). These elements provide a broader understanding of how PLONK handles various computational scenarios.

## Arithmetization

The first step in transforming an equation into a Zero-Knowledge Proof (ZKP) is arithmetization. As mentioned in the previous article, to generate a proper proof for the statement, it’s essential to capture all the states of the program during its execution. Arithmetization achieves this by converting the computation into a polynomial form, which allows for the efficient generation and verification of the proof within the ZKP framework.

[

## Under the hood of zkSNARKs — PLONK protocol: Part 2

### In the previous article of the PLONK series, we covered the core of the protocol, the KZG commitment scheme, and how…

medium.com



][7]

Using Circom, we can translate our problem into code like this:

pragma circom 2.1.6;  
  
template Example () {  
    signal input x;  
    signal input y;  
    signal output out;  
  
    signal xx;  
    signal yy;  
  
    xx <== x \* x; // Quadratic term: x squared  
    yy <== y \* y; // Quadratic term: y squared  
  
    // Intermediate signal for x\*x\*y\*y  
    signal xxyy;  
    xxyy <== xx \* yy;  
  
    // Final expression  
    out <== xx \* 2 - xxyy + 3;  
}  
  
component main { public \[ x, y \] } = Example();  
  
/\* INPUT = {  
    "x": "2",  
    "y": "3"  
} \*/

This approach differs significantly from what an average developer might write in a language like Python:

x = 2  
y = 3  
out = 2\*x\*\*2 - x\*\*2\*y\*\*2 + 3

Circom’s nature as a Domain Specific Language (DSL) aids in organizing code specifically for compiling into an arithmetic circuit. This structured format not only ensures that the code aligns with the mathematical requirements of the proofs but also optimizes the process of circuit generation and proof computation.

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*ziBTJn54Cajksz3iwWioSA.png)

Arithmetic Circuit

So, a clear pattern emerges in the circuit structure: each operation corresponds to a separate gate, denoted by _g0_, _g1_, …, etc. Each gate has two inputs (a, b) and one output ( c). Take gates g0 and g1, for example; both represent the same _x²_ operation. In Circom, this calculation is performed once and then reused wherever needed, such as in _g0_ and _g1_. The reason for having two identical gates relates to the permutation check, which ensures a 1-to-1 mapping of the wires (for example, output _c0_ connects to input _a3_, _c1_ to _a4_, and so on).

Another point of interest is gate _g3_. Here, you might notice that the second input for the constant 2 is not marked by ‘b’. This is because, in PLONK, multiplication can only involve variables, so ‘b’ is set to 0 in this case. However, constants are permissible in addition gates. The constant 2 for multiplication in gate _g3_ comes from a selector vector, which we will address later, and its depicting in this gate is more symbolic.

Lastly, it’s important to note that there are no subtraction gates. All calculations are performed using Finite Field arithmetic. In this system, negative numbers are transformed into positive ones through modulo operations, aligning with the field’s properties.

## Constraining the gates

The next step involves describing each gate in the arithmetic circuit using polynomial equations.

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*rg-WtwNJPgWKe-8TRAE6xA.png)

In these equation, the variables _a_, _b_, and _c_ represent the wires associated with each gate, corresponding to their inputs and output. Alongside these, we have selector variables, denoted as _q_, which function like activators for the gates. They determine which wires and operations are engaged in the circuit at any given point.

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*rPNCyXiyGdP_MgXyIXW59A.png)

Each gate in our arithmetic circuit is represented by specific polynomial equations, influenced by selector variables. For example, gate _g0_ performs a multiplication of inputs _a_0 and _b_0\. To enable this operation, the multiplication selector variable _qm_ is set to 1, and the gate outputs _c_0\. The output activation is facilitated by setting the output selector _qo_ to -1, adhering to the constraint that our polynomial equations must sum to 0. Gates _g1_, _g2_, and _g4_ are configured similarly to _g0_.

In gate g3, to multiply the value _a_3 by the constant 2, we set the selector variable _ql_ to 2. This gate showcases how constants are incorporated into the computations. For subtraction, as seen in gate _g5_, the right selector _qr_ is set to -1, representing the subtraction operation in polynomial form. Addition is handled in gate _g6_, where the addition of a constant 3 is facilitated through the constant selector _qc_.

We have all the values for the gates so we can encode them into following vectors:

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*RbF8x1Y--LLr53FwOG7_Uw.png)

## Dummy Gates

In the previous article, we discussed an optimization technique involving the vanishing polynomial in PLONK, where steps are marked using roots of unity instead of sequential numbers. This optimisation necessitates having a number of gates that is a power of 2.

Currently, our circuit comprises 7 gates. However, to meet the 2^n requirement in PLONK, the closest power of 2 is 8 (2³ = 8). This means we need to add an additional gate to make the total number 8. This extra gate is a ‘dummy gate’ — it doesn’t perform any real computation but serves as a placeholder to satisfy the power of 2 structure. By adding this dummy gate, we ensure that our circuit adheres to the optimal structure for PLONK’s efficiency mechanisms.

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
\# -- results ---  
\# a = \[2, 2, 3, 4, 4, 8, -28, 0\]  
\# b = \[2, 2, 3, 0, 9, 36, 3, 0\]  
\# c = \[4, 4, 9, 8, 36, -28, -25, 0\]  
\# ql = \[0, 0, 0, 2, 0, 1, 1, 0\]  
\# qr = \[0, 0, 0, 0, 0, -1, 0, 0\]  
\# qm = \[1, 1, 1, 1, 1, 0, 0, 0\]  
\# qc = \[0, 0, 0, 0, 0, 0, 3, 0\]  
\# qo = \[-1, -1, -1, -1, -1, -1, -1, 0\]

In the next article we will discuss permutation constraint.

[

## Under the hood of zkSNARKs — PLONK protocol: part4

### This article will discuss the permutation check in PLONK, which I believe is the second most challenging aspect of the…

medium.com



][8]

[

## zkSNARK-under-the-hood/plonk\_latest.ipynb at main · tarassh/zkSNARK-under-the-hood

### Implementation of zero knowledge proof protocol - Groth16, Plonk. For education purposes. Not a production ready code…

github.com



][9]

[1]: /@cryptofairy?source=post_page-----821855e49ce6--------------------------------
[2]: /@cryptofairy?source=post_page-----821855e49ce6--------------------------------
[3]: /m/signin?actionUrl=https%3A%2F%2Fmedium.com%2F_%2Fsubscribe%2Fuser%2Fb3a405d735c6&operation=register&redirect=https%3A%2F%2Fmedium.com%2F%40cryptofairy%2Funder-the-hood-of-zksnarks-plonk-protocol-part3-821855e49ce6&user=Crypto+Fairy&userId=b3a405d735c6&source=post_page-b3a405d735c6----821855e49ce6---------------------post_header-----------
[4]: /m/signin?actionUrl=https%3A%2F%2Fmedium.com%2F_%2Fvote%2Fp%2F821855e49ce6&operation=register&redirect=https%3A%2F%2Fmedium.com%2F%40cryptofairy%2Funder-the-hood-of-zksnarks-plonk-protocol-part3-821855e49ce6&user=Crypto+Fairy&userId=b3a405d735c6&source=-----821855e49ce6---------------------clap_footer-----------
[5]: /m/signin?actionUrl=https%3A%2F%2Fmedium.com%2F_%2Fbookmark%2Fp%2F821855e49ce6&operation=register&redirect=https%3A%2F%2Fmedium.com%2F%40cryptofairy%2Funder-the-hood-of-zksnarks-plonk-protocol-part3-821855e49ce6&source=-----821855e49ce6---------------------bookmark_footer-----------
[6]: https://vitalik.ca/general/2019/09/22/plonk.html?source=post_page-----821855e49ce6--------------------------------
[7]: /@cryptofairy/under-the-hood-of-zksnarks-plonk-protocol-part-2-ee00d6accb4d?source=post_page-----821855e49ce6--------------------------------
[8]: /@cryptofairy/under-the-hood-of-zksnarks-plonk-protocol-part4-5e74bddebedb?source=post_page-----821855e49ce6--------------------------------
[9]: https://github.com/tarassh/zkSNARK-under-the-hood/blob/main/plonk_latest.ipynb?source=post_page-----821855e49ce6--------------------------------