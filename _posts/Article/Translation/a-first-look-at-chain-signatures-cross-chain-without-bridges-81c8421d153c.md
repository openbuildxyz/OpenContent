---
title: "A First Look at Chain Signatures: Cross-Chain Without Bridges"
authorURL: ""
originalURL: https://medium.com/@ProximityFi/a-first-look-at-chain-signatures-cross-chain-without-bridges-81c8421d153c
translator: ""
reviewer: ""
---

# A First Look at Chain Signatures: Cross-Chain Without Bridges

<!-- more -->

[

![Proximity](https://miro.medium.com/v2/resize:fill:88:88/1*cwvpLsOGCf3YIM_lBN7qEg.jpeg)









][1]

[Proximity][2]

·

[Follow][3]

12 min read

·

Mar 12, 2024

[

][4]

\--

[][5]

Listen

Share

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*nQP0q_CaPSF5i4udbF5i3Q.png)

As 2023 witnessed the Cambrian explosion of L2s and modular blockchains, it simultaneously observed the pushback against Web3’s increasing complexity and fragmentation. Reflected in the buzzwords of _abstraction_ and _aggregation_, we saw the ongoing discourse about account abstraction on Ethereum, Polygon’s [AggLayer][6], and the chain abstraction narrative spearheaded by NEAR or Agoric, to name a few. Though all different in detail, the underlying idea is the same: Web3 is vowing to become simpler, unified, and more usable.

In the recent [chain abstraction thesis][7], Illia Polosukhin, NEAR Protocol co-founder and NEAR Foundation CEO, hinted at the idea of “account aggregation:”

> _From a user perspective, this should be a single account where they interact with apps on different chains, and assets either get bridged or swapped automatically. I call this “account aggregation” \[…\]._

[Account aggregation][8] is the ability to sign transactions on any given chain from a single (NEAR) account, through a single interface, in a single transaction. In it lies the idea of chain abstraction: instead of having to separately manage accounts for multiple chains, often requiring a different wallet interface (e.g. Metmask for EVM, Keplr for Comos chains, Phantom for Solana), and going through several hurdles to send assets from one chain to another, the entire experience is boiled down to a single layer and a single transaction for the end user.

The technology enabling this is **Chain Signatures**, a novel threshold signature protocol utilizing an MPC (multi-party computation) signer network on NEAR. With Chain Signatures live on testnet, let us look at the technology and its potential for cross-chain development.

# **Challenges in Cross-Chain Interoperability**

Recent peaks in bridge volumes testify to the growing demand for cross-chain transactions.

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*JOunoaP9V9m2EvG4pY6oHA.png)

Bridge volume has more than doubled since November 2023. Source: [DeFiLlama][9]

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*jLQpUUyD74N2kaOj4gbqAA.png)

Bridge TVL has also doubled since November 2023. Source: [DeFiLlama][10]

However, cross-chain bridging is not without its challenges. First and foremost, bridges have been proven time and again to be **vulnerable to hacks**. Bridges inherently have a large attack surface.

[

## Cross-chain Bridge Orbit Chain Confirms Exploit Worth Over $81 Million | CoinMarketCap

### Cross-chain bridge Orbit Chain has confirmed that it suffered an unauthorized breach of access to its ecosystem on…

coinmarketcap.com



][11]

[

## DeFi hackers stole $3.2bn last year amid 35% surge in 'bridge' exploits

### Hackers stole more than $3.2 billion from decentralised finance protocols in 2022. So-called crypto bridges were the…

www.dlnews.com



][12]

Secondly, bridges today **lack comprehensive chain support**. Many bridges or messaging protocols lack support for non-EVM chains, not to mention non-smart contract chains like Bitcoin, Doge, or Ripple. While bridges can cover a lot of space, they are limited from achieving 100% interoperability.

Finally, **bridge UX is inconsistent and cumbersome**. Not only is every bridge designed differently — which means a learning curve for each one — but each bridge operates at a different speed. An additional UX hurdle comes when you try to bridge, only to learn that you need to acquire another token separately to pay for gas, for which you normally need to use a centralized exchange. A further challenge is that bridged assets are often a “wrapped” version of an asset, which is different from its native version on the origination chain, and may not even be the version the user needs for their intended action. After bridging, the user may end up with an unsupported token or a token with low liquidity.

# **Solving Cross-Chain Challenges with Programmable MPCs**

Fortunately, there is a new design pattern for cross-chain messaging that can alleviate the challenges of bridging. Enter _programmable MPCs_.

MPC, or multi-party computation, is a privacy-preserving coordination protocol in cryptography where multiple parties can perform a computation together without revealing the data to each other. Many of us are familiar with the concept from MPC wallets such as the Coinbase Wallet or Fireblocks, where the private key of the user’s wallet is shared among multiple parties to eliminate a single point of failure. In this approach, the user retains a key, while centralized parties retain the other keys.

![](https://miro.medium.com/v2/resize:fit:640/format:webp/0*jVosyJN5VnV9j_Dl.png)

Source: [Seedless Self-Custody: On MPC and Smart Contract Wallets][13]

Programmable MPCs allow developers to tap into this and also decentralize key sharing. Core advantages include:

-   **Comprehensive coverage**: programmable MPCs can support _any_ chain including non-smart contract chains such as Bitcoin, Doge, or Ripple
-   **Instant support**: as long as the MPC provider supports the elliptic curve for the chain in question, chain support is instantaneous and does not require separate integration efforts (as is the case for bridges)

e.g. ECDSA-based chains: EVM, Bitcoin, many Cosmos chains, etc; EdDSA-based chains: NEAR, Solana, Cardano

-   **Standardized DevX and UX**: For devs, it’s just a simple API to utilize. For users, it means the same transaction speed across all chains in cross-chain swaps and even no gas token required, depending on the implementation (see **Multichain Gas Relayer** below). This marks the true potential for chain abstraction, where users do not have to know which chain they are on.

# **Chain Signatures: NEAR’s Approach to Programmable MPCs**

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*6UHadZfBuq-UtCaCZDnC4A.png)

Today, there are mainly two approaches to programmable MPCs. One is deposit-based and the other account-based. Deposit-based MPC networks maintain deposit addresses to create a canonical bridged asset. This is the case of Dfinity $ETH or Zetachain $BTC; Thorchain, where users deposit to a specific address to enact swaps with another user; or Axelar, where, as a generalized bridge protocol, it maintains the deposit address for the user without the user having direct access to it.

The efforts of NEAR and Lit Protocol fall under the category of account-based programmable MPCs. On NEAR, such an account-based approach to programmable MPCs is called _Chain Signatures:_ threshold signatures controlled by NEAR accounts and smart contracts and signed by MPC nodes on the NEAR network.

At a high level, the mechanism of Chain Signatures is as follows:

![](https://miro.medium.com/v2/resize:fit:640/format:webp/0*zXoUHLExpV6c3mxb)

Source: [Unlocking Web3 Usability with Account Aggregation][14]

1.  An account on NEAR, which can also be a smart contract, requests validators (i.e. MPC nodes) to sign an arbitrary payload (e.g. a transaction on Bitcoin or Optimism)
2.  Validators sign the payload via MPC and relay it to the destination chain
3.  This entire process is abstracted away for the user: as a user, you are simply signing a transaction on a NEAR account and getting the amount on the destination chain (e.g. Bitcoin, Optimism, etc).

One of the biggest advantages of the NEAR account model is that it allows end users to hold any number of sub-accounts under a single top-level account. This means that with Chain Signatures, users can control 10s of 1000s of accounts across different chains from a single NEAR account. _This_ is what we mean by “account aggregation.”

For devs, Chain Signatures is a simple API that looks something like this:

![](https://miro.medium.com/v2/resize:fit:640/format:webp/0*OlsABXGzbj9bhsjm)

-   **Function call** to sign payload
-   **Key derivation path**: because a single NEAR account can have an almost infinite number of accounts it can sign for on any given chain, this designates which account is being signed for
-   **Key type**: designates the elliptic curve of the chain in use (at launch, Chain Signatures will first support ECDSA, then later, EdDSA).

In the following [example][15], Proximity Labs’ Matt Lockyer demonstrates Chain Signatures by sending $ETH between two accounts on the Ethereum Sepolia network using a single NEAR account, on a single interface, and in a single transaction.

Demo Video by Matt

Here, the Ethereum address (from which your $ETH will be sent) is generated from a path variable, akin to an HD path on a Ledger account. This address is an offset of your NEAR account that is uniquely generated for this purpose.

![](https://miro.medium.com/v2/resize:fit:640/format:webp/0*Kr2ntbyGPXyUXeK1)

When you sign the transaction, the MPC contract generates a signature for this particular Ethereum transaction without revealing the private key to any party.

![](https://miro.medium.com/v2/resize:fit:640/format:webp/0*SFTa2ZKssUIoxrYb)

The [example][16] above is open-source for anyone who wants to check it out.

# **Multichain Gas Relayer**

An additional protocol that can help maximize the potential of account aggregation is the multichain gas relayer, which realizes gas abstraction.

Let us visualize this flow with an [actual implementation of chain signatures by Sweat Wallet][17] to enable users to send $BNB from NEAR to BNB Smart Chain from the Sweat Wallet (a NEAR account).

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*00RpYpSg2ev8nueeWpJxew.png)

1.  The user initiates a signature from the Sweat Wallet (= NEAR account) to send 0.02 $BNB on NEAR to an address on BNB Smart Chain.
2.  The MPC nodes sign the payload and send it to the relayer network.
3.  The gas relayer picks up the transaction; the user pays for gas in $SWEAT on the NEAR network.
4.  The gas relayer takes the payment and funds the destination address on BNB with the appropriate gas tokens (in this case, $BNB). The relayer network then routes the signed transaction. All of this happens in the same block (2–3 seconds).
5.  The user has successfully sent 0.02 $BNB to their BNB address from their NEAR account.

When used in combination with Chain Signatures, the multichain gas relayer prevents apps and users from having to deal with multiple gas tokens on multiple chains. Behind the scenes, the gas relayer handles the gas payment on the respective chains — the user only needs to use one token for gas.

Check out the [demo][18] by Sweat Wallet. The transaction hash is [here][19].

# **Use Cases**

The examples above illustrate Chain Signatures use cases in which a single NEAR account signs a cross-chain transaction. A whole new design space for cross-chain _dApps_ emerges when we consider that a NEAR account is _natively a_ _smart contract._

Here are a few:

1.  **DeFi on Non-Smart Contract Chains**: Chain Signatures can enable DeFi on non-smart contract chains such as Bitcoin, Doge, or Ripple, which thus far only supports transfers on the network. NEAR smart contracts can act as escrow contracts and manage who controls what. On top of this primitive, you can build swaps or lending protocols, for instance, that support any asset on any chain, including assets in unique states (e.g. staked, in liquidity pools). Imagine swapping your staked $TIA for an NFT on Solana or using it as collateral to borrow $ETH on Optimism.
2.  **Multichain account abstraction** (with support for gas relayer): Because NEAR accounts are natively smart contracts, there is a lot of flexibility out of the box: any NEAR account can have any number of keys, rotate keys for security, and have multi-signer patterns. Additionally, with the Multichain Gas Relayer, you can even abstract the complexity of multiple gas tokens for different chains. With Chain Signatures, one can essentially “NEAR-ify” any account on any chain by extension, thereby bringing about account abstraction on a multichain scale, from Ethereum to Solana to Bitcoin.

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*Xh322gSp73akKiz91R-nSA.png)

The NEAR account model natively embodies the idea of account abstraction. (Source: [Unlocking Web3 Usability with Account Aggregation][20])

1.  **Bridgeless Cross-chain DeFi**: As seen in the overview above, one of the most powerful things about Chain Signatures is that they eliminate the need for bridging and instead power cross-chain transactions via an MPC signature protocol. We can imagine _net new DeFi products_ such as,

-   Native cross-chain swaps (e.g. swap $XRP on Ripple for an NFT on Solana)
-   Cross-chain lending orderbook (e.g. use X on Optimism as collateral to borrow Y on Arbitrum)
-   Restake any asset on any chain; even handle the reward or slashing conditions from NEAR
-   Trustless Bitcoin Ordinals Marketplace

Additionally, smart contract-based Chain Signatures would open up the possibility of privacy-focused apps as well as trustless multichain deployments.

# **Case Study: Bitcoin Ordinals Marketplace**

Let us take one of the use cases and understand how it would work: a trustless Bitcoin Ordinals marketplace. Currently, one implementation of this idea is underway by the [EastBlue][21] team.

In this scenario, we are dealing with a trade between a seller whose asset (Ordinal) is on Bitcoin and a buyer whose asset ($USDC) is on NEAR; but this can be generalized to any asset on any chain.

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*M1akep7qIScE18YMVxZiQg.png)

1.  The marketplace contract exists on NEAR.
2.  The seller generates a deposit account on Bitcoin via the marketplace contract.
3.  O_nly_ the marketplace contract can request the MPC signers to sign a transaction on behalf of this Bitcoin account.
4.  Within the smart contract state, the seller is recognized as the owner of this Bitcoin account and only they are allowed to deposit and withdraw the Ordinal out of that account.
5.  The seller deposits the Ordinal to the Bitcoin account and creates a listing for 10 $USDC.
6.  To protect the buyer, the contract disallows the seller from withdrawing the Ordinal if there is an open order to their listing. No changes on the Bitcoin network will affect the account, only the marketplace contract and allow withdrawals.
7.  The buyer deposits $USDC to the marketplace contract.
8.  The buyer accepts the 10 $USDC listing.
9.  The sale is executed. In other words, the marketplace contract has verified that two users have agreed to swap their accounts and atomically executes the swap in a single block (2–3 seconds).
10.  The seller now has control over the 10 $USDC deposited by the buyer and withdraws the funds. In turn, the buyer is now the owner of the Bitcoin account holding the Ordinal and can withdraw.

# **Conclusion**

Chain Signatures is currently on testnet and awaiting mainnet release before the end of Q1. In combination with the flexible NEAR account model and the Multichain Gas Relayer, Chain Signatures have great potential to bring cross-chain interoperability to the next level. Furthermore, Chain Signatures have the potential to drive chain abstraction, where the complexity and fragmentation of today’s Web3 are abstracted away with a single user layer that can interact with any asset on any chain.

If you are a developer, now is the perfect time to get acquainted with the concept and explore this technology for your cross-chain dApp. If you are building, do not hesitate to reach out to us via [hello@proximity.dev][22] or on the [Chain Abstraction Dev Chat][23]. We are more than happy to provide technical assistance, strategic advice on product and business development, as well as GTM support.

_The content of this article is based on Kendall Cole’s keynote at ETH Denver and recreated by Rim Berjack. Find the full talk_ [_here_][24]_._

# **Chain Signatures Resources**

_Find all the resources below and more in this_ [_Chain Signatures Linktree_][25]_._

-   [Chain Abstraction Thesis][26]
-   [Account Aggregation Deep Dive][27]
-   [Chain Signatures Overview: Cross-Chain without Bridges (Video)][28]
-   [Chain Signatures Docs][29]
-   [NEAR Account Model Docs][30]
-   [Chain Signatures API][31]
-   [Generate MPC Private Keys for NEAR][32]
-   [Github Discussions][33]
-   [Sample Request and Response][34]
-   [Multichain Gas Relayer][35]
-   (Example) Send ETH on Ethereum from a NEAR account: [Demo video][36]; [Open-source component][37]
-   (Example) Sign a transaction on BNB from a NEAR wallet: [Demo video][38]; [Transaction hash][39]
-   [Official Chain Abstraction Dev Chat][40]

# **Further Reading**

[BOS, The User Layer for Chain Abstraction][41]

_Disclosures: Proximity Labs holds $NEAR and other tokens or investments that may be associated with protocols or projects mentioned in this article. The authors of this article have not purchased or sold any token for which the authors had material non-public information while researching or drafting this report. The statements and content in this article should not be misconstrued as a recommendation to purchase or sell any token, or to use any protocol. This article also contains forward-looking statements about third-party projects that the authors have no control over and, as such, actual future developments may be substantially different from the expectations described in the forward-looking statements for a number of reasons, including those that are not under the control of the authors. The content of this article reflects the opinions of its authors and is presented for informational purposes only. This is not and should not be construed to be investment advice._

[1]: /@ProximityFi?source=post_page-----81c8421d153c--------------------------------
[2]: /@ProximityFi?source=post_page-----81c8421d153c--------------------------------
[3]: /m/signin?actionUrl=https%3A%2F%2Fmedium.com%2F_%2Fsubscribe%2Fuser%2F41d0f266807b&operation=register&redirect=https%3A%2F%2Fmedium.com%2F%40ProximityFi%2Fa-first-look-at-chain-signatures-cross-chain-without-bridges-81c8421d153c&user=Proximity&userId=41d0f266807b&source=post_page-41d0f266807b----81c8421d153c---------------------post_header-----------
[4]: /m/signin?actionUrl=https%3A%2F%2Fmedium.com%2F_%2Fvote%2Fp%2F81c8421d153c&operation=register&redirect=https%3A%2F%2Fmedium.com%2F%40ProximityFi%2Fa-first-look-at-chain-signatures-cross-chain-without-bridges-81c8421d153c&user=Proximity&userId=41d0f266807b&source=-----81c8421d153c---------------------clap_footer-----------
[5]: /m/signin?actionUrl=https%3A%2F%2Fmedium.com%2F_%2Fbookmark%2Fp%2F81c8421d153c&operation=register&redirect=https%3A%2F%2Fmedium.com%2F%40ProximityFi%2Fa-first-look-at-chain-signatures-cross-chain-without-bridges-81c8421d153c&source=-----81c8421d153c---------------------bookmark_footer-----------
[6]: https://polygon.technology/blog/aggregated-blockchains-a-new-thesis
[7]: https://pages.near.org/blog/why-chain-abstraction-is-the-next-frontier-for-web3/
[8]: https://pages.near.org/blog/unlocking-web3-usability-with-account-aggregation/
[9]: https://defillama.com/bridges
[10]: https://defillama.com/categories?pool2=true
[11]: https://coinmarketcap.com/academy/article/cross-chain-bridge-orbit-chain-confirms-exploit-worth-over-dollar81-million?source=post_page-----81c8421d153c--------------------------------
[12]: https://www.dlnews.com/articles/defi/defi-hackers-stole-32bn-amid-surge-in-crypto-bridge-hacks-lazarus-group/?source=post_page-----81c8421d153c--------------------------------
[13]: /1kxnetwork/wallets-91c7c3457578
[14]: https://pages.near.org/blog/unlocking-web3-usability-with-account-aggregation/
[15]: https://test.near.social/md1.testnet/widget/chainsig-sign-eth-tx
[16]: https://test.near.social/md1.testnet/widget/chainsig-sign-eth-tx
[17]: https://www.youtube.com/shorts/QkrgMbDrVRg
[18]: https://www.youtube.com/shorts/QkrgMbDrVRg
[19]: https://testnet.bscscan.com/tx/0xdd97e8b7cb01b87f58408891f90f4cfd958f373e032451bf78b75d3c6a23df5c
[20]: https://pages.near.org/blog/unlocking-web3-usability-with-account-aggregation/
[21]: https://eastblue.io/
[22]: mailto:hello@proximity.dev
[23]: https://t.me/+RXYjlPob_XM5N2Ex
[24]: https://www.youtube.com/watch?v=gCkOTjbJD3c
[25]: https://linktr.ee/chainsignatures
[26]: https://pages.near.org/blog/why-chain-abstraction-is-the-next-frontier-for-web3/
[27]: https://pages.near.org/blog/unlocking-web3-usability-with-account-aggregation/
[28]: https://www.youtube.com/watch?v=gCkOTjbJD3c
[29]: https://docs.near.org/abstraction/chain-signatures
[30]: https://docs.near.org/concepts/basics/accounts/model
[31]: https://github.com/near/near-fastauth-wallet/blob/dmd/chain_sig_docs/docs/chain_signature_api.org
[32]: https://github.com/mattlockyer/near-mpc-kdf
[33]: https://github.com/near/NEPs/issues/503
[34]: https://testnet.nearblocks.io/txns/9d1DHqWY3fwjScujsqEhJER8qnc5so33T2VbeCVBcPFQ
[35]: https://docs.near.org/develop/relayers/multichain-server
[36]: https://www.youtube.com/watch?v=0h-uY_fEoZE
[37]: https://test.near.social/md1.testnet/widget/chainsig-sign-eth-tx
[38]: https://www.youtube.com/shorts/QkrgMbDrVRg
[39]: https://testnet.bscscan.com/tx/0xdd97e8b7cb01b87f58408891f90f4cfd958f373e032451bf78b75d3c6a23df5c
[40]: https://t.me/+RXYjlPob_XM5N2Ex
[41]: /@ProximityFi/bos-the-universal-access-layer-to-the-blockchain-71ad392f5a2d