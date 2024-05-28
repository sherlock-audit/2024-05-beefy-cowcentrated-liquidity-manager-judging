Soft Sangria Lion

medium

# StrategyPassiveManagerVelodrome::setDeviation incorrectly checks new _maxDeviation

## Summary

`StrategyPassiveManagerVelodrome::setDeviation()` incorrectly checks new `_maxDeviation`, which causes the owner to not be allowed to set it to the intended upper bound that should be allowed.

## Vulnerability Detail
[StrategyPassiveManagerVelodrome::setDeviation](https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L705-L712)
```solidity
    function setDeviation(int56 _maxDeviation) external onlyOwner {
        emit SetDeviation(_maxDeviation);

>>      // Require the deviation to be less than or equal to 4 times the tick spacing.
>>      if (_maxDeviation >= _tickDistance() * 4) revert InvalidInput();

        maxTickDeviation = _maxDeviation;
    }
```
When the owner sets the new `maxTickDeviation`, they are supposed to be allowed to set `deviation to be less than or equal to 4 times the tick spacing`, however the require statement will revert if the new `_maxDeviation == _tickDistance() * 4`.

## Impact

The owner is unable to set the `maxTickDeviation` to `_tickDistance() * 4` when this is intended to be the **inclusive** maximum upper bound. This breaks owner functionality due to the incorrect check.

Medium risk due to breaking intended functionality of the protocol to allow `maxTickDeviation` to be set to a maximum of `_tickDistance() * 4` as mentioned in the code comment.

## Code Snippet

[StrategyPassiveManagerVelodrome::setDeviation](https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L705-L712)

## Tool used

Manual Review

## Recommendation

Change the `>=` to `>` to allow the `maxTickDeviation` to be set to the upper intended value:
```diff
    function setDeviation(int56 _maxDeviation) external onlyOwner {
        emit SetDeviation(_maxDeviation);

        // Require the deviation to be less than or equal to 4 times the tick spacing.
-        if (_maxDeviation >= _tickDistance() * 4) revert InvalidInput();
+        if (_maxDeviation > _tickDistance() * 4) revert InvalidInput();

        maxTickDeviation = _maxDeviation;
    }
```
