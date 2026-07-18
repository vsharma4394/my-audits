# [M-01] Incorrect required underlying value check used in mintFromFreeUnderlying function

## Brief/Intro

Incorrect required underlying value check used in mintFromFreeUnderlying function which causes less amount to be minted than the available amount.

## Vulnerability Details

In the minFromFreeUnderlying function the vault owner can mint FAsset using the underlying free balance the agent has. So for minting a certain amount of FAsset tokens, it is required that the agent must hold a minimum underlying balance otherwise it will get liquidated. For calculating how much amount of underlying the agent must have which is calculating by multiplying the amount of FAsset which will be minted with minUnderlyingBackingBIPS and the value of minUnderlyingBackingBIPS is less than 100% so less than minted amount is required to available in the agent underlying balance.Now the issue is in the mint from free underlying function minted amount is not multiplied with minUnderlyingBackingBIPS due to which incorrect amount is used when checking that the underlying balance of the agent is sufficient.

## Impact Details

This will prevent the agent from minting using the underlying balance even when they have sufficient balance.

## References

<https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/library/Minting.sol#L132>

## Proof of Concept

## Proof of Concept

1. Lets suppose that the available free lots are 2 lots and the agent vault owner calls the mint from free underlying function which is as follows

```solidity

    function mintFromFreeUnderlying(
        address _agentVault,
        uint64 _lots
    )
        internal
    {
        AssetManagerState.State storage state = AssetManagerState.get();
        Agent.State storage agent = Agent.get(_agentVault);
        Agents.requireAgentVaultOwner(agent);
        Agents.requireWhitelistedAgentVaultOwner(agent);
        Collateral.CombinedData memory collateralData = AgentCollateral.combinedData(agent);
        require(state.mintingPausedAt == 0, "minting paused");
        require(_lots > 0, "cannot mint 0 lots");
        require(agent.status == Agent.Status.NORMAL, "self-mint invalid agent status");
        require(collateralData.freeCollateralLots(agent) >= _lots, "not enough free collateral");
        uint64 valueAMG = _lots * Globals.getSettings().lotSizeAMG;
        checkMintingCap(valueAMG);
        uint256 mintValueUBA = Conversion.convertAmgToUBA(valueAMG);
        uint256 poolFeeUBA = calculateCurrentPoolFeeUBA(agent, mintValueUBA);
        uint256 requiredUnderlyingAfter = UnderlyingBalance.requiredUnderlyingUBA(agent) + mintValueUBA + poolFeeUBA;
        require(requiredUnderlyingAfter.toInt256() <= agent.underlyingBalanceUBA, "free underlying balance to small");
        _performMinting(agent, MintingType.FROM_FREE_UNDERLYING, 0, msg.sender, valueAMG, 0, poolFeeUBA);
    }
```

2. So the value amg will be 2*lotSize. After that the pool fee will be calculated using 2*lotSize amg.

3 . Total amount of FAsset minted will be mintValyeUBA + The poolFeeUBA

4. So for minting mintValueUBA + poolFeeUBA to check that the agent should have free underlying balance = (mintValueUBA + poolFeeUBA).mulBIPS(minUnderlyingBackingBIPS )
5. But in the current implementation it is checked that the whole minting value UBA + the pool fee UBA balance must be there which is incorrect because more than the required balance is used.

```solidity
 uint256 requiredUnderlyingAfter = UnderlyingBalance.requiredUnderlyingUBA(agent) + mintValueUBA + poolFeeUBA;
        require(requiredUnderlyingAfter.toInt256() <= agent.underlyingBalanceUBA, "free underlying balance to small");
      
```

# [M-02]check minting cap function checks on incorrect amount in mintFromFreeUnderlying function


## Brief/Intro

check minting cap function checks on incorrect amount in mintFromFreeUnderlying function because it doesn't takes into account the amount that will be minted because of the pool fee.

## Vulnerability Details

In the mint from free underlying function there is a check minting cap call which ensures that the minted amount should be less than the minting cap set but in mint from free underlying function it doesn't take into account the amount that will be minted because of the pool fee. in the proof of concept i have clearly shown where the pool fees has not been included.

## Impact Details

It causes an important invariant to fail as it will allow more than minting cap to be minted.

## References

<https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/library/Minting.sol#L129>

## Proof of Concept

## Proof of Concept

1. The agent vault owner calls the self mint function and following is mint from free underlying function

```solidity
 function mintFromFreeUnderlying(
        address _agentVault,
        uint64 _lots
    )
        internal
    {
        AssetManagerState.State storage state = AssetManagerState.get();
        Agent.State storage agent = Agent.get(_agentVault);
        Agents.requireAgentVaultOwner(agent);
        Agents.requireWhitelistedAgentVaultOwner(agent);
        Collateral.CombinedData memory collateralData = AgentCollateral.combinedData(agent);
        require(state.mintingPausedAt == 0, "minting paused");
        require(_lots > 0, "cannot mint 0 lots");
        require(agent.status == Agent.Status.NORMAL, "self-mint invalid agent status");
        require(collateralData.freeCollateralLots(agent) >= _lots, "not enough free collateral");
        uint64 valueAMG = _lots * Globals.getSettings().lotSizeAMG;
        checkMintingCap(valueAMG);
        uint256 mintValueUBA = Conversion.convertAmgToUBA(valueAMG);
        uint256 poolFeeUBA = calculateCurrentPoolFeeUBA(agent, mintValueUBA);
        uint256 requiredUnderlyingAfter = UnderlyingBalance.requiredUnderlyingUBA(agent) + mintValueUBA + poolFeeUBA;
        require(requiredUnderlyingAfter.toInt256() <= agent.underlyingBalanceUBA, "free underlying balance to small");
        _performMinting(agent, MintingType.FROM_FREE_UNDERLYING, 0, msg.sender, valueAMG, 0, poolFeeUBA);
    }
```

2. We can see there is a checkMintingCap(valueAMG) and valueAMG = \_lots\*Globals.getSettings().lotSizeAMG;
3. Then we can see pool fee being calculated as follows poolFeeUBA = calculateCurrentPoolFeeUBA(agent, mintValueUBA);

Then in perform minting function we can see that the poolfeeuba is also minted to the pool \_performMinting(agent, MintingType.SELF\_MINT, 0, msg.sender, valueAMG, 0, poolFeeUBA);

```solidity
function _performMinting(
        Agent.State storage _agent,
        MintingType _mintingType,
        uint64 _crtId,
        address _minter,
        uint64 _mintValueAMG,
        uint256 _receivedAmountUBA,
        uint256 _poolFeeUBA
    )
        private
    {
        uint64 poolFeeAMG = Conversion.convertUBAToAmg(_poolFeeUBA);
        Agents.createNewMinting(_agent, _mintValueAMG + poolFeeAMG);
        // update agent balance with deposited amount (received amount is 0 in mintFromFreeUnderlying)
        UnderlyingBalance.increaseBalance(_agent, _receivedAmountUBA);
        // perform minting
        uint256 mintValueUBA = Conversion.convertAmgToUBA(_mintValueAMG);
        Globals.getFAsset().mint(_minter, mintValueUBA);
        Globals.getFAsset().mint(address(_agent.collateralPool), _poolFeeUBA);
        _agent.collateralPool.fAssetFeeDeposited(_poolFeeUBA);
        // notify
        if (_mintingType == MintingType.PUBLIC) {
            uint256 agentFeeUBA = _receivedAmountUBA - mintValueUBA - _poolFeeUBA;
            emit IAssetManagerEvents.MintingExecuted(_agent.vaultAddress(), _crtId,
                mintValueUBA, agentFeeUBA, _poolFeeUBA);
        } else {
            bool fromFreeUnderlying = _mintingType == MintingType.FROM_FREE_UNDERLYING;
            emit IAssetManagerEvents.SelfMint(_agent.vaultAddress(), fromFreeUnderlying,
                mintValueUBA, _receivedAmountUBA, _poolFeeUBA);
        }
    }
```
# [L-01]Incorrect calculation of total available amount in core vault in a certain case when a user redeems from the core vault

## Brief/Intro

When a user who already has a available non cancelable transfer request again then the total available amount returned is incorrect in that case because it again considers the fee which in reality is not charged again.

## Vulnerability Details

In the case of non cancelable transfer requests fees is only charged when there is a new request added but if a redeemer whose request has already been added again requests a transfer no fees is applied to it but when total available amount in the core vault is calculated it again takens into account the fee which is incorrect.

## Impact Details

It incorrectly charges fees and causes less available amount calculation even when there is certainly enough tokens available.

## References

<https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/library/CoreVault.sol#L220> <https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/library/CoreVault.sol#L270>

## Proof of Concept

## Proof of Concept

1. Suppose the user has already called redeem from core vault therefore it will have a transfer request in nonCancelableTransferRequests array

```solidity
 uint256[] private nonCancelableTransferRequests;
```

2. Now suppose a user again calls redeem from core vault with the same destination address so the following calculation for the available amount will be done

```solidity
function getCoreVaultAvailableAmount()
        internal view
        returns (uint256 _immediatelyAvailableUBA, uint256 _totalAvailableUBA)
    {
        State storage state = getState();
        uint256 availableFunds = state.coreVaultManager.availableFunds();
        uint256 escrowedFunds = state.coreVaultManager.escrowedFunds();
        // account for fee for one more request, because this much must remain available on any transfer
        uint256 requestedAmountWithFee =
            state.coreVaultManager.totalRequestAmountWithFee() + getCoreVaultUnderlyingPaymentFee();
        _immediatelyAvailableUBA = MathUtils.subOrZero(availableFunds, requestedAmountWithFee);
        _totalAvailableUBA = MathUtils.subOrZero(availableFunds + escrowedFunds, requestedAmountWithFee);
    }
```

We can clearly see that again fee amount is taken into account while calculating the amount available. Now see in the case of non cancelable requests no new transfer request is added for the available destination address

```solidity
else {
            uint256 index = 0;
            while (index < nonCancelableTransferRequests.length) {
                TransferRequest storage req = transferRequestById[nonCancelableTransferRequests[index]];
==>                if (keccak256(bytes(req.destinationAddress)) == destinationAddressHash) {
                    // add the amount to the existing request
                    req.amount += _amount;
                    _paymentReference = req.paymentReference;   // use the old payment reference when merged
                    break;
                }
                index++;
            }
            nonCancelableTransferRequestsAmount += _amount;
            // if the request does not exist, add a new one
            if (index == nonCancelableTransferRequests.length) {
                nonCancelableTransferRequests.push(nextTransferRequestId);
                newTransferRequest = true;
            }
```

Therefore incorrect calculation of available amount is done. This can also cause less available lots available for example

```solidity
uint256 requestedAmountWithFee =
            state.coreVaultManager.totalRequestAmountWithFee() + getCoreVaultUnderlyingPaymentFee();
        _immediatelyAvailableUBA = MathUtils.subOrZero(availableFunds, requestedAmountWithFee);
        _totalAvailableUBA = MathUtils.subOrZero(availableFunds + escrowedFunds, requestedAmountWithFee);
```

Suppose total available amount after taking into account the fee is less than the lot size then the following function will return zero.

```solidity
 function getCoreVaultAmountLots()
        internal view
        returns (uint64)
    {
        AssetManagerSettings.Data storage settings = Globals.getSettings();
        (, uint256 totalAmountUBA) = getCoreVaultAvailableAmount();
        return Conversion.convertUBAToAmg(totalAmountUBA) / settings.lotSizeAMG;
    }
```

Now hadn't the fees been taken into account it would have returned 1 lot availble which is the correct amount available.
