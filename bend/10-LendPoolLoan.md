# LendPoolLoan
维护nft和贷款关系，存储贷款信息
## method
主要交互接口

### createLoan
创建贷款 存储贷款信息
```js
function createLoan(
    address initiator, // lendpool address
    address onBehalfOf, // 贷款接受地址
    address nftAsset,   // nft地址
    uint256 nftTokenId, // nft id
    address bNftAddress, // nft 对应的 bnft地址
    address reserveAsset, // 贷款资产
    uint256 amount,      // 金额
    uint256 borrowIndex     // 缩放指数（贷款累计利率） 
  ) external override onlyLendPool returns (uint256) {
    // 检查该nft没有已经存在的贷款
    require(_nftToLoanIds[nftAsset][nftTokenId] == 0, Errors.LP_NFT_HAS_USED_AS_COLLATERAL);

    // index is expressed in Ray, so:
    // amount.wadToRay().rayDiv(index).rayToWad() => amount.rayDiv(index)
    // 将贷款金额缩放至t_0
    uint256 amountScaled = amount.rayDiv(borrowIndex);

    uint256 loanId = _loanIdTracker.current();
    _loanIdTracker.increment();

    _nftToLoanIds[nftAsset][nftTokenId] = loanId;

    // transfer underlying NFT asset to pool and mint bNFT to onBehalfOf
    // 借贷的nft转到池子里抵押
    IERC721Upgradeable(nftAsset).safeTransferFrom(_msgSender(), address(this), nftTokenId);
    // 生成bnft给借贷人
    IBNFT(bNftAddress).mint(onBehalfOf, nftTokenId);

    // Save Info
    // 存储信息
    DataTypes.LoanData storage loanData = _loans[loanId];
    loanData.loanId = loanId;
    loanData.state = DataTypes.LoanState.Active;
    loanData.borrower = onBehalfOf;
    loanData.nftAsset = nftAsset;
    loanData.nftTokenId = nftTokenId;
    loanData.reserveAsset = reserveAsset;
    loanData.scaledAmount = amountScaled;

    //用户贷款+1
    _userNftCollateral[onBehalfOf][nftAsset] += 1;
    // nft系列贷款数量+1
    _nftTotalCollateral[nftAsset] += 1;
    // 创建贷款事件
    emit LoanCreated(initiator, onBehalfOf, loanId, nftAsset, nftTokenId, reserveAsset, amount, borrowIndex);

    return (loanId);
  }
```


### updateLoan
在增加贷款/偿还后更新信息
```js
function updateLoan(
    address initiator,
    uint256 loanId,
    uint256 amountAdded, // 追加贷款金额
    uint256 amountTaken, // 还款金额
    uint256 borrowIndex
  ) external override onlyLendPool {
    // Must use storage to change state
    DataTypes.LoanData storage loan = _loans[loanId];

    // Ensure valid loan state
    require(loan.state == DataTypes.LoanState.Active, Errors.LPL_INVALID_LOAN_STATE);

    uint256 amountScaled = 0;
    // 追加贷款 更新债务金额
    if (amountAdded > 0) {
      // 缩放后金额 += 缩放后追加金额
      amountScaled = amountAdded.rayDiv(borrowIndex);
      require(amountScaled != 0, Errors.LPL_INVALID_LOAN_AMOUNT);

      loan.scaledAmount += amountScaled;
    }
    // 部分还款 更新债务
    if (amountTaken > 0) {
      amountScaled = amountTaken.rayDiv(borrowIndex);
      require(amountScaled != 0, Errors.LPL_INVALID_TAKEN_AMOUNT);
      // 缩放后债务 -= 缩放后还款金额
      require(loan.scaledAmount >= amountScaled, Errors.LPL_AMOUNT_OVERFLOW);
      loan.scaledAmount -= amountScaled;
    }
    // 贷款更新事件
    emit LoanUpdated(
      initiator,
      loanId,
      loan.nftAsset,
      loan.nftTokenId,
      loan.reserveAsset,
      amountAdded,
      amountTaken,
      borrowIndex
    );
  }
```

### auctionLoan
创建或更新拍卖出价
```js
/**
   * @inheritdoc ILendPoolLoan
   */
  function auctionLoan(
    address initiator,
    uint256 loanId,
    address onBehalfOf,
    uint256 bidPrice,   // 出价
    uint256 borrowAmount, 
    uint256 borrowIndex
  ) external override onlyLendPool {
    // Must use storage to change state
    DataTypes.LoanData storage loan = _loans[loanId];
    address previousBidder = loan.bidderAddress;
    uint256 previousPrice = loan.bidPrice;

    // Ensure valid loan state
    if (loan.bidStartTimestamp == 0) {
      require(loan.state == DataTypes.LoanState.Active, Errors.LPL_INVALID_LOAN_STATE);

      loan.state = DataTypes.LoanState.Auction;
      loan.bidStartTimestamp = block.timestamp;
      loan.firstBidderAddress = onBehalfOf;
    } else {
      require(loan.state == DataTypes.LoanState.Auction, Errors.LPL_INVALID_LOAN_STATE);

      require(bidPrice > loan.bidPrice, Errors.LPL_BID_PRICE_LESS_THAN_HIGHEST_PRICE);
    }
    // 更新信息
    loan.bidBorrowAmount = borrowAmount;
    loan.bidderAddress = onBehalfOf;
    loan.bidPrice = bidPrice;
    // 创建拍卖事件
    emit LoanAuctioned(
      initiator,
      loanId,
      loan.nftAsset,
      loan.nftTokenId,
      loan.bidBorrowAmount,
      borrowIndex,
      onBehalfOf,
      bidPrice,
      previousBidder,
      previousPrice
    );
  }
```

### redeemLoan
用户还款赎回nft 拍卖信息清除
```js
/**
   * @inheritdoc ILendPoolLoan
   */
  function redeemLoan(
    address initiator,
    uint256 loanId,
    uint256 amountTaken,  // 还款金额
    uint256 borrowIndex
  ) external override onlyLendPool {
    // Must use storage to change state
    DataTypes.LoanData storage loan = _loans[loanId];

    // Ensure valid loan state
    require(loan.state == DataTypes.LoanState.Auction, Errors.LPL_INVALID_LOAN_STATE);
    // 将还款金额缩放至t_0时刻
    uint256 amountScaled = amountTaken.rayDiv(borrowIndex);
    require(amountScaled != 0, Errors.LPL_INVALID_TAKEN_AMOUNT);
    // 更新nft对应缩放债务 债务 -= 债务 - 缩放后的还款金额  
    require(loan.scaledAmount >= amountScaled, Errors.LPL_AMOUNT_OVERFLOW);
    loan.scaledAmount -= amountScaled;
    // 清除拍卖信息（拍卖结束）
    loan.state = DataTypes.LoanState.Active;
    loan.bidStartTimestamp = 0;
    loan.bidBorrowAmount = 0;
    loan.bidderAddress = address(0);
    loan.bidPrice = 0;
    loan.firstBidderAddress = address(0);
    // 赎回事件
    emit LoanRedeemed(initiator, loanId, loan.nftAsset, loan.nftTokenId, loan.reserveAsset, amountTaken, borrowIndex);
  }
```


### liquidateLoan
nft清算
```js
/**
   * @inheritdoc ILendPoolLoan
   */
  function liquidateLoan(
    address initiator,
    uint256 loanId,
    address bNftAddress,
    uint256 borrowAmount,
    uint256 borrowIndex
  ) external override onlyLendPool {
    // Must use storage to change state
    DataTypes.LoanData storage loan = _loans[loanId];

    // Ensure valid loan state
    require(loan.state == DataTypes.LoanState.Auction, Errors.LPL_INVALID_LOAN_STATE);

    _handleBeforeLoanRepaid(loan.nftAsset, loan.nftTokenId);

    // state changes and cleanup
    // NOTE: these must be performed before assets are released to prevent reentrance
    // 清除nft贷款状态
    _loans[loanId].state = DataTypes.LoanState.Defaulted;
    _loans[loanId].bidBorrowAmount = borrowAmount;
    _nftToLoanIds[loan.nftAsset][loan.nftTokenId] = 0;
    // 检查用户贷款nft数量
    require(_userNftCollateral[loan.borrower][loan.nftAsset] >= 1, Errors.LP_INVALIED_USER_NFT_AMOUNT);
    _userNftCollateral[loan.borrower][loan.nftAsset] -= 1;
    // 检查该系列nft存在的债务数量
    require(_nftTotalCollateral[loan.nftAsset] >= 1, Errors.LP_INVALIED_NFT_AMOUNT);
    _nftTotalCollateral[loan.nftAsset] -= 1;

    // burn bNFT and transfer underlying NFT asset to user
    // 销毁bnft
    IBNFT(bNftAddress).burn(loan.nftTokenId);
    // 把被清算的nft转给清算人
    IERC721Upgradeable(loan.nftAsset).safeTransferFrom(address(this), _msgSender(), loan.nftTokenId);
    // 清算事件
    emit LoanLiquidated(
      initiator,
      loanId,
      loan.nftAsset,
      loan.nftTokenId,
      loan.reserveAsset,
      borrowAmount,
      borrowIndex
    );

    _handleAfterLoanRepaid(loan.nftAsset, loan.nftTokenId);
  }
```