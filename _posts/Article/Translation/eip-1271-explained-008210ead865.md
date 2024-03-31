---
title: EIP-1271 Explained
authorURL: ""
originalURL: https://blog.cow.fi/eip-1271-explained-008210ead865
translator: ""
reviewer: ""
---

# EIP-1271 Explained

<!-- more -->

## EIP-1271 is an important upgrade to the Ethereum network that’s stirred up quite a buzz, but what exactly is it?

[

![CoW Protocol](https://miro.medium.com/v2/resize:fill:88:88/1*OCgwNH8S6Pzr-TGJKUjrcw.png)









][1]

[CoW Protocol][2]

·

[Follow][3]

8 min read

·

Jan 29, 2024

[

][4]

\--

[][5]

Listen

Share

![](https://miro.medium.com/v2/resize:fit:640/format:webp/0*C5Y8qyY5EkFLB10B)

EIP-1271 (also known as ERC-1271) is an Ethereum improvement that enables smart contracts to validate signatures, allowing them to sign transactions just like traditional EOA wallets.

While at first EIP-1271 may seem like a niche technical bit of implementation, the standard unlocks a whole world of functionality for smart contracts that includes intent-based trading, advanced order types, and every other type of blockchain interaction that requires a wallet signature.

# What is EIP-1271

EIP-1271 serves as a “[standard signature validation method for contracts][6],” which gives smart contracts the ability to verify whether a signature on behalf of a given contract is valid — a feature that Externally Owned Accounts (EOA) already have built-in.

Before EIP-1271, smart contracts could send transactions, but they could not sign messages like traditional wallets. EOA have private keys which they use for signatures, validating that the message came from that particular wallet. Smart contracts, on the other hand, do not have private keys, so EIP-1271 was introduced as a standard way for smart contracts to validate a signature, enabling them to also sign messages.

## EIP-1271 vs ERC-1271

“EIP-1271” and “ERC-1271” are sometimes used interchangeably as terms that refer to the same concept. Ethereum accepts ideas for changes to the Ethereum network through a proposal process, hence the name “Ethereum Improvement Proposal” (EIP). “ERC,” on the other hand, stands for “Ethereum Request for Comment” and gets assigned to additions to the Ethereum network which do not change the underlying protocol itself.

This means that “EIP-1271” is more accurately called “ERC-1271,” as the addition does not change protocol functionality. As mentioned earlier, however, the terms are often used interchangeably.

# Why is EIP-1271 so useful?

Over the last few years, Ethereum has seen a boom in specialized smart contracts leveraging ERC-1271 to sign messages. Signed messages are necessary for a variety of things including placing orders on decentralized exchanges using off-chain order books, verifying that a given wallet belongs to a user, and much more.

ERC-1271-enabled smart contracts, however, unlock a whole new set of use-cases. This is because the ERC-1271 “isValidSignature” method can take in any arbitrary order logic, turning smart contract wallets into autonomous agents that can automatically sign messages based on on-chain conditions.

Let’s take a look at the various use-cases that ERC-1271 makes possible for smart contracts.

## Off-Chain Order Books

One of the most common uses of ERC-1271 involves orders on decentralized exchanges (DEX’s). Some exchanges, like CoW Protocol and Matcha, use an off-chain order book to collect signed messages before they become orders, allowing for additional optimizations. In order for smart contracts to be able to trade using these systems, they need ERC-1271 to validate signatures.

Limit orders in particular are a common type of DEX order that relies on an off-chain order book, so smart contracts require ERC-1271 in order to place limit orders.

## Sign-to-Verify

Authentication is a necessity in web3.

Many web3 applications require users to sign a message proving that they control a connected wallet in order to use the application. NFT marketplaces such as OpenSea and Rarible, meanwhile, require users to sign a message to place bids, update profile images, and perform other gas-less transactions.

In both of these cases, smart contract wallets leverage ERC-1271 to sign verification messages.

## Intents for Smart Contracts

Recently, many DeFi solutions have begun relying on [intents][7] as a mechanism for providing users with better prices, MEV protection, advanced order types, and a whole host of other benefits.

CoW Protocol is a meta DEX aggregator that pioneered many use-cases of intents, all of which rely on signed messages. Regular EOA orders are signed ahead of time, but thanks to ERC-1271, smart contract orders can hold off on the order signature until settlement time and check that conditions such as oracle prices, current balances, block times, and more are met before the trade goes through. This unlocks a number of unique trading strategies only available through smart contract wallets.

**Milkman for Safe Trading**

In collaboration with the [CoW Grants program][8], the [Yearn.fi][9] team launched [Milkman][10] to streamline DAO trades and protect them from MEV. The DAO governance process means that significant time passes between the time a trade is proposed and when the trade is actually executed. During this time, the market can move, invalidating the trade, or worse — an MEV searcher can lie in wait for the exact trade time and manipulate prices to exploit the trade.

Milkman provides a solution by allowing the DAO to specify a price feed which the smart contract checks at the moment of execution, making a trade based on the up-to-the-second price of the asset.

This means that DAOs no longer have to commit to an execution price ahead of time, but can leverage smart contracts and EIP-1271 to make a conditional order whose price is decided at the moment of execution.

**Advanced Order Types**

Smart contract signatures on CoW Protocol allow for a number of advanced order types. One notable example is CoW Protocol’s [TWAP orders][11], which use ERC-1271 smart contract signatures to autonomously place orders in even time intervals.

**The Programmatic Order Framework**

The [Programmatic Order Framework][12] from CoW Protocol takes full advantage of ERC-1271 by allowing anyone to code custom order logic for any type of order.

Programmatic orders execute indefinitely into the future based on on-chain conditions. These conditions can include any arbitrary logic including checking on-chain prices, smart contract wallet balances, block times, volumes, and more.

There are already a number of deployed use-cases for ERC-1271-enabled programmatic orders including:

-   Automated DAO operations including payroll, treasury diversification, and fee collection
-   Portfolio rebalancing based on the state of assets in a smart contract wallet
-   Special order types including stop-loss, good-after-time, and take-profit

## Any Application Requiring a Signature

We’ve covered a number of the use-cases that require signed messages, but ERC-1271 enables a potentially limitless number of uses… Any time a smart contract wants to use an application that requires a wallet signature, they need ERC-1271.

Here are a few final examples requiring signatures:

-   **Gasless Trading**: Some decentralized exchanges leveraging intents have begun offering “gasless trading,” a feature which allows users to make trades without paying for gas in ETH. On CoW Protocol, for example, all trades take gas fees in the sell token, leveraging signed messages to specify the order details.
-   **Native Token Flow**: On CoW Protocol, users can trade ETH natively, without having to use the ERC-20 “WETH” wrapped version. This is once again enabled by signed messages which specify the order details while bonded third parties ([solvers][13]) handle the execution.
-   **Web3 Login**: Services such as [Dynamic.xyz][14] build account abstraction functionality which allows users to use decentralized apps with just a traditional web2 email address. When using smart contract wallets, these services heavily rely on ERC-1271 to provide full signing functionality.

# How EIP-1271 works

With the introduction of ERC-1271, smart contracts can now define an “isValidSignature” method which determines the conditions necessary for the smart contract to verify a signature and execute a transaction.

This implementation gives smart contracts a leg up over EOAs when it comes to signing transactions. This is because the “isValidSignature” method can include any arbitrary logic, allowing each smart contract to define its own parameters for generating a valid signature.

Some example criteria for returning a valid “isValidSignature” include:

-   Signatures from a set of hard-coded EOA addresses (used in Safe’s multi-signature model)
-   On-chain conditions like time passing between orders or reaching a given oracle price

If the owners of the smart contract would like to delegate the order placement to someone else, they can code the necessary conditions for validating a signature in the “isValidSignature.” Whenever anyone calls the smart contract, it will check these conditions and only execute when they are met.

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*ENRRQis22l7l-FFS4Nwy5g.png)

ERC-1271’s “isValidSignature” method

The method takes a “bytes32 hash” representing an order hash and an arbitrary byte array which is where the conditions for the signature go. This array can be any number of things — an EOA signature from an owner or delegate of the smart contract, the smart contract checking an oracle or an on-chain state to determine if the order should execute, or any other arbitrary logic.

Finally, the “isValidSignature” method returns a response that determines whether the order should execute.

In essence, the entire signature verification process looks like this:

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*ZM3Ar3SbDTyDZ-66oRykWQ.png)

An order that needs to be signed is created

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*GMEOyrtJm19x0uBXuuX4LQ.png)

The order generates an order hash

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*Pveow971Ch3TRKHKnTBs_A.png)

The smart contract’s “isValidSignature” method is called with the hash and signature passed in as parameters

# EIP-1271: Enabling the future of smart contract trading

As Ethereum adoption grows, the needs of traders become more complex, eclipsing the built-in functionality of traditional EOAs. ERC-1271 opens the door to a new set of primitives for trading using smart contracts which can create autonomous, conditional orders that provide much greater trading power for users.

CoW Protocol is at the forefront of innovations relying on ERC-1271, and you can already take advantage of new order types today by swapping on [CoW Swap][15] or by developing custom solutions through the [Programmatic Order Framework][16] (also known as ComposableCoW).

As always, happy trading!

# About CoW DAO

CoW DAO is an open organization of developers, market makers, and community contributors on a mission to create a fairer, more protective financial system. CoW DAO currently supports [CoW Swap][17], [CoW Protocol][18], and [MEV Blocker][19] — DeFi products that leverage order flow auctions to protect users from MEV attacks and give them more than they ask for.

🐦 [Twitter][20]| 📒 [Documentation][21]| 💬 [Forum][22] | 📊 [Analytics][23] | 📸 [Snapshot][24]

[1]: /?source=post_page-----008210ead865--------------------------------
[2]: /?source=post_page-----008210ead865--------------------------------
[3]: https://medium.com/m/signin?actionUrl=https%3A%2F%2Fmedium.com%2F_%2Fsubscribe%2Fuser%2F9cc742e442c8&operation=register&redirect=https%3A%2F%2Fblog.cow.fi%2Feip-1271-explained-008210ead865&user=CoW+Protocol&userId=9cc742e442c8&source=post_page-9cc742e442c8----008210ead865---------------------post_header-----------
[4]: https://medium.com/m/signin?actionUrl=https%3A%2F%2Fmedium.com%2F_%2Fvote%2Fp%2F008210ead865&operation=register&redirect=https%3A%2F%2Fblog.cow.fi%2Feip-1271-explained-008210ead865&user=CoW+Protocol&userId=9cc742e442c8&source=-----008210ead865---------------------clap_footer-----------
[5]: https://medium.com/m/signin?actionUrl=https%3A%2F%2Fmedium.com%2F_%2Fbookmark%2Fp%2F008210ead865&operation=register&redirect=https%3A%2F%2Fblog.cow.fi%2Feip-1271-explained-008210ead865&source=-----008210ead865---------------------bookmark_footer-----------
[6]: https://eips.ethereum.org/EIPS/eip-1271
[7]: https://docs.cow.fi/cow-protocol/concepts/introduction/intents
[8]: https://grants.cow.fi/
[9]: https://yearn.fi/
[10]: https://medium.com/@cow-protocol/what-is-milkman-cow-swaps-solution-for-delayed-execution-trading-c0d647832d38
[11]: /cow-swap-launches-twap-orders-d5583135b472
[12]: /introducing-the-programmatic-order-framework-from-cow-protocol-088a14cb0375
[13]: https://docs.cow.fi/cow-protocol/concepts/introduction/solvers
[14]: https://www.dynamic.xyz/
[15]: https://swap.cow.fi/#/swap?utm_source=medium&utm_campaign=EIP-1271-article
[16]: https://hackmd.io/pmZX_qT2Q1yw59KiiT6TPQ
[17]: http://swap.cow.fi/
[18]: https://cow.fi/
[19]: https://mevblocker.io/
[20]: https://twitter.com/CoWSwap
[21]: https://docs.cow.fi/
[22]: https://forum.cow.fi/
[23]: https://dune.com/cowprotocol
[24]: https://snapshot.org/#/cow.eth