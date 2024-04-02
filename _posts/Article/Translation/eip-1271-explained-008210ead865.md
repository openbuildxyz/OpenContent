---
title: EIP-1271 Explained
authorURL: ""
originalURL: https://blog.cow.fi/eip-1271-explained-008210ead865
translator: "sum"
reviewer: ""
---

# EIP-1271 Explained
# EIP-1271 解读

<!-- more -->

## EIP-1271 is an important upgrade to the Ethereum network that’s stirred up quite a buzz, but what exactly is it?
## EIP-1271是以太坊网络的一个重大升级，引发了热议，但它究竟是什么呢？

EIP-1271 (also known as ERC-1271) is an Ethereum improvement that enables smart contracts to validate signatures, allowing them to sign transactions just like traditional EOA wallets.
EIP-1271（也称为ERC-1271）是一个能够让智能合约验证签名的以太坊改进方案，使其能够像传统的EOA钱包一样对交易签名。

While at first EIP-1271 may seem like a niche technical bit of implementation, the standard unlocks a whole world of functionality for smart contracts that includes intent-based trading, advanced order types, and every other type of blockchain interaction that requires a wallet signature.
起初EIP-1271看起来像是一个小的技术改进方案，但这一标准为智能合约解锁了一个全新的世界，包括基于意图的交易、高级订单类型以及其他所有需要钱包签名的区块链交互。

# What is EIP-1271
# 什么是EIP-1271

EIP-1271 serves as a “[standard signature validation method for contracts](https://eips.ethereum.org/EIPS/eip-1271)” which gives smart contracts the ability to verify whether a signature on behalf of a given contract is valid — a feature that Externally Owned Accounts (EOA) already have built-in.
EIP-1271 充当了“[合约的标准签名验证方法](https://eips.ethereum.org/EIPS/eip-1271)”，这使得智能合约能够验证某个合约上的签名是否有效——这是外部拥有账户（EOA）已经内置的功能。

Before EIP-1271, smart contracts could send transactions, but they could not sign messages like traditional wallets. EOA have private keys which they use for signatures, validating that the message came from that particular wallet. Smart contracts, on the other hand, do not have private keys, so EIP-1271 was introduced as a standard way for smart contracts to validate a signature, enabling them to also sign messages.
在EIP-1271之前，智能合约可以发送交易，但无法像传统钱包那样签署消息。EOA拥有私钥，使用私钥签名可以验证消息是否来自该钱包。然而，智能合约没有私钥，因此，EIP-1271作为一种智能合约验证签名的标准被引入，使它们也能签署信息。

## EIP-1271 vs ERC-1271
## EIP-1271和ERC-1271的区别

“EIP-1271” and “ERC-1271” are sometimes used interchangeably as terms that refer to the same concept. Ethereum accepts ideas for changes to the Ethereum network through a proposal process, hence the name “Ethereum Improvement Proposal” (EIP). “ERC,” on the other hand, stands for “Ethereum Request for Comment” and gets assigned to additions to the Ethereum network which do not change the underlying protocol itself.
“EIP-1271”和“ERC-1271”有时可以互换使用，这两个术语指的是同一个概念。以太坊通过提案流程接受对以太坊网络的改进想法，因此有了“以太坊改进提案（EIP）”这个名称。另一方面，“ERC”代表“以太坊意见征求稿”，用来指加入到以太坊网络的内容，这些内容并不改变底层协议本身。

This means that “EIP-1271” is more accurately called “ERC-1271,” as the addition does not change protocol functionality. As mentioned earlier, however, the terms are often used interchangeably.
这意味着“EIP-1271”更准确地应该被称为“ERC-1271”，因为该补充并未改变协议功能。然而如前所述，这两个术语常常被互换使用。

# Why is EIP-1271 so useful?
# 为何EIP-1271如此有用？

Over the last few years, Ethereum has seen a boom in specialized smart contracts leveraging ERC-1271 to sign messages. Signed messages are necessary for a variety of things including placing orders on decentralized exchanges using off-chain order books, verifying that a given wallet belongs to a user, and much more.
在过去的几年中，以太坊见证了利用ERC-1271签署信息的智能合约的蓬勃发展。签署信息对于诸多事务至关重要，包括使用链下订单簿在去中心化交易所下单、验证给定的钱包是否属于某个用户等等。

ERC-1271-enabled smart contracts, however, unlock a whole new set of use-cases. This is because the ERC-1271 “isValidSignature” method can take in any arbitrary order logic, turning smart contract wallets into autonomous agents that can automatically sign messages based on on-chain conditions.
然而，支持ERC-1271的智能合约解锁了一组全新的用例。这是因为ERC-1271的“isValidSignature”方法可以接受任意的指令逻辑，使得智能合约钱包变成了可以基于链上条件自动签署消息的自治代理。

Let’s take a look at the various use-cases that ERC-1271 makes possible for smart contracts.
让我们看看ERC-1271使智能合约成为可能的各种用例。

## Off-Chain Order Books
## 链下订单簿

One of the most common uses of ERC-1271 involves orders on decentralized exchanges (DEX’s). Some exchanges, like CoW Protocol and Matcha, use an off-chain order book to collect signed messages before they become orders, allowing for additional optimizations. In order for smart contracts to be able to trade using these systems, they need ERC-1271 to validate signatures.
ERC-1271的一项最常见的用途是关于去中心化交易所（DEX）上的订单。像CoW协议和Matcha这样的交易所使用链下订单簿来收集签名信息，在这些信息变成订单之前，允许做出额外的优化。为了使智能合约能够使用这些系统进行交易，它们需要ERC-1271来验证签名。

Limit orders in particular are a common type of DEX order that relies on an off-chain order book, so smart contracts require ERC-1271 in order to place limit orders.
限价订单是一种常见的DEX订单类型，它依赖于链下订单簿，因此智能合约需要支持ERC-1271标准才能下限价订单。

## Sign-to-Verify
## 签名验证

Authentication is a necessity in web3.
在Web3中，身份验证是必须的。

Many web3 applications require users to sign a message proving that they control a connected wallet in order to use the application. NFT marketplaces such as OpenSea and Rarible, meanwhile, require users to sign a message to place bids, update profile images, and perform other gas-less transactions.
很多web3应用程序要求用户签名一条消息，用于证明他们控制连接的钱包后才能使用该应用程序。与此同时，OpenSea和Rarible等NFT市场要求用户签署消息来进行出价、更新个人资料图像以及执行其他无Gas费的交易。

In both of these cases, smart contract wallets leverage ERC-1271 to sign verification messages.
在这两种情况下，智能合约钱包都利用ERC-1271来签署验证信息。

## Intents for Smart Contracts
## 智能合约意图(Intents for Smart Contracts)

Recently, many DeFi solutions have begun relying on [intents][7] as a mechanism for providing users with better prices, MEV protection, advanced order types, and a whole host of other benefits.
最近，许多DeFi解决方案开始使用[意图](https://docs.cow.fi/cow-protocol/concepts/introduction/intents)（intents）作为一种机制，为用户提供更好的价格、MEV保护、高级订单类型和其他一系列好处。

CoW Protocol is a meta DEX aggregator that pioneered many use-cases of intents, all of which rely on signed messages. Regular EOA orders are signed ahead of time, but thanks to ERC-1271, smart contract orders can hold off on the order signature until settlement time and check that conditions such as oracle prices, current balances, block times, and more are met before the trade goes through. This unlocks a number of unique trading strategies only available through smart contract wallets.
CoW协议是一个元DEX聚合器，它开创了许多意图（intents）用例，所有用例都依赖于签名消息。常规EOA订单需要提前签名，但得益于ERC-1271，智能合约订单可以在结算时才对订单进行签名，并在交易完成前检查是否满足价格预言机的价格、当前余额、区块时间等条件。这解锁了很多只能通过智能合约钱包实现的特殊交易策略。

**Milkman for Safe Trading**
**让交易更安全的Milkman(Milkman for Safe Trading)**

In collaboration with the [CoW Grants program](https://grants.cow.fi/), the [Yearn.fi](https://yearn.fi/) team launched [Milkman](https://medium.com/@cow-protocol/what-is-milkman-cow-swaps-solution-for-delayed-execution-trading-c0d647832d38) to streamline DAO trades and protect them from MEV. The DAO governance process means that significant time passes between the time a trade is proposed and when the trade is actually executed. During this time, the market can move, invalidating the trade, or worse — an MEV searcher can lie in wait for the exact trade time and manipulate prices to exploit the trade.
[Yearn.fi](https://yearn.fi/)团队与[CoW Grants项目](https://grants.cow.fi/)合作，推出了[Milkman](https://medium.com/@cow-protocol/what-is-milkman-cow-swaps-solution-for-delayed-execution-trading-c0d647832d38)，以简化DAO交易并保护其免受MEV影响。DAO治理流程意味着从提议交易到实际执行交易之间会有相当长的时间间隔。在此期间，市场可能会发生变化，导致交易无效，或者更糟糕的是-MEV搜索者可能会等待确切的交易时间并操纵价格以利用这笔交易获利。

Milkman provides a solution by allowing the DAO to specify a price feed which the smart contract checks at the moment of execution, making a trade based on the up-to-the-second price of the asset.
Milkman提供了一种解决方案，允许DAO指定一个价格源，智能合约在执行时检查这个价格源并根据资产的实时价格进行交易。

This means that DAOs no longer have to commit to an execution price ahead of time, but can leverage smart contracts and EIP-1271 to make a conditional order whose price is decided at the moment of execution.
这意味着DAO不再必须提前承诺执行价格，而是可以利用智能合约和EIP-1271发出条件订单，其价格在执行时决定。

**Advanced Order Types**
**高级订单类型（Advanced Order Types）**

Smart contract signatures on CoW Protocol allow for a number of advanced order types. One notable example is CoW Protocol’s [TWAP orders](https://blog.cow.fi/cow-swap-launches-twap-orders-d5583135b472), which use ERC-1271 smart contract signatures to autonomously place orders in even time intervals.
CoW协议上的智能合约签名允许多种高级订单类型。值得一提的是CoW协议的[TWAP](https://blog.cow.fi/cow-swap-launches-twap-orders-d5583135b472)订单，它使用ERC-1271智能合约签名在固定的时间间隔自主下单。

**The Programmatic Order Framework**
**程序化订单框架（The Programmatic Order Framework）**

The [Programmatic Order Framework](https://blog.cow.fi/introducing-the-programmatic-order-framework-from-cow-protocol-088a14cb0375
) from CoW Protocol takes full advantage of ERC-1271 by allowing anyone to code custom order logic for any type of order.
CoW协议的程序化订单框架充分利用了ERC-1271的优势，允许任何人为任何类型的订单编写自定义逻辑。

Programmatic orders execute indefinitely into the future based on on-chain conditions. These conditions can include any arbitrary logic including checking on-chain prices, smart contract wallet balances, block times, volumes, and more.
程序化订单根据链上条件长期执行。这些条件可以包括检查链上价格、智能合约钱包余额、区块时间、交易量等任意逻辑。

There are already a number of deployed use-cases for ERC-1271-enabled programmatic orders including:
有许多已经部署的支持ERC-1271程序化订单的用例，包括：

-   Automated DAO operations including payroll, treasury diversification, and fee collection
-   自动化DAO操作，包括工资单、资金多元化和手续费归集
-   Portfolio rebalancing based on the state of assets in a smart contract wallet
-   根据智能合约钱包中的资产状态进行投资组合再平衡
-   Special order types including stop-loss, good-after-time, and take-profit
-   特殊订单类型，包括止损、延时和止盈

## Any Application Requiring a Signature
## 任何需要签名的应用程序

We’ve covered a number of the use-cases that require signed messages, but ERC-1271 enables a potentially limitless number of uses… Any time a smart contract wants to use an application that requires a wallet signature, they need ERC-1271.
我们已经介绍了许多需要签名消息的用例，但ERC-1271开启了无限的潜在使用场景……只要智能合约想使用需要钱包签名的应用程序，都需要ERC-1271。

Here are a few final examples requiring signatures:
以下是需要签名的几个示例:

-   **Gasless Trading**: Some decentralized exchanges leveraging intents have begun offering “gasless trading,” a feature which allows users to make trades without paying for gas in ETH. On CoW Protocol, for example, all trades take gas fees in the sell token, leveraging signed messages to specify the order details.
-   **无Gas交易**：一些基于意图（intents）的去中心化交易所已经开始提供“无Gas交易”，该功能允许用户无需支付ETH Gas费用即可进行交易。例如，在CoW协议上，所有交易都采用出售代币支付gas费，利用签名消息来指定订单详细信息。
-   **Native Token Flow**: On CoW Protocol, users can trade ETH natively, without having to use the ERC-20 “WETH” wrapped version. This is once again enabled by signed messages which specify the order details while bonded third parties ([solvers](https://docs.cow.fi/cow-protocol/concepts/introduction/solvers)) handle the execution.
-   **原生代币流程**：在CoW协议上，用户可以交易原生ETH，而无需使用ERC-20 “WETH”包装版本。这再次得益于签名消息，该消息指定订单详细信息，同时委托第三方（[solvers](https://docs.cow.fi/cow-protocol/concepts/introduction/solvers)）负责执行。
-   **Web3 Login**: Services such as [Dynamic.xyz](https://www.dynamic.xyz/) build account abstraction functionality which allows users to use decentralized apps with just a traditional web2 email address. When using smart contract wallets, these services heavily rely on ERC-1271 to provide full signing functionality.
-   **Web3登录**：[Dynamic.xyz](https://www.dynamic.xyz/)等服务构建帐户抽象功能，允许用户仅通过传统的web2电子邮件地址即可使用去中心化应用程序。当使用智能合约钱包时，这些服务严重依赖ERC-1271来提供完整的签名功能。

# How EIP-1271 works
# EIP-1271工作原理

With the introduction of ERC-1271, smart contracts can now define an “isValidSignature” method which determines the conditions necessary for the smart contract to verify a signature and execute a transaction.
随着ERC-1271的引入，智能合约现在可以定义“isValidSignature”方法，该方法确定智能合约验证签名和执行交易所需的条件。

This implementation gives smart contracts a leg up over EOAs when it comes to signing transactions. This is because the “isValidSignature” method can include any arbitrary logic, allowing each smart contract to define its own parameters for generating a valid signature.
在签署交易方面，这种实现使智能合约比EOA更具优势。这是因为“isValidSignature”方法可以包含任意逻辑，允许每个智能合约定义自己的参数来生成有效签名。

Some example criteria for returning a valid “isValidSignature” include:
举例说明“isValidSignature”返回有效的条件包括：

-   Signatures from a set of hard-coded EOA addresses (used in Safe’s multi-signature model)
-   一组硬编码EOA地址的签名（用于Safe的多重签名模型）
-   On-chain conditions like time passing between orders or reaching a given oracle price
-   链上条件，例如订单之间的时间间隔或达到给定的预言机价格
If the owners of the smart contract would like to delegate the order placement to someone else, they can code the necessary conditions for validating a signature in the “isValidSignature.” Whenever anyone calls the smart contract, it will check these conditions and only execute when they are met.
如果智能合约的所有者们希望将订单委托给其他人，他们可以在“isValidSignature”中编写验证签名所需的条件。每次有人调用智能合约时都会检查这些条件，并且仅在满足条件时才执行。

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*ENRRQis22l7l-FFS4Nwy5g.png)

ERC-1271’s “isValidSignature” method
ERC-1271’的“isValidSignature”方法

The method takes a “bytes32 hash” representing an order hash and an arbitrary byte array which is where the conditions for the signature go. This array can be any number of things — an EOA signature from an owner or delegate of the smart contract, the smart contract checking an oracle or an on-chain state to determine if the order should execute, or any other arbitrary logic.
该方法接受两个参数，一个代表订单哈希的“bytes32 hash”和一个包含签名条件的任意字节数组。这个数组可以是很多内容——来自智能合约所有者或委托人的EOA签名、检查预言机价格或链上状态以确定订单是否应该执行的智能合约，或任何其他任意逻辑。

Finally, the “isValidSignature” method returns a response that determines whether the order should execute.
最后，通过“isValidSignature”方法返回值决定订单是否应该执行。

In essence, the entire signature verification process looks like this:
本质上，整个签名验证过程是这样的：

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*ZM3Ar3SbDTyDZ-66oRykWQ.png)

An order that needs to be signed is created
创建需要签署的订单

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*GMEOyrtJm19x0uBXuuX4LQ.png)

The order generates an order hash
该订单生成一个订单哈希

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*Pveow971Ch3TRKHKnTBs_A.png)

The smart contract’s “isValidSignature” method is called with the hash and signature passed in as parameters
调用智能合约的“isValidSignature”方法，并将哈希值和签名作为参数传入

# EIP-1271: Enabling the future of smart contract trading
# EIP-1271：开启智能合约交易的未来

As Ethereum adoption grows, the needs of traders become more complex, eclipsing the built-in functionality of traditional EOAs. ERC-1271 opens the door to a new set of primitives for trading using smart contracts which can create autonomous, conditional orders that provide much greater trading power for users.
随着以太坊的普及和发展，交易者的需求变得更加复杂，超越了传统EOA的内置功能。ERC-1271为使用智能合约进行交易打开了新的大门，这些智能合约可以创建自主、有条件的订单，为用户提供更强大的交易能力。

CoW Protocol is at the forefront of innovations relying on ERC-1271, and you can already take advantage of new order types today by swapping on [CoW Swap](https://swap.cow.fi/#/swap?utm_source=medium&utm_campaign=EIP-1271-article) or by developing custom solutions through the Programmatic Order Framework(also known as ComposableCoW).
CoW协议依托ERC-1271走在了创新前沿，你已经可以通过在CoW Swap上进行交换或通过[程序化订单框架](https://hackmd.io/pmZX_qT2Q1yw59KiiT6TPQ)（也称为ComposableCoW）开发自定义解决方案来创建新型订单。

As always, happy trading!
一如既往，交易愉快！