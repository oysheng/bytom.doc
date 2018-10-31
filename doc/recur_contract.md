# 递归合约
  
  在`equity`合约中，`lock`语句锁定资产的接收对象也可以是合约，这样的合约称为递归合约。接收对象的合约名字必须跟原合约相同，合约的参数类型和顺序也要完全匹配，但是合约的参数值是可以修改的。

## 可修改价格的币币交易合约
`PriceChanger`源代码如下：
```
contract PriceChanger(askAmount: Amount, askAsset: Asset, sellerKey: PublicKey, sellerProg: Program) locks valueAmount of valueAsset {
  clause changePrice(newAmount: Amount, newAsset: Asset, sig: Signature) {
    verify checkTxSig(sellerKey, sig)
    lock valueAmount of valueAsset with PriceChanger(newAmount, newAsset, sellerKey, sellerProg)
  }
  clause redeem() {
    lock askAmount of askAsset with sellerProg
    unlock valueAmount of valueAsset
  }
}
```
  - 合约编译之后的字节码为：`557a6432000000557a5479ae7cac6900c3c25100597a89587a89587a89587a89557a890274787e008901c07ec1633a000000007b537a51567ac1`
  - 合约对应的指令码为：`5 05 ROLL JUMPIF:$redeem $changePrice 5 05 ROLL 4 04 PICK TXSIGHASH SWAP CHECKSIG VERIFY FALSE AMOUNT ASSET 1 01 FALSE 9 09 ROLL CATPUSHDATA 8 08 ROLL CATPUSHDATA 8 08 ROLL CATPUSHDATA 8 08 ROLL CATPUSHDATA 5 05 ROLL CATPUSHDATA DATA_2 7478 CAT FALSE CATPUSHDATA DATA_1 c0 CAT CHECKOUTPUT JUMP:$_end $redeem FALSE ROT 3 03 ROLL 1 01 6 06 ROLL CHECKOUTPUT $_end`

  假如合约参数如下：
  - `askAmount` : 20000
  - `askAsset` : c6b12af8326df37b8d77c77bfa2547e083cbacde15cc48da56d4aa4e4235a3ee
  - `sellerKey` : 055539eb36abcaaf127c63ae20e3d049cd28d0f1fe569df84da3aedb018ca1bf
  - `sellerProg` : 0014dedfd406c591aa221a047a260107f877da92fec5

  添加合约参数后的合约程序如下：
  `160014dedfd406c591aa221a047a260107f877da92fec520055539eb36abcaaf127c63ae20e3d049cd28d0f1fe569df84da3aedb018ca1bf20c6b12af8326df37b8d77c77bfa2547e083cbacde15cc48da56d4aa4e4235a3ee02204e3a557a6432000000557a5479ae7cac6900c3c25100597a89587a89587a89587a89557a890274787e008901c07ec1633a000000007b537a51567ac1747800c0`

  对应的解锁合约的参数如下：（注意该合约包含两个`clause`，在解锁合约的时候任选其一即可，其中`clause_selector`指的是选择的解锁`clause`在合约中的位置（位置序号从`0`开始计算），此外还需注意解锁合约参数的顺序，否则会执行VM失败）
  - `clause changePrice` 解锁参数 :
    - `newAmount` :
    ```js
    {
      "type": "integer",
      "raw_data": {
        "value": 10000
      }
    }
    ```

    - `newAsset` :
    ```js
    {
      "type": "data",
      "raw_data": {
        "value": "3addca837514d599c509aed802c0b7671838b363a298cbcb0acb06bc24076cf4"
      }
    }
    ```

    - `sig` :
    ```js
    {
      "type": "raw_tx_signature",
      "raw_data": {
        "xpub": "a4d4f09a04371516d37e1d27f92c9cb41e4b1e7f62762cf23ed3904a9dfd2d794195862fffd00bf7ac373e5891c8d2eb660dc5ff9c040ec4e01f973bbfd31c23",
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

    合约参数构造完成之后，需要对合约`lock`语句中接收对象进行指定。该合约中的接收对象是一个新的合约程序，`lock valueAmount of valueAsset with PriceChanger(newAmount, newAsset, sellerKey, sellerProg)`语句表示合约锁定的资产被重新锁定到合约对象中，因此需要构造实例化的合约程序作为接收对象。接收合约的参数如下：
    - `newAmount` : 10000
    - `newAsset` : 3addca837514d599c509aed802c0b7671838b363a298cbcb0acb06bc24076cf4
    - `sellerKey` : 055539eb36abcaaf127c63ae20e3d049cd28d0f1fe569df84da3aedb018ca1bf
    - `sellerProg` : 0014dedfd406c591aa221a047a260107f877da92fec5

    &nbsp; 
    添加合约参数后的实例化合约程序如下： 
    `160014dedfd406c591aa221a047a260107f877da92fec520055539eb36abcaaf127c63ae20e3d049cd28d0f1fe569df84da3aedb018ca1bf203addca837514d599c509aed802c0b7671838b363a298cbcb0acb06bc24076cf40210273a557a6432000000557a5479ae7cac6900c3c25100597a89587a89587a89587a89557a890274787e008901c07ec1633a000000007b537a51567ac1747800c0`

    &nbsp;
    对应的解锁合约交易模板示例如下：
    ```js
    // clause changePrice
    {
    "base_transaction": null,
    "actions": [
        {
        "output_id": "31feac2f482f48dd99117569a3148d1b84211d41b64a980dc9ec37bda922b0c8",
        "arguments": [
            {
            "type": "integer",
            "raw_data": {
                "value": 10000
            }
            },
            {
            "type": "data",
            "raw_data": {
                "value": "3addca837514d599c509aed802c0b7671838b363a298cbcb0acb06bc24076cf4"
            }
            },
            {
            "type": "raw_tx_signature",
            "raw_data": {
                "xpub": "a4d4f09a04371516d37e1d27f92c9cb41e4b1e7f62762cf23ed3904a9dfd2d794195862fffd00bf7ac373e5891c8d2eb660dc5ff9c040ec4e01f973bbfd31c23",
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
        "amount": 900,
        "asset_id": "2a62180553e70131f668639116d6d1e29417537766b5244cbe49fa6b36c5d7b0",
        "control_program": "160014dedfd406c591aa221a047a260107f877da92fec520055539eb36abcaaf127c63ae20e3d049cd28d0f1fe569df84da3aedb018ca1bf203addca837514d599c509aed802c0b7671838b363a298cbcb0acb06bc24076cf40210273a557a6432000000557a5479ae7cac6900c3c25100597a89587a89587a89587a89557a890274787e008901c07ec1633a000000007b537a51567ac1747800c0",
        "type": "control_program"
        },
        {
        "account_id": "0ILGLSTC00A02",
        "amount": 20000000,
        "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
        "type": "spend_account"
        }
    ],
    "ttl": 0,
    "time_range": 1521625823
    }
    ```

  - `clause redeem` 解锁参数 :
    - `clause_selector` :
    ```js
    {
      "type": "integer",
      "raw_data": {
        "value": 1
      }
    }
    ```

    其对应的解锁合约交易模板示例如下:
    ```js
    // clause redeem
    {
      "base_transaction": null,
      "actions": [
        {
          "output_id": "31feac2f482f48dd99117569a3148d1b84211d41b64a980dc9ec37bda922b0c8",
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
          "amount": 20000,
          "asset_id": "c6b12af8326df37b8d77c77bfa2547e083cbacde15cc48da56d4aa4e4235a3ee",
          "control_program": "0014dedfd406c591aa221a047a260107f877da92fec5",
          "type": "control_program"
        },
        {
          "account_id": "0JPC5DBOG0A02",
          "amount": 20000,
          "asset_id": "c6b12af8326df37b8d77c77bfa2547e083cbacde15cc48da56d4aa4e4235a3ee",
          "type": "spend_account"
        },
        {
          "account_id": "0JPC5DBOG0A02",
          "amount": 20000000,
          "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
          "type": "spend_account"
        },
        {
          "amount": 900,
          "asset_id": "2a62180553e70131f668639116d6d1e29417537766b5244cbe49fa6b36c5d7b0",
          "control_program": "0014bf54f5adbbd2dc11bffb50277b5a993cec75e924",
          "type": "control_program"
        }
      ],
      "ttl": 0,
      "time_range": 1521625823
    }
    ```

## 可部分借贷的抵押贷款合约
`PartLoanCollateral`源代码如下：
```
contract PartLoanCollateral(assetLoaned: Asset,
                        amountLoaned: Amount,
                        blockHeight: Integer,
                        lender: Program,
                        borrower: Program) locks valueAmount of valueAsset {
  clause repay(amountPartLoaned: Amount) {
    if amountPartLoaned > 0 && amountPartLoaned < amountLoaned {
      lock amountPartLoaned of assetLoaned with lender
      lock valueAmount*amountPartLoaned/amountLoaned of valueAsset with borrower
      lock valueAmount-(valueAmount*amountPartLoaned/amountLoaned) of valueAsset with PartLoanCollateral(assetLoaned, amountLoaned-amountPartLoaned, blockHeight, lender, borrower)
    } else {
      lock amountLoaned of assetLoaned with lender
      lock valueAmount of valueAsset with borrower
    }
  }
  clause default() {
    verify above(blockHeight)
    lock valueAmount of valueAsset with lender
  }
}
```
  - 合约编译之后的字节码为：`567a6477000000567900a0577954799f9a916164630000000057795379515879c16951c3587995547996c2515979c16952c3767c59799555799694c251005a798959798958798957795c7994895679895579890274787e008901c07ec16963720000000070515879c16951c3c2515979c1696383000000537acd9f6900c3c251577ac1`
  - 合约对应的指令码为：`6 06 ROLL JUMPIF:$default $repay 6 06 PICK FALSE GREATERTHAN 7 07 PICK 4 04 PICK LESSTHAN BOOLAND NOT NOP JUMPIF 63000000 FALSE 7 07 PICK 3 03 PICK 1 01 8 08 PICK CHECKOUTPUT VERIFY 1 01 AMOUNT 8 08 PICK MUL 4 04 PICK DIV ASSET 1 01 9 09 PICK CHECKOUTPUT VERIFY 2 02 AMOUNT DUP SWAP 9 09 PICK MUL 5 05 PICK DIV SUB ASSET 1 01 FALSE 10 0a PICK CATPUSHDATA 9 09 PICK CATPUSHDATA 8 08 PICK CATPUSHDATA 7 07 PICK 12 0c PICK SUB CATPUSHDATA 6 06 PICK CATPUSHDATA 5 05 PICK CATPUSHDATA DATA_2 7478 CAT FALSE CATPUSHDATA DATA_1 c0 CAT CHECKOUTPUT VERIFY JUMP 72000000 FALSE 2OVER 1 01 8 08 PICK CHECKOUTPUT VERIFY 1 01 AMOUNT ASSET 1 01 9 09 PICK CHECKOUTPUT VERIFY JUMP:$_end $default 3 03 ROLL BLOCKHEIGHT LESSTHAN VERIFY FALSE AMOUNT ASSET 1 01 7 07 ROLL CHECKOUTPUT $_end`

  假如合约参数如下：
  - `assetLoaned` : c6b12af8326df37b8d77c77bfa2547e083cbacde15cc48da56d4aa4e4235a3ee
  - `amountLoaned` : 8000
  - `blockHeight` : 1900
  - `lender` : 0014dedfd406c591aa221a047a260107f877da92fec5
  - `borrower` : 0014bf54f5adbbd2dc11bffb50277b5a993cec75e924

  添加合约参数后的合约程序如下：`160014bf54f5adbbd2dc11bffb50277b5a993cec75e924160014dedfd406c591aa221a047a260107f877da92fec5026c0702401f20c6b12af8326df37b8d77c77bfa2547e083cbacde15cc48da56d4aa4e4235a3ee4c83567a6477000000567900a0577954799f9a916164630000000057795379515879c16951c3587995547996c2515979c16952c3767c59799555799694c251005a798959798958798957795c7994895679895579890274787e008901c07ec16963720000000070515879c16951c3c2515979c1696383000000537acd9f6900c3c251577ac1747800c0`

  对应的解锁合约的参数如下：（注意该合约包含两个`clause`，在解锁合约的时候任选其一即可，其中`clause_selector`指的是选择的解锁`clause`在合约中的位置（位置序号从`0`开始计算），此外还需注意解锁合约参数的顺序，否则会执行VM失败）
  - `clause changePrice` 解锁参数 :
    - `amountPartLoaned` :
    ```js
    {
      "type": "integer",
      "raw_data": {
        "value": 2000
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

    从合约中可以看出`lock`语句位于`if-else`语句块中，并且`if`语句下面的`lock`语句个数跟`else`语句下的`lock`语句个数不相同，因此合约解锁之前，需要对语句的执行流程进行预判断。当`amountPartLoaned`等于2000时，是小于`amountLoaned`的值8000，所以执行的流程是`if`语句下面合约流程。此外，`lock`语句中`valueAmount*amountPartLoaned/amountLoaned`和`valueAmount-(valueAmount*amountPartLoaned/amountLoaned)`需要用户在使用之前先计算好，假如合约锁定的资产值`valueAmount`为3000, 那么`valueAmount*amountPartLoaned/amountLoaned`的计算结果为750，而`valueAmount-(valueAmount*amountPartLoaned/amountLoaned)`的计算结果为2250。最后，该合约中的接收对象是一个新的合约程序，`lock valueAmount-(valueAmount*amountPartLoaned/amountLoaned) of valueAsset with PartLoanCollateral(assetLoaned, amountLoaned-amountPartLoaned, blockHeight, lender, borrower)`语句表示合约锁定的资产被重新锁定到合约对象中，因此需要构造实例化的合约程序作为接收对象。接收合约的参数如下：
    - `assetLoaned` : c6b12af8326df37b8d77c77bfa2547e083cbacde15cc48da56d4aa4e4235a3ee
    - `amountLoaned` : 6000
    - `blockHeight` : 1900
    - `lender` : 0014dedfd406c591aa221a047a260107f877da92fec5
    - `borrower` : 0014bf54f5adbbd2dc11bffb50277b5a993cec75e924

    &nbsp; 
    添加合约参数后的实例化合约程序如下： 
    `160014bf54f5adbbd2dc11bffb50277b5a993cec75e924160014dedfd406c591aa221a047a260107f877da92fec5026c0702701720c6b12af8326df37b8d77c77bfa2547e083cbacde15cc48da56d4aa4e4235a3ee4c83567a6477000000567900a0577954799f9a916164630000000057795379515879c16951c3587995547996c2515979c16952c3767c59799555799694c251005a798959798958798957795c7994895679895579890274787e008901c07ec16963720000000070515879c16951c3c2515979c1696383000000537acd9f6900c3c251577ac1747800c0`
    
    &nbsp;
    对应的解锁合约交易模板示例如下：
    ```js
    // clause repay
    {
      "base_transaction": null,
      "actions": [
        {
          "output_id": "a80552a346e271aaa7c71d0fa362a6b02aa844bb71a72934db64bc2b9906d0b9",
          "arguments": [
            {
              "type": "integer",
              "raw_data": {
                "value": 2000
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
          "amount": 2000,
          "asset_id": "c6b12af8326df37b8d77c77bfa2547e083cbacde15cc48da56d4aa4e4235a3ee",
          "control_program": "0014dedfd406c591aa221a047a260107f877da92fec5",
          "type": "control_program"
        },
        {
          "amount": 750,
          "asset_id": "2a62180553e70131f668639116d6d1e29417537766b5244cbe49fa6b36c5d7b0",
          "control_program": "0014bf54f5adbbd2dc11bffb50277b5a993cec75e924",
          "type": "control_program"
        },
        {
          "amount": 2250,
          "asset_id": "2a62180553e70131f668639116d6d1e29417537766b5244cbe49fa6b36c5d7b0",
          "control_program": "160014bf54f5adbbd2dc11bffb50277b5a993cec75e924160014dedfd406c591aa221a047a260107f877da92fec5026c0702701720c6b12af8326df37b8d77c77bfa2547e083cbacde15cc48da56d4aa4e4235a3ee4c83567a6477000000567900a0577954799f9a916164630000000057795379515879c16951c3587995547996c2515979c16952c3767c59799555799694c251005a798959798958798957795c7994895679895579890274787e008901c07ec16963720000000070515879c16951c3c2515979c1696383000000537acd9f6900c3c251577ac1747800c0",
          "type": "control_program"
        },
        {
          "account_id": "0JPC5DBOG0A02",
          "amount": 2000,
          "asset_id": "c6b12af8326df37b8d77c77bfa2547e083cbacde15cc48da56d4aa4e4235a3ee",
          "type": "spend_account"
        },
        {
          "account_id": "0JPC5DBOG0A02",
          "amount": 20000000,
          "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
          "type": "spend_account"
        }
      ],
      "ttl": 0,
      "time_range": 1521625823
    }
    ```

    当`amountPartLoaned`大于或等于`amountLoaned`的值8000时，执行的流程是`else`语句下面合约流程。对应的解锁合约交易模板示例如下：
    ```js
    // clause repay
    {
      "base_transaction": null,
      "actions": [
        {
          "output_id": "a80552a346e271aaa7c71d0fa362a6b02aa844bb71a72934db64bc2b9906d0b9",
          "arguments": [
            {
              "type": "integer",
              "raw_data": {
                "value": 8000
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
          "amount": 8000,
          "asset_id": "c6b12af8326df37b8d77c77bfa2547e083cbacde15cc48da56d4aa4e4235a3ee",
          "control_program": "0014dedfd406c591aa221a047a260107f877da92fec5",
          "type": "control_program"
        },
        {
          "amount": 3000,
          "asset_id": "2a62180553e70131f668639116d6d1e29417537766b5244cbe49fa6b36c5d7b0",
          "control_program": "0014bf54f5adbbd2dc11bffb50277b5a993cec75e924",
          "type": "control_program"
        },
        {
          "account_id": "0JPC5DBOG0A02",
          "amount": 8000,
          "asset_id": "c6b12af8326df37b8d77c77bfa2547e083cbacde15cc48da56d4aa4e4235a3ee",
          "type": "spend_account"
        },
        {
          "account_id": "0JPC5DBOG0A02",
          "amount": 20000000,
          "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
          "type": "spend_account"
        }
      ],
      "ttl": 0,
      "time_range": 1521625823
    }
    ```

  - `clause default` 解锁参数 :
    - `clause_selector` :
    ```js
    {
      "type": "integer",
      "raw_data": {
        "value": 1
      }
    }
    ```

    对应的解锁合约交易模板示例如下:
    ```js
    // clause default
    {
      "base_transaction": null,
      "actions": [
        {
          "output_id": "a80552a346e271aaa7c71d0fa362a6b02aa844bb71a72934db64bc2b9906d0b9",
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
          "amount": 3000,
          "asset_id": "2a62180553e70131f668639116d6d1e29417537766b5244cbe49fa6b36c5d7b0",
          "control_program": "0014905df16bc248790676744bab063a1ae810803bd7",
          "type": "control_program"
        },
        {
          "account_id": "0ILGLSTC00A02",
          "amount": 20000000,
          "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
          "type": "spend_account"
        }
      ],
      "ttl": 0,
      "time_range": 1521625823
    }
    ```

