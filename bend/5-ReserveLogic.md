# ReserveLogic

资产相关的计算逻辑库

## preparation

运算理论的铺垫

---

### 时间间隔的计算

`delta_T_year` 是时间间隔的秒数 / 一年的总秒数

> T_year = 31536000

> delta_T_year = delta_T / T_year

---
### VariableBorrowRate 浮动利率借贷

    用户以浮动利率借贷资产，将以浮动利率来计息，会实时的跟随市场波动。

    跟随市场波动，在合约中需要记录时间加权的累计值，然后以类似价格预言机的模式使用，计算任意时间区间内的真实累计利息。

### LiquidityRate 流动性收益利率 

    针对借出资产的用户的收益利率，即用户向协议注入流动性时，获得的收益率。

### LiquidityIndex 流动性收益累计值 

    收益的来源是借款人归还的利息，即浮动利率借贷，需要记录时间累计值。

## variables Reserve主要变量

### liquidityIndex

`liquidity cumulative index` 每单位 liquidity (用户往协议中注入的抵押资产)累计的本息总额。 `R_t` 是利率

    LI_t(liquidity cumulative index) = (1 + delta_T * R_t) * old_LI_t

**注意：** liquidty 池子资产流动性的数量是 amountScaled ，即任意时刻存入的抵押资产数量，都会被缩放至 t_0 池子创建时刻的数量，详细逻辑参考 [amount and amountScaled](./2-BToken.md#amount%20and%20amountScaled)

### variableBorrowIndex

`variable borrow index` 累计每单位浮动利率类型债务的本息总额。`VR_t` 代表当前的浮动利率.

    VI_t(variable borrow index) = (1 + VR_t / Tyear)^delta_Time * old_VI_t 


## methods

### updateState

更新 `liquidity cumulative index` 和 `variable borrow index` 两个变量，若有新增资产，将其中一部分存入金库(Treasury)。

```solidity

/**
   * @dev Updates the liquidity cumulative index and the variable borrow index.
   * @param reserve the reserve object
   **/
  function updateState(DataTypes.ReserveData storage reserve) internal {
    // 获取浮动债务的缩放后的数量， 即t_0时刻总债务量
    uint256 scaledVariableDebt = IDebtToken(reserve.debtTokenAddress).scaledTotalSupply();
    
    // 累计每单位浮动利率类型债务的本息总额
    uint256 previousVariableBorrowIndex = reserve.variableBorrowIndex;
    
    // 每单位 liquidity (用户往协议中注入的抵押资产)累计的本息总额。
    uint256 previousLiquidityIndex = reserve.liquidityIndex;

    // 上次更新池子的时间戳
    uint40 lastUpdatedTimestamp = reserve.lastUpdateTimestamp;

    // 更新index
    (uint256 newLiquidityIndex, uint256 newVariableBorrowIndex) = _updateIndexes(
      reserve,
      scaledVariableDebt,
      previousLiquidityIndex,
      previousVariableBorrowIndex,
      lastUpdatedTimestamp
    );
    // 如新增资产将部分存入金库
    _mintToTreasury(
      reserve,
      scaledVariableDebt,
      previousVariableBorrowIndex,
      newLiquidityIndex,
      newVariableBorrowIndex,
      lastUpdatedTimestamp
    );
  }
```

### \_updateIndexes

更新指数变量的具体逻辑。

parameters:

- reserve 需要更新的资产数据
- scaledVariableDebt 浮动债务数量（统一缩放到 t_0 时刻）
- liquidityIndex 每单位流动性的收益时间加权累计值
- variableBorrowIndex 每单位浮动利率类型债务的累计本息总额

```solidity
 /**
   * @dev Updates the reserve indexes and the timestamp of the update
   * @param reserve The reserve reserve to be updated  需要更新的资产
   * @param scaledVariableDebt The scaled variable debt t_0时刻的浮动债务数量
   * @param liquidityIndex The last stored liquidity index 每单位资产的收益时间累计值
   * @param variableBorrowIndex The last stored variable borrow index 每单位浮动利率债务累计本息
   **/
  function _updateIndexes(
    DataTypes.ReserveData storage reserve,
    uint256 scaledVariableDebt,
    uint256 liquidityIndex,
    uint256 variableBorrowIndex,
    uint40 timestamp
  ) internal returns (uint256, uint256) {
    // 之前的流动性收益率
    uint256 currentLiquidityRate = reserve.currentLiquidityRate;
    // 之前每单位资产累计本息总额
    uint256 newLiquidityIndex = liquidityIndex;
    // 每单位浮动利率债务累计本息总额
    uint256 newVariableBorrowIndex = variableBorrowIndex;

    //only cumulating if there is any income being produced
    // 收益率存在 才执行逻辑
    if (currentLiquidityRate > 0) {
        // 计算累计收益率  年化利率换算成秒利率，线性累加从上次更新至今的收益率 
        // delta_time = currentTimestamp - lastUpdateTimestamp, rate 年化利率 , totalSecondPerYear 一年的总秒数
        // 1 + rate * (delta_time / totalSecondPerYear)
        uint256 cumulatedLiquidityInterest = MathUtils.calculateLinearInterest(currentLiquidityRate, timestamp);
      
        // 更新每单位流动性本息总额 新本息总额 = 这段时间的累积收益率 * 原本息总额
        newLiquidityIndex = cumulatedLiquidityInterest.rayMul(liquidityIndex);
        
        // check 最大限制
        require(newLiquidityIndex <= type(uint128).max, Errors.RL_LIQUIDITY_INDEX_OVERFLOW);
        reserve.liquidityIndex = uint128(newLiquidityIndex);

        //as the liquidity rate might come only from stable rate loans, we need to ensure
        //that there is actual variable debt before accumulating
        // 存在浮动利率债务时（有人借了钱） 更新浮动债务的每单位累积本息总额
        if (scaledVariableDebt != 0) {
            // 将年化利率换算成秒利率，以秒为单位做复利指数计算的每单位累积本息总额 取近似值（泰勒展开）
            // (1 + ratePerSecond) ^ delta_time
            uint256 cumulatedVariableBorrowInterest = MathUtils.calculateCompoundedInterest(
            reserve.currentVariableBorrowRate,
            timestamp
        );
        // 更新浮动债务每单位累积本息总额 这段时间累积复利收益率 * 每单位浮动债务累积本息
        newVariableBorrowIndex = cumulatedVariableBorrowInterest.rayMul(variableBorrowIndex);
        require(newVariableBorrowIndex <= type(uint128).max, Errors.RL_VARIABLE_BORROW_INDEX_OVERFLOW);
        reserve.variableBorrowIndex = uint128(newVariableBorrowIndex);
      }
    }

    //solium-disable-next-line
    // 更新时间戳
    reserve.lastUpdateTimestamp = uint40(block.timestamp);
    return (newLiquidityIndex, newVariableBorrowIndex);
  }
```

### updateInterestRates

更新利率的方法，浮动利率，每单位流动性收益率

parameters:

- reserve 需要更新的目标资产数据
- reserveAddress 资产地址
- bTokenAddress bToken 地址
- liquidityAdded 增加的流动性数量
- liquidityTaken 减少的流动性数量

```solidity
struct UpdateInterestRatesLocalVars {
  uint256 availableLiquidity;
  uint256 newLiquidityRate;
  uint256 newVariableRate;
  uint256 totalVariableDebt;
}
```

```solidity
  /**
   * @dev Updates the reserve current stable borrow rate, the current variable borrow rate and the current liquidity rate
   * @param reserve The address of the reserve to be updated
   * @param liquidityAdded The amount of liquidity added to the protocol (deposit or repay) in the previous action
   * @param liquidityTaken The amount of liquidity taken from the protocol (withdraw or borrow)
   **/
  function updateInterestRates(
    DataTypes.ReserveData storage reserve,
    address reserveAddress,
    address bTokenAddress,
    uint256 liquidityAdded,
    uint256 liquidityTaken
  ) internal {
    UpdateInterestRatesLocalVars memory vars;

    //calculates the total variable debt locally using the scaled borrow amount instead
    //of borrow amount(), as it's noticeably cheaper. Also, the index has been
    //updated by the previous updateState() call

    // 债务总量（本息总额） = 债务总量（t_0时刻的总供应量，不含利息) * 浮动利率指数
    vars.totalVariableDebt = IDebtToken(reserve.debtTokenAddress).scaledTotalSupply().rayMul(
      reserve.variableBorrowIndex
    );
    // 更新利率
    // newLiquidityRate 流动性收益率（每单位流动性获取的利息）
    // newVariableRate 浮动债务的最新利率
    (vars.newLiquidityRate, vars.newVariableRate) = IInterestRate(reserve.interestRateAddress).calculateInterestRates(
      reserveAddress,
      bTokenAddress,
      liquidityAdded,
      liquidityTaken,
      vars.totalVariableDebt,
      reserve.configuration.getReserveFactor()
    );

    // 检查溢出
    require(vars.newLiquidityRate <= type(uint128).max, Errors.RL_LIQUIDITY_RATE_OVERFLOW);
    require(vars.newVariableRate <= type(uint128).max, Errors.RL_VARIABLE_BORROW_RATE_OVERFLOW);

    reserve.currentLiquidityRate = uint128(vars.newLiquidityRate);
    reserve.currentVariableBorrowRate = uint128(vars.newVariableRate);

    emit ReserveDataUpdated(
      reserveAddress,
      vars.newLiquidityRate,
      vars.newVariableRate,
      reserve.liquidityIndex,
      reserve.variableBorrowIndex
    );
  }
```

相关代码

- 计算利率的方法 [IInterestRate.calculateInterestRates()](./6-InterestRate.md#calculateInterestRates)

### getNormalizedDebt

查询每单位债务的本息总额（归一化债务数量）。

```solidity
/**
   * @dev Returns the ongoing normalized variable debt for the reserve
   * A value of 1e27 means there is no debt. As time passes, the income is accrued
   * A value of 2*1e27 means that for each unit of debt, one unit worth of interest has been accumulated
   * @param reserve The reserve object
   * @return The normalized variable debt. expressed in ray
   **/
function getNormalizedDebt(DataTypes.ReserveData storage reserve)
  internal
  view
  returns (uint256)
{
  uint40 timestamp = reserve.lastUpdateTimestamp;

  // 若最近更新在当前区块内，直接返回 variableBorrowIndex
  //solium-disable-next-line
  if (timestamp == uint40(block.timestamp)) {
    //if the index was updated in the same block, no need to perform any calculation
    return reserve.variableBorrowIndex;
  }

  // 不在同一区块内，使用复利计算这段时间的增长
  uint256 cumulated =
    MathUtils.calculateCompoundedInterest(reserve.currentVariableBorrowRate, timestamp).rayMul(
      reserve.variableBorrowIndex
    );

  return cumulated;
}
```

---
参考文档：[Dapp-learning AAVE v2](https://github.com/Dapp-Learning-DAO/Dapp-Learning/blob/main/defi/Aave/contract/6-ReserveLogic.md)