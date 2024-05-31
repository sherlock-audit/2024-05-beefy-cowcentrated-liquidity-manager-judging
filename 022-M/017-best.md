Smooth Amethyst Urchin

high

# `uniswapV3MintCallback()` will be reverted if uniswap pool's internal logic is postponed.

## Summary

Originally, the expected uniswap mint execution flow might be as follows.
1. Set `minting` to `true` first time.  (#L248)
2. Call UniswapV3Pool's `mint()` first time. (#L249)
3. `uniswapV3MintCallback()` is called first time from uniswap pool. (#L593)
4. Set `minting` to `false` first time. (#L599)

5. Set `minting` to `true` second time.  (#L265)
6. Call UniswapV3Pool's `mint()` second time. (#L266)
7. `uniswapV3MintCallback()` is called second time from uniswap pool. (#L593)
8. Set `minting` to `false` second time. (#L599)

And the calling order (Step3 and Step7) of  `uniswapV3MintCallback()` can be reversed since it is called from uniswap pool, not from same transaction.
But on `uniswapV3MintCallback()`, `minting` state variable is set to `false` regardless of the existence of pending `mint()` call.

## Vulnerability Detail

The calling order of  `uniswapV3MintCallback()` can be reversed as follows if uniswap pool's internal logic is postponed. .
1. Set `minting` to `true` first time.  (#L248)
2. Call UniswapV3Pool's `mint()` first time. (#L249)
3. Set `minting` to `true` second time.  (#L265)
4. Call UniswapV3Pool's `mint()` second time. (#L266)

5. `uniswapV3MintCallback()` is called first time from uniswap pool. (#L593)
6. Set `minting` to `false` first time. (#L599)
7. `uniswapV3MintCallback()` is called second time from uniswap pool. (#L593)
8. Since `minting` is already `false`, `uniswapV3MintCallback` will be reverted. (#L595)

## Impact

If the first calling of `uniswapV3MintCallback()`is postponed after the second calling of `mint`, the second calling of `mint`will be failed.

## Code Snippet

https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/tree/main/cowcentrated-contracts/contracts/strategies/uniswap/StrategyPassiveManagerUniswap.sol#L77

https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/tree/main/cowcentrated-contracts/contracts/strategies/uniswap/StrategyPassiveManagerUniswap.sol#L229-L268

https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/tree/main/cowcentrated-contracts/contracts/strategies/uniswap/StrategyPassiveManagerUniswap.sol#L593-L600

## Tools Used

Manual Review

## Recommendation

Please update `StrategyPassiveManagerUniswap` as follows.
1. Update `minting` state variable type from `bool` to `uin8` .

```diff
--	bool private minting;
++	uint8 private minting;
```

2. Update `_addLiquidity()` as follows.

```diff
function _addLiquidity() private {
	...
	if (liquidity > 0 && amountsOk) {
--		minting = true;
++		minting += 1;
		IUniswapV3Pool(pool).mint(
			address(this), 
			positionMain.tickLower, 
			positionMain.tickUpper, 
			liquidity, 
			"Beefy Main"
		);
	}
	...
	if (liquidity > 0) {
--		minting = true;
++		minting += 1;
		IUniswapV3Pool(pool).mint(
			address(this), 
			positionAlt.tickLower, 
			positionAlt.tickUpper, 
			liquidity, 
			"Beefy Alt"
		);
	}
}
```

3. Update `uniswapV3MintCallback()` as follows.

```diff
function uniswapV3MintCallback(
	uint256 amount0, uint256 amount1, bytes memory /*data*/) external {

	if (msg.sender != pool) revert NotPool();
	
--	if (!minting) revert InvalidEntry();
++	if (minting == 0) revert InvalidEntry();
	
	if (amount0 > 0) IERC20Metadata(lpToken0).safeTransfer(pool, amount0);
	if (amount1 > 0) IERC20Metadata(lpToken1).safeTransfer(pool, amount1);
	
--	minting = false;
++	if (minting > 0) minting -= 1;	
}
```