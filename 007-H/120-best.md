Alert Lemonade Boa

high

# Accounting Error in `lpToken0ToNativePrice()` and `lpToken1ToNativePrice()` Functions in `StrategyPassiveManagerVelodrome`

## Summary
The `lpToken0ToNativePrice()` and `lpToken1ToNativePrice()` functions in the `StrategyPassiveManagerVelodrome` contract are designed to obtain the price of the LP tokens in native tokens. However, these functions fail to scale up the result, leading to a loss of precision.

## Vulnerability Detail
The vulnerability lies in the following functions:

```solidity
    /// @notice Returns the price of the first token in native token.
    function lpToken0ToNativePrice() public returns (uint256) {
@-->    uint amount = 10**IERC20Metadata(lpToken0).decimals() / 10;
        if (lpToken0 == native) return amount;
        return IQuoter(quoter).quoteExactInput(lpToken0ToNativePath, amount);
    }

    /// @notice Returns the price of the second token in native token.
    function lpToken1ToNativePrice() public returns (uint256) {
@-->    uint amount = 10**IERC20Metadata(lpToken1).decimals() / 10;
        if (lpToken1 == native) return amount;
        return IQuoter(quoter).quoteExactInput(lpToken1ToNativePath, amount);
    }
```
https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/42ef5f0eac1bc954e888cf5bfb85cbf24c08ec76/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L723C1-L735C6

The `amount` is scaled down by a factor of `10` when calculated from the decimals, but it is not scaled up when returned, causing precision loss.

Please see how `StrategyPassiveManagerUniswap` contract handled this

https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/42ef5f0eac1bc954e888cf5bfb85cbf24c08ec76/cowcentrated-contracts/contracts/strategies/uniswap/StrategyPassiveManagerUniswap.sol#L720C1-L731C6

## Impact
This miscalculation leads to precision loss and accounting errors in the contract due to getting a output amount scaled down by 10. 

## Code Snippet
https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/42ef5f0eac1bc954e888cf5bfb85cbf24c08ec76/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L725C1-L727C78

https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/42ef5f0eac1bc954e888cf5bfb85cbf24c08ec76/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L732C1-L734C78

## Tool used

Manual Review

## Recommendation
Update the `lpToken0ToNativePrice()` and `lpToken1ToNativePrice()` functions to scale the result correctly.

```diff
    /// @notice Returns the price of the first token in native token.
    function lpToken0ToNativePrice() public returns (uint256) {
        uint amount = 10**IERC20Metadata(lpToken0).decimals() / 10;
-       if (lpToken0 == native) return amount;
+       if (lpToken0 == native) return amount * 10;
-       return IQuoter(quoter).quoteExactInput(lpToken0ToNativePath, amount);
+       return IQuoter(quoter).quoteExactInput(lpToken0ToNativePath, amount) * 10; 
    }


    /// @notice Returns the price of the second token in native token.
-   function lpToken1ToNativePrice() public returns (uint256) {
+   function lpToken1ToNativePrice() public returns (uint256 amountOut) {
        uint amount = 10**IERC20Metadata(lpToken1).decimals() / 10;
-       if (lpToken0 == native) return amount;
+       if (lpToken0 == native) return amount * 10;
-       return IQuoter(quoter).quoteExactInput(lpToken1ToNativePath, amount);
+       return IQuoter(quoter).quoteExactInput(lpToken1ToNativePath, amount) * 10;
    }
```