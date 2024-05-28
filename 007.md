Trendy Plastic Tiger

medium

# `skipToken() & decodeFirstPool()` are broken for swaps like `V2_SWAP_EXACT_IN`

## Summary

See _Vulnerability Detail_.

## Vulnerability Detail

Take a look at https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/42ef5f0eac1bc954e888cf5bfb85cbf24c08ec76/cowcentrated-contracts/contracts/utils/Path.sol#L42-L55

```solidity
    function decodeFirstPool(bytes memory path)
    internal
    pure
    returns (
        address tokenA,
        address tokenB,
        uint24 fee
    )
    {
        tokenA = path.toAddress(0);
        fee = path.toUint24(ADDR_SIZE);
        tokenB = path.toAddress(NEXT_OFFSET);
    }

```

Also see https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/42ef5f0eac1bc954e888cf5bfb85cbf24c08ec76/cowcentrated-contracts/contracts/utils/Path.sol#L66-L68

```solidity
    function skipToken(bytes memory path) internal pure returns (bytes memory) {
        return path.slice(NEXT_OFFSET, path.length - NEXT_OFFSET);
    }
```

Keep in mind that the `NEXT_OFFSET` value takes[ into account both the length of the bytes encoded address and the length of the bytes encoded fee](https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/42ef5f0eac1bc954e888cf5bfb85cbf24c08ec76/cowcentrated-contracts/contracts/utils/Path.sol#L11-L16)

Now the two functions above work properly whenever the swaps we are to process also need a fee data to be attached like [V3_SWAP_EXACT_IN](https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/42ef5f0eac1bc954e888cf5bfb85cbf24c08ec76/cowcentrated-contracts/contracts/utils/VeloSwapUtils.sol#L64-L69), however these functions are broken for swaps like `V2_SWAP_EXACT_IN` that don't expect a fee value because implementations of `skipToken() & decodeFirstPool()` are hardcoded to always query **the tokenA, tokenB and fee** from the encoded `input` data, however the input data for the `V2_SWAP_EXACT_IN` swap type would always not include these fee implementation, as the `V2_SWAP_EXACT_IN` swap type does not expect a fee, this can also be proven by the `pathToRoute()` function that always gets called before swapping via `V2_SWAP_EXACT_IN` in `VeloSwapUtils.sol`: https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/42ef5f0eac1bc954e888cf5bfb85cbf24c08ec76/cowcentrated-contracts/contracts/utils/VeloSwapUtils.sol#L79-L91

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

We can see that decoding does not expect a `fee` value to be returned from `decodeFirstPool()` it would even go ahead and set the wrong value for `tokenB` due to the `fee = path.toUint24(ADDR_SIZE);` in `decodeFirstPool()` which would corrupt the real data for `tokenB`. And then querying ` _path.skipToken()` wrongly skips more data than necessary, cause as shown above the function always expects fee to be attached to the encoded data breaking the route array that's been returned.

Note that this has also been stated in the readMe: https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/42ef5f0eac1bc954e888cf5bfb85cbf24c08ec76/README.md#L38-L41


> ### Q: Should potential issues, like broken assumptions about function behavior, be reported if they could pose risks in future integrations, even if they might not be an issue in the context of the scope? If yes, can you elaborate on properties/invariants that should hold?

> Yes

---


Which means this issue is **in scope** for this contest even if the `V2_SWAP_EXACT_IN` swap option is not currently being used in any of the in-scope contracts but is a core integration for swapping that doesn't work.

## Impact

Inability to swap via `V2_SWAP_EXACT_IN`, core functionality is broken.

## Code Snippet

https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/42ef5f0eac1bc954e888cf5bfb85cbf24c08ec76/cowcentrated-contracts/contracts/utils/Path.sol#L42-L55

https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/42ef5f0eac1bc954e888cf5bfb85cbf24c08ec76/cowcentrated-contracts/contracts/utils/Path.sol#L66-L68

https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/42ef5f0eac1bc954e888cf5bfb85cbf24c08ec76/cowcentrated-contracts/contracts/utils/Path.sol#L11-L16

## Tool used

Manual Review

## Recommendation

Consider creating other implementations for the `skipToken() & decodeFirstPool()` that do not take any fees into account when decoding the `input` data and then these new implementations should be used when getting the route for swaps like `V2_SWAP_EXACT_IN`
