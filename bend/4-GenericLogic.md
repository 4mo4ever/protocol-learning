# GenericLogic
主要验证计算 用户清算相关的状态逻辑的库
## methods

- calculateNftDebtData
计算nft相关债务 价格需要转换成eth单位
```js
function calculateNftDebtData(
    address reserveAddress,
    DataTypes.ReserveData storage reserveData,
    address loanAddress,
    uint256 loanId,
    address reserveOracle
  ) internal view returns (uint256, uint256) {
    CalculateLoanDataVars memory vars;

    // all asset price has converted to ETH based, unit is in WEI (18 decimals)
    // 精度转换
    vars.reserveDecimals = reserveData.configuration.getDecimals();
    vars.reserveUnit = 10**vars.reserveDecimals;
    // 获取资产对eth价格
    vars.reserveUnitPrice = IReserveOracleGetter(reserveOracle).getAssetPrice(reserveAddress);
    // 获取loanid的nft借贷金额
    (, vars.totalDebtInReserve) = ILendPoolLoan(loanAddress).getLoanReserveBorrowAmount(loanId);
    // 将贷款金额转换成eth单位
    vars.totalDebtInETH = (vars.totalDebtInReserve * vars.reserveUnitPrice) / vars.reserveUnit;

    return (vars.totalDebtInETH, vars.totalDebtInReserve);
  }
```

- calculateNftCollateralData
计算nft的抵押金额
```js
function calculateNftCollateralData(
    address reserveAddress,
    DataTypes.ReserveData storage reserveData,
    address nftAddress,
    DataTypes.NftData storage nftData,
    address reserveOracle,
    address nftOracle
  ) internal view returns (uint256, uint256) {
    reserveData;
    nftData;

    CalculateLoanDataVars memory vars;

    // calculate total collateral balance for the nft
    // all asset price has converted to ETH based, unit is in WEI (18 decimals)
    // 获取nft/eth价格兑价格
    vars.nftUnitPrice = INFTOracleGetter(nftOracle).getAssetPrice(nftAddress);
    vars.totalCollateralInETH = vars.nftUnitPrice;

    if (reserveAddress != address(0)) {
      vars.reserveDecimals = reserveData.configuration.getDecimals();
      vars.reserveUnit = 10**vars.reserveDecimals;
      // 获取借贷资产对eth价格
      vars.reserveUnitPrice = IReserveOracleGetter(reserveOracle).getAssetPrice(reserveAddress);
      // nft价格 * 精度 / 资产对eth价格
      vars.totalCollateralInReserve = (vars.totalCollateralInETH * vars.reserveUnit) / vars.reserveUnitPrice;
    }

    return (vars.totalCollateralInETH, vars.totalCollateralInReserve);
  }
```

- calculatLoanLiquidatePrice
计算nft清算价格
```js
function calculateLoanLiquidatePrice(
    uint256 loanId,
    address reserveAsset,
    DataTypes.ReserveData storage reserveData,
    address nftAsset,
    DataTypes.NftData storage nftData,
    address poolLoan,
    address reserveOracle,
    address nftOracle
  )
    internal
    view
    returns (
      uint256,
      uint256,
      uint256
    )
  {
    CalcLiquidatePriceLocalVars memory vars;

    /*
     * 0                   CR                  LH                  100
     * |___________________|___________________|___________________|
     *  <       Borrowing with Interest        <
     * CR: Callteral Ratio;
     * LH: Liquidate Threshold;
     * Liquidate Trigger: Borrowing with Interest > thresholdPrice;
     * Liquidate Price: (100% - BonusRatio) * NFT Price;
     */

    vars.reserveDecimals = reserveData.configuration.getDecimals();
    // 获取贷款金额
    (, vars.borrowAmount) = ILendPoolLoan(poolLoan).getLoanReserveBorrowAmount(loanId);

    (vars.ltv, vars.liquidationThreshold, vars.liquidationBonus) = nftData.configuration.getCollateralParams();
    // 获取nft对eth价格
    vars.nftPriceInETH = INFTOracleGetter(nftOracle).getAssetPrice(nftAsset);
    // 获取贷款资产对eth价格
    vars.reservePriceInETH = IReserveOracleGetter(reserveOracle).getAssetPrice(reserveAsset);
    // nft对贷款资产价格 
    vars.nftPriceInReserve = ((10**vars.reserveDecimals) * vars.nftPriceInETH) / vars.reservePriceInETH;
    // 清算阀值价格 = 价格 * 清算百分比
    vars.thresholdPrice = vars.nftPriceInReserve.percentMul(vars.liquidationThreshold);

    if (vars.liquidationBonus < PercentageMath.PERCENTAGE_FACTOR) {
      // 清算价格 = nft价格 * （100% - bonus）
      vars.liquidatePrice = vars.nftPriceInReserve.percentMul(PercentageMath.PERCENTAGE_FACTOR - vars.liquidationBonus);
    }
    // 清算价格小于借款金额 清算价格 = 借款金额
    if (vars.liquidatePrice < vars.borrowAmount) {
      vars.liquidatePrice = vars.borrowAmount;
    }

    return (vars.borrowAmount, vars.thresholdPrice, vars.liquidatePrice);
  }
```

- calculateHealthFactorFromBalances  
计算用户的健康因子, 计算公式为:  
( 总质押资产 * 清算阀值 ) / 当前债务总值 
```js
function calculateHealthFactorFromBalances(
    uint256 totalCollateral,
    uint256 totalDebt,
    uint256 liquidationThreshold
  ) internal pure returns (uint256) {
    if (totalDebt == 0) return type(uint256).max;

    return (totalCollateral.percentMul(liquidationThreshold)).wadDiv(totalDebt);
  }
```

-  calculateAvailableBorrows
计算用户可借贷的资产总值. 
计算公式为: ( 用户总质押资产 * 借贷系数 ) - 已借贷资产总值
```js
function calculateAvailableBorrows(
    uint256 totalCollateral,
    uint256 totalDebt,
    uint256 ltv
  ) internal pure returns (uint256) {
    uint256 availableBorrows = totalCollateral.percentMul(ltv);

    if (availableBorrows < totalDebt) {
      return 0;
    }

    availableBorrows = availableBorrows - totalDebt;
    return availableBorrows;
  }
```

- calculateLoanAuctionEndTimestamp
计算拍卖和赎回的结束时间
```js
function calculateLoanAuctionEndTimestamp(
    DataTypes.NftData storage nftData,
    DataTypes.LoanData memory loanData,
    uint256 pauseStartTime,
    uint256 pauseDurationTime
  ) internal view returns (uint256 auctionEndTimestamp, uint256 redeemEndTimestamp) {
    uint256 extraDuration = 0;

    if ((pauseDurationTime > 0) && (loanData.bidStartTimestamp <= pauseStartTime)) {
      extraDuration = pauseDurationTime;
    }

    auctionEndTimestamp =
      loanData.bidStartTimestamp +
      extraDuration +
      (nftData.configuration.getAuctionDuration() * 1 hours);

    redeemEndTimestamp =
      loanData.bidStartTimestamp +
      extraDuration +
      (nftData.configuration.getRedeemDuration() * 1 hours);
  }
```

- calculateLoanBidFine
计算罚款金额
```js
    function calculateLoanBidFine(
        address reserveAsset,
        DataTypes.ReserveData storage reserveData,
        address nftAsset,
        DataTypes.NftData storage nftData,
        DataTypes.LoanData memory loanData,
        address poolLoan,
        address reserveOracle
    ) internal view returns (uint256, uint256) {
        nftAsset;
        // 没有人竞价
        if (loanData.bidPrice == 0) {
        return (0, 0);
        }

        CalcLoanBidFineLocalVars memory vars;

        vars.reserveDecimals = reserveData.configuration.getDecimals();
        // 获取资产折合eth价格（bend中为weth）
        vars.reservePriceInETH = IReserveOracleGetter(reserveOracle).getAssetPrice(reserveAsset);
        vars.baseBidFineInReserve = (1 ether * 10**vars.reserveDecimals) / vars.reservePriceInETH;

        // 最小罚款0.2eth
        vars.minBidFinePct = nftData.configuration.getMinBidFine();
        vars.minBidFineInReserve = vars.baseBidFineInReserve.percentMul(vars.minBidFinePct);
        // 获取债务金额
        (, vars.debtAmount) = ILendPoolLoan(poolLoan).getLoanReserveBorrowAmount(loanData.loanId);
        // 
        vars.bidFineInReserve = vars.debtAmount.percentMul(nftData.configuration.getRedeemFine());
        // 二者取大值
        if (vars.bidFineInReserve < vars.minBidFineInReserve) {
        vars.bidFineInReserve = vars.minBidFineInReserve;
        }

        return (vars.minBidFineInReserve, vars.bidFineInReserve);
    }
```

- calculateLoanData
计算nft贷款信息
```js
/**
   * @dev Calculates the nft loan data.
   * this includes the total collateral/borrow balances in Reserve,
   * the Loan To Value, the Liquidation Ratio, and the Health factor.
   * @param reserveData Data of the reserve
   * @param nftData Data of the nft
   * @param reserveOracle The price oracle address of reserve
   * @param nftOracle The price oracle address of nft
   * @return The total collateral and total debt of the loan in Reserve, the ltv, liquidation threshold and the HF
   **/
  function calculateLoanData(
    address reserveAddress,
    DataTypes.ReserveData storage reserveData,
    address nftAddress,
    DataTypes.NftData storage nftData,
    address loanAddress,
    uint256 loanId,
    address reserveOracle,
    address nftOracle
  )
    internal
    view
    returns (
      uint256,
      uint256,
      uint256
    )
  {
    CalculateLoanDataVars memory vars;
    // 获取 贷款价值比，清算阀值
    (vars.nftLtv, vars.nftLiquidationThreshold, ) = nftData.configuration.getCollateralParams();

    // calculate total borrow balance for the loan
    // 存在债务
    if (loanId != 0) {
      // 获取以eth计价的nft债务
      (vars.totalDebtInETH, vars.totalDebtInReserve) = calculateNftDebtData(
        reserveAddress,
        reserveData,
        loanAddress,
        loanId,
        reserveOracle
      );
    }

    // calculate total collateral balance for the nft
    // 获取nft价值（eth计价）
    (vars.totalCollateralInETH, vars.totalCollateralInReserve) = calculateNftCollateralData(
      reserveAddress,
      reserveData,
      nftAddress,
      nftData,
      reserveOracle,
      nftOracle
    );

    // calculate health by borrow and collateral
    // 计算健康因子
    vars.healthFactor = calculateHealthFactorFromBalances(
      vars.totalCollateralInReserve,
      vars.totalDebtInReserve,
      vars.nftLiquidationThreshold
    );

    return (vars.totalCollateralInReserve, vars.totalDebtInReserve, vars.healthFactor);
  }
```