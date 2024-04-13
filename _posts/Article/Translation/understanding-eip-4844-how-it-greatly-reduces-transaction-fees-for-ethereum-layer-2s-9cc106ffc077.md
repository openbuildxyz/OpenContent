---
title: "Understanding EIP-4844: How it Greatly Reduces Transaction Fees for
  Ethereum Layer 2's"
authorURL: ""
originalURL: https://blog.coinshares.com/understanding-eip-4844-how-it-greatly-reduces-transaction-fees-for-ethereum-layer-2s-9cc106ffc077
translator: ""
reviewer: ""
---

# Understanding EIP-4844: How it Greatly Reduces Transaction Fees for Ethereum Layer 2's

<!-- more -->

[

![Luke Nolan](https://miro.medium.com/v2/resize:fill:88:88/1*_CxaNUSj3a-nkuqajBGK8Q.png)









][1][

![CoinShares Research Blog](https://miro.medium.com/v2/resize:fill:48:48/1*2wcnHokRTIblWXkSuP18nQ.png)











][2]

[Luke Nolan][3]

·

[Follow][4]

Published in

[

CoinShares Research Blog

][5]

·

8 min read

·

Mar 18, 2024

[

][6]

\--

[][7]

Listen

Share

# **Executive Summary**

-   **EIP-4844 introduces “blobs” as a cheaper way for Layer 2 Rollups to post transaction data to Ethereum.**
-   **Blobs leverage temporary storage and a separate fee market, reducing transaction costs by up to 98% for some rollups.**
-   **This upgrade significantly enhances the scalability of Ethereum by enabling higher Transactions Per Second (TPS) on Layer 2 solutions.**
-   **Users transacting on Layer 2 rollups will benefit from much lower transaction fees, enhancing Ethereum’s network effect and ability to retain previously priced out users.**

# **Introduction**

The long awaited Dencun upgrade was implemented on Wednesday the 13th of March, after 5 successful testnet implementations. The upgrade includes [9 EIPs][8], with the most important, and the one we will be discussing in this article being EIP-4844.

EIP-4844, titled [_Shard Blob Transactions_][9], introduces a concept known as “Proto-danksharding”, named after the EIP authors and Ethereum Researchers Proto Lambda and Dankrad Feist. This upgrade significantly reduces transaction fees for rollups, as well as enhances the amount of transactions that Layer 2s can post to a single block.

In this article, we will be going over a refresher on Layer 2 rollups, the implementation of blobs and how this differs from the previous mechanism of transaction posting, how this will affect rollup economics, and what this means for the Ethereum scaling roadmap.

# **What are Layer 2 Rollups?**

Layer 2s are scaling solutions built on top of Ethereum, that aim to enhance transaction speed and reduce transaction fees for participants, as a result of the inherent limitations of Ethereum Layer 1 (~12 TPS and [high transaction costs in times of congestion][10]). Layer 2s manage to achieve this by processing and bundling transactions off chain, and then settling these batches’ final state on Layer 1. The key here is that, in simple terms, users transacting on Layer 2s get to share the cost of 1 large transaction by many participants, as opposed to the cost of a single transaction which would happen when transacting on Layer 1.

There are a number of types of Layer 2 solutions: Rollups, State Channels, Plasma Chains and Sidechains — each with their own design and implementations. The most popular are _rollups_, and will be the focus of this article, as EIP-4844 primarily impacts them, and not the other types of Layer 2 solutions.

There are 2 types of rollups:

**Optimistic Rollups:** Transactions posted by optimistic rollups to Layer 1 are assumed to be valid (hence _Optimistic_). Since there is a chance that transactions posted by these rollups are invalid, there is a 7 day challenge period before ultimate finality. If a transaction is deemed to be invalid, the update is reverted and the Layer 2 node that published the transaction is punished. A real world analogy for optimistic rollups would be like keeping a tab at a coffee shop: the barista keeps track of how much you owe, and at the end of the day, you pay the full sum. The barista trusts you will pay the tab (optimistic), and the manager might verify that the payment was actually made (challenge period).

**ZK-Rollups:** These rollups use cryptographic, _Zero-Knowledge_ proofs to prove that transactions are valid **at the time of** posting, offering faster finality than those of optimistic rollups, at the cost of being more computationally intensive, and thus handle slightly lower TPS than optimistic rollups. ZK-Rollups omit user-specific transaction data, enhancing privacy for its users; an analogy would be getting an anonymous credit check, where the agency can actually verify your score with a track record of proof, but doesn’t actually know who is getting checked.

# **How Layer 2s Post Transactions To Ethereum Pre-EIP-4844**

Optimistic Rollups, as previously mentioned, bundle and execute transactions off the main chain, and then settle them by posting the new state to Layer 1 (with the 7 day challenge period). ZK-Rollups do the same, with the key difference being they are proven valid from the get-go, through a _zero knowledge proof_. Both rollups use _Merkle Trees_, which are essentially compact representations of the _state_ that is posted back to Layer 1. Merkle Trees facilitate the creation of fraud proofs, verifying if transactions happened as claimed by the rollup.

![](https://miro.medium.com/v2/resize:fit:640/format:webp/0*w1bBt9SBfNcyUi6L)

_Source: CoinShares_

In both instances, as seen above, the aggregated transactions get posted to a smart contract on Layer 1, encoded by what is known as _calldata. Calldata_ was not created primarily to store data from Layer 2, but has been the appropriate outlet where all the transaction data gets posted and stored forever by validators and node operators. A nuance for optimistic rollups is that they have an execution environment called the Optimistic Virtual Machine (OVM), where general computation is run, generating the Merkle Tree which holds the proposed _state_.

The vast majority of costs that Layer 2s have when posting transactions to Layer 1 comes in the form of _calldata_. In fact, over **85% of the cost that Layer 2s have comes from the use of _calldata_.**

_Calldata_ is costly for a few reasons. Firstly, it directly consumes space and bandwidth on the network in perpetuity, requiring nodes to use up disk space to propagate transactions. Computational capability also is, of course, a limited resource on the blockchain, and there is only a finite amount available at any given moment in time. The second reason as to why _calldata_ is expensive, is because it is priced using the same gas market that all computation competes for. Every byte of data posted through _calldata_ is priced using the same gas price (for _calldata_ the gas cost is ~16 gas per byte) that all other Layer 1 transactions post. This means in times of high congestion, the cost for Layer 2 transactions, although greatly reduced due to their shared nature as explained previously, can still increase 3–10x depending on network congestion.

And so, with this in mind, came the introduction of _blobs_.

# **How Layer 2s Post Transactions to Ethereum Post-EIP-4844**

_Blobs_ offer an alternative to _calldata_ as the storage outlet for Layer 2s posting transactions to Layer 1. _Blobs_ are temporary data _chunks_ that get attached to blocks, allowing for far more transaction data to be posted into each block by rollups at a fraction of the cost. The main differences between using blockspace (_calldata)_ and _blobspace (blobs)_ is:

![](https://miro.medium.com/v2/resize:fit:640/format:webp/0*mYPKJPb4Q7eOp9UU)

_Source: CoinShares_

The main cost reduction comes in two forms: Firstly, blobs are not visible to the EVM, and are only stored temporarily in the beacon node (consensus client) as opposed to the execution client, and pruned after ~2 weeks (may vary). This reduction of computational load makes it far more cost effective.

The second, and key attribute that makes _blobs_ significantly cheaper is that the data posted to them is priced using a **separate, segregated fee market**. As previously stated, _calldata_ pricing is subject to demand to use Layer 1, and in times of high network congestion, can increase very quickly.

By having a separate fee market, blob transactions compete with each other, and the pricing is greatly reduced. The actual gas _price_ per blob varies, but the median gas fee across all Layer 2’s has reduced dramatically so far (non-exhaustive list of protocols):

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*btXgF6rKAF297J0Hm6GPvw.png)

Source: CoinShares, Dune Analytics

Popular prediction market Polymarket shows that the market believes the gas price per blob after [1 month of EIP 4844 will be <0.001 ETH][11]. As well [pointed out by Vitalik][12], at an average gas price of 30 gwei, this would be 60x cheaper than using calldata. One important thing to note, is that when using _calldata_, Layer 2 solutions get charged by the amount of data used, and for _blobs_, a Layer 2 must buy an **entire** blob to post their data.

Given the segregated fee market, even in times of high _blob_ demand, it is likely to always be cheaper than using _calldata_. A secondary effect that EIP-4844 will have is the increase in Transactions Per Second (TPS) that Layer 2s can handle, which we have [outlined in our outlook earlier this year.][13]

# **Conclusion**

Overall, EIP 4844 reduces transaction fees on rollups by at least 2x (zk rollups like zksync above will reduce fees much more at bigger scale due to their high base overhead) and in some cases, up to 60x+. This will have a huge impact on rollup unit economics, which warrants its own, deeper analysis which we will save for a later date, once we have more historical data to annualize profit margins and the likes. End users will benefit from extremely cheap fees on Layer 2s, and as seen above, we are already seeing this in action.

The Ethereum developer community has decided that, at least in the short and medium term, the only viable scaling roadmap with minimal secondary tradeoffs is through rollups, and Layer 1 will continue to be a secure settlement layer, with low TPS (~12) and higher gas fees in times of congestion.

We expect that over time, as Layer 2 implementations become more sophisticated, as well as the applications and services that get built on top of them, that low value users who are priced out of execution on Layer 1, will move over fully to using rollups. High value transactions on Layer 1, where the transaction fee still represents a low % of the total cost, are likely to remain, given the inherent higher settlement assurances and proven track record with 100% uptime.

Overall, what is clear, is that TPS for Layer 1 + Layer 2 is set to increase greatly, and the reduction in transaction costs will, in the long term, strengthen Ethereum’s network effect and help to stop pricing out users from using the blockchain.

![](https://miro.medium.com/v2/resize:fit:640/format:webp/0*R3MALLJozzUUpPU8)

\* Estimates from [Source][14] \*\*4844 TPS gain = ((128k/byte cost)\*3) / (15m / L1 gas cost)) / 12 — Baseline TPS is based on Gas Target of 15m and not Gas Limit of 30m as continuous block production of 30m Gas is unsustainable

The table above shows us the theoretical TPS that Layer 2’s can achieve to meet demand, and given the successful implementation of Dencun, a pickup in demand for rollups has already been observed, reflecting a noticeably higher aggregate TPS for Layer 2’s.

![](https://miro.medium.com/v2/resize:fit:640/format:webp/0*rx9XpWrEfxaOleKC)

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*hEh-EVmAgyuwle86R2ZUoA.png)

[1]: https://medium.com/@lnolan_91005?source=post_page-----9cc106ffc077--------------------------------
[2]: https://blog.coinshares.com/?source=post_page-----9cc106ffc077--------------------------------
[3]: https://medium.com/@lnolan_91005?source=post_page-----9cc106ffc077--------------------------------
[4]: https://medium.com/m/signin?actionUrl=https%3A%2F%2Fmedium.com%2F_%2Fsubscribe%2Fuser%2F9d2a1c0c2531&operation=register&redirect=https%3A%2F%2Fblog.coinshares.com%2Funderstanding-eip-4844-how-it-greatly-reduces-transaction-fees-for-ethereum-layer-2s-9cc106ffc077&user=Luke+Nolan&userId=9d2a1c0c2531&source=post_page-9d2a1c0c2531----9cc106ffc077---------------------post_header-----------
[5]: https://blog.coinshares.com/?source=post_page-----9cc106ffc077--------------------------------
[6]: https://medium.com/m/signin?actionUrl=https%3A%2F%2Fmedium.com%2F_%2Fvote%2Fcoinshares%2F9cc106ffc077&operation=register&redirect=https%3A%2F%2Fblog.coinshares.com%2Funderstanding-eip-4844-how-it-greatly-reduces-transaction-fees-for-ethereum-layer-2s-9cc106ffc077&user=Luke+Nolan&userId=9d2a1c0c2531&source=-----9cc106ffc077---------------------clap_footer-----------
[7]: https://medium.com/m/signin?actionUrl=https%3A%2F%2Fmedium.com%2F_%2Fbookmark%2Fp%2F9cc106ffc077&operation=register&redirect=https%3A%2F%2Fblog.coinshares.com%2Funderstanding-eip-4844-how-it-greatly-reduces-transaction-fees-for-ethereum-layer-2s-9cc106ffc077&source=-----9cc106ffc077---------------------bookmark_footer-----------
[8]: https://eips.ethereum.org/EIPS/eip-7569
[9]: https://eips.ethereum.org/EIPS/eip-4844
[10]: https://cn.etherscan.com/gasTracker#heatmap_gasprice
[11]: https://polymarket.com/event/gas-price-per-blob-1-month-after-eip-4844?tid=1710176462068
[12]: https://twitter.com/VitalikButerin/status/1764139239233749047?ref_src=twsrc%5Egoogle%7Ctwcamp%5Eserp%7Ctwgr%5Etweet
[13]: https://a.storyblok.com/f/155294/x/5f04cf4bff/coinshares-outlook-2024.pdf
[14]: https://vitalik.ca/general/2021/01/05/rollup.html