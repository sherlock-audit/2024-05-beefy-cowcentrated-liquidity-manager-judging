Harsh Strawberry Wasp

medium

# `TickMath` might revert in solidity version 0.8

## Summary
The `TickMath` library was designed by uniswap V3 team to perform tick mathematics in an efficient manner.

The compiler version used  was ` > 0.5  &   < 0.8` it is because the library has performed many operations

that were based on overflow and underflow assumptions to work correctly , 

however Beefy's Tick Math , which is copied from [Uniswap V3's Offcial TickMath.sol for <0.8 ](https://github.com/Uniswap/v3-core/blob/main/contracts/libraries/TickMath.sol)

has changed the compiler version to `>=0.8.0` which means if any result overflows or underflows , the calculation will revert

because solidity >=0.8 will always revert on over/under flow scenarios.

This behavior can also be seen in `FullMath.sol`

Due to this , the protocol might not work as intended breaking the core assumptions of the library of Uniswap v3.

Uniswap has recognized this issue and they have made a specific version for 0.8 version too.

## Vulnerability Detail
Continuing the summary , the Uniswap's `0.8` branch contains the [Tick Math with unchecked blocks](https://github.com/Uniswap/v3-core/blob/0.8/contracts/libraries/TickMath.sol#L27)

The `FullMath.sol`  also misses an unchecked block


```solidity
 
    function mulDivRoundingUp(
        uint256 a,
        uint256 b,
        uint256 denominator
    ) internal pure returns (uint256 result) {
        result = mulDiv(a, b, denominator);
        if (mulmod(a, b, denominator) > 0) {
            require(result < type(uint256).max);
            result++;
        }
    }
```
this is necessary because the uniswap v3's 0.8 FullMath imposes unchecked block on this method [here](https://github.com/Uniswap/v3-core/blob/0.8/contracts/libraries/FullMath.sol#L120)

demonstrating the importance of it

```solidity
    function mulDivRoundingUp(
        uint256 a,
        uint256 b,
        uint256 denominator
    ) internal pure returns (uint256 result) {
        unchecked {
            result = mulDiv(a, b, denominator);
            if (mulmod(a, b, denominator) > 0) {
                require(result < type(uint256).max);
                result++;
            }
        }
    }
```

The protocol should use that version for itself to avoid any revert scenarios in the transactions that utilize this library

affecting the core logic of the protocol and leaving it to DoS scenarios.

## Impact
Operations involving the use of Tick like 

- `_addLiquidity` 
- `removeLiquidity` 

which are used inside 

- `deposit` 
- `withdraw` 

methods will fail.

As the reported issue `Breaks core contract functionality` like deposits and withdraws  rendering the contract useless at some scenarios , the reported bug is at least a valid medium severity bug.

## Code Snippet
https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/42ef5f0eac1bc954e888cf5bfb85cbf24c08ec76/cowcentrated-contracts/contracts/utils/TickMath.sol#L2

## Tool used

Manual Review

## Recommendation
Use the 0.8 branch [Uniswap V3's Offcial TickMath.sol for <0.8 ](https://github.com/Uniswap/v3-core/blob/main/contracts/libraries/TickMath.sol)