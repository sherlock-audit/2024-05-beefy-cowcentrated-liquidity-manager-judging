Daring Basil Osprey

high

# StrategyPassiveManagerVelodrome.sol - `_addLiquidity` can be DoS'ed constantly

## Summary

## Vulnerability Detail
The protocol interacts with the Velodrome `nftManager`, when it attempts to `_addLiquidity`.

If we look at `nftManager.mint`:

```jsx
 function mint(MintParams calldata params)
        external
        payable
        override
        checkDeadline(params.deadline)
        returns (uint256 tokenId, uint128 liquidity, uint256 amount0, uint256 amount1)
    {
        if (params.sqrtPriceX96 != 0) {
            ICLFactory(factory).createPool({
                tokenA: params.token0,
                tokenB: params.token1,
                tickSpacing: params.tickSpacing,
                sqrtPriceX96: params.sqrtPriceX96
            });
        }
        PoolAddress.PoolKey memory poolKey =
            PoolAddress.PoolKey({token0: params.token0, token1: params.token1, tickSpacing: params.tickSpacing});

        ICLPool pool = ICLPool(PoolAddress.computeAddress(factory, poolKey));

        (liquidity, amount0, amount1) = addLiquidity(
            AddLiquidityParams({
                poolAddress: address(pool),
                poolKey: poolKey,
                recipient: address(this),
                tickLower: params.tickLower,
                tickUpper: params.tickUpper,
                amount0Desired: params.amount0Desired,
                amount1Desired: params.amount1Desired,
                amount0Min: params.amount0Min,
                amount1Min: params.amount1Min
            })
        );

        _mint(params.recipient, (tokenId = _nextId++));

        bytes32 positionKey = PositionKey.compute(address(this), params.tickLower, params.tickUpper);
        (, uint256 feeGrowthInside0LastX128, uint256 feeGrowthInside1LastX128,,) = pool.positions(positionKey);

        // idempotent set
        uint80 poolId = cachePoolKey(address(pool), poolKey);

        _positions[tokenId] = Position({
            nonce: 0,
            operator: address(0),
            poolId: poolId,
            tickLower: params.tickLower,
            tickUpper: params.tickUpper,
            liquidity: liquidity,
            feeGrowthInside0LastX128: feeGrowthInside0LastX128,
            feeGrowthInside1LastX128: feeGrowthInside1LastX128,
            tokensOwed0: 0,
            tokensOwed1: 0
        });

        refundETH();

        emit IncreaseLiquidity(tokenId, liquidity, amount0, amount1);
    }
```

The important part for this issue is at the end.

`refundETH` will attempt to transfer any ETH (native token) that the contract currently has, to `msg.sender`.

```jsx
/// @inheritdoc IPeripheryPayments
    function refundETH() public payable override nonReentrant {
        if (address(this).balance > 0) TransferHelper.safeTransferETH(msg.sender, address(this).balance);
    }
```

At first glance this isn't a problem, as `StrategyPassiveManagerVelodrome` doesn't send any native tokens when calling the `nftManager`.

`nftManager` also can't receive native tokens, unless it's WETH that's calling the `receive` function.

```jsx
receive() external payable {
        require(msg.sender == WETH9, "NW9");
    }
```

But there is another way to forcefully send native tokens to any address and that's using `selfdestruct`.

When a contract calls `selfdestruct`, its entire native token balance is sent to a specified address. Note that this is happens even if the contract can't receive native tokens, i.e missing receive/fallback or in the case of the `nftManager` it has a `receive`, but it only allows for WETH to call it.

Knowing this, if `nftManager` has any native balance, it will attempt to send it to `msg.sender`, in our case that is the `StrategyPassiveManagerVelodrome`.

This will make the tx revert, as `StrategyPassiveManagerVelodrome` doesn't have a `receive/fallback` function.

Because of this any calls to `_addLiquidity` can be DoS'ed at any time.

This is especially problematic for `withdraw`, as if the strategy isn't paused, it will call `_addLiquidity`, which as stated above can be DoSed.

The attack can be pulled off on any chain, as the attacker can simply spam deploying contracts, funding them with 1 wei, then self-destructing them and sending the funds to the `nftManager`.

This will be even easier to do when the protocol and Velodrome deploy on other chains other than OP, as the possibility of front-running a `withdraw/deposit`, makes the attack even easier to pull off.


## Impact
DoS of `_addLiquidity`, which will directly DoS `deposit/withdraw/setPositionWidth/unpause/moveTicks`

## Code Snippet
https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/42ef5f0eac1bc954e888cf5bfb85cbf24c08ec76/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L28
## Tool used
Manual Review

## Recommendation
Add a `receive` function to the strategy and possibly add a `onlyOwner` withdraw function that will transfer out any native tokens that the strategy has accumulated.