Flaky Goldenrod Chinchilla

high

# _addLiquidty can be reverted when guage is not alive


## Summary
The gauge may not be active, but this is not validated in the _addLiquidity.
This can lead to reverted transactions when deposit and withdraw functions are called, preventing users from withdrawing their tokens.

## Vulnerability Detail

https://github.com/velodrome-finance/slipstream/blob/main/contracts/gauge/CLGauge.sol#L185

```solidity
File: contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L300
        if (positionMain.nftId != 0) ICLGauge(gauge).deposit(positionMain.nftId); // @audit voter.isAlive()? "GK"
        if (positionAlt.nftId != 0) ICLGauge(gauge).deposit(positionAlt.nftId);
    
```

## Impact
If the gauge is not active, transactions involving _addLiquidity may revert, causing users to be unable to withdraw their tokens.


## Code Snippet
https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L300

## Tool used

Manual Review

## Recommendation
It's recommended to add the validation to ensure the guage is alive and update the related code.
```diff
File: contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L300
+    if (IVoter(ICLGauge(gauge).voter()).isAlive(gauge)) {
        if (positionMain.nftId != 0) ICLGauge(gauge).deposit(positionMain.nftId);
        if (positionAlt.nftId != 0) ICLGauge(gauge).deposit(positionAlt.nftId);
+    }

File: contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L333
        if (positionMain.nftId != 0) {
            (,,,,,,,liquidity,,,,) = INftPositionManager(nftManager).positions(positionMain.nftId);
-            ICLGauge(gauge).withdraw(positionMain.nftId);
+            if (ICLGauge(gauge).stakedContains(address(this), positionMain.nftId)) ICLGauge(gauge).withdraw(positionMain.nftId);
        } 

        if (positionAlt.nftId != 0) {
            (,,,,,,,liquidityAlt,,,,) = INftPositionManager(nftManager).positions(positionAlt.nftId);
-            ICLGauge(gauge).withdraw(positionAlt.nftId);
+            if (ICLGauge(gauge).stakedContains(address(this), positionAlt.nftId)) ICLGauge(gauge).withdraw(positionAlt.nftId);
        }

File: contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L463~L464
-        ICLGauge(gauge).getReward(positionMain.nftId);
+        if (positionMain.nftId != 0 && ICLGauge(gauge).stakedContains(address(this), positionMain.nftId)) ICLGauge(gauge).getReward(positionMain.nftId);
-        ICLGauge(gauge).getReward(positionAlt.nftId);
+        if (positionAlt.nftId != 0 && ICLGauge(gauge).stakedContains(address(this), positionAlt.nftId)) ICLGauge(gauge).getReward(positionAlt.nftId);
```