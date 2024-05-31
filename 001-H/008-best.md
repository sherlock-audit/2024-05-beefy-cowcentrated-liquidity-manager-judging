Trendy Plastic Tiger

high

# Protocol mints and decreases liquidity from Velodrome without slippage/deadline


## Summary

No implementation of any slippage or deadline protection whatsoever when minting a new position or decreasing liquidity of a previous position.

Would be key to note that **this is not the same** with the listed OOS in the known issue section from the readMe, i.e: https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/42ef5f0eac1bc954e888cf5bfb85cbf24c08ec76/README.md#L42-L44

```markdown
### Q: Please list any known issues/acceptable risks that should not result in a valid finding.

We know that swapping our fees via the router can cause loss due to lack of slippage/priceProtection, it is a known issue. We have mitigation in place via frequent harvest, harvesting via private rpcs, and a swapper oracle which will be implemented in the future.

---
```

This is because:

1. In the cases shown in the report the context is not about fees.
2. In the cases shown in the report no swaps occur, rather we are attempting to mint a new position or reduce the liquidity attached to a previous position.

## Vulnerability Detail

Take a look at https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/42ef5f0eac1bc954e888cf5bfb85cbf24c08ec76/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L330-L393

```solidity
    function _removeLiquidity() private {
        uint128 liquidity;
        uint128 liquidityAlt;
        if (positionMain.nftId != 0) {
            (,,,,,,,liquidity,,,,) = INftPositionManager(nftManager).positions(positionMain.nftId);
            ICLGauge(gauge).withdraw(positionMain.nftId);
        }

        if (positionAlt.nftId != 0) {
            (,,,,,,,liquidityAlt,,,,) = INftPositionManager(nftManager).positions(positionAlt.nftId);
            ICLGauge(gauge).withdraw(positionAlt.nftId);
        }

        // init our params
        INftPositionManager.DecreaseLiquidityParams memory decreaseLiquidityParams;
        INftPositionManager.CollectParams memory collectParams;

        // If we have liquidity in the positions we remove it and collect our tokens.
        if (liquidity > 0) {
            decreaseLiquidityParams = INftPositionManager.DecreaseLiquidityParams({//@audit
                tokenId: positionMain.nftId,
                liquidity: liquidity,
                amount0Min: 0,
                amount1Min: 0,
                deadline: block.timestamp
            });

            collectParams = INftPositionManager.CollectParams({
                tokenId: positionMain.nftId,
                recipient: address(this),
                amount0Max: type(uint128).max,
                amount1Max: type(uint128).max
            });

            INftPositionManager(nftManager).decreaseLiquidity(decreaseLiquidityParams);
            INftPositionManager(nftManager).collect(collectParams);
            INftPositionManager(nftManager).burn(positionMain.nftId);
            positionMain.nftId = 0;

        }

        if (liquidityAlt > 0) {
            decreaseLiquidityParams = INftPositionManager.DecreaseLiquidityParams({//@audit decreasing liquidity is done without slippage or read deadline
                tokenId: positionAlt.nftId,
                liquidity: liquidityAlt,
                amount0Min: 0,
                amount1Min: 0,
                deadline: block.timestamp
            });

            collectParams = INftPositionManager.CollectParams({
                tokenId: positionAlt.nftId,
                recipient: address(this),
                amount0Max: type(uint128).max,
                amount1Max: type(uint128).max
            });

            INftPositionManager(nftManager).decreaseLiquidity(decreaseLiquidityParams);
            INftPositionManager(nftManager).collect(collectParams);
            INftPositionManager(nftManager).burn(positionAlt.nftId);
            positionAlt.nftId = 0;

        }
    }
```

This function is used to remove liquidity from the main and alternative positions.

Note that this is a core function as it is called on deposit, withdraw and harvest.

However, we can see that both instances of attaching the `DecreaseLiquidityParams` struct we can see that acceptable slippage is 100% and an invalid deadline is being passed, cause the deadline being `block.timestamp` means that no matter how long the transaction sits in the mempool it's going to be perceived as valid.

Similarly, take a look at https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/42ef5f0eac1bc954e888cf5bfb85cbf24c08ec76/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L305-L327

```solidity
    function _mintPosition(int24 _tickLower, int24 _tickUpper, uint256 _amount0, uint256 _amount1, bool _mainPosition) private {
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

        (uint256 nftId,,,) = INftPositionManager(nftManager).mint(mintParams);

        if (_mainPosition) positionMain.nftId = nftId;
        else positionAlt.nftId = nftId;

        IERC721(nftManager).approve(gauge, nftId);
    }
```

This function is used to mint a new position for the main or alternative position, we can see that in this instance too no real deadline is passed and even worse no slippage is also applied.

Both instances of querying the `INftPositionManager` would be perceived as unsafe.

## Impact

Not having any valid slippage or deadline protection leaves this attempt vulnerable to attacks in the case of minting this could cause having an unbalanced pool and in the case of decreasing the liquidity this could lead to unfair rate between `amount0`/`amount1` and the liquidity burnt.

## Code Snippet

https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/42ef5f0eac1bc954e888cf5bfb85cbf24c08ec76/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L305-L327

https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/42ef5f0eac1bc954e888cf5bfb85cbf24c08ec76/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L330-L393

## Tool used

Manual Review

## Recommendation

Apply real slippage and deadline when minting a new position or decreasing liquidity.
