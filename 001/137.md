Elegant Menthol Jellyfish

high

# Incorrect reward calculation for Previously Withdrawn Staking Rewards on Updating `durationToMultiplier`

## Summary

The contract allows the adjustment of `durationToMultiplier` via the `setDurationMultipliers` function, which permits authorized callers to update these values for future staking durations. However, altering these multipliers affects past staking rewards, causing discrepancies in the claimed rewards.

## Vulnerability Detail

Main aim of `VVVETHStaking` contract is to allow users to stake `ETH` for a particular duration and claim `VVVToken` as a reward.

The reward calculation is based on a formula which uses `durationToMultiplier`:

```solidity

236:      accruedVvv =
237:            (nominalAccruedEth * ethToVvvExchangeRate() * durationToMultiplier[_stake.stakeDuration]) /
238:            DENOMINATOR;

```
[Link to Code](https://github.com/sherlock-audit/2024-03-vvv-vesting-staking/blob/main/vvv-platform-smart-contracts/contracts/staking/VVVETHStaking.sol#L236-L238)

Initially, these multipliers are set in the constructor:

```solidity

    constructor(
        address _authorizationRegistryAddress
    ) VVVAuthorizationRegistryChecker(_authorizationRegistryAddress) {
        durationToSeconds[StakingDuration.ThreeMonths] = 90 days;
        durationToSeconds[StakingDuration.SixMonths] = 180 days;
        durationToSeconds[StakingDuration.OneYear] = 360 days;

        durationToMultiplier[StakingDuration.ThreeMonths] = 10_000;
        durationToMultiplier[StakingDuration.SixMonths] = 15_000;
        durationToMultiplier[StakingDuration.OneYear] = 30_000;
    }

```
[Link to Code](https://github.com/sherlock-audit/2024-03-vvv-vesting-staking/blob/main/vvv-platform-smart-contracts/contracts/staking/VVVETHStaking.sol#L236-L238)

However, a scenario arises where:

1. Alice stakes 1 ETH for 3 months.
2. Alice is entitled to claim 1 VVV Token at the end of 3 months (given a 1:1 Conversion & 1x Multiplier).
3. Alice then restakes the 1 ETH for an additional 3 months.
4. At this point, Admin Opts to update multiplier factor `durationToMultiplier` for 3 months to `15_000` (which is a normal Behaviour & role of `setDurationMultipliers` Function).
5. Due to this adjustment, Alice should receive `1.5` VVV Tokens for the second stake at the end of the next 3 months.

But when alice calls `claimVvv` after 3 months, Calculation unfolds as follows:

1. `claimableVvv` will be calculated by calling `calculateClaimableVvvAmount` function.

```solidity

    function claimVvv(uint256 _vvvAmount) external {
        if (_vvvAmount == 0) revert CantClaimZeroVvv();

@->     uint256 claimableVvv = calculateClaimableVvvAmount();
        if (_vvvAmount > claimableVvv) revert InsufficientClaimableVvv();

        userVvvClaimed[msg.sender] += _vvvAmount;

        vvvToken.safeTransfer(msg.sender, _vvvAmount);

        emit VvvClaim(msg.sender, _vvvAmount);
    }

```

2. `calculateClaimableVvvAmount` function will return: return value of `calculateAccruedVvvAmount` - 1e18 (as Alice has previously claimed 1 VVVToken).

```solidity

    function calculateClaimableVvvAmount() public view returns (uint256) {
@->     return calculateAccruedVvvAmount() - userVvvClaimed[msg.sender];
    }

```

3. Since Alice has two stakes - one withdrawn and the other currently withdrawable - the `calculateAccruedVvvAmount` function is called for both stakes, with the results aggregated into `totalVvvAccrued`.

```solidity

    function calculateAccruedVvvAmount() public view returns (uint256) {
        uint256[] memory stakeIds = _userStakeIds[msg.sender];
        if (stakeIds.length == 0) return 0;

        uint256 totalVvvAccrued;
        for (uint256 i = 0; i < stakeIds.length; ++i) {
            StakeData memory stake = userStakes[msg.sender][stakeIds[i]];
            unchecked {
@->             totalVvvAccrued += calculateAccruedVvvAmount(stake);
            }
        }

        return totalVvvAccrued;
    }

```

4. Within the `calculateAccruedVvvAmount(stake)` function, `accruedVvv` for the first stake is now `1.5e18` (due to the updated `durationToMultiplier`). Similarly, `accruedVvv` for the second stake is also `1.5e18`.

```solidity

    function calculateAccruedVvvAmount(StakeData memory _stake) public view returns (uint256) {
        uint256 stakeDuration = durationToSeconds[_stake.stakeDuration];
        uint256 secondsSinceStakingStarted = block.timestamp - _stake.stakeStartTimestamp;
        uint256 secondsStaked;
        uint256 nominalAccruedEth;
        uint256 accruedVvv;

        unchecked {
            secondsStaked = secondsSinceStakingStarted >= stakeDuration
                ? stakeDuration
                : secondsSinceStakingStarted;

            nominalAccruedEth = (secondsStaked * _stake.stakedEthAmount) / stakeDuration;

@->         accruedVvv =
                (nominalAccruedEth * ethToVvvExchangeRate() * durationToMultiplier[_stake.stakeDuration]) /
                DENOMINATOR;
        }

        return accruedVvv;
    }

```

Consequently, the net `claimableVvv` for Alice amounts to `2e18` instead of `1.5e18`. This extra `0.5e18` is the reward for previously withdrawn `ETH` of first stake.

This happened because `calculateClaimableVvvAmount` function fails to differentiate between withdrawn and currently staked, resulting in inaccurate calculations.

Similarly, If the `durationToMultiplier` is reduced, it will lead to race condition where users may frontrun the `setDurationMultipliers` transaction and prematurely claim VVV Tokens. If users refrain from frontrunning, they will lose their already accrued rewards.

## Impact

Normal Admin function call leading to Incorrect computation of rewards.

## Code Snippet

Shown Above.

## Tool used

Manual Review

## Recommendation

There is no straightforward solution to this issue. However, a potential refactor could be:

1. Modifying the `calculateAccruedVvvAmount` function to skip withdrawn stakes during reward computation.

```diff

    for (uint256 i = 0; i < stakeIds.length; ++i) {
        StakeData memory stake = userStakes[msg.sender][stakeIds[i]];
        unchecked {
+           if(!stake.stakeIsWithdrawn){
              totalVvvAccrued += calculateAccruedVvvAmount(stake);
+           }
        }
    }

```

2. Adjusting the `withdrawStake` function to update `UserVvvClaimed` and exclude already withdrawn stakes from future reward calculations.

```diff

    function withdrawStake(uint256 _stakeId) external {
        StakeData storage stake = userStakes[msg.sender][_stakeId];
        _withdrawChecks(stake);

        stake.stakeIsWithdrawn = true;
        (bool withdrawSuccess, ) = payable(msg.sender).call{ value: stake.stakedEthAmount }("");
        if (!withdrawSuccess) revert WithdrawFailed();

+       claimVvv();
+       userVvvClaimed[msg.sender] -= calculateAccruedVvvAmount(stake);

        // SNIP
    }

```