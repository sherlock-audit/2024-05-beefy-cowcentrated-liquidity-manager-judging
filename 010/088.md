Bouncy Midnight Mustang

high

# `chargeFees()` only affects the output token, not fee0 and fee1


## Summary
`chargeFees()` only affects the output token, not fee0 and fee1
## Vulnerability Detail
```javascript
    function _harvest (address _callFeeRecipient) private {
        // Claim rewards from gauge
        _claimEarnings();

        // Charge fees for Beefy and send them to the appropriate addresses, charge fees to accrued state fee amounts.
    @>    (uint256 feeLeft) = _chargeFees(_callFeeRecipient, fees);

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
we can see that `chargeFees()` only affects the output token, not fee0 and fee1. _callFeeRecipient ,  strategist and beefyFeeRecipient() get less than what they are entitled to.
## Impact
_callFeeRecipient ,  strategist and beefyFeeRecipient() get less than what they are entitled to。
## Code Snippet
https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L438C1-L456C6
## Tool used

Manual Review

## Recommendation
Increase the allocation related to fee0 and fee1