Bouncy Midnight Mustang

high

# Liquidity providers cannot receive the trading fee rewards they are entitled to

## Summary
Liquidity providers cannot receive the trading fee rewards they are entitled to
## Vulnerability Detail
```javascript
    function _harvest (address _callFeeRecipient) private {
        // Claim rewards from gauge
        _claimEarnings();

        // Charge fees for Beefy and send them to the appropriate addresses, charge fees to accrued state fee amounts.
        (uint256 feeLeft) = _chargeFees(_callFeeRecipient, fees);

        // Reset state fees to 0. 
        fees = 0;

        // Notify rewards with our velo. 
@>      IRewardPool(rewardPool).notifyRewardAmount(output, feeLeft, 1 days);

        // Log the last time we claimed fees. 
        lastHarvest = block.timestamp;

        // Log the fees post Beefy fees.
        emit Harvest(feeLeft);
    }
```
We can see that left fee rewards arg given to rewardPool. 
There are two scenarios below：
1. The liquidity provider has not staked in the rewardPool and will not receive any rewards.
2. Even if staked, since the rewardPool releases rewards linearly over a day, a portion of the rewards will always be inaccessible.
#### poc for 2
- a. alice provides lptoken0 and lptoken1 ,meanwhile gets vault share token
- b. then stakes share token to rewardPool
- c. After a day has passed.
- d. alice call harvest() 
- e. After a day has passed.
- f. alice get all rewards of the first day. Then she wants to get her liquidity back.
Even if she calls harvest(), she cannot receive her rewards for the second day.
## Impact
Liquidity providers lose their fee rewards.
## Code Snippet
https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L438
## Tool used

Manual Review

## Recommendation
Output tokens swap to lpToken0 and lpToken1