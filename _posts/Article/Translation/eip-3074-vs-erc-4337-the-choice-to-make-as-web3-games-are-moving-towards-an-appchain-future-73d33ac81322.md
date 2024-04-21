---
title: "EIP-3074 vs. ERC-4337: the Choice to Make as Web3 Games Are Moving
  Towards an Appchain Future"
authorURL: ""
originalURL: https://medium.com/@mixmarveldaoventure/eip-3074-vs-erc-4337-the-choice-to-make-as-web3-games-are-moving-towards-an-appchain-future-73d33ac81322
translator: ""
reviewer: ""
---

# **EIP-3074 vs. ERC-4337: the Choice to Make as Web3 Games Are Moving Towards an Appchain Future**

<!-- more -->

[

![MixMarvel DAO Venture](https://miro.medium.com/v2/resize:fill:88:88/1*JGlIUENlL1hmc6QqoveWLg.jpeg)









][1]

[MixMarvel DAO Venture][2]

·

[Follow][3]

12 min read

·

Mar 9, 2023

[

][4]

\--

[][5]

Listen

Share

# Key Takeaways:

EIP-3074 effectively solves the problems of single user structure and insufficient gameplay caused by frequent on-chain transactions and complex transaction process of Web3 games:

-   Securing a convenient use of contracts by eliminating the need to publish smart contracts on each chain;
-   Saving player’s cost by paying Gas fee for them;
-   Ensuring a smooth gaming experience by authorizing players in advance to avoid real-time on-chain confirmation in games;
-   The scope of account authority for contract callers raises concerns about the security of EIP-3074. However, the core logic in this is that users authorize the Invoker through the signature in a restricted way, which can effectively prevent an over-authorized third-party from malpracticing.

ERC-4337 addresses the issues of “reinventing the wheels” and high transaction costs at the developer level:

-   Supporting multi-chain deployments and freeing developers from utilizing separate relays on each chain;
-   Saving players the cost of on-chain transactions with the function of bulk payments through custom code;
-   The proxy mode adopted during the inevitable upgrade process of smart contract wallets will lead to weakness where codes are vulnerable to malpractices. This can be risk-avoided to a certain extent through third-party security audits at the current stage.

![](https://miro.medium.com/v2/resize:fit:640/format:webp/0*4uhT9X0gezIA0eZy)

# Major Challenge to Web3 Games

As predicted by various organizations, Web3 gaming actually led the application field in 2022 as expected. Along with the growth of gaming applications, the problem of poor user experience is gradually exposed as one of the most important issues that need to be addressed nowadays.

Poor user experience is reflected in the following aspects:

-   High entry threshold for mainstream users (Web2 users);
-   Frequent in-chain confirmations, which affect the fun of the game;
-   High transaction Gas fee and complicated transaction process.

These will cause:

-   Traffic barriers;
-   A single user structure with more speculative players yet a very low retention rate of real gamers;
-   High capital cost and time cost of players, which may even lead to the loss of Web3 users.

We have certainly seen that there are already many tools trying to solve the above problems, such as some user aggregators and Web3 integrated SDKs. Yet, these tools are only for solving the specific problem of a particular product, and a universal solution is still needed in the industry.

Logically speaking, to create a generic solution, there are two paths to follow:

1) To start from the bottom, upgrading the underlying protocol, which is related to the real-time on-chain confirmation of users;

2) To begin with the middleware, developing Web3 tools that can access the traditional game engines Unity3D and Unreal Engine.

We will elaborate on these separately, starting with the underlying protocol in this piece.

# EIP-3074 and ERC-4337 with Respective Functional Advantages

To solve the problem of poor user experience from the bottom end, we have to mention EIP-3074 and its major competitor ERC-4337, which is also an Ethereum protocol. The competition between EIP-3074 and ERC-4337 has been going on for a long time. It is mostly believed that ERC-4337 will definitely replace EIP-3074 as the main protocol in the future, even though it still hasn’t landed yet. Reserving our opinion on this for now, we believe that the underlying logic of implementing different protocols varies, and that discussing the future without setting the environment is tantamount to building a castle in the air. We might as well take a look at the functional focus of the two protocols respectively.

## EIP-3074: An underlying protocol that enables sponsored transactions

To explain EIP-3074 technically and in a simple way:

There are two key words in EIP-3074: AUTH/AUTHCALL, which are key opcodes that contribute to the main function of EIP-3074: calling methods.

AUTH/AUTHCALL allows smart contracts to gain control over the EOA accounts that authorized them and to call functions to operate the accounts. These smart contracts are hence called Invokers.

1) AUTH means the Invoker can take control of accounts that have given out authorization. The technical principle is that the user signs a message using a private key and sends it to a relay, which sends the message to the on-chain contract of the calling method, and that the Invoker controls the account with the signed message and the opcode AUTH (mainly used for authentication).

2) AUTHCALL means that the Invoker can perform the account operation on behalf of the user, so the user-signed message can be sent by the Invoker, freeing the user from the dependency of having to pay ETH (Gas fee) to send the message.

This seemingly complex protocol, if applied to Web3 games, would solve the looming problem of poor user experience at hand.

In the logic of Web3 games , each time a player calls the wallet, signs, sends a transaction, and generates Gas fee in the process, it is a direct interaction between the player and the underlying chain. The game official simply provides service to players after receiving these messages. Although EIP-3074 is a protocol for the underlying chain, it allows game officials to be a part of the chain-player interaction, making the entire transaction process simple. The game official calls the player’s wallet through the AUTH/AUTHCALL opcode and interacts with the underlying chain on behalf of the player. The player only needs to sign once to authorize the game official to operate all the interactions, such as Gas fee payment and bulk transaction, with the underlying chain for a certain period of time.

![](https://miro.medium.com/v2/resize:fit:640/format:webp/0*oYe6aFbhkvuo56y_)

In this way, the entry threshold for players is drastically lowered. Whether they are traditional gamers with no Web3 background or Web3 users with insufficient wallet balance, the players no longer need to purchase Gas Token separately. On-chain signatures for bulk transactions are also omitted, thus ensuring the fun of gaming to the greatest extent.

However, opponents of EIP-3074 generally believe that there is a significant security risk in the way EIP-3074 functions, which is to enhance the intelligence of EOA accounts. The excessive authority of Invokers is likely to lead to malpractices.

## ERC-4337: Empowering smart contract wallets on multiple chains

Concerns over EIP-3074 have led to the birth of ERC-4337. Proponents of ERC-4337 argue that, in contrast to EIP-3074, which requires modification of the core underlying protocol, ERC-4337 provides EOA functionality for smart contract wallets without modifying the underlying protocol. Current mainstream smart contract wallets do not have a common development standard and therefore do not support a multi-chain environment. ERC-4337 has established versatility through two modules, UserOperation Mempool and Bundler. Here is Ethereum’s official description of ERC-4337:

_An account abstraction proposal which completely avoids the need for consensus-layer protocol changes. Instead of adding new protocol features and changing the bottom-layer transaction type, this proposal instead introduces a higher-layer pseudo-transaction object called a UserOperation. Users send UserOperation objects into a separate mempool. A special class of actor called bundlers (either block builders, or users that can send transactions to block builders through a bundle marketplace) package up a set of these objects into a transaction making a handleOps call to a special contract, and that transaction then gets included in a block._

For more technicalities, please feel free to read the official post: [https://eips.ethereum.org/EIPS/eip-4337][6]

ERC-4337 mainly solves the problem of “reinventing the wheel” and high cost with existing smart contract wallets. Transactions are packed and then sent in a bulk, eliminating the inefficiency of operating separate relays. Transactions are sent to the Ethernet transaction mempool uniformly and the transaction cost is shared, thus reducing the transaction cost for users.

On the basis of the benefit of smart contract wallets, plus the above-mentioned advantages regarding underlying protocol and separate relays, the ERC-4337-based smart contract wallet is considered to be the future of Ethereum wallet.

# Choosing Between EIP-3074 and ERC-4337: Which holds more promise for Web3 Games as we head to an Appchain future?

The rising Appchain has opened the era of “One chain, One Dapp”. Just as the name implies, Appchain means that each Dapp will have its own space on the blockchain and will not have to compete with other Dapps for web space. We believe that, along with the emergence of large-scale Web3 games, Appchain will become the next mainstream trend in the field of underlying infrastructure.

Let’s take a look at some of the latest developments with Appchains.

-   The licensing branch of the Polygon PoS network has gone live, allowing project Tokens to be used as Gas Token;
-   Cosmos 2.0 is being developed, and dYdX has thus been migrated from Starknet to Cosmos;
-   Avalanche has launched a resilient subnet, allowing subnet validators to use subnet tokens for staking;
-   BSC plans on developing application chains;
-   Rangers Protocol plans to launch subchains.

In the area of Appchain, we can also find early projects that we are familiar with, such as Polkadot, Near Aurora, and Octopus Network. And the advantages of Appchain have been revealed along with the development of various applications.

From the current trend of each project, we think 2023 will be a year when various Appchains show their respective strengths and all kinds of applications choose Appchains for cooperation more often as they are allowed to use project Tokens as Gas. When Appchain is adopted on a large scale, how should EIP-3074 and ERC-4337 play to their respective advantages? Let’s return to the technical level and continue with the discussion.

As mentioned earlier, ERC-4337 solves the biggest problem of smart contract wallets. Thus in essence, ERC-4337-based wallets are also smart contract wallets with the basic advantages of smart contract wallets: the implementation logic of wallets can be customized by code, and the management of wallets is not subject to the control of private keys. The customization logic is very friendly for developers, who can customize the wallet functions according to their business needs. For example, the bulk transactions implemented in the EOA wallet by the EIP-3074 protocol mentioned earlier can also be implemented in ERC-4337 by code. For users, smart contract wallets can also overcome their first barrier to Web3 — understanding and managing private keys and recovery words of the wallets. Smart contract wallets allow users to manage their wallets through social recovery (e.g. email). A case in point is Unipass, whose social recovery solution for wallets is already widely adopted in the industry.

Of course, within the context of Appchain, smart contract wallets also have their inconveniences. To use a smart contract wallet, a separate contract needs to be published on each subnet/subchain; whereas an EOA wallet can be called at any time without publishing a separate contract, regardless of how many subnets/subchains there are. In terms of ease of use for developers, the EOA wallet is far more convenient than the smart contract wallet.

The convenience of EIP-3074 needs to be analyzed in a business logic. The prevailing business logic is that the higher the Token price, the higher the staking revenue. And then more users will participate in staking and the Token price will continue to rise, thus forming a virtuous circle. When there are more and more users on a chain, the network will start to become congested and the Gas fee will rise, which is a clichéd problem. At this point, Appchain and EIP-3074 can come into play.

-   Appchain provides the earlier mentioned solution of “One chain, One Dapp”;
-   The sponsored transactions enabled by EIP-3074 can alleviate the pressure of Gas fee placed on the users due to multiple on-chain confirmations. In the case of gaming applications, sponsored transactions can also enhance the gaming experience. Imagine when players are experiencing an MMORPG immersively and the wallet just appears frequently on the game interface due to on-chain confirmations happening in real time, the fun of the game and the immersive experience of players will be greatly affected. The sponsored transactions can support players to click and authorize the game official or third-party service provider before the game starts. The interaction with the underlying layer, such as calling the wallet to pay the fees, will be done by the authorized party, and players will not be affected by these complicated interactions at all during the game. This solves exactly the problems we mentioned at the beginning, i.e. frequent on-chain confirmations that affect playability, high Gas fees, and complicated transaction processes.

# Security issues that have to be considered

## ERC-4337 : Possible security loopholes in smart contract wallet upgrading

From the perspective of security, ERC-4337 itself is well conceived, but it does not mean it is absolutely secure. Its security vulnerability comes from the nature of smart contracts.

Smart contracts are built from codes and are considered to be the most stable wallets. Whether it is fixing bugs, improving functionality, or optimizing code in response to product demand, smart contracts are, however, bound to be constantly upgraded. Due to the tamper-proof nature of the blockchain, a smart contract cannot be modified once it is released. A new contract needs to be deployed in place of the original contract as a way to complete an upgrade. Therefore, the upgrade of smart contracts will involve Proxy, i.e. proxy mode. A Proxy usually needs to conduct a forwarding and that’s it. But the proxy mode here for smart contract upgrade generally refers to the Reverse Proxy. Reverse proxy means that the Proxy, before performing contract forwarding, executes a piece of code that represents the instructions for upgrading the smart contract. Therefore, all smart contracts to be upgraded using a Proxy will execute such code before being forwarded to the target contract.

The crux of the problem lies in this piece of code.

This code is not monitored. It does not require validation like a transaction signature does. A hacker can easily add in it codes that are irrelevant to contract upgrade, such as sending assets from the contract wallet to other accounts. In this way, all smart contracts that are upgraded using the Proxy contract will suffer asset loss. One might ask whether this security risk can be avoided by an audit. Indeed, a third-party security audit is the only way to monitor the security of this code. However, if a clever hacker hides the malicious code too deeply, the auditor will usually only warn about the malicious code, rather than pointing out the malicious code directly. Though the probability of this is relatively low, once these warnings are ignored, the consequences are unmeasurable. Some people may also wonder whether we can eliminate security problems at source if not using Proxy. In fact, there is currently no way to bypass Proxy when upgrading smart contracts. Even if we use a more advanced protocol like ERC-4337, we cannot avoid this problem, but we are looking forward to a more secure solution in the future.

## EIP-3074: Invoker authorization in question

Moving along with the security issues, some of you might wonder if there will be malpractices coming out, since sponsored transactions enabled by EIP-3074 also grant the Invoker the permission to operate the account.

Indeed, the security of EIP-3074 is not reflected in codes. Opponents of EIP-3074 may argue that the security of account authorization needs to subjectively rely on the integrity of developers, which is actually very dangerous.

Yet, following the logic of authorization, EIP-3074 does not completely expose user accounts to an insecure environment. In fact, EIP-3074 relies on the user’s signature to grant authority. The user’s signature represents his real intention to be sponsored with Gas fee payment, and the signature grants only one permission to pay Gas fees, not unlimited authorization. Therefore, the Invoker can only complete the intent contained in the signature through the AUTH/AUTHCALL commands, and does not obtain unlimited rights over asset disposal.

# Conclusion

The “One Chain, One Dapp” model of Appchain is more suitable for the construction of Web3 games. We predict that Appchain will become the next hot direction in the area of infrastructure. On this basis, we have a new view on the competition between EIP-3074 and ERC-4337. From the user side, the combination of Appchain and EIP-3074 is probably the better choice, for it combines security and convenience, saves players’ cost, and maximizes the fun of games. And from the developer’s point of view, ERC-4337 is advantageous in that it needs no modification with the underlying protocol and supports a multi-chain environment, thus likely to be a more favorable solution.

However, there are concerns over the security of both. EIP-3074 allows the authorized third-party to operate accounts, which is more likely to run into third-party malpractices, whereas the ERC-4337-based smart contract wallets have weaknesses in codes that hackers can exploit. Therefore, in selecting the underlying protocol, we suggest developers to fully consider the features of the game itself and the real needs of users, hit the pain points that need to be solved, and go for the solution that can really solve the problem of complex transactions and transaction costs on the chain.

What we need to solve next is the problem of traffic barriers, i.e. how to lower the entry threshold of traditional players. As we all know, the entrance to Web3.0 is the wallet, which should be our starting point to help players enter into the Web3 world. We will get to the wallet-related solution in our following post.

# About MixMarvel DAO Venture

MixMarvel DAO Venture is a decentralized investment organization that unites builders and investors from the Web2 and Web3 worlds. Concentrating on pioneering Web3 applications, tools, and infrastructures, MixMarvel DAO Venture empowers Web3 ecosystem constructors through financial support and consulting services. It comprises a diverse portfolio of GameFi, Metaverse, Web3 engine, and infrastructure projects.

[Website][7] | [Telegram][8] | [Medium][9] | [Twitter][10] | [Business Contact][11]

[1]: /?source=post_page-----73d33ac81322--------------------------------
[2]: /?source=post_page-----73d33ac81322--------------------------------
[3]: https://medium.com/m/signin?actionUrl=https%3A%2F%2Fmedium.com%2F_%2Fsubscribe%2Fuser%2Fe42268a35131&operation=register&redirect=https%3A%2F%2Fmixmarveldaoventure.medium.com%2Feip-3074-vs-erc-4337-the-choice-to-make-as-web3-games-are-moving-towards-an-appchain-future-73d33ac81322&user=MixMarvel+DAO+Venture&userId=e42268a35131&source=post_page-e42268a35131----73d33ac81322---------------------post_header-----------
[4]: https://medium.com/m/signin?actionUrl=https%3A%2F%2Fmedium.com%2F_%2Fvote%2Fp%2F73d33ac81322&operation=register&redirect=https%3A%2F%2Fmixmarveldaoventure.medium.com%2Feip-3074-vs-erc-4337-the-choice-to-make-as-web3-games-are-moving-towards-an-appchain-future-73d33ac81322&user=MixMarvel+DAO+Venture&userId=e42268a35131&source=-----73d33ac81322---------------------clap_footer-----------
[5]: https://medium.com/m/signin?actionUrl=https%3A%2F%2Fmedium.com%2F_%2Fbookmark%2Fp%2F73d33ac81322&operation=register&redirect=https%3A%2F%2Fmixmarveldaoventure.medium.com%2Feip-3074-vs-erc-4337-the-choice-to-make-as-web3-games-are-moving-towards-an-appchain-future-73d33ac81322&source=-----73d33ac81322---------------------bookmark_footer-----------
[6]: https://eips.ethereum.org/EIPS/eip-4337
[7]: https://daoventure.mixmarvel.com/
[8]: https://t.me/MixMarvel_DAO_Venture
[9]: /
[10]: https://twitter.com/MMDaoVenture
[11]: mailto:daoventure@mixmarvel.com