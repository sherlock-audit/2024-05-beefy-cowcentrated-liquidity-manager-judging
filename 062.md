Mythical Tangerine Peacock

high

# The `getAmountsForLiquidity` function in LiquidityAmounts.sol is not implemented correctly

## Summary
The function `getAmountsForLiquidity()` is used to compute the quantity of `token0` and `token1` value to add to the position given a amount of liquidity. These quantities depend on the amount of liquidity, the current pool prices and the prices at the tick boundaries. 
Actually, `getAmountsForLiquidity()` uses the sqrt prices instead of the ticks, but it doesn't matter because they are equivalent since each tick represents a sqrt price.

Albeit, 3 cases exist:

- The current tick is outside the range from the left, this means only token0 should be added.
- The current tick is within the range, this means both token0 and token1 should be added.
- The current tick is outside the range from the right, this means only token1 should be added.

## Vulnerability Detail
The issue on the implementation is on the first case, which is coded as follows:

```solidity
 if (sqrtRatioX96 <= sqrtRatioAX96) {
            amount0 = getAmount0ForLiquidity(
                sqrtRatioAX96,
                sqrtRatioBX96,
                liquidity
            );
```

As seen above, the implementation indicates that if the current price is equal to the price of the lower tick, it means that it is outside of the range and hence only token0 should be added to the position.

But for the [UniswapV3 implementation](https://github.com/Uniswap/v3-core/blob/d8b1c635c275d2a9450bd6a78f3fa2484fef73eb/contracts/UniswapV3Pool.sol#L328-L336), the current price must be lower in order to consider it outside:

```solidity
  if (_slot0.tick < params.tickLower) {
                // current tick is below the passed range; liquidity can only become in range by crossing from left to
                // right, when we'll need _more_ token0 (it's becoming more valuable) so user must provide it
                amount0 = SqrtPriceMath.getAmount0Delta(
                    TickMath.getSqrtRatioAtTick(params.tickLower),
                    TickMath.getSqrtRatioAtTick(params.tickUpper),
                    params.liquidityDelta
                );
            } 
```

In the discord channel, one of the members of the dev team said the libraries are from `uniswap`:

![IMG_0835](https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager-Rhaydden/assets/154236469/e47dd30f-6bf6-48e0-a86d-54cac7c7a936)




## Impact
When the current price is equal to the left boundary of the range, the uniswap pool will request both `token0` and `token1`, but Beefy will only request from the user `token0` so the pool will lose some `token1` if it has enough to cover it.

## Code Snippet
https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/utils/LiquidityAmounts.sol#L155-L160

## Tool used

Manual Review

## Recommendation
```diff
- if (sqrtRatioX96 <= sqrtRatioAX96) {
+ if (sqrtRatioX96 < sqrtRatioAX96) {
            amount0 = getAmount0ForLiquidity(
                sqrtRatioAX96,
                sqrtRatioBX96,
                liquidity
            );
```