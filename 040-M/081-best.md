Late Cherry Viper

medium

# _getTokensRequired calculation is wrong when the current price tick is out of the current position range, causing loss of potential earnings

## Summary
The calculation in _getTokensRequired (which calculates how much to take of the depositor's amount0 and amount1 for optimized utility) produces an incorrect result in the case that the price is out of the main position range and rebalance hasn't been called yet, causing inefficient utilization of liquidity. 

## Vulnerability Detail

### background information
When a user deposits to the ConcLiq vault, the order of operations is:
1. First, uniswap fees are claimed and all liquidity is removed from the Uniswap pool:
```solidity
function beforeAction() external {
    _onlyVault();
    _claimEarnings();
    _removeLiquidity();
}
```

2. Then _getTokensRequired is called to find out the ideal amount to take from the depositor's offered amount0 and amount1. This calculation considers the current balances in the strategy (after they've been removed from the pool) and the current price, and tries to utilize the user's amount0 and amount1 to the maximum, while getting the balances ratio to be as close to the current price ratio (so that when liquidity is added back to the main position, a maximum portion of the balances is utilized).
```solidity
function _getTokensRequired(uint256 _price, uint256 _amount0, uint256 _amount1, uint256 _bal0, uint256 _bal1) private pure returns (uint256 depositAmount0, uint256 depositAmount1) {
    // get the amount of bal0 that is equivalent to bal1
    if (_bal0 == 0 && _bal1 == 0) return (_amount0, _amount1);

    uint256 bal0InBal1 = (_bal0 * _price) / PRECISION;

    // check which side is lower and supply as much as possible
    if (_bal1 < bal0InBal1) {
        uint256 finalBalanceForAmount1 = _bal1 + _amount1;
        uint256 owedAmount0 = finalBalanceForAmount1 > bal0InBal1 
            ? (finalBalanceForAmount1 - bal0InBal1) * PRECISION / _price 
            : 0;
        if (owedAmount0 > _amount0) {
            depositAmount0 = _amount0;
            depositAmount1 = _amount1 - ( (owedAmount0 - _amount0) * _price / PRECISION );
        } else {
            depositAmount0 = owedAmount0;
            depositAmount1 = _amount1;
        }
    } else {
        uint256 finalBalanceForAmount0 = bal0InBal1 + ( _amount0 * _price / PRECISION );
        uint256 owedAmount1 = finalBalanceForAmount0 > _bal1 
            ? finalBalanceForAmount0 - _bal1
            : 0;
        if (owedAmount1 > _amount1) {
            depositAmount0 = _amount0 - ( (owedAmount1 - _amount1) * PRECISION / _price );
            depositAmount1 = _amount1;
        } else {
            depositAmount0 = _amount0;
            depositAmount1 = owedAmount1;
        }
    }
}
```  
3. After the calculated amount0 and amount1 have been transfered from the depositor to the StrategyPassiveManagerUniswap, the StrategyPassiveManagerUniswap *deposit* function is called. This function calls setTicks (which updates the position ticks) **only if the ticks were never initialized before**, and then calls _addLiquidity.  

4. The _addLiquidity function first tries to add as much of the balances to the current main position range and the rest to the alt postion, but if the current price is out of the main position range (checked by _checkAmounts) the main position is skipped and all liquidity is added only to the alt position
```solidity
bool amountsOk = _checkAmounts(liquidity, positionMain.tickLower, positionMain.tickUpper);

// Flip minting to true and call the pool to mint the liquidity. 
if (liquidity > 0 && amountsOk) {
    minting = true;
    IUniswapV3Pool(pool).mint(address(this), positionMain.tickLower, positionMain.tickUpper, liquidity, "Beefy Main");
}
```

5. Finally, the amount of shares due to the depositor is calculated and shares are minted for the depositor.

The problem is that if the current price is out of the main position range, and rebalance hasn't been called yet, the deposit process described above produces the oppsite of the desired result: is uses only one of the tokens the depositor offers, but it's exactly the token that currently cannot be added as liquidity to the position ranges (i.e. token0 if the curr price is above the range and token1 if the price is below the range.)

### vulnerability scenario
consider the following scenario (numbers are simplified to make the example easier to understand):  

A. The current price tick is 100, position width is 20, main position range is 80-120 and alt position range is 80-99, the strategy currently holds 1000 liquidity units in the pool.  
B. The price tick moves up to tick 130 (rebalance is not called yet)  
C. A user tries to deposit to the vault where amount0 = 1000 and amount1 = 1000 (assume the price at tick 130 is 1 token0 = 1 token1)  
D. When liquidity is removed from the pool (step 1), it is all in token1 (since now the position ranges are both below the price tick)  
E. When _getTokensRequired is called (step 2), it will determine that only amount0 of token0 should be used (since it tries to balance token0/token1 reserves to the current price ratio with is 1/1)  
F. When liquidity is re-added to the pool (step 4) the main position will be skipped (see step 4 above) and liquidity will be added only to the alt position.  
G. Since the alt position range is also entirely to the left (below) the current price tick, only token1 will be added (the same amount that was in the pool before) and the amount0 of token0 taken from the depositor will remain outside the pool (on the strategy contract's balance).  
H. The result is that the deposit process will use the depositor's amount0 (which will not be added to the position as liquidity since below-current-price ranges can only accept token1) instead of amount1 (which would have been added to the position as liquidity).   
I. The price now drops back within the range of the current position, however the depositor's liquidity stays out of the pool and is not utilized.

### root cause
The root cause for this vulnerability is the fact that the _getTokensRequired function assumes that the current price is within the current position range. If this assumption is broken (as in the case above) it produces a faulty result that takes from the depositor a token that can not be currently used as liquidity instead of one that can.

## Impact
The above scenario causes loss to existing depositors because when the price slides back to the current main (or alt) position range, the new depositor's liquidity will not be utilized (it stays on the strategy's balance instead of entering the pool liquidity), however their shares will grant them a share of the profits. This situation will only be remedied the next time rebalance is called (however if the price slides back to range, there will be no apparent need to call rebalance and the probelm will persist).

## Code Snippet
https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/vault/BeefyVaultConcLiq.sol#L145

https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/strategies/uniswap/StrategyPassiveManagerUniswap.sol#L229

## Tool used

Manual Review

## Recommendation
There are two possible solutions:
1. In _getTokensRequired, check first if the current price is within the current (main) position range, and if not, use only the token that the current position accepts (token0 if price is below the range or token1 if price is above the range)
2. For every deposit, call _setTicks before processing the deposit to enforce the current position range to include the current price.