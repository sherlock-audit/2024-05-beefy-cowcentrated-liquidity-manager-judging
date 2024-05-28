Virtual Boysenberry Bear

high

# The `_addLiquidity` function utilizes a cached sqrtPrice

## Summary
The `deposit, withdraw, moveTicks, setPositionWidth, and unpause` functions all call the internal `_addLiquidity` function, which adds liquidity to the main and alternative positions. The problem lies in the fact that the function uses the cached `sqrtPriceX96` from `slot0` to calculate how much liquidity is obtained by adding ticks to the main position with our token balances and for the alternative position.

## Vulnerability Detail
1)The user calls the `deposit` function in the `Vault`, which in turn calls the `deposit` function in `StrategyPassiveManagerVelodrome`.
2)Inside the `_addLiquidity()` function, we determine the amount of liquidity gained by adding ticks to the main position based on our token balances and mint position:
```solidity
function _addLiquidity() private {
        _whenStrategyNotPaused();

        (uint256 bal0, uint256 bal1) = balancesOfThis();

        int24 mainLower = positionMain.tickLower;
        int24 mainUpper = positionMain.tickUpper;
        int24 altLower = positionAlt.tickLower;
        int24 altUpper = positionAlt.tickUpper;

        // Then we fetch how much liquidity we get for adding at the main position ticks with our token balances. 
-->     uint160 sqrtprice = sqrtPrice();
        uint128 liquidity = LiquidityAmounts.getLiquidityForAmounts(
-->         sqrtprice,
            TickMath.getSqrtRatioAtTick(mainLower),
            TickMath.getSqrtRatioAtTick(mainUpper),
            bal0,
            bal1
        );

        (uint256 amount0, uint256 amount1) = LiquidityAmounts.getAmountsForLiquidity(
-->         sqrtprice,
            TickMath.getSqrtRatioAtTick(mainLower),
            TickMath.getSqrtRatioAtTick(mainUpper),
            liquidity
        );

        bool amountsOk = _checkAmounts(liquidity, mainLower, mainUpper);

        // Mint or add liquidity to the position. 
        if (liquidity > 0 && amountsOk) {
            _mintPosition(mainLower, mainUpper, amount0, amount1, true);
        }
        ///code
 }       
```
3)Next, we incorrectly determine the amount of liquidity gained by adding ticks to the alternative position based on our token balances, using the cached `sqrtPrice`. 
After we modify the `main` position, the `sqrtPriceX96` changes, but we continue to use it in calculations for the `alternative` position, which is incorrect:
```solidity
// Fetch how much liquidity we get for adding at the alternative position ticks with our token balances.
        liquidity = LiquidityAmounts.getLiquidityForAmounts(
-->         sqrtprice,
            TickMath.getSqrtRatioAtTick(altLower),
            TickMath.getSqrtRatioAtTick(altUpper),
            bal0,
            bal1
        );

        (amount0, amount1) = LiquidityAmounts.getAmountsForLiquidity(
-->         sqrtprice,
            TickMath.getSqrtRatioAtTick(altLower),
            TickMath.getSqrtRatioAtTick(altUpper),
            liquidity
        );

        // Mint or add liquidity to the position.
        if (liquidity > 0 && (amount0 > 0 || amount1 > 0)) {
            _mintPosition(altLower, altUpper, amount0, amount1, false);
        }
```

## Impact
The `_addLiquidity` function uses the cached `sqrtPriceX96` value for both the main and alternative positions, which will affect the calculations of `AmountsForLiquidity` for the alternative position after minting the main position, which changes the `sqrtPriceX96` value.

## Code Snippet
[contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L254-L293](https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L254-L293)

## Tool used

Manual Review

## Recommendation
Consider using the new `sqrtPriceX96` value:
```diff
         (bal0, bal1) = balancesOfThis();
+        sqrtprice = sqrtPrice();

        // Fetch how much liquidity we get for adding at the alternative position ticks with our token balances.
        liquidity = LiquidityAmounts.getLiquidityForAmounts(
            sqrtprice,
            TickMath.getSqrtRatioAtTick(altLower),
            TickMath.getSqrtRatioAtTick(altUpper),
            bal0,
            bal1
        );

        (amount0, amount1) = LiquidityAmounts.getAmountsForLiquidity(
            sqrtprice,
            TickMath.getSqrtRatioAtTick(altLower),
            TickMath.getSqrtRatioAtTick(altUpper),
            liquidity
        );

        // Mint or add liquidity to the position.
        if (liquidity > 0 && (amount0 > 0 || amount1 > 0)) {
            _mintPosition(altLower, altUpper, amount0, amount1, false);
        }
```
