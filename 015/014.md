Soft Sangria Lion

medium

# VeloSwapUtils::swap() passes address[] instead of route[] to V2_SWAP_EXACT_IN due to incorrect pathToRoute function

## Summary

`VeloSwapUtils::swap()` passes `address[]` instead of `route[]` to `V2_SWAP_EXACT_IN`, which will cause a revert due to type mismatch.

## Vulnerability Detail
As confirmed by the Sponsor:
> The "unirouter" for this contract uses Velodromes Slipstream Universal Router which has execute.
> https://optimistic.etherscan.io/address/0xF132bdb9573867cD72f2585C338B923F973EB817

[VeloSwapUtils::swap()](https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/utils/VeloSwapUtils.sol#L22-L41)
```solidity
    function swap(
        address _router,
        bytes memory _path,
        uint256 _amountIn,
        bool _isV3
    ) internal {
        if (_isV3) {
            bytes memory input = abi.encode(address(this), _amountIn, 0, _path, true); // @audit-issue 0 minAmountOut
            bytes[] memory inputs = new bytes[](1);
            inputs[0] = input;
            IVeloRouter(_router).execute(abi.encodePacked(V3_SWAP_EXACT_IN), inputs, block.timestamp);
        } else {
>>          address[] memory route = pathToRoute(_path);
            bytes memory input = abi.encode(address(this), _amountIn, 0, route, true);
            bytes[] memory inputs = new bytes[](1);
            inputs[0] = input;

            IVeloRouter(_router).execute(abi.encodePacked(V2_SWAP_EXACT_IN), inputs, block.timestamp);
        }
    }
```
As seen `route` is an array of addresses, rather than routes. `Routes` should contain:
```solidity
struct Route {
    address from;
    address to;
    bool stable;
}
```
This will cause the call to revert or be corrupted when calling `execute`:
[Dispatcher::L131-L141](https://optimistic.etherscan.deth.net/address/0xf132bdb9573867cd72f2585c338b923f973eb817)
```solidity
                    if (command == Commands.V2_SWAP_EXACT_IN) {
                        // equivalent: abi.decode(inputs, (address, uint256, uint256, Route[], bool))
                        address recipient;
                        uint256 amountIn;
                        uint256 amountOutMin;
                        bool payerIsUser;
                        Route[] memory routes;
>>                      (recipient, amountIn, amountOutMin, routes, payerIsUser) =
>>                          abi.decode(inputs, (address, uint256, uint256, Route[], bool));
                        address payer = payerIsUser ? lockedBy : address(this);
                        v2SwapExactInput(map(recipient), amountIn, amountOutMin, routes, payer);
```
When `abi.decode` attempts to decode it will revert due to the miss match of datatypes between `Route[]` and `address[]`.

This is caused by [VeloSwapUtils::pathToRoute()](https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/utils/VeloSwapUtils.sol#L80-L90) returning `address[]` instead of `Route[]`:
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

## Impact

All calls using `VeloSwapUtils::swap()` where `V2_SWAP_EXACT_IN` is utilised as the command (or `_isV3` is false) will revert or be corrupted due to the wrong array being utilised, breaking code functionality.

## Code Snippet

[VeloSwapUtils::swap()](https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/utils/VeloSwapUtils.sol#L22-L41)
[VeloSwapUtils::pathToRoute()](https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/utils/VeloSwapUtils.sol#L80-L90)

## Tool used

Manual Review

## Recommendation

Ensure that `pathToRoute` returns `Route[]` if it's going to be utilised as an input to `unirouter`.

------

However ensure to check `StrategyPassiveManagerVelodrome.sol` for usage of the current version of `pathToRoute` as any current usage will expect a `address[]` return, such as:

[StrategyPassiveManagerVelodrome::setOutputToNativePath()](https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L691-L699)

[StrategyPassiveManagerVelodrome::outputToNative()](https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L718-L721) 