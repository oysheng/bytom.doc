# bytom开发者文档（交易部分）

# 账户管理模式
该部分主要针对用户使用bytom自带的账户模式发送交易

## 1、构建交易
API接口 build-transaction，代码[api/transact.go#L120](https://github.com/Bytom/bytom/blob/master/api/transact.go#L120)

### 请求 Request
输入请求的json格式如下：（以标准的转账交易为例）
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
- `Tx` 交易的TxData部分，该字段暂未使用，为空即可
- `TTL` 构建交易的生存时间（单位为毫秒），意味着在该时间范围内，已经缓存的utxo不能用于再一次build交易，除非剩余的utxo足以构建一笔新的交易，否则会报错。当ttl为0时会被默认设置为600s，即5分钟
- `TimeRange` 时间戳，意味着该交易将在该时间戳（区块高度）之后不会被提交上链，为了防止交易在网络中传输延迟而等待太久时间，如果交易没有在特定的时间范围内被打包，该交易便会失效
- `Actions` 交易的actions结构，所有的交易都是由action构成的，`map`类型的`interface{}`保证了action类型的可扩展性。其中action中必须包含type字段，用于区分不同的action类型，action分为input和output两种类型，其详细介绍如下：
    - input action 类型：
        - issue 发行资产
        - spend_account 以账户的模式花费utxo
        - spend_account_unspent_output 直接花费指定的utxo
    - output action 类型：
        - control_address 接收方式为地址模式
        - control_program 接收方式为（program）合约模式
        - retire 销毁资产

*注意事项*：
- 交易必须至少包含一个input和output（coinbase交易除外，因为coinbase交易是由系统产生，故不在此加以描述），否则交易将会报错。
- 除了BTM资产（input减去output的BTM资产作为手续费）之外，其他资产在构建input和output时，所有输入和输出的资产总和必须相等，否则交易会报出输入输出不平衡的错误信息。
- 交易的手续费： 所有inputs的BTM数量 - 所有outputs的BTM数量

下面对所有的action进行详细说明：

#### 1. issue
issueAction结构体：
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
- `assets` 主要用于资产的管理，无需用户设置参数
- `AssetAmount` 表示用户需要发行的资产ID和对应的资产数目，这里的`AssetID`需要通过`create-asset`创建

issueAction输入的json格式如下：
```js
{
  "amount": 100000000,
  "asset_id": "3152a15da72be51b330e1c0f8e1c0db669269809da4f16443ff266e07cc43680",
  "type": "issue"
}
```

#### 2. spend_account
spendAction结构体：
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
- `accounts` 主要用于账户的管理，无需用户设置参数
- `AccountID` 表示需要花费资产的账户ID
- `AssetAmount` 表示花费的资产ID和对应的资产数目
- `ClientToken` 表示Reserve用户UTXO的限制条件，目前不填或为空即可

spendAction输入的json格式如下：
```js
{
  "account_id": "0BF63M2U00A04",
  "amount": 2000000000,
  "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
  "type": "spend_account"
}
```

#### 3. spend_account_unspent_output
spendUTXOAction结构体：
```go
type spendUTXOAction struct {
	accounts *Manager
	OutputID *bc.Hash `json:"output_id"`
	ClientToken *string `json:"client_token"`
}
```
- `accounts` 主要用于账户的管理，无需用户设置参数
- `OutputID` 表示需要花费的UTXO的ID，可以根据`list-unspent-outputs`查询可用的UTXO，其中`OutputID`对应该API返回结果的`id`字段
- `ClientToken` 表示Reserve用户UTXO的限制条件，目前不填或为空即可

spendUTXOAction输入的json格式如下：
```js
{
  "type": "spend_account_unspent_output",
  "output_id": "58f29f0f85f7bd2a91088bcbe536dee41cd0642dfb1480d3a88589bdbfd642d9"
}
```

#### 4. control_address
controlAddressAction结构体：
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
- `Address` 表示接收资产的地址，可以根据 `create-account-receiver` API接口创建地址
- `AssetAmount` 表示接收的资产ID和对应的资产数目

controlAddressAction输入的json格式如下：
```js
{
  "amount": 100000000,
  "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
  "address": "bm1q50u3z8empm5ke0g3ngl2t3sqtr6sd7cepd3z68",
  "type": "control_address"
}
```

#### 5. control_program
controlProgramAction结构体：
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
- `Program` 表示接收资产的合约脚本，可以根据 `create-account-receiver` API接口创建接收`program`（返回结果的 `program` 和 `address` 是一一对应的）
- `AssetAmount` 表示接收的资产ID和对应的资产数目

controlProgramAction输入的json格式如下：
```js
{
  "amount": 100000000,
  "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
  "control_program":"0014a3f9111f3b0ee96cbd119a3ea5c60058f506fb19",
  "type": "control_program"
}
```

#### 6. retire
retireAction结构体：
```go
type retireAction struct {
	bc.AssetAmount
}

type AssetAmount struct {
	AssetId *AssetID `protobuf:"bytes,1,opt,name=asset_id,json=assetId" json:"asset_id,omitempty"`
	Amount  uint64   `protobuf:"varint,2,opt,name=amount" json:"amount,omitempty"`
}
```
- `AssetAmount` 表示销毁的资产ID和对应的资产数目

retireAction输入的json格式如下：
```js
{
  "amount": 900000000,
  "asset_id": "3152a15da72be51b330e1c0f8e1c0db669269809da4f16443ff266e07cc43680",
  "type": "retire"
}
```

### 输出 Response
build-transaction返回的json结果如下：
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

对应源代码的响应对象如下：
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

## 2、签名交易
API接口 sign-transaction，代码[api/hsm.go#L53](https://github.com/Bytom/bytom/blob/master/api/hsm.go#L53)

### 请求 Request
输入请求的json格式如下：
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

对应源代码的请求对象如下：
```go
type SignRequest struct {    //function pseudohsmSignTemplates request
	Password string             `json:"password"`
	Txs      txbuilder.Template `json:"transaction"`
}
```

结构字段说明如下：
- `Password` 签名的密码，根据密码可以从节点服务器上解析出用户的私钥，然后用私钥对交易进行签名
- `Txs` 交易模板，build-transaction的返回结果，结构类型为 `txbuilder.Template`，相关字段的说明见build-transaction的结果描述

### 输出 Response
sign-transaction返回的json结果如下：
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

对应源代码的响应对象如下：
```go
type signResp struct {
	Tx           *txbuilder.Template `json:"transaction"`
	SignComplete bool                `json:"sign_complete"`
}
```

结构字段说明如下：
- `Tx` 签名之后的交易模板`txbuilder.Template`，如果签名成功则`signatures`会由null变成签名的值，而`raw_transaction`的长度会变长，是因为`bc.Tx`部分添加了验证签名的参数信息
- `SignComplete` 签名是否完成标志，如果为`true`表示签名完成，否则为`false`表示签名未完成，单签的话一般可能为签名密码错误; 而多签的话一般为还需要其他签名

## 3、提交交易
API接口 submit-transaction，代码[api/transact.go#L135](https://github.com/Bytom/bytom/blob/master/api/transact.go#L135)


### 请求 Request
输入请求的json格式如下：
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
- `Tx` 签名完成之后的交易信息


### 输出 Response
submit-transaction返回的json结果如下：
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

# 用户UTXO自己管理模式
该部分主要针对用户自己管理私钥和地址，并通过utxo来构建和发送交易。

## 注意事项
以下步骤以及功能改造仅供参考，具体代码实现需要用户根据实际情况进行调试，具体可以参考单元测试案例代码[blockchain/txbuilder/txbuilder_test.go#L255](https://github.com/Bytom/bytom/blob/master/blockchain/txbuilder/txbuilder_test.go#L255)

## 1、创建私钥和公钥
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

## 2、根据公钥创建地址`address`和对应的`program`
其中创建单签地址参考代码[account/accounts.go#L267](https://github.com/Bytom/bytom/blob/master/account/accounts.go#L267)进行相应改造为：
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

创建多签地址参考代码[account/accounts.go#L294](https://github.com/Bytom/bytom/blob/master/account/accounts.go#L294)进行相应改造为：
```go
func (m *Manager) createP2SH(xpubs []chainkd.XPub) (*CtrlProgram, error) {
	derivedPKs := chainkd.XPubKeys(xpubs)
	signScript, err := vmutil.P2SPMultiSigProgram(derivedPKs, len(derivedPKs))
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

## 3、找到属于你自己的utxo，即找到接收地址或接收`program`是你自己的`unspend_output`
其中utxo的结构为：（参考代码[account/reserve.go#L39](https://github.com/Bytom/bytom/blob/master/account/reserve.go#L39)）
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

## 4、通过`utxo`构造交易输入`TxInput`和签名需要的数据信息`SigningInstruction`
该部分功能可以参考代码[account/builder.go#L169](https://github.com/Bytom/bytom/blob/master/account/builder.go#L169)进行相应改造为:
```go
// UtxoToInputs convert an utxo to the txinput
func UtxoToInputs(xpubs []chainkd.XPub, u *UTXO) (*types.TxInput, *txbuilder.SigningInstruction, error) {
	txInput := types.NewSpendInput(nil, u.SourceID, u.AssetID, u.Amount, u.SourcePos, u.ControlProgram)
	sigInst := &txbuilder.SigningInstruction{}

	if u.Address == "" {
		return txInput, sigInst, nil
	}

	address, err := common.DecodeAddress(u.Address, &consensus.ActiveNetParams)
	if err != nil {
		return nil, nil, err
	}

	switch address.(type) {
	case *common.AddressWitnessPubKeyHash:
		derivedPK := xpubs[0].PublicKey()
		sigInst.WitnessComponents = append(sigInst.WitnessComponents, txbuilder.DataWitness([]byte(derivedPK)))

	case *common.AddressWitnessScriptHash:
		derivedPKs := chainkd.XPubKeys(xpubs)
		script, err := vmutil.P2SPMultiSigProgram(derivedPKs, len(derivedPKs))
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

## 5、通过`utxo`构造交易输出`TxOutput`
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

## 6、组合交易的input和output构成交易模板
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
	for _, in := range inputs {
		// Empty signature arrays should be serialized as empty arrays, not null.
		in.sigInst.Position = uint32(len(inputs))
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

## 7、对构造的交易进行签名
账户模型是根据密码找到对应的私钥对交易进行签名，这里用户可以直接使用私钥对交易进行签名，可以参考签名代码[blockchain/txbuilder/txbuilder.go#L82](https://github.com/Bytom/bytom/blob/master/blockchain/txbuilder/txbuilder.go#L82)进行改造为:（以下改造仅支持单签交易，多签交易用户可以参照该示例进行改造）
```go
// Sign will try to sign all the witness
func Sign(tpl *Template, xprv chainkd.XPrv) error {
	for i, sigInst := range tpl.SigningInstructions {
		h := tpl.Hash(uint32(i)).Byte32()
		sig := xprv.Sign(h[:])
		rawTxSig := &RawTxSigWitness{
			Quorum: 1,
			Sigs:   []json.HexBytes{sig},
		}
		sigInst.WitnessComponents = append([]witnessComponent(rawTxSig), sigInst.WitnessComponents...)
	}
	return materializeWitnesses(tpl)
}
```

## 8、提交交易上链
该步骤无需更改任何内容，直接参照`submit-transaction`的功能即可
