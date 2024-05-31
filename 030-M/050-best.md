Curly Burlap Falcon

medium

# token like UNI, COMP will not work with this protocol

## Summary
token like UNI, COMP will not work with this protocol .
## Vulnerability Detail
Some tokens (e.g. UNI, COMP) revert if the value passed to approve or transfer is larger than uint96. 
in `_giveAllowances` function contract set Approve to type(uint256).max this will revert for UNI or COMP tokens.
## Impact
contract will not work .
## Code Snippet
```solidity
        IERC20Metadata(output).forceApprove(unirouter, type(uint256).max);
        IERC20Metadata(output).forceApprove(rewardPool, type(uint256).max);
        IERC20Metadata(lpToken0).forceApprove(nftManager, type(uint256).max);
        IERC20Metadata(lpToken1).forceApprove(nftManager, type(uint256).max);
```
https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/42ef5f0eac1bc954e888cf5bfb85cbf24c08ec76/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L828C1-L831C78
## Tool used

Manual Review

## Recommendation
add support for UNI , COMP tokens