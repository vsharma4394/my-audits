# [H-01]In execute repay function updation of interests will be incorrect.

## Summary
In execute repay function before updating the value of debt shares to correct value interest rates are updating leading to incorrect calcualtions of the reward.
## Vulnerability Detail
Following is execute repay function
```solidity
function executeRepay(
    DataTypes.ReserveData storage reserve,
    DataTypes.PositionBalance storage balances,
    DataTypes.ReserveSupplies storage totalSupplies,
    DataTypes.UserConfigurationMap storage userConfig,
    DataTypes.ExecuteRepayParams memory params
  ) external returns (DataTypes.SharesType memory payback) {
    DataTypes.ReserveCache memory cache = reserve.cache(totalSupplies);
    reserve.updateState(params.reserveFactor, cache);
    payback.assets = balances.getDebtBalance(cache.nextBorrowIndex);

    // Allows a user to max repay without leaving dust from interest.
    if (params.amount == type(uint256).max) {
      params.amount = payback.assets;
    }

    ValidationLogic.validateRepay(params.amount, payback.assets);

    // If paybackAmount is more than what the user wants to payback, the set it to the
    // user input (ie params.amount)
    if (params.amount < payback.assets) payback.assets = params.amount;

    reserve.updateInterestRates(
      totalSupplies,
      cache,
      params.asset,
      IPool(params.pool).getReserveFactor(),
      payback.assets,
      0,
      params.position,
      params.data.interestRateData
    );

    // update balances and total supplies
    payback.shares = balances.repayDebt(totalSupplies, payback.assets, cache.nextBorrowIndex);
    cache.nextDebtShares = totalSupplies.debtShares;

    if (balances.getDebtBalance(cache.nextBorrowIndex) == 0) {
      userConfig.setBorrowing(reserve.id, false);
    }

    IERC20(params.asset).safeTransferFrom(msg.sender, address(this), payback.assets);
    emit PoolEventsLib.Repay(params.asset, params.position, msg.sender, payback.assets);
  }
}
```
As can be seen rom above that first update interest call is made and after that  cache.nextDebtShares = totalSupplies.debtShares; value is set to correct value. As repay debt function is called after updating the interest rates the new debt shares decrease .
So the new debt shares would be less than the previous shares but as update interest rates function call is made before hand this leads to incorrect updation of interest shares. 
updateInterestRates function is as follows
```solidity
function updateInterestRates(
    DataTypes.ReserveData storage _reserve,
    DataTypes.ReserveSupplies storage totalSupplies,
    DataTypes.ReserveCache memory _cache,
    address _reserveAddress,
    uint256 _reserveFactor,
    uint256 _liquidityAdded,
    uint256 _liquidityTaken,
    bytes32 _position,
    bytes memory _data
  ) internal {
    UpdateInterestRatesLocalVars memory vars;

    vars.totalDebt = _cache.nextDebtShares.rayMul(_cache.nextBorrowIndex);

    (vars.nextLiquidityRate, vars.nextBorrowRate) = IReserveInterestRateStrategy(_reserve.interestRateStrategyAddress)
      .calculateInterestRates(
      _position,
      _data,
      DataTypes.CalculateInterestRatesParams({
        liquidityAdded: _liquidityAdded,
        liquidityTaken: _liquidityTaken,
        totalDebt: vars.totalDebt,
        reserveFactor: _reserveFactor,
        reserve: _reserveAddress
      })
    );

    _reserve.liquidityRate = vars.nextLiquidityRate.toUint128();
    _reserve.borrowRate = vars.nextBorrowRate.toUint128();

    if (_liquidityAdded > 0) totalSupplies.underlyingBalance += _liquidityAdded.toUint128();
    else if (_liquidityTaken > 0) totalSupplies.underlyingBalance -= _liquidityTaken.toUint128();

    emit PoolEventsLib.ReserveDataUpdated(
      _reserveAddress, vars.nextLiquidityRate, vars.nextBorrowRate, _cache.nextLiquidityIndex, _cache.nextBorrowIndex
    );
  }
```
As can be seen that the next debt shares are used above and loaded to var.totalDebt so incorrect value will be used.

It support my argument following is execute borrow function 
```solidity
 function executeBorrow(
    mapping(address => DataTypes.ReserveData) storage reservesData,
    mapping(uint256 => address) storage reservesList,
    DataTypes.UserConfigurationMap storage userConfig,
    mapping(address => mapping(bytes32 => DataTypes.PositionBalance)) storage _balances,
    DataTypes.ReserveSupplies storage totalSupplies,
    DataTypes.ExecuteBorrowParams memory params
  ) public returns (DataTypes.SharesType memory borrowed) {
    DataTypes.ReserveData storage reserve = reservesData[params.asset];
    DataTypes.ReserveCache memory cache = reserve.cache(totalSupplies);

    reserve.updateState(params.reserveFactor, cache);

    ValidationLogic.validateBorrow(
      _balances,
      reservesData,
      reservesList,
      DataTypes.ValidateBorrowParams({
        cache: cache,
        userConfig: userConfig,
        asset: params.asset,
        position: params.position,
        amount: params.amount,
        reservesCount: params.reservesCount,
        pool: params.pool
      })
    );

    // mint debt tokens
    DataTypes.PositionBalance storage b = _balances[params.asset][params.position];
    bool isFirstBorrowing;
    (isFirstBorrowing, borrowed.shares) = b.borrowDebt(totalSupplies, params.amount, cache.nextBorrowIndex);
    cache.nextDebtShares = totalSupplies.debtShares;

    // if first borrowing, flag that
    if (isFirstBorrowing) userConfig.setBorrowing(reserve.id, true);

    reserve.updateInterestRates(
      totalSupplies,
      cache,
      params.asset,
      IPool(params.pool).getReserveFactor(),
      0,
      params.amount,
      params.position,
      params.data.interestRateData
    );

    IERC20(params.asset).safeTransfer(params.destination, params.amount);

    emit PoolEventsLib.Borrow(params.asset, params.user, params.position, params.amount, reserve.borrowRate);

    borrowed.assets = params.amount;
  }
```
As can be see that first next debt shares are updated and then interest rates are updated so this shows inconsistency in the logic therefore an issue.

## Impact
Incorrect updation of the interest rates.
## Code Snippet
https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/BorrowLogic.sol#L141
https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/BorrowLogic.sol#L152
## Tool used

Manual Review

## Recommendation
call update interest rates after updating the debt shares as follows
```solidity
function executeRepay(
    DataTypes.ReserveData storage reserve,
    DataTypes.PositionBalance storage balances,
    DataTypes.ReserveSupplies storage totalSupplies,
    DataTypes.UserConfigurationMap storage userConfig,
    DataTypes.ExecuteRepayParams memory params
  ) external returns (DataTypes.SharesType memory payback) {
    DataTypes.ReserveCache memory cache = reserve.cache(totalSupplies);
    reserve.updateState(params.reserveFactor, cache);
    payback.assets = balances.getDebtBalance(cache.nextBorrowIndex);

    // Allows a user to max repay without leaving dust from interest.
    if (params.amount == type(uint256).max) {
      params.amount = payback.assets;
    }

    ValidationLogic.validateRepay(params.amount, payback.assets);

    // If paybackAmount is more than what the user wants to payback, the set it to the
    // user input (ie params.amount)
    if (params.amount < payback.assets) payback.assets = params.amount;
 // update balances and total supplies
    payback.shares = balances.repayDebt(totalSupplies, payback.assets, cache.nextBorrowIndex);
    cache.nextDebtShares = totalSupplies.debtShares;

    reserve.updateInterestRates(
      totalSupplies,
      cache,
      params.asset,
      IPool(params.pool).getReserveFactor(),
      payback.assets,
      0,
      params.position,
      params.data.interestRateData
    );



    if (balances.getDebtBalance(cache.nextBorrowIndex) == 0) {
      userConfig.setBorrowing(reserve.id, false);
    }

    IERC20(params.asset).safeTransferFrom(msg.sender, address(this), payback.assets);
    emit PoolEventsLib.Repay(params.asset, params.position, msg.sender, payback.assets);
  }
```

# [H-02]reserves state  of pool in which the vault has position is not updated before accruing fees shares.

## Summary
Reserves state  of pool in which the vault has position is not updated before accruing fees shares which leads to old liquidity index being used thus not accruing fee shares.
## Vulnerability Detail
_accrueFees function is called multiple times in the contract
```solidity
 function _accrueFee() internal returns (uint256 newTotalAssets) {
    uint256 feeShares;
    (feeShares, newTotalAssets) = _accruedFeeShares();
    if (feeShares != 0) _mint(feeRecipient, feeShares);
    emit CuratedEventsLib.AccrueInterest(newTotalAssets, feeShares);
  }
```
_accruedFeeShares function is as follows
```solidity
function _accruedFeeShares() internal view returns (uint256 feeShares, uint256 newTotalAssets) {
    newTotalAssets = totalAssets();

    uint256 totalInterest = newTotalAssets.zeroFloorSub(lastTotalAssets);
    if (totalInterest != 0 && fee != 0) {
      // It is acknowledged that `feeAssets` may be rounded down to 0 if `totalInterest * fee < WAD`.
      uint256 feeAssets = totalInterest.mulDiv(fee, 1e18);
      // The fee assets is subtracted from the total assets in this calculation to compensate for the fact
      // that total assets is already increased by the total interest (including the fee assets).
      feeShares = _convertToSharesWithTotals(feeAssets, totalSupply(), newTotalAssets - feeAssets, MathUpgradeable.Rounding.Down);
    }
  }
```
totalAssets() function is as follows
```solidity
function totalAssets() public view override returns (uint256 assets) {
    for (uint256 i; i < withdrawQueue.length; ++i) {
      assets += withdrawQueue[i].getBalanceByPosition(asset(), positionId);
    }
  }
```
Now as can be seen that the total assets are calculated as above and getBalance by position function is as follows
```solidity 
function getBalanceByPosition(address asset, bytes32 positionId) external view returns (uint256 balance) {
    return _balances[asset][positionId].getSupplyBalance(_reserves[asset].liquidityIndex);
  }
```
So if the liquidity index of a reserve is not updated before accruing fees then fees won't be accrued.
That is the main issue because in several instance like some are as follows
```solidity
function setFee(uint256 newFee) external onlyOwner {
    if (newFee == fee) revert CuratedErrorsLib.AlreadySet();
    if (newFee > MAX_FEE) revert CuratedErrorsLib.MaxFeeExceeded();
    if (newFee != 0 && feeRecipient == address(0)) revert CuratedErrorsLib.ZeroFeeRecipient();

    // Accrue fee using the previous fee set before changing it.
    _updateLastTotalAssets(_accrueFee());

    // Safe "unchecked" cast because newFee <= MAX_FEE.
    fee = uint96(newFee);

    emit CuratedEventsLib.SetFee(_msgSender(), fee);
  }

  /// @inheritdoc ICuratedVaultBase
  function setFeeRecipient(address newFeeRecipient) external onlyOwner {
    if (newFeeRecipient == feeRecipient) revert CuratedErrorsLib.AlreadySet();
    if (newFeeRecipient == address(0) && fee != 0) revert CuratedErrorsLib.ZeroFeeRecipient();

    // Accrue fee to the previous fee recipient set before changing it.
    _updateLastTotalAssets(_accrueFee());

    feeRecipient = newFeeRecipient;

    emit CuratedEventsLib.SetFeeRecipient(newFeeRecipient);
  }
```
as seen above that no where in _accrue fee function reserve state is updated to latest liquidity index so fees wouldn't be accrued at all.

## Impact
fees is not accrued.
## Code Snippet
https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/vaults/CuratedVaultSetters.sol#L178
## Tool used

Manual Review

## Recommendation
Update the reserve state before performing any calculation regarding fees accruel
One way can be by doing so in totalAssets function
```solidity
function totalAssets() public view override returns (uint256 assets) {
    for (uint256 i; i < withdrawQueue.length; ++i) {
      withdrawQueue[i].forceUpdateReserve(asset());
      assets += withdrawQueue[i].getBalanceByPosition(asset(), positionId);
    }
  }
```


# [H-03]executeMintToTreasury incorrectly deducts the treasury shares from totalSupply reserve

## Summary
ExecuteMintToTreasury incorrectly deducts the treasury shares from totalSupply reserve because those shares were never added to the  totalSupply mapping for a reserve. 
## Vulnerability Detail
Whenever assets are deposited or withdrawn from a particular reserve of a pool firstly  updateState is called which is as follows 
```solidity
 function updateState(DataTypes.ReserveData storage self, uint256 _reserveFactor, DataTypes.ReserveCache memory _cache) internal {
    // If time didn't pass since last stored timestamp, skip state update
    if (self.lastUpdateTimestamp == uint40(block.timestamp)) return;

    _updateIndexes(self, _cache);
    _accrueToTreasury(_reserveFactor, self, _cache);

    self.lastUpdateTimestamp = uint40(block.timestamp);
  }
```
Then accrueToTreasury function is called which is as follows
```solidity
function _accrueToTreasury(uint256 reserveFactor, DataTypes.ReserveData storage _reserve, DataTypes.ReserveCache memory _cache) internal {
    if (reserveFactor == 0) return;
    AccrueToTreasuryLocalVars memory vars;

    // calculate the total variable debt at moment of the last interaction
    vars.prevtotalDebt = _cache.currDebtShares.rayMul(_cache.currBorrowIndex);

    // calculate the new total variable debt after accumulation of the interest on the index
    vars.currtotalDebt = _cache.currDebtShares.rayMul(_cache.nextBorrowIndex);

    // debt accrued is the sum of the current debt minus the sum of the debt at the last update
    vars.totalDebtAccrued = vars.currtotalDebt - vars.prevtotalDebt;

    vars.amountToMint = vars.totalDebtAccrued.percentMul(reserveFactor);

    if (vars.amountToMint != 0) _reserve.accruedToTreasuryShares += vars.amountToMint.rayDiv(_cache.nextLiquidityIndex).toUint128();
  }
```
As can be seen from above only accrued to treasury shares are increased for a reserve these shares are not added to the totalSupply shares of a reserve.
Now issue is when withdraw function is called for a particular reserve by any position holder then following function is called 
```solidity
function _withdraw(
    address asset,
    address to,
    uint256 amount,
    bytes32 pos,
    DataTypes.ExtraData memory data
  ) internal nonReentrant(RentrancyKind.LENDING) returns (DataTypes.SharesType memory res) {
    require(to != address(0), PoolErrorsLib.ZERO_ADDRESS_NOT_VALID);

    DataTypes.ExecuteWithdrawParams memory params = DataTypes.ExecuteWithdrawParams({
      reserveFactor: _factory.reserveFactor(),
      asset: asset,
      amount: amount,
      position: pos,
      destination: to,
      data: data,
      pool: address(this)
    });

    if (address(_hook) != address(0)) _hook.beforeWithdraw(params);

    res = SupplyLogic.executeWithdraw(_reserves, _reservesList, _usersConfig[pos], _balances, _totalSupplies[asset], params);
    PoolLogic.executeMintToTreasury(_totalSupplies[asset], _reserves, _factory.treasury(), asset);

    if (address(_hook) != address(0)) _hook.afterWithdraw(params);
  }
```
As can be seen that at the end  execute mint to treasury is called which is as follows
```solidity
 function executeMintToTreasury(
    DataTypes.ReserveSupplies storage totalSupply,
    mapping(address => DataTypes.ReserveData) storage reservesData,
    address treasury,
    address asset
  ) external {
    DataTypes.ReserveData storage reserve = reservesData[asset];

    uint256 accruedToTreasuryShares = reserve.accruedToTreasuryShares;

    if (accruedToTreasuryShares != 0) {
      reserve.accruedToTreasuryShares = 0;
      uint256 normalizedIncome = reserve.getNormalizedIncome();
      uint256 amountToMint = accruedToTreasuryShares.rayMul(normalizedIncome);

      IERC20(asset).safeTransfer(treasury, amountToMint);
      totalSupply.supplyShares -= accruedToTreasuryShares;

      emit PoolEventsLib.MintedToTreasury(asset, amountToMint);
    }
  }
```
Issue is in the following line
```solidity
 totalSupply.supplyShares -= accruedToTreasuryShares;
```
This is a issue because shares accrued to treasury were never added to the totalSupply of a reserve when updateState function was called so no need of deducting them from totalSupply.supplyshares. This will cause incorrect accounting of the supply shares and will lead to underflow.
## Impact
Incorrect Accounting of supply shares of a reserve and will lead to underflow ultimately.
## Code Snippet
https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/PoolLogic.sol#L99
## Tool used

Manual Review

## Recommendation
Exclude the following line 
```solidity
 totalSupply.supplyShares -= accruedToTreasuryShares;
```



# [H-04]Incorrect value of debt is accessed in executeLiquidationcall function

## Summary
Incorrect value of debt is accessed in executeLiquidationcall function because debt shares are accesed instead of debt accrued in terms of actual assets.
## Vulnerability Detail
Following is executeLiquidation call function 
```solidity
function executeLiquidationCall(
    mapping(address => DataTypes.ReserveData) storage reservesData,
    mapping(uint256 => address) storage reservesList,
    mapping(address => mapping(bytes32 => DataTypes.PositionBalance)) storage balances,
    mapping(address => DataTypes.ReserveSupplies) storage totalSupplies,
    mapping(bytes32 => DataTypes.UserConfigurationMap) storage usersConfig,
    DataTypes.ExecuteLiquidationCallParams memory params
  ) external {
    LiquidationCallLocalVars memory vars;

    DataTypes.ReserveData storage collateralReserve = reservesData[params.collateralAsset];
    DataTypes.ReserveData storage debtReserve = reservesData[params.debtAsset];
    DataTypes.UserConfigurationMap storage userConfig = usersConfig[params.position];
    vars.debtReserveCache = debtReserve.cache(totalSupplies[params.debtAsset]);
    debtReserve.updateState(params.reserveFactor, vars.debtReserveCache);

    (,,,, vars.healthFactor,) = GenericLogic.calculateUserAccountData(
      balances,
      reservesData,
      reservesList,
      DataTypes.CalculateUserAccountDataParams({userConfig: userConfig, position: params.position, pool: params.pool})
    );

    (vars.userDebt, vars.actualDebtToLiquidate) = _calculateDebt(
      // vars.debtReserveCache,
      params,
      vars.healthFactor,
      balances
    );

    ValidationLogic.validateLiquidationCall(
      userConfig,
      collateralReserve,
      DataTypes.ValidateLiquidationCallParams({
        debtReserveCache: vars.debtReserveCache,
        totalDebt: vars.userDebt,
        healthFactor: vars.healthFactor
      })
    );

    (vars.collateralPriceSource, vars.debtPriceSource, vars.liquidationBonus) = _getConfigurationData(collateralReserve, params);

    vars.userCollateralBalance = balances[params.collateralAsset][params.position].supplyShares;

    (vars.actualCollateralToLiquidate, vars.actualDebtToLiquidate, vars.liquidationProtocolFeeAmount) =
    _calculateAvailableCollateralToLiquidate(
      collateralReserve,
      vars.debtReserveCache,
      vars.actualDebtToLiquidate,
      vars.userCollateralBalance,
      vars.liquidationBonus,
      IPool(params.pool).getAssetPrice(params.collateralAsset),
      IPool(params.pool).getAssetPrice(params.debtAsset),
      IPool(params.pool).factory().liquidationProtocolFeePercentage()
    );

    if (vars.userDebt == vars.actualDebtToLiquidate) {
      userConfig.setBorrowing(debtReserve.id, false);
    }

    // If the collateral being liquidated is equal to the user balance,
    // we set the currency as not being used as collateral anymore
    if (vars.actualCollateralToLiquidate + vars.liquidationProtocolFeeAmount == vars.userCollateralBalance) {
      userConfig.setUsingAsCollateral(collateralReserve.id, false);
      emit PoolEventsLib.ReserveUsedAsCollateralDisabled(params.collateralAsset, params.position);
    }

    _repayDebtTokens(params, vars, balances[params.debtAsset], totalSupplies[params.debtAsset]);

    debtReserve.updateInterestRates(
      totalSupplies[params.debtAsset],
      vars.debtReserveCache,
      params.debtAsset,
      IPool(params.pool).getReserveFactor(),
      vars.actualDebtToLiquidate,
      0,
      '',
      ''
    );

    _burnCollateralTokens(
      collateralReserve, params, vars, balances[params.collateralAsset][params.position], totalSupplies[params.collateralAsset]
    );

    // Transfer fee to treasury if it is non-zero
    if (vars.liquidationProtocolFeeAmount != 0) {
      uint256 liquidityIndex = collateralReserve.getNormalizedIncome();
      uint256 scaledDownLiquidationProtocolFee = vars.liquidationProtocolFeeAmount.rayDiv(liquidityIndex);
      uint256 scaledDownUserBalance = balances[params.collateralAsset][params.position].supplyShares;

      if (scaledDownLiquidationProtocolFee > scaledDownUserBalance) {
        vars.liquidationProtocolFeeAmount = scaledDownUserBalance.rayMul(liquidityIndex);
      }

      IERC20(params.collateralAsset).safeTransfer(IPool(params.pool).factory().treasury(), vars.liquidationProtocolFeeAmount);
    }

    // Transfers the debt asset being repaid to the aToken, where the liquidity is kept
    IERC20(params.debtAsset).safeTransferFrom(msg.sender, address(params.pool), vars.actualDebtToLiquidate);

    emit PoolEventsLib.LiquidationCall(
      params.collateralAsset, params.debtAsset, params.position, vars.actualDebtToLiquidate, vars.actualCollateralToLiquidate, msg.sender
    );
  }
```
Issue is in the following line 
```solidity
(vars.userDebt, vars.actualDebtToLiquidate) = _calculateDebt(
      // vars.debtReserveCache,
      params,
      vars.healthFactor,
      balances
    );
```
_calculateDebt function is as follows
```solidity
function _calculateDebt(
    DataTypes.ExecuteLiquidationCallParams memory params,
    uint256 healthFactor,
    mapping(address => mapping(bytes32 => DataTypes.PositionBalance)) storage balances
  ) internal view returns (uint256, uint256) {
    uint256 userDebt = balances[params.debtAsset][params.position].debtShares;

    uint256 closeFactor = healthFactor > CLOSE_FACTOR_HF_THRESHOLD ? DEFAULT_LIQUIDATION_CLOSE_FACTOR : MAX_LIQUIDATION_CLOSE_FACTOR;

    uint256 maxLiquidatableDebt = userDebt.percentMul(closeFactor);

    uint256 actualDebtToLiquidate = params.debtToCover > maxLiquidatableDebt ? maxLiquidatableDebt : params.debtToCover;

    return (userDebt, actualDebtToLiquidate);
  }
```
As can be seen from above that userDebt = debt shares and not what it has accrued till now(i.e debt balance of this position) based on the borrow index.
So incorrect calculation of user debt is done. Further calculation involved in the function would be also wrong due to this.
Plus when _repayDebtTokens(params, vars, balances[params.debtAsset], totalSupplies[params.debtAsset]);
 call happens then following is executed
```solidity
function _repayDebtTokens(
    DataTypes.ExecuteLiquidationCallParams memory params,
    LiquidationCallLocalVars memory vars,
    mapping(bytes32 => DataTypes.PositionBalance) storage balances,
    DataTypes.ReserveSupplies storage totalSupplies
  ) internal {
    uint256 burnt = balances[params.position].repayDebt(totalSupplies, vars.actualDebtToLiquidate, vars.debtReserveCache.nextBorrowIndex);
    vars.debtReserveCache.nextDebtShares = burnt;
  }
```
And due to wrong calculation of user debt vars.actualDebtToLiquidate value will be in shares so in repay debt function it will again divide this by next borrow index assuming that vars.actualDebtToLiquidate is in assets so wrong updation

## Impact
Incorrect logic implemented thus causing whole function to behave way differently.
## Code Snippet
https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/LiquidationLogic.sol#L117
## Tool used

Manual Review

## Recommendation
Access user debt by taking in account the accrued debt and not the debt shares by using next borrow index.



# [H-05]Interest rates are updated wrongly due to incorrect debt shares used.

## Summary
When liquidation call is made in liquidation logic library wrong value of next debt shares are set which ultimately leads to incorrect updation of interest rates.
## Vulnerability Detail
Following is execute liquidation call function 
```solidity
function executeLiquidationCall(
    mapping(address => DataTypes.ReserveData) storage reservesData,
    mapping(uint256 => address) storage reservesList,
    mapping(address => mapping(bytes32 => DataTypes.PositionBalance)) storage balances,
    mapping(address => DataTypes.ReserveSupplies) storage totalSupplies,
    mapping(bytes32 => DataTypes.UserConfigurationMap) storage usersConfig,
    DataTypes.ExecuteLiquidationCallParams memory params
  ) external {
    LiquidationCallLocalVars memory vars;

    DataTypes.ReserveData storage collateralReserve = reservesData[params.collateralAsset];
    DataTypes.ReserveData storage debtReserve = reservesData[params.debtAsset];
    DataTypes.UserConfigurationMap storage userConfig = usersConfig[params.position];
    vars.debtReserveCache = debtReserve.cache(totalSupplies[params.debtAsset]);
    debtReserve.updateState(params.reserveFactor, vars.debtReserveCache);

    (,,,, vars.healthFactor,) = GenericLogic.calculateUserAccountData(
      balances,
      reservesData,
      reservesList,
      DataTypes.CalculateUserAccountDataParams({userConfig: userConfig, position: params.position, pool: params.pool})
    );

    (vars.userDebt, vars.actualDebtToLiquidate) = _calculateDebt(
      // vars.debtReserveCache,
      params,
      vars.healthFactor,
      balances
    );

    ValidationLogic.validateLiquidationCall(
      userConfig,
      collateralReserve,
      DataTypes.ValidateLiquidationCallParams({
        debtReserveCache: vars.debtReserveCache,
        totalDebt: vars.userDebt,
        healthFactor: vars.healthFactor
      })
    );

    (vars.collateralPriceSource, vars.debtPriceSource, vars.liquidationBonus) = _getConfigurationData(collateralReserve, params);

    vars.userCollateralBalance = balances[params.collateralAsset][params.position].supplyShares;

    (vars.actualCollateralToLiquidate, vars.actualDebtToLiquidate, vars.liquidationProtocolFeeAmount) =
    _calculateAvailableCollateralToLiquidate(
      collateralReserve,
      vars.debtReserveCache,
      vars.actualDebtToLiquidate,
      vars.userCollateralBalance,
      vars.liquidationBonus,
      IPool(params.pool).getAssetPrice(params.collateralAsset),
      IPool(params.pool).getAssetPrice(params.debtAsset),
      IPool(params.pool).factory().liquidationProtocolFeePercentage()
    );

    if (vars.userDebt == vars.actualDebtToLiquidate) {
      userConfig.setBorrowing(debtReserve.id, false);
    }

    // If the collateral being liquidated is equal to the user balance,
    // we set the currency as not being used as collateral anymore
    if (vars.actualCollateralToLiquidate + vars.liquidationProtocolFeeAmount == vars.userCollateralBalance) {
      userConfig.setUsingAsCollateral(collateralReserve.id, false);
      emit PoolEventsLib.ReserveUsedAsCollateralDisabled(params.collateralAsset, params.position);
    }

    _repayDebtTokens(params, vars, balances[params.debtAsset], totalSupplies[params.debtAsset]);

    debtReserve.updateInterestRates(
      totalSupplies[params.debtAsset],
      vars.debtReserveCache,
      params.debtAsset,
      IPool(params.pool).getReserveFactor(),
      vars.actualDebtToLiquidate,
      0,
      '',
      ''
    );

    _burnCollateralTokens(
      collateralReserve, params, vars, balances[params.collateralAsset][params.position], totalSupplies[params.collateralAsset]
    );

    // Transfer fee to treasury if it is non-zero
    if (vars.liquidationProtocolFeeAmount != 0) {
      uint256 liquidityIndex = collateralReserve.getNormalizedIncome();
      uint256 scaledDownLiquidationProtocolFee = vars.liquidationProtocolFeeAmount.rayDiv(liquidityIndex);
      uint256 scaledDownUserBalance = balances[params.collateralAsset][params.position].supplyShares;

      if (scaledDownLiquidationProtocolFee > scaledDownUserBalance) {
        vars.liquidationProtocolFeeAmount = scaledDownUserBalance.rayMul(liquidityIndex);
      }

      IERC20(params.collateralAsset).safeTransfer(IPool(params.pool).factory().treasury(), vars.liquidationProtocolFeeAmount);
    }

    // Transfers the debt asset being repaid to the aToken, where the liquidity is kept
    IERC20(params.debtAsset).safeTransferFrom(msg.sender, address(params.pool), vars.actualDebtToLiquidate);

    emit PoolEventsLib.LiquidationCall(
      params.collateralAsset, params.debtAsset, params.position, vars.actualDebtToLiquidate, vars.actualCollateralToLiquidate, msg.sender
    );
  }
```
Key lines where issue are the following 
```solidity
repayDebtTokens(params, vars, balances[params.debtAsset], totalSupplies[params.debtAsset]);

    debtReserve.updateInterestRates(
      totalSupplies[params.debtAsset],
      vars.debtReserveCache,
      params.debtAsset,
      IPool(params.pool).getReserveFactor(),
      vars.actualDebtToLiquidate,
      0,
      '',
      ''
    );
```
lets see repaydebt tokens function which is as follows 
```solidity
 function _repayDebtTokens(
    DataTypes.ExecuteLiquidationCallParams memory params,
    LiquidationCallLocalVars memory vars,
    mapping(bytes32 => DataTypes.PositionBalance) storage balances,
    DataTypes.ReserveSupplies storage totalSupplies
  ) internal {
    uint256 burnt = balances[params.position].repayDebt(totalSupplies, vars.actualDebtToLiquidate, vars.debtReserveCache.nextBorrowIndex);
    vars.debtReserveCache.nextDebtShares = burnt;
  }
```
Its sets next debt shares wrongly to burnt shares as can be seen in repay debt function 
```solidity
function repayDebt(
    DataTypes.PositionBalance storage self,
    DataTypes.ReserveSupplies storage supply,
    uint256 amount,
    uint128 index
  ) internal returns (uint256 sharesBurnt) {
    sharesBurnt = amount.rayDiv(index);
    require(sharesBurnt != 0, PoolErrorsLib.INVALID_BURN_AMOUNT);
    self.lastDebtLiquidtyIndex = index;
    self.debtShares -= sharesBurnt;
    supply.debtShares -= sharesBurnt;
  }
```
As can be seen that function returns shares burnt not total debt shares remaining for that particular reserve supply.
So  vars.debtReserveCache.nextDebtShares = burnt; sets wrong value of next debt shares.
After that interest rates are updated using following function call
```solidity
 debtReserve.updateInterestRates(
      totalSupplies[params.debtAsset],
      vars.debtReserveCache,
      params.debtAsset,
      IPool(params.pool).getReserveFactor(),
      vars.actualDebtToLiquidate,
      0,
      '',
      ''
    );
```
update interest rates function is as follows
```solidity
function updateInterestRates(
    DataTypes.ReserveData storage _reserve,
    DataTypes.ReserveSupplies storage totalSupplies,
    DataTypes.ReserveCache memory _cache,
    address _reserveAddress,
    uint256 _reserveFactor,
    uint256 _liquidityAdded,
    uint256 _liquidityTaken,
    bytes32 _position,
    bytes memory _data
  ) internal {
    UpdateInterestRatesLocalVars memory vars;

    vars.totalDebt = _cache.nextDebtShares.rayMul(_cache.nextBorrowIndex);

    (vars.nextLiquidityRate, vars.nextBorrowRate) = IReserveInterestRateStrategy(_reserve.interestRateStrategyAddress)
      .calculateInterestRates(
      _position,
      _data,
      DataTypes.CalculateInterestRatesParams({
        liquidityAdded: _liquidityAdded,
        liquidityTaken: _liquidityTaken,
        totalDebt: vars.totalDebt,
        reserveFactor: _reserveFactor,
        reserve: _reserveAddress
      })
    );

    _reserve.liquidityRate = vars.nextLiquidityRate.toUint128();
    _reserve.borrowRate = vars.nextBorrowRate.toUint128();

    if (_liquidityAdded > 0) totalSupplies.underlyingBalance += _liquidityAdded.toUint128();
    else if (_liquidityTaken > 0) totalSupplies.underlyingBalance -= _liquidityTaken.toUint128();

    emit PoolEventsLib.ReserveDataUpdated(
      _reserveAddress, vars.nextLiquidityRate, vars.nextBorrowRate, _cache.nextLiquidityIndex, _cache.nextBorrowIndex
    );
  }
```
As can be seen from above vars.totalDebt = _cache.nextDebtShares.rayMul(_cache.nextBorrowIndex); it uses next debt shares which were wrongly set and further vars.totalDebt is used in totalDebt parameter which is used to calculate next liquidity rate and borrow rate. Thus leading to incorrect updation of interest rates.

## Impact
Incorrect updation of interest rates.
## Code Snippet
https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/LiquidationLogic.sol#L246
## Tool used

Manual Review

## Recommendation
Set the value as follows
```solidity
   function _repayDebtTokens(
    DataTypes.ExecuteLiquidationCallParams memory params,
    LiquidationCallLocalVars memory vars,
    mapping(bytes32 => DataTypes.PositionBalance) storage balances,
    DataTypes.ReserveSupplies storage totalSupplies
  ) internal {
    uint256 burnt = balances[params.position].repayDebt(totalSupplies, vars.actualDebtToLiquidate, vars.debtReserveCache.nextBorrowIndex);
    vars.debtReserveCache.nextDebtShares = totalSupplies.debtShares;
  }

```


# [H-06]getSupplyBalance returns wrong amount of assets

## Summary
getSupplyBalance returns wrong amount of assets because it doesn't converts previous shares to assets.
## Vulnerability Detail
Following is getSupplyBalance function 
```solidity
function getSupplyBalance(DataTypes.PositionBalance storage self, uint256 index) public view returns (uint256 supply) {
    uint256 increase = self.supplyShares.rayMul(index) - self.supplyShares.rayMul(self.lastSupplyLiquidtyIndex);
    return self.supplyShares + increase;
  }
```
Following are comments of this function
```solidity
/**
   * @notice Get the rebased assets worth of collateral the position has
   * @dev Converts `shares` into `amount` and returns the rebased value
   * @param self The position to fetch the value for
   * @param index The current liquidity index
   */
```
It is clearly mentioned that the function should return rebased assets but as we can see that it returns self.supplyShares + increase 
we can see that the increase represents the increased amount of rebased assets which is correct but self.supplyShares are not converted to assets.
For example lets assume last supply index was 20 and 10 assets were supplied so shares would be 10/20. After some time the liquidity index(index in above function) becomes 40 so the above function will return the balance as 10/20 +10 assets  whereas 10/20 shares are worth 20 assets. So incorrect value returned.
## Impact
This is a critical function used for calculating the collateral added so as to borrow some other assets. Not only this if the user calls withdraw then its supply balance is checked which will represent incorrect balance thus causing loss to the position holder.
## Code Snippet
https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/configuration/PositionBalanceConfiguration.sol#L128
## Tool used

Manual Review

## Recommendation
The code should be as follows
```solidity
function getSupplyBalance(DataTypes.PositionBalance storage self, uint256 index) public view returns (uint256 supply) {
    uint256 increase = self.supplyShares.rayMul(index) - self.supplyShares.rayMul(self.lastSupplyLiquidtyIndex);
    return self.supplyShares.rayMul(self.lastSupplyLiquidtyIndex)+ increase;
  }
```



# [M-01]assets are not withdrawn fully if allocation to that reserve is zero in reallocate function

## Summary
assets are not withdrawn fully if allocation to that reserve is zero in reallocate function due to wrong logic in certain if condition
## Vulnerability Detail
Following is reallocate function 
```solidity
function reallocate(MarketAllocation[] calldata allocations) external onlyAllocator {
    uint256 totalSupplied;
    uint256 totalWithdrawn;

    for (uint256 i; i < allocations.length; ++i) {
      MarketAllocation memory allocation = allocations[i];
      IPool pool = allocation.market;

      (uint256 supplyAssets, uint256 supplyShares) = _accruedSupplyBalance(pool);
      uint256 toWithdraw = supplyAssets.zeroFloorSub(allocation.assets);

      if (toWithdraw > 0) {
        if (!config[pool].enabled) revert CuratedErrorsLib.MarketNotEnabled(pool);

        // Guarantees that unknown frontrunning donations can be withdrawn, in order to disable a market.
        uint256 shares;
        if (allocation.assets == 0) {
          shares = supplyShares;
          toWithdraw = 0;
        }

        DataTypes.SharesType memory burnt = pool.withdrawSimple(asset(), address(this), toWithdraw, 0);
        emit CuratedEventsLib.ReallocateWithdraw(_msgSender(), pool, burnt.assets, burnt.shares);
        totalWithdrawn += burnt.assets;
      } else {
        uint256 suppliedAssets =
          allocation.assets == type(uint256).max ? totalWithdrawn.zeroFloorSub(totalSupplied) : allocation.assets.zeroFloorSub(supplyAssets);

        if (suppliedAssets == 0) continue;

        uint256 supplyCap = config[pool].cap;
        if (supplyCap == 0) revert CuratedErrorsLib.UnauthorizedMarket(pool);

        if (supplyAssets + suppliedAssets > supplyCap) revert CuratedErrorsLib.SupplyCapExceeded(pool);

        // The market's loan asset is guaranteed to be the vault's asset because it has a non-zero supply cap.
        IERC20(asset()).forceApprove(address(pool), type(uint256).max);
        DataTypes.SharesType memory minted = pool.supplySimple(asset(), address(this), suppliedAssets, 0);
        emit CuratedEventsLib.ReallocateSupply(_msgSender(), pool, minted.assets, minted.shares);
        totalSupplied += suppliedAssets;
      }
    }

    if (totalWithdrawn != totalSupplied) revert CuratedErrorsLib.InconsistentReallocation();
  }
```
Now issue lies in the following lines 
```solidity
(uint256 supplyAssets, uint256 supplyShares) = _accruedSupplyBalance(pool);
      uint256 toWithdraw = supplyAssets.zeroFloorSub(allocation.assets);

      if (toWithdraw > 0) {
        if (!config[pool].enabled) revert CuratedErrorsLib.MarketNotEnabled(pool);

        // Guarantees that unknown frontrunning donations can be withdrawn, in order to disable a market.
        uint256 shares;
        if (allocation.assets == 0) {
          shares = supplyShares;
          toWithdraw = 0;
        }
```
If allocation.assets = 0 then all of the supplyAssets must be withdrawn but due to the if condition it wrongly sets the toWithdraw value to 0 thus preventing any withdrawal.
## Impact
Assets are not withdrawn even when they should be 
## Code Snippet
https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/vaults/CuratedVault.sol#L250
## Tool used

Manual Review

## Recommendation
Remove the toWithdraw = 0 line from the above mentioned if condition



