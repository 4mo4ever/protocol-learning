# SupplyLogic
处理用户抵押/赎回资产的逻辑库

## methods

### executeDeposit

处理资产抵押的逻辑 用户存入一定数量的资产，转给用户（onBehalfOf)当时等价值的btoken

```solidity
/**
   * @notice Implements the supply feature. Through `deposit()`, users deposit assets to the protocol.
   * @dev Emits the `Deposit()` event.
   * @param reservesData The state of all the reserves
   * @param params The additional parameters needed to execute the deposit function
   */
  function executeDeposit(
    mapping(address => DataTypes.ReserveData) storage reservesData,
    DataTypes.ExecuteDepositParams memory params
  ) external {
    // 接受人不能为0地址
    require(params.onBehalfOf != address(0), Errors.VL_INVALID_ONBEHALFOF_ADDRESS);

    // 根据抵押资产地址获取对应资产的各种信息
    DataTypes.ReserveData storage reserve = reservesData[params.asset];
    // 获取资产对应的bToken
    address bToken = reserve.bTokenAddress;
    // 判断btoken是否存在
    require(bToken != address(0), Errors.VL_INVALID_RESERVE_ADDRESS);
    // 验证抵押是否符合条件 抵押数量不能为0，资产必须为激活状态 且没有冻结
    ValidationLogic.validateDeposit(reserve, params.amount);

    // 更新资产的状态变量
    reserve.updateState();
    // 更新资产利率
    reserve.updateInterestRates(params.asset, bToken, params.amount, 0);

    // 抵押资产转入协议中资产对应的btoken地址
    IERC20Upgradeable(params.asset).safeTransferFrom(params.initiator, bToken, params.amount);

    // mint btoken给用户  用户得到缩放至t_0的amount数量的btoken，具体解释看 2-BToken.md
    IBToken(bToken).mint(params.onBehalfOf, params.amount, reserve.liquidityIndex);

    // 广播质押事件
    emit Deposit(params.initiator, params.asset, params.amount, params.onBehalfOf, params.referralCode);
  }
```
相关代码

- [reserve.updateState()](./5-ReserveLogic.md#updateState)
- [reserve.updateInterestRates()](./5-ReserveLogic.md#updateInterestRates)
- [BToken.mint()](./2-BToken.md#mint)


### executeWithdraw
处理资产赎回的逻辑 `executeDeposit`的反向操作 销毁btoken，返还原资产给用户
```solidity
/**
   * @notice Implements the supply feature. Through `withdraw()`, users withdraw assets from the protocol.
   * @dev Emits the `Withdraw()` event.
   * @param reservesData The state of all the reserves
   * @param params The additional parameters needed to execute the withdraw function
   */
  function executeWithdraw(
    mapping(address => DataTypes.ReserveData) storage reservesData,
    DataTypes.ExecuteWithdrawParams memory params
  ) external returns (uint256) {
    // 接收资产的地址不能为 0地址
    require(params.to != address(0), Errors.VL_INVALID_TARGET_ADDRESS);

    // 根据抵押资产地址获取对应资产的各种信息
    DataTypes.ReserveData storage reserve = reservesData[params.asset];
    // 获取btoken的地址
    address bToken = reserve.bTokenAddress;
    // 判断btoken地址不为0地址
    require(bToken != address(0), Errors.VL_INVALID_RESERVE_ADDRESS);

    // 获取用户atoken对应的实际资产数量 （本息总和）
    uint256 userBalance = IBToken(bToken).balanceOf(params.initiator);

    // 要赎回的资产数量
    uint256 amountToWithdraw = params.amount;

    // 如果值为uint256的最大值 默认赎回全部资产
    if (params.amount == type(uint256).max) {
      amountToWithdraw = userBalance;
    }
    // 验证参数是否合法
    ValidationLogic.validateWithdraw(reserve, amountToWithdraw, userBalance);
    
    // 更新资产的状态变量
    reserve.updateState();
    // 更新资产利率
    reserve.updateInterestRates(params.asset, bToken, 0, amountToWithdraw);
    // 销毁btoken 将资产转给to
    IBToken(bToken).burn(params.initiator, params.to, amountToWithdraw, reserve.liquidityIndex);

    // 广播赎回事件
    emit Withdraw(params.initiator, params.asset, amountToWithdraw, params.to);

    return amountToWithdraw;
  }
```

相关代码

- [reserve.updateState()](./5-ReserveLogic.md#updateState)
- [reserve.updateInterestRates()](./5-ReserveLogic.md#updateInterestRates)
- [BToken.burn()](./2-BToken.md#burn)