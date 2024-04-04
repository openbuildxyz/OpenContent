---
title: A call, a precompile and a compiler walk into a bar（调用，预编译和编译器到底是怎么工作的）
author URL: ""
original URL: https://blog.theredguild.org/a-call-a-precompile-and-a-compiler-walk-into-a-bar/
translator: "张云帆"
reviewer: ""
---

![A call, a precompile and a compiler walk into a bar](/content/images/size/w500/2024/02/photo-1568644396922-5c3bfae12521.jpeg)

<!-- more -->

照片由 [Drew Beamer][1] / [Unsplash][2] 提供

写于2024年3月15日  作者 [tincho][3] — 阅读时间8分钟

# 函数调用（call），预编译（precompile）和编译器（compiler）到底是怎么工作的

用Solidity写了5年程序，我以为我知道调用（calls）是如何工作的。但这一切在我遇到一段L2中的不可能的Solidity代码时发生了改变。

我遇到了一段代码它应该是不能运行的。如果我对Solidity的了解都是正确的，那么我遇到的这个合约就不应该正常运行。但不知什么原因，事实并非如此。

测试显示没有错误。 测试网已经运行了好几周。这个系统经过了多次安全审查。这样一个损坏的代码不应该已经被报告并修复吗？ 甚至另一个更流行的 L2 也使用类似的代码。

我所看到的一切都与我对Solidity外部调用的了解相矛盾。我会错得这么离谱吗？

我的debug技巧让我失败了。这里面有很多令人感动的事情。如果你曾经尝试调试一个交易，这个交易使用预部署，调用自定义预编译，这个预编译对L2的自定义版本的geth中的内容进行 ABI解码，而该版本派生了另一个L2的代码，你就会明白我的感受。

怀疑演变成绝望。盲目信仰的诱惑愈演愈烈。但我不会屈服！幸运的是，我只用了几个小时就完成了突然的启发、理解到解脱的过程。

有些人在宗教书籍中发现了揭示真相的真理。 有些人则在机场休息室浏览自助书籍。而我在 C++文件的第2718行找到了它。


## 先检查 再调用

外部调用（external call）的Solidity语法如下所示：

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

使用外部调用的示例合约

如果你编译这个合约，会弹出没有包含许可证标识符（license identifier）的警告，你会看到这个字节码

```
...
CALL
...
```

用solc编译后的EVM字节码0.8.15

不出所料，编译器将Solidity高级调用转换为`CALL`操作码。你觉得过于简单？好吧，让我们深入一点。

那些处理Solidity超过一个去中心化金融夏天（defi summer）的人知道编译器包括安全检查。

在`CALL`之前，编译器放置字节码来验证调用的目标是否有代码。它放置了一个`EXTCODESIZE`，包括在`CALL`之前到达`REVERT`的必要逻辑，以防目标的`EXTCODESIZE`为0。

```
EXTCODESIZE
...
REVERT
...
CALL
```

使用solc编译后更准确的字节码0.8.15

但是，即使是一个在2021年夏天去中心化金融后半段开始并从那以后一直在编写Solidity，为下一个牛市做好准备的开发人员也知道这一点。他们可能已经在字节码中看到了它，或者，更准确地说，可能已经在Solidity文档[4]中发现了它：

>由于EVM认为对不存在的合约的调用总是成功的，Solidity在执行外部调用时使用`extcodesize`操作码进行额外检查。这确保了即将被调用的合约要么实际存在（它包含代码），要么引发异常。

我对上述内容深信不疑。以至于当我第一次看到这样的代码时，我很难相信：  

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
在深入研究之前，我们先熟悉一下一些概念。

## 预编译

预编译是没有存储字节码但可以执行代码的EVM帐户。它们的执行代码存储在节点本身中。通常你会发现它们[在可能地址的最低范围内][5]。

要执行预编译，您需要调用它所在的地址。例如，`ecRecovery`是地址为“0x00…01”的EVM的一个预编译。

让我们看看它的代码：

```bash
cast code 0x0000000000000000000000000000000000000001
0x
```

它没有EVM字节码。它的实际代码[在节点中][6]。

虽然以太坊有自己的预编译，但没有什么可以阻止L2将新编译包含到其节点中。这可能是增强EVM功能的强大方式。

## 从Solidity调用预编译

预编译没有EVM字节码。我认为Solidity不允许对没有字节码的帐户进行高级调用。它会在调用之前恢复。

因此，要调用预编译，我会使用Solidity低级调用（对地址而不是合约实例进行操作的调用）。正如[文档][7]所解释的那样，这种调用不包括`EXTCODESIZE`。

例如，要在0x04调用预编译：

```Solidity
// Call precompile at address 0x04
(, bytes memory returndata) = address(4).call(somedata)
```

标准的EVM预编译非常简单，因此用这种方式调用它们也很简单。你发送一些原始数据字节，它们执行一些计算，并返回一组带有结果的原始字节。

Solc确实有内置函数来调用一些（但不是全部）预编译，例如`ecRecovery`。只是为了让你不用编写低级调用。但这在这里是无关紧要的。

L2的预编译可能比EVM中的“标准”编译更复杂。它们可能在单个预编译中包含不同的`_functions_`。例如，可能有一个预编译实现了我们之前看到的接口：

```Solidity
interface IPrecompile {
  function foo() external returns (uint256);
  function bar() external;
}
```

因此，假设预编译可以以某种方式处理它（我们稍后会看到一个示例），你可以使用以下内容调用它的`foo`函数：

```Solidity
(, bytes memory returndata) = address(customPrecompileAddress).call(abi.encodeWithSelector(IPrecompile.foo.selector));
uint256 result = abi.decode(returndata, (uint256));
```

但不是像这样的调用

```Solidity
uint256 result = IPrecompile(precompileAddress).foo();
```

那会失败的。我告诉你。我读到的文档是这么说的，我们之前看到了`EXTCODESIZE`检查。

不要坚持了，这是行不通的。

哈哈，我只是开个玩笑。高级调用也有效。为了理解背后的原因，首先我们需要创建一个自定义预编译，然后做一些测试，最后检查solc是如何在后台工作的。

## 添加新的预编译

让我们首先在[go-ethereum][8]的“core/vm/contracts.go”文件中创建一个自定义预编译。
💡
有更聪明的方法可以将一组复杂的自定义预编译添加到EVM。这是一个更实际的例子，研究[ArbOS是如何做到的][9]。

我将创建的预编译根据`foo`和`bar`的函数选择器检查输入字节。当`foo`的选择器匹配时，它返回数字43。当`bar`的选择器匹配时，它不返回任何内容。

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

预编译会在'0x0b'地址：

```Go
var PrecompiledContractsCancun = map[common.Address]PrecompiledContract{
  // [...]
  common.BytesToAddress([]byte{0x0b}): &myPrecompile{},
}
```

然后构建go-ethereum（'make geth'）并在开发模式下运行它（'./build/bin/geth--dev--http'）。

使用[cast][10]验证预编译是否有效：  


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

快速测试从cast调用新的预编译

都准备好了！现在让我们转向Solidity。

## 调用自定义预编译

是时候调用我在地址“0x0b”新创建的预编译`foo`函数了。

我将使用一个高级调用。据我所知，这应该不起作用。它应该在触发调用之前恢复，因为编译器包含的`EXTCODESIZE`检查将为“0x0b”地址返回0，因此在字节码中到达`REVERT`。

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

测试对预编译的高级调用的示例合约

这是一个简单的[Hardhat][11]测试来执行它：

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

怎么回事？ 这应该是不能运行的 🤔

让我们看看。如果调用`foo`有效，那么调用`bar`也应该有效。我将在合约中添加一些代码来调用预编译的`bar`函数。

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

扩展的Hardhat测试现在如下所示：

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

糟糕。

## 我不知道调用是如何工作的

看到了吗？我告诉过你。在写了那么多年代码后，我不知道调用是如何工作的。这是Solidity代码：

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

我们现在是处于简单模式。在这个例子中，这两个函数有一个明显的区别。真实的案例更难，我不太能理解。

这里的区别在于返回值（returns）。声明的返回值可能与这一切有关吗？

## 如果返回则不检查

我了解到Solidity不总是在高级调用中包含`EXTCODESIZE`检查。

让我们分析一下“PrecompileCaller”合约的函数`callFoo`和`callBar`生成的Yul代码。

对于`callFoo`：

```Solidity
function fun_callFoo_32() {

  // ...

  let _3 := call(gas(), expr_21_address,  0,  _1, sub(_2, _1), _1, 32)
```

对于 `callBar`:

```Solidity
function fun_callBar_45() {
  // ...

  if iszero(extcodesize(expr_41_address)) { revert_error_0cc013b6b3b6beabea4e3a74a6d380f0df81852ca99887912475e1f66b2a2c20() }

  // ...

 let _8 := call(gas(), expr_41_address,  0,  _6, sub(_7, _6), _6, 0)
```

在`callFoo`中，编译器在调用前没有包含`EXTCODESIZE`检查。与它在`callBar`中所做的相反。它为什么要这样做？

答案隐藏在C++文件的第[2718和2719行][12]中。

>如果我们期望返回数据，我们不需要检查extcodesize，因为如果没有代码，调用将返回空数据并且ABI解码器将恢复。

这是什么意思？

还记得我在Solidity中使用的`interface`吗：

```Solidity
interface IPrecompile {
    function foo() external returns (uint256);
    function bar() external;
}
```

根据这个定义，编译器期望`foo`返回一些东西（“uint256”）。 因此，它不会在调用之前进行`EXTCODESIZE`检查！

Solc假设目标没有代码，实际上无论如何都不会返回数据，因此将无返回数据的 ABI 解码作为返回类型（“uint256”）将会失败。 因此，它可能会在调用之前跳过代码大小检查。

更让我困惑的是，编译器并不总是这样。 当需要返回数据时，跳过外部调用的代码大小检查[在 0.8.10 中引入的][13]。 这意味着这至少是在2年前。 我想我发现得太晚了？

即使在写完这篇文章后，我仍然认为文档不完整且过时。但事实并非如此。我亲爱的[matta][14]发现这种特殊行为[在另一节][15]中有记录，但我没有读过🤦

该文档还有改进的空间。 所以我们提出了[一个小PR][16]，让它们更清晰、更一致。

我希望我现在可以说我知道Solidity调用是如何工作的了。但也许转角处会有新的惊喜在等着我。

![](https://blog.theredguild.org/content/images/2023/11/file-Luuf7zu3dIoPwrwlME2PVISM-1-1-1.png)

## 想要更多故事？订阅博客！

没关系，免费的。我们也不发垃圾邮件。我懒得发垃圾邮件。

[i'm a subscriboooooor][17]

[][18][][19][][20]链接已复制！

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
