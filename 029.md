Flaky Goldenrod Chinchilla

medium

# The `harvest` function should check if the feeLeft is greater than 0 before calling `notifyRewardAmount`


## Summary
The feeLeft can be set to 0 in the _chargeFees function if the fee.total equals DIVISOR (1e18).
Some ERC20 tokens do not allow transferring 0 value, so when the output token doesn't allow transferring 0 value, the harvest function will always fail.

## Vulnerability Detail
In the contract, the `_chargeFees` function may result in `feeLeft` being 0 under specific conditions.
Consequently, when the harvest function calls `notifyRewardAmount` with feeLeft set to 0, it will fail due to certain ERC20 tokens' restrictions on zero-value transfers.

## Impact
The harvest may fails if the fee.total is equal to DIVISOR(1e18).

## Code Snippet
https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L449
```solidity
    function _harvest (address _callFeeRecipient) private {
        // Claim rewards from gauge
        _claimEarnings();

        // Charge fees for Beefy and send them to the appropriate addresses, charge fees to accrued state fee amounts.
        (uint256 feeLeft) = _chargeFees(_callFeeRecipient, fees);

        // Reset state fees to 0. 
        fees = 0;
        // Notify rewards with our velo. 
        IRewardPool(rewardPool).notifyRewardAmount(output, feeLeft, 1 days); // @audit feeLeft can be zero 
        ...
    }
```
## Tool used

Manual Review

## Recommendation
It's recommended to add the validation before calling the `notifyRewardAmount` function.

```diff
    function _harvest (address _callFeeRecipient) private {
        // Claim rewards from gauge
        _claimEarnings();

        // Charge fees for Beefy and send them to the appropriate addresses, charge fees to accrued state fee amounts.
        (uint256 feeLeft) = _chargeFees(_callFeeRecipient, fees);

        // Reset state fees to 0. 
        fees = 0;
        // Notify rewards with our velo. 
+        if (feeLeft > 0)
        IRewardPool(rewardPool).notifyRewardAmount(output, feeLeft, 1 days);
        ...
    }
```
