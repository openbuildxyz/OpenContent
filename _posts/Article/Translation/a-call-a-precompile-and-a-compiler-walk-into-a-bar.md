---
title: A call, a precompile and a compiler walk into a bar
authorURL: ""
originalURL: https://blog.theredguild.org/a-call-a-precompile-and-a-compiler-walk-into-a-bar/
translator: ""
reviewer: ""
---

![A call, a precompile and a compiler walk into a bar](/content/images/size/w500/2024/02/photo-1568644396922-5c3bfae12521.jpeg)

<!-- more -->

Photo by [Drew Beamer][1] / [Unsplash][2]

March 15, 2024 by [tincho][3] — 8 min read

# A call, a precompile and a compiler walk into a bar

After 5 years of using Solidity, I thought I knew how calls worked. I didn't.

After 5 years of reading and writing Solidity, I thought I knew how external calls worked.

After 5 years of reading and writing Solidity, I should have known better.

It all started when I faced an impossible piece of Solidity code in a new L2.

It just couldn't work. If all I knew about Solidity was right, then the contract JUST. COULDN'T. WORK. Plain and simple. All of it was so obviously broken. Yet somehow, it wasn't.

Tests were green. A dummy testnet had been running for weeks. The system had undergone many security reviews. Shouldn't such a broken code have been reported and fixed already? Even another more popular L2 was using similar code.

Everywhere I looked contradicted what I knew about Solidity external calls. Could I be so wrong?

My debugging skills failed me. There were so many moving pieces. If you've ever tried to debug a transaction hitting a predeploy calling a custom precompile that ABI-decodes stuff in a custom version of geth of a L2 that forked another L2's code, you feel me.

Disbelief evolved into hopelessness. The temptation of blind faith intensified. But I wouldn't give in! Luckily, because I was only hours away from sudden enlightenment, understanding and relief.

Some people find revealing truth in a religious book. Others skimming self-help books in airport lounges. I found it in line 2718 of a C++ file.

## Check then call

The high-level Solidity syntax for an external call can look like this:

```Solidity
pragma solidity ^0.8.0;
interface ISomeInterface {
  function foo() external;
}

contract Example {
  function callAccount(address account) external {
    ISomeInterface(account).foo();
  }
}
```

Example contract with an external call

If you compile that, after the always useful warning for not having included a license identifier, you'll find this bytecode:

```
...
CALL
...
```

Almost complete EVM bytecode after compiling with solc 0.8.15

Unsurprisingly, the compiler translates the Solidity high-level call to the `CALL` opcode. Oversimplifying you say? Ok, let's dig a bit deeper.

Those who've dealt with Solidity longer than one DeFi summer know that the compiler includes safety checks.

Before a `CALL`, the compiler puts bytecode to validate that the call's target has code. It places an `EXTCODESIZE`, including the necessary logic to reach a `REVERT` before the `CALL` in case the `EXTCODESIZE` of the target is 0.

```
EXTCODESIZE
...
REVERT
...
CALL
```

More accurate bytecode after compiling with solc 0.8.15

But even a developer that arrived late to the DeFi summer 2021 and has been coding Solidity ever since, just to be ready for the next bull market, knows that. They may have seen it in the bytecode, or, the more mentally sane, may have read it [in Solidity's docs][4]:

> Due to the fact that the EVM considers a call to a non-existing contract to always succeed, Solidity includes an extra check using the extcodesize opcode when performing external calls. This ensures that the contract that is about to be called either actually exists (it contains code) or an exception is raised.

I was convinced of the above. So much that when I first saw something like this code, I found it hard to believe:

```Solidity
pragma solidity ^0.8.0;
interface IPrecompile {
  function foo() external returns (uint256);
  function bar() external;
}

contract Example {
  // Function to execute a custom precompile
  function doSomething() external {
    // [...]
    IPrecompile(customPrecompileAddress).foo();
  }
}
```

Before diving into it, let's make sure we're on the same page.

## On precompiles

Precompiles are EVM accounts that don't have bytecode stored, but can execute code. Their executable code is stored in the nodes' themselves. Usually you'll find them [in the lowest range of possible addresses][5].

To execute a precompile, you call the address where it lives. For example, `ecrecover` is one precompile of the EVM at address `0x00...01`.

Let's see its code:

```bash
cast code 0x0000000000000000000000000000000000000001
0x
```

Told you, it doesn't have EVM bytecode. Its actual [code is in the node][6].

While Ethereum has its own precompiles, there's nothing preventing L2s from including new ones into their nodes. It can be a powerful way to supercharge the EVM's functionality.

## Calling precompiles from Solidity

Precompiles don't have EVM bytecode. And I thought Solidity wouldn't allow high-level calls to an account with no bytecode. It'd revert before reaching the call.

So, to call a precompile, I'd use Solidity low-level calls. The ones that operate on addresses instead of contract instances. These kind of calls don't include the `EXTCODESIZE` check, as [the docs][7] explain.

For example, to call a precompile at 0x04:

```Solidity
// Call precompile at address 0x04
(, bytes memory returndata) = address(4).call(somedata)
```

The standard EVM precompiles are rather straightforward, so it's simple to call them that way. You send some raw bytes of data, they perform some calculation, and return a raw set of bytes with the results.

Solc does have built-in functions to call some (but not all) precompiles, like `ecrecover`. Just to spare you from writing low-level calls. But that's not relevant here.

Precompiles of L2s can be more complex than the "standard" ones in the EVM. They may include different _functions_ within a single precompile. For instance, there could be a precompile that implemented the interface we saw earlier:

```Solidity
interface IPrecompile {
  function foo() external returns (uint256);
  function bar() external;
}
```

So, assuming the precompile can somehow handle it (we'll see an example later), you might call its `foo` function with something along these lines:

```Solidity
(, bytes memory returndata) = address(customPrecompileAddress).call(abi.encodeWithSelector(IPrecompile.foo.selector));
uint256 result = abi.decode(returndata, (uint256));
```

But not with a high-level call like this:

```Solidity
uint256 result = IPrecompile(precompileAddress).foo();
```

That would fail. I'm telling you. The documentation I read says so, we saw the `EXTCODESIZE` check earlier.

C'mon please, don't insist, it won't work.

Nah, just kidding. The high-level call works too. To understand why, let's first create a custom precompile, then do some tests, and finally inspect how solc _really_ works under the hood.

## Adding a new precompile

Let's start by creating a custom precompile in the `core/vm/contracts.go` file of [go-ethereum][8].

💡

There're smarter ways to add a complex set of custom precompiles to the EVM. For a more realistic example, study [how ArbOS does it][9].

The precompile I'll create checks the input bytes against the function selectors for `foo` and `bar`. When the selector for `foo` matches, it returns the number 43. When the selector for `bar` matches, it returns nothing.

```Go
type myPrecompile struct{}

func (p *myPrecompile) RequiredGas(_ []byte) uint64 {
	return 0
}

func (p *myPrecompile) Run(input []byte) ([]byte, error) {
	if len(input) < 4 {
		return nil, errors.New("short input")
	}

	if input[0] == 0xC2 && input[1] == 0x98 && input[2] == 0x55 && input[3] == 0x78 { // function selector of `foo()`
		return common.LeftPadBytes([]byte{43}, 32), nil
	} else if input[0] == 0xFE && input[1] == 0xBB && input[2] == 0x0F && input[3] == 0x7E { // function selector of `bar()
		return nil, nil
	} else {
		return nil, errors.New("bad input")
	}
}
```

The precompile will be at the `0x0b` address:

```Go
var PrecompiledContractsCancun = map[common.Address]PrecompiledContract{
  // [...]
  common.BytesToAddress([]byte{0x0b}): &myPrecompile{},
}
```

Then build go-ethereum (`make geth`) and run it in dev mode (`./build/bin/geth --dev --http`).

Validate the precompile is alive with [cast][10]:

```bash
cast call 0x000000000000000000000000000000000000000b "foo()"
0x000000000000000000000000000000000000000000000000000000000000002b

cast call 0x000000000000000000000000000000000000000b "bar()"
0x

cast call 0x000000000000000000000000000000000000000b
Error: 
(code: -32000, message: short input, data: None)

cast call 0x000000000000000000000000000000000000000b "somefunction()"
Error: 
(code: -32000, message: bad input, data: None)
```

Quick tests calling the new precompile from cast

All set! Let's move on to Solidity now.

## Calling the custom precompile

Time to call the `foo` function of the new precompile I created at address `0x0b`.

I'll use a high-level call. According to what I knew, this should NOT work. It should revert before triggering the call, because the `EXTCODESIZE` check included by the compiler will return 0 for the `0x0b` address, and therefore reach a `REVERT` in the bytecode.

```Solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.15;

interface IPrecompile {
    function foo() external returns (uint256);
    function bar() external;
}

contract PrecompileCaller {
    function callFoo() external {
        // This call to `foo` should revert
        uint256 result = IPrecompile(address(0x0b)).foo();
        
        require(result == 43, "Unexpected result");
    }
}
```

Example contract to test high-level call to a precompile

Here's a simple [Hardhat][11] test to execute it:

```Javascript
describe("PrecompileCaller", function () {

  let precompileCaller;

  before(async function () {
    const PrecompileCallerFactory = await ethers.getContractFactory("PrecompileCaller");
    precompileCaller = await PrecompileCallerFactory.deploy();
  });
  
  it("Calls foo", async function () {
    await precompileCaller.callFoo();
  });
});
```

```
$ yarn hardhat test --network localhost

  PrecompileCaller
    ✔ Calls foo

  1 passing (224ms)
```

Wat? That shouldn't have worked 🤔

Let's see. If calling `foo` works, then calling `bar` should also work. I'll add some code in the contract to call the precompile's `bar` function.

```Solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.15;

interface IPrecompile {
    function foo() external returns (uint256);
    function bar() external;
}

contract PrecompileCaller {
    // Somehow this works
    function callFoo() external {
        uint256 result = IPrecompile(address(0x0b)).foo();
        require(result == 43, "Unexpected result");
    }

    // If calling `foo` works, this should also work
    function callBar() external {
        IPrecompile(address(0x0b)).bar();
    }
}
```

The extended Hardhat test now looks like:

```Javascript
const { expect } = require("chai");

describe("PrecompileCaller", function () {

  let precompileCaller;

  before(async function () {
    const PrecompileCallerFactory = await ethers.getContractFactory("PrecompileCaller");
    precompileCaller = await PrecompileCallerFactory.deploy();
  });
  
  it("Calls foo", async function () {
    // This works (doesn't revert)
    await precompileCaller.callFoo();
  });

  it("Calls bar", async function () {
    // This should also work. Does it?
    await precompileCaller.callBar();
  });
});
```

```
$ yarn hardhat test --network localhost

  PrecompileCaller
    ✔ Calls foo
    1) Calls bar


  1 passing (252ms)
  1 failing

  1) PrecompileCaller
       Calls bar:
     ProviderError: execution reverted
```

Crap.

## I don't know how calls work

See? I told you. I didn't know how calls work. After all these years. Here's the Solidity code again:

```Solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.15;

interface IPrecompile {
    function foo() external returns (uint256);
    function bar() external;
}

contract PrecompileCaller {
    // Somehow this works
    function callFoo() external {
        uint256 result = IPrecompile(address(0x0b)).foo();
        require(result == 43, "Unexpected result");
    }

    // Somehow this doesn't work
    function callBar() external {
        IPrecompile(address(0x0b)).bar();
    }
}
```

We're in easy mode here. There's one clear difference between the two functions in this example. The real case was more difficult and I couldn't see it so clearly.

Yes, the difference is in the returns. Could a declared return value have something to do with all this?

## No checks if return

This is how I learned that Solidity doesn't _always_ include the `EXTCODESIZE` check in high-level calls.

Let's analyse the Yul code produced for the functions `callFoo` and `callBar` of the `PrecompileCaller` contract of the example.

For `callFoo`:

```Solidity
function fun_callFoo_32() {

  // ...

  let _3 := call(gas(), expr_21_address,  0,  _1, sub(_2, _1), _1, 32)
```

For `callBar`:

```Solidity
function fun_callBar_45() {
  // ...

  if iszero(extcodesize(expr_41_address)) { revert_error_0cc013b6b3b6beabea4e3a74a6d380f0df81852ca99887912475e1f66b2a2c20() }

  // ...

 let _8 := call(gas(), expr_41_address,  0,  _6, sub(_7, _6), _6, 0)
```

In `callFoo`, the compiler didn't include an `EXTCODESIZE` check before the call. Opposite to what it did in `callBar`. Why would it do that?

The answer lies buried in line [2718 and 2719 of this C++ file][12]. Lo and behold:

> We do not need to check extcodesize if we expect return data, since if there is no code, the call will return empty data and the ABI decoder will revert.

What does this mean?

Remember the interface I used in Solidity:

```Solidity
interface IPrecompile {
    function foo() external returns (uint256);
    function bar() external;
}
```

Based on this definition, the compiler expects `foo` to return something (a `uint256`). Because of that, it won't place an `EXTCODESIZE` check prior to calling it!

Solc assumes that if the target has no code, in practice there won't be return data anyway, and therefore attempting to ABI-decode no return data to the declared return type (the `uint256`) will inevitably fail. So it might as well skip the code size check prior to the call.

Adding to my confusion, the compiler didn't always behave this way. Skipping the code size check for external calls when return data is expected was [introduced in 0.8.10][13]. That means +2 years ago. I guess I was late to find out?

Even after writing this article, I thought the documentation was incomplete and outdated. Well, it kind of isn't. My dear [matta][14] found that this special behavior is documented [in another section][15], which I hadn't read 🤦

There's room to improve that documentation. So we proposed [a small PR][16] to make them clearer and more consistent.

I wish I could say that I now know how Solidity calls work. But there might be a new surprise waiting for me around the corner.

![](https://blog.theredguild.org/content/images/2023/11/file-Luuf7zu3dIoPwrwlME2PVISM-1-1-1.png)

## Want more stories? Subscribe to the blog!

It's cool. It's free. And we don't spam.Too lazy to do that.

[i'm a subscriboooooor][17]

[][18][][19][][20]The link has been copied!

[1]: https://unsplash.com/@dbeamer_jpg?utm_source=ghost&utm_medium=referral&utm_campaign=api-credit
[2]: https://unsplash.com/?utm_source=ghost&utm_medium=referral&utm_campaign=api-credit
[3]: /author/tincho/
[4]: https://docs.soliditylang.org/en/v0.8.24/units-and-global-variables.html#members-of-address-types
[5]: https://github.com/ethereum/go-ethereum/blob/v1.13.14/core/vm/contracts.go#L84-L92
[6]: https://github.com/ethereum/go-ethereum/blob/v1.13.14/core/vm/contracts.go#L188-L217
[7]: https://docs.soliditylang.org/en/v0.8.24/units-and-global-variables.html#members-of-address-types
[8]: https://github.com/ethereum/go-ethereum/
[9]: https://docs.arbitrum.io/arbos/#precompiles
[10]: https://book.getfoundry.sh/cast/
[11]: https://hardhat.org/
[12]: https://github.com/ethereum/solidity/blob/v0.8.15/libsolidity/codegen/ExpressionCompiler.cpp#L2718-L2719
[13]: https://github.com/ethereum/solidity/commit/a1aa9d2d90f2f7e7390408e9005d62c7159d4bd4
[14]: https://twitter.com/mattaereal
[15]: https://docs.soliditylang.org/en/v0.8.24/control-structures.html#external-function-calls
[16]: https://github.com/ethereum/solidity/pull/14931
[17]: #/portal/signup/
[18]: https://twitter.com/intent/tweet?text=A%20call%2C%20a%20precompile%20and%20a%20compiler%20walk%20into%20a%20bar&url=https://blog.theredguild.org/a-call-a-precompile-and-a-compiler-walk-into-a-bar/
[19]: https://www.facebook.com/sharer/sharer.php?u=https://blog.theredguild.org/a-call-a-precompile-and-a-compiler-walk-into-a-bar/
[20]: javascript: