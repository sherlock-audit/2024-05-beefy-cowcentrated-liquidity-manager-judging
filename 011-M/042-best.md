Wild Gunmetal Horse

high

# Incorrect total Balance calculation

## Summary
In the [`StrategyPassiveManagerVelodrome::balances()`](https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L520-L529) the total balance is not subtracted by the fees unharvested resulting in an overstated value.
## Vulnerability Detail
In the `StrategyPassiveManagerVelodrome::balances()`  it is required that the total balance to be returned must be subtracted by the fees unharvested to get the actual balance. But the total balance returned is not returned as intended by the invariant in the [comment](https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L527).
## Impact
This will result in higher total balance than expected and therefore breaking the invariant stated int the comment
## Code Snippet
```solidity
    function balances() public view returns (uint256 token0Bal, uint256 token1Bal) {
        (uint256 thisBal0, uint256 thisBal1) = balancesOfThis();
        (uint256 poolBal0, uint256 poolBal1,,,,) = balancesOfPool();

@-->        uint256 total0 = thisBal0 + poolBal0;
@-->        uint256 total1 = thisBal1 + poolBal1;
        // @audit fees unhaversted isnt subtracted

        // For token0 and token1 we return balance of this contract + balance of positions - feesUnharvested.
@-->        return (total0, total1);
    
```
## Tool used

Manual Review

## Recommendation
Subtract the unharvested fee from the total balance being returned