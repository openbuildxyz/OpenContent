---
title: "Under the hood of zkSNARKs — PLONK protocol: Part 2"
authorURL: ""
originalURL: https://medium.com/coinmonks/under-the-hood-of-zksnarks-plonk-protocol-part-2-ee00d6accb4d
translator: ""
reviewer: ""
---

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*UDrnDHC57-74c48wyK2-WA.png)

<!-- more -->

Polynomials — seen by OpenAI

# Under the hood of zkSNARKs — PLONK protocol: Part 2

[

![Crypto Fairy](https://miro.medium.com/v2/resize:fill:88:88/1*nrbTgZM_zY_Isf4qDqpfjA.png)









][1][

![Coinmonks](https://miro.medium.com/v2/resize:fill:48:48/1*-_aiJHzJPz655N7iSSrLrQ.png)











][2]

[Crypto Fairy][3]

·

[Follow][4]

Published in

[

Coinmonks

][5]

·

6 min read

·

Nov 7, 2023

[

][6]

\--

[][7]

Listen

Share

In the previous article of the PLONK series, we covered the core of the protocol, the KZG commitment scheme, and how polynomial linear independence can aid in evaluating multiple polynomials within a single equation.

[

## Under the hood of zkSNARKs — PLONK protocol: Part 1

### PLONK is one of the zero-knowledge proving systems that belong to the SNARK group. Compared to Groth16, which is an…

medium.com



][8]

Here, I want to discuss another important optimization used in PLONK, which is related to the vanishing polynomial.

## Vanishing Polynomial

_Note: The explanations in this chapter are based on my understanding of the role of vanishing polynomials. If you notice any inaccuracies, I am more than happy to address and correct them._

In the previous article, I mentioned that polynomials are an efficient way to encode the states of a particular process into one function. Consider the naive example illustrated below: at state #1, the process has a value of -6; at state #2, the value changes to 2; at state #3, it changes to 162, and so on.

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*RvqXOii44iIK8tS9d9bvyQ.png)

This concept can be applied to an example involving program execution. At a particular execution point, the program had a state value of -6, and at another point, a value of 852, and so on. In the context of zero-knowledge proofs, we aim to prove the correctness of a program’s execution or the validation of a fact. To achieve this, we need to capture all the states. Delving deeper into the context of zero-knowledge proofs, and PLONK in particular, we construct polynomials in such a way that their evaluation at all states equals zero (the next article will provide a very specific example). If a program takes 5 steps to execute something, this means that we would have a polynomial where the evaluation result is zero for each of these steps.

Here comes into play the vanishing polynomial — it is the simplest type of polynomial that equals zero when evaluated at all points from 1 to 5.

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*L1hvG9A6iy0b_FaMCDaWwQ.png)

Vanishing polynomial for all x (1, … 5)

As you can see, if you pick any _x_ from 1 to 5, the evaluation result will be zero. Why do we need it? Let’s say we have a polynomial _P_(_x_) that we need to prove:

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*pARhYIiU0XTO4joy9CA9wg.png)

Polynomial _P_(_x_) is also equal to zero at all given points from 1 to 5. Therefore, if we divide _P_(_x_) by _Zh_(_x_) using polynomial division, there should be no remainder in the result; the polynomial _Zh_(_x_) should ‘vanish.’ This indicates that the prover indeed has a polynomial that is zero for all the _x_ from 1 to 5. In PLONK, the vanishing polynomial is used in several places, so if you are constructing a proof and, after applying the division where it is required, your polynomial has a remainder, it means that there are error(s) in your calculations.

Another reason why there should be no remainder after division is that the prover would not be able to construct the proof using the Common Reference String (CRS) from the Trusted Setup.

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*qOKVEtoCi8EZycimh1l6fg.png)

Division with reminder

If we have a remainder x² after the division, we cannot evaluate this on Elliptic Curves because they do not support a division operation. Alternatively, we can represent division as multiplication by the inverse, effectively raising to the power of (-1). The problem here is that the given CRS is calculated for positive power values:

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*HtUbIh0L63zfYhBMZiB8dA.png)

## Vanishing polynomial optimisation

Earlier, we stated that if a program requires 5 steps to compute or verify something, then we need to construct a vanishing polynomial that equals zero for all the corresponding states. On average, zk-SNARK programs may require 1 million steps (gates) to generate a proof. Consequently, for a prover to construct and use such a lengthy polynomial, there would be a significant impact on the prover’s performance, exemplified by a polynomial like (_x_−1)(_x_−2)…(_x_−1000000). The Groth16 protocol does exactly this. PLONK, on the other hand, utilizes multiplicative subgroups, which are also built on roots of unity. Does this sound complicated? I will explain it very simply by addressing the following points:

1\. All calculations are performed using modular arithmetic in a finite field of some prime number _p_, for example, 23. This means there are only numbers from 0 to 22, and any computation that exceeds this range will be adjusted by the modulo 23 operation. The best visual explanation can be found at the link below. In real-world applications, we would use a much larger prime number for _p._

[

## The Animated Elliptic Curve

### Visualize elliptic curve cryptography with animated examples

curves.xargs.org







][9]

2\. The program requires 5 steps to compute the proof. To use roots of unity for evaluation, we need to extend our program to 8 steps because 8 is the next power of two closest to 5 (since 2³=8), and there is no power of two that equals 5. Now our program will have 8 steps, but the last three will not affect the program’s state.

3\. Unfortunately, we cannot remain consistent with the previous example of the finite field _p_\=23, because this set has too few elements to cover all 8 steps of the program’s execution. Therefore, we must use a finite field of a higher order, such as _p_\=73. Within this finite field, there should be a number (let’s name it omega — _ω_) such that _ω⁸_ equals 1. In the case of _p_\=73, this number is 10.

4\. Next, we need to generate other values using 10, which serves as a generator. These generated values are called roots of unity.

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*zDeT1rpUJxqEV_VPpA_wqQ.png)

import galois  
Fp = galois.GF(73)  
  
omega = Fp.primitive\_root\_of\_unity(8)  
roots = \[omega\*\*i for i in range(8)\]  
print(roots)  
\# ---- roots ---  
\# \[1, 10, 27, 51, 72, 63, 46, 22\]

So now, instead of using indices from 1 to 8 for each program step, we use these values, and the resulting vanishing polynomial should appear as follows:

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*YR_bGYJTactw2-hSUQmb5Q.png)

You may wonder why this polynomial is longer than the original one considered more efficient than the original one, where we used indices from 1 to 8. But there is a catch: it can be represented in a much shorter form:

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*LsK3w02ahhSBUXw2nA2pkg.png)

line #1 — vanishing polynomial for 8 steps/gates; line #2 — generic formula for vanishing polynomial.

This is the vanishing polynomial that saves time for the prover during proof generation.

Additionally, using roots of unity as evaluation points has a performance benefit. In the protocol, there are number of polynomial multiplications, which are one of the performance bottlenecks for proof generation. The most efficient way to perform these multiplications is by using Fast Fourier Transforms (FFTs), and FFTs are feasible only when roots of unity are used as evaluation points. The video below provides some good insights into why this is the case:

This is all for now. In the next article, we will start to delve into the PLONK protocol.

[

## Under the hood of zkSNARKs — PLONK protocol: part3

### Many materials available online explain the basics of the PLONK protocol, often referencing Vitalik’s example…

medium.com



][10]

[1]: /@cryptofairy?source=post_page-----ee00d6accb4d--------------------------------
[2]: https://medium.com/coinmonks?source=post_page-----ee00d6accb4d--------------------------------
[3]: /@cryptofairy?source=post_page-----ee00d6accb4d--------------------------------
[4]: /m/signin?actionUrl=https%3A%2F%2Fmedium.com%2F_%2Fsubscribe%2Fuser%2Fb3a405d735c6&operation=register&redirect=https%3A%2F%2Fmedium.com%2Fcoinmonks%2Funder-the-hood-of-zksnarks-plonk-protocol-part-2-ee00d6accb4d&user=Crypto+Fairy&userId=b3a405d735c6&source=post_page-b3a405d735c6----ee00d6accb4d---------------------post_header-----------
[5]: https://medium.com/coinmonks?source=post_page-----ee00d6accb4d--------------------------------
[6]: /m/signin?actionUrl=https%3A%2F%2Fmedium.com%2F_%2Fvote%2Fcoinmonks%2Fee00d6accb4d&operation=register&redirect=https%3A%2F%2Fmedium.com%2Fcoinmonks%2Funder-the-hood-of-zksnarks-plonk-protocol-part-2-ee00d6accb4d&user=Crypto+Fairy&userId=b3a405d735c6&source=-----ee00d6accb4d---------------------clap_footer-----------
[7]: /m/signin?actionUrl=https%3A%2F%2Fmedium.com%2F_%2Fbookmark%2Fp%2Fee00d6accb4d&operation=register&redirect=https%3A%2F%2Fmedium.com%2Fcoinmonks%2Funder-the-hood-of-zksnarks-plonk-protocol-part-2-ee00d6accb4d&source=-----ee00d6accb4d---------------------bookmark_footer-----------
[8]: /@cryptofairy/under-the-hood-of-zksnarks-plonk-protocol-part-1-34bc406d8303?source=post_page-----ee00d6accb4d--------------------------------
[9]: https://curves.xargs.org/?source=post_page-----ee00d6accb4d--------------------------------#finite-field-math
[10]: /@cryptofairy/under-the-hood-of-zksnarks-plonk-protocol-part3-821855e49ce6?source=post_page-----ee00d6accb4d--------------------------------