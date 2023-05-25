# InterestRate

默认的利率更新策略模块，负责浮动的更新

## Formal Definitions

利率相关公式定义：

- `U` 流动性利用率，债务总额 和 抵押总额 的比例
- `U_optimal` 调节利率的界限，低于此会鼓励借贷，高于此会鼓励抵押
- `R_slope1` 低于界限时，利率增长的斜率
- `R_slope2` 高于界限时，利率增长的斜率
- `R_base` 基准利率，浮动基准利率提前设定

当 `U < U_optimal` 时：

<!-- $R_{t}=R_{base}+\frac{U_t}{U_{optimal}}R_{slope1}$ -->
<img src="https://render.githubusercontent.com/render/math?math=R_{t}=R_{base}%2B\frac{U_t}{U_{optimal}}R_{slope1}" style="display: block;margin: 24px auto;" />

当 `U >= U_optimal` 时：

<!-- $R_{t}=R_{base}+R_{slope1}+\frac{U_t-U_{optimal}}{1-U_{optimal}}R_{slope2}$ -->
<img src="https://render.githubusercontent.com/render/math?math=R_{t}=R_{base}%2BR_{slope1}%2B\frac{U_t-U_{optimal}}{1-U_{optimal}}R_{slope2}" style="display: block;margin: 24px auto;" />

## 公式相关变量

所有全局变量都是 `immutable` 类型，即初始化赋值后不可更改。

`ray` 是 aave 中规定的计数方法，精度为 27 的数值类型, 即 `1ray = 1e27`

| variable name            | 公式对应        | 变量类型 |
| ------------------------ | --------------- | -------- |
| OPTIMAL_UTILIZATION_RATE | U_optimal       | ray      |
| EXCESS_UTILIZATION_RATE  | 1 - U_optimal   | ray      |
| \_baseVariableBorrowRate | R_base (浮动)   | ray      |
| \_variableRateSlope1     | R_slope1 (浮动) | ray      |
| \_variableRateSlope2     | R_slope2 (浮动) | ray      |

针对各个资产的不同利率相关参数详见 [bend borrow interest rate](https://github.com/BendDAO/bend-gitbook-portal/blob/main/lending-protocol/interest-model.md)

## methods

### constructor

利率更新策略模块初始化，会将公式相关变量初始化

```solidity
constructor(
    ILendPoolAddressesProvider provider,
    uint256 optimalUtilizationRate_,
    uint256 baseVariableBorrowRate_,
    uint256 variableRateSlope1_,
    uint256 variableRateSlope2_
  ) {
    addressesProvider = provider;
    OPTIMAL_UTILIZATION_RATE = optimalUtilizationRate_;
    EXCESS_UTILIZATION_RATE = WadRayMath.ray() - (optimalUtilizationRate_);
    _baseVariableBorrowRate = baseVariableBorrowRate_;
    _variableRateSlope1 = variableRateSlope1_;
    _variableRateSlope2 = variableRateSlope2_;
}
```

### calculateInterestRates

计算利率的具体方法。

```solidity
// 缓存计算结果的struct
struct CalcInterestRatesLocalVars {
  uint256 totalDebt;  // 债务总额
  uint256 currentVariableBorrowRate;  // 当前浮动借贷利率
  uint256 currentLiquidityRate;   // 当前流动性回报率
  uint256 utilizationRate;  // 流动性利用率 U
}
```

根据池子资产的状态和配置计算利率。函数返回 流动性回报率，固定利率，浮动利率

```solidity
/**
   * @dev Calculates the interest rates depending on the reserve's state and configurations.
   * NOTE This function is kept for compatibility with the previous DefaultInterestRateStrategy interface.
   * New protocol implementation uses the new calculateInterestRates() interface
   * @param reserve The address of the reserve
   * @param availableLiquidity The liquidity available in the corresponding bToken
   * @param totalVariableDebt The total borrowed from the reserve at a variable rate
   * @param reserveFactor The reserve portion of the interest that goes to the treasury of the market
   * @return The liquidity rate and the variable borrow rate
   **/
  function calculateInterestRates(
        address reserve,  // 池子地址
        uint256 availableLiquidity, // 可用的流动性（可借贷资产）
        uint256 totalVariableDebt, // 当前浮动利率借贷债务总额
        uint256 reserveFactor // 划入准备金池中的数量
  ) public view override returns (uint256, uint256) {
    reserve;

    CalcInterestRatesLocalVars memory vars;
    // 因为bend没有固定债务 所以总债务 = 浮动债务
    vars.totalDebt = totalVariableDebt;

    // 初始化
    vars.currentVariableBorrowRate = 0;
    vars.currentLiquidityRate = 0;

    // 流动性利用率 = totalDebt / totalLiquidity
    // totalLiquidity = 可用流动性 + 总债务（已借出的流动性）
    vars.utilizationRate = vars.totalDebt == 0 ? 0 : vars.totalDebt.rayDiv(availableLiquidity + (vars.totalDebt));

    // U > U_optimal时 提高利率 鼓励抵押， 要计算两段slope
    if (vars.utilizationRate > OPTIMAL_UTILIZATION_RATE) {
        // 超过部分的利用比例
        // excessUtilizationRateRatio = （U - U_optimal) / (1 - U_optimal)
        uint256 excessUtilizationRateRatio = (vars.utilizationRate - (OPTIMAL_UTILIZATION_RATE)).rayDiv(
            EXCESS_UTILIZATION_RATE
        );
        // 当前可变债务利率 = R_base + R_slope1 + excessUtilizationRateRatio * R_slope2
        vars.currentVariableBorrowRate =
            _baseVariableBorrowRate +
            (_variableRateSlope1) +
            (_variableRateSlope2.rayMul(excessUtilizationRateRatio));
    } else {
        // U <= U_optimal
        // 当前可变债务利率 = R_base + U / U_optimal * R_slope1
        vars.currentVariableBorrowRate =
            _baseVariableBorrowRate +
            (vars.utilizationRate.rayMul(_variableRateSlope1).rayDiv(OPTIMAL_UTILIZATION_RATE));
    }
    // 计算流动性收益率（每单位流动性获取的利息）
    // liquidityRate = overallBorrowRate * U * (100 - reserveFactor)%
    // overallBorrowRate 浮动债务利率
    // reserveFactor 是设定的划入池子准备金的百分比份额
    vars.currentLiquidityRate = _getOverallBorrowRate(totalVariableDebt, vars.currentVariableBorrowRate)
      .rayMul(vars.utilizationRate)
      .percentMul(PercentageMath.PERCENTAGE_FACTOR - (reserveFactor));

    return (vars.currentLiquidityRate, vars.currentVariableBorrowRate);
  }
```

---
参考文档：[Dapp-learning AAVE v2](https://github.com/Dapp-Learning-DAO/Dapp-Learning/blob/main/defi/Aave/contract/7-DefaultReserveInterestRateStrategy.md)