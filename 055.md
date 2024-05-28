Wild Peanut Canary

medium

# Missing deadline check on swaps

## Summary
The [VeloSwapUtils::swap](https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/utils/VeloSwapUtils.sol#L22-L41) functions lack a deadline parameter, which is essential for users to limit the execution time of their pending transactions. Without this parameter, transactions can be executed at unexpected times when market conditions may be unfavorable.

This also applies to the `UniV3Utils::swap` functions which are not in the scope of this audit.

## Vulnerability Detail
The swap functions in `VeloSwapUtils` do not include a deadline parameter, which allows users to specify a maximum time limit for their transactions. This omission means that a transaction might be executed long after it was initially submitted, potentially during periods of unfavorable market conditions.

## Impact
Medium.

Impact: Low - Users are not at a loss and would only miss positive slippage.
Likelihood: High - The deadline parameter is missing.

Without a deadline parameter, users may execute transactions at unexpected times, potentially leading to less favorable outcomes. While this does not result in a direct loss (due to the slippage protection ensuring minimum output), users could miss out on positive slippage if market conditions improve before the transaction is mined.

## Code Snippet
https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/utils/VeloSwapUtils.sol#L22-L41
https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/utils/VeloSwapUtils.sol#L44-L54
https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/utils/VeloSwapUtils.sol#L57-L77

## Tool used
Manual Review

## Recommendation
Introduce a `deadline` parameter in all swap functions to provide users with the ability to limit the execution time of their transactions.