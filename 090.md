Eager Raspberry Orca

medium

# Incorrect amounts checking causes liquidity to be added to the wrong position.

## Summary
The check for amounts in `_checkAmounts` is incorrect, causing the liquidity to be added to the "alt" position when it should be added to the "main" position.

## Vulnerability Detail
In `_addLiquidity`, non zero liquidity and amounts are required to mint for "main" position, otherwise the "alt" position is minted.
```solidity
Function: _addLiquidity

270:@>      bool amountsOk = _checkAmounts(liquidity, mainLower, mainUpper);
271:
272:        // Mint or add liquidity to the position. 
273:@>      if (liquidity > 0 && amountsOk) {
274:@>          _mintPosition(mainLower, mainUpper, amount0, amount1, true);
275:        }
```
https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L270-L275

Let's dive into `_checkAmounts`. It requires that neither `amount0` nor `amount1` is zero. But according to `LiquidityAmounts.getAmountsForLiquidity`, when the `sqrtPrice` is not within the price range between `_tickLower` and `_tickUpper`, the result `amount0` or `amount1` will be zero. Therefore, it is valid for `amount0` or `amount1` to be zero, and the checking in L410 should be `if (amount0 == 0 && amount1 == 0) return false;`.
```solidity
402:    function _checkAmounts(uint128 _liquidity, int24 _tickLower, int24 _tickUpper) private view returns 40:(bool) {
403:        (uint256 amount0, uint256 amount1) = LiquidityAmounts.getAmountsForLiquidity(
404:            sqrtPrice(),
405:            TickMath.getSqrtRatioAtTick(_tickLower),
406:            TickMath.getSqrtRatioAtTick(_tickUpper),
407:            _liquidity
408:        );
409:
410:@>      if (amount0 == 0 || amount1 == 0) return false;
411:        else return true;
412:    }
```
https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L402-L412

```solidity
146:    function getAmountsForLiquidity(
147:        uint160 sqrtRatioX96,
148:        uint160 sqrtRatioAX96,
149:        uint160 sqrtRatioBX96,
150:        uint128 liquidity
151:    ) internal pure returns (uint256 amount0, uint256 amount1) {
152:        if (sqrtRatioAX96 > sqrtRatioBX96)
153:            (sqrtRatioAX96, sqrtRatioBX96) = (sqrtRatioBX96, sqrtRatioAX96);
154:
155:@>      if (sqrtRatioX96 <= sqrtRatioAX96) {
156:            amount0 = getAmount0ForLiquidity(
157:                sqrtRatioAX96,
158:                sqrtRatioBX96,
159:                liquidity
160:            );
161:        } else if (sqrtRatioX96 < sqrtRatioBX96) {
162:            amount0 = getAmount0ForLiquidity(
163:                sqrtRatioX96,
164:                sqrtRatioBX96,
165:                liquidity
166:            );
167:            amount1 = getAmount1ForLiquidity(
168:                sqrtRatioAX96,
169:                sqrtRatioX96,
170:                liquidity
171:            );
172:@>      } else {
173:            amount1 = getAmount1ForLiquidity(
174:                sqrtRatioAX96,
175:                sqrtRatioBX96,
176:                liquidity
177:            );
178:        }
179:    }
```
https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/utils/LiquidityAmounts.sol#L146-L179


## Impact
If `amount0` or `amount1` is zero, liquidity will be added to the "alt" position, but should be added to the "main" position.

## Code Snippet
https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L270-L275

https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L402-L412

https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/utils/LiquidityAmounts.sol#L146-L179

## Tool used

Manual Review

## Recommendation
Change the amounts checking as follows:
```solidity
    function _checkAmounts(uint128 _liquidity, int24 _tickLower, int24 _tickUpper) private view returns 40:(bool) {
        (uint256 amount0, uint256 amount1) = LiquidityAmounts.getAmountsForLiquidity(
            sqrtPrice(),
            TickMath.getSqrtRatioAtTick(_tickLower),
            TickMath.getSqrtRatioAtTick(_tickUpper),
            _liquidity
        );

-       if (amount0 == 0 || amount1 == 0) return false;
+       if (amount0 == 0 && amount1 == 0) return false;
        else return true;
    }
```
