Harsh Strawberry Wasp

medium

# Path.sol library returns incorrect results on passed data instead of reverting

## Summary
The path library used inside velodrome strategy is stand-alonely vulnerable to returning incorrect data .
## Vulnerability Detail
The path library ensures that one pool has to contain at least an address + fee + address , where address corresponds to token addresses.

The `hasMultiplePools` method ensures that the length of `path` is `>=` the collective length of 

```solidity
// addr +fee +add + add + FEE_SIZE
        assert(MULTIPLE_POOLS_MIN_LENGTH == 66);
```

and based on those sizes , the path library's other functions operate .

For example , to navigate from one Pool to another `skipToken` method is used

```solidity
 /// @notice Skips a token + fee element from the buffer and returns the remainder
    /// @param path The swap path
    /// @return The remaining token + fee elements in the path
    function skipToken(bytes memory path) internal pure returns (bytes memory) {
        return path.slice(NEXT_OFFSET, path.length - NEXT_OFFSET);
    }
```

and there are other methods like 

```solidity
    /// @notice Decodes the first pool in path
    /// @param path The bytes encoded swap path
    /// @return tokenA The first token of the given pool
    /// @return tokenB The second token of the given pool
    /// @return fee The fee level of the pool
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

    /// @notice Gets the segment corresponding to the first pool in the path
    /// @param path The bytes encoded swap path
    /// @return The segment containing all data necessary to target the first pool in the path
    function getFirstPool(bytes memory path) internal pure returns (bytes memory) {
        return path.slice(0, POP_OFFSET);
    }
```

however , these methods when used in conjunction returns incorrect decoded data 

because for a `bytes` variable to be a valid pool , these methods only check `if there length meets certain threshold`

in case the bytes length criteria is met but the way encoding is done is incorrect ,

the decoded data will be incorrectly valid ( because the function to decode the invalid pool data will not revert )

### Proof of concept

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Test} from "forge-std/Test.sol";
import {console} from "forge-std/console.sol";

import "../src/Path.sol";

contract PathTest is Test {
    using Path for bytes;

    function setUp() public {}

    function test_PathLib() public {
     
        bytes memory mulPoolPath = abi.encodePacked(
            vm.addr(1),
            uint24(300),
            vm.addr(2),
            vm.addr(2),
            uint(8)
        );
        bool hasMultiplePools = mulPoolPath.hasMultiplePools();

        assert(hasMultiplePools == true);
        (address tokenA, address tokenB, uint24 fee) = mulPoolPath
            .decodeFirstPool();
        console.log("tokenA: ", tokenA);
        console.log("tokenB: ", tokenB);
        console.log("Fee: ", fee);

        mulPoolPath = mulPoolPath.skipToken();
        (tokenA, tokenB, fee) = mulPoolPath.decodeFirstPool();
        console.log("tokenA: ", tokenA);
        console.log("tokenB: ", tokenB);
        console.log("Fee: ", fee);
    }
}

```

### PoC Output

```sh
Ran 1 test for test/Path.t.sol:PathTest
[PASS] test_PathLib() (gas: 14034)
Logs:
  tokenA:  0x7E5F4552091A69125d5DfCb7b8C2659029395Bdf
  tokenB:  0x2B5AD5c4795c026514f8317c7a215E218DcCD6cF
  Fee:  300
  tokenA:  0x2B5AD5c4795c026514f8317c7a215E218DcCD6cF
  tokenB:  0xC4795c026514F8317C7A215e218dcCD6Cf000000
  Fee:  2841301

Traces:
  [121] PathTest::setUp()
    └─ ← ()

  [14034] PathTest::test_PathLib()
    ├─ [0] VM::addr(<pk>) [staticcall]
    │   └─ ← 0x7E5F4552091A69125d5DfCb7b8C2659029395Bdf
    ├─ [0] VM::addr(<pk>) [staticcall]
    │   └─ ← 0x2B5AD5c4795c026514f8317c7a215E218DcCD6cF
    ├─ [0] VM::addr(<pk>) [staticcall]
    │   └─ ← 0x2B5AD5c4795c026514f8317c7a215E218DcCD6cF
    ├─ [0] console::log("tokenA: ", 0x7E5F4552091A69125d5DfCb7b8C2659029395Bdf) [staticcall]
    │   └─ ← ()
    ├─ [0] console::log("tokenB: ", 0x2B5AD5c4795c026514f8317c7a215E218DcCD6cF) [staticcall]
    │   └─ ← ()
    ├─ [0] console::log("Fee: ", 300) [staticcall]
    │   └─ ← ()
    ├─ [0] console::log("tokenA: ", 0x2B5AD5c4795c026514f8317c7a215E218DcCD6cF) [staticcall]
    │   └─ ← ()
    ├─ [0] console::log("tokenB: ", 0xC4795c026514F8317C7A215e218dcCD6Cf000000) [staticcall]
    │   └─ ← ()
    ├─ [0] console::log("Fee: ", 2841301 [2.841e6]) [staticcall]
    │   └─ ← ()
    └─ ← ()

Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 3.13ms
```
We can see the returned data is incorrect 

## Impact
Incorrectly decoded data will be returned by the library because the library inherently relies on abi.encodepacked data

## Code Snippet
https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/utils/Path.sol#L42-L68
## Tool used

Manual Review

## Recommendation
Update the logic to rely on strict types of data using abi encode without packing .
