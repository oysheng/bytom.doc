### txvm区块结构
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

### txvm交易结构
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
	contract  *contract     //调用的合约栈
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