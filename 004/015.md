Icy Onyx Tapir

medium

# Due to downcasting, an invalid lock duration check may occur

medium
## Summary

Due to the downcasting issue in `_stakeEth` function, the start time `stakeStartTimestamp` is uint type, which will be forcibly converted to uint32 in the StakeData. This downcasting can potentially lead to stakers can claim reward directly after they stake their token.


## Vulnerability Detail

The `stakeEth` function in "VVVETHStaking.sol" is used to call the `_stakeEth` function to perform staking with the stake amount. And then after the duration, the staker can claim their reward. However, `stakeStartTimestamp: uint32(block.timestamp)` , which has downcast issue, when the  `block.timestamp` (a uint type variable) is bigger than `uint32`, then the staker can claim reward directly after they stake token.


### poc
```solidity
   function testStakeEth() public {
        vm.startPrank(sampleUser, sampleUser);
        uint256 newTimestamp = 1099511627776; 
        vm.warp(newTimestamp); 
        uint256 stakeEthAmount = 1;

        uint256 stakeIdBefore = EthStakingInstance.stakeId();
        EthStakingInstance.stakeEth{ value: stakeEthAmount }(VVVETHStaking.StakingDuration.OneYear);


        EthStakingInstance.stakeEth{ value: stakeEthAmount }(VVVETHStaking.StakingDuration.OneYear);
        uint256 stakeIdAfter = EthStakingInstance.stakeId();


        uint256[] memory stakeIds = EthStakingInstance.userStakeIds(sampleUser);
        uint256 userStakeIdIndex = stakeIds.length - 1;

        (
            uint256 stakedEthAmount,
            uint256 stakedTimestamp,
            bool stakeIsWithdrawn,
            VVVETHStaking.StakingDuration stakedDuration
        ) = EthStakingInstance.userStakes(sampleUser, stakeIds[userStakeIdIndex]);


        console.log("log staketimestap:", stakedTimestamp);
        console.log("real time",block.timestamp);

        uint256 claimableVvv = EthStakingInstance.calculateClaimableVvvAmount();
        console.log("claimableVvv: ",claimableVvv);

        vm.stopPrank();
    }
```

## Impact

When `block.timestamp` > `uint32.max`, the staker can claim rewards whithout limitation.

## Code Snippet

https://github.com/sherlock-audit/2024-03-vvv-vesting-staking/blob/main/vvv-platform-smart-contracts/contracts/staking/VVVETHStaking.sol#L296


## Tool used

Manual Review

## Recommendation

To avoid the issues caused by downcasting, you should make sure that the data types used for value storage correspond to the actual types that can be handled without loss.
