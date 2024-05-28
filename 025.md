Witty Lime Albatross

medium

# in `StratFeeManagerInitializable::setStratFeeId()` setting new `stratFeeId` retrospectively applies new fees to pending LP rewards yet to be claimed

## Summary
in `StratFeeManagerInitializable::setStratFeeId()` it sets the feeData of specific strategy,  while LP rewards are collected and fees charged via `StrategyPassiveManagerVelodrome::_harvest()`
this override new fees on unharvested rewards
## Vulnerability Detail
in `StratFeeManagerInitializable::setStratFeeId()` it sets the feeData of specific strategy as shown here
```solidity
    function setStratFeeId(uint256 _feeId) external onlyManager {
        beefyFeeConfig().setStratFeeId(_feeId);
        emit SetStratFeeId(_feeId);
    }
 ```
 while LP rewards are collected and fees charged via `StrategyPassiveManagerVelodrome::_harvest()` as shown here
 ```solidity
     function _harvest (address _callFeeRecipient) private {
        // Claim rewards from gauge
        _claimEarnings();

        // Charge fees for Beefy and send them to the appropriate addresses, charge fees to accrued state fee amounts.
        (uint256 feeLeft) = _chargeFees(_callFeeRecipient, fees);

        // Reset state fees to 0. 
        fees = 0;

        // Notify rewards with our velo. 
        IRewardPool(rewardPool).notifyRewardAmount(output, feeLeft, 1 days);

        // Log the last time we claimed fees. 
        lastHarvest = block.timestamp;

        // Log the fees post Beefy fees.
        emit Harvest(feeLeft);
    }
 ```
 the problem here is that when (owner) calling `StratFeeManagerInitializable::setStratFeeId()` it changes the fee without harvesting rewards before doing so, rendering past unharvested rewards to the new fee which shouldn't be the case, as the owner want to change the fee of upcoming operations but this hurts stakers who have kept their money their with knowledge to the old fees.
 this bug lies under the sherlok rules of :
 > Admin functions are assumed to be used properly, unless a list of requirements is listed and it's incomplete or if there is no scenario where a permissioned funtion can be used properly.
 
 specifically:
 > if there is no scenario where a permissioned funtion can be used properly.
## Impact
medium: loss of user rewards
## Code Snippet
[here](https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L438-L456)
 ```solidity
     function _harvest (address _callFeeRecipient) private {
        // Claim rewards from gauge
        _claimEarnings();

        // Charge fees for Beefy and send them to the appropriate addresses, charge fees to accrued state fee amounts.
        (uint256 feeLeft) = _chargeFees(_callFeeRecipient, fees);

        // Reset state fees to 0. 
        fees = 0;

        // Notify rewards with our velo. 
        IRewardPool(rewardPool).notifyRewardAmount(output, feeLeft, 1 days);

        // Log the last time we claimed fees. 
        lastHarvest = block.timestamp;

        // Log the fees post Beefy fees.
        emit Harvest(feeLeft);
    }
 ```
 [here](https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/strategies/StratFeeManagerInitializable.sol#L139-L142)
 ```solidity
    function setStratFeeId(uint256 _feeId) external onlyManager {
        beefyFeeConfig().setStratFeeId(_feeId);
        emit SetStratFeeId(_feeId);
    }
 ```
## Tool used

Manual Review

## Recommendation
atomically add a call to harvest function before changing fees
```diff
    function setStratFeeId(uint256 _feeId) external onlyManager {
+       _harvest();   
        beefyFeeConfig().setStratFeeId(_feeId);
        emit SetStratFeeId(_feeId);
    }
 ```