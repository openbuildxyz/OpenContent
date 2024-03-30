
想要比较深入的理解以太坊虚拟机是具体如何具体操作数据和指令的就一定要去学习以太坊的opcode是如何工作的。对于虚拟机而言，opcode种类有很多，但最核心的opcode一种是语言特色相关，另外一种是数据处理和函数相关的。其他的比如Block系列指令或者JMP系列指令以及堆栈数据操作指令等等，虽然都是虚拟机功能重要的支撑部分，但是对于深入理解虚拟机是如何工作的，其实帮助没有很大。

Yul语言是一种低级语言，其主要作用就是直接调用各种以太坊底层的opcode，通常情况下通过`assembly`方式嵌入到Solidity程序中，具体代码展示类似这样：

```js
contract CalledContract {
    uint256 public number;

    function setNumber(uint256 num) external {
        assembly {
            sstore(0, num)
        }
    }

    function getNumber() public view returns (uint256) {
        assembly {
            mstore(0x00, sload(0))
            return(0x00, 0x20)
        }
    }
}
```

可以看到通过Yul编程，可以只关心EVM的核心数据处理和函数相关的opcode，而不需要用户再关心JMP或者PUSH/POP等之类的辅助基本opcode。Yul语言是一个非常好的帮助深入学习EVM的工具。该语言没有什么特别的语法，所使用的opcode函数也全部都是以太坊的opcode，所以学习和上手是比较容易的，关于该语言详细介绍可以看Solidity官方文档[Yul](https://docs.soliditylang.org/en/latest/yul.html#yul)语言的部分。接下来本文就会使用Yul语言对以太坊虚拟机做一个比较深入解读。

## EVM简述

以太坊虚拟机是一种堆栈结构的虚拟机，虚拟机常见的结构设计除了堆栈方式之外还有一种是寄存器的方式（常用于硬件）。本文接下来所指的虚拟机全部都是堆栈方式的。EVM大致结构如下图所示。

<img width="700" alt="Pasted image 20240226154231" src="https://github.com/Einstellung/search/assets/26652483/ca35c244-c6db-467b-9c60-e0eece3373f2">

在实际编程中，编译器会将我们编写好的代码按照约定规则（opcode和对应的二进制代码提前约定好）将Solidity代码转换成字节码。然后EVM会解读字节码（对应关系提前约定）并将其翻译成实际的计算机可执行程序然后开始具体执行代码。下图是opcode和对应的类型别名约定关系示例：

<img width="477" alt="Pasted image 20240226161014" src="https://github.com/Einstellung/search/assets/26652483/e953c362-ccd9-492d-abb5-4a83abbae460" style="display: block; margin: auto;">

实际的编译好的二进制代码不只是包含字节码，还包含数据类型参数类型等等很多其他信息，要复杂的多。下图是webassembly编译之后的字节码和原始代码对比，可以看到还包含很多其他信息，不过不是本文的重点，就此略过。

<img width="700" alt="Pasted image 20240226161632" src="https://github.com/Einstellung/search/assets/26652483/424dc537-354c-474a-afb6-90c534087d7e">


当EVM解读完字节码之后就可以开始执行程序了。EVM会将解码之后的指令逐个执行。一个指令由opcode和operand组成。有的opcode没有operand（如之前图中`STOP`，或者如`ADD`需要两个operand）。一段指令可能会是这个样子：

```
PUSH1 0x07
ADD
SWAP1
PUSH1 0x07
ADD
```

可以看到上述指令涉及到不少数据处理操作比如`PUSH`和`SWAP1`， 这些数据处理都需要在堆栈中完成。而具体EVM要执行哪个指令则由point counter来控制（不只是顺序执行，也可以由`JUMP`之类的指令更改位置）。Memory中会存放全局变量或者函数的局部变量。Storage是一个比较特别的数据结构，它比较类似于一种key-value数据库，该部分会在后文详细分析。

<img width="700" alt="Pasted image 20240226163401" src="https://github.com/Einstellung/search/assets/26652483/ffb851b1-19c6-4552-ac39-ffa9d4e44cd6">

## Memory

因为Yul的设计已经帮我们内置处理了operations和Stack部分的操作了（这也是直接写汇编最麻烦最容易出错的地方），所以我们不需要关心这部分，直接从Memory入手即可。

Memory是一个以32 bytes增长（或者计算）的一块很大线性存储单元，你可以在你需要的位置存储临时数据。操作Memory对应的opcode有两个：

```
MLOAD(offset)

MSTORE(offset, value)
```

其中`offset`是相对于起始地址的偏移地址，Memory的起始地址是`0x00`。我们在实际编程的时候尽可能重复利用已用过的内存地址然后去覆盖原有的数据，这是EVM的gas设计机制决定的。该机制计算gas开销是根据新分配内存地址累加的，就是说如果我们只是使用内存的前面32 bytes的话（后面使用不停的覆盖原来的内存数据），那么内存开销只计算一次，而如果我们不停在后面使用64 bytes， 96 bytes的内存数据，那么每次使用新的内存地址都要累加一次gas开销。

接下来看一个Memory使用的例子：

```js
contract Hash {
    function hash(uint256 a, uint256 b) public pure returns (bytes32) {
        assembly {
            mstore(0x00, a)
            mstore(0x20, b)
            mstore(0x00, keccak256(0x00, 0x40))
            return(0x00, 0x20)
        }
    }

    /// @notice Due to Yul structure, ABI.encode is preferred, but encodePacked isn't.
    // this function is equivalent of the Solidity `abi.encode` operation and hash the result.
    function hashABIEncode(string memory s) public pure returns (bytes32) {
        assembly {
            mstore(0x00, 0x20)
            mstore(0x20, mload(s))
            mstore(0x40, mload(add(s, 0x20)))
            mstore(0x00, keccak256(0x00, 0x60))
            return(0x00, 0x20)
        }
    }

    function padStringTo32ByteBytes(string memory s) public pure returns (bytes memory) {
        bytes32 str = bytes32(bytes(s));
        bytes memory b = new bytes(32);

        for (uint8 i; i < 32;) {
            b[i] = str[i];
            unchecked { ++i; }
        }

        return b;
    }
}
```

该合约中的函数可以用于实现hash功能。

首先来看第一个函数`hash()`。该函数输入两个`uint256`的值然后输出其hash结果。`uint256`占据32 bytes，所以第一个数据a赋值到Memory的`0x00`位置，第二个数据b赋值到`0x20`（10进制下结果是32）位置。然后使用`keccak256`读取前面64 bytes数据的值（`0x40`在10进制下是64）并做hash计算，将计算好的结果赋值到`0x00`位置（此处赋值后覆盖原有的结果），最后把Memory的前32 bytes数据返回。

接下来是`hashABIEncode()`其等效为Solidity的`abi.encode`。它和前面的`uint256`略微不同的是`uint256`标识的是定长数据，也就是数据长度是固定的（比如32 bytes），但是像String类型数据在定义的时候不知道将来数据传递过来的长度实际是多少。所以对于动态类型数据，其数据格式的第一部分是length，第二部分才是实际的数据。所以在`mstore(0x20, mload(s))`中，首先来获取s的length也就是offset值。将结果放在`0x20`中，然后从`0x20`中取offset，和原有s数据的起始地址做add拼接：`add(s, 0x20))`，得到实际string数据存储地址，之后将其放到Memory的`0x40`中（需要注意的是这样操作是假设string的数据长度小于32 bytes否则这样赋值会导致截断丢失），最后hash运算并将结果返回。

第三个函数`padStringTo32ByteBytes`用于补`0`填充，简单来说就是生成一个全都是`0`的b，然后将b中对应位置填充上真实数据。

上述例子特别是第二个例子只是展示了从Memory中读取string，接下来是向Memory赋值的例子，实现一个Yul版的Hello World。

```js
contract HelloWorld {
    function greet() external pure returns (string memory) {
        assembly {
            // Assign the string to var `greet`
            // "Hello World!" => 0x48656c6c6f20576f726c64210000000000000000000000000000000000000000.
            let greet := 0x48656c6c6f20576f726c64210000000000000000000000000000000000000000
            
            mstore(0x00, 0x20)
            // Store the length of the string in mem[offset + 32 bytes].
            mstore(0x20, 0x0c) // 0x0c = 12, length of "Hello World!".
            // Store the string in mem[offset + 64 bytes].
            mstore(0x40, greet)
            // Returns the bytes from mem[offset to offset+size]
            return(0x00, 0x60)
        }
    }
}
```

最开始的`mstore(0x00, 0x20)`是ABI设计要求的，告知这个string实际内容占用数据32 bytes。然后在`0x20`中赋值的是string实际的长度或者说offset，这也是之前例子中需要做`mstore(0x20, mload(s))`需要获取offset的原因。之后在`0x40`中添加实际的string内容。

## Storage

Storage数据结构大致类似一个非常大的寻址范围key-value数据库。其中key的可寻址范围为$0$到$2^{256}-1$，是一个非常大的范围，这种设计支撑起mapping映射结构。

<img width="700" alt="Pasted image 20240226185638" src="https://github.com/Einstellung/search/assets/26652483/54775ba3-cfdf-44c6-85da-5c0980504709">

当然除了mapping，还有很多其他类型的数据存储用到了Storage。比如这样的数据结构：

```js
contract StorageTest {
    uint256 a;
    uint256[2] b;

    struct Entry {
        uint256 id;
        uint256 value;
    }
    Entry c;
}
```

他们实际存储的位置就像下图展示的那样，a放在第`0`个位置（Storage将数据存储的位置称之为slot），b是一个数组所以占据两个位置，c是一个struct，里面有两类数据所以也占据两个位置。

![](https://programtheblockchain.com/storage/fixed.png)

同之前的Memory部分的string一样，对于动态类型的数据要稍微麻烦一点。比如此时有一个d是`Entry[] d;`，那么d中的实际数据存储位置slot是根据一个hash算数算出来的。具体来说计算方式是：

```js
function arrLocation(uint256 slot, uint256 index, uint256 elementSize)
    public
    pure
    returns (uint256)
{
    return uint256(keccak256(slot)) + (index * elementSize);
}
```

所以在slot5只是存储d的length，而实际数据在后面hash运算后的slot中存放。

![](https://programtheblockchain.com/storage/dynamic.png)

同Memory类似，Storage也有两个核心opcode：

```
SLOAD(key)

SSTORE(key, value)
```

Solidity的mapping实际数据是存放在Storage中的，接下来我们从Yul的角度来看一下通过Storage是如何实现mapping的。

```js
contract Mapping {
    // Mapping from address to uint.

    mapping(address => uint256) myMap; // slot 0.

    function get(address _addr) public view returns (uint256) {
        assembly {

            let memptr := mload(0x40)

            mstore(memptr, _addr)
            mstore(add(memptr, 0x20), myMap.slot)

            let addrBalanceSlot := keccak256(memptr, 0x40)

            let addrBalance := sload(addrBalanceSlot)
            mstore(0x00, addrBalance)

            return(0x00, 0x20)
        }
    }

    function set(address _addr, uint256 _i) public {
        assembly {
            let memptr := mload(0x40)

            mstore(memptr, _addr)

            mstore(add(memptr, 0x20), myMap.slot)

            let addrBalanceSlot := keccak256(memptr, 0x40)

            sstore(addrBalanceSlot, _i)
        }
    }
}
```

我们首先从`get()`函数开始。首先从`mload(0x40)`获取一块内存地址数据给`memptr`，接着像该地址中存入`_addr`(`mstore(memptr, _addr)`)，随后把slot数据存入`memptr` + 32 bytes的位置，然后根据`keccak256`计算hash值作为value的slot地址。获得到新的地址之后就可以用`sload`获取Storage存储的实际结果了。`set()`方法与之类似，只不过是获得hash之后不是读取值而是存储值。

## Function

通过Yul实现并执行function是一件非常容易的事情，可以在assembly里内置function，下面代码是一个简单示例：

```js
contract Functions {
    function withoutAssemblyReturn(uint256 a, uint256 b) public pure returns (uint256) {
        // Assembly function without a return value.
        assembly {
            function sum(num1, num2) {
                mstore(0x00, add(num1, num2))
            }
            sum(a, b)
            return(0x00, 0x20)
        }
    }

    function withAssemblyReturn(uint256 a, uint256 b) public pure returns (uint256) {
        // Assembly function with a return value.
        assembly {
            function sum(num1, num2) -> total {
                total := add(num1, num2)
            }
            mstore(0x00, sum(a, b))
            return(0x00, 0x20)
        }
    }
}
```

EVM在执行指令的时候除了执行opcode之外，还有可能需要定位并执行其他合约程序程序，`call` opcode是实现该功能的保证。接下来我们结合几个例子学习该opcode。

```
call(gas, address, value(wei), argsOffset, argsSize, retOffset, retSize)
```

```js
contract SendEther {

    constructor() payable {}
    
    function transferEther(uint256 amount, address to) external {
        assembly {
            let s := call(gas(), to, amount, 0x00, 0x00, 0x00, 0x00)
            if iszero(s) {
                revert(0x00, 0x00)
            }
        }
    }

}
```

这段代码实现的功能和`(bool success, ) = to().call{value: amount}("");`是一样的，用于实现发送以太币。

再看一个稍微复杂点的例子：

```js
contract EtherWallet {
    address owner;
    bytes4 constant UnauthorizedSelector = 0x82b42900;

    constructor() payable {
        assembly {
            // caller() returns the address of the msg.sender.
            // It is stored in the slot for `owner()`.
            sstore(owner.slot, caller())
        }
    }

    receive() external payable {}

    function getBalance() external view returns (uint256) {
        assembly {
            // selfbalance() returns address of this contract.
            mstore(0x00, selfbalance())
            return(0x00, 0x20)
        }
    }

    function withdraw(uint256 _amount) external {
        assembly {

            if iszero(eq(caller(), sload(0x00))) {
                mstore(0x00, UnauthorizedSelector)
                revert(0x00, 0x04)
            }
            let sent := call(gas(), caller(), _amount, 0x00, 0x00, 0x00, 0x00)
            if iszero(sent) {
                revert(0x00, 0x00)
            }
        }
    }
}
```

该合约在`constructor`部分把`msg.sender`设置为`owner`。在`withdraw()`函数中首先比较调用该函数的`msg.sender`和Storage中存储的`owner`数据是否一致，如果不一致则返回错误信息，如果一致就接下来把对应`_amount`的以太币发送给`msg.sender`。

然后是一个更复杂一点的例子：

```js
contract CalledContract {
    uint256 public number;

    function setNumber(uint256 num) external {
        assembly {
            sstore(0, num)
        }
    }

    function getNumber() public view returns (uint256) {
        assembly {
            mstore(0x00, sload(0))
            return(0x00, 0x20)
        }
    }
}

contract CallerContract {
    address public called;

    // Deploy with address of CalledContract.
    constructor(address _address) {
        assembly {
            sstore(0, _address)
        }
    }

    function callContract(uint256 num) public {
        address _called = called;

        assembly {
            mstore(0x00, 0x3fb5c1cb)
            mstore(0x20, num)

            let success := call(gas(), _called, 0, 0x1c, 0x24, 0, 0)

            if iszero(success) { revert(0x00, 0x00) }
        }
    }
}
```

在`CallerContract`中首先在初始化`constructor`执行时就将`CalledContract`地址传递过去。接下来在`callContract`函数中，该语句`mstore(0x00, 0x3fb5c1cb)`中`0x3fb5c1cb`是`setNumber`函数的函数签名（`keccak256(setNumber(uint256))`前4位值）用于确定要调用另外一个合约的哪个函数。之后把想要传递的数据放入第Memory中的`0x20`处。这条语句`call(gas(), _called, 0, 0x1c, 0x24, 0, 0)`中`0x1c`是args offset，函数签名占4 bytes。所以实际num的offset是32-4=28也就是`0x1c`。args size是32+4=36所以是`0x24`。这样就确定了要调用其他合约的哪个函数以及内存中的数据传递过去了。