# BEND contract guide
bend借贷模块设计上基本参考了AAVE v2

## contract constructure

### Main contracts

主要合约

#### - LendPool

Bend 协议最主要的入口合约，因为bend目前全部使用ETH借贷抵押，所以用户基本都与`WETHGateway`交互。

- deposit()
- withdraw()
- borrow()
- repay()
- auction()
- redeem()
- liquidate()

详细内容请戳这里 :point_right: [LendPool](./11-LendPool.md)

#### - LendPoolAddressesProvider

主要的地址存储合约，针对特定的不同市场都有不同的该合约。外部合约调用该合约能得到最新的合约地址（bend其他模块的合约地址）。

#### - LendPoolAddressesProviderRegistry

存储一个列表，罗列出不同市场的 `LendPoolAddressesProvider` 合约地址。

#### - BToken

bend 中的计息代币，当用户存入抵押资产时获得 1：1 对应的 btoken（比如存入 100DAI 抵押，获得 100BDai）。BToken 基本参照 ERC20 标准，增加了以下接口：

- scaledBalanceOf()
- getScaledUserBalanceAndSupply()
- scaledTotalSupply()

详细内容请戳这里 :point_right: [BToken](./2-BToken.md)

#### - Debt Tokens
相比aave， bend简化了债务token模型，去除了稳定利率债务的概念，只保留了可变债务token。
当用户从协议中贷出资产，获得 1:1 数量的 debt tokens。债务代币的大部分 ERC20 方法都被禁用，因为债务是不可转让。

详细内容请戳这里 :point_right: [DebtToken](./3-DebtToken.md)

### Supporting contracts

辅助合约

#### - LendPoolConfigurator

为 LendPool 合约提供配置功能。每项配置的改变都会发送事件到链上，任何人可见。


#### - InterestRate

利率更新策略合约。

详细内容请戳这里 :point_right: [InterestRate](./6-InterestRate.md)

#### - Price Oracle Provider

价格预言机提供合约。

#### - Library contracts

依赖库合约。

#### - WETHGateway

用户使用 ETH 作为资产不能直接和 LendPool 进行交互，需要使用该合约转换成 WETH，代理操作。

#### - Treasury

保留了aave中的协议金库，bToken 中每存入抵押品都会分出一定比例的资产作为准备金？，并且提供了提取准备金到金库的方法，暂时不明具体的使用规则。

---
参考文档
[dapp-learning aave v2](https://github.com/Dapp-Learning-DAO/Dapp-Learning/tree/main/defi/Aave)

[bend lending protocol doc](https://docs.benddao.xyz/developers/lending-protocol/protocol-overview)

[bend portal](https://github.com/BendDAO/bend-gitbook-portal/tree/main/lending-protocol)

