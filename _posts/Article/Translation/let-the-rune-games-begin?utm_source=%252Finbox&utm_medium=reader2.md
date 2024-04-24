---
title: Decentralised.co
authorURL: ""
originalURL: https://www.decentralised.co/p/let-the-rune-games-begin?utm_source=%252Finbox&utm_medium=reader2
translator: ""
reviewer: ""
---

Share this post

<!-- more -->

![](https://substackcdn.com/image/fetch/w_120,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F19a9b81b-4e5a-487e-b621-0cb2170a51b8_1600x900.png)

#### Let the Rune Games Begin

www.decentralised.co

Copy link

Facebook

Email

Note

Other

# Let the Rune Games Begin

### A brand new Bitcoin-native speculation instrument

[![](https://substackcdn.com/image/fetch/w_80,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F510a9e2c-06ce-4be7-b358-73b0e0d4d0e1_300x300.jpeg)][1]

[Saurabh][2]

Apr 19, 2024

15

Share this post

![](https://substackcdn.com/image/fetch/w_120,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F19a9b81b-4e5a-487e-b621-0cb2170a51b8_1600x900.png)

#### Let the Rune Games Begin

www.decentralised.co

Copy link

Facebook

Email

Note

Other

[

1

][3]

[

Share

][4]

Hey there,  

The halving clock turns once more. It is only apt that we write something related to bitcoin on the occasion. With Runes, Casey gave us ample fodder to talk about.  
  
Tldr; Runes protocol brings memecoin trading on Bitcoin.

_Acknowledgement – Thank you, [Web3\_Lord | Potato][5], for all the inputs and for reviewing the document._

_A small note for the readers: we are publishing the article as the halving is just a few blocks away. Friday’s podcast will be released on Monday. On to the article now…_

[

![](https://substackcdn.com/image/fetch/w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F19a9b81b-4e5a-487e-b621-0cb2170a51b8_1600x900.png)



][6]

In August 2023, we [argued][7] that speculation remains one of crypto's core propositions. This is apparent from how this cycle has played out so far. Memecoins have outperformed many so-called blue chip tokens. But this kind of speculation has also helped blockchains. How? When someone wants to make 10,000X on a freshly launched, un-audited token launched by a trust-me-bro developer, they usually crank up the fee.

If they don’t, they’d not be able to buy the token at $200K FDV _(fully diluted valuation)_. Instead, they will likely have to buy it at $5 million FDV after some time. These fees act as a source of revenue for block producers. It also brings in a lot of [MEV][8] revenue for them. So, if we think that revenue for block producers besides block rewards or subsidies is a proxy for security, a high degree of on-chain speculation is good for that blockchain’s security.

Whether trading NFTs or memecoins, speculation has played a big role in making L1 security more robust. Solana's resurgence is a case in point. In addition to serious DeFi protocols like Jito and company, memecoins like Bonk Inu, Dogwifhat, and Jeo Boden have contributed to growing the Solana ecosystem. Every major blockchain has its own set of meme assets. Memecoin outperformance brings in reflexivity, which means people keep buying or trading them because they’ve done well historically.

The point is, from the activity observed so far, it is clear that people will trade memecoins wherever possible. If it is on a chain with more liquidity, the barrier to entry is lower, and more people participate in memecoin speculation.

Until the Ordinals protocol [launched][9], there was no _(easy_) way to launch and trade memecoins on Bitcoin. After Ordinals, [Domo][10] paved the way for a new standard called BRC-20 that allowed the creation of fungible tokens on Bitcoin. Fungibility means that two tokens are identical. Think of two $1 notes (bills for our American friends). If you and I swap a $1 note, the value we possess doesn’t change.

But BRC-20 was built on top of ordinals and inscriptions, so it takes that complexity and adds its own. We will see why BRC-20 is not the most efficient approach to creating fungible tokens on Bitcoin. BRC-20 tokens quickly gained popularity. Some tokens like [ORDI][11] even crossed a billion-dollar market cap barrier only a few months after launch.

It was clear that the market wanted to trade (memecoins). If not on Bitcoin, they will be traded elsewhere, and that chain will benefit from fees. So why not build a more efficient way of creating and trading fungible assets on Bitcoin itself? This was Casey Rodarmor’s motivation to create a new protocol called Runes. It offers a much easier way– more aligned with Bitcoin’s UTXO structure– to trade fungible tokens.

Moreover, it can help bring more fees to Bitcoin, which is crucial for its long-term security. In this piece, we explore how the upcoming Bitcoin halving creates a long-term security budget problem and how the Runes protocol, along with ordinals and inscriptions, can bring a new wave of activity that pays fees to miners, insuring them against declining security budget subsidies.

## The Halving

Bitcoin halving is one of the most attention-grabbing events in the industry. While everyone is excited about the imminent supply shock, every halving takes us closer to the looming problem of Bitcoin’s security budget. **As block rewards halve, miners’ reliance on fees as revenue increases. Either the price has to double or fees need to increase significantly for miners to keep earning the same amount of dollar revenue.**

Dollar revenue matters because we live in a fiat world, and the cost of mining Bitcoin (such as electricity and equipment) is denominated in fiat currencies. In short, Bitcoin network security, aka, total miner revenue = block subsidy + fee revenue. Fee revenue comes from fees paid to miners for including transactions in blocks.

Among all crypto assets, BTC has cemented its place as a store of value. Miners need fees and fees are generated when there’s activity. So far, BTC has not been the best mode of payment. The following chart shows how the velocity of stablecoins is much higher than BTC. (_Velocity is how frequently an asset changes hands_).

The lower volatility of BTC suggests that people are using BTC as a store of value and perceive stablecoins as modes of payment. This makes intuitive sense.

[

![The chart shows that in the year leading to Jan 2023, BTC was transferred ~30 times. During the same time, USDT and USDC were transferred ~60 and ~100 times, respectively. ](https://substackcdn.com/image/fetch/w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F9cac6f30-2fa7-4bf2-962d-e3a5eeaa177a_1600x900.png "The chart shows that in the year leading to Jan 2023, BTC was transferred ~30 times. During the same time, USDT and USDC were transferred ~60 and ~100 times, respectively. ")



][12]

In the year leading to Jan 2023, a single BTC was transferred ~30 times. During the same time, USDT and USDC were transferred ~60 and ~100 times, respectively.

Low velocity, that is, a relatively low number of transfers, means lower fees for miners. So, ‘_BTC is the best form of money_’ is not enough for miners to keep providing the same level of security to the network. With this backdrop, there needs to be other ways to bring activity on top of Bitcoin to generate fees. In 2023, Casey Rodarmor [launched][13] the ordinals protocol, and inscriptions started trading on Bitcoin. This brought in revenue for Bitcoin miners.  
  
The average monthly revenue for Bitcoin miners from fees is ~$77 million through 2024. Typically, ordinals bring in 20% of the fee revenue, 50%+ on some days with heavy activity. 

[

![](https://substackcdn.com/image/fetch/w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F8ac788dc-e874-4fd3-ad03-15c9cd46943f_1600x900.png)



][14]

[

![](https://substackcdn.com/image/fetch/w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F5b09ee11-2ef2-4f5b-b7ed-f73cc37a7555_1600x900.png)



][15]

Are these fees enough? At this halving, the block reward is set to halve from 6.25 BTC to 3.125 BTC. At 144 blocks a day and $70000 BTC, the fees need to compensate for $32.76 million worth of mining revenue per day, which is $928.8 million a month.  The fees are nowhere close to shouldering the burden of halved mining rewards.

This calls for new ways to generate revenue for miners. 

## Runes Protocol

Casey Rodarmor, the inventor of Ordinals protocol, is set to take his Runes protocol live on Bitcoin at block height 840,000, the block where rewards halve. What does it do? It allows the creation of fungible tokens on top of Bitcoin. While the Ordinals protocol allowed to view each Sat (the smallest unit of bitcoin, 10^9 sats = 1 BTC) differently, the Runes protocol allows creating fungible tokens with different names and quantities in the OP\_RETURN space.

Fungible tokens created via the Runes protocol are called Runes.

Before getting into how Runes are created and transferred, here’s a quick refresher on OP\_RETURN. It is one of the operational codes (opcodes) in Bitcoin’s Script. It allows users to insert arbitrary data up to 80 bytes per transaction. This data has no material impact on bitcoins transferred by users. OP\_RETURN data simply lives on the chain as a part of the transaction.

Not every transaction uses the OP\_RETURN space. Miners don’t need to specially process this data, they treat OP\_RETURN transactions as usual transactions and pass it on while mining or relaying. Third-party service providers like wallets, marketplaces, and explorers can look at this data via different lenses. In a nutshell, this opcode provides a way to piggyback customised data onto standard Bitcoin transactions. 

### How does it work?

Runes is not the first attempt to allow the creation of fungible tokens on Bitcoin. [Counterparty][16], [coloured coin implementations][17], and, more recently, BRC-20 were some of the efforts geared toward creating new tokens on Bitcoin. The difference between these attempts and Runes is that  —

-   It relies only on on-chain data
    
-   It keeps the on-chain footprint minimal
    
-   It does not require a native token; 
    

[

![](https://substackcdn.com/image/fetch/w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F291486c4-9b93-4759-a1c3-48174f207e84_1455x823.png)



][18]

Here’s how Runes protocol works – 

Runes are held by unspent transaction outputs (UTXOs). The protocol has two operations: transfer and issuance. All of this will take place in the OP\_RETURN space. The protocol looks for two types of data pushes in the OP\_RETURN space. The first data push is transfer. This data is in the form of (ID, OUTPUT, AMOUNT) tuple. 

_A tuple is a finite, ordered collection of elements. To simplify further, inputs to Microsoft’s Excel functions can be considered tuples. For example, when you use the VLOOKUP function in Excel, all the values you provide (lookup\_value, table\_array, col\_index\_num, \[range\_lookup\]) are taken in the form of a tuple._

1.  ID refers to the ID assigned to a Runes. This is equivalent to the contract address of an ERC-20 token.
    
2.  OUTPUT can be typically thought of as the destination address.
    
3.  AMOUNT is the number of Runes to be transferred.
    

If the decoded output is not in multiples of three, the message is discarded. If there is a second data push, the Runes Protocol takes it as the issuance transaction. It is decoded as two integers SYMBOL, DECIMALS. 

1.  SYMBOL is the base 26-encoded human-readable symbol, similar to that used in ordinal number sat names. The only valid characters are A through Z.
    
2.  DECIMALS is the number of digits that should be used after the decimal point to display the issued Rune
    

No two Runes can have the same symbol or the names BITCOIN, BTC, or XBT. 

## BRC-20 vs Runes

Out of all the earlier attempts to create the fungible standard on Bitcoin, BRC-20, based on the Ordinals protocol, was the most successful in terms of adoption and trading volume. Two ways in which Runes is superior to BRC-20 are – 

1.  It has more efficient UTXO management
    
2.  It needs a lesser number of on-chain transactions to execute transfers
    

Each BRC-20 token transfer mandates creating a new UTXO, which leads to UTXO proliferation. This is not good, as Bitcoin nodes need to maintain all this data. A bulky UTXO set adds a burden on full nodes. The following chart shows how the number of UTXOs increased post- BRC-20 standard launch. 

[

![](https://substackcdn.com/image/fetch/w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F5a5479db-4800-4d1c-8529-a42491be623d_1600x900.png)



][19]

Why does it create so many UTXOs? BRC-20 tokens are based on the Ordinals protocol. When you mint a BRC-20 token you are inscribing a particular sat (Satoshi). So, the tokens you own are technically inscriptions. Whenever you want to make a transfer, you destroy your inscription and create two new inscriptions.

For example, say Joel owns 50 VCoffee, 20,000 JJ tokens, and 0.3 BTC on his address, and he wants to transfer 20 VCoffee and 12,000 JJ tokens to Sid. We will see how these transfers work under BRC-20 versus a Rune.

Under the BRC-20 standard, addresses cannot hold different BRC-20 tokens in the same UTXO. Each token has to be in a separate UTXO. If these transfers take place in the BTC-20 system, new inscriptions are created—

1.  First, containing 20 VCoffee tokens
    
2.  Second, containing 30 VCoffee tokens
    
3.  Third, containing 12,000 JJ tokens
    
4.  Fourth, containing 8,000 JJ tokens
    

Sid gets 20 VCoffee tokens and 12,000 JJ tokens, and Joel gets his change (30 VCoffee, 8,000 JJ, and 0.3 BTC) back.

With the Runes protocol these transfers become much simpler. Say Joel holds his 50 VCoffee, 10,000 JJ, and 0.1 BTC in one UTXO and 10,000 JJ and 0.2 BTC in another UTXO. He can initiate the same transfer via the Runes protocol in a single transaction. Since the output cannot be satisfied with only one UTXO, both Joel’s UTXOs produce two UTXOs

1.   20 VCoffee and 12,000 JJ, transferred to Sid
    
2.  30 VCoffee, 8,000 JJ, and 0.3 BTC returned to Joel
    

The BRC-20 standard is complicated, as we ended up creating many new UTXOs and intermediate transactions. The Runes protocol works in the same way as Bitcoin, keeping UTXO management tidy.

Think of it this way. Say you have a bunch of marbles in a pouch. If you want to share a few marbles with a friend, you’ll remove some and put them in another pouch. So, every time you want to share marbles with friends, you’ll need an additional pouch, which requires extra transactions. This pouch is like an inscription in a  BRC-20 implementation.

In the case of Runes, you can keep the marbles in the same pouch but change the colour of the marbles so that they can be easily differentiated.

[

![](https://substackcdn.com/image/fetch/w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F988fff5d-f64b-4c7d-905c-e7d19cbf4549_1500x1000.png)



][20]

Now, let’s see how much users have to wait for transfers. Let’s consider a full trading cycle in which a user buys and sells a token. With BRC-20 tokens, the user can buy a listed token with 1 block confirmation. At the time of selling, they have to create a new inscription and transfer it to a DEX, which takes 3 block confirmations. So, the whole cycle takes 4 block confirmations.

At 10-minute average block time, this is 40 minutes. In the case of a Rune, the whole cycle can be completed in 2 blocks or 20 minutes.

As the biggest use case for both these protocols is creating and trading fungible tokens on Bitcoin, the Runes Protocol will likely have an edge over BRC-20.

## The Runes Ecosystem

Runes protocol is currently one of the most sought-after narratives in the Bitcoin ecosystem. It goes live right at the time of halving. Various projects are using Ordinals to distribute their Runes ‘fairly’. Runestone, RSIC, and Rune Pups are among the more popular ordinals that are going to launch Runes. 

Led by [Leonidas][21], Runestones were dropped to ~112K addresses that held 3 or more inscriptions. When the Runes protocol goes live, Runestone inscription holders will get Runstone Runes.

Runecoin (_the project behind RSIC_) strives to be among the first Runes etched (minted or issued). 21000 Rune Specific  Inscription Circuit or RSIC inscriptions were randomly dropped to over 9000 wallets. These were holders of Bitcoin Puppets, Nodemonkes, Bitcoin Frogs, etc. Each RSIC inscription held in a wallet stands to gain 21 Runes for every Bitcoin block. Each [boosted][22] RSIC stands to gain 42 Runes for every block.

Rune Pups is an inscription, and $PUPS is the BRC-20 token, which will switch to Runes with 23% supply going to Rune Pups holders.

Most wallets, such as Leather, XVerse, Unisat, etc., and marketplaces like Magic Eden and OKX will support the Runes protocol. Given how quickly BRC-20 tokens rose to [over $2 billion in total market capitalisation][23], and how much attention there is on Runes, the overall Runes ecosystem will likely be worth many billions of dollars.

## The Launch

The Runes protocol launches at block height 840,000, the same as the halving. The initial plan was to hardcode the first 10 Runes through 0 to 9. But now, only the first Rune, [UNCOMMON•GOODS][24], will be hardcoded. Notice that it has 13 characters (excluding the dot). All the Runes will have 13+ characters, and Runes with fewer characters will be gradually available. The logic is that until the next halving (210,000 blocks or 4 years), 1 character Runes should be available. To get there, Rune names with one less character will be available every 4 months. So, Runes with 12 characters will be available 4 months from the halving.

Provenance has always mattered in crypto. So, as many traders rush to mint Runes right after the first hardcoded Rune, fees on the Bitcoin blockchain are expected to increase. 

## Second-order effects

If things are as they are on Bitcoin, the Runes protocol doesn’t mean much for finance applications on Bitcoin. Why? Runes allows the creation and transfer of fungible tokens with optimal on-chain footprint and no off-chain ties. While this is great, is it enough? A new Bitcoin block takes approximately 10 minutes to mine. This means every time you buy or sell a Rune, it will take ~10 minutes to confirm. This time is 12 seconds on Ethereum and 400 milliseconds on Solana.

It is possible that the Runes protocol integrates with Lightning Network, and this changes as all the activity moves there. But Lightning Network has limitations in terms of BTC volume. And every time someone wants to trade there, they need at least one on-chain transaction to move funds onto the Lightning Network.

So, it is highly likely that a solution that helps parallelise transactions and get around Bitcoin’s block times gets built as there’s an incentive now. This did not make sense in the pre-Runes era. Yes, inscriptions brought activity to Bitcoin, but there’s always a difference between trading NFTs and tokens. With tokens, the frequency is usually higher. So, a 10-minute wait to see how the prices of tokens are changed is too much. Moreover, mempool dynamics may significantly change, and the fees will probably jump higher to the point that normal transfers may become exorbitant on Bitcoin.

While assessing the importance of the Runes protocol, I wanted to know what separates it (_and, by extension, Bitcoin_) from other venues for trading memecoins. The answer – liquidity. **Bitcoin has over $1 trillion worth of native liquidity, compared to $360 billion on Ethereum and $60 billion on Solana** (not including stablecoins).

As of now, the only thing that the Runes protocol unlocks is creating and trading memecoins. **If there was a way to create native programmability on Bitcoin and tokens could represent more than just memes, the Runes protocol would be foundationa**l. We don’t want to get into a debate about whether DeFi protocols need tokens. The point is that tokens exist and some projects that add value to DeFi enabled chains would not exist without tokens. 

Whatever your opinion is on Ethereum, it is a sustainable protocol. What I mean by that is that fees are more than enough for validators to not rely on validator yield (emissions). But Bitcoin has a long way to go before it reaches that point. Whether L2s solve this problem remains to be seen. But Bitcoiners are not big fans of anything that forces them to expand the security assumptions of Bitcoin.

So, we need Bitcoin-native ways of putting BTC (_and other assets_) to work. **Runes protocol facilitates fungibility on Bitcoin without changing anything about the protocol. This is a big plus because making changes to Bitcoin with soft or hard forks is extremely challenging. Achieving social consensus to do so could take months, if not years.**

[Arch Network][25] and [Mezo][26] claim that they will bring native programmability to Bitcoin. Something like this, coupled with Runes, is definitely a huge step in solving Bitcoin’s long-term security budget problem. And I, for one, am excited to see how these building blocks fuse together.

Diamond handing the pre-halving dip,  
[Saurabh Deshpande][27]  
  
_Disclosure: Decentralised.co is an investor in Mezo._

---

A few things that I enjoyed reading or watching on Runes – 

1.  [Good beginner’s guide to Runes][28] by [@0xCygaar][29]
    
2.  [What runes are plus the future][30] by [@0xren\_cf][31]
    
3.  [Runes glossary][32] by [@LeionadsNFT][33]
    
4.  [Casey’s Podcast][34] on Runes
    

15

Share this post

![](https://substackcdn.com/image/fetch/w_120,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F19a9b81b-4e5a-487e-b621-0cb2170a51b8_1600x900.png)

#### Let the Rune Games Begin

www.decentralised.co

Copy link

Facebook

Email

Note

Other

[

1

][35]

[

Share

][36]

PreviousNext

[1]: https://substack.com/profile/134411682-saurabh
[2]: https://substack.com/profile/134411682-saurabh
[3]: https://www.decentralised.co/p/let-the-rune-games-begin/comments
[4]: javascript:void(0)
[5]: https://twitter.com/web3_lord
[6]: https://substackcdn.com/image/fetch/f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F19a9b81b-4e5a-487e-b621-0cb2170a51b8_1600x900.png
[7]: https://www.decentralised.co/p/volatility-as-a-service
[8]: https://www.decentralised.co/p/mev-on-solana
[9]: https://www.decentralised.co/p/from-jpegs-to-ai-models
[10]: https://twitter.com/domodata
[11]: https://www.coingecko.com/en/coins/ordi
[12]: https://substackcdn.com/image/fetch/f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F9cac6f30-2fa7-4bf2-962d-e3a5eeaa177a_1600x900.png
[13]: https://www.decentralised.co/p/from-jpegs-to-ai-models
[14]: https://substackcdn.com/image/fetch/f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F8ac788dc-e874-4fd3-ad03-15c9cd46943f_1600x900.png
[15]: https://substackcdn.com/image/fetch/f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F5b09ee11-2ef2-4f5b-b7ed-f73cc37a7555_1600x900.png
[16]: https://www.counterparty.io/platform
[17]: https://www.youtube.com/watch?v=r2FaaLS9vqE
[18]: https://substackcdn.com/image/fetch/f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F291486c4-9b93-4759-a1c3-48174f207e84_1455x823.png
[19]: https://substackcdn.com/image/fetch/f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F5a5479db-4800-4d1c-8529-a42491be623d_1600x900.png
[20]: https://substackcdn.com/image/fetch/f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F988fff5d-f64b-4c7d-905c-e7d19cbf4549_1500x1000.png
[21]: https://twitter.com/LeonidasNFT
[22]: https://www.ord.io/57066799
[23]: https://www.coingecko.com/en/categories/brc-20
[24]: https://twitter.com/ord_io/status/1780686294907052252
[25]: https://arch-network.gitbook.io/arch-documentation/fundamentals/how-it-works
[26]: https://mezo.org/about
[27]: https://twitter.com/desh_saurabh
[28]: https://twitter.com/0xCygaar/status/1780810840725409938
[29]: https://twitter.com/0xCygaar
[30]: https://twitter.com/0xren_cf/status/1773397548222337468
[31]: https://twitter.com/0xren_cf
[32]: https://twitter.com/LeonidasNFT/status/1775149982523252745
[33]: https://twitter.com/LeonidasNFT
[34]: https://www.youtube.com/watch?v=ysoxbnqiCgQ
[35]: https://www.decentralised.co/p/let-the-rune-games-begin/comments
[36]: javascript:void(0)