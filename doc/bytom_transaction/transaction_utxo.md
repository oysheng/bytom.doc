# bytom交易说明（UTXO用户自己管理模式）
该部分主要针对用户自己管理私钥和地址，并通过utxo来构建和发送交易。

* [1.创建私钥和公钥](#1.创建私钥和公钥)
* [2.根据公钥创建接收对象](#2.根据公钥创建接收对象)
* [3.找到可花费的`utxo`](#3.找到可花费的utxo)
* [4.通过`utxo`构造交易](#4.通过utxo构造交易)
* [5.组合交易的`input`和`output`构成交易模板](#5.组合交易的input和output构成交易模板)
* [6.对构造的交易进行签名](#6.对构造的交易进行签名)
* [7.提交交易上链](#7.提交交易上链)


*注意事项*:

以下步骤以及功能改造仅供参考，具体代码实现需要用户根据实际情况进行调试，具体可以参考单元测试案例代码[blockchain/txbuilder/txbuilder_test.go#L255](https://github.com/Bytom/bytom/blob/master/blockchain/txbuilder/txbuilder_test.go#L255)

## 1.创建私钥和公钥
该部分功能可以参考代码[crypto/ed25519/chainkd/util.go#L11](https://github.com/Bytom/bytom/blob/master/crypto/ed25519/chainkd/util.go#L11)，可以通过 `NewXKeys(nil)` 创建主私钥和主公钥 
```go
func NewXKeys(r io.Reader) (xprv XPrv, xpub XPub, err error) {
	xprv, err = NewXPrv(r)
	if err != nil {
		return
	}
	return xprv, xprv.XPub(), nil
}
```

## 2.根据公钥创建接收对象
接收对象包含两种形式：`address`形式和`program`形式，两者是一一对应的，任选其一即可。其中创建单签地址参考代码[account/accounts.go#L267](https://github.com/Bytom/bytom/blob/master/account/accounts.go#L267)进行相应改造为：
```go
func (m *Manager) createP2PKH(xpub chainkd.XPub) (*CtrlProgram, error) {
	pubKey := xpub.PublicKey()
	pubHash := crypto.Ripemd160(pubKey)

	// TODO: pass different params due to config
	address, err := common.NewAddressWitnessPubKeyHash(pubHash, &consensus.ActiveNetParams)
	if err != nil {
		return nil, err
	}

	control, err := vmutil.P2WPKHProgram([]byte(pubHash))
	if err != nil {
		return nil, err
	}

	return &CtrlProgram{
		Address:        address.EncodeAddress(),
		ControlProgram: control,
	}, nil
}
```

创建多签地址参考代码[account/accounts.go#L294](https://github.com/Bytom/bytom/blob/master/account/accounts.go#L294)进行相应改造为：(quorum指的是多签地址需要的验证的个数，比如说3-2多签地址，指的是3个主公钥，需要两个签名才能验证通过)
```go
func (m *Manager) createP2SH(xpubs []chainkd.XPub, quorum int) (*CtrlProgram, error) {
	derivedPKs := chainkd.XPubKeys(xpubs)
	signScript, err := vmutil.P2SPMultiSigProgram(derivedPKs, quorum)
	if err != nil {
		return nil, err
	}
	scriptHash := crypto.Sha256(signScript)

	// TODO: pass different params due to config
	address, err := common.NewAddressWitnessScriptHash(scriptHash, &consensus.ActiveNetParams)
	if err != nil {
		return nil, err
	}

	control, err := vmutil.P2WSHProgram(scriptHash)
	if err != nil {
		return nil, err
	}

	return &CtrlProgram{
		Address:        address.EncodeAddress(),
		ControlProgram: control,
	}, nil
}
```

## 3.找到可花费的utxo
找到可花费的utxo，其实就是找到接收地址或接收`program`是你自己的`unspend_output`。其中utxo的结构为：（参考代码[account/reserve.go#L39](https://github.com/Bytom/bytom/blob/master/account/reserve.go#L39)）
```go
// UTXO describes an individual account utxo.
type UTXO struct {
	OutputID bc.Hash
	SourceID bc.Hash

	// Avoiding AssetAmount here so that new(utxo) doesn't produce an
	// AssetAmount with a nil AssetId.
	AssetID bc.AssetID
	Amount  uint64

	SourcePos      uint64
	ControlProgram []byte

	AccountID           string
	Address             string
	ControlProgramIndex uint64
	ValidHeight         uint64
	Change              bool
}
```

涉及utxo构造交易的相关字段说明如下：
- `SourceID` 前一笔关联交易的mux_id, 根据该ID可以定位到前一笔交易的output
- `AssetID` utxo的资产ID
- `Amount` utxo的资产数目
- `SourcePos` 该utxo在前一笔交易的output的位置
- `ControlProgram` utxo的接收program
- `Address` utxo的接收地址

上述这些utxo的字段信息可以从`get-block`接口返回结果的transaction中找到，其相关的结构体如下：（参考代码[api/block_retrieve.go#L26](https://github.com/Bytom/bytom/blob/master/api/block_retrieve.go#L26)）
```go
// BlockTx is the tx struct for getBlock func
type BlockTx struct {
	ID         bc.Hash                  `json:"id"`
	Version    uint64                   `json:"version"`
	Size       uint64                   `json:"size"`
	TimeRange  uint64                   `json:"time_range"`
	Inputs     []*query.AnnotatedInput  `json:"inputs"`
	Outputs    []*query.AnnotatedOutput `json:"outputs"`
	StatusFail bool                     `json:"status_fail"`
	MuxID      bc.Hash                  `json:"mux_id"`
}

//AnnotatedOutput means an annotated transaction output.
type AnnotatedOutput struct {
	Type            string             `json:"type"`
	OutputID        bc.Hash            `json:"id"`
	TransactionID   *bc.Hash           `json:"transaction_id,omitempty"`
	Position        int                `json:"position"`
	AssetID         bc.AssetID         `json:"asset_id"`
	AssetAlias      string             `json:"asset_alias,omitempty"`
	AssetDefinition *json.RawMessage   `json:"asset_definition,omitempty"`
	Amount          uint64             `json:"amount"`
	AccountID       string             `json:"account_id,omitempty"`
	AccountAlias    string             `json:"account_alias,omitempty"`
	ControlProgram  chainjson.HexBytes `json:"control_program"`
	Address         string             `json:"address,omitempty"`
}
```

utxo跟get-block返回结果的字段对应关系如下：
```js
`SourceID`       - `json:"mux_id"`
`AssetID`        - `json:"asset_id"`
`Amount`         - `json:"amount"`
`SourcePos`      - `json:"position"`
`ControlProgram` - `json:"control_program"`
`Address`        - `json:"address,omitempty"`
```

## 4.通过`utxo`构造交易
通过utxo构造交易就是使用spend_account_unspent_output的方式来花费指定的utxo。

第一步，通过`utxo`构造交易输入`TxInput`和签名需要的数据信息`SigningInstruction`，该部分功能可以参考代码[account/builder.go#L169](https://github.com/Bytom/bytom/blob/master/account/builder.go#L169)进行相应改造为:
```go
// UtxoToInputs convert an utxo to the txinput
func UtxoToInputs(xpubs []chainkd.XPub, quorum int， u *UTXO) (*types.TxInput, *txbuilder.SigningInstruction, error) {
	txInput := types.NewSpendInput(nil, u.SourceID, u.AssetID, u.Amount, u.SourcePos, u.ControlProgram)
	sigInst := &txbuilder.SigningInstruction{}

	if u.Address == "" {
		return txInput, sigInst, nil
	}

	address, err := common.DecodeAddress(u.Address, &consensus.ActiveNetParams)
	if err != nil {
		return nil, nil, err
	}

    sigInst.AddRawWitnessKeys(xpubs, nil, quorum)
	switch address.(type) {
	case *common.AddressWitnessPubKeyHash:
		derivedPK := xpubs[0].PublicKey()
		sigInst.WitnessComponents = append(sigInst.WitnessComponents, txbuilder.DataWitness([]byte(derivedPK)))

	case *common.AddressWitnessScriptHash:
		derivedPKs := chainkd.XPubKeys(xpubs)
		script, err := vmutil.P2SPMultiSigProgram(derivedPKs, quorum)
		if err != nil {
			return nil, nil, err
		}
		sigInst.WitnessComponents = append(sigInst.WitnessComponents, txbuilder.DataWitness(script))

	default:
		return nil, nil, errors.New("unsupport address type")
	}

	return txInput, sigInst, nil
}
```

第二步，通过`utxo`构造交易输出`TxOutput`
该部分功能可以参考代码[protocol/bc/types/txoutput.go#L20](https://github.com/Bytom/bytom/blob/master/protocol/bc/types/txoutput.go#L20):
```go
// NewTxOutput create a new output struct
func NewTxOutput(assetID bc.AssetID, amount uint64, controlProgram []byte) *TxOutput {
	return &TxOutput{
		AssetVersion: 1,
		OutputCommitment: OutputCommitment{
			AssetAmount: bc.AssetAmount{
				AssetId: &assetID,
				Amount:  amount,
			},
			VMVersion:      1,
			ControlProgram: controlProgram,
		},
	}
}
```

## 5.组合交易的input和output构成交易模板
通过上面已经生成的交易信息构造交易`txbuilder.Template`，该部分功能可以参考[blockchain/txbuilder/builder.go#L92](https://github.com/Bytom/bytom/blob/master/blockchain/txbuilder/builder.go#L92)进行改造为:
```go
type InputAndSigInst struct {
	input *types.TxInput
	sigInst *SigningInstruction
}

// Build build transactions with template
func BuildTx(inputs []InputAndSigInst, outputs []*types.TxOutput) (*Template, *types.TxData, error) {
	tpl := &Template{}
	tx := &types.TxData{}
	// Add all the built outputs.
	tx.Outputs = append(tx.Outputs, outputs...)

	// Add all the built inputs and their corresponding signing instructions.
	for p, in := range inputs {
		// Empty signature arrays should be serialized as empty arrays, not null.
		in.sigInst.Position = uint32(p)
		if in.sigInst.WitnessComponents == nil {
			in.sigInst.WitnessComponents = []witnessComponent{}
		}
		tpl.SigningInstructions = append(tpl.SigningInstructions, in.sigInst)
		tx.Inputs = append(tx.Inputs, in.input)
	}

	tpl.Transaction = types.NewTx(*tx)
	return tpl, tx, nil
}
```

## 6.对构造的交易进行签名
账户模型是根据密码找到对应的私钥对交易进行签名，这里用户可以直接使用私钥对交易进行签名，可以参考签名代码[blockchain/txbuilder/txbuilder.go#L82](https://github.com/Bytom/bytom/blob/master/blockchain/txbuilder/txbuilder.go#L82)进行改造为:（以下改造仅支持单签交易，多签交易用户可以参照该示例进行改造）
```go
// Sign will try to sign all the witness
func Sign(tpl *Template, xprv chainkd.XPrv) error {
	for i, sigInst := range tpl.SigningInstructions {
		for _, wc := range sigInst.WitnessComponents {
			switch sw := wc.(type) {
			case *RawTxSigWitness:
				h := tpl.Hash(uint32(i)).Byte32()
				sig := xprv.Sign(h[:])
				sw.Sigs = append(sw.Sigs, sig)
			}
		}
	}
	return materializeWitnesses(tpl)
}
```

多签的方式可以参考以下修改：（xprvs需要跟签名的个数Quorum相同，另外注意一下多签的顺序）
```go
func Sign(tpl *Template, xprvs []chainkd.XPrv) error {
	for i, sigInst := range tpl.SigningInstructions {
		for _, wc := range sigInst.WitnessComponents {
			switch sw := wc.(type) {
			case *RawTxSigWitness:
				h := tpl.Hash(uint32(i)).Byte32()
				for k, xprv := range xprvs {
					if len(sw.Sigs[k]) > 0 {
						// Already have a signature for this key
						continue
					}
					sig := xprv.Sign(h[:])
					sw.Sigs[k] = sig
					break  // the one private sign this tx only once
				}
			}
		}
	}
	return materializeWitnesses(tpl)
}
```

## 7.提交交易上链
该步骤无需更改任何内容，直接参照wiki中提交交易的API[submit-transaction](https://github.com/Bytom/bytom/wiki/API-Reference#submit-transaction)的功能即可
