### txvm区块结构
txvm目前采用了pos共识的模式产生新块，txvm区块由3个部分组成：区块头（BlockHeader）、交易列表（Transactions）和区块签名（Arguments）。

txvm区块头包含了3棵状态树根：交易树（merkle tree）、合约树（patricia tree）和 nonce树（patricia tree）。
- 交易树 用于检测交易是否在该区块中存在
- 合约树 用于检测交易合约是否存在，是一个全局的世界状态树，如果合约被花费则从树中减去，如果产生新的合约则在树中加上
- nonce树 用于检测交易合约是否在之前的区块(blockID)中存在并匹配，也是一个全局的世界状态树，主要用于防止交易输入伪造

Predicate包含了出块公钥信息（Pubkeys），用于标识出块人信息。

```go
type Block struct {
	*UnsignedBlock
	Arguments []interface{}
}

// UnsignedBlock describes a block with its transactions but no signatures (predicate args).
type UnsignedBlock struct {
	*BlockHeader
	Transactions []*Tx
}

// BlockHeader is the header of a Block: everything except the block's
// transactions and predicate args.
type BlockHeader struct {
	Version          uint64     `protobuf:"varint,1,opt,name=version" json:"version,omitempty"`
	Height           uint64     `protobuf:"varint,2,opt,name=height" json:"height,omitempty"`
	PreviousBlockId  *Hash      `protobuf:"bytes,3,opt,name=previous_block_id,json=previousBlockId" json:"previous_block_id,omitempty"`
	TimestampMs      uint64     `protobuf:"varint,4,opt,name=timestamp_ms,json=timestampMs" json:"timestamp_ms,omitempty"`
	Runlimit         int64      `protobuf:"varint,5,opt,name=runlimit" json:"runlimit,omitempty"`
	RefsCount        int64      `protobuf:"varint,6,opt,name=refs_count,json=refsCount" json:"refs_count,omitempty"`
	TransactionsRoot *Hash      `protobuf:"bytes,7,opt,name=transactions_root,json=transactionsRoot" json:"transactions_root,omitempty"`
	ContractsRoot    *Hash      `protobuf:"bytes,8,opt,name=contracts_root,json=contractsRoot" json:"contracts_root,omitempty"`
	NoncesRoot       *Hash      `protobuf:"bytes,9,opt,name=nonces_root,json=noncesRoot" json:"nonces_root,omitempty"`
	NextPredicate    *Predicate `protobuf:"bytes,10,opt,name=next_predicate,json=nextPredicate" json:"next_predicate,omitempty"`
	// Fields added by future versions of the protocol.
	ExtraFields []*DataItem `protobuf:"bytes,11,rep,name=extra_fields,json=extraFields" json:"extra_fields,omitempty"`
}

// Predicate contains the quorum and pubkeys needed to authenticate a block.
type Predicate struct {
	Version int64 `protobuf:"varint,1,opt,name=version" json:"version,omitempty"`
	// These fields apply only when version is 1.
	Quorum  int32    `protobuf:"varint,2,opt,name=quorum" json:"quorum,omitempty"`
	Pubkeys [][]byte `protobuf:"bytes,3,rep,name=pubkeys,proto3" json:"pubkeys,omitempty"`
	// Fields for predicate versions other than 1.
	OtherFields []*DataItem `protobuf:"bytes,4,rep,name=other_fields,json=otherFields" json:"other_fields,omitempty"`
}
```

### txvm区块链Chain结构
BlockBuilder用于构建新的区块，其中snapshot用于保存合约和nonce的世界状态信息，在打包区块的时候，根据已经完成验证的交易CommitmentsTx进行相关状态的更新。

```go
// Chain provides a complete, minimal blockchain database. It
// delegates the underlying storage to other objects, and uses
// validation logic from package validation to decide what
// objects can be safely stored.
type Chain struct {
	InitialBlockHash bc.Hash
	bb               *BlockBuilder

	state struct {
		cond     sync.Cond // protects height, block, snapshot
		height   uint64
		snapshot *state.Snapshot // current only if leader
	}
	store Store

	lastQueuedSnapshotHeight uint64 // atomic access only
	blocksPerSnapshot        uint64
	pendingSnapshots         chan *state.Snapshot
}

type BlockBuilder struct {
	Version        uint64
	MaxNonceWindow time.Duration
	MaxBlockWindow int64
	MaxBlockTxs    int

	snapshot    *state.Snapshot
	txs         []*bc.CommitmentsTx
	timestampMS uint64
	runlimit    int64
}

// Snapshot contains a blockchain's state.
//
// TODO: consider making type Snapshot truly immutable.  We already
// handle it that way in many places (with explicit calls to Copy to
// get the right behavior).  PruneNonces and the Apply functions would
// have to produce new Snapshots rather than updating Snapshots in
// place.
type Snapshot struct {
	ContractsTree *patricia.Tree
	NonceTree     *patricia.Tree

	Header         *bc.BlockHeader
	InitialBlockID bc.Hash
	RefIDs         []bc.Hash
}
```

### txvm交易结构
txvm交易中RawTx是交易的基础信息，其中RawTx中Program是根据交易输入的Action结构构造出来的，执行虚拟机的时候会将相关的状态信息保存到对应的执行Action（Inputs、Issuances、Outputs和Retirements）中，而RawTx中的Version用于区分虚拟机版本，runlimit用于解决图灵完备的虚拟机的停机问题。除了RawTx之外，结构中的其他字段都是执行txvm虚拟机时的保存的状态信息，交易正常结束之后，会将对应日志log保存到Log结构中。 

```go
// Tx contains the input to an instance of the txvm virtual machine,
// plus parsed copies of its various side effects.
type Tx struct {
	RawTx
	Finalized bool
	ID        Hash
	Log       []txvm.Tuple

	// Used in protocol validation and state updates
	Contracts  []Contract
	Timeranges []Timerange
	Nonces     []Nonce
	Anchor     []byte

    // the composition of transactions with actions
	Inputs      []Input
	Issuances   []Issuance
	Outputs     []Output
	Retirements []Retirement
}

// RawTx is a raw transaction, before processing through txvm.
type RawTx struct {
	Version  int64  `protobuf:"varint,1,opt,name=version" json:"version,omitempty"`
	Runlimit int64  `protobuf:"varint,2,opt,name=runlimit" json:"runlimit,omitempty"`
	Program  []byte `protobuf:"bytes,3,opt,name=program,proto3" json:"program,omitempty"`
}

// ======== state change ========
// Contract contains the ID of an input or an output from the txvm
// log. The Type field tells which kind of contract it is (InputType or OutputType).
type Contract struct {
	Type int
	ID   Hash
}

// Timerange is a parsed timerange-typed txvm log entry.
type Timerange struct {
	MinMS, MaxMS int64
}

// Nonce is a parsed nonce-typed txvm log entry.
type Nonce struct {
	ID      Hash
	BlockID Hash
	ExpMS   uint64
}

// ======== transaction actions ========
// Input is a parsed input-typed txvm log entry, plus information
// derived from stack introspection during execution.
type Input struct {
	ID      Hash
	Seed    Hash
	Stack   []txvm.Data
	Program []byte
	LogPos  int
}

// Output is a parsed output-typed txvm log entry, plus information
// derived from stack introspection during execution.
type Output struct {
	ID      Hash
	Seed    Hash
	Stack   []txvm.Data
	Program []byte
	LogPos  int
}

// Issuance is a parsed issuance-typed txvm log entry.
type Issuance struct {
	Seed    Hash
	Stack   []txvm.Data
	Program []byte
	Amount  int64
	AssetID Hash
	Anchor  []byte
	LogPos  int
}

// Retirement is a parsed retirement-typed txvm log entry.
type Retirement struct {
	Amount  int64
	AssetID Hash
	Anchor  []byte
	LogPos  int
}
```

### txvm虚拟机结构
txvm虚拟机结构主要包含两个栈：参数栈（argstack）和合约栈（contract），参数栈保存外部输入的参数信息，而合约栈主要把保存当前调用合约的相关状态信息。此外，完成合约虚拟机的执行后，其结构中包含的字段 TxID、Log 和 Finalized 都会保存到txvm的交易中。 

```go
type VM struct {
	// Config/setup fields
	txVersion         int64
	runlimit          int64
	extension         bool
	stopAfterFinalize bool
	onFinalize        []func(*VM)
	onLog             []func(*VM)
	beforeStep        []func(*VM)
	afterStep         []func(*VM)
	onExit            []func(*VM)

	// Runtime fields
	argstack  stack         //执行的参数栈. //type stack []Item
	run       run 
	runstack  []run
	unwinding bool
	contract  *contract     //当前调用的合约栈
	caller    []byte
	data      []byte
	opcode    byte

	// Results
	TxID [32]byte   // TxID is the unique id of the transaction. It is only set if Finalized is true.
	Log []Tuple     // Log is the record of the transaction's effects.
	Finalized bool  // Finalized is true if and only if the finalize instruction was executed.
}

type contract struct {
	typecode byte
	seed     []byte
	program  []byte
	stack    stack
}
```