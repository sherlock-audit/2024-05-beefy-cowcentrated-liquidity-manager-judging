Future Carmine Giraffe

high

# Accumulation of Out Of Range Impermanent Loss by LPs because `StrategyPassiveManagerVelodrome::withdraw()` allows the deposit of liquidity in uncalm period

## Summary

## Vulnerability Detail
Deposits by `StrategyPassiveManagerVelodrome` into the Concentrated Liquidity (CL) pools is predicated on two things. First is that the direct call to make the deposit has to come from the vault contract and the second is that deposits are only allowed when the current tick is within acceptable deviation. The purpose of the latter is to ensure that price sits within a reasonable margin of the TWAP so users don't lose their funds in period with high volatility, whether artifically (flash loans) or not.

```solidity
    function isCalm() public view returns (bool) {
        int24 tick = currentTick();
        int56 twapTick = twap();

        int56 minCalmTick = int56(SignedMath.max(twapTick - maxTickDeviation, MIN_TICK)); 
        int56 maxCalmTick = int56(SignedMath.min(twapTick + maxTickDeviation, MAX_TICK));

        if(minCalmTick > tick  || maxCalmTick < tick) return false;
        else return true;
    }
```

The importance of this function embedded in the `onlyCalmPeriods()` modifier can be seen in many functions including important ones like `StrategyPassiveManagerVelodrome::moveTicks()` which is a function that the rebalancer calls when the ticks for the positions are to be adjusted as this function is also subject to this requirement.

However, in [`StrategyPassiveManagerVelodrome::withdraw()`](https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L226C1-L240C6), deposits of tokens into the CLPool in uncertain and uncalm periods still occurs, which antagonizes what onlyCalmPeriod modifier ought to protect LPs from.

```solidity
    function withdraw(uint256 _amount0, uint256 _amount1) external {
        _onlyVault();

        if (_amount0 > 0) IERC20Metadata(lpToken0).safeTransfer(vault, _amount0);
        if (_amount1 > 0) IERC20Metadata(lpToken1).safeTransfer(vault, _amount1);

>>>     if (!_isPaused()) _addLiquidity();

        (uint256 bal0, uint256 bal1) = balances();

        emit TVL(bal0, bal1);
    }
```

## Impact

Loss of Funds to LPs due to Out of Range Impermanent Loss. When in uncalm period, the tick cannot be adjusted by the rebalancer as established earlier. Well, when the tokens will be redeposited in `_addLiquidity()`, the deposits would be made to the previous range which the price has either gone beyond or below. When this happens, the LPs funds will be sitting in the CLPool contract with no utilization and no trade fees.

Also, if the shift in ticks causing the uncalm period is artificial, the loss is seriously magnified if the price will not revert back to range anytime soon.

## Code Snippet

```solidity
    function withdraw(uint256 _amount0, uint256 _amount1) external {
        _onlyVault();

        if (_amount0 > 0) IERC20Metadata(lpToken0).safeTransfer(vault, _amount0);
        if (_amount1 > 0) IERC20Metadata(lpToken1).safeTransfer(vault, _amount1);

>>>     if (!_isPaused()) _addLiquidity();

        (uint256 bal0, uint256 bal1) = balances();

        emit TVL(bal0, bal1);
    }
```

## Tool used

Manual Review

## Recommendation

```diff
    function withdraw(uint256 _amount0, uint256 _amount1) external {
        _onlyVault();

        if (_amount0 > 0) IERC20Metadata(lpToken0).safeTransfer(vault, _amount0);
        if (_amount1 > 0) IERC20Metadata(lpToken1).safeTransfer(vault, _amount1);

-       if (!_isPaused()) _addLiquidity();
+       if (!_isPaused() && isCalm()) _addLiquidity();

        (uint256 bal0, uint256 bal1) = balances();

        emit TVL(bal0, bal1);
    }
```