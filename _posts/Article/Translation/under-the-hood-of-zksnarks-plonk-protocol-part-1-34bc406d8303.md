---
title: "Under the hood of zkSNARKs — PLONK protocol: Part 1"
authorURL: ""
originalURL: https://medium.com/coinmonks/under-the-hood-of-zksnarks-plonk-protocol-part-1-34bc406d8303
translator: ""
reviewer: ""
---

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*4pLqLYSKvmAjftJMjVmrZg.jpeg)

<!-- more -->

Plonk — seen by OpenAI

# Under the hood of zkSNARKs — PLONK protocol: Part 1

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

8 min read

·

Nov 2, 2023

[

][6]

\--

1

[][7]

Listen

Share

PLONK is one of the zero-knowledge proving systems that belong to the SNARK group. Compared to Groth16, which is an older system, PLONK offers advantages such as a universal and updatable trusted setup. In Groth16, a circuit-specific (or task-specific, for those unfamiliar with the term) trusted setup is required, whereas in Plonk, the same setup can be reused for any circuit. The term ‘updatable’ refers to the ability for anyone to add randomness to the setup, reinforcing trust in its integrity.

However, a downside of PLONK is that it results in larger proof sizes, which can impact gas costs on the Ethereum network. This is a potential reason why Groth16 may still be considered competitive.

The PLONK protocol encapsulates a variety of different techniques, some of which must be understood before delving into the protocol itself. Let’s start with the basics.

## Polynomials

Polynomials are one of the foundational elements of zero-knowledge proving systems. Many of us learned about them in school, but their application in this context might not be immediately obvious. Why are they used?

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*H5kMilzYoAFDJb-NsCKnzA.png)

Polynomial of degree n

A polynomial provides a form that allows us to encapsulate the states of a particular process within a single function. This is a simplified way I understand its role. So, when we want to prove something, such as the correctness of a program’s execution or the verification of a statement, we can represent these processes using polynomials. This remarkable characteristic enables us to construct sophisticated proving systems.

## KZG/Kate Polynomial Commitment Schema

In a nutshell, Plonk utilizes the KZG (Kate, Zaverucha, Goldberg) scheme for verification. Let’s consider a toy example to understand how commitment schemes work.

**Setup Phase**:

-   Alice has something she wants to commit to, like her decision on where to go for dinner. Let’s say she has chosen “Italian.”
-   She writes her choice on a piece of paper.

**Commit Phase**:

-   Alice places the paper inside a box.
-   She locks the box with a padlock and keeps the key.
-   She gives the locked box to Bob, ensuring him that her dinner choice is inside.
-   At this point, Bob has the box with the commitment inside, but he cannot access it because it is locked.

**Reveal Phase**:

-   When it’s time to reveal her dinner choice, Alice gives Bob the key.
-   Bob unlocks the box, takes out the paper, and reads Alice’s dinner choice: “Italian.”
-   Bob now knows Alice’s commitment, and he is assured that Alice did not change her mind after giving him the locked box because the box was securely locked the entire time.

In the KZG commitment scheme, the prover commits to a polynomial (e.g., representing the correctness of a program execution).

Assume a prover has a polynomial _P_(_x_) of degree _n_ to which he needs to commit. To do this, he would require a trusted setup. For additional information about trusted setups, please refer to the following source:

[

## Trusted Setup in zkSNARKs— Powers of Tau vs Lagrange basis

### In the spirit of unraveling the mystery of the black box known as “zero-knowledge proofs”, this article seeks to…

medium.com



][8]

The trusted setup in this context refers to the generation of powers of a random secret value, τ (tau), up to the _n-th_ degree. These powers are then ‘coated’ or represented as points on an elliptic curve. If you are not familiar with elliptic curve points, you can think of them as the product of the secret value τ and a base point on the curve, which obscures the actual value of τ. This process prevents anyone from deducing the original secret value, which is critical for the security of the system.

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*9_5KwJjeP7ELf7IE_0GHQg.png)

Trusted Setup

Commitment to our polynomial would look like this:

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*6c6ePXtQXbarRY3TxnOMtg.png)

\[…\] — denotes point on elliptic curve

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*3g5ZA51lzQhPlSfrfz7fZw.png)

If we would use τ and multiply evaluation result by \[G\] we would get the same result P(τ). That’s why it is valid to use CRS for evaluation.

The prover calculates a commitment to the polynomial _P_(_τ_) using the Common Reference String obtained from the trusted setup, and sends this commitment, which is a point on an elliptic curve, to the verifier.

This is because a scalar multiple of the curve’s base point \[G\] (such as _cτ_) results in another point on the curve. The verifier receives the commitment to _P_(_τ_), but to maintain the zero-knowledge property, the actual polynomial _P_(_x_) is not revealed.

Instead, the verifier sends a random value _r_ to the prover, who then evaluates _P_(_x_) at that point and proceeds with the protocol.

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*P_Yxbmqh2785M0QIepNobQ.png)

Evaluation at value r

The prover sends the evaluated value _P_(_r_) back to the verifier. The verifier now has the commitment _P_(_τ_), the evaluated _P_(_r_), and knowledge of the Polynomial Remainder Theorem (often confused with Bézout’s Theorem in this context), which states:

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*Kc2e0yOcTfg18hY-zDfjOw.png)

Quotient Polynomial

If you subtract _P_(_r_) from _P_(_x_) and then divide by _x_−_r_, the division should have no remainder, giving you the quotient polynomial _Q_(_x_). Along with _P_(_r_), the prover also sends the commitment to _Q_(_τ_) to the verifier. Now the verifier has _P_(_τ_), _Q_(_τ_), _r_, and _P_(_r_). Since division as such is not defined on elliptic curves, the commitments _P_ and _Q_ (which are points on the elliptic curve) undergo a series of transformations to enable the verifier to confirm the validity of the equation.

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*MrbpIvvh5XPzM6BtnIWGWw.png)

The verifier does not know _τ_, but he have the commitments \[_Q_(_τ_)\] and \[_P_(_τ_)\] from the prover. When we multiply both sides of the equation by the generator point \[G\], the scenario becomes more tractable.

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*3AmdFCfUXEmmCnK4arw63A.png)

Note: Notation _P_(_τ_)\[G\] is same as \[_P_(_τ_)\]

The verifier is aware of \[_Q_(_τ_)\] and \[_P_(_τ_)\] — the commitments corresponding to _Q_(_τ_) and _P_(_τ_) multiplied by \[G\]. The verifier can also compute _P_(_r_)\[_G_\] since _P_(_r_) is known. To resolve (_τ_−_r_)\[_G_\], elliptic curve pairing techniques are employed.

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*5erKOw0KO25tBRwlWEOpnA.png)

e — pairing function. G1 and G2 are know points.

The function _e_ represents a bilinear pairing, which takes a point from each of two separate elliptic curve groups and ‘pairs’ them to produce an element in a third target group. The bilinear property of this pairing function is that _e_(_k_⋅_G_1​,_G_2​)=_e_(_G_1​,_k_⋅_G_2​), where _k_ is a scalar and _G_1​,_G_2​ are points from the two respective groups. This property means that multiplying a point in one group by _k_ before applying the pairing function is equivalent to applying the pairing function first and then multiplying its result by _k_ in the target group.

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*koTS-AGybCEZBMZBl8Euow.png)

In the context of bilinear pairings:

1.  We establish that all previously evaluated elements are associated with a point on curve _G_1​.
2.  When we refer to ‘multiplication’ in the pairing process, for the term (_τ_−_r_)_Q_(_τ_) to interact with the pairing function _e_, we first map (_τ_−_r_) onto curve _G_1​ by multiplying it with _G_1​’s generator point, and similarly, map _Q_(_τ_) onto curve _G_2​.
3.  The commutative property of the pairing function allows us to flip the scalar (_τ_−_r_) with the commitment _Q_(_τ_), without affecting the result.
4.  Then, we apply the distributive property to ‘open the brackets,’ which means to distribute the multiplication across the terms within the pairing operation.

So now we can state that the verifier is aware of the commitments \[_Q_(_τ_)\]1​​, \[_P_(_τ_)\]1​​, and \[_P_(_r_)\]1​​, and can also compute _r_\[_G_2​\] because _r_ is known. As for _τ_\[_G_2​\] — this value (point) should come from the trusted setup, which is a part of the common reference string (CRS).

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*XvyJAq6U4rZkK4Jvh1f3dg.png)

Updated Trusted Setup

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*BqPc1uwWBo6g9QS-ch_aWQ.png)

Final equation. Without elliptic curve part equation would look next: Q(_τ)(τ-r) = P(τ) — P(r)_

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*y5m7pFY872co5IarOoTxvg.png)

KZG schema

With these elements, the verifier is in a position to check if the final equation, which embodies the proof of knowledge, holds true.

## Bathed Polynomial Commitment Schema

In PLONK, the prover is required to commit to not just one polynomial but to multiple polynomials. Although the prover and verifier could theoretically handle all the commitments one by one using the schema mentioned above, the creators of PLONK have designed an altered schema that allows committing to multiple polynomials within a single step. This is possible using method called Linear Independence.

Suppose we have three polynomials, _P_1​(_x_), _P_2​(_x_), and _P_3​(_x_), and our goal is to demonstrate that each polynomial evaluates to zero at a some value _r_. To efficiently prove this within a single step, we can construct a new polynomial that encapsulates _P_1​(_x_), _P_2​(_x_), and _P_3​(_x_).

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*D2XEMuW2ptyIiJRhEKt7MA.png)

New polynomial _F_(_x_) is a combination of _P_1​(_x_), _P_2​(_x_), and _P_3​(_x_). If we choose a value _r_ such that when evaluated, _P_1​(_r_), _P_2​(_r_), and _P_3​(_r_) yield the results 2, 3, and -5, respectively, we encounter an issue. The sum of these evaluations is zero, which might incorrectly suggest that each individual polynomial evaluates to zero at _r_ when, in fact, none of them do. This is a potential problem because our aim is to prove that each polynomial individually equals zero at _r_, not just their sum.

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*oTLkGIVLEHEXvI7s42xPgA.png)

To guarantee that each polynomial _P_1​(_x_), _P_2​(_x_), and _P_3​(_x_) individually equals zero at a specific point _r_, we choose a non-zero constant _v_ (for example, _v_\=2) and use it to scale the polynomials by increasing powers. We then define a new polynomial _F_(_x_) as _F_(_x_)=v⁰_P_1​(_x_)+_v¹P_2​(_x_)+_v²P_3​(_x_). When _F_(_r_) is evaluated and the result is zero, it implies that _P_1​(_r_), _P_2​(_r_), and _P_3​(_r_) must all be zero, because _v_ is non-zero and the polynomials are scaled by distinct powers of _v_, making them linearly independent. This clever approach allows us to confirm the individual zeros of _P_1​, _P_2​, and _P_3​ within a single combined equation.

In next article, we will discuss several other aspects that deserve attention before we commence our detailed work on the protocol.

[

## Under the hood of zkSNARKs — PLONK protocol: Part 2

### In the previous article of the PLONK series, we covered the core of the protocol, the KZG commitment scheme, and how…

medium.com



][9]

[1]: /@cryptofairy?source=post_page-----34bc406d8303--------------------------------
[2]: https://medium.com/coinmonks?source=post_page-----34bc406d8303--------------------------------
[3]: /@cryptofairy?source=post_page-----34bc406d8303--------------------------------
[4]: /m/signin?actionUrl=https%3A%2F%2Fmedium.com%2F_%2Fsubscribe%2Fuser%2Fb3a405d735c6&operation=register&redirect=https%3A%2F%2Fmedium.com%2Fcoinmonks%2Funder-the-hood-of-zksnarks-plonk-protocol-part-1-34bc406d8303&user=Crypto+Fairy&userId=b3a405d735c6&source=post_page-b3a405d735c6----34bc406d8303---------------------post_header-----------
[5]: https://medium.com/coinmonks?source=post_page-----34bc406d8303--------------------------------
[6]: /m/signin?actionUrl=https%3A%2F%2Fmedium.com%2F_%2Fvote%2Fcoinmonks%2F34bc406d8303&operation=register&redirect=https%3A%2F%2Fmedium.com%2Fcoinmonks%2Funder-the-hood-of-zksnarks-plonk-protocol-part-1-34bc406d8303&user=Crypto+Fairy&userId=b3a405d735c6&source=-----34bc406d8303---------------------clap_footer-----------
[7]: /m/signin?actionUrl=https%3A%2F%2Fmedium.com%2F_%2Fbookmark%2Fp%2F34bc406d8303&operation=register&redirect=https%3A%2F%2Fmedium.com%2Fcoinmonks%2Funder-the-hood-of-zksnarks-plonk-protocol-part-1-34bc406d8303&source=-----34bc406d8303---------------------bookmark_footer-----------
[8]: /coinmonks/trusted-setup-in-zksnarks-powers-of-tau-vs-lagrange-basis-7f12978f1eb9?source=post_page-----34bc406d8303--------------------------------
[9]: /@cryptofairy/under-the-hood-of-zksnarks-plonk-protocol-part-2-ee00d6accb4d?source=post_page-----34bc406d8303--------------------------------