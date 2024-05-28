Harsh Strawberry Wasp

medium

# Incorrect twap interval can jeopardise deposit and withdraw protection

## Summary
The `twapInterval` is incorrectly set to 120 seconds = 2 minutes instead of 60 seconds = 1 minute ,
as stated in `twap` method's Natspec and CLM's Docs which poses threats of performing activities across invalid ticks.

## Vulnerability Detail
The CLM introduces [Calmness Check](https://docs.beefy.finance/beefy-products/clm#calmness-check)
to protect against the flashloan attacks and other scenarios as stated in docs

```markdown

A wider general issue with many forms of concentrated liquidity products is that prices can be manipulated by attackers, for instance using flash loans. To protect against this, CLM incorporates checks on deposit to ensure that deposits are only made in "calm" periods, meaning periods where the relative change of the current price arising from the deposit is not large relative to the time-weighted average price (or "TWAP") of the pool. 

The "calm zone" is shown in the blue area of the diagram below, and is bound by the pool's TWAP over the last 60 seconds. Where in some cases the current price exits the blue "calm zone", the contract's logic has identified large relative changes and the deposit transaction will be reverted to safeguard against attacks aimed at stealing part of the user's deposit funds.
```
The calmness is ensured through the following modifier

```solidity
  /// @notice function to only allow deposit/setTick actions when current price is within a certain deviation of twap.
    function _onlyCalmPeriods() private view {
        if (!isCalm()) revert NotCalm();
    }

  /// @notice function to only allow deposit/setTick actions when current price is within a certain deviation of twap.
    function isCalm() public view returns (bool) {
        int24 tick = currentTick();
        int56 twapTick = twap();

        int56 minCalmTick = int56(SignedMath.max(twapTick - maxTickDeviation, MIN_TICK));
        int56 maxCalmTick = int56(SignedMath.min(twapTick + maxTickDeviation, MAX_TICK));

        // Calculate if tick move more than allowed from twap and revert if it did. 
        if(minCalmTick > tick  || maxCalmTick < tick) return false;
        else return true;
    }
```

that internally relies on `twap` method for fetching the observation of twap per `twap interval`

The `twap` method is defined as follows

```solidity
 /** 
     * @notice The twap of the last minute from the pool.
     * @return twapTick The twap of the last minute from the pool.
    */
    function twap() public view returns (int56 twapTick) {
        uint32[] memory secondsAgo = new uint32[](2);
        secondsAgo[0] = uint32(twapInterval);
        secondsAgo[1] = 0;

        (int56[] memory tickCuml,) = IVeloPool(pool).observe(secondsAgo);
        twapTick = (tickCuml[1] - tickCuml[0]) / int32(twapInterval);
    }
```

if we look at it's Natspec 

it states 

```solidity
     * @notice The twap of the last minute from the pool.
     * @return twapTick The twap of the last minute from the pool.
```

additionally documentation also [states the same fact ](https://docs.beefy.finance/beefy-products/clm#calmness-check)


```markdown
The "calm zone" is shown in the blue area of the diagram below, and is bound by the pool's `TWAP over the last 60 seconds`
```

however the constructor/initialize method of the contract declares it to be `2 minutes ` or `120 seconds`

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
        //snip
        // Set the twap interval to 120 seconds.
        twapInterval = 120;

        //snip
    
    }
    
```

This conflicts with the documentation and Natspec of the protocol leading to incorrect assumptions around twap.

## Impact
The calm period will allow to use the twap prices of 2 minutes instead of 1 minute due to incorrect value set.

which indeed will allow executing operations like deposit etc. along the tick ranges 

that are not viable for protocol to operate on instead the calm period will be incorrectly calculated

and operations will be executed leading the system into crisis situation.
## Code Snippet
https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L185
## Tool used

Manual Review

## Recommendation
Set the twap interval to 60 seconds or upgrade the documentation and Natspec of the function.
