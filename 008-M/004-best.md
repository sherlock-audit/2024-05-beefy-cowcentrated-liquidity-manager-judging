Soft Sangria Lion

medium

# StrategyPassiveManagerVelodrome::retireVault() can be front-run to prevert retirement

## Summary

When a manager calls `retireVault()` the function checks to ensure that only the initial supply minted during creation is present. However a malicious griefer can deposit a minimum amount to prevent the `totalSupply` check from passing, causing manager functionality to be broken.

## Vulnerability Detail
[StrategyPassiveManagerVelodrome::retireVault()](https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L792-L800)
```solidity
    function retireVault() external onlyOwner {
>>      if (IBeefyVaultConcLiq(vault).totalSupply() != 10**3) revert NotAuthorized();
        panic(0,0);
        address feeRecipient = beefyFeeRecipient();
        IERC20Metadata(lpToken0).safeTransfer(feeRecipient, IERC20Metadata(lpToken0).balanceOf(address(this)));
        IERC20Metadata(lpToken1).safeTransfer(feeRecipient, IERC20Metadata(lpToken1).balanceOf(address(this)));
        _transferOwnership(address(0));
    }
```
The owner has the functionality to retire a vault when it is no longer utilised, however a malicious user can frontrun the call and add a minimum deposit to cause the call to revert due to the `totalSupply` check.

The contract also cannot be already paused (which would prevent deposits) as the `panic()` function will call `_pause()` which reverts if it's already paused:
[PausableUpgradeable::_pause()](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/master/contracts/utils/PausableUpgradeable.sol#L115-L126)
```solidity
    /**
     * @dev Triggers stopped state.
     *
     * Requirements:
     *
     * - The contract must not be paused.
     */
    function _pause() internal virtual whenNotPaused {
        PausableStorage storage $ = _getPausableStorage();
        $._paused = true;
        emit Paused(_msgSender());
    }
```
The modifier `whenNotPaused` will cause the call to revert if the contract is already paused.

## Impact

A malicious griefer can prevent a manager from utilising intended functionality of retiring a vault by frontrunning. The contract cannot be paused before calling `retireVault()` due to the `_pause()` call within the function meaning owner has no way to prevent this.
This breaks intended functionality of the contract, classifying this as a Medium risk issue. 

## Code Snippet

[StrategyPassiveManagerVelodrome::retireVault()](https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L792-L800)
[PausableUpgradeable::_pause()](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/master/contracts/utils/PausableUpgradeable.sol#L115-L126)

## Tool used

Manual Review

## Recommendation

Don't use `panic()` within the `retireVault()` function to allow the manager to pause the contract before calling `retireVault()`, preventing any deposit transactions front-running the admin's `retireVault()` call.
