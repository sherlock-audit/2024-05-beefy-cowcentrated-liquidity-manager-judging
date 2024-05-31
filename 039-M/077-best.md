Future Carmine Giraffe

medium

# Use of CLPools where LPToken0 == output || LPToken1 == output would result in loss of LP rewards for LPs and temporary inability to harvest fees

## Summary

## Vulnerability Detail
During the initialization of the strategy, the initializer is required to provide the CLPool address. If a CLPool where LPToken0 == output || LPToken1 == output is used, there will be serious complications and this is possible because the LPToken0 and LPToken1 isn't checked that they are not the same as the output address in [`StrategyPassiveManagerVelodrome::initialize`](https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L155C5-L189C6). 

**Why is it important?**
Well, the output token is the reward token. The token in which the trading fees gathered from providing liquidity in the CLPool is paid. This token is also the token that the fees to the call recipient, DAO, etc. is paid on Beefy. 

**What complications would this cause?**
Let's take `StrategyPassiveManagerVelodrome::beforeAction()` for example:

```solidity
    function beforeAction() external {
        _onlyVault();
        _claimEarnings();
        _removeLiquidity();
    }
```

This function is called before `StrategyPassiveManagerVelodrome::withdraw()`. When this happens, the fees are accumulated (in output) and we remove liquidity (assuming LPToken1 is also output), the total of LPToken1 will be the removed liquidity + fees. If `_harvest()` is not called quickly and all of the tokens are supplied again as liquidity, both the fee recipients and LPs will lose their entitlements. Particularly, the LPs. Since the tokens are reinvested, they will not be able to earn LP rewards. As for the fee recipients, because all of the tokens have been reinvested, they will temporarily be unable to claim their fees.

## Impact

Loss of LP rewards for LPs and temporary inability to harvest fees.

## Code Snippet

```solidity
    function initialize (
        address _pool,
        address _quoter,
        address _nftManager,
        address _gauge,
        address _rewardPool,
        address _output, 
        int24 _positionWidth,
        bytes[] calldata _paths,
        CommonAddresses calldata _commonAddresses
    ) external initializer {
        ..........................

        lpToken0 = IVeloPool(_pool).token0();
        lpToken1 = IVeloPool(_pool).token1();

        ..........................
    }
```

## Tool used

Manual Review

## Recommendation
```diff
    function initialize (
        address _pool,
        address _quoter,
        address _nftManager,
        address _gauge,
        address _rewardPool,
        address _output, 
        int24 _positionWidth,
        bytes[] calldata _paths,
        CommonAddresses calldata _commonAddresses
    ) external initializer {
        __StratFeeManager_init(_commonAddresses);

        pool = _pool;
        quoter = _quoter;
        output = _output;
        nftManager = _nftManager;
        gauge = _gauge;
        rewardPool = _rewardPool;   


        lpToken0 = IVeloPool(_pool).token0();
        lpToken1 = IVeloPool(_pool).token1();

+       if(lpToken0 == output ||lpToken1 == output) revert();

        positionWidth = _positionWidth;

        outputToNativePath = _paths[0];
        lpToken0ToNativePath = _paths[1];
        lpToken1ToNativePath = _paths[2];
    
        twapInterval = 120;

        _giveAllowances();
    
    }
```