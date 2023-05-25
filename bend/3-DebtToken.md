# DebtToken
当用户从协议中贷出资产，获得 1:1 数量的 debt tokens。债务代币的大部分 ERC20 方法都被禁用，因为债务是不可转让。
- `Variable debt tokens` 浮动利率债务币, bend中没有aave 固定利率债务token的概念, bend中debtToken 即 aave v2中 `variableDebtToken`

## EIP20 Methods

虽然 debt tokens 大部分遵循 ERC20 标准，但由于债务的特殊性，没有实现 `transfer()` 和 `allowance()`（这两个方法会直接revert，不会执行）。

> `balanceOf()` 返回用户的债务数量，包含累计的利息

> `totalSupply()` 返回协议上所有用户在该资产上的总债务，包含累计利息。

## Debt Token Methods

### UNDERLYING_ASSET_ADDRESS

返回 bToken 对应的抵押资产地址，例如存入 WETH, 那么生成的 bWETH 对应的资产为 WETH

```javascript
/**
   * @dev Returns the address of the underlying asset of this bToken
   **/
  function UNDERLYING_ASSET_ADDRESS() public view returns (address) {
    return _underlyingAsset;
  }
```

### POOL

返回 LendPool 的地址

```javascript
/**
   * @dev Returns the address of the lend pool where this token is used
   **/
  function POOL() public view returns (ILendPool) {
    return _getLendPool();
}
```

### approveDelegation

`msg.sender` 授权给目标地址 `delegatee` 债务额度，承诺可代 `delegatee` 偿还一定数量的债务。

```javascript
/**
   * @dev delegates borrowing power to a user on the specific debt token
   * @param delegatee the address receiving the delegated borrowing power
   * @param amount the maximum amount being delegated. Delegation will still
   * respect the liquidation constraints (even if delegated, a delegatee cannot
   * force a delegator HF to go below 1)
   **/
  function approveDelegation(address delegatee, uint256 amount) external override {
    _borrowAllowances[_msgSender()][delegatee] = amount;
    emit BorrowAllowanceDelegated(_msgSender(), delegatee, _getUnderlyingAssetAddress(), amount);
  }
```

### borrowAllowance

查询目标地址 `fromUser` 提供给 `toUser` 可代还债务的数量

```javascript
/**
   * @dev returns the borrow allowance of the user
   * @param fromUser The user to giving allowance
   * @param toUser The user to give allowance to
   * @return the current allowance of toUser
   **/
  function borrowAllowance(address fromUser, address toUser) external view override returns (uint256) {
    return _borrowAllowances[fromUser][toUser];
  }
```

### scaledBalanceOf

返回用户的债务数量（不包含利息）

### scaledTotalSupply

返回总债务数量（不包含利息）

### getScaledUserBalanceAndSupply

返回用户的债务数量（不包含利息）和总债务数量（不包含利息）

### mint

生成浮动利率的 DebtToken 转给借贷还款人。实际mint数量是 amountScaled。

```javascript
/**
   * @dev Mints debt token to the `user` address
   * -  Only callable by the LendPool
   * @param initiator The address calling borrow   借款受益人
   * @param amount The amount of debt being minted  数量
   * @param index The variable debt index of the reserve 缩放指数
   * @return `true` if the the previous balance of the user is 0
   **/
  function mint(
    address initiator,
    address onBehalfOf,
    uint256 amount,
    uint256 index
  ) external override onlyLendPool returns (bool) {
    if (initiator != onBehalfOf) {
      _decreaseBorrowAllowance(onBehalfOf, initiator, amount);
    }
    // 还款人的之前债务总额
    uint256 previousBalance = super.balanceOf(onBehalfOf);
    // index is expressed in Ray, so:
    // amount.wadToRay().rayDiv(index).rayToWad() => amount.rayDiv(index)

    // 将贷款金额amount缩放至t_0时刻, 和btoken缩放相似 详细参考2-Btoken.md amountScale 
    uint256 amountScaled = amount.rayDiv(index);
    require(amountScaled != 0, Errors.CT_INVALID_MINT_AMOUNT);

    _mint(onBehalfOf, amountScaled);
    // 发送事件用缩放前真实数量
    emit Transfer(address(0), onBehalfOf, amount);
    emit Mint(onBehalfOf, amount, index);

    // 返回还款人是否之前有债务未偿还
    return previousBalance == 0;
  }
```

### burn

销毁amount数量的VariableDebtToken，实际burn掉amountScaled。只能被 LendingPool 调用。

```javascript

/**
   * @dev Burns user variable debt
   * - Only callable by the LendPool
   * @param user The user whose debt is getting burned
   * @param amount The amount getting burned
   * @param index The variable debt index of the reserve
   **/
  function burn(
    address user,
    uint256 amount,
    uint256 index //// reserve.variableBorrowIndex
  ) external override onlyLendPool {
    uint256 amountScaled = amount.rayDiv(index); // 销毁数量需要缩放到t_0时刻
    require(amountScaled != 0, Errors.CT_INVALID_BURN_AMOUNT);
    // 销毁数量为amountScaled 的 debt token
    _burn(user, amountScaled);

    // 发送事件 数量为缩放前的真实数量
    emit Transfer(user, address(0), amount);
    emit Burn(user, amount, index);
  }
```

### balanceOf

返回用户当前的债务数量。由于浮动利率会经常变化，所以需要利用全局记录的 `variableBorrowIndex` 来对缩放后的token数量还原。

> 债务利率不断随着池子的利用率产生变化，所以每个池子会全局记录一个 `variableBorrowIndex` 来实时更新债务和缩放数量的比例

```javascript
/**
   * @dev Calculates the accumulated debt balance of the user
   * @return The debt balance of the user
   **/
  function balanceOf(address user) public view virtual override returns (uint256) {
    uint256 scaledBalance = super.balanceOf(user);

    if (scaledBalance == 0) {
      return 0;
    }

    // 内部调用了 ReserveLogic 的 getNormalizedDebt 方法
    // 返回最新的 variableBorrowIndex
    // scaledBalance * variableBorrowIndex
    ILendPool pool = _getLendPool();
    return scaledBalance.rayMul(pool.getReserveNormalizedVariableDebt(_underlyingAsset));
  }
```

相关代码

- [getNormalizedDebt](./5-ReserveLogic.md#getNormalizedDebt)

---
参考文档：[Dapp-learning AAVE v2](https://github.com/Dapp-Learning-DAO/Dapp-Learning/blob/main/defi/Aave/contract/4-DebtToken.md)
