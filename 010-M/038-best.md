Polished Lemonade Chinchilla

medium

# Alt ticks will not be set if `bal1 == amount0`

## Summary
Alt ticks will not be set if `bal1 == amount0` 

## Vulnerability Detail
Let's look at the code of `_setAltTicks`

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
            (positionAlt.tickLower, ) = TickUtils.baseTicks(
                tick,
                width,
                distance
            );

            (positionAlt.tickUpper, ) = TickUtils.baseTicks(
                tick,
                distance,
                distance
            ); 
        } else if (bal1 < amount0) {
            (, positionAlt.tickLower) = TickUtils.baseTicks(
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

As we can see, we have a check if `amount0 < bal1` and if `bal1 < amount0`. However, if the two balances are equal, alt ticks will not be set.

This would cause old ticks to once again be used after rebalancing and liquidity being put in wrong/ out-of-range ticks.


## Impact
`AltTicks` will be wrong, resulting in worse liquidity management and ultimately less accrued fees for protocol users

## Code Snippet
https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L672

## Tool used

Manual Review

## Recommendation
Change the `else if` to `else`