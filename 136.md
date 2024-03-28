Genuine Lace Meerkat

medium

# Admin cannot recover the left-out, unclaimed, or unvested $VVV rewards.


## Summary
VVVETHStaking and VVVVesting contracts lack the $VVV recovery implementation.


## Vulnerability Detail


The cases where $VVV can go in vain in the vesting and staking rewards.
1. A staker who hasn't claimed the rewards after years
2. Or a staker who is  the victim of admin's change in `durationMultiplier`, so he could only claim fewer rewards because admin changed the multiplier. So some rewards are left out, and admin cannot recover them.


## Impact
Some rewards can be left out, and admin cannot recover them.


## Code Snippet

https://github.com/sherlock-audit/2024-03-vvv-vesting-staking/blob/2d3540ab8f421055dba60d47e948e1cc55f0a0c7/vvv-platform-smart-contracts/contracts/staking/VVVETHStaking.sol#L8

## Tool used


Manual Review


## Recommendation


Add a recover $VVV function to both VVVETHStaking and VVVVesting contracts.


```solidity
    function recoverVVVV(uint _vvvAmount) external onlyAuthorized {
        vvvToken.safeTransfer(msg.sender, _vvvAmount);
    }
```