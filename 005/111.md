Alert Lemonade Boa

high

# Incompatible Return Values Cause `lpToken0ToNativePrice()` and `lpToken1ToNativePrice()` Functions in `StrategyPassiveManagerVelodrome` to Always Revert

## Summary
The `lpToken0ToNativePrice()` and `lpToken1ToNativePrice()` functions in the `StrategyPassiveManagerVelodrome` contract are intended to fetch the price of LP tokens in the native token. However, these functions always revert due to incorrect handling of the return values from Velodrome's `quoter` contract.

## Vulnerability Detail
The issue lies in the way the return values from the `quoteExactInput()` function of the velodromes `quoter` contract are handled. The functions expect a single `uint256` return value, but `quoteExactInput()` returns a tuple.

```solidity
    /// @notice Returns the price of the first token in native token.
    function lpToken0ToNativePrice() public returns (uint256) {
        uint amount = 10**IERC20Metadata(lpToken0).decimals() / 10;
        if (lpToken0 == native) return amount;
@-->    return IQuoter(quoter).quoteExactInput(lpToken0ToNativePath, amount);
    }

    /// @notice Returns the price of the second token in native token.
    function lpToken1ToNativePrice() public returns (uint256) {
        uint amount = 10**IERC20Metadata(lpToken1).decimals() / 10;
        if (lpToken1 == native) return amount;
@-->    return IQuoter(quoter).quoteExactInput(lpToken1ToNativePath, amount);
    }
```
https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/42ef5f0eac1bc954e888cf5bfb85cbf24c08ec76/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L723C1-L735C6

The `quoteExactInput()` function in the `quoter` contract returns multiple values:

```solidity
    function quoteExactInput(bytes memory path, uint256 amountIn)
        public
        override
        returns (
            uint256 amountOut,
            uint160[] memory sqrtPriceX96AfterList,
            uint32[] memory initializedTicksCrossedList,
            uint256 gasEstimate
        )
    {
```
Both functions in `StrategyPassiveManagerVelodrome` expect a `uint256` return type, causing them to revert due to the mismatch in expected return types.

Please see velodrome `quoter` contract for more clarity

https://vscode.blockscan.com/optimism/0xA2DEcF05c16537C702779083Fe067e308463CE45

## Impact
The functions `lpToken0ToNativePrice()` and `lpToken1ToNativePrice()` are unusable because they always revert. This makes it impossible to obtain the price of LP tokens in the native token.

## Code Snippet
https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/42ef5f0eac1bc954e888cf5bfb85cbf24c08ec76/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L727C1-L727C78
https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/42ef5f0eac1bc954e888cf5bfb85cbf24c08ec76/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L734

## Tool used

Manual Review

## Recommendation
Update the `lpToken0ToNativePrice()` and `lpToken1ToNativePrice()` functions to correctly handle the return values from `quoteExactInput()`.

Suggested Fix
```diff
    /// @notice Returns the price of the first token in native token.
-   function lpToken0ToNativePrice() public returns (uint256) {
+   function lpToken0ToNativePrice() public returns (uint256 amountOut) {
        uint amount = 10**IERC20Metadata(lpToken0).decimals() / 10;
        if (lpToken0 == native) return amount;
-       return IQuoter(quoter).quoteExactInput(lpToken0ToNativePath, amount);
+       (amountOut,,,) = IQuoter(quoter).quoteExactInput(lpToken0ToNativePath, amount);
    }


    /// @notice Returns the price of the second token in native token.
-   function lpToken1ToNativePrice() public returns (uint256) {
+   function lpToken1ToNativePrice() public returns (uint256 amountOut) {
        uint amount = 10**IERC20Metadata(lpToken1).decimals() / 10;
        if (lpToken1 == native) return amount;
-       return IQuoter(quoter).quoteExactInput(lpToken1ToNativePath, amount);
+       (amountOut,,,) = IQuoter(quoter).quoteExactInput(lpToken1ToNativePath, amount);
    }
```