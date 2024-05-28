Expert Ultraviolet Lizard

medium

# Front-running vulnerability in the function withdraw()

## Summary

The contract StrategyPassiveManagerVelodrome.sol exposes the strategy to front-running attacks. 

## Vulnerability Detail

Front-running is a type of issue where someone can see your transaction in the mempool (these are valid transactions that haven't been included in the blockchain yet), and quickly submit their transaction with a much higher gas price, making it likely that theirs will be included first. 

## Impact

In the deposit(), withdraw(), and moveTicks() functions, there is a call to _addLiquidity() where front-running can occur. If the adversary notices these transactions, they can take advantage of the price slippage caused by these liquidity operations. 

## Code Snippet


https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L226

Infected functions:

``` solidity

function withdraw(uint256 _amount0, uint256 _amount1) external {
    _onlyVault();

    // Liquidity has already been removed in beforeAction() so this is just a simple withdraw.
    if (_amount0 > 0) IERC20Metadata(lpToken0).safeTransfer(vault, _amount0);
    if (_amount1 > 0) IERC20Metadata(lpToken1).safeTransfer(vault, _amount1);

    // After we take what is needed we add it all back to our positions. 
    if (!_isPaused()) _addLiquidity();   // <- Here

    (uint256 bal0, uint256 bal1) = balances();

    // TVL Balances after withdraw
    emit TVL(bal0, bal1);
}

function moveTicks() external onlyCalmPeriods onlyRebalancers {
    _claimEarnings();
    _removeLiquidity();
    _setTicks();
    _addLiquidity();                      // <- Here

    (uint256 bal0, uint256 bal1) = balances();
    emit TVL(bal0, bal1);
}

function deposit() external onlyCalmPeriods {
    _onlyVault();

    if (!initTicks) {
        _setTicks();
        initTicks = true;
    }

    // Add all liquidity
    _addLiquidity();                      // <- Here
            
    (uint256 bal0, uint256 bal1) = balances();

    // TVL Balances after deposit
    emit TVL(bal0, bal1);
}

```


## Tool used

Manual Review

## Recommendation

This vulnerability is difficult to mitigate completely. Here are some of the possible ways to reduce its impact:
- Instead of immediately updating the position, a delay could be added so the transaction is processed later (for this, each transaction would need to be split into commit and reveal parts).
- Implement an on-chain TWAP (Time Weighted Average Price) Oracle to ensure that prices are more stable and not easily manipulated by the adversary.
- Do not process user transactions directly; instead, use a keeper-based mechanism or internal user treadmill.
- Alternatively, use an authorization mechanism to limit the manipulation possibilities of potential attackers (i.e., use whitelisted addresses).