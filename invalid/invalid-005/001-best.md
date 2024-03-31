Rare Boysenberry Pheasant

medium

# Centralized Withdraw Functionality

## Summary
An issue has been identified in the `VVVETHStaking` contract, where highly centralized withdrawal functionality could potentially lead to the loss of user funds. The withdrawEth function allows an authorized user (presumably an admin) to withdraw any amount of ETH from the contract without checks against the total staked amount or the state of individual stakes. This could result in a scenario where there are insufficient funds in the contract to honor user withdrawals, directly impacting users' ability to withdraw their staked ETH.

## Vulnerability Detail
The contract provides a function `withdrawEth(uint256 _amount)` that permits an authorized user to withdraw a specified amount of ETH from the contract. There are no validations within this function to ensure that the withdrawal amount does not exceed the total available balance minus the currently staked amounts. Consequently, this could lead to a total depletion of the contract's balance, leaving nothing for the stakers to withdraw.

## Impact
If the contract balance is withdrawn by an authorized user through the withdrawEth function, users who have staked ETH will be unable to retrieve their funds. This not only breaks the trust model of the staking mechanism but also exposes users to potential total loss of their staked assets, undermining the security and integrity of the contract.


## Tool used

Manual Review

## Code Snippet
```solidity
///@notice allows admin to withdraw ETH
function withdrawEth(uint256 _amount) external  {
    (bool success, ) = payable(msg.sender).call{ value: _amount }("");
    if (!success) revert WithdrawFailed();
}
```
[Link](https://github.com/sherlock-audit/2024-03-vvv-vesting-staking/blob/main/vvv-platform-smart-contracts/contracts/staking/VVVETHStaking.sol#L284-L287)

## Recommendation
To mitigate this issue and ensure the security of user funds, the following recommendations are made:

* Implement a Check Against Total Staked Amount: Before processing the withdrawal in withdrawEth, the contract should calculate the total amount of ETH currently staked and ensure that the withdrawal amount does not exceed the contract's balance minus the total staked amount. This will prevent the contract balance from being depleted beyond what is needed to fulfill outstanding stakes.

* Restricting Withdrawal Rights: Consider adding multi-signature functionality or a time lock for withdrawal requests made through the withdrawEth function to add an additional layer of security and oversight.

## POC
```py
>>> ethStaking.userStakes(alice, 1)
(1000000000000000000, 1711393923, False, 0)
>>> assert vvvToken.balanceOf(alice) == 0
>>> assert ethStaking.calculateClaimableVvvAmount({'from': alice}) == 1e18
>>> ethStaking.userStakes(alice, 1)
(1000000000000000000, 1711393923, False, 0)
>>> ethStaking.balance()
1000000000000000000
>>> ethStaking.withdrawEth(1e18, {'from': deployer})
Transaction sent: 0xc5719d27a33ab990d8226842d0c65317015b4f5c0af5cd5c49a890cc6c691b2f
  Gas price: 0.0 gwei   Gas limit: 12000000   Nonce: 6
  VVVETHStaking.withdrawEth confirmed   Block: 9   Gas used: 29290 (0.24%)

<Transaction '0xc5719d27a33ab990d8226842d0c65317015b4f5c0af5cd5c49a890cc6c691b2f'>
>>> ethStaking.balance()
0
>>> ethStaking.withdrawStake(1, {'from': alice})
Transaction sent: 0xa7296f197c1706277d4312280911f7f44b2541a00a15b5cbe172933e12dccd08
  Gas price: 0.0 gwei   Gas limit: 12000000   Nonce: 1
  VVVETHStaking.withdrawStake confirmed (reverted)   Block: 10   Gas used: 53870 (0.45%)

<Transaction '0xa7296f197c1706277d4312280911f7f44b2541a00a15b5cbe172933e12dccd08'>
```
