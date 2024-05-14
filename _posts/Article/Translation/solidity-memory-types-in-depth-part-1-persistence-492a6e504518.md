---
title: "Solidity Memory Types In Depth: Part 1 — Persistence."
authorURL: ""
originalURL: https://atise.medium.com/solidity-memory-types-in-depth-part-1-persistence-492a6e504518
translator: ""
reviewer: ""
---

# Solidity Memory Types In Depth: Part 1 — Persistence.

<!-- more -->

[

![Atis E](https://miro.medium.com/v2/resize:fill:88:88/1*au3fHXCTwSS6i6YmPwVI1Q.jpeg)









][1]

[Atis E][2]

·

[Follow][3]

8 min read

·

Apr 30, 2024

[

][4]

\--

[][5]

Listen

Share

_In most programming languages, you’d only need to worry about two types of memory — the heap and the stack; perhaps a few more if you want to be performance-conscious. In Solidity and EVM, there are more than two._ [_They_][6] _include: calldata, storage, transient storage, “memory”, stack, and others. Each has different semantics, costs, layout, and packing rules. This first article specifically looks in-depth at the persistence of data stored in the different types of memory, in other words, how long a data item stored in memory remains accessible._

_Target audience: programmers experienced in other languages, but intermediate or beginner in Solidity and EVM._

![](https://miro.medium.com/v2/resize:fit:640/format:webp/0*wqtRjcxMB8YiDayV)

Programs need to store and access information to operate. EVM programs are no different, but the memory access costs and patterns for blockchain applications differ greatly from those of more typical software. Unfortunately, the Solidity programming language doesn’t do an especially good job of making the differences and related tradeoffs clear.

Programmers who start learning C or C++ are typically taught about just two types of memory: the heap and the stack. In C, there’s also the `register` keyword. When we also consider the speed of execution, other aspects start to matter: for example, data that is loaded in the cache memory allows for faster access. However, the heap and the stack are the two main players. They are supported at the standard library level both in C and C++. In C++, they’re supported even at the language syntax level, since the keywords `new`and `delete` allocate memory in the heap, by default.

In contrast, the recent versions of EVM have _four_ different read-write memory types. Grouped by their persistence levels:

1\. Stack — has a continuously changing top section.  
2\. Memory — persists across internal function calls.  
3\. Transient storage — persists across external function calls.  
4\. Storage — persists across multiple transactions.

One aspect shared by all of them is the word size, which is 32 bytes (256 bits), [chosen][7] “because it is convenient for Ethereum’s core cryptographic operations”, such as its hash function. This means that most read and write operations work with 32 bytes at once. One exception is the memory, which can be read and written one byte at the time. Another is the stack, which can be

In addition, there are two major read-only memory types (or, more accurately, write-once):  
1\. Calldata — function-local.  
2\. Bytecode — persists across multiple transactions.

The latter is not strictly memory in the usual sense, but sometimes is used as such, for instance, to simulate expensive-to-write-but-cheap-to-read memory due to the constraints and costs of a blockchain — see the [SSTORE2][8] and [SSTORE3][9] libraries as examples. Not to mention even more hacky techniques, like passing information in the remaining gas counter, or using `msg.value`or the calling account’s nonce as parameters with some semantic load. Each data storage approach has distinct gas costs, persistence rules, and they also differ in more obscure aspects such as variable packing and padding rules.

## Read-write memory types

**Stack.** The stack memory, especially the top of the stack that can actually be read, has the shortest lifespan and is similar to the stack memory in traditional programming. Each external function call gets its own stack frame, and the contents of the frame change during execution, as the EVM is a stack-based virtual machine. On the other hand, internal calls all share the same stack frame. Essentially, internal calls behave as inline functions in C code.

**Memory.** The memory in EVM is a continuous chunk of data, essentially a byte array, that can be dynamically expanded — in other words, resized to become larger. It is automatically initialized to zero whenever it’s expanded. The contents and the size of the memory are saved across internal function calls, but not across external calls.

Since the EVM has no registers, Solidity uses a “scratch pad” — predefined parts of the memory effectively to emulate the functionality usually offered by registers. The [layout of the scratch pad][10] is:

-   `0x00–0x3f` (64 bytes or 2 words): scratch space for hashing methods
-   `0x40–0x5f` (32 bytes or 1 word): currently allocated memory size (aka. free memory pointer)
-   `0x60–0x7f`(32 bytes or 1 word): zero slot

Application developers should not overwrite these predefined memory areas. (Unless you code in in-line assembly, then it’s safe to overwrite Solidity’s scratchpad’s first two words for your own purposes. Yes, it is confusing, and no, enforcing that a region of memory is read-only is not possible.)

**Storage** is a [trie-based][11] memory structure that persists across the transaction boundary. In the end, if the smart contract wants to make any changes in the blockchain, it typically writes them into the storage. Using the storage is similar to using hashmaps in traditional programming languages — think of dictionaries in Python. One difference that accessing, for example, Python’s dictionary using a key that doesn’t exist will create a runtime exception, while accessing an uninitialized EVM’s storage slot is not an error, and will always return zero (or, for a Boolean value, return`false`as the value).

Another difference is that hashmaps usually have to accept some probability of key collision, and deal with that somehow. In contrast, there’s essentially no probability of EVM’s storage key collision, as long as the keys are random. The space of the possible keys is huge: 2²⁵⁶, and accidental collisions are so unlikely they can be ignored. Even a dedicated attacker would have to search through approximately _sqrt(2²⁵⁶)_ or 2¹²⁸ combinations to find two random keys that collide, which isn’t feasible. For non-random keys, collisions are of course possible — for instance, most contracts have something stored under the keys “0” and “1”.

**Transient storage** is a recently introduced memory type that fills the conceptual gap between memory and storage. Data in transient storage is saved across external calls (unlike memory), but not across different transactions (unlike storage). There’s currently no support for transient storage at the level of Solidity; a lower-level programming language must be used. For an example, see Uniswap v4 code [here][12].

In the future, other memory types may be introduced. Block-local storage seems to be a natural step beyond transient storage, that is, data that persists across multiple transactions in the same block but not across multiple blocks. We can also easily conceptualize some sort of registers, for example, introduced via extending the EVM with the so-called [precompiles][13]. For instance, there could be a read-only register that always reads as zero, instead of the memory address 0x60 that is conventionally assumed to contain a guaranteed-to-be-zero value.

## Read-only memory types

**Calldata** is used to pass read-only parameters to external function calls.

**Bytecode**, i.e. deploying a new contract with the desired contents, can store data that doesn’t have to be modified after the initial write. In this way, smart contracts can store relatively large amounts (up to ~24 kb) of persistent data. Reading bytecode is cheaper than reading storage. In particular it’s _much_ cheaper than reading cold storage, i.e. storage keys that previously have not been accessed . However, if the data stored in the bytecode need to be updated, then the whole contract has to be redeployed, which is of course relatively expensive.

Both calldata and bytecode eventually write information to the blockchain itself. In this way they’re similar to storage. Even so, there’s a difference: calldata and bytecode are saved in [different data structures in the EVM][14]. Moreover, the calldata used in one transaction isn’t accessible by subsequent transactions. After the function call is over, the calldata effectively becomes invisible from the blockchain’s internal point of view. It is stored in the blockchain, but usable only for external consumption. No wonder solutions like [coprocessors][15] are starting to pop up. Coprocessors allow smart contracts to “see” the historical data stored on the blockchain, in a trustless, verifiable way.

For completeness, it should be mentioned that the most recent version of EVM incorporates another memory type: the **blobspace**. It is defined in [EIP-4844][16] and allows for the creation of data that must be published by the blockchain nodes but not permanently stored by them.As of version 0.8.25, Solidity doesn’t yet support blob-related functionality.

## Solidity calling conventions and example code

Solidity calls can be **internal** (information is passed on the stack) or **external** (information is passed via the EVM’s standard smart contract ABI).

Solidity distinguishes between [value types and reference types][17].

Value types are always passed by value, meaning that they are always copied when used as function arguments or in assignments. Reference types are passed by their address, and a copy is made only if necessary (for instance, if type conversion is needed). Value types typically fit in a single word, and examples include: `int, uint,`and `bytes32`. Reference type examples include `struct, array,`and`bytes`.

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*tDtzM8Zgk4BoSNGuLF2gog.png)

Valid combinations of argument storage types and function call types

For reference types, the programmer must explicitly state the memory location the memory location. For internal calls, either `calldata`, `memory`, or `storage` can be used; for external calls: only `calldata` or `memory`. If the `storage`type is used for an argument, then it is passed as a pointer, and does not require that a copy of the object is made.

In some cases, Solidity allows implicit conversion between argument types, for example:  
— variables of `calldata` type can be passed as `memory` arguments  
— variables of `storage` type also can be passed as `memory` arguments

Either of these create an in-memory copy of the actual argument. This costs gas and is inefficient, especially for storage-to-memory conversions. Most of the other implicit conversions are not allowed by Solidity.

The listing below shows a Solidity code example demonstrating the different persistence levels of the different memory types. It defines two functions:

-   `internalFunction` — for this, Solidity allows either `memory`, `calldata`, or `storage` argument types. From these `calldata`is allowed only for arguments forwarded from external calls.
-   `publicFunction` — for this, Solidity allows either `memory` or `calldata` argument types. The compiler does not allow use of `storage`.

The first function can be called only internally. The second can be called both internally and externally.

-   If a function is called internally and the argument type is `memory`, the changes in the argument ARE visible to the calling function.
-   If a function is called externally and the argument type is `memory`, the changes in the argument ARE NOT visible to the calling function.

Solidity code example that shows the potential combinations of argument storage types, function call types, and their impact on the changes visible in the calling function:

# Common mistakes

A few mistakes related to persistence of function arguments:

1.  Calling a function with `memory`argument type and expecting it to make changes in the storage. For example, this contract’s `test()`function does not change the `_inStorage` structure:

2\. Calling a function externally and expecting the changes in memory to be visible to the caller. In the example below, the first call to `publicFunction`changes the structure `inMemory`, but the second call does not, and also requires additional gas because it creates a copy of the structure.

# Conclusion

This article listed the different EVM memory types and discussed their persistence semantics.

Other aspects of how the memory types differ will be covered in future articles:

-   allocation and usage costs
-   packing and padding
-   size limits and usage measurements

_Thanks to_ [_@danielvf_][18]_, Inga, and_ [_@0xsamgreen_][19] _for reviews._

[1]: /?source=post_page-----492a6e504518--------------------------------
[2]: /?source=post_page-----492a6e504518--------------------------------
[3]: https://medium.com/m/signin?actionUrl=https%3A%2F%2Fmedium.com%2F_%2Fsubscribe%2Fuser%2Fe150b6e52ea5&operation=register&redirect=https%3A%2F%2Fatise.medium.com%2Fsolidity-memory-types-in-depth-part-1-persistence-492a6e504518&user=Atis+E&userId=e150b6e52ea5&source=post_page-e150b6e52ea5----492a6e504518---------------------post_header-----------
[4]: https://medium.com/m/signin?actionUrl=https%3A%2F%2Fmedium.com%2F_%2Fvote%2Fp%2F492a6e504518&operation=register&redirect=https%3A%2F%2Fatise.medium.com%2Fsolidity-memory-types-in-depth-part-1-persistence-492a6e504518&user=Atis+E&userId=e150b6e52ea5&source=-----492a6e504518---------------------clap_footer-----------
[5]: https://medium.com/m/signin?actionUrl=https%3A%2F%2Fmedium.com%2F_%2Fbookmark%2Fp%2F492a6e504518&operation=register&redirect=https%3A%2F%2Fatise.medium.com%2Fsolidity-memory-types-in-depth-part-1-persistence-492a6e504518&source=-----492a6e504518---------------------bookmark_footer-----------
[6]: https://docs.soliditylang.org/en/v0.8.25/introduction-to-smart-contracts.html#storage-memory-and-the-stack
[7]: https://ethereum.org/en/developers/tutorials/yellow-paper-evm/#91-basics
[8]: https://github.com/Vectorized/solady/blob/main/src/utils/SSTORE2.sol
[9]: https://github.com/Philogy/sstore3
[10]: https://docs.soliditylang.org/en/v0.8.25/internals/layout_in_memory.html
[11]: https://en.wikipedia.org/wiki/Trie
[12]: https://github.com/Uniswap/v4-core/blob/main/src/libraries/Lock.sol
[13]: https://www.evm.codes/precompiled
[14]: https://medium.com/@eiki1212/ethereum-state-trie-architecture-explained-a30237009d4e
[15]: https://docs.brevis.network/
[16]: https://medium.com/@encodinglabs/eip-eip-4844-shard-blob-transactions-2f0325130d3a
[17]: https://docs.soliditylang.org/en/v0.8.25/types.html#
[18]: http://twitter.com/danielvf
[19]: http://twitter.com/0xsamgreen