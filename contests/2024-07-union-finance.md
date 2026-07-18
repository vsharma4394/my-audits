

# [H-01] _accrueRewards function in Comptroller.sol uses outdated value of globalTotalStaked variable.

## Summary
_accrueRewards function in Comptroller.sol uses outdated value of globalTotalStaked variable which causes incorrect calculation of the rewards.

## Vulnerability Detail
Following is _accrueRewards function 
```solidity
function _accrueRewards(address account, address token) private returns (uint256) {
        IUserManager userManager = _getUserManager(token);

        // Lookup global state from UserManager
        uint256 globalTotalStaked = userManager.globalTotalStaked();

        // Lookup account state from UserManager
        UserManagerAccountState memory user = UserManagerAccountState(0, 0, false);
        (user.effectiveStaked, user.effectiveLocked, user.isMember) = userManager.onWithdrawRewards(account);

        uint256 amount = _calculateRewardsInternal(account, token, globalTotalStaked, user);

        // update the global states
        gInflationIndex = _getInflationIndexNew(globalTotalStaked, getTimestamp() - gLastUpdated);
        gLastUpdated = getTimestamp();
        users[account][token].inflationIndex = gInflationIndex;

        return amount;
    }
```
From the above it is can be seen that first the global state is looked up using 
```solidity
        uint256 globalTotalStaked = userManager.globalTotalStaked();
```
then account state is looked up using the following 
```solidity
 UserManagerAccountState memory user = UserManagerAccountState(0, 0, false);
        (user.effectiveStaked, user.effectiveLocked, user.isMember) = userManager.onWithdrawRewards(account);
```
Now issue is that the userManager.onWithdrawRewards(account) updates the value of _totalFrozen which can be seen from the following function
```solidity
function onWithdrawRewards(
        address staker
    ) external returns (uint256 effectiveStaked, uint256 effectiveLocked, bool isMember) {
        if (address(comptroller) != msg.sender) revert AuthFailed();
        uint256 memberTotalFrozen = 0;
        (effectiveStaked, effectiveLocked, memberTotalFrozen) = _getEffectiveAmounts(staker);
        _stakers[staker].stakedCoinAge = 0;
        uint256 currTime = getTimestamp();
        _stakers[staker].lastUpdated = currTime.toUint64();
        gLastWithdrawRewards[staker] = currTime;
        _stakers[staker].lockedCoinAge = 0;
        _frozenCoinAge[staker] = 0;

        uint256 memberFrozenBefore = _memberFrozen[staker];
        if (memberFrozenBefore != memberTotalFrozen) {
            _memberFrozen[staker] = memberTotalFrozen;
            _totalFrozen = _totalFrozen - memberFrozenBefore + memberTotalFrozen;
        }

        isMember = _stakers[staker].isMember;
    }
```
As userManager.globalTotalStaked() returns _totalStaked - _totalFrozen therefore only the latest/updated value should be used.
As value returned by globalTotalStaked is used to calculate the gInflationIndex which essentially calculated the reward. Using outdated values will causes wrong calculation of the rewards.



## Impact
Wrong calculation of the rewards for stakers.
## Code Snippet
https://github.com/sherlock-audit/2024-06-union-finance-update-2/blob/7ffe43f68a1b8e8de1dfd9de5a4d89c90fd6f710/union-v2-contracts/contracts/token/Comptroller.sol#L224
## Tool used

Manual Review

## Recommendation
Change the order of calling the functions as follows
```solidity
function _accrueRewards(address account, address token) private returns (uint256) {
        IUserManager userManager = _getUserManager(token);

     
        // Lookup account state from UserManager
        UserManagerAccountState memory user = UserManagerAccountState(0, 0, false);
        (user.effectiveStaked, user.effectiveLocked, user.isMember) = userManager.onWithdrawRewards(account);
        
        // Lookup global state from UserManager
        uint256 globalTotalStaked = userManager.globalTotalStaked();

        uint256 amount = _calculateRewardsInternal(account, token, globalTotalStaked, user);

        // update the global states
        gInflationIndex = _getInflationIndexNew(globalTotalStaked, getTimestamp() - gLastUpdated);
        gLastUpdated = getTimestamp();
        users[account][token].inflationIndex = gInflationIndex;

        return amount;
    }
```


# [H-02] Interest amount is not scaled which can cause various accounting issues.

## Summary
As the codebase intends to work with usdc and usdt which have less than 18 decimals so the input amount is needed to scaled to 18 decimals but wrong function is called for calculating the interest which causes incorrect updation of key variables.
## Vulnerability Detail
Following is repayBorrowWithERC20Permit function which is used to repay the borrowed amount.
```solidity
function repayBorrowWithERC20Permit(
        address borrower,
        uint256 amount,
        uint256 deadline,
        uint8 v,
        bytes32 r,
        bytes32 s
    ) external whenNotPaused {
        IERC20Permit erc20Token = IERC20Permit(underlying);
        erc20Token.permit(msg.sender, address(this), amount, deadline, v, r, s);

        if (!accrueInterest()) revert AccrueInterestFailed();
        uint256 interest = calculatingInterest(borrower);
        _repayBorrowFresh(msg.sender, borrower, decimalScaling(amount, underlyingDecimal), interest);
    }
```
The error is the usage of  calculatingInterest(borrower) function because this function essentially returns interest in terms of underlying decimals whereas interest value returned should be in 18 decimals.

```solidity
 function calculatingInterest(address account) public view override returns (uint256) {
        return decimalReducing(_calculatingInterest(account), underlyingDecimal);
    }
```
All other repay functions use _calculatingInterest(account) function in order to calculate the interest because it returns the value in 18 decimals as can be seen below.
```solidity
function repayInterest(address borrower) external override whenNotPaused nonReentrant {
        if (!accrueInterest()) revert AccrueInterestFailed();
        uint256 interest = _calculatingInterest(borrower);
        _repayBorrowFresh(msg.sender, borrower, interest, interest);
    }

    /**
     * @notice Repay outstanding borrow
     * @dev Repay borrow see _repayBorrowFresh
     */
    function repayBorrow(address borrower, uint256 amount) external override whenNotPaused nonReentrant {
        if (!accrueInterest()) revert AccrueInterestFailed();
        uint256 actualAmount = decimalScaling(amount, underlyingDecimal);
        uint256 interest = _calculatingInterest(borrower);
        _repayBorrowFresh(msg.sender, borrower, actualAmount, interest);
    }
```

## Impact
Impact is incorrect updation of _totalBorrows variable. Not only this,user can also get away by repaying less amount because of the following if condition 
if (repayAmount >= interest)  because repay amount would in 18 decimals whereas interest would be in less than 18 decimals.(in case of usdc and usdt it would be in 6 decimals.) so even by paying a very less amount user can repay their interest.
```solidity
 function _repayBorrowFresh(address payer, address borrower, uint256 amount, uint256 interest) internal {
        uint256 currTime = getTimestamp();
        if (currTime != accrualTimestamp) revert AccrueBlockParity();
        uint256 borrowedAmount = borrowBalanceStoredInternal(borrower);
        uint256 repayAmount = amount > borrowedAmount ? borrowedAmount : amount;
        if (repayAmount == 0) revert AmountZero();

        uint256 toReserveAmount;
        uint256 toRedeemableAmount;

        if (repayAmount >= interest) {
            // Interest is split between the reserves and the uToken minters based on
            // the reserveFactorMantissa When set to WAD all the interest is paid to teh reserves.
            // any interest that isn't sent to the reserves is added to the redeemable amount
            // and can be redeemed by uToken minters.
            toReserveAmount = (interest * reserveFactorMantissa) / WAD;
            toRedeemableAmount = interest - toReserveAmount;

            // Update the total borrows to reduce by the amount of principal that has
            // been paid off
            _totalBorrows -= (repayAmount - interest);

            // Update the account borrows to reflect the repayment
            accountBorrows[borrower].principal = borrowedAmount - repayAmount;
            accountBorrows[borrower].interest = 0;

            uint256 pastTime = currTime - getLastRepay(borrower);
            if (pastTime > overdueTime) {
                // For borrowers that are paying back overdue balances we need to update their
                // frozen balance and the global total frozen balance on the UserManager
                IUserManager(userManager).onRepayBorrow(borrower, getLastRepay(borrower) + overdueTime);
            }

            // Call update locked on the userManager to lock this borrowers stakers. This function
            // will revert if the account does not have enough vouchers to cover the repay amount. ie
            // the borrower is trying to repay more than is locked (owed)
            IUserManager(userManager).updateLocked(
                borrower,
                decimalReducing(repayAmount - interest, underlyingDecimal),
                false
            );

            if (_getBorrowed(borrower) == 0) {
                // If the principal is now 0 we can reset the last repaid time to 0.
                // which indicates that the borrower has no outstanding loans.
                accountBorrows[borrower].lastRepay = 0;
            } else {
                // Save the current block timestamp as last repaid
                accountBorrows[borrower].lastRepay = currTime;
            }
        } else {
            // For repayments that don't pay off the minimum we just need to adjust the
            // global balances and reduce the amount of interest accrued for the borrower
            toReserveAmount = (repayAmount * reserveFactorMantissa) / WAD;
            toRedeemableAmount = repayAmount - toReserveAmount;
            accountBorrows[borrower].interest = interest - repayAmount;
        }

        _totalReserves += toReserveAmount;
        _totalRedeemable += toRedeemableAmount;

        accountBorrows[borrower].interestIndex = borrowIndex;

        // Transfer underlying token that have been repaid and then deposit
        // then in the asset manager so they can be distributed between the
        // underlying money markets
        uint256 sendAmount = decimalReducing(repayAmount, underlyingDecimal);
        IERC20Upgradeable(underlying).safeTransferFrom(payer, address(this), sendAmount);
        _depositToAssetManager(sendAmount);

        emit LogRepay(payer, borrower, sendAmount);
    }
```
## Code Snippet
https://github.com/sherlock-audit/2024-06-union-finance-update-2/blob/7ffe43f68a1b8e8de1dfd9de5a4d89c90fd6f710/union-v2-contracts/contracts/market/UErc20.sol#L20
## Tool used

Manual Review

## Recommendation
instead of calculatingInterest() function use _calculatingInterest() function.



# [H-03] In debtWriteOff function _totalStaked variable is reduced by unscaled amount.

## Summary
In debtWriteOff function _totalStaked variable is reduced by unscaled amount which causes wrong updation of _totalStaked variable.
## Vulnerability Detail
Following is the In debtWriteOff function
```solidity
    function debtWriteOff(address stakerAddress, address borrowerAddress, uint256 amount) external {
        if (amount == 0) revert AmountZero();
        uint256 actualAmount = decimalScaling(amount, stakingTokenDecimal);
        uint256 overdueTime = uToken.overdueTime();
        uint256 lastRepay = uToken.getLastRepay(borrowerAddress);
        uint256 currTime = getTimestamp();

        // This function is only callable by the public if the loan is overdue by
        // overdue time + maxOverdueTime. This stops the system being left with
        // debt that is overdue indefinitely and no ability to do anything about it.
        if (currTime <= lastRepay + overdueTime + maxOverdueTime) {
            if (stakerAddress != msg.sender) revert AuthFailed();
        }

        Index memory index = voucherIndexes[borrowerAddress][stakerAddress];
        if (!index.isSet) revert VoucherNotFound();
        Vouch storage vouch = _vouchers[borrowerAddress][index.idx];
        uint256 locked = vouch.locked;
        if (actualAmount > locked) revert ExceedsLocked();

        comptroller.accrueRewards(stakerAddress, stakingToken);

        Staker storage staker = _stakers[stakerAddress];

        staker.stakedAmount -= actualAmount.toUint96();
        staker.locked -= actualAmount.toUint96();
        staker.lastUpdated = currTime.toUint64();
==>     _totalStaked -= amount;

        // update vouch trust amount
        vouch.trust -= actualAmount.toUint96();
        vouch.locked -= actualAmount.toUint96();
        vouch.lastUpdated = currTime.toUint64();

        // Update total frozen and member frozen. We don't want to move th
        // burden of calling updateFrozenInfo into this function as it is quite
        // gas intensive. Instead we just want to remove the actualAmount that was
        // frozen which is now being written off. However, it is possible that
        // member frozen has not been updated prior to calling debtWriteOff and
        // the actualAmount being written off could be greater than the actualAmount frozen.
        // To avoid an underflow here we need to check this condition
        uint256 stakerFrozen = _memberFrozen[stakerAddress];
        if (actualAmount > stakerFrozen) {
            // The actualAmount being written off is more than the actualAmount that has
            // been previously frozen for this staker. Reset their frozen stake
            // to zero and adjust _totalFrozen
            _memberFrozen[stakerAddress] = 0;
            _totalFrozen -= stakerFrozen;
        } else {
            _totalFrozen -= actualAmount;
            _memberFrozen[stakerAddress] -= actualAmount;
        }

        if (vouch.trust == 0) {
            _cancelVouchInternal(stakerAddress, borrowerAddress);
        }

        // Notify the AssetManager and the UToken market of the debt write off
        // so they can adjust their balances accordingly
        IAssetManager(assetManager).debtWriteOff(stakingToken, amount);
        uToken.debtWriteOff(borrowerAddress, amount);
        emit LogDebtWriteOff(msg.sender, borrowerAddress, amount);
    }
```
As can be seen from the line  _totalStaked -= amount;  it is clear that the _totalStaked value is decreased by unscaled amount thus it causes _totalStaked value to reduce by a way less number than it really should be reduced by.

## Impact
As _totalStaked variable is used in the calculation of rewards for the stakers in comptroller contract it causes wrong calculation of the rewards thus high severity.

## Code Snippet
https://github.com/sherlock-audit/2024-06-union-finance-update-2/blob/7ffe43f68a1b8e8de1dfd9de5a4d89c90fd6f710/union-v2-contracts/contracts/user/UserManager.sol#L834
## Tool used

Manual Review

## Recommendation
reduce the _totalStaked value as follows
_totalStaked -= actualAmount.toUint96();



# [H-04] In vouchFaucet value of claimedTokens[token][msg.sender] is never set.

## Summary
In vouchFaucet value of claimedTokens[token][msg.sender] is never set due to which a user can claim tokens beyond maxClaimable limit.
## Vulnerability Detail
Following is claimTokens function
```solidity
function claimTokens(address token, uint256 amount) external {
        require(claimedTokens[token][msg.sender] <= maxClaimable[token], "amount>max");
        IERC20(token).transfer(msg.sender, amount);
        emit TokensClaimed(msg.sender, token, amount);
    }
```
In the contract no where the value of claimedTokens[token][msg.sender] is set due to which its value will always be zero and thus less than maxClaimable[token].Even if the value of maxClaimable = 0 then also the require check passes.
## Impact
This causes users to claim arbitrary amount of tokens.
## Code Snippet
https://github.com/sherlock-audit/2024-06-union-finance-update-2/blob/7ffe43f68a1b8e8de1dfd9de5a4d89c90fd6f710/union-v2-contracts/contracts/peripheral/VouchFaucet.sol#L93
## Tool used

Manual Review

## Recommendation
make the following change in the claimTokens function
```solidity
 function claimTokens(address token, uint256 amount) external {
        require(claimedTokens[token][msg.sender] + amount <= maxClaimable[token], "amount>max");
        claimedTokens[token][msg.sender] += amount;
        IERC20(token).transfer(msg.sender, amount);
        emit TokensClaimed(msg.sender, token, amount);
    }
```
