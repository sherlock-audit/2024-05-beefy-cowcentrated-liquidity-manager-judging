Daring Basil Osprey

medium

# StrategyPassiveManagerVelodrome.sol#setRewardPool() - When changing `rewardPool`, fees accumulated for the current `rewardPool` will go to the new `rewardPooll`

## Summary
StrategyPassiveManagerVelodrome.sol#setRewardPool() - When changing `rewardPool`, fees accumulated for the current `rewardPool` will go to the new `rewardPooll`

## Vulnerability Detail
The protocol "harvests" the VELO fees, through `harvest`.

```jsx
function _harvest(address _callFeeRecipient) private {
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

We call `notifyRewardAmount` on the `rewardPool` which will then pull the `output` tokens out of the strategy.

It is possible to change the `rewardPool`, through `setRewardPool`.
```jsx
function setRewardPool(address _rewardPool) external onlyOwner {
        rewardPool = _rewardPool;
        emit SetRewardPool(_rewardPool);
    }
```

You'll notice that the function only changes the `rewardPool` and nothing else. This will lead to the case, where there are already accumulated fees that are supposed to go the current `rewardPool`, but `setRewardPool` is changed, which means next time `harvest` is called, the fees will go to the new `rewardPool`, which technically hasn't earned them.

Example:
1. There are 100 `output` tokens that are ready to be harvested and are supposed to go to `rewardPoolA`.
2. `setRewardPool` is called and now the `rewardPool` is `rewardPoolB`.
3. When `harvest` is called, all the fees will go to `rewardPoolB` instead of `rewardPoolA`.

## Impact
Fees won't be correctly distributed.

## Code Snippet
https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/42ef5f0eac1bc954e888cf5bfb85cbf24c08ec76/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L776-L779

## Tool used
Manual Review

## Recommendation
Call `harvest` inside `setRewardPool` prior to changing it.