Soft Sangria Lion

medium

# StrategyPassiveManagerVelordrome::setRewardPool() doesn't set rewardPool allowance

## Summary

When the `owner` calls `setRewardPool` with a new reward pool, the old approval for the old `rewardPool` is not cleared and the new `rewardPool` is not set an approval. Which means the new reward pool will not have an approval to spend the contract's tokens.

## Vulnerability Detail
[StrategyPassiveManagerVelodrome::setRewardPool()](https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L776-L779)
```solidity
    function setRewardPool(address _rewardPool) external onlyOwner {
        rewardPool = _rewardPool;
        emit SetRewardPool(_rewardPool);
    }
```
[StrategyPassiveManagerVelodrome.sol#L827-L840](https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L827-L840)
```solidity
    function _giveAllowances() private {
        IERC20Metadata(output).forceApprove(unirouter, type(uint256).max);
>>      IERC20Metadata(output).forceApprove(rewardPool, type(uint256).max);
        IERC20Metadata(lpToken0).forceApprove(nftManager, type(uint256).max);
        IERC20Metadata(lpToken1).forceApprove(nftManager, type(uint256).max);
    }

    /// @notice removes swap permisions for the tokens from the unirouter.
    function _removeAllowances() private {
        IERC20Metadata(output).forceApprove(unirouter, 0);
>>      IERC20Metadata(output).forceApprove(rewardPool, 0);
        IERC20Metadata(lpToken0).forceApprove(nftManager, 0);
        IERC20Metadata(lpToken1).forceApprove(nftManager, 0);
    }
```

## Impact

`rewardPool` will be unable to transfer `output` tokens from `StrategyPassiveManagerVelordrom`. The manager could technically use `panic` to clear allowances and then `unpause` however this would cause a lot of users to be worried about the state of the protocol and is clearly not what the function is intended for. 

## Code Snippet

[StrategyPassiveManagerVelodrome::setRewardPool()](https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L776-L779)
[StrategyPassiveManagerVelodrome.sol#L827-L840](https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L827-L840)

## Tool used

Manual Review

## Recommendation

Utilise `_removeAllowances` and `_giveAllowances` just like in `setUnirouter`:
```solidity
    function setUnirouter(address _unirouter) external override onlyOwner {
        _removeAllowances();
        unirouter = _unirouter;
        _giveAllowances();
        emit SetUnirouter(_unirouter);
    }
```