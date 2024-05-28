Upbeat Boysenberry Seal

high

# Incorrect Assignment in `StrategyPassiveManagerVelodrome::_setAltTick` Function

## Summary
The `_setAltTick` function in the `StrategyPassiveManagerVelodrome` contract has incorrect assignments for `positionAlt.tickLower` and `positionAlt.tickUpper` inside the `if` and `else if` blocks.

## Vulnerability Detail

When we call the `_setTicks` function, it sets the tick positions for the main and alternative positions.

```solidity
    function _setTicks() private onlyCalmPeriods {
        int24 tick = currentTick();
        int24 distance = _tickDistance();
        int24 width = positionWidth * distance;

        _setMainTick(tick, distance, width);
@>>        _setAltTick(tick, distance, width);
    }
```

The assignments for `positionAlt.tickLower` and `positionAlt.tickUpper` in the `_setAltTick` function are incorrectly paired within the `if` and `else if` blocks. The function calls `TickUtils.baseTicks`, which returns `(int24 tickLower, int24 tickUpper)`. However, these values are incorrectly assigned to `positionAlt.tickLower` and `positionAlt.tickUpper` in the current implementation.

```solidity
    function _setAltTick(int24 tick, int24 distance, int24 width) private {
        (uint256 bal0, uint256 bal1) = balancesOfThis();

        // We calculate how much token0 we have in the price of token1.
        uint256 amount0;

        if (bal0 > 0) {
            amount0 = bal0 * price() / PRECISION;
        }

        // We set the alternative position based on the token that has the most value available.
        if (amount0 < bal1) {
@>>            (positionAlt.tickLower, ) = TickUtils.baseTicks(
                tick,
                width,
                distance
            );

@>>            (positionAlt.tickUpper, ) = TickUtils.baseTicks(
                tick,
                distance,
                distance
            );
        } else if (bal1 < amount0) {
@>>            (, positionAlt.tickLower) = TickUtils.baseTicks(
                tick,
                distance,
                distance
            );

@>>            (, positionAlt.tickUpper) = TickUtils.baseTicks(
                tick,
                width,
                distance
            );
        }
    }
```

```solidity
    function baseTicks(
        int24 currentTick,
        int24 baseThreshold,
        int24 tickSpacing
@>>    ) internal pure returns (int24 tickLower, int24 tickUpper) {
        int24 tickFloor = floor(currentTick, tickSpacing);

        tickLower = tickFloor - baseThreshold;
        tickUpper = tickFloor + baseThreshold;
    }
```

## Impact
The incorrect assignments for `positionAlt.tickLower` and `positionAlt.tickUpper` may result in improper setting of alternative positions, leads to an imbalance in the alternative position's tick range. This means that the `StrategyPassiveManagerVelodrome` might end up with more liquidity on one side of the alternative position than the other.

If a large price move occurs and the alternative position is on the wrong side, it could lead to losses that otherwise may have been gains.

## Code Snippet
https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L649-L686

https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/utils/TickUtils.sol#L19-L28

## Tool used

Manual Review

## Recommendation
Swap the assignments for `positionAlt.tickLower` and `positionAlt.tickUpper` in the `if` and `else if` blocks to maintain consistency with the alternative tick setting logic.

```diff
    function _setAltTick(int24 tick, int24 distance, int24 width) private {
        (uint256 bal0, uint256 bal1) = balancesOfThis();

        // We calculate how much token0 we have in the price of token1.
        uint256 amount0;

        if (bal0 > 0) {
            amount0 = bal0 * price() / PRECISION;
        }

        // We set the alternative position based on the token that has the most value available.
        if (amount0 < bal1) {
            (positionAlt.tickLower, ) = TickUtils.baseTicks(
                tick,
                width,
                distance
            );
+            ( ,positionAlt.tickUpper) = TickUtils.baseTicks(
-            (positionAlt.tickUpper, ) = TickUtils.baseTicks(
                tick,
                distance,
                distance
            );
        } else if (bal1 < amount0) {
+            (positionAlt.tickLower , ) = TickUtils.baseTicks(
-            (, positionAlt.tickLower) = TickUtils.baseTicks(
                tick,
                distance,
                distance
            );

            (, positionAlt.tickUpper) = TickUtils.baseTicks(
                tick,
                width,
                distance
            );
        }
    }
```
