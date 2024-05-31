Soft Sangria Lion

medium

# VeloSwapUtils.sol utilises wrong Route struct for interacting with unirouter

## Summary

`VeloSwapUtils.sol` and `IVeloRouter.sol` utilise an incorrect `Route` struct when interacting with `unirouter`.

## Vulnerability Detail
As confirmed by the Sponsor:
> The "unirouter" for this contract uses Velodromes Slipstream Universal Router which has execute.
> https://optimistic.etherscan.io/address/0xF132bdb9573867cD72f2585C338B923F973EB817

The [UniRouter ](https://optimistic.etherscan.deth.net/address/0xf132bdb9573867cd72f2585c338b923f973eb817) utilises this `Route` struct:
```solidity
struct Route {
    address from;
    address to;
    bool stable;
}
```
However [VeloSwapUtils.sol](https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/utils/VeloSwapUtils.sol#L9-L14) utilises this `Route` struct:
```solidity
library VeloSwapUtils {
    struct Route {
        address from;
        address to;
        bool stable;
        address factory;
    }
```
Which is based off of the [Velodrome Router](https://optimistic.etherscan.deth.net/address/0xa062ae8a9c5e11aaa026fc2670b0d65ccc8b2858#writeContract):
```solidity
    struct Route {
        address from;
        address to;
        bool stable;
        address factory;
    }
```
However as previously mentioned, `VeloSwapUtils.sol` will be utilised with the `unirouter` therefore should contain the same `Route` struct structure.

[StrategyPassiveManagerVelodrome::_chargeFees()](https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L493) calls:
```solidity
VeloSwapUtils.swap(unirouter, outputToNativePath, amountToSwap, true);
```
Here the `unirouter` is passed into `VeloSwapUtils::swap()` therefore the `Route` struct within  `VeloSwapUtils.sol` should match the structure of `unirouter` however currently it matches the `Velodrome Router` structure. All the functions within `VeloSwapUtils.sol` call `execute()` which is only present in `unirouter`. Meaning that the intended integration is with `unirouter` therefore structs need to be based on it's implementation.

Similarly, [IVeloRouter.sol](https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/interfaces/velodrome/IVeloRouter.sol#L4-L10) also utilises the wrong `Route` struct:
```solidity
interface IVeloRouter { 
        struct Route { 
            address from;
            address to;
            bool stable;
            address factory;
        }
```

## Impact

Incorrect `Route[]` will be passed to `unirouter`:
```solidity
    function swap(
        address _router,
>>      IVeloRouter.Route[] memory _route,
        uint256 _amountIn
    ) internal {
        bytes memory input = abi.encode(address(this), _amountIn, 0, _route, true);
        bytes[] memory inputs = new bytes[](1);
        inputs[0] = input;

        IVeloRouter(_router).execute(abi.encodePacked(V2_SWAP_EXACT_IN), inputs, block.timestamp);
    }
```
Which will cause reverts or data corruption, breaking functionality of the integration with `unirouter`.

## Code Snippet

[VeloSwapUtils.sol](https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/utils/VeloSwapUtils.sol#L9-L14)
[StrategyPassiveManagerVelodrome::_chargeFees()](https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L493)
[IVeloRouter.sol](https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/interfaces/velodrome/IVeloRouter.sol#L4-L10)

## Tool used

Manual Review

## Recommendation

 Update the struct to the `uniroute` format as follows:
```diff
library VeloSwapUtils {
    struct Route {
        address from;
        address to;
        bool stable;
-        address factory;
    }
```
