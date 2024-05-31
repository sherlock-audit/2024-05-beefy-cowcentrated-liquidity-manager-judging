Curly Burlap Falcon

medium

# `_removeAllowances` will revert on zero Value approvals

## Summary
`_removeAllowances` will revert on zero Value approvals
## Vulnerability Detail
function `_removeAllowances` will revert on zero Value approvals , Some tokens (e.g. BNB) revert when approving a zero value amount (i.e. a call to approve(address, 0)).
## Impact
Dos
## Code Snippet
https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/42ef5f0eac1bc954e888cf5bfb85cbf24c08ec76/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L835C5-L840C6
```solidity
    function _removeAllowances() private {
        IERC20Metadata(output).forceApprove(unirouter, 0);
        IERC20Metadata(output).forceApprove(rewardPool, 0);
        IERC20Metadata(lpToken0).forceApprove(nftManager, 0);
        IERC20Metadata(lpToken1).forceApprove(nftManager, 0);
    }
```
## Tool used

Manual Review

## Recommendation
support BNB or revert if the output token is BNB