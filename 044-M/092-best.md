Square Charcoal Toad

medium

# Paused Strategy Contracts can be Frozen When Trying to Unlock Them in Non Calm Periods

## Summary
The unpause function in StrategyPassiveManagerVelodrome can be forced to fail.

## Vulnerability Detail
The unpause() function
```solidity
    function unpause() external onlyManager {
        if (owner() == address(0)) revert NotAuthorized();
        _giveAllowances();
        _unpause();
        _setTicks();
        _addLiquidity();
    }
```
can be called after the Panic() function has been used for emergency reasons. The unpause function re-deploys liquidity. Therefore, it calls setTicks()
```solidity
    /// @notice Sets the tick positions for the main and alternative positions.
    function _setTicks() private onlyCalmPeriods {
        int24 tick = currentTick();
        int24 distance = _tickDistance();
        int24 width = positionWidth * distance;

        _setMainTick(tick, distance, width);
        _setAltTick(tick, distance, width);
    }
```
to set the tick range since the ideal position on Veldrome may have changed from when the contract was stopped.
Since the unpause function will revert if the isCalmPeriod is false, the contract will remain in a paused state. An attacker can perform transactions on velodrome pool with enough volume which forces the isCalm() check to return false. 
https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager-mohmoniem281/blob/1cc2325f43c8d7048b046cc595ce4b52c8d97061/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L133

```solidity
    function isCalm() public view returns (bool) {
        int24 tick = currentTick();
        int56 twapTick = twap();

        int56 minCalmTick = int56(SignedMath.max(twapTick - maxTickDeviation, MIN_TICK));
        int56 maxCalmTick = int56(SignedMath.min(twapTick + maxTickDeviation, MAX_TICK));

        // Calculate if tick move more than allowed from twap and revert if it did. 
        if(minCalmTick > tick  || maxCalmTick < tick) return false;
        else return true;
    }
```

note: Since velodrome is only deployed on L2 with no mempools, the attacker in this case will have to keep using veldrome within each block compared to front running the unpause function on L1.

## Impact
Freezing of paused strategies and not being able to unlock them.

## Code Snippet

## Tool used

Manual Review

## Recommendation
Add another emergency function which allows the manager/owner to unpause the contract without calling the setTicks() function.
