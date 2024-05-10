
## Transaction

在之前一篇文章，我们分析了State。明确以太坊执行层整体来说可以看成是一个状态机。而对于Transaction而言，可以将其当作状态机的状态转移函数，来触发状态机从一个状态转移为另一个状态。我们接下来具体分析一下[Transaction](https://preethikasireddy.medium.com/how-does-ethereum-work-anyway-22d1df506369)。

![](https://miro.medium.com/v2/resize:fit:720/format:webp/1*jZ-VRXBJtOnePofB0z2Q8A.png)
以太坊的交易可以由外部账户发起也可以由合约发起，但是合约本身就算和另外一个合约发起交易也不可能是合约自主发起的，一定是外部账户调用合约的某个函数，之后一个合约再和另外一个合约通信。

![](https://miro.medium.com/v2/resize:fit:720/format:webp/1*I635Y9btMh667inOhDBQ_g.png)

所以最终实际交易总是实际由外部账户发起的。

代码位于`core/types/transaction.go`，同之前分析trie代码一样，该处代码只是定义了transaction类型而没有具体实现（来自[EIP-2718](https://eips.ethereum.org/EIPS/eip-2718)提案）。具体实现transaction分成了（伦敦升级之后，详情看[黄皮书](https://ethereum.github.io/yellowpaper/paper.pdf)）DynamicFeeTx（[EIP-1559](https://eips.ethereum.org/EIPS/eip-1559)）, LegacyTx（原始版本） ， AccessListTx（[EIP-2930](https://eips.ethereum.org/EIPS/eip-2930)），BlobTx（[EIP-4844](https://docs.web3j.io/4.11.0/transactions/EIP_transaction_types/eip4844_blob_transaction/)），接口和实现解耦设计的一个好处是方便将来以太坊升级，代码改动会更加非侵入式。

```go
// Transaction types.
const (
	LegacyTxType     = 0x00
	AccessListTxType = 0x01
	DynamicFeeTxType = 0x02
	BlobTxType       = 0x03
)

type Transaction struct {
	inner TxData    // Consensus contents of a transaction
	time  time.Time // Time first seen locally (spam avoidance)

	// caches
	hash atomic.Value
	size atomic.Value
	from atomic.Value
}

type TxData interface {
	txType() byte // returns the type ID
	copy() TxData // creates a deep copy and initializes all fields

	chainID() *big.Int
	accessList() AccessList
	data() []byte
	gas() uint64
	gasPrice() *big.Int
	gasTipCap() *big.Int
	gasFeeCap() *big.Int
	value() *big.Int
	nonce() uint64
	to() *common.Address

	rawSignatureValues() (v, r, s *big.Int)
	setSignatureValues(chainID, v, r, s *big.Int)

	// effectiveGasPrice computes the gas price paid by the transaction, given
	// the inclusion block baseFee.
	//
	// Unlike other TxData methods, the returned *big.Int should be an independent
	// copy of the computed value, i.e. callers are allowed to mutate the result.
	// Method implementations can use 'dst' to store the result.
	effectiveGasPrice(dst *big.Int, baseFee *big.Int) *big.Int

	encode(*bytes.Buffer) error
	decode([]byte) error
}
```

我们先从原始版本transaction开始看。代码位于`core/types/tx_legacy.go`。

```go
type LegacyTx struct {
	Nonce    uint64          // nonce of sender account
	GasPrice *big.Int        // wei per gas
	Gas      uint64          // gas limit
	To       *common.Address `rlp:"nil"` // nil means contract creation
	Value    *big.Int        // wei amount
	Data     []byte          // contract invocation input data
	V, R, S  *big.Int        // signature values
}
```

各字段概念解释如下。

- nonce：发送者发送的交易数量计数。nonce表示是number only use once，它是该账户发起交易的计数器，从0开始，每次账户发起一笔交易该值就会增加1。这样做的目的是为了防止重放攻击（同一交易被恶意或误操作多次发送）。通过要求每个交易nonce递增，网络能够区分独立的交易并拒绝重复交易。
- gasPrice：发送者愿意为每单位gas支付的Wei数。
- gas：发送者愿意为执行此交易支付的最大gas量。这个数额是在任何计算之前就设定并支付的。
- to：接收者的地址。在创建合约的交易中，如果合约账户地址尚未存在，则使用一个空值。
- value：从发送者转移到接收者的Wei数额。在创建合约的交易中，这个值作为新创建的合约账户的初始余额。
- v, r, s：用来生成标识交易发送者的签名。
- data（只存在于消息调用中的可选字段）：输入数据（即参数），用于消息调用。例如，如果一个智能合约提供域名注册服务，那么对该合约的调用可能需要输入诸如域名和IP地址等字段。

接下来是AccessListTx，`core/types/tx_access_list.go`

```go
type AccessListTx struct {
	ChainID    *big.Int        // destination chain ID
	Nonce      uint64          // nonce of sender account
	GasPrice   *big.Int        // wei per gas
	Gas        uint64          // gas limit
	To         *common.Address `rlp:"nil"` // nil means contract creation
	Value      *big.Int        // wei amount
	Data       []byte          // contract invocation input data
	AccessList AccessList      // EIP-2930 access list
	V, R, S    *big.Int        // signature values
}

// AccessList is an EIP-2930 access list.
type AccessList []AccessTuple

// AccessTuple is the element type of an access list.
type AccessTuple struct {
	Address     common.Address `json:"address"     gencodec:"required"`
	StorageKeys []common.Hash  `json:"storageKeys" gencodec:"required"`
}
```

可以看到AccessListTx在LegacyTx基础上多了ChainID和AccessList这两个变量。EIP-2929改动整体来说就是提高state access code的gas消耗，EIP2930目的是为了在Tx执行之前，预先执行cold access。解决contract breakage问题，关于这方面具体解释可以看[这里](http://liwuzhi.art/?p=1140)。

传统的Tx在执行过程中，根据实际操作执行扣除gas费。而AccessList内部包含了accessed_addresses和accessed_storage_keys，说明本Tx可能会涉及到读取哪些合约，以及合约状态。那么系统在执行Tx之前，预先把指定的状态读取出来，缓存起来，然后再正常执行Tx，执行过程中需要扣费的话，属于warm access，费用就降低了。

如果交易模式有access lists也就是AccessListTx交易，那么gas消耗计算方式会有所不同。gas消耗计算会分成两部分一部分是交易本身的，另外一部分是access list获取数据所需的gas，虽然总的gas消耗可能会有所增涨，但是交易本身的gas可能会下降。这是因为在执行过程中，以太坊虚拟机（EVM）不需要将这些值从磁盘加载到内存中；它们已经在一开始就加载好了。

一个实际的AccessList写入数据可能是这个[样子](https://docs.web3j.io/4.11.0/transactions/EIP_transaction_types/eip2930_transaction/)：

```json
{
  "accessList": [
    {
      "address": "0xa02457e5dfd32bda5fc7e1f1b008aa5979568150",
      "storageKeys": ["0x0000000000000000000000000000000000000000000000000000000000000081",
      ]
    }
  ]
  "gasUsed": "0x125f8"
}
```

```java
private static RawTransaction createEip2930RawTransaction() {
    // Test example from https://eips.ethereum.org/EIPS/eip-2930
    List<AccessListObject> accessList = Stream.of(
            new AccessListObject(
                    "0xde0b295669a9fd93d5f28d9ec85e40f4cb697bae",
                    Stream.of(      "0x0000000000000000000000000000000000000000000000000000000000000003",                           "0x0000000000000000000000000000000000000000000000000000000000000007"
                    ).collect(Collectors.toList())),
            new AccessListObject(
                    "0xbb9bc244d798123fde783fcc1c72d3bb8c189413",
                    Collections.emptyList())
    ).collect(Collectors.toList());
}
```

实际整个交互过程类似于：
1. 用户发送调用A合约的tx, gas limit为3000。并且在Tx的AccessList中指定了要访问的状态S1。
2. EVM执行tx之前，预先往缓存中加载了S1，扣除gas 1900（没错，扣除1900，而不是2100，因为通过EIP2930的方式加载缓存比直接cold access更便宜)
3. A合约调用B合约，发送800gas。
4. B合约从存储中读区状态，消耗100gas（此时属于warm access，只需要100，而不是2100）。执行成功。
5. A合约执行执行成功。总消耗gas: 1900+100=2000。返回用户gas:3000-1000=2000。

接下来是DynamicFeeTx，代码位于`core/types/tx_dynamic_fee.go`

```go
type DynamicFeeTx struct {
	ChainID    *big.Int
	Nonce      uint64
	GasTipCap  *big.Int // a.k.a. maxPriorityFeePerGas
	GasFeeCap  *big.Int // a.k.a. maxFeePerGas
	Gas        uint64
	To         *common.Address `rlp:"nil"` // nil means contract creation
	Value      *big.Int
	Data       []byte
	AccessList AccessList

	// Signature values
	V *big.Int `json:"v" gencodec:"required"`
	R *big.Int `json:"r" gencodec:"required"`
	S *big.Int `json:"s" gencodec:"required"`
}
```

DynamicFeeTx在AccessListTX 的定义的基础上额外的增加了GasTipCap（矿工小费）与 GasFeeCap（基础费）这两个字段。如果对EIP-1559不太清楚可以参考这篇[文章](https://help.tokenpocket.pro/cn/faq/ethwallet/eip-1559)。

### 4844

然后是BlobTx，代码在`core/types/tx_blob.go`

```go
type BlobTx struct {
	ChainID    *uint256.Int
	Nonce      uint64
	GasTipCap  *uint256.Int // a.k.a. maxPriorityFeePerGas
	GasFeeCap  *uint256.Int // a.k.a. maxFeePerGas
	Gas        uint64
	To         common.Address
	Value      *uint256.Int
	Data       []byte
	AccessList AccessList
	BlobFeeCap *uint256.Int // a.k.a. maxFeePerBlobGas
	BlobHashes []common.Hash

	// A blob transaction can optionally contain blobs. This field must be set when BlobTx
	// is used to create a transaction for signing.
	Sidecar *BlobTxSidecar `rlp:"-"`

	// Signature values
	V *uint256.Int `json:"v" gencodec:"required"`
	R *uint256.Int `json:"r" gencodec:"required"`
	S *uint256.Int `json:"s" gencodec:"required"`
}

// BlobTxSidecar contains the blobs of a blob transaction.
type BlobTxSidecar struct {
	Blobs       []kzg4844.Blob       // Blobs needed by the blob pool
	Commitments []kzg4844.Commitment // Commitments needed by the blob pool
	Proofs      []kzg4844.Proof      // Proofs needed by the blob pool
}

// Blob represents a 4844 data blob.
type Blob [131072]byte

// Commitment is a serialized commitment to a polynomial.
type Commitment [48]byte

// Proof is a serialized commitment to the quotient polynomial.
type Proof [48]byte
```

可以看到BlobTx相比DynamicFeeTx增加了3个关于Blob的字段。`BlobFeeCap`表示提交者愿意为每个Blob出价的最大价格是多少，`BlobHashes`是一个数组，是对每个Blob信息的hash结果。需要注意的是实际存储的transaction tree是不包含Blob的具体信息，只有Blob的hash，可以理解成执行层只会存储Blob的[引用](https://domothy.com/blobspace/)。另外一个字段是`SideCar`，非常符合字面意思，想象一个[挎斗摩托](https://mp.weixin.qq.com/s?__biz=Mzg5ODU3NjkwNg==&mid=2247496102&idx=1&sn=6593f4d4f80fb39c020b50789ddf8570&scene=21#wechat_redirect)，SideCar就是这样的一个挎斗数据，是Blob交易信息的副产品，随时可以从BlobTx中摘除。一个交易总共可以挂6个blob，一个blob大小是128KB（131072 / 1024 == 128），每个数据大小按照32 bytes计算，因此一个blob实际可以存储4096（131072 / 32 == 4096）个数据。需要注意的是由于椭圆曲线素数的限制，实际存储的每个数据范围只能是

```
0 <= x < 52435875175126190479447740508185965837690552500527637822603658699938581184513
```

关于Blob的具体交易示例可以参考这篇[文章](https://www.blog.blockscout.com/blobs/)。

增加BlobTx主要是为了方便L2 Rollup打包，在过去L2数据存储到以太坊主链上主要通过calldata的方式，在4844之后，有一个专属的Blob用于存储L2打包的数据。在未来，L1 将越来越作为数据层保证去中心化和安全性，而L2 负责执行具体去中心化应用的计算功能。整个以太坊逐渐走向[模块化区块链](https://mirror.xyz/web3cn-pro.eth/GShK_xYtpZS9PKUkyPwbudzaIxHJfpv-ifKtRD48v70)方向。

![123](https://github.com/Einstellung/search/assets/26652483/5637f4a9-21c7-49e2-ae79-cf53699f83c9)


L2和L1关于Blob交互整体流程如图所示。整体来看，就是在原有的数据交互和存储之外再单独开辟一个关于Blob的数据交互和存储单元。结合之前的BlobTx代码我们也可以看到，L2提交数据的时候SideCar数据也要提交，就是L2负责proof和commitment生成，L1只负责数据验证工作。看到这里你可能会有一个疑惑，我只是存储一个临时数据，之前数据存储都是用hash来做校验就好了，为什么这次改用kzg？commit到底想要commit什么？事实上在Blob中引入kzg和L2 rollup欺诈证明没有任何关系，它是为了将来erasure coding服务的，通过引入erasure coding未来可以实现数据分片，不再需要每一个节点都保存全部数据，同时依旧可以保证整个网络的数据可用性，进而大大增强整个以太坊网络的数据吞吐能力，提高TPS。（目前坎昆升级版本没有包含erasure coding，4844只是Danksharding的前置版本）

关于[Blob交易费用](https://mp.weixin.qq.com/s/0qOb0Vh3YPFXDzdSlsDh6g)，blob 交易中，blob 数据存储部分的价格是单独计算的，而且会随之前区块 blob 的使用情况而变动，如果上一个区块中 blob 的使用超过了 blob 最大限制的 50%，那么费用就会上调，如果低于 50%，那么费用就会降低。

![](https://i.imgur.com/sB5JR3W.png)

接下来具体分析一下如何对Blob做kzg计算。对blob做kzg需要使用以太坊为4844写的基础依赖package [gokzg4844](github.com/crate-crypto/go-kzg-4844)，我们就先从gokzg4844开始看。

从`examples_test.go`作为代码分析入口，第一个测试代码是：

```go
func TestBlobProveVerifyRandomPointIntegration(t *testing.T) {
	blob := GetRandBlob(123)
	commitment, err := ctx.BlobToKZGCommitment(blob, NumGoRoutines)
	require.NoError(t, err)
	proof, err := ctx.ComputeBlobKZGProof(blob, commitment, NumGoRoutines)
	require.NoError(t, err)
	err = ctx.VerifyBlobKZGProof(blob, commitment, proof)
	require.NoError(t, err)
}
```

首先进入`BlobToKZGCommitment`。

第一步是`DeserializeBlob`，对Blob以每32 bytes数据做一个分割，分割后的32 bytes数据按照大端序重新编码，然后4096个数据为一组合成polynomial，为将来commit做准备。

第二步是计算Commit，Commit实际就是一个多项式复制的过程，其中blob是y值，trusted setup是x（Lagrange basis形式）。在计算时可以使用double and add方式将原来的时间复杂度从$O(n)$[减少](https://andrea.corbellini.name/2015/05/17/elliptic-curve-cryptography-a-gentle-introduction/)到$O(log(n))$。

第三步是对第二步生成的Commitment再做序列化处理。第二步的commitment实际上就是一个椭圆曲线上的坐标（x，y），接下来要做一些压缩计算然后将其转换成bytes数组（之前代码中我们看到是48 byte）。

```go
func (c *Context) BlobToKZGCommitment(blob *Blob, numGoRoutines int) (KZGCommitment, error) {
	// 1. Deserialization
	//
	// Deserialize blob into polynomial
	polynomial, err := DeserializeBlob(blob)
	if err != nil {
		return KZGCommitment{}, err
	}

	// 2. Commit to polynomial
	commitment, err := kzg.Commit(polynomial, c.commitKey, numGoRoutines)
	if err != nil {
		return KZGCommitment{}, err
	}

	// 3. Serialization
	//
	// Serialize commitment
	serComm := SerializeG1Point(*commitment)

	return KZGCommitment(serComm), nil
}
```

接下来我们再看一下Proof的计算过程。

第一步和之前的分析类似，此处略过。第二步是使用非交互[Fiat-Shamir](https://en.wikipedia.org/wiki/Fiat%E2%80%93Shamir_heuristic)方式利用hash方法生成challenge。

```go
func (c *Context) ComputeBlobKZGProof(blob *Blob, blobCommitment KZGCommitment, numGoRoutines int) (KZGProof, error) {
	// 1. Deserialization
	//
	polynomial, err := DeserializeBlob(blob)
	if err != nil {
		return KZGProof{}, err
	}

	// Deserialize commitment
	//
	// We only do this to check if it is in the correct subgroup
	_, err = DeserializeKZGCommitment(blobCommitment)
	if err != nil {
		return KZGProof{}, err
	}

	// 2. Compute Fiat-Shamir challenge
	evaluationChallenge := computeChallenge(blob, blobCommitment)

	// 3. Create opening proof
	openingProof, err := kzg.Open(c.domain, polynomial, evaluationChallenge, c.commitKey, numGoRoutines)
	if err != nil {
		return KZGProof{}, err
	}

	// 4. Serialization
	//
	// Quotient commitment
	kzgProof := SerializeG1Point(openingProof.QuotientCommitment)

	return KZGProof(kzgProof), nil
}
```

第三步是做打开操作。打开的点就是用的之前`evaluationChallenge`，在这里我们将其命名为z。

```go
func Open(domain *Domain, p Polynomial, evaluationPoint fr.Element, ck *CommitKey, numGoRoutines int) (OpeningProof, error) {
	if len(p) == 0 || len(p) > len(ck.G1) {
		return OpeningProof{}, ErrInvalidPolynomialSize
	}

	outputPoint, indexInDomain, err := domain.evaluateLagrangePolynomial(p, evaluationPoint)
	if err != nil {
		return OpeningProof{}, err
	}

	// Compute the quotient polynomial
	quotientPoly, err := domain.computeQuotientPoly(p, indexInDomain, *outputPoint, evaluationPoint)
	if err != nil {
		return OpeningProof{}, err
	}

	// Commit to Quotient polynomial
	quotientCommit, err := Commit(quotientPoly, ck, numGoRoutines)
	if err != nil {
		return OpeningProof{}, err
	}

	res := OpeningProof{
		InputPoint:   evaluationPoint,
		ClaimedValue: *outputPoint,
	}

	res.QuotientCommitment.Set(quotientCommit)

	return res, nil
}

// OpeningProof is a struct holding a (cryptographic) proof to the claim that a polynomial f(X) (represented by a
// commitment to it) evaluates at a point `z` to `f(z)`.
type OpeningProof struct {
	// Commitment to quotient polynomial (f(X) - f(z))/(X-z)
	QuotientCommitment bls12381.G1Affine

	// Point that we are evaluating the polynomial at : `z`
	InputPoint fr.Element

	// ClaimedValue purported value : `f(z)`
	ClaimedValue fr.Element
}
```

其中`evaluateLagrangePolynomial`用于计算f(z)，也就是评估z点的函数计算结果f(z)。

我们可以用拉格朗日的方式表示一个多项式，其中 $L_i(x)$ 表示Lagrange basis多项式，$P(x)$ 是经过插值计算后得到的实际多项式：

$$
\begin{align*}
P(x) &= \sum_{i=0}^{n} y_i \cdot L_i(x) \\
L_i(x) &= \prod_{\substack{j=0 \\ j \neq i}}^{n} \frac{x - x_j}{x_i - x_j}
\end{align*}
$$

这样当 $x=x_i$ 分子和分母所有项就相同，$L_i(x)=1$，如果 $x \neq x_i$ 那么分子中会有一项 $x = x_j$ 此时 $L_i(x)=0$ 。这也是为什么要计算`indexInDomain = domain.findRootIndex(evalPoint)`，如果能找到对应的根，直接返回y值即可（输入的`Polynomial` data格式是按照evaluation方式给定的，也就是y值）。

如果找不到就用[质心](https://hackmd.io/@vbuterin/barycentric_evaluation)拉格朗日的方式完整计算一下最终结果。计算时上述式子中 $L_i(x)$ 可以分子分母都乘以 $x-x_i$ 这样对于分子而言将缺失项补齐算是完整的 $M(x)$

$$
\begin{gather*}
M(x) = \prod_{j=0}^{n} x - x_j \\
L_i(x) = \frac{M(x)}{d_i\cdot(x-x_i)} \\
d_i = (x_i - x_1)(x_i - x_2)\cdots(x_i - x_{i-1})(x_i - x_{i+1})\cdots(x_i - x_N)
\end{gather*}
$$

显然 $d_i$ 是可以在外部计算得到的（如果我们把x的选取点设为[root of unity](https://hackmd.io/@vbuterin/barycentric_evaluation)这样的特殊点，就可以进一步大大简化d的计算量。），对于实际的P(x)而言可以看作一个常数。那么这样一来计算P(x)的值只需要把n个x点带入求sum合即可，因此时间复杂度是 $O(n)$。

```go
// evaluateLagrangePolynomial is the implementation for [EvaluateLagrangePolynomial].
// It evaluates a Lagrange polynomial at the given point of evaluation and reports whether the given point was among the points of the domain:

// - The input polynomial is given in evaluation form, that is, a list of evaluations at the points in the domain.

// - The evaluationResult is the result of evaluation at evalPoint.

// - indexInDomain is the index inside domain.Roots, if evalPoint is among them, -1 otherwise
func (domain *Domain) evaluateLagrangePolynomial(poly Polynomial, evalPoint fr.Element) (*fr.Element, int64, error) {
	var indexInDomain int64 = -1

	if domain.Cardinality != uint64(len(poly)) {
		return nil, indexInDomain, ErrPolynomialMismatchedSizeDomain
	}

	// If the evaluation point is in the domain
	// then evaluation of the polynomial in lagrange form
	// is the same as indexing it with the position
	// that the evaluation point is in, in the domain
	indexInDomain = domain.findRootIndex(evalPoint)
	if indexInDomain != -1 {
		return &poly[indexInDomain], indexInDomain, nil
	}
	// denominator
	denom := make([]fr.Element, domain.Cardinality)
	for i := range denom {
		denom[i].Sub(&evalPoint, &domain.Roots[i])
	}
	invDenom := fr.BatchInvert(denom)

	var result fr.Element
	for i := 0; i < int(domain.Cardinality); i++ {
		var num fr.Element
		num.Mul(&poly[i], &domain.Roots[i])

		var div fr.Element
		div.Mul(&num, &invDenom[i])

		result.Add(&result, &div)
	}

	// result * (x^width - 1) * 1/width
	var tmp fr.Element
	tmp.Exp(evalPoint, big.NewInt(0).SetUint64(domain.Cardinality))
	one := fr.One()
	tmp.Sub(&tmp, &one)
	tmp.Mul(&tmp, &domain.CardinalityInv)
	result.Mul(&tmp, &result)

	return &result, indexInDomain, nil
}
```

`computeQuotientPoly`用于计算商多项式。得到h(x)后，我们使用之前setup的secret key来得到h(s)。

有了Proof数据，Verify就比较简单。做一下双线性映射查看等式是否成立即可。

#### Trusted Setup

kzg承诺要想保证安全离不开trusted setup设置。至于为什么要使用trusted setup方式而不是使用IPA这样的无trust设计？在于对于IPA而言，每次计算verify的时间复杂度都是 $O(n)$ ，当DAS进一步完善，以太坊扩容之后n可能会变得比较大，每一次对blob数据的响应都要有n的计算开销，对于终端节点的负担会比较大，不太利于以太坊节点的去中心化。而如果使用trusted setup方式，verify的计算时间复杂度就会降低到 $O(1)$ ，而且这个trusted setup设置过程只需要一次（ceremony），只要保证这个设置过程是安全的，后续就可以一直使用。从综合成本考量来看，采用trusted setup方式会更合适一点。

[trusted setup](https://vitalik.eth.limo/general/2022/03/14/trustedsetup.html)设置对于多项式承诺至关重要，举例来说，如果我们泄露了s，那么其他人就可以重新构造一个虚假的s（比如2s），这样改变原有的多项式系数（也就是要承诺的值），但是依旧可以保持原有的承诺结果不变，这样就造成了欺骗和系统安全问题。

<div align="center">
 <img src = "https://vitalik.eth.limo/images/trustedsetup/ipa_bases.png"/>
  <br />
</div>

在去中心化系统中，我们不能相信任何一个节点，因为任何一个节点都可能隐藏s或者故意泄露s的值。所以整个trusted setup设置过程需要尽可能多的参与者，这样除非整个网络的所有参与节点都作弊，否则只要有一个参与者保持诚实（不泄露s或者参与结束之后立刻遗弃s）那么trusted setup结果就是安全可靠的。

<div align="center">
 <img src = "https://vitalik.eth.limo/images/trustedsetup/multiparticipants.png"/>
  <br />
</div>

但是这样的顺序操作会带来一个问题，就是每一个新的参与者都在前一个参与者基础之上生成一个新的s，但是最后一个参与者如果故意丢弃之前参与者的s只是提交自己的s那样的话setup设置又成了中心化的方式而起不到共同参与去中心化的效果。因此我们还需要对每个参与者生成的setup添加一个验证过程，验证其生产的trusted setup确实是根据前面参与者的内容基础生成对应的新的setup。

生成proof的过程需要用到双线性映射，这也是为什么需要两个群 $G_1$ 和 $G_2$ 的原因。实际的proof证明公式为

$$
e(G_1 \cdot S_i \cdot s, G_2) = e(G_1 \cdot S_i, G_2 \cdot s)
$$

<div align="center">
 <img src = "https://vitalik.eth.limo/images/trustedsetup/verifying.png"/>
  <br />
</div>

其中 $G_1 \cdot S_i$ 为之前生成的trusted secret，s为新参与者的secret。接下来结合[代码](https://github.com/ethereum/research/blob/master/trusted_setup/trusted_setup.py)再具体分析一下。

```python
def generate_one_sided_setup(length, secret, generator=b.G1):
    o = [generator]
    for i in range(1, length):
        o.append(b.multiply(o[-1], secret))
    return o

# Generate a trusted setup with the given secret
def generate_setup(G1_length, G2_length, secret):
    return (
        generate_one_sided_setup(G1_length, secret, b.G1),
        generate_one_sided_setup(G2_length, secret, b.G2),
    )
S1 = generate_setup(8, 4, 42)
```

可以看到，这个函数用于生成trusted setup，数学形式类似这样

$$
\begin{gather*}
[G_1, & G_1 * s, & G_1 * s^2, & \dots, & G_1 * s^{n1-1}] \\
[G_2, & G_2 * s, & G_2 * s^2, & \dots, & G_2 * s^{n2-1}]
\end{gather*}
$$

G1的数组是将来实际要用来做commitment的数组，实际应用是要按照blob length来生成，G2主要是用来生成proof，相对来说其长度可以更小一点，值更灵活。

随后extend setup就是在前一个的基础上生成对应的extend

```python
def extend_one_sided_setup(setup, secret):
    return [
        b.multiply(point, pow(secret, i, b.curve_order))
        for i, point in enumerate(setup)
    ]

# Extends an existing setup with a new secret
def extend_setup(setup, secret):
    return (
        extend_one_sided_setup(setup[0], secret),
        extend_one_sided_setup(setup[1], secret),
    )
    
S1 = generate_setup(8, 4, 42)
S2 = extend_setup(S1, 69)
S3 = extend_setup(S2, 1337)
assert check_setup_eq(S3, generate_setup(8, 4, 42 * 69 * 1337))
```

类似这样：

$$
\begin{gather*}
[G_1, & G_1 * s * t, & G_1 * s^2 * t^2, & \dots, & G_1 * s^{n1-1} * t^{n1-1}] \\
[G_2, & G_2 * s * t, & G_2 * s^2 * t^2, & \dots, & G_2 * s^{n2-1} * t^{n1-1}]
\end{gather*}
$$

随后要对生成的数据做验证。验证的时候可以先对最终结果做一下验证，看最终结果是否满足proof证明关系，我们可以通过将两个群做拆分，再拆出2项来，构造一个双线性映射。首先我们对之前的符号做一个重新定义， $S_0=G_1 \cdot s$
$S_1 = S_0 \cdot s$ ，$T_0 = G_2 \cdot s$  $T_1 = T_0 \cdot s$ 。举一个简单例子

$$
\begin{gather*}
L_1 = r_0S_0 + r_1S_1 \\
L_2 = r_0S_1 + r_1S_2 \\
L_3 = q_0T_0 + q_1T_1 \\
L_4 = q_0T_1 + q_1T_2 \\
\end{gather*}
$$

$$
\begin{gather*}
e(P + Q, R) = e(P, R) \cdot e(Q, R) \\
e(L_1, L_4) = e(r_0S_0, q_0T_1) \cdot e(r_0S_0, q_1T_2) \cdot e(r_1S_1, q_0T_1) \cdot e(r_1S_1, q_1T_2) \\
e(L_1, L_4) = e(L_2, L_3)
\end{gather*}
$$

其中r和q都是随机数。看一下具体代码实现：

```python
def linear_combination(points, coeffs, zero=b.Z1):
    o = zero
    for point, coeff in zip(points, coeffs):
        o = b.add(o, b.multiply(point, coeff))
    return o

def verify_setup(setup):
    G1_setup, G2_setup = setup
    G1_random_coeffs = [random.randrange(2**40) for _ in range(len(G1_setup) - 1)]
    G1_lower = linear_combination(G1_setup[:-1], G1_random_coeffs, b.Z1)
    G1_upper = linear_combination(G1_setup[1:], G1_random_coeffs, b.Z1)
    G2_random_coeffs = [random.randrange(2**40) for _ in range(len(G2_setup) - 1)]
    G2_lower = linear_combination(G2_setup[:-1], G2_random_coeffs, b.Z2)
    G2_upper = linear_combination(G2_setup[1:], G2_random_coeffs, b.Z2)
    return (
        G1_setup[0] == b.G1 and 
        G2_setup[0] == b.G2 and 
        b.pairing(G2_lower, G1_upper) == b.pairing(G2_upper, G1_lower)
    )
```

其中`b.Z1`和`b.Z2`可以看成G1和G2曲线上的0点。`G1_lower`类似公式中的L1，不包含最高项S2，`G1_upper`类似公式的L2，不包含最低项S0。

有了整体验证还需要分步验证，验证生成的secret确实是一步一步生成的，防止最后一个人抛弃前面所有secret做欺骗。证明的机制还是用错位的方式构造双线性映射。

```python
# Generate a proof that can be used to check that a new setup actually is
# an extension of a previous one
def get_extension_proof(previous_setup, secret):
    return (
        # G1*s from the previous setup
        b.G1 if previous_setup is None else previous_setup[0][1],
        # G2*t (the new secret being introduced)
        b.multiply(b.G2, secret)
    )

# Given a sequence of proofs from N participants, verifies that each of those
# participants actually participated
def verify_extension_proofs(final_setup, proofs):
    G1_points = [proof[0] for proof in proofs] + [final_setup[0][1]]
    G2_points = [proof[1] for proof in proofs]
    return G1_points[0] == b.G1 and all(
        b.pairing(G2_points[i], G1_points[i]) == b.pairing(b.G2, G1_points[i+1])
        for i in range(len(G2_points))
    )
proof1 = get_extension_proof(None, 42)
proof2 = get_extension_proof(S1, 69)
proof3 = get_extension_proof(S2, 1337)
assert verify_extension_proofs(S3, [proof1, proof2, proof3])
```

在实际使用的时候会把G1_points提前转换成Lagrange basis的形式（采用fft，x值选择root of unity还可以加速计算）。使用Lagrange basis形式有很多好处，比如实际的blob输入是y值，计算commit只需要算一次就可以了，不需要重新求系数。计算Proof时时间复杂度也只有 $O(n)$。

#### KZG Ceremony

为了生成这个公共的trusted setup，以太坊组织了一场[kzg ceremony](https://ceremony.ethereum.org/)，除了一般通过网站访问提交的方式参与贡献之外，还有很多[别具一格](https://github.com/ethereum/kzg-ceremony/blob/main/special_contributions.md#04---czg-keremony---a-pure-js-kzg-ceremony-client)的贡献方式，大大提高了secret的安全性和多样性。

