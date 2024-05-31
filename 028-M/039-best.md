Polished Lemonade Chinchilla

medium

# Changing `positionWidth` also centers the position

## Summary
Changing `positionWidth` also centralizes the position

## Vulnerability Detail
Changing `setPositionWidth`, calls `_setTicks` which always sets the main position in such way that it is currently 50/50 split between the two tokens.

```solidity
    function _setMainTick(int24 tick, int24 distance, int24 width) private {
        (positionMain.tickLower, positionMain.tickUpper) = TickUtils.baseTicks(
            tick,
            width,
            distance
        );
    }
```

This is problematic as it would result in significant part of the funds unusable as if the position is currently in a 20/80 split, there's no need to change it into 50/50, when it can just spread the ticks while maintaining that same token ratio, in order to preserve most of the token supply in liquidity circulation.

## Impact
Suboptimal liquidity management resulting in less fees accrued for the users

## Code Snippet
https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L768

## Tool used

Manual Review

## Recommendation
Keep the token ratio upon changing `positionWidth`