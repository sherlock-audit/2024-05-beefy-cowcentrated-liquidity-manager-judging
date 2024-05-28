Square Charcoal Toad

medium

# Manipulability of Tick Data in Velodrome Pool Leading to Vulnerabilities in Volatility Check Mechanism

## Summary

The Tick value, which is the latest Tick data from the Velodrome pool, can be easily manipulated by anyone. This can be achieved by swapping tokens before the Tick value is read by the Strategy and swapping back immediately after.

## Vulnerability Detail

The function `StrategyPassiveManagerVelodrome::currentTick()` in the contract can be easily manipulated by anyone. This manipulation is possible because the function reads the current Tick from the pool, which can be temporarily altered through token swaps.

## Impact

This vulnerability impacts the `StrategyPassiveManagerVelodrome::isCalm` function, which is used to check if the market is not too volatile to prevent users from experiencing direct Impermanent Loss.

## Code Snippet

- [isCalm function](https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/42ef5f0eac1bc954e888cf5bfb85cbf24c08ec76/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L133-L143)
- [currentTick function](https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/42ef5f0eac1bc954e888cf5bfb85cbf24c08ec76/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L600-L602)

## Tool used

Manual Review

## Recommendation

Use an Oracle to verify the price or compare two TWAPs (Time-Weighted Average Prices) with different durations to determine if there are significant price changes.
