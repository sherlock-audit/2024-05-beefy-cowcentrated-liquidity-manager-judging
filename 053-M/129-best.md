Expert Ultraviolet Lizard

medium

# Unsafe approval of tokens in Beefy Passive Position Manager

## Summary

The Beefy Passive Position Manager contract approves an unlimited amount of tokens to the Uniswap Router and the reward pool without first nullifying the allowance, creating a possible vulnerability known as "race condition allowance". 

## Vulnerability Detail

A malicious contract could potentially utilize the allowance set to the Uniswap router and the reward pool between the `forceApprove` function is called and when the token is actually used, to draining the remaining token amounts.

## Impact

This vulnerability might become effective in a scenario where this strategy’s ownership is maliciously transferred, allowing the new owner to drain tokens meant for rewards. This could occur if another contract, which was set as the Manager, that the Manager interacts with is malicious.

## Code Snippet

https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L827-L832

```solidity
function _giveAllowances() private {
    IERC20Metadata(output).forceApprove(unirouter, type(uint256).max);
    IERC20Metadata(lpToken0).forceApprove(nftManager, type(uint256).max);
    IERC20Metadata(lpToken1).forceApprove(nftManager, type(uint256).max);
}
```

In the function above, token allowances are set to the maximum allowable value without checking if there is no remaining allowance first.

## Tool used

Manual Review

## Recommendation

Before calling `approve()` for a new allowance, the contract should call `approve()` to set the allowance to `0`. This is due to the fact that changing allowances set by the `approve()` function in a single transaction is an error that can be exploited. 

For example,

```solidity
IERC20Metadata(output).approve(unirouter, 0);
IERC20Metadata(output).approve(unirouter, <new_amount>);
```