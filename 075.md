Bouncy Midnight Mustang

high

# The balancesOfPool() function has an internal calculation error.


## Summary
The balancesOfPool() function has an internal calculation error.
## Vulnerability Detail
```javascript
 function balancesOfPool() public view returns (uint256 token0Bal, uint256 token1Bal, uint256 mainAmount0, uint256 mainAmount1, uint256 altAmount0, uint256 altAmount1) {
        uint160 sqrtPriceX96 = sqrtPrice();

        uint128 liquidity; 
        uint128 altLiquidity;
        uint256 owed0;
        uint256 owed1;
        uint256 altOwed0;
        uint256 altOwed1;
        if (positionMain.nftId != 0) (,,,,,,,liquidity,,,owed0, owed1) = INftPositionManager(nftManager).positions(positionMain.nftId);
        if (positionAlt.nftId != 0) (,,,,,,, altLiquidity,,,altOwed0, altOwed1) = INftPositionManager(nftManager).positions(positionAlt.nftId);

        (mainAmount0, mainAmount1) = LiquidityAmounts.getAmountsForLiquidity(
            sqrtPriceX96,
            TickMath.getSqrtRatioAtTick(positionMain.tickLower),
            TickMath.getSqrtRatioAtTick(positionMain.tickUpper),
            liquidity
        );

        (altAmount0, altAmount1) = LiquidityAmounts.getAmountsForLiquidity(
            sqrtPriceX96,   
            TickMath.getSqrtRatioAtTick(positionAlt.tickLower),
            TickMath.getSqrtRatioAtTick(positionAlt.tickUpper),
            altLiquidity
        );

@>        mainAmount0 += owed0;
@>        mainAmount1 += owed1;

@>        altAmount0 += altOwed0;
@>        altAmount1 += altOwed1;
        
        token0Bal = mainAmount0 + altAmount0;
        token1Bal = mainAmount1 + altAmount1;
    }
```
owed0 and owed1 are fee0 and fee1 that are not collected. But In velodrome, a pool does not pay trading fees but instead provide rewards in other tokens not included in the pool.  
owed0 amount of lpToken0 and owed1 amount of lpToken1 will not be given to this contract StrategyPassiveManagerVelodrome. Therefore, it should not be considered as part of the value of the contract. Especially since the output token will be transferred out of the contract, rather than being converted into lpToken0 and lpToken1 and added to the liquidity pool.
```javascript
function balances() public view returns (uint256 token0Bal, uint256 token1Bal) {
        (uint256 thisBal0, uint256 thisBal1) = balancesOfThis();
        (uint256 poolBal0, uint256 poolBal1,,,,) = balancesOfPool();

        uint256 total0 = thisBal0 + poolBal0;
        uint256 total1 = thisBal1 + poolBal1;

        // For token0 and token1 we return balance of this contract + balance of positions - feesUnharvested.
        return (total0, total1);
    }

```
`balances()` calls `balancesOfPool()`. And other contracts such as (vault) use balances() as the basis for the entire value, leading to calculation errors.
```javascript
function balances() public view returns (uint amount0, uint amount1) {
        (amount0, amount1) = IStrategyConcLiq(strategy).balances();
    }
```
## Impact
 other contracts such as (vault) use balances() as the basis for the entire value, leading to calculation errors.
## Code Snippet
https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L549C1-L583C6
https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L520C1-L529C6
## Tool used

Manual Review

## Recommendation
```diff
-       mainAmount0 += owed0;
-       mainAmount1 += owed1;

-       altAmount0 += altOwed0;
-       altAmount1 += altOwed1;
```