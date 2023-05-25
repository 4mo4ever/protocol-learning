# LendPool

协议最主要的入口合约，大部分情况下，用户与此合约交互（bend目前都操作ETH, 需要转换原生ETH和wrapETH, 实际用户交互的是`WETHGateway`）.

> deposit, borrow, withdraw 和 repay 方法仅针对 ERC20 类型代币。 
> `batchBorrow`, `batchRepay`本质是套了循环的`borrow`和`repay`,内部细节不再展开。
## methods

主要的交互接口。

### deposit

存入资产，将**相同数量**的 aToken 转给调用者（`onBehalfOf`，一般是 `msg.sender`）。

**注意：**

1. 存入资产之前需要确保其有足够数量的 `allowance()` ，若不足需要先调用它的 `approve()` 授权给 `LendPool`
2. 转给用户的 bToken 查询余额与 amount 相等，但实际上 mint 的数量要进行缩放，详见 [bToken.mint](./2-BToken.md#mint)

| Parameter Name | Type    | Description                                                |
| -------------- | ------- | ---------------------------------------------------------- |
| asset          | address | 抵押资产的 token 地址                                      |
| amount         | uint256 | 抵押资产的数量                                             |
| onBehalfOf     | address | aToken 转账的目标地址，即债权人，默认填写 `msg.sender`     |
| referralCode   | uint16  | 推介码，用于广播的自定义事件代码，当用户直接调用时推荐写 0 |

```javascript
/**
   * @dev Deposits an `amount` of underlying asset into the reserve, receiving in return overlying bTokens.
   * - E.g. User deposits 100 USDC and gets in return 100 bUSDC
   * @param asset The address of the underlying asset to deposit
   * @param amount The amount to be deposited
   * @param onBehalfOf The address that will receive the bTokens, same as msg.sender if the user
   *   wants to receive them on his own wallet, or a different address if the beneficiary of bTokens
   *   is a different wallet
   * @param referralCode Code used to register the integrator originating the operation, for potential rewards.
   *   0 if the action is executed directly by the user, without any middle-man
   **/
function deposit(
    address asset,
    uint256 amount,
    address onBehalfOf,
    uint16 referralCode
  ) external override nonReentrant whenNotPaused {
    SupplyLogic.executeDeposit(
      _reserves,
      DataTypes.ExecuteDepositParams({
        initiator: _msgSender(),
        asset: asset,
        amount: amount,
        onBehalfOf: onBehalfOf,
        referralCode: referralCode
      })
    );
  }
```

相关代码

- [DataTypes](#DataTypes)

- [SupplyLogic.executeDeposit](./7-SupplyLogic.md#executeDeposit)

### withdraw

赎回 `amount` 数量抵押资产，销毁相应数量的 `bToken`。例如赎回抵押的 100 USDC 资产，则会销毁 100 bUSDC。

| Parameter Name | Type    | Description                                            |
| -------------- | ------- | ------------------------------------------------------ |
| asset          | address | 抵押资产的 token 地址                                  |
| amount         | uint256 | 赎回的数量，如果使用 `type(uint).max` 则会赎回最大数量 |
| to             | address | 赎回资产的转账的目标地址                               |

```solidity
/**
 * @dev Withdraws an `amount` of underlying asset from the reserve, burning the equivalent bTokens owned
 * E.g. User has 100 bUSDC, calls withdraw() and receives 100 USDC, burning the 100 bUSDC
 * @param asset The address of the underlying asset to withdraw
 * @param amount The underlying amount to be withdrawn
 *   - Send the value type(uint256).max in order to withdraw the whole bToken balance
 * @param to Address that will receive the underlying, same as msg.sender if the user
 *   wants to receive it on his own wallet, or a different address if the beneficiary is a
 *   different wallet
 * @return The final amount withdrawn
 **/
function withdraw(
    address asset,
    uint256 amount,
    address to
) external override nonReentrant whenNotPaused returns (uint256) {
    return
      SupplyLogic.executeWithdraw(
        _reserves,
        DataTypes.ExecuteWithdrawParams({initiator: _msgSender(), asset: asset, amount: amount, to: to})
      );
}
```

相关代码

- [DataTypes](#DataTypes)
- [SupplyLogic.executeWithdraw](./7-SupplyLogic.md#executeWithdraw)


### borrow

选择浮动利率或者固定利率，调用方法后借贷资产转给调用者(`msg.sender`)，债务由 `onBehalfOf` 承担，一般也是 `msg.sender` 。调用者即借款人必须有足够的抵押资产，或被授予了足够的信用额度。信用额度即为 AToken 拥有者对其他用户授予的 allowance 数量。

parameters:

| Parameter Name   | Type    | Description                                             |
| ---------------- | ------- | ------------------------------------------------------- |
| asset            | address | 借贷资产的 token 地址                                   |
| amount           | uint256 | 借贷资产的数量                                          |
| nftAsset         | address | 抵押nft的地址                                          |
| nftTokenId       | address | 抵押nft的id                                           |
| onBehalfOf       | address | debtToken 转账地址，即债务偿还人，默认填写 `msg.sender` |
| referralCode     | uint16  | 用于广播的自定义事件代码，当用户直接调用时推荐写 0      |

债务偿还人和债务受益人可以不同，比如抵押资产在冷钱包中，将信用额度转让给热钱包，便于操作。

```solidity
/**
   * @dev Allows users to borrow a specific `amount` of the reserve underlying asset
   * - E.g. User borrows 100 USDC, receiving the 100 USDC in his wallet
   *   and lock collateral asset in contract
   * @param asset The address of the underlying asset to borrow
   * @param amount The amount to be borrowed
   * @param nftAsset The address of the underlying nft used as collateral
   * @param nftTokenId The token ID of the underlying nft used as collateral
   * @param onBehalfOf Address of the user who will receive the loan. Should be the address of the borrower itself
   * calling the function if he wants to borrow against his own collateral
   * @param referralCode Code used to register the integrator originating the operation, for potential rewards.
   *   0 if the action is executed directly by the user, without any middle-man
   **/
function borrow(
    address asset,
    uint256 amount,
    address nftAsset,
    uint256 nftTokenId,
    address onBehalfOf,
    uint16 referralCode
  ) external override nonReentrant whenNotPaused {
    BorrowLogic.executeBorrow(
      _addressesProvider,
      _reserves,
      _nfts,
      DataTypes.ExecuteBorrowParams({
        initiator: _msgSender(),
        asset: asset,
        amount: amount,
        nftAsset: nftAsset,
        nftTokenId: nftTokenId,
        onBehalfOf: onBehalfOf,
        referralCode: referralCode
      })
    );
  }
```



相关代码

- [DataTypes](#DataTypes)
- [BorrowLogic.executeBorrow](./8-BorrowLogic.md#executeBorrow)

### repay

偿还债务，并销毁相应的 debtToken。

parameters:

| Parameter Name | Type    | Description                                  |
| -------------- | ------- | -------------------------------------------- |
| nftAsset       | address | 借贷nft的地址                        |
| nftTokenId     | address |  借贷nft的tokenId                   |
| amount         | uint256 | 偿还的数量，使用 `uint(-1)` 表示偿还所有债务 |

```solidity
/**
   * @notice Repays a borrowed `amount` on a specific reserve, burning the equivalent loan owned
   * - E.g. User repays 100 USDC, burning loan and receives collateral asset
   * @param nftAsset The address of the underlying NFT used as collateral
   * @param nftTokenId The token ID of the underlying NFT used as collateral
   * @param amount The amount to repay
   **/
function repay(
    address nftAsset,
    uint256 nftTokenId,
    uint256 amount
  ) external override nonReentrant whenNotPaused returns (uint256, bool) {
    return
      BorrowLogic.executeRepay(
        _addressesProvider,
        _reserves,
        _nfts,
        DataTypes.ExecuteRepayParams({
          initiator: _msgSender(),
          nftAsset: nftAsset,
          nftTokenId: nftTokenId,
          amount: amount
        })
      );
  }
```

相关代码

- [DataTypes](#DataTypes)
- [BorrowLogic.executeRepay](./8-BorrowLogic.md#executeRepay)

### auction

拍卖不健康的抵押nft

parameters:

| Parameter Name | Type    | Description                                  |
| -------------- | ------- | -------------------------------------------- |
| nftAsset       | address | 借贷nft的地址                        |
| nftTokenId     | address |  借贷nft的tokenId                   |
| bidPrice         | uint256 | bidder的出价 |

```solidity
/**
   * @dev Function to auction a non-healthy position collateral-wise
   * - The bidder want to buy collateral asset of the user getting liquidated
   * @param nftAsset The address of the underlying NFT used as collateral
   * @param nftTokenId The token ID of the underlying NFT used as collateral
   * @param bidPrice The bid price of the bidder want to buy underlying NFT
   * @param onBehalfOf Address of the user who will get the underlying NFT, same as msg.sender if the user
   *   wants to receive them on his own wallet, or a different address if the beneficiary of NFT
   *   is a different wallet
   **/
function auction(
    address nftAsset,
    uint256 nftTokenId,
    uint256 bidPrice,
    address onBehalfOf
  ) external override nonReentrant whenNotPaused {
    LiquidateLogic.executeAuction(
      _addressesProvider,
      _reserves,
      _nfts,
      _buildLendPoolVars(),
      DataTypes.ExecuteAuctionParams({
        initiator: _msgSender(),
        nftAsset: nftAsset,
        nftTokenId: nftTokenId,
        bidPrice: bidPrice,
        onBehalfOf: onBehalfOf
      })
    );
  }
```

相关代码

- [DataTypes](#DataTypes)
- [LiquidateLogic.executeAuction](./9-LiquidateLogic.md#executeAuction)

### redeem
赎回处于拍卖状态的不健康nft msg。sender必须是贷款的借款人，只能在赎回到期前触发
parameters:

| Parameter Name | Type    | Description                                  |
| -------------- | ------- | -------------------------------------------- |
| nftAsset       | address | 借贷nft的地址                        |
| nftTokenId     | address |  借贷nft的tokenId                   |
| amount         | uint256 | 还款金额 |
| bidFine         | uint256 | 出价罚款 |

```solidity
/**
   * @notice Redeem a NFT loan which state is in Auction
   * - E.g. User repays 100 USDC, burning loan and receives collateral asset
   * @param nftAsset The address of the underlying NFT used as collateral
   * @param nftTokenId The token ID of the underlying NFT used as collateral
   * @param amount The amount to repay the debt
   * @param bidFine The amount of bid fine
   **/
function redeem(
    address nftAsset,
    uint256 nftTokenId,
    uint256 amount,
    uint256 bidFine
  ) external override nonReentrant whenNotPaused returns (uint256) {
    return
      LiquidateLogic.executeRedeem(
        _addressesProvider,
        _reserves,
        _nfts,
        _buildLendPoolVars(),
        DataTypes.ExecuteRedeemParams({
          initiator: _msgSender(),
          nftAsset: nftAsset,
          nftTokenId: nftTokenId,
          amount: amount,
          bidFine: bidFine
        })
      );
  }
```

相关代码

- [DataTypes](#DataTypes)
- [LiquidateLogic.executeRedeem](./9-LiquidateLogic.md#executeRedeem)

### liquidate
清算处于拍卖状态不健康（health factory < 1）的nft，清算人出价 获得被清算用户的抵押资产
parameters:

| Parameter Name | Type    | Description                                  |
| -------------- | ------- | -------------------------------------------- |
| nftAsset       | address | 借贷nft的地址                        |
| nftTokenId     | address |  借贷nft的tokenId                   |
| amount         | uint256 | 用于偿还债务的额外金额，一般为0 |

```solidity
/**
   * @dev Function to liquidate a non-healthy position collateral-wise
   * - The caller (liquidator) buy collateral asset of the user getting liquidated, and receives
   *   the collateral asset
   * @param nftAsset The address of the underlying NFT used as collateral
   * @param nftTokenId The token ID of the underlying NFT used as collateral
   **/
function liquidate(
    address nftAsset,
    uint256 nftTokenId,
    uint256 amount
  ) external override nonReentrant whenNotPaused returns (uint256) {
    return
      LiquidateLogic.executeLiquidate(
        _addressesProvider,
        _reserves,
        _nfts,
        _buildLendPoolVars(),
        DataTypes.ExecuteLiquidateParams({
          initiator: _msgSender(),
          nftAsset: nftAsset,
          nftTokenId: nftTokenId,
          amount: amount
        })
      );
  }
```

相关代码

- [DataTypes](#DataTypes)
- [LiquidateLogic.executeLiquidate](./9-LiquidateLogic.md#executeLiquidate)

## Struct

### DataTypes

LendPool 中的主要数据类型。

- ReserveData 资产的主要数据变量
- NftData nft的数据变量
- ReserveConfigurationMap 资产的设置，以 bitmap 形式存储，即用一个 unit256 位数字存储，不同位数对应不同的配置。
- NftConfigurationMap nft的配置，以 bitmap 形式存储。
```solidity

library DataTypes {
  struct ReserveData {
    //stores the reserve configuration
    ReserveConfigurationMap configuration;
    //the liquidity index. Expressed in ray
    uint128 liquidityIndex;
    //variable borrow index. Expressed in ray
    uint128 variableBorrowIndex;
    //the current supply rate. Expressed in ray
    uint128 currentLiquidityRate;
    //the current variable borrow rate. Expressed in ray
    uint128 currentVariableBorrowRate;
    uint40 lastUpdateTimestamp;
    //tokens addresses
    address bTokenAddress;
    address variableDebtTokenAddress;
    //address of the interest rate strategy
    address interestRateStrategyAddress;
    //the id of the reserve. Represents the position in the list of the active reserves
    uint8 id; // 这个id如果放到lastUpdateTimestamp这个变量后面, 应该可以节省一些插槽位置
  }

  struct NftData {
    //stores the nft configuration
    NftConfigurationMap configuration;
    //address of the bNFT contract nft对应的bnft地址
    address bNftAddress;
    //the id of the nft. Represents the position in the list of the active nfts
    uint8 id;
    uint256 maxSupply;
    uint256 maxTokenId;
  }

  // 这里把多个变量合并成一个256bit的数字, 很节省空间
  struct ReserveConfigurationMap {
    //bit 0-15: LTV
    //bit 16-31: Liq. threshold
    //bit 32-47: Liq. bonus
    //bit 48-55: Decimals
    //bit 56: Reserve is active
    //bit 57: reserve is frozen
    //bit 58: borrowing is enabled
    //bit 59: stable rate borrowing enabled
    //bit 60-63: reserved
    //bit 64-79: reserve factor
    uint256 data;
  }

  struct NftConfigurationMap {
    //bit 0-15: LTV
    //bit 16-31: Liq. threshold
    //bit 32-47: Liq. bonus
    //bit 56: NFT is active
    //bit 57: NFT is frozen
    uint256 data;
  }

}
```