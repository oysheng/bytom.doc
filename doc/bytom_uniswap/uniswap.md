# bytom的uniswap合约实现模型

## uniswap简介

uniswap实现了一个恒定乘积做市商的模型。

### 兑换模型

初始化信息：
X : Y = 1000 / 1
M(invariant) = X * Y = 10 * 10000 = 100000
交易手续费为 0.1%

1） 用 X 来兑换 Y

假如用户A用 1 个 X 来兑换 Y，则可兑换的Y数量为：
手续费（X）为： 1 * 0.1% = 0.001
此时参与计算的 X 池为： 10 + 1 - 0.001 = 10.999
此时参与计算的 Y 池为： 100000 ／ 10.999 = 9091.735
用户A收到的 Y 为： 10000 - 9091.735 = 908.265


兑换完成之后池子变化为：
X 池为： 10.999 + 0.001 = 11
Y 池为： 9091.735
新的流动性池： M = 11 * 9091.735 = 100009.085



2） 用 Y 来兑换 X

基于步骤1）的基础上，假如用户B用 1000 个 Y 来兑换 X，则可兑换的X数量为：

手续费（Y）为： 1000 * 0.1% = 1
此时 Y 池为： 9091.735 + 1000 - 1 = 10090.735
此时 X 池为： 100009.085 ／ 10090.735 = 9.910
用户B收到的 X 为： 11 - 9.910 = 1.090


兑换完成之后池子变化为：
X 池为： 9.910
Y 池为： 10090.735 + 1 = 10091.735
新的流动性池： M = 9.910 * 10091.735 = 100009.09385


### 流动性池管理

假如当前的流动性池为：

M = X * Y 
  = 9.910 * 10091.735 
  = 100009.09385

当前的X和Y的比例为：
X : Y = 9.910 : 10091.735
      = 1 : 1018.338

1) 增加流动性池

假如用户C使用 1 个 X，此时等比例的 Y 为 1018.338，来增加流动性池

用户C能得到的流动性代币为：
C_liquidity = current_C_X * total_liquidity / reserve_X
            = 1 * 100009.09385 / 9.910
            = 1018.338


此时的流动性池变化为：
X池为：9.910 + 1 = 10.910
Y池为：10091.735 + 1018.338 = 11110.073
流动性池M为：M = 10.910 * 11110.073 = 121210.896


2）减少流动性池

基于步骤1）的基础上，假如用户C使用 1 个 X，此时等比例的 Y 为 1018.338，来减少流动性池

用户C将的流动性代币销毁：
Liquidity = total_liquidity - C_liquidity
          = 121210.896 - 1018.338
          = 120192.558

C(X) = C_liquidity * total_X / total_liquidity 
     = 1018.338 * 10.910 / 121210.896
     = 1


C(Y) = C(X) * total_liquidity / total_Y 
     = 1 * 121210.896 / 11110.073
     = 1018.338

此时的流动性池变化为：
X池为：total_X - C(X) = 10.910 - 1 = 9.910
Y池为：total_Y - C(Y) = 11110.073 - 1018.338 = 10091.735
流动性池M为：M = 9.910 * 10091.735 = 100009.09385


## bytom的uniswap实现模型

bytom是utxo模型的区块链，合约中没有单独的存储空间来保存中间变量信息，所以在实现uniswap模型的时候需要借助中心化的存储来保存一些变量信息，然后通过中心化的秘钥来验证参数的合法性。

bytom系统中的uniswap流程图如下：
  
  ![image](images/uniswap_flow.png)


上述流程图大致描述了uniswap的交互流程，里面主要包含了3个合约：X资产池、Y资产池、M资产的总流动性池。其主要功能描述如下：

1）X资产池 和 Y资产池是两个相同模型的币币交易合约，里面包含了币币兑换和减少流动性的功能函数，由于减少流动性需要两个合约联动的操作，但是两个合约联动的操作在UTXO模型上实现上比较困难，所以中间的异步操作需要中心化的控制

2）M资产池合约主要用于分发（issue）流动性资产，该合约主要使用了等比例的资产X和资产Y来兑换资产M（equity合约可以实现使用两个及以上的资产来兑换一个资产，但是目前没法实现使用一个资产来兑换多个资产），主要用于增加流动性的过程中

3）从合约上来看，减少流动性其实不是单独的一个合约，而是位于X资产池 和 Y资产池合约模版中，这是因为减少流动性是使用流动性资产M来兑换资产池的X和Y。此外，流动性的回收／销毁是有管理员来控制的，至于是否销毁操作，根据接收program来设置。


### UniswapTrade合约（币币兑换合约）

资产池X和资产池Y的合约模版如下：
```
contract UniswapTrade(assetX: Asset,
                     assetM: Asset,
                     contractX: Program,
                     agentKey: PublicKey,
                     manager: Program,
                     managerKey: PublicKey) locks amountY of assetY {
  clause trade(unlockAmount: Amount, poolAmountX: Amount, poolAmountY: Amount, totalAmountM: Amount, agentSig: Signature) {
    verify unlockAmount > 0 && poolAmountX > 0 && poolAmountY > 0 && totalAmountM > 0
    verify poolAmountX * poolAmountY == totalAmountM
    define fee: Integer = unlockAmount/1000
    define newPoolAmountX: Integer = poolAmountX + unlockAmount - fee
    define newPoolAmountY: Integer = totalAmountM / newPoolAmountX
    define gain: Integer = poolAmountY - newPoolAmountY
    verify checkTxSig(agentKey, agentSig)
    if gain < amountY {
      lock unlockAmount of assetX with contractX
      lock amountY - gain of assetY with UniswapTrade(assetX, assetM, contractX, agentKey, manager, managerKey)
      unlock gain of assetY
    } else {
      lock unlockAmount of assetX with contractX
      unlock amountY of assetY
    }
  }
  clause remove(unlockLiquidityAmount: Amount, poolAmountX: Amount, poolAmountY: Amount, totalAmountM: Amount, agentSig: Signature) {
    verify unlockLiquidityAmount > 0 && poolAmountX > 0 && poolAmountY > 0 && totalAmountM > 0
    verify poolAmountX * poolAmountY == totalAmountM
    define liquidity: Integer = unlockLiquidityAmount * poolAmountY / totalAmountM
    verify checkTxSig(agentKey, agentSig)
    if liquidity < amountY {
      lock unlockLiquidityAmount of assetM with manager
      lock amountY - liquidity of assetY with UniswapTrade(assetX, assetM, contractX, agentKey, manager, managerKey)
      unlock liquidity of assetY
    } else {
      lock unlockLiquidityAmount of assetX with manager
      unlock amountY of assetY
    }
  }
  clause cancel(managerSig: Signature) {
    verify checkTxSig(managerKey, managerSig)
    unlock amountY of assetY
  }
}
```

合约说明如下：
1）trade 币币交易，用户只需要输入兑换的金额X，poolAmountX、poolAmountX、totalAmountM可以通过统计对应合约中的utxo的金额统计结果来输入，同时使用第三方签名来验证poolAmountX、poolAmountX、totalAmountM是否正确，否则交易不予通过

2）remove 减少流动性，流动性提供者用户推出流动性池的时候调用，由于资产池X和资产池Y分属于两个不同的合约，所以在减少流动性返回资产对应比例的XY资产时，不一定时原子交易，需要中心化来控制两笔交易同时执行（因为checkoutput需要验证位置信息，后续可以根据合约改造，将两笔交易放在同一笔交易中）。同时这里的流动性池资产是放到 manager账户中，正常情况下应该是销毁资产，可以根据实际情况来调整

3）cancel 取消合约，合约的管控措施


### AddLiquidity合约（issue合约）

流动性池X的合约模版如下：
```
contract AddLiquidity(assetX: Asset,
                      assetY: Asset,
                      agentKey: PublicKey,
                      contractX: Program,
                      contractY: Program) locks amountM of assetM {
  clause add(newPoolAmountX: Amount, newPoolAmountY: Amount, 
    oldPoolAmountM: Amount, oldPoolAmountX: Amount, oldPoolAmountY: Amount, agentSig: Signature) {
    verify newPoolAmountX > 0 && newPoolAmountY > 0
    verify oldPoolAmountM > 0 && oldPoolAmountX > 0 && oldPoolAmountY > 0
    verify oldPoolAmountX * oldPoolAmountY == oldPoolAmountM
    verify oldPoolAmountX / oldPoolAmountY == newPoolAmountX / newPoolAmountY
    // verify oldPoolAmountX * newPoolAmountY == newPoolAmountX * oldPoolAmountY
    verify checkTxSig(agentKey, agentSig)
    lock newPoolAmountX of assetX with contractX
    lock newPoolAmountY of assetY with contractY
    unlock newPoolAmountX*newPoolAmountY of assetM
  }
}
```

合约说明如下：
1）add 增加流动性，用户需要输入等比例的 X:Y，即 newPoolAmountX: newPoolAmountY == oldPoolAmountX : oldPoolAmountY， 由于参数是clause解锁合约的参数，所以也使用了 agentSig 和 agentKey 来对输入的值进行验证，防止用户随意操作， contractX 和 contractY 对应的是资产池 X 和 资产池 Y 对应的合约，后续可以通过定义合约模版的形式来解决里面递归调用的问题


合约注意事项如下：

1） UniswapTrade 其实是 poolX 和 poolY 共同使用的合约模版，contractX 和  contractY 分别表示对应的合约程序，这么设计是为了防止合约编译器递归调用问题，使用合约参数模式可以解决该递归问题

2）AddLiquidity 合约中的 contractX 和 contractY 也是为了解决递归问题而设计的

3）agentSig 是中心化的管控，用于保证合约执行过程中，各个池子的变化在用户输入的情况下也能保持输入数据的正确性


### uniswap交易

流程图如下：
  
  ![image](images/uniswap_trade.jpg)


### 增加流动性

流程图如下：
  
  ![image](images/invest_liquidity.png)


### 减少流动性

流程图如下：
  
  ![image](images/divest_liquidity.png)