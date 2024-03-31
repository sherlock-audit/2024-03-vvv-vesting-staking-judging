Wonderful Raisin Fish

medium

# `VVVETHStaking` has wrong default multiplier calculation. User stake for longer period receive less reward token than user restake shorter period repeatedly.



## Summary

User 1 who stake 1e18 token for 90 days then restake again for 90 days for 180 days of staking in total.
User 2 who stake 1e18 token for 180 days in one go.
User 1 will get 20e18 as reward while user 2 will get 15e18 as reward.

Basically User 2 who stake longer period will get less reward than user 1 who stake shorter period.
This go against the rule of staking rewards where user whom take risk to lock token for longer period should get more reward than user who stake for shorter period.

## Vulnerability Detail

Here is default config for reward multiplier in `VVVETHStaking.sol` contract:

```solidity
constructor(
        address _authorizationRegistryAddress
    ) VVVAuthorizationRegistryChecker(_authorizationRegistryAddress) {
        durationToSeconds[StakingDuration.ThreeMonths] = 90 days;
        durationToSeconds[StakingDuration.SixMonths] = 180 days;
        durationToSeconds[StakingDuration.OneYear] = 360 days;

        durationToMultiplier[StakingDuration.ThreeMonths] = 10_000;
        durationToMultiplier[StakingDuration.SixMonths] = 15_000;//@audit H duration multiplier is wrong. stake 180 days get less token than double 90 days stake
        durationToMultiplier[StakingDuration.OneYear] = 30_000;
    }
```

The config here mean user who stake 180 days should get 50% more reward than user who stake 90 days.
User stake 360 days should get 200% more reward than user who stake 90 days.

Here is how reward multiplier is calculated in `VVVETHStaking.sol` contract:

`totalRewardAccrued = (timePassed / stakeDuration) * stakeAmount * rewardMultiplier;`

- `(timePassed / stakeDuration)`: range from 0 to 100%.
- `stakeAmount`: amount of token user stake.
- `rewardMultiplier`: multiplier for reward. 100% for 90 days, 150% for 180 days, 300% for 360 days.

User who stake 180 days suppose to receive 50% more reward than user who stake 90 days.

But because reward multiplier value does not stake in consideration that stakeDuration differ between staking mode, rewards is incorrect.

Using same amount of initial stake token.
Staking 90 days get 100% reward token.
Then restake again for 90 days get 100% reward token again for total of 200% reward token.
While spending 180 days staking get 150% reward token.

### Foundry POC Test

Put this test in `VVVETHStaking.unit.t.sol` file. And run with command `forge test -vv --mt "DebugStake"`

```solidity
    function testDebugStakeRewards() public {
        address user1= address(0x1000);
        address user2= address(0x2000);
        vm.deal(user1, 100 ether);
        vm.deal(user2, 100 ether);
        vm.prank(user1);
        uint user1StakeId = EthStakingInstance.stakeEth{value: 10 ether}(VVVETHStaking.StakingDuration.ThreeMonths);
        vm.prank(user2);
        EthStakingInstance.stakeEth{value: 10 ether}(VVVETHStaking.StakingDuration.SixMonths);

        skip(90 days);
        vm.prank(user1);
        uint256 user1Claim = EthStakingInstance.calculateClaimableVvvAmount();
        console.log("user1 90days claim: %e", user1Claim);
        //restake 90 days again
        vm.prank(user1);
        EthStakingInstance.restakeEth(user1StakeId, VVVETHStaking.StakingDuration.ThreeMonths);

        vm.prank(user2);
        uint256 user2Claim = EthStakingInstance.calculateClaimableVvvAmount();
        console.log("user2 90days claim: %e", user2Claim);

        skip(90 days);
        vm.prank(user1);
        user1Claim = EthStakingInstance.calculateClaimableVvvAmount();
        console.log("user1 180days claim: %e", user1Claim);
        
        vm.prank(user2);
        user2Claim = EthStakingInstance.calculateClaimableVvvAmount();
        console.log("user2 180days claim: %e", user2Claim);
        

        //   user1 90days claim: 1e19
        //   user2 90days claim: 7.5e18
        //   user1 180days claim: 2e19
        //   user2 180days claim: 1.5e19
        
    }
```

## Impact

Unfair reward distribution for user who stake longer period.

## Code Snippet
<https://github.com/sherlock-audit/2024-03-vvv-vesting-staking/blob/main/vvv-platform-smart-contracts/contracts/staking/VVVETHStaking.sol#L107-L117>
<https://github.com/sherlock-audit/2024-03-vvv-vesting-staking/blob/main/vvv-platform-smart-contracts/contracts/staking/VVVETHStaking.sol#L222-L242>

## Tool used

Manual Review

## Recommendation

Fix multiplier to take in consideration of staking duration.

```solidity
constructor(
        address _authorizationRegistryAddress
    ) VVVAuthorizationRegistryChecker(_authorizationRegistryAddress) {
        durationToSeconds[StakingDuration.ThreeMonths] = 90 days;
        durationToSeconds[StakingDuration.SixMonths] = 180 days;
        durationToSeconds[StakingDuration.OneYear] = 360 days;
        // 90 days get 100%.
        // 10_000 * 1 = 10_000
        // 180 days get 150% more than 90 days.
        // 10_000 * 2 * 1.5 = 30_000
        // 360 days get 300% more than 90 days.
        // 10_000 * 4 * 3 = 120_000
+        durationToMultiplier[StakingDuration.ThreeMonths] = 10_000;
+        durationToMultiplier[StakingDuration.SixMonths] = 30_000;
+        durationToMultiplier[StakingDuration.OneYear] = 120_000;
    }
```
