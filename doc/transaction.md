# bytom交易说明（账户管理模式）
该部分主要针对用户使用bytom自带的账户模式发送交易

* [1、构建交易](#1、构建交易)

  [action简介](#action简介)
  - [issue](#issue)
  - [spend_account](#spend_account)
  - [spend_account_unspent_output](#spend_account_unspent_output)
  - [control_address](#control_address)
  - [control_program](#control_program)
  - [retire](#retire)

  [估算手续费](#估算手续费)
* [2、签名交易](#2、签名交易)
* [3、提交交易](#3、提交交易)

## 1、构建交易
API接口 build-transaction，代码[api/transact.go#L120](https://github.com/Bytom/bytom/blob/master/api/transact.go#L120)

以标准的非BTM资产转账交易为例，资产ID为全F表示BTM资产，在该示例中BTM资产仅作为手续费，该交易表示花费99个特定的资产到指定地址中。其中构建交易的输入请求json格式如下：
```js
{
  "base_transaction": null,
  "actions": [
    {
      "account_id": "0ER7MEFGG0A02",
      "amount": 20000000,
      "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
      "type": "spend_account"
    },
    {
      "account_id": "0ER7MEFGG0A02",
      "amount": 99,
      "asset_id": "42275aacbeda1522cd41580f875c3c452daf5174b17ba062bf0ab71a568c123f",
      "type": "spend_account"
    },
    {
      "amount": 99,
      "asset_id": "42275aacbeda1522cd41580f875c3c452daf5174b17ba062bf0ab71a568c123f",
      "address": "sm1qxe4jwhkekgnxkezu7xutu5gqnnpmyc8ppq98me",
      "type": "control_address"
    }
  ],
  "ttl": 0,
  "time_range": 0
}
```

对应源代码的请求对象如下：
```go
// BuildRequest is main struct when building transactions
type BuildRequest struct {
	Tx        *types.TxData            `json:"base_transaction"`
	Actions   []map[string]interface{} `json:"actions"`
	TTL       json.Duration            `json:"ttl"`
	TimeRange uint64                   `json:"time_range"`
}
```

结构字段说明如下：
- `Tx` 交易的`TxData`部分，该字段为预留字段，为空即可
- `TTL` 构建交易的生存时间（单位为毫秒），意味着在该时间范围内，已经缓存的utxo不能用于再一次build交易，除非剩余的utxo足以构建一笔新的交易，否则会报错。当`ttl`为0时会被默认设置为600s，即5分钟
- `TimeRange` 时间戳，意味着该交易将在该时间戳（区块高度）之后不会被提交上链，为了防止交易在网络中传输延迟而等待太久时间，如果交易没有在特定的时间范围内被打包，该交易便会自动失效
- `Actions` 交易的`actions`结构，所有的交易都是由action构成的，`map`类型的`interface{}`保证了action类型的可扩展性。其中action中必须包含type字段，用于区分不同的action类型，`action`主要包含`input`和`output`两种类型，其详细介绍如下：
  - `input action` 类型：
      - issue 发行资产
      - spend_account 以账户的模式花费utxo
      - spend_account_unspent_output 直接花费指定的utxo
  - `output action` 类型：
      - control_address 接收方式为地址模式
      - control_program 接收方式为（program）合约模式
      - retire 销毁资产

*注意事项*：
- 一个交易必须至少包含一个input和output（coinbase交易除外，因为coinbase交易是由系统产生，故不在此加以描述），否则交易将会报错。
- 除了BTM资产（所有交易都是以BTM资产作为手续费）之外，其他资产在构建input和output时，所有输入和输出的资产总和必须相等，否则交易会报出输入输出不平衡的错误信息。
- 交易的手续费： 所有inputs的BTM资产数量 - 所有outputs的BTM资产数量
- 交易中的资产amount都是neu为单位的，BTM的单位换算如下：1 BTM = 1000 mBTM = 100000000 neu

---

### action简介

下面对构建交易时用到的各种`action`类型进行详细说明：

#### issue
`issueAction`结构体源代码如下：
```go
type issueAction struct {
	assets *Registry
	bc.AssetAmount
}

type AssetAmount struct {
	AssetId *AssetID `protobuf:"bytes,1,opt,name=asset_id,json=assetId" json:"asset_id,omitempty"`
	Amount  uint64   `protobuf:"varint,2,opt,name=amount" json:"amount,omitempty"`
}
```

结构字段说明如下：
- `assets` 主要用于资产的管理，无需用户设置参数
- `AssetAmount` 表示用户需要发行的资产ID和对应的资产数目，这里的`AssetID`需要通过`create-asset`创建，并且这里不能使用`BTM`的资产ID

`issueAction`的`json`格式为：
```js
{
  "amount": 100000000,
  "asset_id": "3152a15da72be51b330e1c0f8e1c0db669269809da4f16443ff266e07cc43680",
  "type": "issue"
}
```

例如发行一笔资产的交易示例如下：
（该交易表示发行数量为`900000000`个`assetID`的`42275aacbeda1522cd41580f875c3c452daf5174b17ba062bf0ab71a568c123f`的资产到接收地址`sm1qxe4jwhkekgnxkezu7xutu5gqnnpmyc8ppq98me`中, 其中手续费为`20000000`neu的BTM资产）
```js
{
  "base_transaction": null,
  "actions": [
    {
      "account_id": "0ER7MEFGG0A02",
      "amount": 20000000,
      "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
      "type": "spend_account"
    },
    {
      "amount": 900000000,
      "asset_id": "42275aacbeda1522cd41580f875c3c452daf5174b17ba062bf0ab71a568c123f",
      "type": "issue"
    },
    {
      "amount": 900000000,
      "asset_id": "42275aacbeda1522cd41580f875c3c452daf5174b17ba062bf0ab71a568c123f",
      "address": "sm1qxe4jwhkekgnxkezu7xutu5gqnnpmyc8ppq98me",
      "type": "control_address"
    }
  ],
  "ttl": 0,
  "time_range": 0
}
```

---

#### spend_account
`spendAction`结构体源代码如下：
```go
type spendAction struct {
	accounts *Manager
	bc.AssetAmount
	AccountID   string  `json:"account_id"`
	ClientToken *string `json:"client_token"`
}

type AssetAmount struct {
	AssetId *AssetID `protobuf:"bytes,1,opt,name=asset_id,json=assetId" json:"asset_id,omitempty"`
	Amount  uint64   `protobuf:"varint,2,opt,name=amount" json:"amount,omitempty"`
}
```

结构字段说明如下：
- `accounts` 主要用于账户的管理，无需用户设置参数
- `AccountID` 表示需要花费资产的账户ID
- `AssetAmount` 表示花费的资产ID和对应的资产数目
- `ClientToken` 表示Reserve用户UTXO的限制条件，目前不填或为空即可

`spendAction`的`json`格式为：
```js
{
  "account_id": "0BF63M2U00A04",
  "amount": 2000000000,
  "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
  "type": "spend_account"
}
```

例如转账一笔资产的交易示例如下：
（该交易表示通过账户的方式转账`100000000`neu的BTM资产到地址`sm1qxe4jwhkekgnxkezu7xutu5gqnnpmyc8ppq98me`中, 其中手续费`20000000`neu = 输入BTM资产数量 - 输出BTM资产数量）
```js
{
  "base_transaction": null,
  "actions": [
    {
      "account_id": "0ER7MEFGG0A02",
      "amount": 120000000,
      "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
      "type": "spend_account"
    },
    {
      "amount": 100000000,
      "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
      "address": "sm1qxe4jwhkekgnxkezu7xutu5gqnnpmyc8ppq98me",
      "type": "control_address"
    }
  ],
  "ttl": 0,
  "time_range": 0
}
```

---

#### spend_account_unspent_output
`spendUTXOAction`结构体源代码如下：
```go
type spendUTXOAction struct {
	accounts *Manager
	OutputID *bc.Hash `json:"output_id"`
	ClientToken *string `json:"client_token"`
}
```

结构字段说明如下：
- `accounts` 主要用于账户的管理，无需用户设置参数
- `OutputID` 表示需要花费的UTXO的ID，可以根据`list-unspent-outputs`查询可用的UTXO，其中`OutputID`对应该API返回结果的`id`字段
- `ClientToken` 表示Reserve用户UTXO的限制条件，目前不填或为空即可

`spendUTXOAction`的`json`格式为：
```js
{
  "type": "spend_account_unspent_output",
  "output_id": "58f29f0f85f7bd2a91088bcbe536dee41cd0642dfb1480d3a88589bdbfd642d9"
}
```

例如通过花费UTXO的方式转账一笔资产的交易示例如下：
（该交易表示通过直接花费UTXO的方式转账`100000000`neu的BTM资产到地址`sm1qxe4jwhkekgnxkezu7xutu5gqnnpmyc8ppq98me`中, 其中手续费 = 输入BTM资产的UTXO值 - 输出BTM资产数量）
```js
{
  "base_transaction": null,
  "actions": [
    {
      "output_id": "58f29f0f85f7bd2a91088bcbe536dee41cd0642dfb1480d3a88589bdbfd642d9",
      "type": "spend_account_unspent_output"
    },
    {
      "amount": 100000000,
      "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
      "address": "sm1qxe4jwhkekgnxkezu7xutu5gqnnpmyc8ppq98me",
      "type": "control_address"
    }
  ],
  "ttl": 0,
  "time_range": 0
}
```

---

#### control_address
`controlAddressAction`结构体源代码如下：
```go
type controlAddressAction struct {
	bc.AssetAmount
	Address string `json:"address"`
}

type AssetAmount struct {
	AssetId *AssetID `protobuf:"bytes,1,opt,name=asset_id,json=assetId" json:"asset_id,omitempty"`
	Amount  uint64   `protobuf:"varint,2,opt,name=amount" json:"amount,omitempty"`
}
```

结构字段说明如下：
- `Address` 表示接收资产的地址，可以根据 `create-account-receiver` API接口创建地址
- `AssetAmount` 表示接收的资产ID和对应的资产数目

`controlAddressAction`的`json`格式为：
```js
{
  "amount": 100000000,
  "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
  "address": "bm1q50u3z8empm5ke0g3ngl2t3sqtr6sd7cepd3z68",
  "type": "control_address"
}
```

例如转账一笔资产的交易示例如下：
（该交易表示通过账户的方式转账`100000000`neu的BTM资产到地址`sm1qxe4jwhkekgnxkezu7xutu5gqnnpmyc8ppq98me`中, 其中`control_address`类型表示以地址作为接收方式）
```js
{
  "base_transaction": null,
  "actions": [
    {
      "account_id": "0ER7MEFGG0A02",
      "amount": 120000000,
      "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
      "type": "spend_account"
    },
    {
      "amount": 100000000,
      "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
      "address": "sm1qxe4jwhkekgnxkezu7xutu5gqnnpmyc8ppq98me",
      "type": "control_address"
    }
  ],
  "ttl": 0,
  "time_range": 0
}
```

---

#### control_program
`controlProgramAction`结构体源代码如下：
```go
type controlProgramAction struct {
	bc.AssetAmount
	Program json.HexBytes `json:"control_program"`
}

type AssetAmount struct {
	AssetId *AssetID `protobuf:"bytes,1,opt,name=asset_id,json=assetId" json:"asset_id,omitempty"`
	Amount  uint64   `protobuf:"varint,2,opt,name=amount" json:"amount,omitempty"`
}
```

结构字段说明如下：
- `Program` 表示接收资产的合约脚本，可以根据 `create-account-receiver` API接口创建接收`program`（返回结果的 `program` 和 `address` 是一一对应的）
- `AssetAmount` 表示接收的资产ID和对应的资产数目

`controlProgramAction`的`json`格式为：
```js
{
  "amount": 100000000,
  "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
  "control_program":"0014a3f9111f3b0ee96cbd119a3ea5c60058f506fb19",
  "type": "control_program"
}
```

例如转账一笔资产的交易示例如下：
（该交易表示通过账户的方式转账`100000000`neu的BTM资产到接收`program`（跟`address`是一一对应的）`0014a3f9111f3b0ee96cbd119a3ea5c60058f506fb19`中, 其中`control_program`类型表示以`program`作为接收方式）
```js
{
  "base_transaction": null,
  "actions": [
    {
      "account_id": "0ER7MEFGG0A02",
      "amount": 120000000,
      "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
      "type": "spend_account"
    },
    {
      "amount": 100000000,
      "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
      "control_program": "0014a3f9111f3b0ee96cbd119a3ea5c60058f506fb19",
      "type": "control_program"
    }
  ],
  "ttl": 0,
  "time_range": 0
}
```

---

#### retire
`retireAction`结构体源代码如下：
```go
type retireAction struct {
	bc.AssetAmount
	Arbitrary json.HexBytes `json:"arbitrary"`
}

type AssetAmount struct {
	AssetId *AssetID `protobuf:"bytes,1,opt,name=asset_id,json=assetId" json:"asset_id,omitempty"`
	Amount  uint64   `protobuf:"varint,2,opt,name=amount" json:"amount,omitempty"`
}
```

结构字段说明如下：
- `AssetAmount` 表示销毁的资产ID和对应的资产数目
- `Arbitrary` 表示任意的附加信息（十六进制字符串数据），可为空

`retireAction`的`json`格式为：
```js
{
  "amount": 900000000,
  "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
  "arbitrary": "77656c636f6d65efbc8ce6aca2e8bf8ee69da5e588b0e58e9fe5ad90e4b896e7958c",
  "type": "retire"
}
```

例如销毁一笔资产的交易示例如下：
（该交易表示通过账户的方式将`1`neu的BTM资产销毁并添加了附加信息, 其中`retire`表示销毁指定数量的资产）
```js
{
  "base_transaction": null,
  "actions": [
    {
      "account_id": "0ER7MEFGG0A02",
      "amount": 900000000,
      "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
      "type": "spend_account"
    },
    {
      "amount": 1,
      "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
      "arbitrary": "77656c636f6d65efbc8ce6aca2e8bf8ee69da5e588b0e58e9fe5ad90e4b896e7958c",
      "type": "retire"
    }
  ],
  "ttl": 0,
  "time_range": 0
}
```

---

`build-transaction`的输入构造完成之后，便可以通过http的调用方式进行发送交易，构建交易请求成功之后返回的json结果如下：
```js
{
  "allow_additional_actions": false,
  "raw_transaction": "070100020161015f1190c60818b4aff485c865113c802942f29ce09088cae1b117fc4c8db2292212ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff8099c4d599010001160014a86c83ee12e6d790fb388345cc2e2b87056a077301000161015fb018097c4040c8dd86d95611a13c24f90d4c9d9d06b25f5c9ed0556ac8abd73442275aacbeda1522cd41580f875c3c452daf5174b17ba062bf0ab71a568c123f80a094a58d1d0101160014068840e56af74038571f223b1c99f1b60caaf456010003013effffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff80bfffcb9901011600140b946646626c55a52a325c8bb48de792284d9b7200013e42275aacbeda1522cd41580f875c3c452daf5174b17ba062bf0ab71a568c123f9d9f94a58d1d01160014c8b4391bab4923a83b955170d24ee4ca5b6ec3fb00013942275aacbeda1522cd41580f875c3c452daf5174b17ba062bf0ab71a568c123f6301160014366b275ed9b2266b645cf1b8be51009cc3b260e100",
  "signing_instructions": [
    {
      "position": 0,
      "witness_components": [
        {
          "keys": [
            {
              "derivation_path": [
                "010100000000000000",
                "0100000000000000"
              ],
              "xpub": "de0db655c091b2838ccb6cddb675779b0a9a4204b122e61699b339867dd10eb0dbdc926882ff6dd75c099c181c60d63eab0033a4b0a4d0a8c78079e39d7ad1d8"
            }
          ],
          "quorum": 1,
          "signatures": null,
          "type": "raw_tx_signature"
        },
        {
          "type": "data",
          "value": "d174db6506e35f2decb5be148c2984bfd0f6c67f043365bf642d1af387c04fd5"
        }
      ]
    },
    {
      "position": 1,
      "witness_components": [
        {
          "keys": [
            {
              "derivation_path": [
                "010100000000000000",
                "0800000000000000"
              ],
              "xpub": "de0db655c091b2838ccb6cddb675779b0a9a4204b122e61699b339867dd10eb0dbdc926882ff6dd75c099c181c60d63eab0033a4b0a4d0a8c78079e39d7ad1d8"
            }
          ],
          "quorum": 1,
          "signatures": null,
          "type": "raw_tx_signature"
        },
        {
          "type": "data",
          "value": "05cdbcc705f07ad87521835bbba226ad7b430cc24e5e3f008edbe61540535419"
        }
      ]
    }
  ]
}
```

对应响应对象的源代码如下：
```go
// Template represents a partially- or fully-signed transaction.
type Template struct {
	Transaction         *types.Tx             `json:"raw_transaction"`
	SigningInstructions []*SigningInstruction `json:"signing_instructions"`

	// AllowAdditional affects whether Sign commits to the tx sighash or
	// to individual details of the tx so far. When true, signatures
	// commit to tx details, and new details may be added but existing
	// ones cannot be changed. When false, signatures commit to the tx
	// as a whole, and any change to the tx invalidates the signature.
	AllowAdditional bool `json:"allow_additional_actions"`
}
```

结构字段说明如下：
- `Transaction` 交易相关信息，该字段包含`TxData`和`bc.Tx`两个部分：
  - `TxData` 表示给用户展示的交易数据部分，该部分对用户可见
    - `Version` 交易版本
    - `SerializedSize` 交易序列化之后的size
    - `TimeRange` 交易提交上链的最大时间戳（区块高度）（主链区块高度到达该时间戳（区块高度）之后，如果交易没有被提交上链，该交易便会失效）
    - `Inputs` 交易输入
    - `Outputs` 交易输出
  - `bc.Tx` 表示系统中处理交易用到的转换结构，该部分对用户不可见，故不做详细描述
- `SigningInstructions` 交易的签名信息
  - `Position` 对`input action`签名的位置
  - `WitnessComponents` 对`input action`签名需要的数据信息，其中build交易的`signatures`为`null`，表示没有签名; 如果交易签名成功，则该字段会存在签名信息。该字段是一个interface接口，主要包含3种不同的类型：
    - `SignatureWitness` 对交易模板`Template`中交易`input action`位置的合约program进行哈希，然后对hash值进行签名
      - `signatures` （数组类型）交易的签名，`sign-transaction`执行完成之后才会有值存在
      - `keys` （数组类型）包含主公钥`xpub`和派生路径`derivation_path`，通过它们可以在签名阶段找到对应的派生私钥`child_xprv`，然后使用派生私钥进行签名
      - `quorum` 账户`key` 的个数，必须和上面的`keys`的长度相等。如果`quorum` 等于1，则表示单签账户，否则为多签账户
      - `program` 签名的数据部分，`program`的hash值作为签名数据。如果`program`为空，则会根据当前交易ID和对应action位置的InputID两部分生成一个hash，然后把它们作为指令数据自动构造一个`program` 
    - `RawTxSigWitness` 对交易模板`Template`的交易ID和对应`input action`位置的InputID(该字段位于bc.Tx中)进行哈希，然后对hash值进行签名
      - `signatures` （数组类型）交易的签名，`sign-transaction`执行完成之后才会有值存在
      - `keys` （数组类型）包含主公钥`xpub`和派生路径`derivation_path`，通过它们可以在签名阶段找到对应的派生私钥`child_xprv`，然后使用派生私钥进行签名
      - `quorum` 账户`key`的个数，必须和上面的`keys` 的长度相等。如果`quorum` 等于1，则表示单签账户，否则为多签账户
    - `DataWitness` 该类型无需签名，验证合约program的附加数据
- `AllowAdditional` 是否允许交易的附加数据，如果为`true`，则交易的附加数据会添加到交易中，但是不会影响交易的执行的`program`脚本，对签名结果不会造成影响; 如果为`false`，则整个交易作为一个整体进行签名，任何数据的改变将影响整个交易的签名

---

### 估算手续费

估算手续费接口`estimate-transaction-gas`是对`build-transaction`的结果进行手续费的预估，估算的总手续费`total_neu`需要加到`build-transaction`的请求结构中，然后对交易进行签名和提交。其主要流程如下：
```
  build - estimate - build - sign - submit
```

估算手续费的输入请求json格式如下：
```js
{
  "transaction_template": {
    "allow_additional_actions": false,
    "raw_transaction": "070100020161015f1190c60818b4aff485c865113c802942f29ce09088cae1b117fc4c8db2292212ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff8099c4d599010001160014a86c83ee12e6d790fb388345cc2e2b87056a077301000161015fb018097c4040c8dd86d95611a13c24f90d4c9d9d06b25f5c9ed0556ac8abd73442275aacbeda1522cd41580f875c3c452daf5174b17ba062bf0ab71a568c123f80a094a58d1d0101160014068840e56af74038571f223b1c99f1b60caaf456010003013effffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff80bfffcb9901011600140b946646626c55a52a325c8bb48de792284d9b7200013e42275aacbeda1522cd41580f875c3c452daf5174b17ba062bf0ab71a568c123f9d9f94a58d1d01160014c8b4391bab4923a83b955170d24ee4ca5b6ec3fb00013942275aacbeda1522cd41580f875c3c452daf5174b17ba062bf0ab71a568c123f6301160014366b275ed9b2266b645cf1b8be51009cc3b260e100",
    "signing_instructions": [
      {
        "position": 0,
        "witness_components": [
          {
            "keys": [
              {
                "derivation_path": [
                  "010100000000000000",
                  "0100000000000000"
                ],
                "xpub": "de0db655c091b2838ccb6cddb675779b0a9a4204b122e61699b339867dd10eb0dbdc926882ff6dd75c099c181c60d63eab0033a4b0a4d0a8c78079e39d7ad1d8"
              }
            ],
            "quorum": 1,
            "signatures": null,
            "type": "raw_tx_signature"
          },
          {
            "type": "data",
            "value": "d174db6506e35f2decb5be148c2984bfd0f6c67f043365bf642d1af387c04fd5"
          }
        ]
      },
      {
        "position": 1,
        "witness_components": [
          {
            "keys": [
              {
                "derivation_path": [
                  "010100000000000000",
                  "0800000000000000"
                ],
                "xpub": "de0db655c091b2838ccb6cddb675779b0a9a4204b122e61699b339867dd10eb0dbdc926882ff6dd75c099c181c60d63eab0033a4b0a4d0a8c78079e39d7ad1d8"
              }
            ],
            "quorum": 1,
            "signatures": null,
            "type": "raw_tx_signature"
          },
          {
            "type": "data",
            "value": "05cdbcc705f07ad87521835bbba226ad7b430cc24e5e3f008edbe61540535419"
          }
        ]
      }
    ]
  }
}
```

对应响应对象的源代码如下：
```go
type request struct{
	TxTemplate txbuilder.Template `json:"transaction_template"`
}

// Template represents a partially- or fully-signed transaction.
type Template struct {
	Transaction         *types.Tx             `json:"raw_transaction"`
	SigningInstructions []*SigningInstruction `json:"signing_instructions"`

	// AllowAdditional affects whether Sign commits to the tx sighash or
	// to individual details of the tx so far. When true, signatures
	// commit to tx details, and new details may be added but existing
	// ones cannot be changed. When false, signatures commit to the tx
	// as a whole, and any change to the tx invalidates the signature.
	AllowAdditional bool `json:"allow_additional_actions"`
}
```
其中`TxTemplate`相关字段的说明见build-transaction的结果描述

调用`estimate-transaction-gas`接口成功之后返回的json结果如下：
```js
{
  "total_neu": 5000000,
  "storage_neu": 3840000,
  "vm_neu": 1419000
}
```

对应响应对象的源代码如下：
```go
// EstimateTxGasResp estimate transaction consumed gas
type EstimateTxGasResp struct {
	TotalNeu   int64 `json:"total_neu"`
	StorageNeu int64 `json:"storage_neu"`
	VMNeu      int64 `json:"vm_neu"`
}
```

结构字段说明如下：
- `TotalNeu` 预估的总手续费（单位为neu），该值直接加到build-transaction的BTM资产输入action中即可
- `StorageNeu` 存储交易的手续费
- `VMNeu` 运行虚拟机的手续费

---

## 2、签名交易
API接口 sign-transaction，代码[api/hsm.go#L53](https://github.com/Bytom/bytom/blob/master/api/hsm.go#L53)

签名交易的输入请求json格式如下：
```js
{
  "password": "123456",
  "transaction": {
    "allow_additional_actions": false,
    "raw_transaction": "070100020161015f1190c60818b4aff485c865113c802942f29ce09088cae1b117fc4c8db2292212ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff8099c4d599010001160014a86c83ee12e6d790fb388345cc2e2b87056a077301000161015fb018097c4040c8dd86d95611a13c24f90d4c9d9d06b25f5c9ed0556ac8abd73442275aacbeda1522cd41580f875c3c452daf5174b17ba062bf0ab71a568c123f80a094a58d1d0101160014068840e56af74038571f223b1c99f1b60caaf456010003013effffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff80bfffcb9901011600140b946646626c55a52a325c8bb48de792284d9b7200013e42275aacbeda1522cd41580f875c3c452daf5174b17ba062bf0ab71a568c123f9d9f94a58d1d01160014c8b4391bab4923a83b955170d24ee4ca5b6ec3fb00013942275aacbeda1522cd41580f875c3c452daf5174b17ba062bf0ab71a568c123f6301160014366b275ed9b2266b645cf1b8be51009cc3b260e100",
    "signing_instructions": [
      {
        "position": 0,
        "witness_components": [
          {
            "keys": [
              {
                "derivation_path": [
                  "010100000000000000",
                  "0100000000000000"
                ],
                "xpub": "de0db655c091b2838ccb6cddb675779b0a9a4204b122e61699b339867dd10eb0dbdc926882ff6dd75c099c181c60d63eab0033a4b0a4d0a8c78079e39d7ad1d8"
              }
            ],
            "quorum": 1,
            "signatures": null,
            "type": "raw_tx_signature"
          },
          {
            "type": "data",
            "value": "d174db6506e35f2decb5be148c2984bfd0f6c67f043365bf642d1af387c04fd5"
          }
        ]
      },
      {
        "position": 1,
        "witness_components": [
          {
            "keys": [
              {
                "derivation_path": [
                  "010100000000000000",
                  "0800000000000000"
                ],
                "xpub": "de0db655c091b2838ccb6cddb675779b0a9a4204b122e61699b339867dd10eb0dbdc926882ff6dd75c099c181c60d63eab0033a4b0a4d0a8c78079e39d7ad1d8"
              }
            ],
            "quorum": 1,
            "signatures": null,
            "type": "raw_tx_signature"
          },
          {
            "type": "data",
            "value": "05cdbcc705f07ad87521835bbba226ad7b430cc24e5e3f008edbe61540535419"
          }
        ]
      }
    ]
  }
}
```

对应请求对象的源代码如下：
```go
type SignRequest struct {    //function pseudohsmSignTemplates request
	Password string             `json:"password"`
	Txs      txbuilder.Template `json:"transaction"`
}
```

结构字段说明如下：
- `Password` 签名的密码，根据密码可以从节点服务器上解析出用户的私钥，然后用私钥对交易进行签名
- `Txs` 交易模板，build-transaction的返回结果，结构类型为 `txbuilder.Template`，相关字段的说明见build-transaction的结果描述

签名交易`sign-transaction`请求成功之后返回的json结果如下：
```js
{
  "sign_complete": true,
  "transaction": {
    "allow_additional_actions": false,
    "raw_transaction": "070100020161015f1190c60818b4aff485c865113c802942f29ce09088cae1b117fc4c8db2292212ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff8099c4d599010001160014a86c83ee12e6d790fb388345cc2e2b87056a0773630240273d5fc4fb06909fbc2968ea91c411fd20f690c88e74284ce2732052400129948538562fe432afd6cf17e590e8645b80edf80b9d9581d0a980d5f9f859e3880620d174db6506e35f2decb5be148c2984bfd0f6c67f043365bf642d1af387c04fd50161015fb018097c4040c8dd86d95611a13c24f90d4c9d9d06b25f5c9ed0556ac8abd73442275aacbeda1522cd41580f875c3c452daf5174b17ba062bf0ab71a568c123f80a094a58d1d0101160014068840e56af74038571f223b1c99f1b60caaf4566302400cf0beefceaf9fbf1efadedeff7aee5b38ee7a25a20d78b630b01613bc2f8c9230555a6e09aaa11a82ba68c0fc9e98a47c852dfe3de851d93f9b2b7ce256f90d2005cdbcc705f07ad87521835bbba226ad7b430cc24e5e3f008edbe6154053541903013effffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff80bfffcb9901011600140b946646626c55a52a325c8bb48de792284d9b7200013e42275aacbeda1522cd41580f875c3c452daf5174b17ba062bf0ab71a568c123f9d9f94a58d1d01160014c8b4391bab4923a83b955170d24ee4ca5b6ec3fb00013942275aacbeda1522cd41580f875c3c452daf5174b17ba062bf0ab71a568c123f6301160014366b275ed9b2266b645cf1b8be51009cc3b260e100",
    "signing_instructions": [
      {
        "position": 0,
        "witness_components": [
          {
            "keys": [
              {
                "derivation_path": [
                  "010100000000000000",
                  "0100000000000000"
                ],
                "xpub": "de0db655c091b2838ccb6cddb675779b0a9a4204b122e61699b339867dd10eb0dbdc926882ff6dd75c099c181c60d63eab0033a4b0a4d0a8c78079e39d7ad1d8"
              }
            ],
            "quorum": 1,
            "signatures": [
              "273d5fc4fb06909fbc2968ea91c411fd20f690c88e74284ce2732052400129948538562fe432afd6cf17e590e8645b80edf80b9d9581d0a980d5f9f859e38806"
            ],
            "type": "raw_tx_signature"
          },
          {
            "type": "data",
            "value": "d174db6506e35f2decb5be148c2984bfd0f6c67f043365bf642d1af387c04fd5"
          }
        ]
      },
      {
        "position": 1,
        "witness_components": [
          {
            "keys": [
              {
                "derivation_path": [
                  "010100000000000000",
                  "0800000000000000"
                ],
                "xpub": "de0db655c091b2838ccb6cddb675779b0a9a4204b122e61699b339867dd10eb0dbdc926882ff6dd75c099c181c60d63eab0033a4b0a4d0a8c78079e39d7ad1d8"
              }
            ],
            "quorum": 1,
            "signatures": [
              "0cf0beefceaf9fbf1efadedeff7aee5b38ee7a25a20d78b630b01613bc2f8c9230555a6e09aaa11a82ba68c0fc9e98a47c852dfe3de851d93f9b2b7ce256f90d"
            ],
            "type": "raw_tx_signature"
          },
          {
            "type": "data",
            "value": "05cdbcc705f07ad87521835bbba226ad7b430cc24e5e3f008edbe61540535419"
          }
        ]
      }
    ]
  }
}
```

对应响应对象的源代码如下：
```go
type signResp struct {
	Tx           *txbuilder.Template `json:"transaction"`
	SignComplete bool                `json:"sign_complete"`
}
```

结构字段说明如下：
- `Tx` 签名之后的交易模板`txbuilder.Template`，如果签名成功则`signatures`会由null变成签名的值，而`raw_transaction`的长度会变长，是因为`bc.Tx`部分添加了验证签名的参数信息
- `SignComplete` 签名是否完成标志，如果为`true`表示签名完成，否则为`false`表示签名未完成，单签的话一般可能为签名密码错误; 而多签的话一般为还需要其他签名。签名失败只需将签名的交易数据用正确的密码重新签名即可，无需再次`build-transaction`构建交易

---

## 3、提交交易
API接口 submit-transaction，代码[api/transact.go#L135](https://github.com/Bytom/bytom/blob/master/api/transact.go#L135)

提交交易的输入请求json格式如下：
```js
{
  "raw_transaction": "070100020161015f1190c60818b4aff485c865113c802942f29ce09088cae1b117fc4c8db2292212ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff8099c4d599010001160014a86c83ee12e6d790fb388345cc2e2b87056a0773630240273d5fc4fb06909fbc2968ea91c411fd20f690c88e74284ce2732052400129948538562fe432afd6cf17e590e8645b80edf80b9d9581d0a980d5f9f859e3880620d174db6506e35f2decb5be148c2984bfd0f6c67f043365bf642d1af387c04fd50161015fb018097c4040c8dd86d95611a13c24f90d4c9d9d06b25f5c9ed0556ac8abd73442275aacbeda1522cd41580f875c3c452daf5174b17ba062bf0ab71a568c123f80a094a58d1d0101160014068840e56af74038571f223b1c99f1b60caaf4566302400cf0beefceaf9fbf1efadedeff7aee5b38ee7a25a20d78b630b01613bc2f8c9230555a6e09aaa11a82ba68c0fc9e98a47c852dfe3de851d93f9b2b7ce256f90d2005cdbcc705f07ad87521835bbba226ad7b430cc24e5e3f008edbe6154053541903013effffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff80bfffcb9901011600140b946646626c55a52a325c8bb48de792284d9b7200013e42275aacbeda1522cd41580f875c3c452daf5174b17ba062bf0ab71a568c123f9d9f94a58d1d01160014c8b4391bab4923a83b955170d24ee4ca5b6ec3fb00013942275aacbeda1522cd41580f875c3c452daf5174b17ba062bf0ab71a568c123f6301160014366b275ed9b2266b645cf1b8be51009cc3b260e100"
}
```

对应源代码的请求对象如下：
```go
type SubmitRequest struct {    //function submit request
	Tx types.Tx `json:"raw_transaction"`
}
```

结构字段说明如下：
- `Tx` 签名完成之后的交易信息。这里需要注意该字段中的`raw_transaction`不是签名交易`sign-transaction`的全部返回结果，而是签名交易返回结果中`transaction`中的`raw_transaction`字段。

`submit-transaction`请求成功之后返回的json结果如下：
```js
{
  "tx_id": "2c0624a7d251c29d4d1ad14297c69919214e78d995affd57e73fbf84ece361cd"
}
```

对应源代码的响应对象如下：
```go
type submitTxResp struct {
	TxID *bc.Hash `json:"tx_id"`
}
```

结构字段说明如下：
- `TxID` 交易ID，当交易被提交到交易池之后会显示该信息，否则表示交易失败
