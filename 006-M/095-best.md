Virtual Boysenberry Bear

medium

# The `isCalm()` function incorrectly checks whether the current price is within a certain deviation

## Summary
The `StrategyPassiveManagerVelodrome.twap()` function does not round to negative infinity for negative ticks, which affects the calculation of `minCalmTick and maxCalmTick` in `isCalm()`.

## Vulnerability Detail
The problem is that in case if `(tickCuml[1] - tickCuml[0])` is negative, then the tick should be rounded down as it's done in the [uniswap library](https://github.com/Uniswap/v3-periphery/blob/main/contracts/libraries/OracleLibrary.sol#L36). In this case returned tick will be bigger then it should be.
```solidity
(int56[] memory tickCuml,) = IVeloPool(pool).observe(secondsAgo);
twapTick = (tickCuml[1] - tickCuml[0]) / int32(twapInterval);
```
The `twapTick` will be larger, which will affect the calculation of `minCalmTick` and `maxCalmTick` in `isCalm()`, leading to transactions failing:
```solidity
        int56 twapTick = twap();

        int56 minCalmTick = int56(SignedMath.max(twapTick - maxTickDeviation, MIN_TICK));
        int56 maxCalmTick = int56(SignedMath.min(twapTick + maxTickDeviation, MAX_TICK));

        // Calculate if tick move more than allowed from twap and revert if it did. 
        if(minCalmTick > tick  || maxCalmTick < tick) return false;
```

## Impact
All function calls that have the `onlyCalmPeriods` modifier will fail due to the incorrect check for whether the current price is within a certain deviation.

## Code Snippet
[contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L747](https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L747)

## Tool used

Manual Review

## Recommendation
Consider modifying the `twap` function as follows:
```diff
function twap() public view returns (int56 twapTick) {
        uint32[] memory secondsAgo = new uint32[](2);
        secondsAgo[0] = uint32(twapInterval);
        secondsAgo[1] = 0;

        (int56[] memory tickCuml,) = IVeloPool(pool).observe(secondsAgo);
        twapTick = (tickCuml[1] - tickCuml[0]) / int32(twapInterval);
+     if ((tickCuml[1] - tickCuml[0]) < 0 && ((tickCuml[1] - tickCuml[0]) % twapInterval != 0)) twapTick--;        
    }
```
