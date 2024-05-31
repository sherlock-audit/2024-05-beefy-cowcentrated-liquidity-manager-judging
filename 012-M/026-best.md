Flaky Goldenrod Chinchilla

medium

# `retireVault` function can fail on zero amount transfer if lpToken balance is 0


## Summary
Both lpToken0 & lpToken1 amount can be zero, when calling `retireVault`.
Some ERC20 tokens do not allow zero value transfers, so it will be reverting.
In this case, it's not possible to retire the vault.

## Vulnerability Detail

## Impact
Some tokens may prevent transfers of zero value.
If either token0 or token1 has this behavior (reverts on zero transfer), then users won't be able to decrease their liquidity position.

## Code Snippet
https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L797
```solidity
    function retireVault() external onlyOwner {
        if (IBeefyVaultConcLiq(vault).totalSupply() != 10**3) revert NotAuthorized();
        panic(0,0);
        address feeRecipient = beefyFeeRecipient();
        IERC20Metadata(lpToken0).safeTransfer(feeRecipient, IERC20Metadata(lpToken0).balanceOf(address(this))); // @audit some token do not allow zero value transfer
        IERC20Metadata(lpToken1).safeTransfer(feeRecipient, IERC20Metadata(lpToken1).balanceOf(address(this)));
        _transferOwnership(address(0));
    }
```

## Tool used

Manual Review

## Recommendation
Consider checking the balance before transferring.

```diff
    function retireVault() external onlyOwner {
        if (IBeefyVaultConcLiq(vault).totalSupply() != 10**3) revert NotAuthorized();
        panic(0,0);
        address feeRecipient = beefyFeeRecipient();
+        if (IERC20Metadata(lpToken0).balanceOf(address(this)) > 0)
        IERC20Metadata(lpToken0).safeTransfer(feeRecipient, IERC20Metadata(lpToken0).balanceOf(address(this)));
+        if (IERC20Metadata(lpToken1).balanceOf(address(this)) > 0)
        IERC20Metadata(lpToken1).safeTransfer(feeRecipient, IERC20Metadata(lpToken1).balanceOf(address(this)));
        _transferOwnership(address(0));
    }
```