# [H-01] ibgt rewards will be lost when claiming rewards for the wrapped vault with staking token as ibgt.



## Description
In the comments of wrapped vault contract it is clearly said that there will 1 wrapped vault corresponding to each staking token so this means that there will also be a wrapped vault corresponding to ibgt as it is also used as a staking token. Now similar to other infrared vaults ibgt vault would also earn bgt rewards which will be sent to the infrared contract and corresponding ibgt will be minted and those ibgt will be sent to the ibgt vault as rewards. Now the issue is that when a user uses the wrapped vault corresponding to ibgt staking token then these ibgt rewards would be ignored because of the if condition which is referenced along with this finding. This if condition works absolutely fine with the other staking tokens because no vault earns rewards in terms of staking token but in case of ibgt vault they do earn rewards in terms of the staking token thus this issue arises. Now as these rewards will be ignored this will lead to loss of rewards.

## Proof of Concept
Following is how ibgt vault is initialised 
```solidity
  function setIBGT(address _ibgt) external {
        if (_ibgt == address(0)) revert Errors.ZeroAddress();
        if (address(ibgt) != address(0)) revert Errors.AlreadySet();
        if (
            !InfraredBGT(_ibgt).hasRole(
                InfraredBGT(_ibgt).MINTER_ROLE(), address(this)
            )
        ) {
            revert Errors.Unauthorized(address(this));
        }
        ibgt = InfraredBGT(_ibgt);
        _vaultStorage().updateWhitelistedRewardTokens(address(ibgt), true);
        ibgtVault = IInfraredVault(_vaultStorage().registerVault(address(ibgt)));

        emit NewVault(msg.sender, address(ibgt), address(ibgtVault));
        emit IBGTSet(msg.sender, _ibgt);
    }
```
RegisterVault function is as follows 
```solidity
function registerVault(VaultStorage storage $, address asset)
        external
        notPaused($)
        returns (address)
    {
        if (asset == address(0)) revert Errors.ZeroAddress();

        // Check for duplicate staking asset address
        if (address($.vaultRegistry[asset]) != address(0)) {
            revert Errors.DuplicateAssetAddress();
        }

        address newVault =
            InfraredVaultDeployer.deploy(asset, $.rewardsDuration);
        $.vaultRegistry[asset] = IInfraredVault(newVault);
        return newVault;
    }

```
Deploy function is as follows
```solidity
 function deploy(address _stakingToken, uint256 _rewardsDuration)
        public
        returns (address _new)
    {
        return address(new InfraredVault(_stakingToken, _rewardsDuration));
    }
```
When a new infrared vault is created a new berachain reward vault is also created as can be seen from the constructor. All the staked ibgt is sent to this reward vault and all the bgt rewards earned by the ibgt vault are sent from this berachain rewards vault.
```solidity
 constructor(address _stakingToken, uint256 _rewardsDuration)
        MultiRewards(_stakingToken)
    {
        // infrared factory/coordinator
        infrared = msg.sender;

        if (_stakingToken == address(0)) revert Errors.ZeroAddress();
        if (_rewardsDuration == 0) revert Errors.ZeroAmount();

        // set the berachain rewards vault and operator as infrared
        rewardsVault = _createRewardsVaultIfNecessary(infrared, _stakingToken);
        rewardsVault.setOperator(infrared);

        address _ibgt = address(IInfrared(infrared).ibgt());

        _addReward(_ibgt, infrared, _rewardsDuration);

        // to be able to recover rewards which where distributed during periods where there was no stake
        // infrared will have a stake of 1 wei in the vault
        _totalSupply = _totalSupply + 1;
        _balances[msg.sender] = _balances[msg.sender] + 1;
    }

```
It is clear from all the above explained functions that the ibgt vault also earns bgt rewards beacuse of the staked ibgt tokens.
The earned bgt rewards are sent to the infrared contract and corresponding ibgt tokens are minted to the ibgt vault using the harvest vault function.
```solidity
function harvestVault(address _asset) external {
        IInfraredVault vault = vaultRegistry(_asset);
        uint256 bgtAmt = _rewardsStorage().harvestVault(
            vault,
            address(_bgt),
            address(ibgt),
            address(voter),
            address(red),
            rewardsDuration()
        );
        emit VaultHarvested(msg.sender, _asset, address(vault), bgtAmt);
    }


    function harvestVault(
        RewardsStorage storage $,
        IInfraredVault vault,
        address bgt,
        address ibgt,
        address voter,
        address red,
        uint256 rewardsDuration
    ) external returns (uint256 bgtAmt) {
        // Ensure the vault is valid
        if (vault == IInfraredVault(address(0))) {
            revert Errors.VaultNotSupported();
        }

        // Record the BGT balance before claiming rewards
        uint256 balanceBefore = _getBGTBalance(bgt);

        // Get the rewards from the vault's reward vault
        IBerachainRewardsVault rewardsVault = vault.rewardsVault();
        rewardsVault.getReward(address(vault), address(this));

        // Calculate the amount of BGT rewards received
        bgtAmt = _getBGTBalance(bgt) - balanceBefore;

        // If no BGT rewards were received, exit early
        if (bgtAmt == 0) return bgtAmt;

        // Mint InfraredBGT tokens equivalent to the BGT rewards
        IInfraredBGT(ibgt).mint(address(this), bgtAmt);

        // Calculate and distribute fees on the BGT rewards
        (uint256 _amt, uint256 _amtVoter, uint256 _amtProtocol) =
        _chargedFeesOnRewards(
            bgtAmt,
            $.fees[uint256(ConfigTypes.FeeType.HarvestVaultFeeRate)],
            $.fees[uint256(ConfigTypes.FeeType.HarvestVaultProtocolRate)]
        );
        _distributeFeesOnRewards(
            $.protocolFeeAmounts, voter, ibgt, _amtVoter, _amtProtocol
        );

        // Send the remaining InfraredBGT rewards to the vault
        if (_amt > 0) {
            ERC20(ibgt).safeApprove(address(vault), _amt);
            vault.notifyRewardAmount(ibgt, _amt);
        }

        uint256 mintRate = $.redMintRate;

        // If RED token is set and mint rate is greater than zero, handle RED rewards
        if (red != address(0) && mintRate > 0) {
            // Calculate the amount of RED tokens to mint
            uint256 redAmt = bgtAmt * mintRate / RATE_UNIT;
            try IRED(red).mint(address(this), redAmt) {
                {
                    // Check if RED is already a reward token in the vault
                    (, uint256 redRewardsDuration,,,,,) = vault.rewardData(red);
                    if (redRewardsDuration == 0) {
                        // Add RED as a reward token if not already added
                        vault.addReward(red, rewardsDuration);
                    }
                }

                // Calculate and distribute fees on the RED rewards
                (_amt, _amtVoter, _amtProtocol) =
                    _chargedFeesOnRewards(redAmt, 0, 0);
                _distributeFeesOnRewards(
                    $.protocolFeeAmounts, voter, red, _amtVoter, _amtProtocol
                );

                // Send the remaining RED rewards to the vault
                if (_amt > 0) {
                    ERC20(red).safeApprove(address(vault), _amt);
                    vault.notifyRewardAmount(red, _amt);
                }
            } catch {
                emit RedNotMinted(redAmt);
            }
        }
    }
```
So now when claim function is called in wrapped vault these igbt rewards earned will be lost
```solidity
 function claimRewards() external {
        // Claim rewards from the InfraredVault
        iVault.getReward();
        // Retrieve all reward tokens
        address[] memory _tokens = iVault.getAllRewardTokens();
        uint256 len = _tokens.length;
        // Loop through reward tokens and transfer them to the reward distributor
        for (uint256 i; i < len; ++i) {
            ERC20 _token = ERC20(_tokens[i]);
            // Skip if the reward token is the staking token
@==>            if (_token == asset) continue;
            uint256 bal = _token.balanceOf(address(this));
            if (bal == 0) continue;
            (bool success, bytes memory data) = address(_token).call(
                abi.encodeWithSelector(
                    ERC20.transfer.selector, rewardDistributor, bal
                )
            );
            if (success && (data.length == 0 || abi.decode(data, (bool)))) {
                emit RewardClaimed(address(_token), bal);
            } else {
                continue;
            }
        }
    }
```


## Recommendation
Don't skip reward token in case of ibgt staking token.


