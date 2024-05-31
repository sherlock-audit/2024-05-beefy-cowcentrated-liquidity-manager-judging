Trendy Plastic Tiger

medium

# Swaps that need to use the `VeloSwapUtils#pathToRoute()` would not work

## Summary

Swaps that need to use the `VeloSwapUtils#pathToRoute() `like Velodrome Finance's `V2_SWAP_EXACT_IN` would not work cause the `isStable` pool data is not being attached to the encoded `route` data.

## Vulnerability Detail

Take a look at https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/42ef5f0eac1bc954e888cf5bfb85cbf24c08ec76/cowcentrated-contracts/contracts/utils/VeloSwapUtils.sol#L80-L91

```solidity
    function pathToRoute(bytes memory _path) internal pure returns (address[] memory) {
        uint256 numPools = _path.numPools();
        address[] memory route = new address[](numPools + 1);
        for (uint256 i; i < numPools; i++) {
            (address tokenA, address tokenB,) = _path.decodeFirstPool();
            route[i] = tokenA;
            route[i + 1] = tokenB;
            _path = _path.skipToken();
        }
        return route;
    }

```

This is the function that's used to convert the encoded path to token route in the case where the swap to be made is Velodrome Finance's `V2_SWAP_EXACT_IN`, this is confirmed as this function is called [here](https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/42ef5f0eac1bc954e888cf5bfb85cbf24c08ec76/cowcentrated-contracts/contracts/utils/VeloSwapUtils.sol#L34) & [here](https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/42ef5f0eac1bc954e888cf5bfb85cbf24c08ec76/cowcentrated-contracts/contracts/utils/VeloSwapUtils.sol#L70) which are all instances before the execution is passed to the Velo router via ` IVeloRouter(_router).execute(abi.encodePacked(V2_SWAP_EXACT_IN), inputs, block.timestamp);`, issue however is that going to the implementation of Velodrome's `V2_SWAP_EXACT_IN`: https://github.com/velodrome-finance/universal-router/blob/d6e73715651420c85b3d20da3e58427d1dad1cf1/README.md#how-the-input-bytes-are-structures we can see that the `V2_SWAP_EXACT_IN` expects not only `tokenA` & `tokenB` in the routes, but also a specification of the `isStable` bool, quoting them:

```markdown
For `V2_SWAP_EXACT_IN` The following parameters are required:

- `address` The recipient of the output of the trade
- `uint256` The amount of input tokens for the trade
- `amountOutMin` The minimum amount of output tokens the user wants
- `Route[]` The route struct representing the V2 pool (address from, address to, bool stable)
- `bool` A flag for whether the input funds should come from the caller
```

Now, evidently the `pathToRoute()` function above does not pass any encoding of the `isStable` bool which would then make this attempts to swap not work for `V2_SWAP_EXACT_IN`.

## Impact

Core functionality is broken, protocol's intended functionality of swaps via the velo router with `V2_SWAP_EXACT_IN` would not work since the routes array only include the logic of `tokenA`, `tokenB` and not the `isStable` bool params.

## Code Snippet

https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/42ef5f0eac1bc954e888cf5bfb85cbf24c08ec76/cowcentrated-contracts/contracts/utils/VeloSwapUtils.sol#L80-L91

```solidity
    function pathToRoute(bytes memory _path) internal pure returns (address[] memory) {
        uint256 numPools = _path.numPools();
        address[] memory route = new address[](numPools + 1);
        for (uint256 i; i < numPools; i++) {
            (address tokenA, address tokenB,) = _path.decodeFirstPool();
            route[i] = tokenA;
            route[i + 1] = tokenB;
            _path = _path.skipToken();
        }
        return route;
    }

```

## Tool used

Manual Review

## Recommendation

Reimplement the `pathToRoute` function and attach the `isStable` bool to the `route` array returned.

### Additional Note

Would be key to note that this has been stated in the readMe: https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/42ef5f0eac1bc954e888cf5bfb85cbf24c08ec76/README.md#L38-L41

> ### Q: Should potential issues, like broken assumptions about function behavior, be reported if they could pose risks in future integrations, even if they might not be an issue in the context of the scope? If yes, can you elaborate on properties/invariants that should hold?

> Yes

---


Which means this issue is **in scope** for this contest even if the `V2_SWAP_EXACT_IN` swap option is not currently being used from the in-scope `StrategyPassiveManagerVelodrome.sol` cause [the only instance where `VeloSwapUtils.swap()` currently gets called sets the `isV3` bool value to true](https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/42ef5f0eac1bc954e888cf5bfb85cbf24c08ec76/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L493) which means that not the `V2_SWAP_EXACT_IN` swap option is going to be used.
