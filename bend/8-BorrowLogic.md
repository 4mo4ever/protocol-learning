# BorrowLogic
处理用户贷款还款的逻辑库
## methods
`executeBatchBorrow`, `executeBatchRepay`逻辑和`executeBorrow`, `executeRepay`没有太大区别 只是套了循环 不再展开描述
### executeBorrow

处理借贷的逻辑，接受抵押的nft，将债务和借款转给用户（onBehalfOf)，更新利率

```solidity
/**
   * @notice Implements the borrow feature. Through `borrow()`, users borrow assets from the protocol.
   * @dev Emits the `Borrow()` event.
   * @param reservesData The state of all the reserves
   * @param nftsData The state of all the nfts
   * @param params The additional parameters needed to execute the borrow function
   */
  function executeBorrow(
    ILendPoolAddressesProvider addressesProvider,
    mapping(address => DataTypes.ReserveData) storage reservesData,
    mapping(address => DataTypes.NftData) storage nftsData,
    DataTypes.ExecuteBorrowParams memory params
  ) external {
    _borrow(addressesProvider, reservesData, nftsData, params);
  }
```
内部方法 _borrow
```solidity
function _borrow(
    ILendPoolAddressesProvider addressesProvider,
    mapping(address => DataTypes.ReserveData) storage reservesData,
    mapping(address => DataTypes.NftData) storage nftsData,
    DataTypes.ExecuteBorrowParams memory params
  ) internal {
    // 判断收钱人地址是否为0地址
    require(params.onBehalfOf != address(0), Errors.VL_INVALID_ONBEHALFOF_ADDRESS);

    ExecuteBorrowLocalVars memory vars;
    // lendpool地址 由于现在bend借款都是ETH原生，所以该地址为WETHGateWay
    vars.initiator = params.initiator;

    DataTypes.ReserveData storage reserveData = reservesData[params.asset];
    DataTypes.NftData storage nftData = nftsData[params.nftAsset];

    // update state MUST BEFORE get borrow amount which is depent on latest borrow index
    // 更新资产状态变量
    reserveData.updateState();

    // Convert asset amount to ETH
    vars.reserveOracle = addressesProvider.getReserveOracle();
    vars.nftOracle = addressesProvider.getNFTOracle();
    vars.loanAddress = addressesProvider.getLendPoolLoan();
    
    // 获取id 为nftTokenId的nft的loanId
    vars.loanId = ILendPoolLoan(vars.loanAddress).getCollateralLoanId(params.nftAsset, params.nftTokenId);

    vars.totalSupply = IERC721EnumerableUpgradeable(params.nftAsset).totalSupply();
    require(vars.totalSupply <= nftData.maxSupply, Errors.LP_NFT_SUPPLY_NUM_EXCEED_MAX_LIMIT);
    require(params.nftTokenId <= nftData.maxTokenId, Errors.LP_NFT_TOKEN_ID_EXCEED_MAX_LIMIT);
    // 检查各参数合法性
    ValidationLogic.validateBorrow(
      params.onBehalfOf,
      params.asset,
      params.amount,
      reserveData,
      params.nftAsset,
      nftData,
      vars.loanAddress,
      vars.loanId,
      vars.reserveOracle,
      vars.nftOracle
    );
    // 此id的nft第一次借贷
    if (vars.loanId == 0) {
        // 第一次需要把用户nft转过来
        IERC721Upgradeable(params.nftAsset).safeTransferFrom(vars.initiator, address(this), params.nftTokenId);

        // 创建贷款信息
        vars.loanId = ILendPoolLoan(vars.loanAddress).createLoan(
            vars.initiator,
            params.onBehalfOf,
            params.nftAsset,
            params.nftTokenId,
            nftData.bNftAddress,
            params.asset,
            params.amount,
            reserveData.variableBorrowIndex
        );
    } else {
        // 不是第一次借贷  直接更新借贷信息
        ILendPoolLoan(vars.loanAddress).updateLoan(
            vars.initiator,
            vars.loanId,
            params.amount,
            0,
            reserveData.variableBorrowIndex
        );
    }

    // 将债务token mint给onBehalfOf
    IDebtToken(reserveData.debtTokenAddress).mint(
      vars.initiator,
      params.onBehalfOf,
      params.amount,
      reserveData.variableBorrowIndex
    );

    // update interest rate according latest borrow amount (utilizaton)
    // 更新资产利率
    reserveData.updateInterestRates(params.asset, reserveData.bTokenAddress, 0, params.amount);

    // 从btoken池子中把借贷的资产转给lendpool（wethGateway）
    IBToken(reserveData.bTokenAddress).transferUnderlyingTo(vars.initiator, params.amount);

    // 发送借款事件
    emit Borrow(
      vars.initiator,
      params.asset,
      params.amount,
      params.nftAsset,
      params.nftTokenId,
      params.onBehalfOf,
      reserveData.currentVariableBorrowRate,
      vars.loanId,
      params.referralCode
    );
  }
```
相关代码

- [reserve.updateState()](./5-ReserveLogic.md#updateState)
- [LendPoolLoan.createLoan()](./10-LendPoolLoan.md#createLoan)
- [LendPoolLoan.updateLoan()](./10-LendPoolLoan.md#updateLoan)
- [reserve.updateInterestRates()](./5-ReserveLogic.md#updateInterestRates)
- [debtToken.mint()](./3-DebtToken.md#mint)


### executeRepay
处理还款的逻辑
```solidity
/**
   * @notice Implements the borrow feature. Through `repay()`, users repay assets to the protocol.
   * @dev Emits the `Repay()` event.
   * @param reservesData The state of all the reserves
   * @param nftsData The state of all the nfts
   * @param params The additional parameters needed to execute the repay function
   */
  function executeRepay(
    ILendPoolAddressesProvider addressesProvider,
    mapping(address => DataTypes.ReserveData) storage reservesData,
    mapping(address => DataTypes.NftData) storage nftsData,
    DataTypes.ExecuteRepayParams memory params
  ) external returns (uint256, bool) {
    return _repay(addressesProvider, reservesData, nftsData, params);
  }
```
内部结构参数
```solidity
struct RepayLocalVars {
    address initiator; //wethgateway
    address poolLoan;   // poolloan地址
    address onBehalfOf; // 接收人地址
    uint256 loanId;     // nft对应id的贷款id
    bool isUpdate;      // 是否被更新， 用于判断还款是否完成
    uint256 borrowAmount;   // 贷款金额
    uint256 repayAmount;    // 还款金额
  }
```
内部方法 _repay
```solidity
function _repay(
    ILendPoolAddressesProvider addressesProvider,
    mapping(address => DataTypes.ReserveData) storage reservesData,
    mapping(address => DataTypes.NftData) storage nftsData,
    DataTypes.ExecuteRepayParams memory params
  ) internal returns (uint256, bool) {
    RepayLocalVars memory vars;
    vars.initiator = params.initiator;
    // 缓存poolloan地址
    vars.poolLoan = addressesProvider.getLendPoolLoan();
    // 获取id 为nftTokenId的nft的loanId
    vars.loanId = ILendPoolLoan(vars.poolLoan).getCollateralLoanId(params.nftAsset, params.nftTokenId);

    // 检查该nft是否被抵押
    require(vars.loanId != 0, Errors.LP_NFT_IS_NOT_USED_AS_COLLATERAL);

    // 借款信息
    DataTypes.LoanData memory loanData = ILendPoolLoan(vars.poolLoan).getLoan(vars.loanId);

    DataTypes.ReserveData storage reserveData = reservesData[loanData.reserveAsset];
    DataTypes.NftData storage nftData = nftsData[loanData.nftAsset];

    // update state MUST BEFORE get borrow amount which is depent on latest borrow index
    // 更新资产的状态变量
    reserveData.updateState();

    // 获取真实借贷金额 非缩放
    (, vars.borrowAmount) = ILendPoolLoan(vars.poolLoan).getLoanReserveBorrowAmount(vars.loanId);
    
    // 检验参数有效性
    ValidationLogic.validateRepay(reserveData, nftData, loanData, params.amount, vars.borrowAmount);


    vars.repayAmount = vars.borrowAmount;
    vars.isUpdate = false;
    // 如果还款金额小于借贷金额 更新参数
    if (params.amount < vars.repayAmount) {
      vars.isUpdate = true;
      // 更新已还款金额
      vars.repayAmount = params.amount;
    }
    // 如果没还完 更新贷款信息
    if (vars.isUpdate) {
      ILendPoolLoan(vars.poolLoan).updateLoan(
        vars.initiator,
        vars.loanId,
        0,
        vars.repayAmount,
        reserveData.variableBorrowIndex
      );
    } else {
      // 还完了， 清除该id的nft的贷款信息， 销毁bnft
      ILendPoolLoan(vars.poolLoan).repayLoan(
        vars.initiator,
        vars.loanId,
        nftData.bNftAddress,
        vars.repayAmount,
        reserveData.variableBorrowIndex
      );
    }
    // 销毁对应数量的debttoken 数量为缩放至t_0的repayamount
    IDebtToken(reserveData.debtTokenAddress).burn(loanData.borrower, vars.repayAmount, reserveData.variableBorrowIndex);

    // update interest rate according latest borrow amount (utilizaton)
    // 更新利率
    reserveData.updateInterestRates(loanData.reserveAsset, reserveData.bTokenAddress, vars.repayAmount, 0);

    // transfer repay amount to bToken
    // 还款的钱转入btoken池子
    IERC20Upgradeable(loanData.reserveAsset).safeTransferFrom(
      vars.initiator,
      reserveData.bTokenAddress,
      vars.repayAmount
    );

    // transfer erc721 to borrower
    // 如果钱还完了 抵押的nft还给borrower
    if (!vars.isUpdate) {
      IERC721Upgradeable(loanData.nftAsset).safeTransferFrom(address(this), loanData.borrower, params.nftTokenId);
    }
    // 发送还款事件
    emit Repay(
      vars.initiator,
      loanData.reserveAsset,
      vars.repayAmount,
      loanData.nftAsset,
      loanData.nftTokenId,
      loanData.borrower,
      vars.loanId
    );

    return (vars.repayAmount, !vars.isUpdate);
  }
```
相关代码

- [reserve.updateState()](./5-ReserveLogic.md#updateState)
- [LendPoolLoan.repayLoan()](./10-LendPoolLoan.md#repayLoan)
- [LendPoolLoan.updateLoan()](./10-LendPoolLoan.md#updateLoan)
- [reserve.updateInterestRates()](./5-ReserveLogic.md#updateInterestRates)
- [debtToken.burn()](./3-DebtToken.md#burn)