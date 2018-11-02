# bytom智能合约

- [Equity简介](#Equity简介)
  - [合约组成](#合约组成)
  - [条款组成](#条款组成)
  - [参数列表](#参数列表)
  - [语句组成](#语句组成)
  - [数据类型](#数据类型)
  - [表达式](#表达式)
  - [内置函数](#内置函数)
- [合约交易构造流程](#合约交易构造流程)
  - [合约参数构造](#合约参数构造)
  - [编译合约](#编译合约)
  - [锁定合约](#锁定合约)
  - [查找合约utxo](#查找合约utxo)
  - [解锁合约](#解锁合约)
- [典型合约模板解析](#典型合约模板解析)
  - [单签验证合约](#单签验证合约)
  - [多签验证合约](#多签验证合约)
  - [pubkey哈希校验和单签验证合约](#pubkey哈希校验和单签验证合约)
  - [字符串哈希校验合约](#字符串哈希校验合约)
  - [币币交易合约](#币币交易合约)
  - [第三方信任机构托管合约](#第三方信任机构托管合约)
  - [抵押贷款合约](#抵押贷款合约)
  - [看涨期权合约](#看涨期权合约)

## Equity简介
  Equity是用于表达合约程序的高级语言，专门用来编写运行在[Bytom](https://github.com/Bytom/bytom)上的合约程序，Equity智能合约主要用于描述对`Bytom`上的各类资产的操作管理。合约的主要特征如下：

  - `Bytom`采用`BUTXO`结构，区块链上记录着由多种不同类型的`UTXO`构成的账本。每一笔`UTXO`都有两个重要属性：资产编号`assetID`和资产数量`amount`，一般将指定数量`amount`的资产`assetID`抽象地指代一笔`UTXO`。
  - 比原链上的所有资产都是锁定在合约`program`中，`valueAmount`数量的`valueAsset`资产(即`UTXO`）一旦被一个合约解锁，仅仅是为了被一个或多个其他合约来进行锁定
  - 合约保护资产`valueAmount of valueAsset`的方式是只有用户输入正确的解锁参数才能使合约程序在虚拟机中执行成功
  
  因此，用`Equity`语言编写的智能合约，其目的就是 "描述用智能合约锁定哪些资产，以及定义在哪些条件下可以解锁指定的资产"。

### 合约组成
  `Equity`合约程序是由一个用`contract`关键字定义的合约结构组成。一个合约的形式为：

  `contract ContractName ( parameters ) locks valueAmount of valueAsset { clauses }`

  - `ContractName` 合约名，是一个标识符，代表合约的名称，在编写合约时自定义。
  - `parameters` 合约参数列表，其类型名必须是合约语言的基本类型 
  - `valueAmount` 合约锁定的资产数量，即`UTXO`中的`amount`，标识符可以自定义
  - `valueAsset` 合约锁定的资产类型，即`UTXO`中的`assetID`，标识符可以自定义
  - `clauses` 条款（即函数）列表，一个合约至少包含一个`clause`

### 条款组成
  每个`clause`条款函数描述了一种解锁合约`UTXO`的方法和解锁所需的参数信息。`clause`的结构为：

  `clause ClauseName ( parameters ) { statements }`

  - `ClauseName` 条款名，是一个标识符，代表条款函数的名称，在编写时自定义。
  - `parameters` 条款参数列表
  - `statements` 合约语句列表，一个`clause`至少包含一条语句    

### 参数列表
  合约和条款的参数需指明变量名和变量类型。参数定义的格式是：

  - `name : TypeName`

  参数列表的格式为：

  - `name1 : TypeName1 , name2 : TypeName2 , …`

  为简洁起见，可以像这样合并相同类型的相邻参数：

  - `name1 , name2 , … : TypeName`

  所以这两种合约的声明是等价的：

  - `contract LockWithMultiSig(key1: PublicKey, key2: PublicKey, key3: PublicKey)`
  - `contract LockWithMultiSig(key1, key2, key3: PublicKey)`

  可用的变量类型有：

  - `Integer` `Amount` `Boolean` `String` `Hash` `Asset` `PublicKey` `Signature` `Program`

  在[数据类型](#数据类型)中将会详细介绍这些变量类型。  

### 语句组成
  `statement`合约语句，除了`verify`、`lock`和`unlock`基本语句类型之外，目前还新增加了`define`、`assign`和`if-else`扩展语句类型的支持，语句的格式如下：

  * `verify` 验证条件语句，用来验证表达式的结果是否为真，模式如下:
      
    &nbsp;
    `verify expression`

    &nbsp;
    其中`expression`的结果必须是`bool`类型，`expression`表达式的结果必须为`true`时表示验证成功。示例如下：
      - `verify above(blockNumber)` 检测当前块的区块高度是否高于 `blockNumber`。
      - `verify checkTxSig(key, sig)` 检测给定的签名是否与预先设定的公钥相匹配。
      - `verify newBid > currentBid` 检测`newBid`是否大于`currentBid`。
  &nbsp;
  * `lock` 锁定合约资产语句，模式如下：
  
    &nbsp;
    `lock valueAmount of valueAsset with program`
    
    &nbsp;
    其中`valueAmount`表示资产数量，`valueAsset`表示资产类型，而`program`表示接收对象且必须为`Program`类型。
  &nbsp;
  * `unlock` 解锁合约资产语句，模式如下：

    &nbsp;
    `unlock valueAmount of valueAsset`

    &nbsp;  
    其中`valueAmount`表示资产数量，`valueAsset`表示资产类型，`unlock`语句表示解锁的资产可以指定给任意接收对象。
  &nbsp;
  * `define` 自定义变量语句，模式如下：
    
    &nbsp;
    `define identifier : TypeName = expression`
    或
    `define identifier : TypeName`
  
    &nbsp;
    其中`identifier`表示用户定义了数据类型为`TypeName`的变量，如果自定义的变量没有赋值，则该变量必须`assign`语句中赋值。示例如下：
    - `define value : Integer = amount` 定义了整型的变量`value`，并且将`amount`赋值给该变量。
    - `define value : Integer = amount + shift` 定义了整型的变量`value`，并且将表达式`amount + shift`的结果赋值给该变量。
    - `define value : Integer` 定义了整型的变量`value`，并没有赋值，该变量需要在`assign`语句中赋值，否则会报错“变量未赋值”。
  &nbsp;
  * `assign` 自定义变量赋值语句，模式如下
    
    &nbsp;
    `assign identifier = expression`
  
    &nbsp;
    其中`identifier`必须为`define`语句中用户自定义的变量，禁止修改`contract`和`clause`中的变量。示例如下：
    - `assign value = amount` 将`amount`赋值给变量`value`，并且变量`amount`和`value`的数据类型必须相同。
  &nbsp;
  * `if-else` 条件判断语句，模式如下：
  
    &nbsp;
    `if expression { statements }`
    或
    `if expression { statements } else { statements }`

    &nbsp;
    其中`expression`为`if-else`语句的条件判断表达式，并且该表达式的结果必须是`bool`类型，当结果为`true`时执行`if`下面的`statements`语句块，否则执行`else`下面的`statements`语句块。

### 数据类型
  Equity语言支持的数据类型如下：

  - `Boolean` 布尔类型，值为`true`或`false`.
  - `Integer` 整数类型，取值范围为`[-2^63, 2^63-1]`.
  - `Amount` 无符号整数类型，取值范围为`[0, 2^63-1]`.
  - `Asset` 资产类型，32个字节长度的资产ID.
  - `Hash` 哈希类型，32个字节长度的`hash`值.
  - `PublicKey` 公钥类型，32个字节长度的`publickey`.
  - `Signature` 签名类型，该类型需要根据`publickey`对应的主公钥`root_xpub`和`derivation_path`来构造，且只能用于`clause`的参数列表中.
  - `Program` 程序类型，接收`program`，跟地址是一一对应.
  - `String` 字符串类型，16进制字符串.

## 表达式
  `Equity`表达式可用于上述语句中的`expression`中，支持的表达式类别如下：
  
  * 一元表达式
    - `- expr` : 对数学表达式取负值
    - `~ expr` : 对字节串做按位翻转
  &nbsp;
  * 条件表达式，下面的表达式都必须为数字类型操作数(即`Integer`或`Amount`类型)，并且返回一个 `Boolean` 型的结果：
    - `expr1 > expr2` : 检测`expr1`是否大于`expr2`
    - `expr1 < expr2` : 检测`expr1`是否小于`expr2`
    - `expr1 >= expr2` : 检测`expr1`是否大于或等于`expr2`
    - `expr1 <= expr2` : 检测`expr1`是否小于或等于`expr2`
    - `expr1 == expr2` : 检测`expr1`是否等于`expr2`
    - `expr1 != expr2` : 检测`expr1`是否不等于`expr2`
  &nbsp;
  * 按位操作表达式，下面的表达式为字节类型，且返回值也是字节类型：
    - `expr1 ^ expr2` : 得到两操作数按位异或(XOR)的结果
    - `expr1 | expr2` : 得到两操作数按位或(OR)的结果
    - `expr1 & expr2` : 得到两操作数按位与(AND)的结果
  &nbsp;
  * 数值表达式，下面的表达式都是数值型操作数(`Integer`或`Amount`)，并且返回数值型的结果：
    - `expr1 + expr2` : 两操作数相加
    - `expr1 - expr2` : 两操作数相减，`expr1`减去`expr2`
    - `expr1 * expr2` : 两操作数相乘
    - `expr1 / expr2` : 两操作数相除，`expr1`除以`expr2`
    - `expr1 % expr2` : 操作数取余，即`expr1`对`expr2`取余
    - `expr1 << expr2` : 将`expr1`按位左移`expr2`位
    - `expr1 >> expr2` : 将`expr1`按位右移`expr2`位
  &nbsp;
  * 其他的表达式类型：
    - `( expr )` : 表示`expr`本身
    - `expr ( arguments )` : 表示函数调用，传入的参数列表`arguments`是用逗号分隔的，具体可查阅下面的[内置函数](#内置函数)
    - 单独出现的标识符表示变量本身
    - `[ exprs ]` : 表示一个列表，其中`exprs`是以逗号分隔的表达式列表(列表参数目前仅用于`checkTxMultiSig`中)
    - 以 `-` 开头的一串数字序列表示整型数据列表
    - 单引号 `'...'` 之间的字节序列表示一个字符串
    - 前缀 `0x` 后跟 `2n` 个十六进制数字，其长度为`n`

### 内置函数
  `Equity`提供了一下内置函数，相关函数如下：

  - `abs(n)` 返回数值`n`的绝对值.
  - `min(x, y)` 返回两个数值`x`和`y`中最小的一个.
  - `max(x, y)` 返回两个数值`x`和`y`中最大的一个.
  - `size(s)` 返回任意类型的字节大小`size`.
  - `concat(s1, s2)` 返回连接两个字符串`s1`和`s2`生成新的字符串.
  - `concatpush(s1, s2)` 将两个字符串类型的虚拟机执行操作码`s1`和`s2`连接起来(即将`s2`拼接在`s1`的后面），然后将他们`push`到栈中. 该操作函数主要用于嵌套合约中.
  - `below(height)` 判断当前区块高度是否低于参数`height`，如果是则返回`true`，否则返回`false`.
  - `above(height)` 判断当前区块高度是否高于参数`height`，如果是则返回`true`，否则返回`false`.
  - `sha3(s)` 返回字节类型字符串参数`s`的`SHA3-256`的哈希运算结果.
  - `sha256(s)` 返回字节类型字符串参数`s`的`SHA-256`的哈希运算结果.
  - `checkTxSig(key, sig)` 根据一个`PublicKey`和一个`Signature`验证交易的签名是否正确.
  - `checkTxMultiSig([key1, key2, ...], [sig1, sig2, ...])` 根据多个`PublicKey`和多个`Signature`验证交易的多重签名是否正确.

----

## 合约交易构造流程
### 合约参数构造
合约参数主要包括两个方面，一个是编译合约`contract`中的参数，另一个是解锁合约`clause`中的参数。其中参数的相关注意事项如下：
  - 调用编译合约API接口`compile`不加参数是直接编译合约，按照`contract`中的参数列表顺序加上参数是将合约实例化
  - 构造解锁合约交易需要添加`clause`中的参数列表
  - `Signature`参数类型只能在`clause`的参数列表中出现，不允许出现在`contract`的参数列表中
  - 如果合约包含多个`clause`，那么用户只需选择任意一个`clause`来解锁就可以了。在构造解锁合约的交易过程中，需要添加额外的参数`clause_selector`（无符号整数类型，小端存储格式），`clause_selector`是根据合约`clause`的顺序来指定的，假如`clause`的个数`n`，那么选择对应的`clause_selector`为`0 ~ n-1`，即第一个`clause`的`clause_selector`为`0`，第二个`clause`的`clause_selector`为`1`，以此类推。

如果合约的`clause`参数列表中包含`Signature`，那么在构造解锁合约交易的时候需要通过签名的必要条件`root_xpub`和`derivation_path`来间接获得，因为签名必须通过调用`sign-transaction`API接口才能得到。参数`root_xpub`和`derivation_path`是通过调用`list-pubkeys`接口获取的，此外`Signature`一般需要跟`PublicKey`配套使用，即参数`root_xpub`和`derivation_path`需要跟公钥`pubkey`一一对应，否则合约会执行失败。

其中[API接口`list-pubkeys`](https://github.com/Bytom/bytom/wiki/API-Reference#list-pubkeys)的参数如下：
- `String` - *account_id*, 账户ID.
- `String` - *account_alias*, 账户别名.
- `String` - *public_key*, 根据指定pubkey来查询.

其请求和响应的json格式如下：
```js
// Request
{
  "account_id": "0G1JIR6400A02"
}

// Result
{
  "pubkey_infos": [
    {
      "derivation_path": [
        "010100000000000000",
        "0300000000000000"
      ],
      "pubkey": "c37d5531f393bc6a3568628c0c0e17801ea452e75d604deb01403c4b161659a3"
    },
    {
      "derivation_path": [
        "010100000000000000",
        "0200000000000000"
      ],
      "pubkey": "117d12e84bb19e956451e0b1eb2bffc662ecb7aac7e63d77e524ddd467eb3617"
    },
    {
      "derivation_path": [
        "010100000000000000",
        "0100000000000000"
      ],
      "pubkey": "e9108d3ca8049800727f6a3505b3a2710dc579405dde03c250f16d9a7e1e6e78"
    }
  ],
  "root_xpub": "5c6145b241b1147987565719657a0506ebb417a2e110a235a42cfb40951880f447432f930ce9fd1a6b7e51b3ddbfdc7adb57d33448f93c0defb4de630703a144"
}
```

----

### 编译合约
编译合约是将合约编译成可执行的虚拟机指令流程。如果合约有参数列表`contract parameters`的话，在锁定合约之前需要对这些合约参数进行实例化，因为这些参数是解锁合约的限制条件。

编译合约目前支持两种方式，一种是使用`equity`编译工具，另一种是调用编译合约的API接口`compile`。其中通过[`equity`编译工具](https://github.com/Bytom/equity)的方式如下：
```
./equity <contract_file> [flags]
```

其中`flag`标志如下：
```
    --bin        Binary of the contracts in hex.
    --instance   Object of the Instantiated contracts.
    --shift      Function shift of the contracts.
```

以`LockWithPublicKey`为例，编译并实例化合约如下：
```
./equity LockWithPublicKey --instance e9108d3ca8049800727f6a3505b3a2710dc579405dde03c250f16d9a7e1e6e78
```

返回结果如下：
```
======= LockWithPublicKey =======
Instantiated program:
20e9108d3ca8049800727f6a3505b3a2710dc579405dde03c250f16d9a7e1e6e787403ae7cac00c0
```

另一种是通过调用编译合约[API接口`compile`](https://github.com/Bytom/bytom/wiki/API-Reference#compile)的方式，其接口参数如下：（新版编译器暂未提交）
- `String` - *contract*, 合约内容.
- `Array of Object` - *args*, 合约参数结构体（数组类型）.
  - `Boolean` - *boolean*, 布尔类型的合约参数，对应的基本类型是`Boolean`.
  - `Integer` - *integer*, 整数类型的合约参数，对应的基本类型包括：`Integer`、`Amount`.
  - `String` - *string*, 字符串类型的合约参数，对应的基本类型包括：`String`、`Asset`、`Hash`、`Program`、`PublicKey`.

以`LockWithPublicKey`为例，其请求和响应的json格式如下：
```js
// Request
{
  "contract": "contract LockWithPublicKey(publicKey: PublicKey) locks valueAmount of valueAsset { clause unlockWithSig(sig: Signature) { verify checkTxSig(publicKey, sig) unlock valueAmount of valueAsset }}",
  "args": [
    {
      "string": "e9108d3ca8049800727f6a3505b3a2710dc579405dde03c250f16d9a7e1e6e78"
    }
  ]
}

// Result
{
  "name": "LockWithPublicKey",
  "source": "contract LockWithPublicKey(publicKey: PublicKey) locks valueAmount of valueAsset { clause unlockWithSig(sig: Signature) { verify checkTxSig(publicKey, sig) unlock valueAmount of valueAsset }}",
  "program": "20e9108d3ca8049800727f6a3505b3a2710dc579405dde03c250f16d9a7e1e6e787403ae7cac00c0",
  "params": [
    {
      "name": "publicKey",
      "type": "PublicKey"
    }
  ],
  "value": "locked",
  "clause_info": [
    {
      "name": "unlockWithSig",
      "args": [
        {
          "name": "sig",
          "type": "Signature"
        }
      ],
      "value_info": [
        {
          "name": "locked"
        }
      ],
      "block_heights": [],
      "hash_calls": null
    }
  ],
  "opcodes": "0xe9108d3ca8049800727f6a3505b3a2710dc579405dde03c250f16d9a7e1e6e78 DEPTH 0xae7cac FALSE CHECKPREDICATE",
  "error": ""
}
```

----

### 锁定合约
`lock`锁定合约，即部署合约，其本质是调用`build-transaction`接口将资产发送到合约特定的`program`，只需将接收方`control_program`设置为指定合约即可，构造锁定合约交易的模板如下：（注意：合约交易暂时不支持接收方资产为BTM资产的交易）
```js
// Request
{
  "base_transaction": null,
  "actions": [
    {
      "account_id": "0G1JIR6400A02",
      "amount": 20000000,
      "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
      "type": "spend_account"
    },
    {
      "account_id": "0G1JIR6400A02",
      "amount": 900000000,
      "asset_id": "1e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c22",
      "type": "spend_account"
    },
    {
      "amount": 900000000,
      "asset_id": "1e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c22",
      "control_program": "20e9108d3ca8049800727f6a3505b3a2710dc579405dde03c250f16d9a7e1e6e787403ae7cac00c0",
      "type": "control_program"
    }
  ],
  "ttl": 0,
  "time_range": 1521625823
}

// Result
{
  "raw_transaction": "0701dfd5c8d505020161015f150ec246dc739a8c4c3f7b4083ededcb2854ca221e437a49f23ec84c7c47ea80ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff8099c4d599010001160014726e902e30525e01f0157f12be476c904060383b01000160015ed53c1f3388681f62ae778ac8a54c2b091bbdc91d68ec1e94b20aa2183484f8331e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c2280c8afa02501011600145de3c504b41019d11698d572b1a37d9a4c9118c1010003013effffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff80bfffcb990101160014310c2265e8e3b7057a62caf09a9f907763f369ea00013d1e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c2280f69bf32101160014d0d18752a276c94b25f920b02a8edff251b16b7600014f1e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c2280d293ad03012820e9108d3ca8049800727f6a3505b3a2710dc579405dde03c250f16d9a7e1e6e787403ae7cac00c000",
  "signing_instructions": [
    {
      "position": 0,
      "witness_components": [
        {
          "type": "raw_tx_signature",
          "quorum": 1,
          "keys": [
            {
              "xpub": "5c6145b241b1147987565719657a0506ebb417a2e110a235a42cfb40951880f447432f930ce9fd1a6b7e51b3ddbfdc7adb57d33448f93c0defb4de630703a144",
              "derivation_path": [
                "010100000000000000",
                "0100000000000000"
              ]
            }
          ],
          "signatures": null
        },
        {
          "type": "data",
          "value": "e9108d3ca8049800727f6a3505b3a2710dc579405dde03c250f16d9a7e1e6e78"
        }
      ]
    },
    {
      "position": 1,
      "witness_components": [
        {
          "type": "raw_tx_signature",
          "quorum": 1,
          "keys": [
            {
              "xpub": "5c6145b241b1147987565719657a0506ebb417a2e110a235a42cfb40951880f447432f930ce9fd1a6b7e51b3ddbfdc7adb57d33448f93c0defb4de630703a144",
              "derivation_path": [
                "010100000000000000",
                "0700000000000000"
              ]
            }
          ],
          "signatures": null
        },
        {
          "type": "data",
          "value": "54df681ac3174a11d4456265641a204e04f64b8f860f37bf5584cf4187f54e99"
        }
      ]
    }
  ],
  "allow_additional_actions": false
}
```

构建交易成功之后，便可以对交易进行签名`sign-transaction`，返回结果中`sign_complete`为`true`表示签名成功，将签名的交易通过`submit-transaction`提交到交易池中，等待交易被打包上链

----

### 查找合约UTXO
部署合约交易发送成功之后，接下来便需要对合约锁定的资产进行解锁，解锁合约之前需要找到合约的UTXO。

可以通过调用[API接口`list-unspent-outputs`](https://github.com/Bytom/bytom/wiki/API-Reference#list-unspent-outputs)来查找，在查合约UTXO的情况下必须将`smart_contract`设置为`true`，否则会查不到，其参数如下：
- `String` - *id*, UTXO对应的`outputID`，可以根据发布合约交易的输出`action`中的找到.
- `Boolean` - *smart_contract*, 是否展示合约的UTXO，默认不显示.

对应的输入输出结果如下：
```js
// Request
curl -X POST list-unspent-outputs -d '{"id": "413d941faf5a19501ab4c06747fe1eb38c5ae76b74d0f5af524fc40ee6bf7116", "smart_contract": true}'

// Result
{
  "account_alias": "",
  "account_id": "",
  "address": "",
  "amount": 900000000,
  "asset_alias": "GOLD",
  "asset_id": "1e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c22",
  "change": false,
  "control_program_index": 0,
  "id": "413d941faf5a19501ab4c06747fe1eb38c5ae76b74d0f5af524fc40ee6bf7116",
  "program": "20e9108d3ca8049800727f6a3505b3a2710dc579405dde03c250f16d9a7e1e6e787403ae7cac00c0",
  "source_id": "c9680e6dd5e9ae7f825fe7edab9fa35c119eb7feab0ab4e426c84a579daf4ef9",
  "source_pos": 2,
  "valid_height": 0
}
```
找到对应的合约UTXO之后，可以通过[API接口`decode-program`](https://github.com/Bytom/bytom/wiki/API-Reference#decode-program)解析合约的参数信息，用户可以根据已有的参数信息判断该合约能否解锁

----

### 解锁合约
`unlock`解锁合约，即调用合约，其本质是通过给交易添加相应的合约参数以便合约程序`program`在虚拟机中执行成功，目前合约相关的参数都可以通过`build-transaction`中的`Action`结构`spend_account_unspent_output`中的数组参数`arguments`进行添加，其中参数如下：

1） `RawTxSigArgument` 签名相关的参数，`type`类型为`raw_tx_signature`，主要包含主公钥`xpub`和其对应的派生路径`derivation_path`，而待验证的`publickey`是通过该主公钥和派生路径生成的子公钥生成的（这些参数可以通过API接口`list-pubkeys`获取）
  - `xpub` 主公钥
  - `derivation_path` 派生路径，为了形成子私钥和子公钥
  
参数格式如下：
```js
{
  "type": "raw_tx_signature",
  "raw_data": {
    "xpub": "5c6145b241b1147987565719657a0506ebb417a2e110a235a42cfb40951880f447432f930ce9fd1a6b7e51b3ddbfdc7adb57d33448f93c0defb4de630703a144",
    "derivation_path": [
      "010100000000000000",
      "0100000000000000"
    ]
  }
}
```

2） `DataArgument` 数据类型参数，`type`类型为`data`，该类型可以兼容除了`rawTxSigArgument`以外的所有合约参数类型，其值是16进制的字符串，需要注意的是`integer`整数类型是小端存储格式。参数格式如下：（以`publickey`为例）
```js
{
  "type": "data",
  "raw_data": {
    "value": "b3f37834dfa74174e9f0d208302e77c637cfe66c3e37fe1e1574e416b3516e89"
  }
}
```

3）`BoolArgument` 布尔类型参数，`type`类型为`boolean`，该类型取值为`true`或`false`。参数格式如下：
```js
{
  "type": "boolean",
  "raw_data": {
    "value": true
  }
}
```

4）`IntegerArgument` 整型类型参数，`type`类型为`integer`。参数格式如下：
```js
{
  "type": "integer",
  "raw_data": {
    "value": 10000
  }
}
```

5）`StrArgument` 字符串类型参数，`type`类型为`string`。参数格式如下：
```js
{
  "type": "string",
  "raw_data": {
    "value": "this is a test string"
  }
}
```

以合约`LockWithPublicKey`为例，其解锁合约交易的模板如下：
```js
// Request
{
  "base_transaction": null,
  "actions": [
    {
      "type": "spend_account_unspent_output",
      "output_id": "413d941faf5a19501ab4c06747fe1eb38c5ae76b74d0f5af524fc40ee6bf7116",
      "arguments": [
        {
          "type": "raw_tx_signature",
          "raw_data": {
            "xpub": "5c6145b241b1147987565719657a0506ebb417a2e110a235a42cfb40951880f447432f930ce9fd1a6b7e51b3ddbfdc7adb57d33448f93c0defb4de630703a144",
            "derivation_path": [
              "010100000000000000",
              "0100000000000000"
            ]
          }
        }
      ]
    },
    {
      "type": "control_program",
      "asset_id": "1e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c22",
      "amount": 900000000,
      "control_program": "0014726e902e30525e01f0157f12be476c904060383b"
    },
    {
      "type": "spend_account",
      "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
      "amount": 5000000,
      "account_id": "0G1JIR6400A02"
    }
  ],
  "ttl": 0,
  "time_range": 1521625823
}

// Result
{
  "raw_transaction": "0701dfd5c8d5050201720170c9680e6dd5e9ae7f825fe7edab9fa35c119eb7feab0ab4e426c84a579daf4ef91e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c2280d293ad0302012820e9108d3ca8049800727f6a3505b3a2710dc579405dde03c250f16d9a7e1e6e787403ae7cac00c001000160015e8412e8e8c359683f1f5f3a7308b084022f1f149dab176e6e6e8daada895d0e29ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffe0c6c6944f00011600141313e974d19f3d37db29a212d75b4c763e42f433010002013d1e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c2280d293ad0301160014726e902e30525e01f0157f12be476c904060383b00013dffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffa0b095924f011600147076c737d92621e0033899a54d02fa79f362922700",
  "signing_instructions": [
    {
      "position": 0,
      "witness_components": [
        {
          "type": "raw_tx_signature",
          "quorum": 1,
          "keys": [
            {
              "xpub": "5c6145b241b1147987565719657a0506ebb417a2e110a235a42cfb40951880f447432f930ce9fd1a6b7e51b3ddbfdc7adb57d33448f93c0defb4de630703a144",
              "derivation_path": [
                "010100000000000000",
                "0100000000000000"
              ]
            }
          ],
          "signatures": null
        }
      ]
    },
    {
      "position": 1,
      "witness_components": [
        {
          "type": "raw_tx_signature",
          "quorum": 1,
          "keys": [
            {
              "xpub": "5c6145b241b1147987565719657a0506ebb417a2e110a235a42cfb40951880f447432f930ce9fd1a6b7e51b3ddbfdc7adb57d33448f93c0defb4de630703a144",
              "derivation_path": [
                "010100000000000000",
                "0300000000000000"
              ]
            }
          ],
          "signatures": null
        },
        {
          "type": "data",
          "value": "c37d5531f393bc6a3568628c0c0e17801ea452e75d604deb01403c4b161659a3"
        }
      ]
    }
  ],
  "allow_additional_actions": false
}
```

构建交易成功之后，便可以对交易进行签名`sign-transaction`，返回结果中`sign_complete`为`true`表示签名成功，将签名的交易通过`submit-transaction`提交到交易池中，等待交易被打包上链

----

## 典型合约模板解析

### 单签验证合约
  `LockWithPublicKey`源代码如下：
  ```
  contract LockWithPublicKey(publicKey: PublicKey) locks valueAmount of valueAsset {
    clause unlockWithSig(sig: Signature) {
      verify checkTxSig(publicKey, sig)
      unlock valueAmount of valueAsset
    }
  }
  ```

  - 合约编译之后的字节码为：`ae7cac`
  - 合约对应的指令码为：`TXSIGHASH SWAP CHECKSIG`

  假如合约参数如下：
  - `publicKey` : e9108d3ca8049800727f6a3505b3a2710dc579405dde03c250f16d9a7e1e6e78

  添加合约参数后的合约程序如下：`20e9108d3ca8049800727f6a3505b3a2710dc579405dde03c250f16d9a7e1e6e787403ae7cac00c0`

  对应的解锁合约的参数如下：
  - `sig` :
  ```js
  {
    "type": "raw_tx_signature",
    "raw_data": {
      "xpub": "5c6145b241b1147987565719657a0506ebb417a2e110a235a42cfb40951880f447432f930ce9fd1a6b7e51b3ddbfdc7adb57d33448f93c0defb4de630703a144",
      "derivation_path": [
        "010100000000000000",
        "0100000000000000"
      ]
    }
  }
  ```

  对应的解锁合约交易模板示例如下：
  ```js
  {
    "base_transaction": null,
    "actions": [
      {
        "output_id": "913d941faf5a19501ab4c06747fe1eb38c5ae76b74d0f5af524fc40ee6bf7116",
        "arguments": [
          {
            "type": "raw_tx_signature",
            "raw_data": {
              "xpub": "5c6145b241b1147987565719657a0506ebb417a2e110a235a42cfb40951880f447432f930ce9fd1a6b7e51b3ddbfdc7adb57d33448f93c0defb4de630703a144",
              "derivation_path": [
                "010100000000000000",
                "0100000000000000"
              ]
            }
          }
        ],
        "type": "spend_account_unspent_output"
      },
      {
        "amount": 100000000,
        "asset_id": "1e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c22",
        "control_program": "0014c5a5b563c4623018557fb299259542b8739f6bc2",
        "type": "control_program"
      },
      {
        "account_id": "0G1JIR6400A02",
        "amount": 20000000,
        "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
        "type": "spend_account"
      }
    ],
    "ttl": 0,
    "time_range": 1521625823
  }
  ```

  ----

  ### 多签验证合约
  `LockWithMultiSig`源代码如下：
  ```
  contract LockWithMultiSig(publicKey1: PublicKey,
                          publicKey2: PublicKey,
                          publicKey3: PublicKey) locks valueAmount of valueAsset {
    clause spend(sig1: Signature, sig2: Signature) {
      verify checkTxMultiSig([publicKey1, publicKey2, publicKey3], [sig1, sig2])
      unlock valueAmount of valueAsset
    }
  }
  ```
  - 合约编译之后的字节码为：`537a547a526bae71557a536c7cad`
  - 合约对应的指令码为：`3 ROLL 4 ROLL 2 TOALTSTACK TXSIGHASH 2ROT 5 ROLL 3 FROMALTSTACK SWAP CHECKMULTISIG`

  假如合约参数如下：
  - `publicKey1` : e9108d3ca8049800727f6a3505b3a2710dc579405dde03c250f16d9a7e1e6e78
  - `publicKey2` : 1f51c25decab2168835f1292a5432707ed94b90be8f5ec0d62aca1c6daa1ec55
  - `publicKey3` : e386c85178418fc72f8182111aa818ac736f3f7f1eee75ccdd7e5a057abe8fe0

  添加合约参数后的合约程序如下：（注意添加合约参数的顺序与虚拟机入栈的顺序是相反的）`20e386c85178418fc72f8182111aa818ac736f3f7f1eee75ccdd7e5a057abe8fe0201f51c25decab2168835f1292a5432707ed94b90be8f5ec0d62aca1c6daa1ec5520e9108d3ca8049800727f6a3505b3a2710dc579405dde03c250f16d9a7e1e6e78740e537a547a526bae71557a536c7cad00c0`

  对应的解锁合约的参数如下：（注意解锁合约参数的顺序，否则会执行VM失败）
  - `sig1` :
  ```js
  {
    "type": "raw_tx_signature",
    "raw_data": {
      "xpub": "5c6145b241b1147987565719657a0506ebb417a2e110a235a42cfb40951880f447432f930ce9fd1a6b7e51b3ddbfdc7adb57d33448f93c0defb4de630703a144",
      "derivation_path": [
        "010100000000000000",
        "0100000000000000"
      ]
    }
  }
  ```

  - `sig2` :
  ```js
  {
    "type": "raw_tx_signature",
    "raw_data": {
      "xpub": "e0f470a91bcd99497b4804d76264a293e257ca906938a43e34a6a0e619bbf5f1606a2866eca2ceec9f93348c577b052f30bb7ef41b4b09fff28eb1004ca1f8d5",
      "derivation_path": [
        "010200000000000000",
        "0100000000000000"
      ]
    }
  }
  ```

  对应的解锁合约交易模板示例如下：
  ```js
  {
    "base_transaction": null,
    "actions": [
      {
        "output_id": "bf6a3999cfbd25cef24ff1b19c6f2e0988de2e8ef3d40ed7c15402f2a7327bfa",
        "arguments": [
          {
            "type": "raw_tx_signature",
            "raw_data": {
              "xpub": "5c6145b241b1147987565719657a0506ebb417a2e110a235a42cfb40951880f447432f930ce9fd1a6b7e51b3ddbfdc7adb57d33448f93c0defb4de630703a144",
              "derivation_path": [
                "010100000000000000",
                "0100000000000000"
              ]
            }
          },
          {
            "type": "raw_tx_signature",
            "raw_data": {
              "xpub": "e0f470a91bcd99497b4804d76264a293e257ca906938a43e34a6a0e619bbf5f1606a2866eca2ceec9f93348c577b052f30bb7ef41b4b09fff28eb1004ca1f8d5",
              "derivation_path": [
                "010200000000000000",
                "0100000000000000"
              ]
            }
          }
        ],
        "type": "spend_account_unspent_output"
      },
      {
        "amount": 200000000,
        "asset_id": "1e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c22",
        "control_program": "0014c5a5b563c4623018557fb299259542b8739f6bc2",
        "type": "control_program"
      },
      {
        "account_id": "0G1JIR6400A02",
        "amount": 20000000,
        "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
        "type": "spend_account"
      }
    ],
    "ttl": 0,
    "time_range": 1521625823
  }
  ```

  ----

  ### pubkey哈希校验和单签验证合约
  `LockWithPublicKeyHash`源代码如下：
  ```
  contract LockWithPublicKeyHash(pubKeyHash: Hash) locks valueAmount of valueAsset {
    clause spend(pubKey: PublicKey, sig: Signature) {
      verify sha3(pubKey) == pubKeyHash
      verify checkTxSig(pubKey, sig)
      unlock valueAmount of valueAsset
    }
  }
  ```
  - 合约编译之后的字节码为：`5279aa887cae7cac`
  - 合约对应的指令码为：`2 PICK SHA3 EQUALVERIFY SWAP TXSIGHASH SWAP CHECKSIG`

  假如合约参数如下：
  - `pubKeyHash` : b3f37834dfa74174e9f0d208302e77c637cfe66c3e37fe1e1574e416b3516e89

  添加合约参数后的合约程序如下：`20b3f37834dfa74174e9f0d208302e77c637cfe66c3e37fe1e1574e416b3516e8974085279aa887cae7cac00c0`

  对应的解锁合约的参数如下：（注意解锁合约参数的顺序，否则会执行VM失败）
  - `pubKey` :
  ```js
  {
    "type": "data",
    "raw_data": {
      "value": "b3f37834dfa74174e9f0d208302e77c637cfe66c3e37fe1e1574e416b3516e89"
    }
  }
  ```

  - `sig` :
  ```js
  {
    "type": "raw_tx_signature",
    "raw_data": {
      "xpub": "5c6145b241b1147987565719657a0506ebb417a2e110a235a42cfb40951880f447432f930ce9fd1a6b7e51b3ddbfdc7adb57d33448f93c0defb4de630703a144",
      "derivation_path": [
        "010100000000000000",
        "0100000000000000"
      ]
    }
  }
  ```

  对应的解锁合约交易模板示例如下：
  ```js
  {
    "base_transaction": null,
    "actions": [
      {
        "output_id": "f98e0aa007dec76a827e924c25678e8c04b922fb28da9f5513a92787ac53725b",
        "arguments": [
          {
            "type": "data",
            "raw_data": {
              "value": "b3f37834dfa74174e9f0d208302e77c637cfe66c3e37fe1e1574e416b3516e89"
            }
          },
          {
            "type": "raw_tx_signature",
            "raw_data": {
              "xpub": "5c6145b241b1147987565719657a0506ebb417a2e110a235a42cfb40951880f447432f930ce9fd1a6b7e51b3ddbfdc7adb57d33448f93c0defb4de630703a144",
              "derivation_path": [
                "010100000000000000",
                "0100000000000000"
              ]
            }
          }
        ],
        "type": "spend_account_unspent_output"
      },
      {
        "amount": 300000000,
        "asset_id": "1e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c22",
        "control_program": "0014c5a5b563c4623018557fb299259542b8739f6bc2",
        "type": "control_program"
      },
      {
        "account_id": "0G1JIR6400A02",
        "amount": 20000000,
        "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
        "type": "spend_account"
      }
    ],
    "ttl": 0,
    "time_range": 1521625823
  }
  ```

  ----

  ### 字符串哈希校验合约
  `RevealPreimage`源代码如下：
  ```
  contract RevealPreimage(hash: Hash) locks valueAmount of valueAsset {
    clause reveal(string: String) {
      verify sha3(string) == hash
      unlock valueAmount of valueAsset
    }
  }
  ```
  - 合约编译之后的字节码为：`7caa87`
  - 合约对应的指令码为：`SWAP SHA3 EQUAL`

  假如合约参数如下：
  - `hash` : 22e829107201c6b975b1dc60b928117916285ceb4aa5c6d7b4b8cc48038083e0

  添加合约参数后的合约程序如下：`2022e829107201c6b975b1dc60b928117916285ceb4aa5c6d7b4b8cc48038083e074037caa8700c0`

  对应的解锁合约的参数如下：（注意解锁合约参数的顺序，否则会执行VM失败）
  - `string` :
  ```js
  {
    "type": "string",
    "raw_data": {
      "value": "string"
    }
  }
  ```
  或者 （`string`的16进制为`737472696e67`）
  ```js
  {
    "type": "data",
    "raw_data": {
      "value": "737472696e67"
    }
  }
  ```

  对应的解锁合约交易模板示例如下：
  ```js
  {
    "base_transaction": null,
    "actions": [
      {
        "output_id": "35fe216572bea7d81effd2fed2db1bc257f977dfa492299742a0dee2a9ae1c8e",
        "arguments": [
          {
            "type": "string",
            "raw_data": {
              "value": "string"
            }
          }
        ],
        "type": "spend_account_unspent_output"
      },
      {
        "amount": 400000000,
        "asset_id": "1e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c22",
        "control_program": "0014c5a5b563c4623018557fb299259542b8739f6bc2",
        "type": "control_program"
      },
      {
        "account_id": "0G1JIR6400A02",
        "amount": 20000000,
        "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
        "type": "spend_account"
      }
    ],
    "ttl": 0,
    "time_range": 1521625823
  }
  ```

  ----

  ### 币币交易合约
  `TradeOffer`源代码如下：
  ```
  contract TradeOffer(assetRequested: Asset,
                    amountRequested: Amount,
                    seller: Program,
                    cancelKey: PublicKey) locks valueAmount of valueAsset {
    clause trade() {
      lock amountRequested of assetRequested with seller
      unlock valueAmount of valueAsset
    }
    clause cancel(sellerSig: Signature) {
      verify checkTxSig(cancelKey, sellerSig)
      unlock valueAmount of valueAsset
    }
  }
  ```
  - 合约编译之后的字节码为：`547a6413000000007b7b51547ac1631a000000547a547aae7cac`
  - 合约对应的指令码为：`4 ROLL JUMPIF:$cancel $trade FALSE ROT ROT 1 4 ROLL CHECKOUTPUT JUMP:$_end $cancel 4 ROLL 4 ROLL TXSIGHASH SWAP CHECKSIG $_end`

  假如合约参数如下：
  - `assetRequested` : 1e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c22
  - `amountRequested` : 99
  - `seller` : 0014c5a5b563c4623018557fb299259542b8739f6bc2
  - `cancelKey` : e9108d3ca8049800727f6a3505b3a2710dc579405dde03c250f16d9a7e1e6e78

  添加合约参数后的合约程序如下：`20e9108d3ca8049800727f6a3505b3a2710dc579405dde03c250f16d9a7e1e6e78160014c5a5b563c4623018557fb299259542b8739f6bc20163201e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c22741a547a6413000000007b7b51547ac1631a000000547a547aae7cac00c0`

  对应的解锁合约的参数如下：（注意该合约包含两个`clause`，在解锁合约的时候任选其一即可，其中`clause_selector`指的是选择的解锁`clause`在合约中的位置（位置序号从`0`开始计算），此外还需注意解锁合约参数的顺序，否则会执行VM失败）
  - `clause trade` 解锁参数 :
    - `clause_selector` : 
    ```js
    {
      "type": "integer",
      "raw_data": {
        "value": 0
      }
    }
    ```
    或
    ```js
    {
      "type": "data",
      "raw_data": {
        "value": "00"
      }
    }
    ```

    - `lock`语句的交易`action`构造 : `lock amountRequested of assetRequested with seller`语句表示解锁账户需要支付对应的`amountRequested`数量的`assetRequested`资产到对应的接收对象`seller`，注意上述参数必须严格匹配，否则合约执行将失败.
    ```js
    {
      "amount": 99,
      "asset_id": "2a1861ba3a5f7bbdb98392b33289465b462fe8e9d4b9c00f78cbcb1ac20fa93f",
      "control_program": "0014c5a5b563c4623018557f299259542b8739f6bc2",
      "type": "control_program"
    },
    {
      "account_id": "0G1JJF0KG0A06",
      "amount": 99,
      "asset_id": "2a1861ba3a5f7bbdb98392b33289465b462fe8e9d4b9c00f78cbcb1ac20fa93f",
      "type": "spend_account"
    }
    ```

    对应的解锁合约交易模板示例如下：
    ```js
    // clause trade
    {
      "base_transaction": null,
      "actions": [
        {
          "output_id": "9f3a63d2f8352a6891dadd8a8337268873c84a852594f35b0b9815a4b9d56d86",
          "arguments": [
            {
              "type": "integer",
              "raw_data": {
                "value": 0
              }
            }
          ],
          "type": "spend_account_unspent_output"
        },
        {
          "amount": 99,
          "asset_id": "1e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c22",
          "control_program": "0014c5a5b563c4623018557fb299259542b8739f6bc2",
          "type": "control_program"
        },
        {
          "account_id": "0G1JIR6400A02",
          "amount": 99,
          "asset_id": "1e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c22",
          "type": "spend_account"
        },
        {
          "account_id": "0G1JIR6400A02",
          "amount": 20000000,
          "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
          "type": "spend_account"
        },
        {
          "amount": 500000000,
          "asset_id": "2a1861ba3a5f7bbdb98392b33289465b462fe8e9d4b9c00f78cbcb1ac20fa93f",
          "control_program": "0014905df16bc248790676744bab063a1ae810803bd7",
          "type": "control_program"
        }
      ],
      "ttl": 0,
      "time_range": 1521625823
    }
    ```

  - `clause cancel` 解锁参数 :（注意参数顺序）
    - `sellerSig` :
    ```js
    {
      "type": "raw_tx_signature",
      "raw_data": {
        "xpub": "5c6145b241b1147987565719657a0506ebb417a2e110a235a42cfb40951880f447432f930ce9fd1a6b7e51b3ddbfdc7adb57d33448f93c0defb4de630703a144",
        "derivation_path": [
          "010100000000000000",
          "0100000000000000"
        ]
      }
    }
    ```

    - `clause_selector` :
    ```js
    {
      "type": "integer",
      "raw_data": {
        "value": 1
      }
    }
    ```
    或
    ```js
    {
      "type": "data",
      "raw_data": {
        "value": "01"
      }
    }
    ```

    对应的解锁合约交易模板示例如下:
    ```js
    // clause cancel

    {
      "base_transaction": null,
      "actions": [
        {
          "output_id": "9f3a63d2f8352a6891dadd8a8337268873c84a852594f35b0b9815a4b9d56d86",
          "arguments": [
            {
              "type": "raw_tx_signature",
              "raw_data": {
                "xpub": "5c6145b241b1147987565719657a0506ebb417a2e110a235a42cfb40951880f447432f930ce9fd1a6b7e51b3ddbfdc7adb57d33448f93c0defb4de630703a144",
                "derivation_path": [
                  "010100000000000000",
                  "0100000000000000"
                ]
              }
            },
            {
              "type": "integer",
              "raw_data": {
                "value": 1
              }
            }
          ],
          "type": "spend_account_unspent_output"
        },
        {
          "amount": 500000000,
          "asset_id": "1e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c22",
          "control_program": "0014929ec7d92f89d74716ba9591eaea588aa1867f75",
          "type": "control_program"
        },
        {
          "account_id": "0G1JIR6400A02",
          "amount": 20000000,
          "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
          "type": "spend_account"
        }
      ],
      "ttl": 0,
      "time_range": 1521625823
    }
    ```

  ----

  ### 第三方信任机构托管合约
  `Escrow`源代码如下：
  ```
  contract Escrow(agent: PublicKey,
                sender: Program,
                recipient: Program) locks valueAmount of valueAsset {
    clause approve(sig: Signature) {
      verify checkTxSig(agent, sig)
      lock valueAmount of valueAsset with recipient
    }
    clause reject(sig: Signature) {
      verify checkTxSig(agent, sig)
      lock valueAmount of valueAsset with sender
    }
  }
  ```
  - 合约编译之后的字节码为：`537a641a000000537a7cae7cac6900c3c251557ac16328000000537a7cae7cac6900c3c251547ac1`
  - 合约对应的指令码为：`3 ROLL JUMPIF:$reject $approve 3 ROLL SWAP TXSIGHASH SWAP CHECKSIG VERIFY FALSE AMOUNT ASSET 1 5 ROLL CHECKOUTPUT JUMP:$_end $reject 3 ROLL SWAP TXSIGHASH SWAP CHECKSIG VERIFY FALSE AMOUNT ASSET 1 4 ROLL CHECKOUTPUT $_end`

  假如合约参数如下：
  - `agent` : e9108d3ca8049800727f6a3505b3a2710dc579405dde03c250f16d9a7e1e6e78
  - `sender` : 0014905df16bc248790676744bab063a1ae810803bd7
  - `recipient` : 0014929ec7d92f89d74716ba9591eaea588aa1867f75

  添加合约参数后的合约程序如下：`160014929ec7d92f89d74716ba9591eaea588aa1867f75160014905df16bc248790676744bab063a1ae810803bd720e9108d3ca8049800727f6a3505b3a2710dc579405dde03c250f16d9a7e1e6e787428537a641a000000537a7cae7cac6900c3c251557ac16328000000537a7cae7cac6900c3c251547ac100c0`

  对应的解锁合约的参数如下：（注意该合约包含两个`clause`，在解锁合约的时候任选其一即可，其中`clause_selector`指的是选择的解锁`clause`在合约中的位置（位置序号从`0`开始计算），此外还需注意解锁合约参数的顺序，否则会执行VM失败）
  - `clause approve` 解锁参数 :
    - `sellerSig` :
    ```js
    {
      "type": "raw_tx_signature",
      "raw_data": {
        "xpub": "5c6145b241b1147987565719657a0506ebb417a2e110a235a42cfb40951880f447432f930ce9fd1a6b7e51b3ddbfdc7adb57d33448f93c0defb4de630703a144",
        "derivation_path": [
          "010100000000000000",
          "0100000000000000"
        ]
      }
    }
    ```

    - `clause_selector` :
    ```js
    {
      "type": "integer",
      "raw_data": {
        "value": 0
      }
    }
    ```
    或
    ```js
    {
      "type": "data",
      "raw_data": {
        "value": "00"
      }
    }
    ```

    对应的解锁合约交易模板示例如下:（注意接收`control_program`字段必须为`recipient`，否则执行失败）
    ```js
    // clause approve
    {
      "base_transaction": null,
      "actions": [
        {
          "output_id": "15ee3368e77a0699eadaf99bbc096a48595ac975d05b10b198d902dde808e120",
          "arguments": [
            {
              "type": "raw_tx_signature",
              "raw_data": {
                "xpub": "5c6145b241b1147987565719657a0506ebb417a2e110a235a42cfb40951880f447432f930ce9fd1a6b7e51b3ddbfdc7adb57d33448f93c0defb4de630703a144",
                "derivation_path": [
                  "010100000000000000",
                  "0100000000000000"
                ]
              }
            },
            {
              "type": "integer",
              "raw_data": {
                "value": 0
              }
            }
          ],
          "type": "spend_account_unspent_output"
        },
        {
          "amount": 600000000,
          "asset_id": "1e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c22",
          "control_program": "0014929ec7d92f89d74716ba9591eaea588aa1867f75",
          "type": "control_program"
        },
        {
          "account_id": "0G1JIR6400A02",
          "amount": 20000000,
          "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
          "type": "spend_account"
        }
      ],
      "ttl": 0,
      "time_range": 1521625823
    }
    ```  

  - `clause reject` 解锁参数 :
    ```js
    {
      "type": "raw_tx_signature",
      "raw_data": {
        "xpub": "5c6145b241b1147987565719657a0506ebb417a2e110a235a42cfb40951880f447432f930ce9fd1a6b7e51b3ddbfdc7adb57d33448f93c0defb4de630703a144",
        "derivation_path": [
          "010100000000000000",
          "0100000000000000"
        ]
      }
    }
    ```

    - `clause_selector` :
    ```js
    {
      "type": "integer",
      "raw_data": {
        "value": 1
      }
    }
    ```
    或
    ```js
    {
      "type": "data",
      "raw_data": {
        "value": "01"
      }
    }
    ``` 

    对应的解锁合约交易模板示例如下:（注意接收`control_program`字段必须为`sender`，否则执行失败）
    ```js
    // clause reject
    {
      "base_transaction": null,
      "actions": [
        {
          "output_id": "15ee3368e77a0699eadaf99bbc096a48595ac975d05b10b198d902dde808e120",
          "arguments": [
            {
              "type": "raw_tx_signature",
              "raw_data": {
                "xpub": "5c6145b241b1147987565719657a0506ebb417a2e110a235a42cfb40951880f447432f930ce9fd1a6b7e51b3ddbfdc7adb57d33448f93c0defb4de630703a144",
                "derivation_path": [
                  "010100000000000000",
                  "0100000000000000"
                ]
              }
            },
            {
              "type": "integer",
              "raw_data": {
                "value": 1
              }
            }
          ],
          "type": "spend_account_unspent_output"
        },
        {
          "amount": 600000000,
          "asset_id": "1e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c22",
          "control_program": "0014905df16bc248790676744bab063a1ae810803bd7",
          "type": "control_program"
        },
        {
          "account_id": "0G1JIR6400A02",
          "amount": 20000000,
          "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
          "type": "spend_account"
        }
      ],
      "ttl": 0,
      "time_range": 1521625823
    }
    ```  

  ----

  ### 抵押贷款合约
  `LoanCollateral`源代码如下：
  ```
  contract LoanCollateral(assetLoaned: Asset,
                        amountLoaned: Amount,
                        blockHeight: Integer,
                        lender: Program,
                        borrower: Program) locks valueAmount of valueAsset {
    clause repay() {
      lock amountLoaned of assetLoaned with lender
      lock valueAmount of valueAsset with borrower
    }
    clause default() {
      verify above(blockHeight)
      lock valueAmount of valueAsset with lender
    }
  }
  ```
  - 合约编译之后的字节码为：`557a641b000000007b7b51557ac16951c3c251557ac163260000007bcd9f6900c3c251567ac1`
  - 合约对应的指令码为：`5 ROLL JUMPIF:$default $repay FALSE ROT ROT 1 5 ROLL CHECKOUTPUT VERIFY 1 AMOUNT ASSET 1 5 ROLL CHECKOUTPUT JUMP:$_end $default ROT BLOCKHEIGHT LESSTHAN VERIFY FALSE AMOUNT ASSET 1 6 ROLL CHECKOUTPUT $_end`

  假如合约参数如下：
  - `assetLoaned` : 1e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c22
  - `amountLoaned` : 88
  - `blockHeight` : 1920
  - `lender` : 0014905df16bc248790676744bab063a1ae810803bd7
  - `borrower` : 0014929ec7d92f89d74716ba9591eaea588aa1867f75

  添加合约参数后的合约程序如下：`160014929ec7d92f89d74716ba9591eaea588aa1867f75160014905df16bc248790676744bab063a1ae810803bd70280070158201e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c227426557a641b000000007b7b51557ac16951c3c251557ac163260000007bcd9f6900c3c251567ac100c0`

  对应的解锁合约的参数如下：（注意该合约包含两个`clause`，在解锁合约的时候任选其一即可，其中`clause_selector`指的是选择的解锁`clause`在合约中的位置（位置序号从`0`开始计算），此外还需注意解锁合约参数的顺序，否则会执行VM失败）
  - `clause repay` 解锁参数 :
    - `clause_selector` : 
    ```js
    {
      "type": "integer",
      "raw_data": {
        "value": 0
      }
    }
    ```
    或
    ```js
    {
      "type": "data",
      "raw_data": {
        "value": "00"
      }
    }
    ```

    - 连续两个`lock`语句的交易`action`构造 : `lock amountLoaned of assetLoaned with lender`语句表示解锁账户需要支付对应的`amountLoaned`数量的`assetLoaned`资产到对应的接收对象`lender`; `lock valueAmount of valueAsset with borrower`语句表示将合约锁定的`valueAmount`数量的`valueAsset`资产到对应的接收对象`borrower`。注意上述参数必须严格匹配，否则合约执行将失败.
    ```js
    {
      "amount": 88,
      "asset_id": "1e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c22",
      "control_program": "0014905df16bc248790676744bab063a1ae810803bd7",
      "type": "control_program"
    },
    {
      "amount": 700000000,
      "asset_id": "2ee76aeaa110308fcdfb382fa02a4a35823d5a589ffcaddb23f11f8a1fae3302",
      "control_program": "0014929ec7d92f89d74716ba9591eaea588aa1867f75",
      "type": "control_program"
    },
    {
      "account_id": "0G1JIR6400A02",
      "amount": 88,
      "asset_id": "1e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c22",
      "type": "spend_account"
    }
    ```

    对应的解锁合约交易模板示例如下：
    ```js
    // clause repay
    {
      "base_transaction": null,
      "actions": [
        {
          "output_id": "3cf74c303558b7343cfa20b32ccddfd8e66293ae1970f7612ca6cb9a006e76bc",
          "arguments": [
            {
              "type": "integer",
              "raw_data": {
                "value": 0
              }
            }
          ],
          "type": "spend_account_unspent_output"
        },
        {
          "amount": 88,
          "asset_id": "1e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c22",
          "control_program": "0014905df16bc248790676744bab063a1ae810803bd7",
          "type": "control_program"
        },
        {
          "amount": 700000000,
          "asset_id": "2ee76aeaa110308fcdfb382fa02a4a35823d5a589ffcaddb23f11f8a1fae3302",
          "control_program": "0014929ec7d92f89d74716ba9591eaea588aa1867f75",
          "type": "control_program"
        },
        {
          "account_id": "0G1JIR6400A02",
          "amount": 88,
          "asset_id": "1e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c22",
          "type": "spend_account"
        },
        {
          "account_id": "0G1JIR6400A02",
          "amount": 20000000,
          "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
          "type": "spend_account"
        }
      ],
      "ttl": 0,
      "time_range": 1521625823
    }
    ```

  - `clause default` 解锁参数 :（注意参数顺序）
    - `clause_selector` :
    ```js
    {
      "type": "integer",
      "raw_data": {
        "value": 1
      }
    }
    ```
    或
    ```js
    {
      "type": "data",
      "raw_data": {
        "value": "01"
      }
    }
    ```

    对应的解锁合约交易模板示例如下:（注意接收`control_program`字段必须为`lender`，否则执行失败）
    ```js
    // clause default
    {
      "base_transaction": null,
      "actions": [
        {
          "output_id": "3cf74c303558b7343cfa20b32ccddfd8e66293ae1970f7612ca6cb9a006e76bc",
          "arguments": [
            {
              "type": "integer",
              "raw_data": {
                "value": 1
              }
            }
          ],
          "type": "spend_account_unspent_output"
        },
        {
          "amount": 700000000,
          "asset_id": "1e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c22",
          "control_program": "0014905df16bc248790676744bab063a1ae810803bd7",
          "type": "control_program"
        },
        {
          "account_id": "0G1JIR6400A02",
          "amount": 20000000,
          "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
          "type": "spend_account"
        }
      ],
      "ttl": 0,
      "time_range": 1521625823
    }
    ```

  ----

  ### 看涨期权合约
  `CallOption`源代码如下：
  ```
  contract CallOption(strikePrice: Amount,
                    strikeCurrency: Asset,
                    seller: Program,
                    buyerKey: PublicKey,
                    blockHeight: Integer) locks valueAmount of valueAsset {
    clause exercise(buyerSig: Signature) {
      verify below(blockHeight)
      verify checkTxSig(buyerKey, buyerSig)
      lock strikePrice of strikeCurrency with seller
      unlock valueAmount of valueAsset
    }
    clause expire() {
      verify above(blockHeight)
      lock valueAmount of valueAsset with seller
    }
  }
  ```
  - 合约编译之后的字节码为：`557a6420000000547acda069547a547aae7cac69007c7b51547ac1632c000000547acd9f6900c3c251567ac1`
  - 合约对应的指令码为：`5 ROLL JUMPIF:$expire $exercise FALSE ROT ROT 1 5 ROLL CHECKOUTPUT VERIFY 1 AMOUNT ASSET 1 5 ROLL CHECKOUTPUT JUMP:$_end $expire ROT BLOCKHEIGHT LESSTHAN VERIFY FALSE AMOUNT ASSET 1 6 ROLL CHECKOUTPUT $_end`

  假如合约参数如下：
  - `strikePrice` : 199
  - `strikeCurrency` : 1e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c22
  - `seller` : 0014905df16bc248790676744bab063a1ae810803bd7
  - `buyerKey` : e9108d3ca8049800727f6a3505b3a2710dc579405dde03c250f16d9a7e1e6e78
  - `blockHeight` : 3096

  添加合约参数后的合约程序如下：`02180c20e9108d3ca8049800727f6a3505b3a2710dc579405dde03c250f16d9a7e1e6e78160014905df16bc248790676744bab063a1ae810803bd7201e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c2201c7742c557a6420000000547acda069547a547aae7cac69007c7b51547ac1632c000000547acd9f6900c3c251567ac100c0`

  对应的解锁合约的参数如下：（注意该合约包含两个`clause`，在解锁合约的时候任选其一即可，其中`clause_selector`指的是选择的解锁`clause`在合约中的位置（位置序号从`0`开始计算），此外还需注意解锁合约参数的顺序，否则会执行VM失败）
  - `clause exercise` 解锁参数 :
    - `buyerSig` : 
    ```js
    {
      "type": "raw_tx_signature",
      "raw_data": {
        "xpub": "5c6145b241b1147987565719657a0506ebb417a2e110a235a42cfb40951880f447432f930ce9fd1a6b7e51b3ddbfdc7adb57d33448f93c0defb4de630703a144",
        "derivation_path": [
          "010100000000000000",
          "0100000000000000"
        ]
      }
    }
    ```

    - `clause_selector` : 
    ```js
    {
      "type": "integer",
      "raw_data": {
        "value": 0
      }
    }
    ```
    或
    ```js
    {
      "type": "data",
      "raw_data": {
        "value": "00"
      }
    }
    ```

    - `lock`语句的交易`action`构造 : `lock strikePrice of strikeCurrency with seller`语句表示解锁账户需要支付对应的`strikePrice`数量的`strikeCurrency`资产到对应的接收对象`seller`，注意上述参数必须严格匹配，否则合约执行将失败.
    ```js
    {
      "amount": 199,
      "asset_id": "1e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c22",
      "control_program": "0014c5a5b563c4623018557fb299259542b8739f6bc2",
      "type": "control_program"
    },
    {
      "account_id": "0G1JIR6400A02",
      "amount": 199,
      "asset_id": "1e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c22",
      "type": "spend_account"
    }
    ```

    对应的解锁合约交易模板示例如下：
    ```js
    // clause exercise
    {
      "base_transaction": null,
      "actions": [
        {
          "output_id": "7afa15ad39356a7e6dc363fba823146ecb0132967f067ddfa13494a34a984518",
          "arguments": [
            {
              "type": "raw_tx_signature",
              "raw_data": {
                "xpub": "5c6145b241b1147987565719657a0506ebb417a2e110a235a42cfb40951880f447432f930ce9fd1a6b7e51b3ddbfdc7adb57d33448f93c0defb4de630703a144",
                "derivation_path": [
                  "010100000000000000",
                  "0100000000000000"
                ]
              }
            },
            {
              "type": "integer",
              "raw_data": {
                "value": 0
              }
            }
          ],
          "type": "spend_account_unspent_output"
        },
        {
          "amount": 199,
          "asset_id": "1e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c22",
          "control_program": "0014c5a5b563c4623018557fb299259542b8739f6bc2",
          "type": "control_program"
        },
        {
          "account_id": "0G1JIR6400A02",
          "amount": 199,
          "asset_id": "1e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c22",
          "type": "spend_account"
        },
        {
          "account_id": "0G1JIR6400A02",
          "amount": 20000000,
          "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
          "type": "spend_account"
        },
        {
          "amount": 800000000,
          "asset_id": "2a1861ba3a5f7bbdb98392b33289465b462fe8e9d4b9c00f78cbcb1ac20fa93f",
          "control_program": "0014905df16bc248790676744bab063a1ae810803bd7",
          "type": "control_program"
        }
      ],
      "ttl": 0,
      "time_range": 1521625823
    }
    ```

  - `clause expire` 解锁参数 :
    - `clause_selector` : 
    ```js
    {
      "type": "integer",
      "raw_data": {
        "value": 1
      }
    }
    ```
    或
    ```js
    {
      "type": "data",
      "raw_data": {
        "value": "01"
      }
    }
    ```

    对应的解锁合约交易模板示例如下:（注意接收`control_program`字段必须为`seller`，否则执行失败）
    ```js
    // clause expire
    {
      "base_transaction": null,
      "actions": [
        {
          "output_id": "7afa15ad39356a7e6dc363fba823146ecb0132967f067ddfa13494a34a984518",
          "arguments": [
            {
              "type": "integer",
              "raw_data": {
                "value": 1
              }
            }
          ],
          "type": "spend_account_unspent_output"
        },
        {
          "amount": 800000000,
          "asset_id": "1e074b22ed7ae8470c7ba5d8a7bc95e83431a753a17465e8673af68a82500c22",
          "control_program": "0014905df16bc248790676744bab063a1ae810803bd7",
          "type": "control_program"
        },
        {
          "account_id": "0G1JIR6400A02",
          "amount": 20000000,
          "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
          "type": "spend_account"
        }
      ],
      "ttl": 0,
      "time_range": 1521625823
    }
    ```    


