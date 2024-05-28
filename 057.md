Polished Lemonade Chinchilla

medium

# No way to claim rewards in emergency mode if output token is not native

## Summary
No way to claim rewards in emergency mode if output token is not native

## Vulnerability Detail
In certain case of emergency, contract owner can call `panic`  to enter emergency mode. This claims earning, removes liquidity, stops deposits and revokes outstanding approvals.

```solidity
    function panic(uint256 _minAmount0, uint256 _minAmount1) public onlyManager {
        _claimEarnings();
        _removeLiquidity();
        _removeAllowances();
        _pause();

        (uint256 bal0, uint256 bal1) = balances();
        if (bal0 < _minAmount0 || bal1 < _minAmount1) revert TooMuchSlippage();
    }
```

The problem is that in order to claim the rewards, if `output` token is not `native` it has to be swapped through the Velodrome router. However, since contract is in panic mode and approvals are revoked, no funds can be swapped through the router. In case of a permanent emergency (due to an issue with the Velodrome pool/Vault), these funds cannot be retrieved safely.

## Impact
Lost/Stuck funds

## Code Snippet
https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L807

## Tool used

Manual Review

## Recommendation
upon calling `panic`, if `output` token is not `native` send the rewards to owner wallet 