Alert Lemonade Boa

medium

# Dangerous Use of Deadline Parameter

## Summary
The protocol uses `block.timestamp` as the `deadline` argument while interacting with the Velodrome NFT Position Manager, which completely defeats the purpose of using a `deadline`.

## Vulnerability Detail
Actions in the Velodrome `NonfungiblePositionManager` contract are protected by a `deadline` parameter to limit the execution of pending transactions. Functions that modify the liquidity of the pool check this parameter against the current block timestamp to discard expired actions.

These interactions with the Velodrome position are present in the `StrategyPassiveManagerVelodrome` contract. The functions `_mintPosition()` and `_removeLiquidity()` call functions from the Velodrome `NonfungiblePositionManager` contract, providing `block.timestamp` as the argument for the `deadline` parameter:

```solidity
    INftPositionManager.MintParams memory mintParams = INftPositionManager.MintParams({
        token0: lpToken0,
        token1: lpToken1,
        tickSpacing: _tickDistance(),
        tickLower: _tickLower,
        tickUpper: _tickUpper,
        amount0Desired: _amount0,
        amount1Desired: _amount1,
        amount0Min: 0,
        amount1Min: 0,
        recipient: address(this),
        deadline: block.timestamp,
        sqrtPriceX96: 0
    });
```
https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/42ef5f0eac1bc954e888cf5bfb85cbf24c08ec76/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L306C8-L319C12

```solidity
    decreaseLiquidityParams = INftPositionManager.DecreaseLiquidityParams({
        tokenId: positionMain.nftId,
        liquidity: liquidity,
        amount0Min: 0,
        amount1Min: 0,
        deadline: block.timestamp
    });
```
https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/42ef5f0eac1bc954e888cf5bfb85cbf24c08ec76/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L349C1-L355C16

```solidity
    decreaseLiquidityParams = INftPositionManager.DecreaseLiquidityParams({
        tokenId: positionAlt.nftId,
        liquidity: liquidityAlt,
        amount0Min: 0,
        amount1Min: 0,
        deadline: block.timestamp
    });
```
Using `block.timestamp` as the `deadline` is effectively a no-op that provides no protection. Since `block.timestamp` takes the timestamp value when the transaction is mined, the check will end up comparing `block.timestamp` against the same value, i.e., `block.timestamp <= block.timestamp`.

(See `PeripheryValidation` contract in https://vscode.blockscan.com/optimism/0x1d5951dfcd9d7f830a9aed6d127bbeb9f69df276).

Failure to provide a proper `deadline` value enables pending transactions to be maliciously executed at a later point. Transactions that provide an insufficient amount of gas and are not mined within a reasonable amount of time can be picked by malicious actors or MEV bots and executed later to the detriment of the submitter.

## Impact
Using `block.timestamp` as the `deadline` parameter in interactions with Velodrome's `NonfungiblePositionManager` contract nullifies the purpose of the `deadline`, leaving transactions vulnerable to delayed execution by malicious actors or MEV bots. This can lead to the unintended execution of pending transactions, potentially causing financial loss or other detrimental effects.

## Code Snippet

## Tool used

Manual Review

## Recommendation
Add a `deadline` parameter to each of the functions used to manage the liquidity position, `_mintPosition()`, `_removeLiquidity()`. Forward this parameter to the corresponding underlying calls to the Velodrome `NonfungiblePositionManager` contract.