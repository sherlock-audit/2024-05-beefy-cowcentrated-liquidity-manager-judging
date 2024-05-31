Bouncy Midnight Mustang

high

# Lost Fund in `StrategyPassiveManagerVelodrome::retireVault()`


## Summary
Lost Fund in `StrategyPassiveManagerVelodrome::retireVault()`
## Vulnerability Detail
```javascript
function retireVault() external onlyOwner {
        if (IBeefyVaultConcLiq(vault).totalSupply() != 10**3) revert NotAuthorized();
        panic(0,0);
        address feeRecipient = beefyFeeRecipient();
        IERC20Metadata(lpToken0).safeTransfer(feeRecipient, IERC20Metadata(lpToken0).balanceOf(address(this)));
        IERC20Metadata(lpToken1).safeTransfer(feeRecipient, IERC20Metadata(lpToken1).balanceOf(address(this)));        
        _transferOwnership(address(0));
    }
```
Missing to transfer Output Token(from gauge) out of this contract. Output Token will be permanently stuck in this contract.
## Impact
Output Token will be permanently stuck in this contract.
## Code Snippet
https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L793
## Tool used

Manual Review

## Recommendation
```diff
function retireVault() external onlyOwner {
        if (IBeefyVaultConcLiq(vault).totalSupply() != 10**3) revert NotAuthorized();
        panic(0,0);
        address feeRecipient = beefyFeeRecipient();
        IERC20Metadata(lpToken0).safeTransfer(feeRecipient, IERC20Metadata(lpToken0).balanceOf(address(this)));
        IERC20Metadata(lpToken1).safeTransfer(feeRecipient, IERC20Metadata(lpToken1).balanceOf(address(this))); 
+        IERC20Metadata(output).safeTransfer(feeRecipient, IERC20Metadata(output).balanceOf(address(this)));        
        _transferOwnership(address(0));
    }
```