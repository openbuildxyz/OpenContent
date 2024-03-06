---
title: What happens when you send 1 DAI
authorURL: ""
originalURL: https://www.notonlyowner.com/learn/what-happens-when-you-send-one-dai
translator: ""
reviewer: ""
---

# What happens when you send 1 DAI

<!-- more -->

---

![article cover](https://notonlyowner.com/images/what-happens-dai-cover-intro.png)

You have 1 [DAI][2].

Using a wallet's UI (like [Metamask][3]), you click enough buttons and fill enough text inputs to say that you're sending 1 DAI to `0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045` (that's vitalik.eth).

And hit send.

After some time the wallet says the transaction's been confirmed. All of sudden, Vitalik is now 1 DAI richer. WTF just happened?

Let's rewind. And replay in slow motion.

Ready?

---

## Index

1.  [Building the transaction][4]
    -   [The transaction's data field][5]
    -   [Gas wizardry][6]
    -   [Access list and transaction type][7]
    -   [Signing the transaction][8]
    -   [Serialization][9]
    -   [Submitting the transaction][10]
2.  [Reception][11]
    -   [Inspecting the mempool][12]
3.  [Propagation][13]
4.  [Work preparation and transaction inclusion][14]
5.  [Execution][15]
    -   [Preparation (part 1)][16]
    -   [Preparation (part 2)][17]
    -   [The call][18]
    -   [The interpreter (part 1)][19]
    -   [Solidity execution][20]
    -   [EVM execution][21]
        -   [Free memory pointer and call's value][22]
        -   [Validating calldata (part 1)][23]
        -   [Function dispatcher][24]
        -   [Validating calldata (part 2)][25]
        -   [Reading parameters][26]
        -   [The transfer function][27]
        -   [The transferFrom function][28]
        -   [Logging][29]
        -   [Returning][30]
    -   [The interpreter (part 2)][31]
    -   [Gas refunds and payments][32]
    -   [Building the transaction receipt][33]
6.  [Sealing the block][34]
7.  [Broadcasting the block][35]
8.  [Verifying the block][36]
9.  [Retrieving the transaction][37]
10.  [Afterword][38]

---

## Building the transaction

[Wallets][39] are pieces of software that facilitate sending _transactions_ to the Ethereum network.

A transaction is just a way to tell the Ethereum network that you, as a user, want to execute an action. In this case that'd be sending 1 DAI to Vitalik. And a wallet (e.g., Metamask) helps build such transaction in a relatively beginner-friendly way.

Let's first go over the transaction that a wallet would build. It can be represented as an object with fields and their corresponding values.

Ours will start looking like this:

`{     "to": "0x6b175474e89094c44da98b954eedeac495271d0f",     // [...] }`

Where the field _to_ states the target address. In this case, `0x6b175474e89094c44da98b954eedeac495271d0f` is the address of the DAI smart contract.

Wait, what?

Weren't we supposed to be sending 1 DAI to Vitalik ? Shouldn't `to` be Vitalik's address?

Well, no. To send DAI, one must essentially craft a transaction that executes a piece of code stored in the blockchain (the fancy name for Ethereum's database) that will _update_ the recorded balances of DAI. Both the logic and related storage to execute such update is held in an immutable and public computer program stored in Ethereum's database. The DAI smart contract.

Hence, you want to build a transaction that tells the contract "hey buddy, update your internal balances, taking 1 DAI out of my balance, and adding 1 DAI to Vitalik's balance". In Ethereum jargon, the phrase "hey buddy" translates to setting DAI's address in the `to` field of the transaction.

However, the `to` field is not enough. From the information you provide in your favorite wallet's UI, the wallet fills up several other fields to build a well-formatted transaction.

`{     "to": "0x6b175474e89094c44da98b954eedeac495271d0f",     "amount": 0,     "chainId": 31337,     "nonce": 0,     // [...] }`

It fills the `amount` field with a 0. So you're sending 1 DAI to Vitalik, and you neither use Vitalik's address nor put a `1` in the `amount` field. That's how tough life is (and we're just warming up). The `amount` field is actually included in a transaction to specify how much ETH (the native currency of Ethereum) you're sending along your transaction. Since you don't want to send ETH right now, then the wallet would correctly set that field to 0.

As of the `chainId`, it is a field that specifies the chain where the transaction is to be executed. For Ethereum Mainnet, that is 1. However, since **I will be running this experiment on a local copy of mainnet**, I will use its chain ID: 31337. [Other chains have other identifiers][40].

What about the `nonce` field ? That's a number that should be increased every time you send a transaction to the network. It acts a defense mechanism to avoid replaying issues. Wallets usually set it for you. To do so, they query the network asking what's the latest nonce your account used, and then set the current transaction's nonce accordingly. In the example above it's set to 0, though in reality it will depend on the number of transactions your account has executed.

I just said that the wallet "queries the network". What I mean is that the wallet executes a read-only call to an Ethereum node, and the node answers with the requested data. There are multiple ways to read data from an Ethereum node, depending on the node's location, and what kind of APIs it exposes.

Let's imagine the wallet has direct network access to an Ethereum node. More commonly, wallets interact with third-party providers (like Infura, Alchemy, QuickNode and many others). Requests to interact with the node follow a special protocol to execute remote calls. Such protocol is called [JSON-RPC][41].

A request for a wallet that is attempting to fetch an account's nonce would resemble something like this:

`POST / HTTP/1.1 connection: keep-alive Content-Type: application/json content-length: 124  {     "jsonrpc":"2.0",     "method":"eth_getTransactionCount",     "params":["0x6fC27A75d76d8563840691DDE7a947d7f3F179ba","latest"],     "id":6 } --- HTTP/1.1 200 OK Content-Type: application/json Content-Length: 42  {"jsonrpc":"2.0","id":6,"result":"0x0"}`

Where `0x6fC27A75d76d8563840691DDE7a947d7f3F179ba` would be the sender's account. From the response you can see that its nonce is 0.

Wallets fetch data using network requests (in this case, via HTTP) to hit JSON-RPC endpoints exposed by nodes. Above I included just one, but in practice a wallet can query whatever data they need to build a transaction. Don't be surprised if in real-life cases you notice more network requests to lookup other stuff. For instance, following is a snippet of Metamask traffic hitting a local test node in a couple of minutes:

![Wireshark snapshot of Metamak traffic in local network](https://notonlyowner.com/images/metamask-traffic.png)

### The transaction's data field

DAI is a smart contract. Its main logic is implemented at address `0x6b175474e89094c44da98b954eedeac495271d0f` in Ethereum mainnet.

More specifically, DAI is an ERC20-compliant fungible token - quite a special type of contract. This means that _at least_ DAI should implement the interface detailed in the [ERC20 specification][42]. In (somewhat stretched) web2 jargon, DAI is an immutable open-source web service running on Ethereum. Given it follows the ERC20 spec, it's possible to know in advance (without necessarily looking at the source code) the exact exposed endpoints to interact with it.

Short side note: not all ERC20 tokens are created equal. Implementing a certain interface (which facilitates interactions and integrations) certainly does not guarantee behavior. Still, for this exercise we can safely assume that DAI is quite a standard ERC20 token in terms of behavior.

There are a bunch of functions in the DAI smart contract (source code available [here][43]), many of them directly taken from the ERC20 spec. Of particular interest is [the external `transfer` function][44].

`contract Dai is LibNote {     ...     function transfer(address dst, uint wad) external returns (bool) {         ...     } }`

This function allows anyone holding DAI tokens to transfer some of them to another Ethereum account. Its signature is `transfer(address,uint256)`. Where the first parameter is the address of the receiver account, and the second an unsigned integer representing the amount of tokens to be transferred.

For now let's not focus on the specifics of the function's behavior. Just trust me when I tell you that in its happy path, the function reduces the sender's balance by the passed amount, and then increases the receiver's accordingly.

This is important because when building a transaction to interact with a smart contract, one should know which function of the contract is to be executed. And what parameters are to be passed. It's like if in web2 you wanted to send a POST request to a web API. You'd most likely need to specify the exact URL with its parameters in the request. This is the same. We want to transfer 1 DAI, so we must know how to specify in a transaction that it is supposed to execute the `transfer` function on the DAI smart contract.

Luckily, it's SO straightforward and intuitive.

Joking. It's not. This is what you must include in your transaction to send 1 DAI to Vitalik (remember, address `0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045`):

`{     // [...]     "data": "0xa9059cbb000000000000000000000000d8da6bf26964af9d7eed9e03e53415d37aa960450000000000000000000000000000000000000000000000000de0b6b3a7640000" }`

Let me explain.

Aiming to ease integrations and have a standardize way to interact with smart contracts, the Ethereum ecosystem has (kind of) settled into adopting what's called the "Contract ABI specification" (ABI stands for Application Binary Interface). In common use cases, and I stress, IN COMMON USE CASES, in order to execute a smart contract function you must first encode the call following the [Contract ABI specification][45]. More advanced use cases may not follow this spec, but we're _definitely_ not going down that rabbit hole. Suffice to say that regular smart contracts programmed with [Solidity][46], such as DAI, usually follow the Contract ABI spec.

What you can see above are the resulting bytes of ABI-encoding a call to transfer 1 DAI to address `0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045` with DAI's `transfer(address,uint256)` function.

There are a number of tools out there to ABI-encode stuff, and in some way or another most wallets are implementing ABI-encoding to interact with contracts. For the sake of the example, we can just verify that the above sequence of bytes is correct using a command-line utility called [cast][47], which is able to ABI-encode the call with the specific arguments:

`$ cast calldata "transfer(address,uint256)" 0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045 1000000000000000000  0xa9059cbb000000000000000000000000d8da6bf26964af9d7eed9e03e53415d37aa960450000000000000000000000000000000000000000000000000de0b6b3a7640000`

Anything bugging you ? What's wrong ?

Ooooh, sorry, yeah. That 1000000000000000000. Honestly I would _really_ love to have a more robust argument for you here. The thing is: lots of ERC20 tokens are represented with 18 decimals. Such as DAI. Yet we can only use unsigned integers. So 1 DAI is actually stored as 1 \* 10^18 - which is 1000000000000000000. Deal with it.

We have a beautiful ABI-encoded sequence of bytes to be included in the `data` field of the transaction. Which by now looks like:

`{     "to": "0x6b175474e89094c44da98b954eedeac495271d0f",     "amount": 0,     "chainId": 31337,     "nonce": 0,     "data": "0xa9059cbb000000000000000000000000d8da6bf26964af9d7eed9e03e53415d37aa960450000000000000000000000000000000000000000000000000de0b6b3a7640000" }`

We will revisit the contents of this `data` field once we get to the actual execution of the transaction.

### Gas wizardry

Next step is deciding how much to pay for the transaction. Because remember that all transactions must pay a fee to network of nodes that takes the time and resources to execute and validate them.

The cost of executing a transaction is paid in ETH. And the final amount of ETH will depend on how much net [gas][48] your transaction consumes (that is, how computationally expensive it is), how much you're willing to pay for each gas unit spent, and how much the network is willing to accept at a minimum.

From a user perspective, bottomline _usually_ is that the more one pays, the faster transactions are included. So if you want to pay Vitalik 1 DAI in the next block, you'll probably need to set a higher fee than if you're willing to wait a couple of minutes (or longer, sometimes _way_ longer), until gas is cheaper.

Different wallets may take different approaches to deciding how much to pay for gas. I'm not aware of a single bullet-proof mechanism used by everyone. Strategies to determine the right fees may involve querying gas-related information from nodes (such as the minimum base fee accepted by the network).

For example, in the following requests you can see the Metamask browser extension sending a request to a local test node for gas fee data when building a transaction:

![Metamask traffic querying a node for gas-related data](https://notonlyowner.com/images/metamask-gas-traffic.png)

And the simplified request-response look like:

`POST / HTTP/1.1 Content-Type: application/json Content-Length: 99  {     "id":3951089899794639,     "jsonrpc":"2.0",     "method":"eth_feeHistory",     "params":["0x1","0x1",[10,20,30]] } --- HTTP/1.1 200 OK Content-Type: application/json Content-Length: 190  {     "jsonrpc":"2.0",     "id":3951089899794639,     "result":{         "oldestBlock":"0x1",         "baseFeePerGas":["0x342770c0","0x2da4d8cd"],         "gasUsedRatio":[0.0007],         "reward":[["0x59682f00","0x59682f00","0x59682f00"]]     } }`

The `eth_feeHistory` endpoint is exposed by some nodes to allow querying transaction fee data. If you're curious, read [here][49] or play with it [here][50], or see the spec [here][51].

Popular wallets also use more sophisticated off-chain services to fetch gas price estimations and suggest sensible values to their users. Here's one example of a wallet hitting a public endpoint of a web service, and receiving a bunch of useful gas-related data:

![Wireshark traffic including eth_feeHistory request](https://notonlyowner.com/images/gas-data-requests.png)

Take a look at a snippet of the response:

![Wireshark traffic including eth_feeHistory response](https://notonlyowner.com/images/gas-data-response.png)

Cool, right?

Anyway, hopefully you're getting familiar with the idea that setting the gas fee prices is not straightforward, and it is a fundamental step for building a successful transaction. Even if all you want to do is send 1 DAI. [Here][52] is an interesting introductory guide to dig deeper into some of the mechanisms involved to set more accurate fees in transactions.

After some initial context, let's go back to the actual transaction now. There are three gas-related fields that need to be set:

`{     "maxPriorityFeePerGas": ...,     "maxFeePerGas": ...,     "gasLimit": ..., }`

Wallets will use some of the mentioned mechanisms to fill the first two fields for you. Interestingly, whenever a wallet UI lets you choose between some version of "slow", "regular" or "fast" transactions, it's actually trying to decide on what values are the most appropriate for those exact parameters. Now you can better understand the contents of the JSON-formatted response received by a wallet that I showed you a couple of paragraphs back.

To determine the third field's value, the gas limit, there's a handy mechanism that wallets may use to simulate a transaction before it is really submitted. It allows them to closely estimate how much gas a transaction would consume, and therefore set a reasonable gas limit. On top of providing you with an estimate on the final USD cost of the transaction.

Why not just set an absurdly large gas limit ? To defend your funds, of course. Smart contracts may have arbitrary logic, you being the one paying for its execution. By choosing a sensible gas limit right from the start in your transaction, you protect yourself against ugly scenarios that may drain all your account's ETH funds in gas fees.

Gas estimations can be done using a node's endpoint called `eth_estimateGas`. Before sending 1 DAI, a wallet can leverage this mechanism to simulate your transaction, and determine what's the right gas limit for your DAI transfer. This is what a request-response from a wallet might look like.

`POST / HTTP/1.1 Content-Type: application/json  {     "id":2697097754525,     "jsonrpc":"2.0",     "method":"eth_estimateGas",     "params":[         {             "from":"0x6fC27A75d76d8563840691DDE7a947d7f3F179ba",             "value":"0x0",             "data":"0xa9059cbb000000000000000000000000d8da6bf26964af9d7eed9e03e53415d37aa960450000000000000000000000000000000000000000000000000de0b6b3a7640000",             "to":"0x6b175474e89094c44da98b954eedeac495271d0f"         }     ] } --- HTTP/1.1 200 OK Content-Type: application/json  {"jsonrpc":"2.0","id":2697097754525,"result":"0x8792"}`

In the response you can see that the transfer would take approximately 34706 gas units.

Let's incorporate this information to the transaction payload:

`{     "to": "0x6b175474e89094c44da98b954eedeac495271d0f",     "amount": 0,     "chainId": 31337,     "nonce": 0,     "data": "0xa9059cbb000000000000000000000000d8da6bf26964af9d7eed9e03e53415d37aa960450000000000000000000000000000000000000000000000000de0b6b3a7640000",     "maxPriorityFeePerGas": 2000000000,     "maxFeePerGas": 120000000000,     "gasLimit": 40000 }`

Remember that the `maxPriorityFeePerGas` and `maxFeePerGas` will ultimately depend on the network conditions at the moment of sending the transaction. Above I'm just setting somewhat arbitrary values for the sake of this example. As of the value set for the gas limit, I just incremented the estimate a bit to fall on the safe side.

### Access list and transaction type

Let's briefly comment on two additional fields that are set in your transaction.

First, the `accessList` field. Advanced use cases or edge scenarios may require the transaction to specify in advance the account addresses and contracts' storage slots to be accessed, thus making it somewhat cheaper.

However, it may not be straightforward to build such list in advance, and currently the gas savings may not be not so significant. Particularly for simple transactions like sending 1 DAI. Therefore, we can just set it to an empty list. Although remember that it does exist [for a reason][53], and it may become more relevant in the future.

Second, [the transaction type][54]. It is specified in the `type` field. The type is an indicator of what's inside the transaction. Our will be a type 2 transaction - because its following the format specified [here][55].

`{     "to": "0x6b175474e89094c44da98b954eedeac495271d0f",     "amount": 0,     "chainId": 31337,     "nonce": 0,     "data": "0xa9059cbb000000000000000000000000d8da6bf26964af9d7eed9e03e53415d37aa960450000000000000000000000000000000000000000000000000de0b6b3a7640000",     "maxPriorityFeePerGas": 2000000000,     "maxFeePerGas": 120000000000,     "gasLimit": 40000,     "accessList": [],     "type": 2 }`

### Signing the transaction

How can nodes know that it is _your_ account, and not somebody else's, who is sending a transaction ?

We've come to the critical step of building a valid transaction: signing it.

Once a wallet has collected enough information to build the transaction, and you hit SEND, it will digitally sign your transaction. How ? Using your account's private key (that your wallet has access to), and a cryptographic algorithm involving curvy shapes called [ECDSA][56].

For the curious, what's actually being signed is the `keccak256` hash of the concatenation between the transaction's type and the [RLP encoded][57] content of the transaction.

`keccak256(0x02 || rlp([chainId, nonce, maxPriorityFeePerGas, maxFeePerGas, gasLimit, to, amount, data, accessList]))`

You shouldn't be so knowledgeable in cryptography to understand this though. Put simply, this process seals the transaction. It makes it tamper-proof by putting a smart-ass stamp on it that only your private key could have produced. And from now on anyone with access to that signed transaction (for example, Ethereum nodes) can cryptographically verify that it was your account that produced it.

Just in case: signing is _not_ encrypting. Your transactions are always in plaintext. Once they go public, anyone can make sense out of their contents.

The process of signing the transaction produces, no surprise, a signature. In practice: a bunch of weird unreadable values. These travel along the transaction, and you'll usually find them referred to as `v`, `r` and `s`. If you want to dig deeper on what these actually represent, and their importance to recover your account's address, the Internet is your friend.

You can get a better idea on what signing looks like when implemented by checking out the [@ethereumjs/tx][58] package. Also using the [ethers][59] package for some utilities. As an extremely simplified example, signing the transaction to send 1 DAI could look like this:

`const { FeeMarketEIP1559Transaction } = require("@ethereumjs/tx");  const txData = {     to: "0x6b175474e89094c44da98b954eedeac495271d0f",     amount: 0,     chainId: 31337,     nonce: 0,     data: "0xa9059cbb000000000000000000000000d8da6bf26964af9d7eed9e03e53415d37aa960450000000000000000000000000000000000000000000000000de0b6b3a7640000",     maxPriorityFeePerGas: ethers.utils.parseUnits('2', 'gwei').toNumber(),     maxFeePerGas: ethers.utils.parseUnits('120', 'gwei').toNumber(),     gasLimit: 40000,     accessList: [],     type: 2, };  const tx = FeeMarketEIP1559Transaction.fromTxData(txData); const signedTx = tx.sign(Buffer.from(process.env.PRIVATE_KEY, 'hex'));  console.log(signedTx.v.toString('hex')); // 1  console.log(signedTx.r.toString('hex')); // 57d733933b12238a2aeb0069b67c6bc58ca8eb6827547274b3bcf4efdad620a  console.log(signedTx.s.toString('hex')); // e49937ec81db89ce70ebec5e51b839c0949234d8aad8f8b55a877bd78cc293`

The resulting object would look like:

`{     "to": "0x6b175474e89094c44da98b954eedeac495271d0f",     "amount": 0,     "chainId": 31337,     "nonce": 0,     "data": "0xa9059cbb000000000000000000000000d8da6bf26964af9d7eed9e03e53415d37aa960450000000000000000000000000000000000000000000000000de0b6b3a7640000",     "maxPriorityFeePerGas": 2000000000,     "maxFeePerGas": 120000000000,     "gasLimit": 40000,     "accessList": [],     "type": 2,     "v": 1,     "r": "57d733933b12238a2aeb0069b67c6bc58ca8eb6827547274b3bcf4efdad620a",     "s": "e49937ec81db89ce70ebec5e51b839c0949234d8aad8f8b55a877bd78cc293", }`

### Serialization

The next step is _serializing_ the signed transaction. That means encoding the pretty object above into a raw sequence of bytes, such that it can be sent to the Ethereum network and consumed by the receiving node.

The encoding method chosen by Ethereum is called [RLP][60]. The way the transaction is encoded is as follows:

`0x02 || rlp([chainId, nonce, maxPriorityFeePerGas, maxFeePerGas, gasLimit, to, value, data, accessList, v, r, s])`

Where the initial byte is the transaction type.

Building upon the previous code snippet, you can actually see the serialized transaction adding this:

`console.log(signedTx.serialize().toString('hex')); // 02f8b1827a69808477359400851bf08eb000829c40946b175474e89094c44da98b954eedeac495271d0f80b844a9059cbb000000000000000000000000d8da6bf26964af9d7eed9e03e53415d37aa960450000000000000000000000000000000000000000000000000de0b6b3a7640000c001a0057d733933b12238a2aeb0069b67c6bc58ca8eb6827547274b3bcf4efdad620a9fe49937ec81db89ce70ebec5e51b839c0949234d8aad8f8b55a877bd78cc293`

That is the actual payload to send 1 DAI to Vitalik on my local copy of the Ethereum mainnet.

### Submitting the transaction

Once built, signed and serialized, the transaction must be sent to an Ethereum node.

There's a handy JSON-RPC endpoint that nodes may expose where they can receive such requests. It's called `eth_sendRawTransaction`. Here's the network traffic of a wallet employing it upon submitting the transaction:

![Wireshark traffic sending a raw transaction using the eth_sendRawTransaction method](https://notonlyowner.com/images/send-raw-transaction.png)

The summarized request-response looks like:

`POST / HTTP/1.1 Content-Type: application/json Content-Length: 446  {     "id":4264244517200,     "jsonrpc":"2.0",     "method":"eth_sendRawTransaction",     "params":["0x02f8b1827a69808477359400851bf08eb000829c40946b175474e89094c44da98b954eedeac495271d0f80b844a9059cbb000000000000000000000000d8da6bf26964af9d7eed9e03e53415d37aa960450000000000000000000000000000000000000000000000000de0b6b3a7640000c001a0057d733933b12238a2aeb0069b67c6bc58ca8eb6827547274b3bcf4efdad620a9fe49937ec81db89ce70ebec5e51b839c0949234d8aad8f8b55a877bd78cc293"] } --- HTTP/1.1 200 OK Content-Type: application/json Content-Length: 114  {     "jsonrpc":"2.0",     "id":4264244517200,     "result":"0xbf77c4a9590389b0189494aeb2b2d68dc5926a5e20430fb5bc3c610b59db3fb5" }`

The result included in the response contains the transaction's hash: `bf77c4a9590389b0189494aeb2b2d68dc5926a5e20430fb5bc3c610b59db3fb5`. This 32-bytes-long sequence of hex characters is the unique identifier for the submitted transaction.

## Reception

How should we go about figuring out what happens when an Ethereum node receives the serialized signed transaction ?

Some might ask on Twitter, others may read some Medium articles. Other may even read documentation. [Shame!][61]

There's only one place to find the truth: at the source. Let's use [go-ethereum v1.10.18][62] (a.k.a. Geth), a popular implementation of an Ethereum node (an "execution client" once Ethereum moves to Proof-of-Stake). From now on, I'll be including links to Geth's source code for you to follow along.

Upon receiving the JSON-RPC call on its `eth_sendRawTransaction` [endpoint][63], the node needs to make sense out of the serialized transaction included in the request's body. So [it begins with deserializing the transaction][64]. From now on the node will have easier access to the transaction's fields.

At this point the node already starts validating the transaction. [First][65], ensuring that the transaction's fee (i.e., price \* gas limit) does not go above the maximum that the node is willing to accept (apparently, [by default this is 1 ether][66]). And [then][67], ensuring that the transaction is replay-protected (following [EIP 155][68] - remember the `chainID` field we set in the transaction?), or that the node is willing to accept unprotected transactions.

The next steps consists of [sending][69] [the][70] [transaction][71] [to][72] the [transaction pool][73] (a.k.a. the mempool). Put simply, this pool represents the set of transactions that the node is aware of at a specific moment. As far the node knows, these haven't been included in the blockchain yet.

Before _really_ including the transaction in the pool, the node [checks][74] that it doesn't already know about it. And that its ECDSA signature is [valid][75]. Discarding the transaction otherwise.

Then [the heavy mempool stuff][76] begins. As you may notice, there's lots of non-trivial logic to ensure that the transaction pool is all happy and healthy.

There's quite a number of important [validations][77] performed here. Such as the gas limit being [below the block gas limit][78], or the transaction's size [not exceeding][79] the [maximum allowed][80], or the nonce being [the expected one][81], or the sender having [enough funds][82] to cover potential costs (i.e., value + gas limit \* price), and more.

While we could go on, we're not here to become mempool experts. Even if we wanted to, we'd need to consider that, as long as they follow the network consensus rules, each node operator may take different approaches to mempool management. That means performing special validations or following custom transaction prioritization rules. In the interest of just sending 1 DAI, we can treat the mempool as a mere set of transactions eagerly waiting to be picked up and be included in a block.

After successfully adding the transaction to the pool (and doing internal [logging][83] stuff), the node [returns the transaction hash][84]. Exactly what we saw being returned in the JSON-RPC request-response earlier 😎

### Inspecting the mempool

If you send the transaction via Metamask or any similar wallet that is by default connected to traditional nodes, at some point it will land on public nodes' mempools. You can make sure of this by inspecting mempools by yourself.

There's a handy endpoint some nodes expose, called `eth_newPendingTransactionFilter`. Perhaps a good-old friend of [frontrunning][85] [bots][86]. Periodically querying this endpoint should allow us to observe the transaction to send 1 DAI walking into the mempool of a local test node before being included in the chain.

In Javascript code, this can be accomplished with:

`const hre = require("hardhat");  hre.ethers.provider.on('pending', async function (tx) {     // do something with the transaction });`

To see the actual `eth_newPendingTransactionFilter` call, we can just inspect the network traffic:

![Wireshark traffic with JSON-RPC call subscribing to pending transactions](https://notonlyowner.com/images/new-pending-transaction-filter.png)

From now on, the script will (automatically) poll changes in the mempool. Here's the first of many subsequent periodic calls asking for changes:

![Wireshark traffic with JSON-RPC call asking for changes in the mempool](https://notonlyowner.com/images/first-eth-get-filter-changes.png)

And after receiving the transaction, here's the node finally answering with its hash:

![Wireshark traffic with JSON-RPC call answering with detected transaction hash](https://notonlyowner.com/images/eth-get-filter-answer.png)

The summarized request-response looks like:

`POST / HTTP/1.1 Content-Type: application/json content-length: 74  {     "jsonrpc":"2.0",     "method":"eth_getFilterChanges",     "params":["0x1"],     "id":58 } --- HTTP/1.1 200 OK Content-Type: application/json Content-Length: 105  {     "jsonrpc":"2.0",     "id":58,     "result":["0xbf77c4a9590389b0189494aeb2b2d68dc5926a5e20430fb5bc3c610b59db3fb5"] }`

Earlier I said "traditional nodes" without explaining much. What I mean is that there are [more specialized nodes][87] that feature private mempools. They allow users to "hide" transactions from the public before they are included in a block.

Regardless of the specifics, such mechanisms usually consist of establishing private channels between transaction senders and block builders. The [Flashbots Protect service][88] is one notable example. The practical consequence is that even if you're monitoring mempools with the method shown above, you won't be able to fetch transactions that make it to block builders via private channels.

Assume that the transaction to send 1 DAI is submitted to the network via common channels without leveraging this kind of services.

## Propagation

For the transaction to be included in a block, it somehow needs to reach the nodes able to build and propose it. In [Proof-of-Work][89] Ethereum, these are called miners. In [Proof-of-Stake][90] Ethereum, validators. Though reality tends to be a bit more complicated. Be aware that there may be ways in which [block building can be outsourced][91] to specialized services.

As a common user, you shouldn't need to know who these block producers are, nor where they are located. Instead, you may simply send a valid transaction to any regular node in the network, have it included in the transaction pool, and let the peer-to-peer protocols do their thing.

There are [a number of these p2p protocols][92] interconnecting Ethereum nodes. They allow, among other things, the frequent [exchange of transactions][93].

Right from the start all nodes are [listening and broadcasting][94] transactions along with their peers (by default, [maximum 50 peers][95]).

Once a transaction reaches the mempool, it is [sent to all connected peers][96] that do not already know about the transaction.

To favor efficiency, only a random subset of connected nodes ([the square root][97] 🤓) are [sent the full transaction][98]. The rest is [only sent the transaction hash][99]. These could request back the full transaction if needed.

A transaction cannot linger in a node's mempool forever. If it's not first dropped for other reasons (e.g., the pool is full and the transaction is underpriced, or it gets replaced with a new one with higher nonce/price), it may be automatically [removed][100] after a certain period of time (by default, [3 hours][101]).

Valid transactions in the mempool that are considered ready to be picked up and processed by a block builder are [tracked][102] in a list of [pending transactions][103]. This data structure [can be queried][104] by block builders to obtain processable transactions that are allowed to make it into the chain.

## Work preparation and transaction inclusion

The transaction should reach a mining node (at least at the time of writing) after navigating mempools. Nodes of this type are particularly heavy multitaskers. For those familiar with Golang, this translates to quite a number of go routines and channels in the mining-related logic. For those unfamiliar with Golang, this means that miners regular operations cannot be explained as linearly as I'd like.

This section's goal is twofold. First, understanding how and when our transaction is picked up from the mempool by a miner. Second, finding out at which point the transaction's execution starts.

At least two relevant things happen when the node's mining component is initialized. One, it [starts listening][105] for the arrival of new transactions to the mempool. Two, [some fundamental loops][106] are triggered.

In Geth's jargon, the act of building a block with transactions and sealing it is called "committing work". Thus we want to understand under which circumstances this happens.

Focus on [the "new work" loop][107]. That's a standalone routine that, upon the node receiving [different kind of notifications][108], will trigger the commit of work. The trigger essentially entails [sending a work requirement][109] to another of the node's active listeners (running in the miners's ["main" loop][110]). When such work requirement is [received][111], the [commit of work begins][112].

The node starts with some [initial preparation][113]. Mainly consisting of building the block's header. This includes tasks like [finding the parent block][114], ensuring [the timestamp of the block being built][115] is correct, [setting the block number][116], [the gas limit][117], the [coinbase address][118] and [the base fee][119].

Afterwards, the consensus engine is invoked for the header's ["consensus preparation"][120]. This [calculates the right block difficulty][121] ([depending][122] on the current version of the network). If you've ever heard of Ethereum's "difficulty bomb", there you have it.

The block sealing context [is created][123] next. Other actions aside, this consists of [fetching the last known state][124]. This is the state against which the first transaction in the block being built is going to be executed. That might be our transaction sending 1 DAI.

Having prepared the block, it is now [filled with transactions][125].

We've finally reached the point in which our pending transaction, so far just comfortably sitting in the node's mempool, [is picked up][126] along others.

By default [transactions are ordered][127] within a block by price and nonce. For our case, the transaction's position within the block is practically irrelevant.

Now the sequential [execution of these transactions][128] begins. One transaction [is executed][129] after the other, each building upon the resulting state of the previous one.

## Execution

An Ethereum transaction can be thought of as a state transition.

State 0: you have 100 DAI, and Vitalik has 100 as well.

Transaction: you send 1 DAI to Vitalik.

State 1: you have 99 DAI, and Vitalik has 101.

Hence, executing a transaction entails _applying_ a sequence of operations to the current state of the blockchain. Producing a new (different) state as a result. This will be considered the new current state until a another transaction comes in.

In reality this is _far_ more interesting (and complex). Let's see.

### Preparation (part 1)

In Geth's jargon, miners [_commit_ transactions][130] in the block. The act of committing a transaction is done in an [_environment_][131]. Such environment contains, among other things, a given [_state_][132].

So in short, committing a transaction is essentially: [(1)][133] remembering the current state, [(2)][134] modifying it by applying the transaction, [(3)][135] depending on the transaction's success, either accepting the new state or rolling back to the original one.

The juicy stuff happens in (2): [applying the transaction][136].

First thing to notice is that [the transaction is turned into a "message"][137]. If you're familiar with Solidity, where you are usually writing things like `msg.data` or `msg.sender`, finally reading "message" in Geth's code is THE sign on the road welcoming you into friendly lands.

Inspecting [what a message looks like][138] quickly leads to notice at least one difference with the transaction. A message has [a `from` field][139]! This field is the signer's Ethereum address, which is [derived][140] from [the public signature][141] included in the transaction (remember the weird `v`, `r` and `s` fields?).

Now the environment for execution [is further prepared][142]. First, the [block-related context is created][143], which includes stuff like block number, timestamp, the coinbase address and the block gas limit. And then...

[the beast walks in][144].

The Ethereum Virtual Machine (EVM), the [stack-based][145] [256-bit][146] computing engine in charge of executing the transaction, shows up all chill like everything is cool man, cool, and [starts putting some clothes on][147]. Yeap, it was naked. It's the EVM, what were you expecting?

The EVM is a machine. And as a machine, it has a set of instructions (a.k.a. opcodes) it can execute. The instruction set has changed over the years. So _there has to be_ some piece of code telling the EVM which opcodes it should use today. And behold, there is. When the EVM [instances its interpreter][148], it [chooses the correct set of opcodes][149], depending on the version being used.

Lastly, [two final steps][150] prior to [real execution][151]. The EVM's transaction context is created (ever used `tx.origin` or `tx.gasPrice` in your Solidity smart contracts?), and the EVM is given access to the current state.

### Preparation (part 2)

It's turn for the EVM to perform the [state transition][152]. Given a message, an environment and the original state, it will use a limited set of instructions to move to a new state. One in which Vitalik has 1 additional DAI 💰.

Before applying the state transition, the EVM must make sure that it abides to [specific consensus rules][153]. Let's see how that's done in detail.

Validation begins in what Geth calls [the "pre-check"][154]. It consists of:

1.  Validating the message's nonce. It [must match][155] the nonce of the message's `from` address. Also, it [must not be the maximum possible nonce][156] (by checking whether incrementing the nonce by one causes an overflow).
2.  Making sure that the account corresponding to the message's `from` address [does not have code][157]. That is, that the transaction origin is an externally-owned account (EOA). Thus abiding by the [EIP 3607][158] spec.
3.  Validating that the `maxFeePerGas` (the `gasFeeCap` in Geth) and `maxPriorityFeePerGas` (the `gasTipCap` in Geth) fields set in the transaction are [within expected bounds][159]. Moreover, that the priority fee [is not greater][160] than the max fee. And that the `maxFeePerGas` [is greater][161] than the current block's base fee.
4.  [Buying gas][162]. In turn checking that the account [can pay][163] for all the gas it intends to consume. And that there's [enough gas left][164] in the block to process the transaction. Finally making the account [pay in advance][165] for the gas (don't worry, there're refund mechanisms later).

Next, the EVM [accounts for the "intrinsic gas"][166] that the transaction consumes. There are a few factors to consider when calculating intrinsic gas. For starters, [whether the transaction is a contract creation][167]. Ours is not, so the gas starts at [21000 units][168]. Afterwards, the [amount of non-zero bytes][169] in the message's `data` field is also taken into consideration. [16 units][170] are charged per non-zero byte (following [this specification][171]). Only [4 units][172] are charged for each zero byte. Finally, some more gas would be accounted in advance [if we provided access lists][173].

We set the `value` field of the transaction to zero. Had we specified a positive value, now would be the moment for the EVM to check [whether the sender account actually has enough balance][174] to execute the ETH transfer. Furthermore, had we set access lists, [now they would be initialized in state][175].

The transaction being executed is not creating a contract. The EVM knows it [because the `to` field is not zero][176]. Therefore, it will already [increment][177] the sender's account nonce by one, and [execute a call][178].

The call will go from the `from` to the `to` message's addresses, passing along the `data`, no value, and whatever remaining gas is left after consuming the intrinsic gas.

### The call

(not [this call][179])

The DAI smart contract is stored at address `0x6b175474e89094c44da98b954eedeac495271d0f`. That's the address we set in the `to` field of the transaction. This initial call is meant for the EVM to execute whatever code is stored at it. Opcode by opcode.

Opcodes are EVM instructions represented with hex numbers ranging from 00 to FF. Though they're usually referred to with their names. For example, `00` is `STOP` and `FF` is `SELFDESTRUCT`. A handy list of opcodes is available at [evm.codes][180].

So what are DAI's opcodes anyway ? Glad you asked:

![EVM opcodes of the DAI smart contract](https://notonlyowner.com/images/DAI-opcodes.png)

Don't panic. It's still early to make sense out of all of that.

Let's start slowly, tearing [the initial call][181] apart. Its [brief docs][182] provide a good summary:

`// Call executes the contract associated with the addr with the given input as // parameters. It also handles any necessary value transfer required and takes // the necessary steps to create accounts and reverses the state in case of an // execution error or failed value transfer. func (evm *EVM) Call(caller ContractRef, addr common.Address, input []byte, gas uint64, value *big.Int) (ret []byte, leftOverGas uint64, err error) {     ... }`

To begin with, the logic [checks][183] that the call depth limit hasn't been reached. This limit is [set to 1024][184], which means there can only be a maximum of 1024 nested calls in a single transaction. [Here][185] is an interesting article to read about some of the reasoning and subtleties behind this behavior of the EVM. Later we'll explore how the call depth is increased/decreased.

Relevant side note: the call depth limit is _not_ the EVM's stack size limit - which (coincidentally?) [is 1024 elements][186] as well.

The next step is to [make sure][187] that if a positive value was specified in the call, the sender has enough balance to execute the transfer ([performed][188] a few steps later). We can ignore this because our call has zero value. Additionally, a [snapshot of the current state][189] is taken. This allows [easily reverting][190] any state changes upon failure.

We know that DAI's address refers to an account that has code. Thus, it [must already exist][191] in Ethereum's state.

However, let's imagine for a moment this was not a transaction to send 1 DAI. Say it was a trivial transaction with no value targetting a new address. The corresponding account would need to be [_added_ to the state][192]. However, what if said account would end up being just empty ? There doesn't seem to be a reason to keep track of it - other than wasting nodes' disk space. [EIP 158][193] introduced some changes to the Ethereum protocol to help avoid such scenarios. That's why you're seeing [this `if` clause][194] when calling any account.

Another thing we know is that DAI [is _not_ a precompile contract][195]. What's a precompiled contract ? Here's what the [Ethereum yellow paper][196] has to offer:

> \[...\] preliminary piece of architecture that may later become native extensions. The contracts in addresses 1 to 9 execute the elliptic curve public key recovery function, the SHA2 256-bit hash scheme, the RIPEMD 160-bit hash scheme, the identity function, arbitrary precision modular exponentiation, elliptic curve addition, elliptic curve scalar multiplication, an elliptic curve pairing check, and the BLAKE2 compression function F respectively.

In short, there're (so far) 9 different special contracts in Ethereum's state. These accounts (ranging from [0x0000000000000000000000000000000000000001][197] to [0x0000000000000000000000000000000000000009][198]) out-of-the-box include the necessary code to execute the operations mentioned in the yellow paper. Of course, you can [check this by yourself in Geth's code][199].

To add some color to the story of precompiled contracts, note that in Ethereum mainnet all these accounts have _at least_ 1 wei in balance. This was done [intentionally][200] (at least before users started sending Ether by mistake). Look, [here's an almost 5-year-old transaction][201] sending 1 wei to the `0x0000000000000000000000000000000000000009` precompile.

Anyway. Having realized the call's target address does not correspond to a precompiled contract, the node [reads the account's code][202] from the state. Then [ensures it's non-empty][203]. At last, [orders the EVM][204] to use its interpreter to run the code with the given input (the contents of the transaction's `data` field).

### The interpreter (part 1)

It's time for the EVM to actually execute DAI's code. To accomplish this, the EVM has a couple of elements at hand. It has [a stack][205] that can hold [up to 1024 elements][206] (though only the first 16 are directly accessible with the available opcodes). It has a volatile [read/write memory space][207]. It has a [program counter][208]. It has a special read-only memory space called calldata [where the call's input data is kept][209]. Among other stuff.

As usual, there's some necessary setup and validations before jumping into the juicy stuff. First, the [call depth is incremented][210] by one. Second, the [read-only mode is set][211] if necessary. Ours is not a read-only call (see the `false` argument passed [here][212]). Otherwise some EVM operations wouldn't be allowed. These include state-changing EVM instructions [`SSTORE`][213], [`CREATE`][214], [`CREATE2`][215], [`SELFDESTRUCT`][216], [`CALL`][217] with positive value, and [`LOG`][218].

The interpreter now enters [the execution loop][219]. It consists of sequentially [executing][220] the opcodes in DAI's code as [indicated by the program counter][221] and the current EVM instruction set. For the time being we're using the [London instruction set][222] - which was [configured in the jump table][223] when the interpreter was first instantiated.

The loop also takes care of [keeping a healthy stack][224] (avoiding under/overflows). And spending each operation's [fixed gas costs][225], as well as [dynamic gas costs][226] when appropriate. These dynamic costs [include][227], for example, the expansion of EVM memory (read more about memory expansion costs are calculated [here][228]). Note that gas is consumed _before_ execution of an opcode - not after.

The actual behavior of each possible instruction can be found implemented in [this Geth file][229]. By just skimming through it one can begin to see how these instructions work with the stack, the memory, the calldata and the state.

At this point we'd need to jump straight into DAI's opcodes and follow their execution for our transaction. Yet I don't think that's the best way to approach this. I'd rather first walk away from the EVM and Geth, and move into Solidity lands. This should give us a more valuable overview of the high-level behavior of an ERC20 transfer operation.

### Solidity execution

The DAI smart contract was coded in [Solidity][230]. It is an object-oriented, high-level language that when compiled, outputs EVM bytecode able to deploy smart contracts on an EVM-compatible chain (Ethereum in our case).

DAI's source code can be found [verified in block explorers][231], or [in GitHub][232]. For ease of reference, I'll be pointing to the first.

Before we begin, let's always keep in mind that the EVM knows nothing about Solidity. It knows nothing about its variables, functions, the layout of contracts, ABI-encoding, etc. The Ethereum blockchain stores plain hard EVM bytecode, not fancy Solidity code.

You might wonder then how come when you go to any block explorer, they show you Solidity code at Ethereum addresses. Well, it's just a façade. In most block explorers people can upload Solidity source code, and the explorer takes care of compiling the source with specific compiler settings. If the compiler's output produced by the explorer matches what's stored at the specified address on the blockchain, then the contract's source code is said to be "verified". From then on, anyone that navigates to that address sees the Solidity code of that address, instead of only the EVM bytecode stored at it.

A non-trivial consequence of the above is that to some extent we're trusting block explorers to show us the legitimate code (which might not necessarily be true, even if [accidentally][233]). There are might be [alternatives][234] to this though - unless every time you want to read a contract you verify source code against your own node.

Anyway, back to DAI's Solidity code now.

On the DAI smart contract (compiled with Solidity [v0.5.12][235]), let's focus on [the function][236] to execute: `transfer`.

`function transfer(address dst, uint wad) external returns (bool) {     return transferFrom(msg.sender, dst, wad); }`

When `transfer` is run, it will [call another function][237] named `transferFrom`, then returning whatever boolean flag the latter returns. The first and second argument of `transfer` (here named `dst` and `wad`) are passed directly to `transferFrom`. This function additionally reads the sender's address (available as a [Solidity global variable][238] in `msg.sender`).

For our case, these would be the values passed to `transferFrom`:

`return transferFrom(     msg.sender, // 0x6fC27A75d76d8563840691DDE7a947d7f3F179ba (my address on the local testing node)     dst,        // 0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045 (Vitalik's address)     wad         // 1000000000000000000 (1 DAI in wei units) );`

Let's see the [`transferFrom` function][239] then.

`function transferFrom(address src, address dst, uint wad) public returns (bool) {    ... }`

First, the sender's [balance is checked][240] against the amount being transferred.

`require(balanceOf[src] >= wad, "Dai/insufficient-balance");`

It's simple: you cannot transfer more DAI than what you have in balance. If I didn't have 1 DAI, execution would halt at this point, returning an error with a message. Note that each address' balance is tracked on the smart contract storage. In a [map-like data structure named `balanceOf`][241]. If you have at least 1 DAI, I can assure you your account's address has a record somewhere in there.

Second, token [allowances are validated][242].

`// don't bother too much about this :) if (src != msg.sender && allowance[src][msg.sender] != uint(-1)) {     require(allowance[src][msg.sender] >= wad, "Dai/insufficient-allowance");     allowance[src][msg.sender] = sub(allowance[src][msg.sender], wad); }`

This does not concern us right now. Because we're not executing the transfer on behalf of another account. Though do note that's a mechanism all ERC20 tokens should implement - DAI not being the exception. In essence, you can approve other accounts to transfer DAI tokens from your account.

Third, the actual [balance swap][243] happens.

`balanceOf[src] = sub(balanceOf[src], wad); balanceOf[dst] = add(balanceOf[dst], wad);`

When sending 1 DAI, the sender's balance is decreased by 1000000000000000000, and the receiver's balance is incremented by 1000000000000000000. These operations are done reading and writing on the `balanceOf` data structure. It's worth noting the use of two special functions `add` and `sub` to do the math.

Why not simply use the `+` and `-` operators ?

Remember: this contract was compiled with Solidity 0.5.12. At that point in time the compiler did not include over/underflow checks as it does today. Thus developers had to remember (or be reminded 😛) to implement them by themselves where appropriate. Thus the use of `add` and `sub` in the DAI contract. They are just custom internal functions to perform addition and subtraction with bound checks to avoid arithmetic issues.

`function add(uint x, uint y) internal pure returns (uint z) {     require((z = x + y) >= x); }  function sub(uint x, uint y) internal pure returns (uint z) {     require((z = x - y) <= x); }`

The `add` function sums `x` and `y`, halting execution if the result of the operation is lower than `x` (thus preventing integer overflow).

The `sub` function subtracts `y` from `x`, halting execution if the result of the operation is greater than `x` (thus preventing integer underflow).

Fourth, a `Transfer` event is [emitted][244] (as [suggested by the ERC20 spec][245]).

`emit Transfer(src, dst, wad);`

An event is a logging operation. Data emitted in an event can later be retrieved from off-chain services reading the blockchain, though never by other contracts.

In our transfer operation the emitted event appears to log three elements. The sender's address (`0x6fC27A75d76d8563840691DDE7a947d7f3F179ba`), the receiver's address (`0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045`), and the amount sent (`1000000000000000000`).

The first two correspond to the parameters labeled as `indexed` in the event's declaration. Indexed parameters facilitate data retrieval, allowing filtering for any of the corresponding logged values. Unless the event is labeled as `anonymous`, the event identifier is also included as a topic.

Hence, being more specific, the `Transfer` event we're dealing with actually logs 3 topics (the event's identifier, the sender's address and the receiver's address) and 1 value (the amount of DAI transferred). We'll cover more details about this event once we get to lower-level EVM stuff.

At the end of the function, the boolean value `true` is [returned][246] (as [suggested by the ERC20 spec][247]).

`return true;`

That's a way of signaling that the transfer was successfully executed. This boolean flag is passed up to the outer `transfer` function that initiated the call (which simply returns it as well).

And that's it! If you've ever sent DAI, be certain that's the logic you've executed. That's the job you've paid to be done for you by a global decentralized network of nodes.

Hold on. I may have gone too far. That's kind of a lie. Because as I told you earlier, the EVM knows nothing about Solidity. Nodes execute no Solidity. They execute EVM bytecode.

It's time for the real deal.

### EVM execution

I'm turning quite technical in this section. I'll assume you're somewhat familiar with looking at some EVM bytecode. If you're not, I _highly_ recommend reading [this series][248] or [this newer one][249]. There you will find lots of the concepts in this section explained individually and in more depth.

DAI's raw bytecode is tough to look at - we already witnessed it in a previous section. A prettier way to study it is using a disassembled version. You can find Dai's disassembled bytecode [here][250] (I've extracted it to [this gist][251] for ease of reference).

#### Free memory pointer and call's value

The first three instructions shouldn't come as a surprise if you're already familiar with the Solidity compiler. It's simply initializing the free memory pointer.

`0x0: PUSH1     0x80 0x2: PUSH1     0x40 0x4: MSTORE`    

The Solidity compiler reserves memory slots from `0x00` to `0x80` for internal stuff. So the "free memory pointer" is a pointer to the first slot of EVM memory that can be freely used. It is stored at `0x40`, and at initialization points to `0x80`.

Keep in mind that all EVM opcodes have a counterpart implementation in Geth. For example, you can [really see][252] how the implementation of `MSTORE` pops two stack elements and writes to the EVM memory a word of 32 bytes:

`func opMstore(pc *uint64, interpreter *EVMInterpreter, scope *ScopeContext) ([]byte, error) {     // pop value of the stack     mStart, val := scope.Stack.pop(), scope.Stack.pop()     scope.Memory.Set32(mStart.Uint64(), &val)     return nil, nil }`

The next EVM instructions in DAI's bytecode ensure that the call doesn't hold any value. If it had, execution would halt at the `REVERT` instruction. Note the use of the `CALLVALUE` instruction (implemented [here][253]) to read the current call's value.

`0x5: CALLVALUE  0x6: DUP1       0x7: ISZERO     0x8: PUSH2     0x10 0xb: JUMPI      0xc: PUSH1     0x0 0xe: DUP1       0xf: REVERT`

Our call doesn't hold any value (the `value` field of the transaction was set to zero) - so we're good to continue.

#### Validating calldata (part 1)

Next: another check introduced by the compiler. This time it's figuring out whether the calldata's size (obtained with the `CALLDATASIZE` instruction - implemented [here][254]) is lower than 4 bytes (see the `0x4` and the `LT` instruction below ?). In such case it would jump to position `0x142`. Halting execution at the `REVERT` instruction in position `0x146`.

`0x10: JUMPDEST 0x11: POP        0x12: PUSH1     0x4 0x14: CALLDATASIZE 0x15: LT         0x16: PUSH2     0x142 0x19: JUMPI  ...  0x142: JUMPDEST   0x143: PUSH1     0x0 0x145: DUP1       0x146: REVERT` 

That means that in the DAI smart contract calldata's size is enforced to be _at least_ 4 bytes. That's because the ABI-encoding mechanism used by Solidity identifies functions with the first four bytes of the keccak256 hash of their signature (usually called "function selector" - [see the spec][255]).

If calldata didn't have at least 4 bytes, it wouldn't be possible to identify the function. So the compiler introduces the necessary EVM instructions to fail early in that scenario. That's what you witnessed above.

In order to call the `transfer(address,uint256)` function, the first four bytes of the calldata must match the function's selector. These are:

`$ cast sig "transfer(address,uint256)" 0xa9059cbb`

That's right. Exactly the same first 4 bytes of the `data` field of the transaction we built earlier:

`0xa9059cbb000000000000000000000000d8da6bf26964af9d7eed9e03e53415d37aa960450000000000000000000000000000000000000000000000000de0b6b3a7640000`

Now that the length of the calldata has been validated, it's time to use it. See below how the first four bytes calldata are placed at the top of the stack (main EVM instruction to notice here is `CALLDATALOAD`, implemented [here][256]).

`0x1a: PUSH1     0x0 0x1c: CALLDATALOAD 0x1d: PUSH1     0xe0 0x1f: SHR`       

In reality `CALLDATALOAD` pushes [32 bytes of calldata][257] to the stack. It needs to be chopped with [the `SHR` instruction][258] to keep the first 4 bytes.

#### Function dispatcher

Don't try to understand the following line by line. Instead, pay attention to the high-level pattern that stands out. I'll add some dividing lines to make it clearer:

`0x20: DUP1 0x21: PUSH4     0x7ecebe00 0x26: GT         0x27: PUSH2     0xb8 0x2a: JUMPI`

`0x2b: DUP1       0x2c: PUSH4     0xa9059cbb 0x31: GT         0x32: PUSH2     0x7c 0x35: JUMPI`

`0x36: DUP1       0x37: PUSH4     0xa9059cbb 0x3c: EQ         0x3d: PUSH2     0x6b4 0x40: JUMPI`

`0x41: DUP1       0x42: PUSH4     0xb753a98c 0x47: EQ         0x48: PUSH2     0x71a 0x4b: JUMPI`

It's no coincidence that some of the hex values being pushed to the stack are 4 bytes long. Those are, indeed, function selectors.

The set of instructions above is a common structure of the bytecode that the Solidity compiler produces. It's usually referred to as "function dispatcher". It resembles an if-else or switch flow. It's simply trying to match the first four bytes of the calldata against the set of known selectors of the contract's functions. Once it finds a coincidence, execution will jump to another section of the bytecode. Where the instructions for that particular function are placed.

Following the above logic, the EVM matches the first four bytes of calldata against the selector of the ERC20 `transfer` function: `0xa9059cbb`. And jumps to bytecode position `0x6b4`. That's how the EVM is told to start executing the transfer of DAI.

#### Validating calldata (part 2)

Having matched the selector and jumped, now the EVM is about to start running specific code related to the function. But before jumping into its details, it needs to somehow remember where to continue executing once all function-related logic has been executed.

The way to do that is simply keeping the appropriate bytecode position in the stack. See the `0x700` value being pushed below. It will linger in stack until at some point (later down the road) it will be retrieved and be used to jump back to wrap up execution.

`0x6b4: JUMPDEST   0x6b5: PUSH2     0x700`

Now let's get more specific to the `transfer` function.

The compiler embeds some logic to ensure the calldata's size is correct for a function with two parameters of `address` and `uint256` type. For the `transfer` function, that is at least 68 bytes (4 bytes for the selector + 64 bytes for the two ABI-encoded parameters).

`0x6b8: PUSH1     0x4 0x6ba: DUP1       0x6bb: CALLDATASIZE 0x6bc: SUB        0x6bd: PUSH1     0x40 0x6bf: DUP2       0x6c0: LT         0x6c1: ISZERO     0x6c2: PUSH2     0x6ca 0x6c5: JUMPI      0x6c6: PUSH1     0x0 0x6c8: DUP1       0x6c9: REVERT`

If the calldata's size was lower, execution would halt at the `REVERT` in position `0x6c9`. Since our transaction's calldata has been correctly ABI-encoded and therefore has the appropriate length, execution jumps to position `0x6ca`.

#### Reading parameters

Next step is for the EVM to read the two parameters provided in the calldata. Those would be the 20-bytes-long address `0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045` and the number `1000000000000000000` (`0x0de0b6b3a7640000` in hex). Both were ABI-encoded in chunks of 32 bytes. Thus there needs to be some basic manipulation to read the right values and place them at the top of the stack.

`0x6ca: JUMPDEST   0x6cb: DUP2       0x6cc: ADD        0x6cd: SWAP1      0x6ce: DUP1       0x6cf: DUP1       0x6d0: CALLDATALOAD 0x6d1: PUSH20    0xffffffffffffffffffffffffffffffffffffffff 0x6e6: AND        0x6e7: SWAP1      0x6e8: PUSH1     0x20 0x6ea: ADD        0x6eb: SWAP1      0x6ec: SWAP3      0x6ed: SWAP2      0x6ee: SWAP1      0x6ef: DUP1       0x6f0: CALLDATALOAD 0x6f1: SWAP1      0x6f2: PUSH1     0x20 0x6f4: ADD        0x6f5: SWAP1      0x6f6: SWAP3      0x6f7: SWAP2      0x6f8: SWAP1      0x6f9: POP        0x6fa: POP        0x6fb: POP        0x6fc: PUSH2     0x1df4 0x6ff: JUMP`

Just to make it more visual, after sequentially applying the above set of instructions (up to `0x6fb`), the top of stack looks like this:

`0x0000000000000000000000000000000000000000000000000de0b6b3a7640000 0x000000000000000000000000d8da6bf26964af9d7eed9e03e53415d37aa96045`

And that's how the EVM swiftly extracts both arguments from calldata, placing them on the stack for future use.

The final two instructions above (bytecode positions `0x6fc` and `0x6ff`) simply make execution jump to position `0x1df4`. Let's continue there.

#### The `transfer` function

During the brief Solidity analysis, we saw that the `transfer(address,uint256)` function was a thin wrapper that called the more complex `transferFrom(address,address,uint256)` function. The compiler translates such internal call to [these EVM instructions][259]:

`0x1df4: JUMPDEST   0x1df5: PUSH1     0x0 0x1df7: PUSH2     0x1e01 0x1dfa: CALLER     0x1dfb: DUP5       0x1dfc: DUP5       0x1dfd: PUSH2     0xa25 0x1e00: JUMP`

First notice the instruction pushing the value `0x1e01`. That's how the EVM is instructed to "remember" the exact position where it should jump back to continue execution after the upcoming internal call.

Then, pay attention to the use of `CALLER` (because in Solidity the internal call uses `msg.sender`). As well as to the two `DUP5` instructions. Together, these are putting at the top of the stack the three necessary arguments for `transferFrom`: the caller's address, the receiver's address, and the amount to be transferred. The last two were already somewhere in the stack, thus the use of `DUP5`. The top of the stack now holds all necessary arguments:

`0x0000000000000000000000000000000000000000000000000de0b6b3a7640000 0x000000000000000000000000d8da6bf26964af9d7eed9e03e53415d37aa96045 0x0000000000000000000000006fc27a75d76d8563840691dde7a947d7f3f179ba`

Finally, following instructions `0x1dfd` and `0x1e00`, execution jumps to position `0xa25`. Where the EVM will start executing the instructions corresponding to the `transferFrom` function.

#### The `transferFrom` function

First thing that needs to be checked is whether the sender has enough DAI in balance - otherwise reverting. The sender's balance is kept in the contract storage. The fundamental EVM instruction needed is then `SLOAD`. However, `SLOAD` needs to know what storage _slot_ needs to be read. For mappings (the type of Solidity data structure that is holding account balances in the DAI smart contract), that's not so straightforward to tell.

I won't dive here into the internal layout of Solidity state variables in contract storage. You may read about it [here for v0.5.15][260]. Suffice to say that given the key address `k` for the mapping `balanceOf`, its corresponding `uint256` value will be kept at storage slot `keccak256(k . p)`, where `p` is the slot position of the mapping itself and `.` is concatenation. You can do the math yourself.

For simplicity, let's just highlight a couple of operations that need to happen. The EVM must i) calculate the storage slot for the mapping, ii) read the value, iii) compare it against the amount to be transferred (a value already in stack). Therefore we should see instructions like `SHA3` for the hashing, `SLOAD` for reading storage, and `LT` for the comparison.

`0xa25: JUMPDEST   0xa26: PUSH1     0x0 0xa28: DUP2       0xa29: PUSH1     0x2 0xa2b: PUSH1     0x0 0xa2d: DUP7       0xa2e: PUSH20    0xffffffffffffffffffffffffffffffffffffffff 0xa43: AND        0xa44: PUSH20    0xffffffffffffffffffffffffffffffffffffffff 0xa59: AND        0xa5a: DUP2       0xa5b: MSTORE     0xa5c: PUSH1     0x20 0xa5e: ADD        0xa5f: SWAP1      0xa60: DUP2       0xa61: MSTORE     0xa62: PUSH1     0x20 0xa64: ADD        0xa65: PUSH1     0x0 0xa67: SHA3      --> calculating storage slot 0xa68: SLOAD     --> reading storage 0xa69: LT        --> comparing balance against amount 0xa6a: ISZERO     0xa6b: PUSH2     0xadc 0xa6e: JUMPI`    

If the sender didn't have enough DAI, execution would [follow at `0xa6f` and finally hit the `REVERT` at `0xadb`][261]. Since I did not forget to load 1 DAI in my sender account's balance, let's then proceed to position `0xadc`.

The following set of instructions correspond to the EVM validating whether the caller matches the sender's address (remember the `if (src != msg.sender ...) { ... }` code segment in the contract).

`0xadc: JUMPDEST   0xadd: CALLER     0xade: PUSH20    0xffffffffffffffffffffffffffffffffffffffff 0xaf3: AND        0xaf4: DUP5       0xaf5: PUSH20    0xffffffffffffffffffffffffffffffffffffffff 0xb0a: AND        0xb0b: EQ         0xb0c: ISZERO     0xb0d: DUP1       0xb0e: ISZERO     0xb0f: PUSH2     0xbb4 0xb12: JUMPI ... 0xbb4: JUMPDEST   0xbb5: ISZERO     0xbb6: PUSH2     0xdb2 0xbb9: JUMPI`

Since they don't match, continue executing at position `0xdb2`.

Doesn't this code below remind you of something ? Check out the instructions being used. Again, don't focus on them individually. Use your intuition to spot high-level patterns and the most relevant instructions.

`0xdb2: JUMPDEST   0xdb3: PUSH2     0xdfb 0xdb6: PUSH1     0x2 0xdb8: PUSH1     0x0 0xdba: DUP7       0xdbb: PUSH20    0xffffffffffffffffffffffffffffffffffffffff 0xdd0: AND        0xdd1: PUSH20    0xffffffffffffffffffffffffffffffffffffffff 0xde6: AND        0xde7: DUP2       0xde8: MSTORE     0xde9: PUSH1     0x20 0xdeb: ADD        0xdec: SWAP1      0xded: DUP2       0xdee: MSTORE     0xdef: PUSH1     0x20 0xdf1: ADD        0xdf2: PUSH1     0x0 0xdf4: SHA3       0xdf5: SLOAD      0xdf6: DUP4       0xdf7: PUSH2     0x1e77 0xdfa: JUMP`

If it resembles reading a mapping from storage, it's because it is! The above is the EVM reading the sender's balance from the `balanceOf` mapping.

Execution then jumps to position `0x1e77`, where the body of the `sub` function is placed.

The `sub` function subtracts two numbers, reverting upon integer underflow. I'm not including the bytecode, though you can follow it [here][262]. The result of the arithmetic operation is kept on the stack.

Back to the instructions corresponding to `transferFrom` function's body, now the result of the subtraction is to be written to storage - updating the `balanceOf` mapping. Try to notice below the calculation performed to obtain the appropriate storage slot of the mapping entry, which leads to the execution of the `SSTORE` instruction. This instruction is the one that effectively writes data to state - that is, that updates the contract's storage.

`0xdfb: JUMPDEST   0xdfc: PUSH1     0x2 0xdfe: PUSH1     0x0 0xe00: DUP7       0xe01: PUSH20    0xffffffffffffffffffffffffffffffffffffffff 0xe16: AND        0xe17: PUSH20    0xffffffffffffffffffffffffffffffffffffffff 0xe2c: AND        0xe2d: DUP2       0xe2e: MSTORE     0xe2f: PUSH1     0x20 0xe31: ADD        0xe32: SWAP1      0xe33: DUP2       0xe34: MSTORE     0xe35: PUSH1     0x20 0xe37: ADD        0xe38: PUSH1     0x0 0xe3a: SHA3       0xe3b: DUP2       0xe3c: SWAP1      0xe3d: SSTORE` 

A pretty similar set of opcodes is run to update the receiver's account balance. First [is read from the `balanceOf` mapping in storage][263]. Then the balance is added to the amount being transferred [using the `add` function][264]. At last the result is [written to the appropriate storage slot][265].

#### Logging

In the contract's code the `Transfer` event was emitted after updating balances. So there has to be a set of instructions in the bytecode under analysis that take care of emitting such event with the appropriate data.

However, events are yet another thing that belong to Solidity's fantasy world. In EVM world, events correspond to logging operations.

Logging is performed with the available set of `LOG` instructions. There are a couple of variants, depending on how many topics are to be logged. In DAI's case, we already noted that the emitted `Transfer` event has 3 topics.

Then it's no surprise to find a set of instructions that lead to running the `LOG3` instruction.

`0xeca: POP        0xecb: DUP3       0xecc: PUSH20    0xffffffffffffffffffffffffffffffffffffffff 0xee1: AND  0xee2: DUP5 0xee3: PUSH20    0xffffffffffffffffffffffffffffffffffffffff 0xef8: AND        0xef9: PUSH32    0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef 0xf1a: DUP5       0xf1b: PUSH1     0x40 0xf1d: MLOAD      0xf1e: DUP1       0xf1f: DUP3       0xf20: DUP2       0xf21: MSTORE     0xf22: PUSH1     0x20 0xf24: ADD        0xf25: SWAP2      0xf26: POP        0xf27: POP        0xf28: PUSH1     0x40 0xf2a: MLOAD      0xf2b: DUP1       0xf2c: SWAP2      0xf2d: SUB        0xf2e: SWAP1      0xf2f: LOG3`      

There's at least one value that stands out in those instructions: `0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef`. That's the event's main identifier. Also called topic 0. It is a static value calculated by the compiler at compiling time (embedded in the contract's runtime bytecode). As noted previously, no more than the hash of the event's signature:

`$ cast keccak "Transfer(address,address,uint256)" 0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef`

Right before reaching the `LOG3` instruction, the stack looks like this:

`0x0000000000000000000000000000000000000000000000000000000000000080 0x0000000000000000000000000000000000000000000000000000000000000020 0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef -- topic 0 (event identifier) 0x0000000000000000000000006fc27a75d76d8563840691dde7a947d7f3F179ba -- topic 1 (sender's address) 0x000000000000000000000000d8da6bf26964af9d7eed9e03e53415d37aa96045 -- topic 2 (receiver's address)`

Where's the amount of the transfer then ? In memory! Before reaching `LOG3`, the EVM was first instructed to store the amount in memory. So that it can later be consumed by the logging instruction. If you look at position `0xf21`, you'll see the `MSTORE` instruction in charge of doing so.

So once `LOG3` is reached, it is safe for the EVM to grab the actual value logged from memory, starting at offset `0x80` and reading `0x20` bytes (first two stack elements above).

Another way of understanding logging is looking at its [implementation in Geth][266]. In there you'll find a single function in charge of handling all logging instructions. You can see how i) an empty array of topics [is initialized][267], ii) the memory offset and data size are [read from the stack][268], iii) the topics are [read from stack][269] and [inserted in the array][270], iv) the value is [read from memory][271], v) the log, containing address where it was emitted, topics and value, [is appended][272].

How those logs are later recovered, we'll find out soon enough.

#### Returning

Last thing for the `transferFrom` function is to return the boolean value `true`. That's why the first instruction after `LOG3` is simply pushing the `0x1` value to the stack.

`0xf30: PUSH1    0x1`

The next instructions prepare the stack to exit the `transferFrom` function, going back to its wrapper `transfer` function. Remember that the position for this next jump had been already stored in the stack - that's why you don't see it in the opcodes below.

`0xf32: SWAP1      0xf33: POP        0xf34: SWAP4      0xf35: SWAP3      0xf36: POP        0xf37: POP        0xf38: POP        0xf39: JUMP`

Back in the `transfer` function, all there's to do is to prepare the stack for the final jump. To a position where execution will be wrapped up. The position for this upcoming jump had also been stored in the stack previously (remember the `0x700` value being pushed?).

`0x1e01: JUMPDEST   0x1e02: SWAP1      0x1e03: POP        0x1e04: SWAP3      0x1e05: SWAP2      0x1e06: POP        0x1e07: POP        0x1e08: JUMP` 

All that's left is to prepare the stack for the final instruction: `RETURN`. This instruction is in charge of reading some data from memory, and passing it back to the original caller.

For the DAI transfer, the returned data would simply include the `true` boolean flag returned by the `transfer` function. Remember that the value had been already placed at the stack.

The EVM begins with grabbing the first available position of free memory. This is done by reading the free memory pointer:

`0x700: JUMPDEST   0x701: PUSH1     0x40 0x703: MLOAD`

Next, the value must be stored in memory with `MSTORE`. Although not so straightforward to tell, the instructions below are just the ones the compiler found most appropriate to prepare the stack for the `MSTORE` operation.

`0x704: DUP1       0x705: DUP3       0x706: ISZERO     0x707: ISZERO     0x708: ISZERO     0x709: ISZERO     0x70a: DUP2       0x70b: MSTORE`

The `RETURN` instruction copies the returned data from memory. So it needs to be told how much memory to read, and where to start. The instructions below simply tell the EVM to read and return `0x20` bytes from memory starting at the free memory pointer.

`0x70c: PUSH1     0x20 0x70e: ADD        0x70f: SWAP2      0x710: POP        0x711: POP        0x712: PUSH1     0x40 0x714: MLOAD      0x715: DUP1       0x716: SWAP2      0x717: SUB        0x718: SWAP1      0x719: RETURN` 

The value `0x0000000000000000000000000000000000000000000000000000000000000001` (corresponding to the boolean `true`) is returned.

Execution halts.

### The interpreter (part 2)

Bytecode execution has finished. The interpreter must stop [iterating][273]. In Geth, that's done like this:

`// interpreter's execution loop for {      ...     // execute the operation     res, err = operation.execute(&pc, in, callContext)     if err != nil {         break     }     ... }`

That means that the implementation of the `RETURN` opcode should somehow return an error. Even for successful executions such as ours. Indeed, [it does][274]. Though it acts as a flag - the error is actually [deleted][275] when it matches the flag returned by the successful execution of the `RETURN` opcode.

### Gas refunds and payments

With the interpreter run finished, we're [back in the call][276] that originally triggered it. The run [was completed successfully][277]. Thus the returned data and any gas remaining is simply [returned][278].

The call's finished as well. Execution follows wrapping up the state transition.

First [providing gas refunds][279]. These are added to any gas leftovers in the transaction. The refunded amount is capped to 1/5 of the gas used (due to [EIP 3529][280]). All gas available now (remaining plus refunded) is [paid back in ETH][281] to the sender's account, at the rate originally set by the sender in the transaction. All gas left is [re-added][282] to the available gas in the block - so that subsequent transactions can consume it.

Then [paying the coinbase address][283] (the miner's address in PoW, the validator's address in PoS) what was originally promised: the tip. Interestingly, payment is done for all gas used during execution. Even if some of it was later refunded. Moreover, note [here][284] how the _effective_ tip is calculated. Not only noticing that it is capped by the `maxPriorityFeePerGas` transaction field. But more importantly, realizing that it _does not_ include the base fee! That's no mistake - Ethereum enjoys [watching the ETH burn][285].

At last [the execution result is wrapped in a prettier structure][286]. Including the used gas, any EVM error that could have aborted execution (none in our case), along with the returned data from the EVM.

### Building the transaction receipt

The structure representing the execution result is now passed [back][287] [up][288]. At this point Geth does [some internal cleanup][289] of the execution state. Once done, it [accumulates the gas used][290] in the transaction (including refunds).

Most importantly, now is the time the transaction receipt is created. The receipt's an object summarizing data related to the transaction's execution. It includes information such as [execution status][291] (success/failure), the [transaction's hash][292], [gas units used][293], [address of created contract][294] (none in our case), [logs emitted][295], the [transaction's bloom filter][296], and [more][297].

We'll retrieve the full contents of our transaction's receipt soon.

If you'd like to dig deeper into the transaction's logs and the role of the bloom filter, check out [noxx's article][298].

## Sealing the block

Execution of subsequent transactions continue happening until the block runs out of space.

That's when the node invokes the consensus engine to [finalize][299] the block. In PoW that entails [accumulating mining rewards][300] (issuing [full rewards][301] in ETH to the coinbase address, along with [partial rewards][302] for [ommer blocks][303]) and [updating the final state root][304] of the block accordingly.

Next, the actual block [is assembled][305], putting [all data in its right place][306]. Including information such as the header's [transaction hash][307], or the [receipts hash][308].

All ready for the real PoW mining now. [A new "task"][309] is created and [pushed to the right listener][310]. The [sealing task][311], delegated to the consensus engine, starts.

I won't explain in detail how the actual mining is done for PoW. Lots about it on the Internet already. Just note that in Geth this involves a [multithreaded try-and-error process][312] to [find a number][313] that [satisfies a necessary condition][314]. Needless to say, once Ethereum switches to Proof of Stake the sealing process will be handled quite differently.

The mined block is [pushed to the appropriate channel][315] and [received at the results loop][316]. Where [receipts and logs are updated accordingly][317] with the latest block data after its been effectively mined.

The block is finally [written to the chain][318], placing it at its head.

## Broadcasting the block

Next step is to [announce to the whole network][319] that a new block has been mined. Meanwhile the block is internally [stored into the set of pending ones][320]. Patiently awaiting for confirmations from other nodes.

The announcement is done [posting a specific event][321], picked up by the [mined broadcast loop][322]. In there the block is fully [propagated to a subset of peers][323] and [made available][324] to the rest in a lighter fashion.

More specifically, propagation entails [sending block data to the square root of connected peers][325]. Internally this is implemented [pushing the data to the queued blocks channel][326], until its [sent][327] via the [p2p layer][328]. The p2p message is identified as `NewBlockMsg`. The rest receives a [lightweight announcement including the block hash][329].

Note that this is only valid for PoW. [Block propagation will happen on consensus engines][330] in Proof of Stake.

## Verifying the block

Peers are continuously [listening for messages][331]. Each type of [possible message has an associated handler][332] that is [invoked][333] as soon as the corresponding message is received.

Consequently, upon getting the `NewBlockMsg` message with the block's data, [its corresponding handler][334] is executed. The handler [decodes the message][335] and runs some [early validations][336] on the propagated block. These include preliminary [sanity checks][337] on the header's data, mostly ensuring they are filled and bounded. As well as validations for the block's [uncle][338] and [transaction][339] hashes.

Then [the peer that sent the message is marked][340] as owning the block. Thus avoiding to later propagate the block back to it.

At last, the packet is [passed down][341] to [a second handler][342], where the block is going to be [enqueued for import][343] into the local copy of the chain. The enqueuing is done by [sending a direct import request to the corresponding channel][344]. When the request is [picked up][345], it triggers the [actual enqueuing operation][346]. Finally [pushing][347] the block data to the queue.

The block is now in the local queue ready to be processed. This queue is [read from periodically][348] in the node's block fetcher main loop. When the block makes it to the front, the node will pick it up and [attempt to import it][349].

There are a at least two validations worth highlighting prior to the actual insertion of the candidate block.

First, the local chain must already [include the parent of the propagated block][350].

Second, the block's header [must be valid][351]. These validations are _the real ones_. Meaning, the ones that actually matter for consensus, and are specified in Ethereum's [yellow paper][352]. Hence, they are [handled by the consensus engine][353].

Just as examples, the engine checks that the [block's proof of work is valid][354], or that the block's timestamp is [not in the past][355] nor [not too far ahead into the future][356], or that the block's number [has been increased correctly][357]. Among others.

Having verified that its header follows consensus rules, the whole block is further [propagated to a subset of peers][358]. Only then [the actual import is run][359].

There's _a lot_ happening during an import. So I'll cut right to the chase.

After [several additional validations][360], the [parent's block state is retrieved][361]. This is the state on top of which the first transaction of the new block is going to be executed. Using it as a reference point, [the entire block is processed][362]. If you've ever heard that all Ethereum nodes are expected to execute and validate every single transaction, now you can be certain of it. Afterwards, the [post-state is validated][363] (see how [here][364]). Finally, [the block is written][365] to the local chain.

The successful import leads to [announcing][366] (not fully broadcasting) the block to the rest of the node's peers.

The whole verification process is replicated across all nodes that receive the block. A good portion will accept it into their local chains, and later newer blocks will arrive to insert on top of it.

## Retrieving the transaction

After a few blocks have been mined on top of the one where the transaction was included, one can start to safely assume that the transaction has been indeed confirmed.

Retrieving the transaction from the chain is quite simple. All we need is its hash. Conveniently, it was obtained as soon as we first submitted the transaction.

The data of the transaction itself, plus the block's hash and number, can always be retrieved at the node's `eth_getTransactionByHash` endpoint. Unsurprisingly, it now returns:

`{     "hash": "0xbf77c4a9590389b0189494aeb2b2d68dc5926a5e20430fb5bc3c610b59db3fb5",     "type": 2,     "accessList": [],     "blockHash": "0xe880ba015faa9aeead0c41e26c6a62ba4363822ddebde6dd77a759a753ad2db2",     "blockNumber": 15166167,     "transactionIndex": 0,     "confirmations": 6,     "from": "0x6fC27A75d76d8563840691DDE7a947d7f3F179ba",     "maxPriorityFeePerGas": 2000000000,     "maxFeePerGas": 120000000000,     "gasLimit": 40000,     "to": "0x6B175474E89094C44Da98b954EedeAC495271d0F",     "value": 0,     "nonce": 0,     "data": "0xa9059cbb000000000000000000000000d8da6bf26964af9d7eed9e03e53415d37aa960450000000000000000000000000000000000000000000000000de0b6b3a7640000",     "r": "0x057d733933b12238a2aeb0069b67c6bc58ca8eb6827547274b3bcf4efdad620a",     "s": "0x00e49937ec81db89ce70ebec5e51b839c0949234d8aad8f8b55a877bd78cc293",     "v": 1,     "creates": null,     "chainId": 31337 }`

The transaction's receipt can be requested at the `eth_getTransactionReceipt` endpoint. Depending on the node you're running this query, it might be the case you also get additional information on top of the expected transaction receipt data. This is the transaction receipt I got from my local fork of mainnet:

`{     "to": "0x6B175474E89094C44Da98b954EedeAC495271d0F",     "from": "0x6fC27A75d76d8563840691DDE7a947d7f3F179ba",     "contractAddress": null,     "transactionIndex": 0,     "gasUsed": 34706,     "logsBloom": "0x00000000000000000800000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000008000000000000008000000000000000000000000000000000000000011000000000000000000000000000000000000000000000000000010000000000000004000000000000000000000000080000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000002000000000000000000000000000000000000800000000000000000000000000000000000000000000000000000",     "blockHash": "0x8b6d44d6cf39d01181b90677f8a77a2605d6e70c40d649eda659499063a19c77",     "transactionHash": "0xbf77c4a9590389b0189494aeb2b2d68dc5926a5e20430fb5bc3c610b59db3fb5",     "logs": [         {             "transactionIndex": 0,             "blockNumber": 15166167,             "transactionHash": "0xbf77c4a9590389b0189494aeb2b2d68dc5926a5e20430fb5bc3c610b59db3fb5",             "address": "0x6B175474E89094C44Da98b954EedeAC495271d0F",             "topics": [                 "0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef",                 "0x0000000000000000000000006fc27a75d76d8563840691dde7a947d7f3f179ba",                 "0x000000000000000000000000d8da6bf26964af9d7eed9e03e53415d37aa96045"             ],             "data": "0x0000000000000000000000000000000000000000000000000de0b6b3a7640000",             "logIndex": 0,             "blockHash": "0x8b6d44d6cf39d01181b90677f8a77a2605d6e70c40d649eda659499063a19c77"         }     ],     "blockNumber": 15166167,     "confirmations": 6, // number of blocks I waited before fetching the receipt     "cumulativeGasUsed": 34706,     "effectiveGasPrice": 9661560402,     "type": 2,     "byzantium": true,     "status": 1 }`

You saw that ? It says `"status": 1`.

That can only mean one thing: success!

## Afterword

There's definitely _far more_ to this story.

In some sense it's never-ending. There's always one more caveat. One more side note. An alternative execution path. Another node implementation. Another EVM instruction I might have skipped. Another wallet that handles things differently. All things that would take us one step closer to finding out "the real truth" behind what happens when you send 1 DAI.

Luckily, that's not what I intended to do. I hope the last +10k words did not make you think that 😛. Allow me to shed some light here.

In hindsight, this article is the byproduct of mixing curiosity and frustration.

Curiosity because I've been doing Ethereum smart contract security for more than 4 years, yet I hadn't took as much as time as I would've liked to manually explore, in depth, the complexities of the base layer. I really wanted to gain first-hand experience looking at the actual implementation of Ethereum itself. But smart contracts always got in the middle. Now that I've finally managed to found more peaceful times, it seemed the right time to go back to roots and embark on this adventure.

Curiosity wasn't enough though. I needed an excuse. A trigger. I knew what I had in mind was gonna be tough. So I needed a strong enough reason not only to get started. But also, more importantly, to get re-started whenever I would feel I was tired of trying to make sense out of Ethereum's code.

I found it where I wasn't looking. I found it in frustration.

Frustration at the absolutely mind-blowing lack of transparency we've grown so used to when sending money. If you've ever needed to do it in a developing country under increasingly strict capital controls, no doubt you feel me. So I wanted to remind myself that we can do better. I decided to channel my frustration in writing.

This article's also served me as a reminder. That if you can escape the fuzz, the price, the monkeys in JPEGs, the ponzis, the rugpulls, the thefts, there's still value in here. This is no "magic" internet money. There's real math, cryptography and computer science going. Being open source, you can see every single piece moving. You can almost touch them. No matter the day nor the time. No matter who you are. No matter where you come from.

So, I'm sorry for the clickbait-y title. This was never about what happens when you send 1 DAI.

It was about having the possibility to understand it.

[1]: https://notonlyowner.com/learn/que-pasa-cuando-envias-un-dai
[2]: https://makerdao.world/en/learn/Dai/
[3]: https://metamask.io
[4]: #building-the-transaction
[5]: #the-transactions-data-field
[6]: #gas-wizardry
[7]: #access-list-and-transaction-type
[8]: #signing-the-transaction
[9]: #serialization
[10]: #submitting-the-transaction
[11]: #reception
[12]: #inspecting-the-mempool
[13]: #propagation
[14]: #work-preparation-and-transaction-inclusion
[15]: #execution
[16]: #preparation-part-1
[17]: #preparation-part-2
[18]: #the-call
[19]: #the-interpreter-part-1
[20]: #solidity-execution
[21]: #evm-execution
[22]: #free-memory-pointer-and-calls-value
[23]: #validating-calldata-part-1
[24]: #function-dispatcher
[25]: #validating-calldata-part-2
[26]: #reading-parameters
[27]: #the-transfer-function
[28]: #the-transferFrom-function
[29]: #logging
[30]: #returning
[31]: #the-interpreter-part-2
[32]: #gas-refunds-and-payments
[33]: #building-the-transaction-receipt
[34]: #sealing-the-block
[35]: #broadcasting-the-block
[36]: #verifying-the-block
[37]: #retrieving-the-transaction
[38]: #afterword
[39]: https://github.com/ethereumbook/ethereumbook/blob/develop/05wallets.asciidoc
[40]: https://chainlist.org/
[41]: https://www.jsonrpc.org/
[42]: https://eips.ethereum.org/EIPS/eip-20
[43]: https://library.dedaub.com/contracts/Ethereum/6b175474e89094c44da98b954eedeac495271d0f/source
[44]: https://library.dedaub.com/contracts/Ethereum/6b175474e89094c44da98b954eedeac495271d0f/source?line=122
[45]: https://docs.soliditylang.org/en/latest/abi-spec.html
[46]: https://soliditylang.org/
[47]: https://book.getfoundry.sh/reference/cast/cast-calldata.html
[48]: https://ethereum.org/en/developers/docs/gas/#what-is-gas
[49]: https://docs.infura.io/infura/networks/ethereum/json-rpc-methods/eth_feehistory
[50]: https://composer.alchemyapi.io?share=eJwdyEEKgCAQBdC7.LULCyTwBK1atomIoSaKUkMnIqK7J_0e78G40OphtYJnuULcfjuWJUwNOYZF9jAz12uSEG8oHBTJtbSfnGA7FDrfTsJJMrrSKKNVZXr07wcDJR59
[51]: https://github.com/ethereum/execution-apis/blob/main/src/eth/fee_market.json#L14-L94
[52]: https://www.blocknative.com/blog/eip-1559-fees
[53]: https://eips.ethereum.org/EIPS/eip-2930
[54]: https://eips.ethereum.org/EIPS/eip-2718
[55]: https://eips.ethereum.org/EIPS/eip-1559#abstract
[56]: https://en.wikipedia.org/wiki/ECDSA
[57]: https://ethereum.org/en/developers/docs/data-structures-and-encoding/rlp/
[58]: https://github.com/ethereumjs/ethereumjs-monorepo/tree/master/packages/tx
[59]: https://docs.ethers.io/v5/single-page/
[60]: https://ethereum.org/en/developers/docs/data-structures-and-encoding/rlp/
[61]: https://www.youtube.com/watch?v=uZ7vkmUNTPA
[62]: https://github.com/ethereum/go-ethereum/tree/v1.10.18
[63]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/internal/ethapi/api.go#L1698-L1706
[64]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/internal/ethapi/api.go#L1702
[65]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/internal/ethapi/api.go#L1623-L1625
[66]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/ethconfig/config.go#L95
[67]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/internal/ethapi/api.go#L1626-L1629
[68]: https://eips.ethereum.org/EIPS/eip-155
[69]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/internal/ethapi/api.go#L1630
[70]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/api_backend.go#L245-L247
[71]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/tx_pool.go#L852-L857
[72]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/tx_pool.go#L843-L850
[73]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/tx_pool.go#L226-L271
[74]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/tx_pool.go#L896-L901
[75]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/tx_pool.go#L902-L910
[76]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/tx_pool.go#L648-L753
[77]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/tx_pool.go#L586-L646
[78]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/tx_pool.go#L604-L607
[79]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/tx_pool.go#L595-L598
[80]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/tx_pool.go#L49-L53
[81]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/tx_pool.go#L628-L631
[82]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/tx_pool.go#L632-L636
[83]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/internal/ethapi/api.go#L1633-L1645
[84]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/internal/ethapi/api.go#L1646
[85]: https://www.paradigm.xyz/2020/08/ethereum-is-a-dark-forest
[86]: https://arxiv.org/pdf/1904.05234.pdf
[87]: https://github.com/flashbots/mev-geth
[88]: https://docs.flashbots.net/flashbots-protect/rpc/quick-start
[89]: https://ethereum.org/en/developers/docs/consensus-mechanisms/pow/
[90]: https://ethereum.org/en/developers/docs/consensus-mechanisms/pos/
[91]: https://github.com/ethereum/builder-specs
[92]: https://github.com/ethereum/devp2p/
[93]: https://github.com/ethereum/devp2p/blob/master/caps/eth.md#transaction-exchange
[94]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/handler.go#L524-L528
[95]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/node/defaults.go#L64
[96]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/handler.go#L603-L607
[97]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/handler.go#L621-L622
[98]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/protocols/eth/peer.go#L183-L198
[99]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/protocols/eth/peer.go#L213-L223
[100]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/tx_pool.go#L390-L396
[101]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/tx_pool.go#L184
[102]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/tx_pool.go#L1338-L1345
[103]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/tx_pool.go#L254
[104]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/tx_pool.go#L536
[105]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L280
[106]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L293-L296
[107]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L531-L627
[108]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L456-L477
[109]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L436
[110]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L456-L459
[111]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L533
[112]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L534
[113]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L1114-L1117
[114]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L982-L989
[115]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L990-L998
[116]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L1003
[117]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L1004
[118]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L1006
[119]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L1017
[120]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L1024-L1027
[121]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/consensus/ethash/consensus.go#L587
[122]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/consensus/ethash/consensus.go#L337-L351
[123]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L1031-L1035
[124]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L758-L768
[125]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L1128
[126]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L1063
[127]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L1071-L1082
[128]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L849
[129]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L901-L904
[130]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L835
[131]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L87
[132]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L90
[133]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L836
[134]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L838
[135]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L839-L842
[136]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_processor.go#L144
[137]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_processor.go#L145
[138]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/types/transaction.go#L563
[139]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/types/transaction.go#L565
[140]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/types/transaction.go#L612
[141]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/types/transaction_signing.go#L195
[142]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_processor.go#L149-L151
[143]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/evm.go#L58-L69
[144]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_processor.go#L151
[145]: https://en.wikipedia.org/wiki/Stack_machine
[146]: https://en.wikipedia.org/wiki/256-bit_computing
[147]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/evm.go#L128
[148]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/evm.go#L137
[149]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/interpreter.go#L72-L101
[150]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_processor.go#L97-L98
[151]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_processor.go#L101
[152]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_transition.go#L181
[153]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_transition.go#L279-L284
[154]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_transition.go#L287
[155]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_transition.go#L218-L224
[156]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_transition.go#L225-L228
[157]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_transition.go#L229-L233
[158]: https://eips.ethereum.org/EIPS/eip-3607
[159]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_transition.go#L239-L246
[160]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_transition.go#L247-L250
[161]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_transition.go#L253
[162]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_transition.go#L259
[163]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_transition.go#L201-L203
[164]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_transition.go#L204-L206
[165]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_transition.go#L210
[166]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_transition.go#L305-L313
[167]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_transition.go#L124
[168]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/params/protocol_params.go#L32
[169]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_transition.go#L124
[170]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/params/protocol_params.go#L88
[171]: https://eips.ethereum.org/EIPS/eip-2028
[172]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/params/protocol_params.go#L34
[173]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_transition.go#L124
[174]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_transition.go#L316-L318
[175]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_transition.go#L320-L323
[176]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_transition.go#L302
[177]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_transition.go#L332
[178]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_transition.go#L333
[179]: https://www.youtube.com/watch?v=wMOkm57vu0k
[180]: https://www.evm.codes
[181]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/evm.go#L168
[182]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/evm.go#L164-L167
[183]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/evm.go#L169-L172
[184]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/params/protocol_params.go#L75
[185]: https://medium.com/arbitrary-execution/testing-the-limits-of-evm-stack-depth-c40ba55ca78e
[186]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/params/protocol_params.go#L79
[187]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/evm.go#L173-L176
[188]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/evm.go#L196
[189]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/evm.go#L177
[190]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/evm.go#L235-L236
[191]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/evm.go#L180
[192]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/evm.go#L194
[193]: https://eips.ethereum.org/EIPS/eip-158
[194]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/evm.go#L181
[195]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/evm.go#L214-L231
[196]: https://ethereum.github.io/yellowpaper/paper.pdf#section.8
[197]: https://etherscan.io/address/0x0000000000000000000000000000000000000001
[198]: https://etherscan.io/address/0x0000000000000000000000000000000000000009
[199]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/contracts.go#L45-L93
[200]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/instructions.go#L407-L408
[201]: https://etherscan.io/tx/0xbdba0ec52ae2c9785468a453745b9f7024af6e165887a0e9038a5c2d2fea3909
[202]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/evm.go#L219
[203]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/evm.go#L220-L221
[204]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/evm.go#L228
[205]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/interpreter.go#L141
[206]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/params/protocol_params.go#L79
[207]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/interpreter.go#L140
[208]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/interpreter.go#L150
[209]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/interpreter.go#L164
[210]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/interpreter.go#L118-L120
[211]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/interpreter.go#L122-L127
[212]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/evm.go#L228
[213]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/instructions.go#L524-L527
[214]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/instructions.go#L580-L583
[215]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/instructions.go#L626-L629
[216]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/instructions.go#L826-L829
[217]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/instructions.go#L679-L681
[218]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/instructions.go#L846-L848
[219]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/interpreter.go#L181-L244
[220]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/interpreter.go#L239
[221]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/interpreter.go#L188-L189
[222]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/jump_table.go#L94-L99
[223]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/interpreter.go#L74-L75
[224]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/interpreter.go#L191-L196
[225]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/interpreter.go#L197-L199
[226]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/interpreter.go#L221-L225
[227]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/interpreter.go#L201-L217
[228]: https://www.evm.codes/about#memoryexpansion
[229]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/instructions.go
[230]: https://soliditylang.org/
[231]: https://etherscan.io/address/0x6B175474E89094C44Da98b954EedeAC495271d0F#code
[232]: https://github.com/makerdao/dss/blob/master/src/dai.sol
[233]: https://samczsun.com/hiding-in-plain-sight/
[234]: https://sourcify.dev
[235]: https://etherscan.io/address/0x6B175474E89094C44Da98b954EedeAC495271d0F#code#L6
[236]: https://etherscan.io/address/0x6B175474E89094C44Da98b954EedeAC495271d0F#code#L122
[237]: https://etherscan.io/address/0x6B175474E89094C44Da98b954EedeAC495271d0F#code#L123
[238]: https://docs.soliditylang.org/en/v0.5.12/units-and-global-variables.html#block-and-transaction-properties
[239]: https://etherscan.io/address/0x6B175474E89094C44Da98b954EedeAC495271d0F#code#L125
[240]: https://etherscan.io/address/0x6B175474E89094C44Da98b954EedeAC495271d0F#code#L128
[241]: https://etherscan.io/address/0x6B175474E89094C44Da98b954EedeAC495271d0F#code#L90
[242]: https://etherscan.io/address/0x6B175474E89094C44Da98b954EedeAC495271d0F#code#L129
[243]: https://etherscan.io/address/0x6B175474E89094C44Da98b954EedeAC495271d0F#code#L133
[244]: https://etherscan.io/address/0x6B175474E89094C44Da98b954EedeAC495271d0F#code#L135
[245]: https://eips.ethereum.org/EIPS/eip-20#events
[246]: https://etherscan.io/address/0x6B175474E89094C44Da98b954EedeAC495271d0F#code#L136
[247]: https://eips.ethereum.org/EIPS/eip-20#methods
[248]: https://blog.openzeppelin.com/deconstructing-a-solidity-contract-part-i-introduction-832efd2d7737/
[249]: https://noxx.substack.com/p/evm-deep-dives-the-path-to-shadowy
[250]: https://library.dedaub.com/contracts/Ethereum/6b175474e89094c44da98b954eedeac495271d0f/disassembled
[251]: https://gist.github.com/tinchoabbate/508b2b51df7043a524be7debb738992c
[252]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/instructions.go#L505-L506
[253]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/instructions.go#L277-L281
[254]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/instructions.go#L294-L297
[255]: https://docs.soliditylang.org/en/v0.5.15/abi-spec.html#function-selector
[256]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/instructions.go#L283-L292
[257]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/instructions.go#L286
[258]: https://www.evm.codes/#1c
[259]: https://gist.github.com/tinchoabbate/508b2b51df7043a524be7debb738992c#file-dai-asm-L3346-L3353
[260]: https://docs.soliditylang.org/en/v0.5.15/miscellaneous.html#mappings-and-dynamic-arrays
[261]: https://gist.github.com/tinchoabbate/508b2b51df7043a524be7debb738992c#file-dai-asm-L1583-L1620
[262]: https://gist.github.com/tinchoabbate/508b2b51df7043a524be7debb738992c#file-dai-asm-L3444-L3465
[263]: https://gist.github.com/tinchoabbate/508b2b51df7043a524be7debb738992c#file-dai-asm-L1884-L1907
[264]: https://gist.github.com/tinchoabbate/508b2b51df7043a524be7debb738992c#file-dai-asm-L3467-L3487
[265]: https://gist.github.com/tinchoabbate/508b2b51df7043a524be7debb738992c#file-dai-asm-L1908-L1929
[266]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/instructions.go#L844-L869
[267]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/instructions.go#L849
[268]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/instructions.go#L850-L851
[269]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/instructions.go#L853
[270]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/instructions.go#L854
[271]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/instructions.go#L857
[272]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/instructions.go#L858-L865
[273]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/interpreter.go#L181
[274]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/instructions.go#L807
[275]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/interpreter.go#L246-L248
[276]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/evm.go#L228
[277]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/evm.go#L235-L243
[278]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/vm/evm.go#L244
[279]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_transition.go#L339-L342
[280]: https://eips.ethereum.org/EIPS/eip-3529
[281]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_transition.go#L364-L366
[282]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_transition.go#L370
[283]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_transition.go#L343-L347
[284]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_transition.go#L345
[285]: https://watchtheburn.com/
[286]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_transition.go#L349-L353
[287]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_transition.go#L181
[288]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_processor.go#L101
[289]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_processor.go#L109
[290]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_processor.go#L113
[291]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_processor.go#L118-L122
[292]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_processor.go#L123
[293]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_processor.go#L124
[294]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_processor.go#L128
[295]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_processor.go#L132
[296]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_processor.go#L133
[297]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/state_processor.go#L134-L136
[298]: https://noxx.substack.com/i/55077458/bloom-filters
[299]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L1155
[300]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/consensus/ethash/consensus.go#L595
[301]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/consensus/ethash/consensus.go#L645
[302]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/consensus/ethash/consensus.go#L645
[303]: https://ethereum.org/en/glossary/#ommer
[304]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/consensus/ethash/consensus.go#L596
[305]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/consensus/ethash/consensus.go#L606
[306]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/types/block.go#L200-L228
[307]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/types/block.go#L206
[308]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/types/block.go#L214
[309]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L1162
[310]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L648
[311]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L668
[312]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/consensus/ethash/sealer.go#L99
[313]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/consensus/ethash/sealer.go#L182
[314]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/consensus/ethash/sealer.go#L182
[315]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/consensus/ethash/sealer.go#L112
[316]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L687
[317]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L712-L732
[318]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L733-L734
[319]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L742-L743
[320]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/miner/worker.go#L745-L746
[321]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/event/event.go#L97
[322]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/handler.go#L647
[323]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/handler.go#L652
[324]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/handler.go#L653
[325]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/handler.go#L586-L590
[326]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/protocols/eth/peer.go#L291
[327]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/protocols/eth/broadcast.go#L46
[328]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/protocols/eth/peer.go#L281
[329]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/handler.go#L597
[330]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/handler.go#L562-L572
[331]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/protocols/eth/handler.go#L186
[332]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/protocols/eth/handler.go#L167-L182
[333]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/protocols/eth/handler.go#L215
[334]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/protocols/eth/handlers.go#L328
[335]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/protocols/eth/handlers.go#L329-L333
[336]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/protocols/eth/handlers.go#L334-L344
[337]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/types/block.go#L128-L143
[338]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/protocols/eth/handlers.go#L337-L340
[339]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/protocols/eth/handlers.go#L341-L344
[340]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/protocols/eth/handlers.go#L349
[341]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/protocols/eth/handlers.go#L351
[342]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/handler_eth.go#L67-L68
[343]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/handler_eth.go#L123-L124
[344]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/fetcher/block_fetcher.go#L268
[345]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/fetcher/block_fetcher.go#L420
[346]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/fetcher/block_fetcher.go#L429
[347]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/fetcher/block_fetcher.go#L796
[348]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/fetcher/block_fetcher.go#L354
[349]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/fetcher/block_fetcher.go#L376
[350]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/fetcher/block_fetcher.go#L848-L853
[351]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/fetcher/block_fetcher.go#L855
[352]: https://ethereum.github.io/yellowpaper/paper.pdf#subsection.4.3
[353]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/consensus/ethash/consensus.go#L263
[354]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/consensus/ethash/consensus.go#L310
[355]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/consensus/ethash/consensus.go#L274
[356]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/consensus/ethash/consensus.go#L270
[357]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/consensus/ethash/consensus.go#L305
[358]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/fetcher/block_fetcher.go#L859
[359]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/fetcher/block_fetcher.go#L871
[360]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/blockchain.go#L1414-L1595
[361]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/blockchain.go#L1603
[362]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/blockchain.go#L1632
[363]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/blockchain.go#L1654
[364]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/block_validator.go#L81
[365]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/core/blockchain.go#L1669-L1674
[366]: https://github.com/ethereum/go-ethereum/blob/v1.10.18/eth/fetcher/block_fetcher.go#L877