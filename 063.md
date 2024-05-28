Soft Sangria Lion

medium

# StrategyPassiveManagerVelodrome::withdraw does not call _setTicks before re-adding liqudity, which can lead to reduced LP fees

## Summary

`StrategyPassiveManagerVelodrome::withdraw` does not call `_setTicks` before re-adding liqudity, which can lead to reduced LP fees.

## Vulnerability Detail
[StrategyPassiveManagerVelodrome::withdraw()](https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L226-L240)
```solidity
    function withdraw(uint256 _amount0, uint256 _amount1) external {
        _onlyVault();

        // Liquidity has already been removed in beforeAction() so this is just a simple withdraw.
        if (_amount0 > 0) IERC20Metadata(lpToken0).safeTransfer(vault, _amount0);
        if (_amount1 > 0) IERC20Metadata(lpToken1).safeTransfer(vault, _amount1);

        // After we take what is needed we add it all back to our positions. 
        if (!_isPaused()) _addLiquidity();

        (uint256 bal0, uint256 bal1) = balances();

        // TVL Balances after withdraw
        emit TVL(bal0, bal1);
    }
```
During the withdraw function flow, `_addLiquidity()` is called without calling `_setTicks()`. This means that the `main` and `alt` position `ticks` will not have been updated before the `addLiquidity()` call.

## Impact

Not calling `_setTicks()` before `_addLiquidity()` may lead to liquidity being deployed in an old tick range, which will cause the added liquidity to not earn any LP fees as the price has move out of the currently set ticks. This would lead to a loss of yield for users and protocol.

## Code Snippet

[StrategyPassiveManagerVelodrome::withdraw()](https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L226-L240)

## Tool used

Manual Review

## Recommendation

Consider calling `_setTicks()` before `_addLiquidity()` within the `withdraw` function. It may be necessary to check for calm period to ensure liquidity is not deployed in a manipulated tick range though.
