Bouncy Midnight Mustang

high

# Missing reinvesting trading fees in `harvest()`

## Summary
Missing reinvesting trading fees in `harvest()`
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
        IRewardPool(rewardPool).notifyRewardAmount(output, feeLeft, 1 days);

        // Log the last time we claimed fees. 
        lastHarvest = block.timestamp;

        // Log the fees post Beefy fees.
        emit Harvest(feeLeft);
    }
```
Based on the documentation：
“Beefy's CLM products are laser focused on ensuring that the maximum amount of user capital in the product is deployed, in range and earning (or "fully active"). This includes not only the deposited funds, but also any trading fees accruing on the current investment and all past trading fees which have already been compounded back into the position. Where trading fees are constantly reinvested, we unlock the magic of compounding leading to significantly increased returns.  Every time a CLM product either receives a user deposit or is "harvested", Beefy automatically: (1) withdraws all deposits and trading fees to reset the position; (2) redeposits all of one token and most of the other into a precise 50/50 position; and (3) redeposits all other tokens into a single-side position”
And Compared to StrategyPassiveManagerUniswap, there is reinvestment of trading fees in in `harvest()`.
## Impact
Loss of reinvestment earnings
## Code Snippet
https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L438C1-L456C6
## Tool used

Manual Review

## Recommendation
Add reinvestment of trading fees in in `harvest()`
