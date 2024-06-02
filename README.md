# Issue M-1: Accounting will be broken if `output` token is one of the `lpTokens` 

Source: https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager-judging/issues/71 

## Found by 
Dliteofficial, bughuntoor, iamnmt
## Summary
Accounting will be broken if `output` token is one of the `lpTokens`

## Vulnerability Detail
When the strategy's balances is calculated, it counts both the funds in the LP positions and the funds within the strategy contract, fetched via regular `erc20.balanceOf`.
```solidity
    function balances() public view returns (uint256 token0Bal, uint256 token1Bal) {
        (uint256 thisBal0, uint256 thisBal1) = balancesOfThis();
        (uint256 poolBal0, uint256 poolBal1,,,,) = balancesOfPool();

        uint256 total0 = thisBal0 + poolBal0;
        uint256 total1 = thisBal1 + poolBal1;

        // For token0 and token1 we return balance of this contract + balance of positions - feesUnharvested.
        return (total0, total1);
    }
```
```solidity
    function balancesOfThis() public view returns (uint256 token0Bal, uint256 token1Bal) {
        return (IERC20Metadata(lpToken0).balanceOf(address(this)), IERC20Metadata(lpToken1).balanceOf(address(this)));
    }
```

The problem is that the `output` token might be one of the `lpTokens` too and any accrued fees that are not yet harvested will be included in this number.

This would unfairly inflate share value when people are depositing via the Vault. Once rewards are collected though, these same depositors would suffer all the losses. Furthermore, it would lead to insolvency as the accrued fees might be deposited in the LP position, hence it will be hard to harvest them in order to temporarily fix accounting.

## Impact
Broken accounting, loss of funds, insolvency

## Code Snippet
https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L536C1-L538C6

## Tool used

Manual Review

## Recommendation
If one of the `lpTokens` is `output` token, deduct the `fees` from the token balance



## Discussion

**MirthFutures**

This is a nice catch, although this can only happen if output is one of the tokens it should be accounted for. 

