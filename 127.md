Breezy Rainbow Gibbon

high

# Missing Validation of Active Status in Fee Configuration before Fee Application in the function `_chargeFees`

## Summary
The contract does not check the active flag in the FeeCategory struct before applying the fees, potentially leading to the application of inactive or unintended fee configurations in the function `_chargeFees`

## Vulnerability Detail
The IFeeConfig.FeeCategory struct includes an "active" boolean variable that indicates whether the fee category is switched on or not (as stated by protocol documetation and beefy fee config codebase) . However, the contract does not verify if the active flag is set to true before implementing the fee category. This oversight could result in the application of fees that are not intended to be active. 

beefy fee config address: 0x216EEE15D1e3fAAD34181f66dd0B665f556a638d;
Protocol documentation: https://docs.beefy.finance/developer-documentation/other-beefy-contracts/feeconfigurator-contract

## Impact
Incorrect Fee Application: Inactive fee categories might be applied, leading to loss of fund

## Code Snippet
https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/42ef5f0eac1bc954e888cf5bfb85cbf24c08ec76/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L478
## Tool used

Manual Review

## Recommendation
Before applying the fee category, verify that the active flag is set to true to ensure only active fee configurations are used.
` if (fee.active == false) { _amountLeft=_amount}`
The above check should be added before applying fees in the function `_chargeFees`