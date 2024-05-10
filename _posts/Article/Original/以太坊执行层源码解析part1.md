## 以太坊执行层

The Merge之后，原有的作为一个整体的以太坊客户端被分为执行层和共识层两部分。

![](https://ethereum.org/_next/image/?url=%2Fcontent%2Fdevelopers%2Fdocs%2Fnodes-and-clients%2Feth1eth2client.png&w=828&q=75)

执行层部分负责数据读写以及状态更改，共识层负责新的区块结点以及区块结点的验证工作。go ethereum舍弃了原有的共识功能，更多的只负责执行层事务，本文接下来分析以太坊执行层功能设计，所使用的项目代码来自[go ethereum](https://github.com/ethereum/go-ethereum)

## geth

整个项目的启动位置来自`cmd/geth/main.go`，运行时首先会执行`init`函数做一些基本的依赖配置和状态初始化，特别是的该函数的首条语句`app.Action = geth`会绑定一个函数，该函数定义了实际命令解析程序，我们接下来会看到。

随后会执行main函数，该函数主体代码如下。非常简单。

```go
func main() {
	if err := app.Run(os.Args); err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
}
```

`app.Run`实际是一个外部的package，该函数执行的时候首先会执行Setup做一些初始化配置，关键语句来自`err = c.Action(cCtx)`这里，会执行我们之前绑定的geth函数。

```go

func (a *App) Run(arguments []string) (err error) {
	return a.RunContext(context.Background(), arguments)
}

// RunContext is like Run except it takes a Context that will be
// passed to its commands and sub-commands. Through this, you can
// propagate timeouts and cancellation requests
func (a *App) RunContext(ctx context.Context, arguments []string) (err error) {
	a.Setup()

	...

	return a.rootCommand.Run(cCtx, arguments...)
}

func (c *Command) Run(cCtx *Context, arguments ...string) (err error) {
	...
	err = c.Action(cCtx)
	...
}
```

geth是实际的命令行解析函数，该函数定义如下：

```go
// geth is the main entry point into the system if no special subcommand is run.
// It creates a default node based on the command line arguments and runs it in
// blocking mode, waiting for it to be shut down.
func geth(ctx *cli.Context) error {
	if args := ctx.Args().Slice(); len(args) > 0 {
		return fmt.Errorf("invalid command: %q", args[0])
	}

	prepare(ctx)
	stack, backend := makeFullNode(ctx)
	defer stack.Close()

	startNode(ctx, stack, backend, false)
	stack.Wait()
	return nil
}
```

`prepare()` 函数主要用于设置一些节点初始化需要的配置。比如，我们在节点启动时看到的这句话: Starting Geth on Ethereum mainnet... 就是在 `prepare()` 函数中被打印出来的。

`makeFullNode()` 函数会将 Geth 启动 时的命令的上下文加载到配置中，并生成 stack 和 backend 这两个实例。其中stack是**Node**实例（此处stack是初始化空的node实例）。

[**Node**](https://ethereum.org/en/developers/docs/nodes-and-clients/)是一个用于控制和管理哪种服务（p2p, http, rpc等）可以被注册的容器（Node is a container on which services can be registered）。Node不只包含各种通信功能，还包含state状态，以及各种数据存储，可以说是执行层至关重要的顶级抽象。

```go
type Node struct {
	eventmux      *event.TypeMux
	config        *Config
	accman        *accounts.Manager
	log           log.Logger
	keyDir        string        // key store directory
	keyDirTemp    bool          // If true, key directory will be removed by Stop
	dirLock       *flock.Flock  // prevents concurrent use of instance directory
	stop          chan struct{} // Channel to wait for termination notifications
	server        *p2p.Server   // Currently running P2P networking layer
	startStopLock sync.Mutex    // Start/Stop are protected by an additional lock
	state         int           // Tracks state of node lifecycle

	lock          sync.Mutex
	lifecycles    []Lifecycle // All registered backends, services, and auxiliary services that have a lifecycle
	rpcAPIs       []rpc.API   // List of APIs currently provided by the node
	http          *httpServer //
	ws            *httpServer //
	httpAuth      *httpServer //
	wsAuth        *httpServer //
	ipc           *ipcServer  // Stores information about the ipc http server
	inprocHandler *rpc.Server // In-process RPC request handler to process the API requests

	databases map[*closeTrackingDB]struct{} // All open databases
}
```

在实际使用时，Node将作为桥梁，与外部世界通信并改变内部的EVM数据信息。

![](https://ethereum.org/_next/image/?url=%2Fcontent%2Fdevelopers%2Fdocs%2Fnodes-and-clients%2Fnodes.png&w=828&q=75)

回到`makeFullNode()`函数，该函数大致结构如下：

```go
// makeFullNode loads geth configuration and creates the Ethereum backend.
func makeFullNode(ctx *cli.Context) (*node.Node, ethapi.Backend) {
	stack, cfg := makeConfigNode(ctx)
	...

	backend, eth := utils.RegisterEthService(stack, &cfg.Eth)
	...
	
	return stack, backend
}
```

除了stack是初始化Node外，backend是`ethapi.Backend`类型。`utils.RegisterEthService()`函数根据Node生成对应的eth backend并将该backend和Node绑定。

```go
// RegisterEthService adds an Ethereum client to the stack.
// The second return value is the full node instance.
func RegisterEthService(stack *node.Node, cfg *ethconfig.Config) (ethapi.Backend, *eth.Ethereum) {
	backend, err := eth.New(stack, cfg)
	if err != nil {
		Fatalf("Failed to register the Ethereum service: %v", err)
	}
	stack.RegisterAPIs(tracers.APIs(backend.APIBackend))
	return backend.APIBackend, backend
}
```

`eth.New(stack, cfg)`用于生成新的backend，该函数位于`eth/backend.go`。Ethereum用于具体控制以太坊协议层相关功能，其包含 TxPool， Blockchain，consensus.Engine，miner 等最核心的几个数据结构作为成员变量。该结构体事实上也是**Ethereum full node**对应的backend的数据结构。

```go
// Ethereum implements the Ethereum full node service.
type Ethereum struct {
	config *ethconfig.Config

	// Handlers
	txPool *txpool.TxPool

	blockchain         *core.BlockChain
	handler            *handler
	ethDialCandidates  enode.Iterator
	snapDialCandidates enode.Iterator

	// DB interfaces
	chainDb ethdb.Database // Block chain database

	eventMux       *event.TypeMux
	engine         consensus.Engine
	accountManager *accounts.Manager

	bloomRequests     chan chan *bloombits.Retrieval // Channel receiving bloom data retrieval requests
	bloomIndexer      *core.ChainIndexer             // Bloom indexer operating during block imports
	closeBloomHandler chan struct{}

	APIBackend *EthAPIBackend

	miner    *miner.Miner
	gasPrice *big.Int

	networkID     uint64
	netRPCService *ethapi.NetAPI

	p2pServer *p2p.Server

	lock sync.RWMutex // Protects the variadic fields (e.g. gas price and etherbase)

	shutdownTracker *shutdowncheck.ShutdownTracker // Tracks if and when the node has shutdown ungracefully
}
```

接下来我们回到`geth()`，来看一下`startNode()`。该函数的作用就是启动Node，启动诸如RPC/IPC通信，矿工，钱包等。

随后程序会一直处于stack.Wait()状态，响应各种需求以及执行对应操作。当stack状态发生变更为结束时会执行`stack.Close()`，释放各种资源，做好退出程序准备。

## Accounts

比特币系统记账方式所使用的是UTXO。如下图所示，是UTXO的简易模型，比如Alice向Bob发送0.5比特币，此时并不是Alice的账户余额减少0.5比特币，而是将0.5比特币拆分出来，所有权转给Bob。

<div align="center">
<img src="https://miro.medium.com/v2/resize:fit:640/format:webp/1*cyyvFClix3FBrciOvx4Klg.jpeg"  alt="图片名称" align=center />
</div>
<br />

而Bob的账户余额总数来自链上有所有权交易的数值总和。

<div align="center">
<img src="https://miro.medium.com/v2/resize:fit:640/format:webp/1*NMtkerUsrmcw1bOR3pnscw.jpeg"  alt="图片名称" align=center />
</div>
<br />

比特币这种设计使得整个网络结构较为简单，但是能支撑的网络复杂度有限，以太坊的目标是承载去中心化app，需要更复杂的能力，其账户系统设计采用的是Account/State方式，有点类似传统金融机构给每个人有一个账户记录余额。

Ethereum 的运行是一种基于交易的状态机模型 (Transaction- based State Machine)，类似于设计模式中的状态模式。整个系统由若干的账户组成 (Account)，状态 (State) 反应了某一账户 (Account) 在某一时刻下的值 (value)。在以太坊中， State 对应的基本数据结构，称为 StateObject。当 StateObject 的值发生了变化时， 我们称为状态转移。在 Ethereum 的运行模型中，StateObject 所包含的数据会因为 Transaction 的执行引发数据更新/删除/创建，引发状态转移，我们说StateObject的状态从当前的 State 转移到另一个 State。

通常，我们提到的 State 具体指的就是 Account 在某个时刻下所包含的数据的值。目前，在以太坊中，有两种类型的 Account，分别是外部账户 (EOA) 以及合约账户 (Contract)。

在实际代码中，这两种 Account 都是由 stateObject 这一数据结构定义的。stateObject 的相关代码位于`core/state/state_object.go`文件中。

我们来看一下stateObject的类型定义：

```go
type stateObject struct {
	db       *StateDB
	address  common.Address      // address of ethereum account
	addrHash common.Hash         // hash of ethereum address of the account
	origin   *types.StateAccount // Account original data without any change applied, nil means it was not existent
	data     types.StateAccount  // Account data with all mutations applied in the scope of block

	// Write caches.
	trie Trie // storage trie, which becomes non-nil on first access
	code Code // contract bytecode, which gets set when code is loaded

	originStorage  Storage // Storage cache of original entries to dedup rewrites
	pendingStorage Storage // Storage entries that need to be flushed to disk, at the end of an entire block
	dirtyStorage   Storage // Storage entries that have been modified in the current transaction execution, reset for every transaction

	// Cache flags.
	dirtyCode bool // true if the code was updated

	// Flag whether the account was marked as self-destructed. The self-destructed account
	// is still accessible in the scope of same transaction.
	selfDestructed bool

	// Flag whether the account was marked as deleted. A self-destructed account
	// or an account that is considered as empty will be marked as deleted at
	// the end of transaction and no longer accessible anymore.
	deleted bool

	// Flag whether the object was created in the current transaction
	created bool
}
```

首先遇到的数据类型是db，db 这个变量保存了一个 StateDB 类型的指针。这是为了方便调用 StateDB 相关的 API 对 Account 所对应的 stateObject 进行操作。StateDB 本质上是用于管理 stateObject 信息的而抽象出来的内存数据库。所有的 Account 数据的更新，检索都会使用 StateDB 提供的 API。我们后面会对db进一步展开分析。

随后是address和addrHash，前者存储的是该账户的addr（20 byte），后者存储的是该账户addr的Hash（32 byte）。在Ethereum中，每个Account都拥有独一无二的地址。Address 作为每个 Account 的身份信息，类似于现实生活中的身份证，它与用户信息时刻绑定而且不能被修改。

origin的作用似乎是为了可以比较交易前和交易后账户状态信息（#[27349 commit](https://github.com/ethereum/go-ethereum/commit/4b06e4f25e9a97880ddf834b571b05a59fa1290c)）。

data表示的是供外部可以改变的区块数据。其具体定义为：

```go
type StateAccount struct {
	Nonce    uint64
	Balance  *uint256.Int
	Root     common.Hash // merkle root of the storage trie
	CodeHash []byte
}
```

对于EOA而言，Root和CodeHash都为空值，你可以在该[文档](https://ethereum.org/en/developers/docs/accounts/)中找到这4个字段的具体解释。

![](https://ethereum.org/content/developers/docs/accounts/accounts.png)

之前的Root只是存储Merkle Patricia trie 根节点的 256 位哈希，具体的trie在trie中保存，code保存整个智能合约的字节码。剩下的三个Storage字段主要在执行Transaction的时候缓存合约修改的持久化数据，比如 dirtyStorage 就用于缓存在 Block 被 Finalize 之前，Transaction 所修改的合约中的持久化存储数据。

此处的Storage和Solidity编程中的storage概念是一致的。在Solidity中，storage是key-value存储对象。而此处的Storage实际类型是：

```go
type Storage map[common.Hash]common.Hash

// Hash represents the 32 byte Keccak256 hash of arbitrary data.
type Hash [HashLength]byte
```

可以看到，也是一个key-value存储对象，是32 byte -> 32 byte的映射。在Solidity中我们使用[slot概念](https://mp.weixin.qq.com/s/n4Pd5eN8Zg8GoxJ-JBsvQw)来表示这种映射关系，此处的32 byte -> 32 byte是从代码层面定义了slot，因此整个Storage能够存储数据大小为 ($2^{256} - 1$) * 32 byte。

因为外部的EOA账户不涉及到合约自然也不涉及Storage存储，所以三个Storage类型的字段对应的变量的值都为空。

我在[slot概念](https://mp.weixin.qq.com/s/n4Pd5eN8Zg8GoxJ-JBsvQw)这篇文章的part3部分介绍storage的时候所举的yul代码例子只是32 byte大小，实际上我们存储的数据可能不是32 byte大小，比如addr就是20 byte。这个时候如果我们用Solidity写代码的话，数据不足32 byte会默认将数据合并以节省内存空间（比如addr和bool类型会合并占一个slot）。我们之前看到go定义对于Storage而言，数据读写都是按照32 byte为单位的，所以，如果我们想要得到addr数据，但实际上系统会把addr和bool类型数据都读出来，然后再单独获取addr数据并返回。如果这块数据需要反复读写的话，那么额外复杂运算开销很可能会大于合并存储slot所节省的那一点成本。所以在Ethereum使用32 byte的变量， 在某些情况下消耗的Gas反而比更小长度类型的变量要小。这也是为什么Ethereum官方也建议使用长度为32 byte变量的原因。

## Tries

以太坊数据存储采用的是Merkle Patricia Trie的方式。其存储结构类似于

```
[i_0, i_1 ... i_n, value]
```

前面各项数据类似于树的索引，最后的value是实际存储数据。要想理解Merkle Patricia Trie，首先要理解Patricia Trie，关于该部分知识如果不了解的话可以参考该[视频教程](https://www.youtube.com/watch?v=AySCXmoEGrw&ab_channel=BuidlerDAO)，Merkle Patricia Trie部分可以参考这个[视频教程](https://www.youtube.com/watch?v=TelOgcqjKG8&list=PLOGGvFbKWOAQJWncBsun4a1ln5ScTzJu2&index=19&ab_channel=BuidlerDAO)。

具体来说，在以太坊一个区块中会使用[4种 trie ](https://arxiv.org/pdf/2108.05513/1000.pdf)来分别结构化存储各类数据。


<div align="center">
<img width="769" alt="Pasted image 20240408093450" src="https://github.com/Einstellung/search/assets/26652483/c0649f00-39d2-4102-9ee4-e56589796f8b">
</div>
<br />

**state trie** ：
该trie存储了以太坊各个账户（EOA和contract）实际信息。对于contract而言，它还有一个**storage Trie**，按slot的方式存储合约storage信息。其存储结构类似这样：

```
                   +-- (k: "01", v: "hello")
              +----|---- (k: "012", v: "world")
              |    +-- (k: "02", v: "foo")
              |
(root node)---+-- (k: "1a", v: "bar")
              |
              |    +-- (k: "123", v: "baz")
              +----|---- (k: "13", v: "qux")
                   +-- (k: "2", v: "abc")
```

**transaction trie**：
该trie存储了区块的交易信息。当新交易提交到以太坊网络时，它首先由网络上的节点验证，然后添加到交易树中。通过对其交易 ID 进行哈希处理，然后将其作为键值对添加到trie中，key的添加使用[RLP](https://ethereum.org/en/developers/docs/data-structures-and-encoding/rlp/)的方式。以太坊网络使用交易树来跟踪网络已提交和处理的所有交易，交易 Trie 用于验证新块中的交易列表是否与网络状态匹配，并且不存在冲突交易或double-spending问题。

**receipts trie**：
当一个块被添加到区块链时，它还包含与每笔交易相对应的收据列表，表示该交易的结果。收据包括使用的Gas量、创建的合约地址（如果适用）以及交易期间生成的日志等，该trie就是用来存储这些收据信息。

我们接下来简要看一下trie的功能代码。在之前的`StateObject`中，我们已经看到trie的调用，跟着代码跳转到trie的具体使用中，位于文件`core/state/database.go`。在该部分中，我们只是定义了trie的接口

```go
type Trie interface {
	GetKey([]byte) []byte
	GetAccount(address common.Address) (*types.StateAccount, error)
	...
```

更底层的实现在`trie/secure_trie.go`和`trie/verkle.go`。我们之前文章中讲的trie都是merkle tree，接下来所说的trie也是如此，暂时不会涉及到verkle tree。

trie的具体structure结构如图所示：

<div align="center">
<img src="https://github.com/Einstellung/search/assets/26652483/f565d5eb-27e0-493a-911b-bd6ef1e95d7e"  alt="图片名称" align=center />
</div>
<br />

在trie中共有3类节点，分别是leaf node，extension node和branch node。extension node包含压缩部分的key，还有剩下的node。branch node包含17个字段，其中前16个对应于这些点在其遍历中的键的十六个可能的半字节值（nibble）中的每一个。第17个字段是存储那些在当前节点结束了的value(例如， 有三个key,分别是 (abc ,abd, ab) 第17个字段储存了ab节点的值)，如果不太清楚可以回看一下Tries开头介绍的存储结构。

实际代码的名称和上述图片略有出入。branch node改成了fullNode，extension node改成了shortNode。代码在`trie/node.go`

```go
type node interface {
	cache() (hashNode, bool)
	encode(w rlp.EncoderBuffer)
	fstring(string) string
}

type (
	fullNode struct {
		Children [17]node // Actual trie node data to encode/decode (needs custom encoder)
		flags    nodeFlag
	}
	shortNode struct {
		Key   []byte
		Val   node
		flags nodeFlag
	}
	hashNode  []byte
	valueNode []byte
)

// nodeFlag contains caching-related metadata about a node.
type nodeFlag struct {
	hash  hashNode // cached hash of the node (may be nil)
	dirty bool     // whether the node has changes that must be written to the database
}
```

接下来看一下trie的具体结构`trie/trie.go`：

```go
type Trie struct {
	root  node
	owner common.Hash

	// Flag whether the commit operation is already performed. If so the
	// trie is not usable(latest states is invisible).
	committed bool

	// Keep track of the number leaves which have been inserted since the last
	// hashing operation. This number will not directly map to the number of
	// actually unhashed nodes.
	unhashed int

	// reader is the handler trie can retrieve nodes from.
	reader *trieReader

	// tracer is the tool to track the trie changes.
	// It will be reset after each commit operation.
	tracer *tracer
}
```

我们简单分析一下get方法获取trie中的某个元素，可以看到，实际的获取方式就是递归的逐个数据去做比较来最终找到对应的k-v。

```go
func (t *Trie) get(origNode node, key []byte, pos int) (value []byte, newnode node, didResolve bool, err error) {
	switch n := (origNode).(type) {
	case nil:
		return nil, nil, false, nil
	case valueNode:
		return n, n, false, nil
	case *shortNode:
		if len(key)-pos < len(n.Key) || !bytes.Equal(n.Key, key[pos:pos+len(n.Key)]) {
			// key not found in trie
			return nil, n, false, nil
		}
		value, newnode, didResolve, err = t.get(n.Val, key, pos+len(n.Key))
		if err == nil && didResolve {
			n = n.copy()
			n.Val = newnode
		}
		return value, n, didResolve, err
	case *fullNode:
		value, newnode, didResolve, err = t.get(n.Children[key[pos]], key, pos+1)
		if err == nil && didResolve {
			n = n.copy()
			n.Children[key[pos]] = newnode
		}
		return value, n, didResolve, err
	case hashNode:
		child, err := t.resolveAndTrack(n, key[:pos])
		if err != nil {
			return nil, n, true, err
		}
		value, newnode, _, err := t.get(child, key, pos)
		return value, newnode, true, err
	default:
		panic(fmt.Sprintf("%T: invalid node: %v", origNode, origNode))
	}
}
```

## State DB

我们在之前介绍了stateObject，但是stateObject只是对应一个账户，将这些账户组织起来并赋予CRUD能力是通过stateDB实现的，具体代码在`core/state/statedb.go`。StateDB本质上是一个用于管理所有账户状态的位于内存中的抽象组件。从某种意义上说，我们可以把它理解成一个中间层的内存数据库，它会从调用更底层的实现（trie可能对应的是secure trie或者verkle trie。db也有更底层实现）从磁盘中读取k-v数据然后进行数据转换，方便更上层的应用比如block消费数据。
