Daring Flint Aphid

high

# Incorrect Solidity version in `FullMath.sol` can cause permanent freezing of assets for arithmetic underflow-induced revert Vulnerability detail

## Summary
An incorrect Solidity version in FullMath.sol causes arithmetic underflows in StrategyPassiveManagerVelodrome.sol. This results in perpetual reverts during deposit and withdraw operations, potentially locking assets permanently if price conditions remain unfavorable. 

## Vulnerability Detail
`FullMath.sol` library, [imported from UniswapV3](https://github.com/Uniswap/v3-core/blob/main/contracts/libraries/FullMath.sol) was [altered to build with solidity v0.8.x](https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/42ef5f0eac1bc954e888cf5bfb85cbf24c08ec76/cowcentrated-contracts/contracts/utils/FullMath.sol#L2), which has under/overflow protection; the library, however, makes use of these by design, so it won't work properly when compiled in v0.8.0 or later.

_Now coming to the actual vulnerability:_

`StrategyPassiveManagerVelodrome.sol` makes use of the `LiquidityAmounts.getAmountsForLiquidity` helper function in its `_addLiquidity` `_checkAmounts` and `balancesOfPool` functions to convert velodrome pool liquidity into estimated underlying token amounts.

This function `getAmountsForLiquidity` will trigger an arithmetic underflow whenever `sqrtRatioX96` is smaller than `sqrtRatioAX96`, causing these functions to revert until this ratio comes back in range and the math no longer overflows.

_Such oracle price conditions are not only possible but also likely to happen in real market conditions, and they can be permanent (i.e. one asset permanently appreciating over the other one)._

Moving up the stack, if LiquidityAmounts.getAmountsForLiquidity  reverts, then `_addLiquidity` `_checkAmounts` and `balancesOfPool` functions can revert. In particular, the `_addLiquidity`  is called whenever a call to [deposit](https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/42ef5f0eac1bc954e888cf5bfb85cbf24c08ec76/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L213), [withdraw](https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/42ef5f0eac1bc954e888cf5bfb85cbf24c08ec76/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L234) and [moveTicks](https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/42ef5f0eac1bc954e888cf5bfb85cbf24c08ec76/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L419) is made.

## Impact
When the fair exchange price of the pool falls outside the range (higher side), the deposit and withdraw will always revert, locking the underlying assets in the pool until the price swings to a different value that does not trigger an under/overflow. If the oracle price stays within this range indefinitely, the funds are permanently locked.

## Code Snippet
https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/42ef5f0eac1bc954e888cf5bfb85cbf24c08ec76/cowcentrated-contracts/contracts/utils/FullMath.sol#L2
https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/42ef5f0eac1bc954e888cf5bfb85cbf24c08ec76/cowcentrated-contracts/contracts/utils/LiquidityAmounts.sol#L70
https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/42ef5f0eac1bc954e888cf5bfb85cbf24c08ec76/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L263
https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/42ef5f0eac1bc954e888cf5bfb85cbf24c08ec76/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L213
## Tool used
Manual Review

## Recommendation
Restore the [original FullMath.sol](https://github.com/Uniswap/v3-core/blob/main/contracts/libraries/FullMath.sol) library so it compiles with solc versions earlier than 0.8.0.