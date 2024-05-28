Trendy Plastic Tiger

high

# An easily manipulated price data is being used when minting/adding liquidity for positions


## Summary

See _Vulnerability Detail_.

## Vulnerability Detail

Take a look at https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/42ef5f0eac1bc954e888cf5bfb85cbf24c08ec76/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L608-L620

```solidity
    function price() public view returns (uint256 _price) {
        uint160 sqrtPriceX96 = sqrtPrice();
        _price = FullMath.mulDiv(uint256(sqrtPriceX96), SQRT_PRECISION, (2 ** 96)) ** 2;
    }

    function sqrtPrice() public view returns (uint160 sqrtPriceX96) {
        (sqrtPriceX96,,,,,) = IVeloPool(pool).slot0();
    }

```

Evidently in both instances the price from `slot0` is used, issue however is that the price returned from `slot0` is the easily manipulatable since `slot0` is the most recent data point and can be manipulated easily via MEV bots and Flashloans with sandwich attacks.

Would be key to note that protocol has hinted that they would deploy on any EVM compatible chain, which heavily increases the likelihood of the manipulation of this data as some not so popular instances would be way easier to manipulate considering there wouldn't be a lot of liquidation to resist the manipulation.

Also keep in mind that the sqrtprice returned from this is what's been used to query the amount of liquidity the execution of adding liquidity gets with the current balances via `_addLiquidity()` as shown below: https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/42ef5f0eac1bc954e888cf5bfb85cbf24c08ec76/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L243-L303

```solidity
    function _addLiquidity() private {
        _whenStrategyNotPaused();

        (uint256 bal0, uint256 bal1) = balancesOfThis();

        int24 mainLower = positionMain.tickLower;
        int24 mainUpper = positionMain.tickUpper;
        int24 altLower = positionAlt.tickLower;
        int24 altUpper = positionAlt.tickUpper;

        // Then we fetch how much liquidity we get for adding at the main position ticks with our token balances.
|>          uint160 sqrtprice = sqrtPrice();
        uint128 liquidity = LiquidityAmounts.getLiquidityForAmounts(
|>              sqrtprice,
            TickMath.getSqrtRatioAtTick(mainLower),
            TickMath.getSqrtRatioAtTick(mainUpper),
            bal0,
            bal1
        );

        (uint256 amount0, uint256 amount1) = LiquidityAmounts.getAmountsForLiquidity(
|>              sqrtprice,
            TickMath.getSqrtRatioAtTick(mainLower),
            TickMath.getSqrtRatioAtTick(mainUpper),
            liquidity
        );

        bool amountsOk = _checkAmounts(liquidity, mainLower, mainUpper);

        // Mint or add liquidity to the position.
        if (liquidity > 0 && amountsOk) {
            _mintPosition(mainLower, mainUpper, amount0, amount1, true);
        }

        (bal0, bal1) = balancesOfThis();

        // Fetch how much liquidity we get for adding at the alternative position ticks with our token balances.
        liquidity = LiquidityAmounts.getLiquidityForAmounts(
|>              sqrtprice,
            TickMath.getSqrtRatioAtTick(altLower),
            TickMath.getSqrtRatioAtTick(altUpper),
            bal0,
            bal1
        );

        (amount0, amount1) = LiquidityAmounts.getAmountsForLiquidity(
|>            sqrtprice,
            TickMath.getSqrtRatioAtTick(altLower),
            TickMath.getSqrtRatioAtTick(altUpper),
            liquidity
        );

        // Mint or add liquidity to the position.
        if (liquidity > 0 && (amount0 > 0 || amount1 > 0)) {
            _mintPosition(altLower, altUpper, amount0, amount1, false);
        }

        if (positionMain.nftId != 0) ICLGauge(gauge).deposit(positionMain.nftId);
        if (positionAlt.nftId != 0) ICLGauge(gauge).deposit(positionAlt.nftId);
    }

```

Now the [check](https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/42ef5f0eac1bc954e888cf5bfb85cbf24c08ec76/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L270) via `_checkAmounts()` would easily pass even if the price has been manipulated cause the check only returns false if `amount0/amount1 = 0` so having a value as low as `amount0/amount1 = 1` would pass.

Now since as hinted above the `sqrtPrice` data is being used to mint new positions this could easily cause for the minting of unbalanced positions.

Would only be fair to indicate that `deposit()`& `moveTicks()` are _subtly protoected_ ..._(subtly because the [twap duration]() in scope is very short and as such quite manipulatable)_ via the `onlyCalmPeriods` modifier all other instances where `_addliquidity() `gets queried is unprotected, i.e `withdraw()`, `unpause()` & `setPositionWidth()`.

## Impact

As already hinted under _Vulnerability Detail_, since an easily manipulated price data is being used when minting liquidity for positions this would cause for having unbalanced positions or even allowing for unfair liquidity to be minted for the available tokens

## Code Snippet

https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/42ef5f0eac1bc954e888cf5bfb85cbf24c08ec76/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L243-L303

## Tool used

Manual Review

## Recommendation

Use a twap protected price when minting liquidity.
