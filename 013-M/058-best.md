Polished Lemonade Chinchilla

medium

# `maxDeviation` is unusable for low fee percentage pools.

## Summary
`maxDeviation` is unusable for low fee percentage pools.

## Vulnerability Detail
Within the contract, protocol owner sets a `maxDeviation` parameter, which corresponds to what will be considered a `calm` period - if deviation is out of set bounds, certain actions are restricted.

```solidity
    function setDeviation(int56 _maxDeviation) external onlyOwner {
        emit SetDeviation(_maxDeviation);

        // Require the deviation to be less than or equal to 4 times the tick spacing.
        if (_maxDeviation >= _tickDistance() * 4) revert InvalidInput();

        maxTickDeviation = _maxDeviation;
    }
```

However, the problem is that deviation only be less than 4x the `ticxkSpacing` of the pool.

Considering some Velodrome pools have as low `tickSpacing` as of [1](https://github.com/velodrome-finance/slipstream/blob/main/contracts/core/CLFactory.sol#L59), this means that these pools can handle only up to ~0.03% price deviation. Such low deviation will cause strategy to be unusable almost constantly.

## Impact
DoS

## Code Snippet
https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L705C1-L712C6

## Tool used

Manual Review

## Recommendation
Remove this max deviation restriction or adjust it properly for these low tickspacing pools. 