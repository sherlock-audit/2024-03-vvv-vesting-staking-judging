Wonderful Leather Wombat

medium

# User can front-run a call to `_setVestingSchedule` and cause double-spending

## Summary
In certain case, user could cause double-spending

## Vulnerability Detail
With the `_setVestingSchedule` admin can update a user's ongoing vesting schedule.
```solidity
    function _setVestingSchedule(SetVestingScheduleParams memory _params) private {
        VestingSchedule memory newSchedule = _params.vestingSchedule;

        if (_params.vestingScheduleIndex == userVestingSchedules[_params.vestedUser].length) {
            userVestingSchedules[_params.vestedUser].push(newSchedule);
        } else if (_params.vestingScheduleIndex < userVestingSchedules[_params.vestedUser].length) {
            userVestingSchedules[_params.vestedUser][_params.vestingScheduleIndex] = newSchedule;
```

In certain case where the user is being vested a certain amount and expects to get their vesting changed/ updated, they can wait to front-run the `_setVestingSchedule`, claim as much as possible from the old vesting schedule and then their `vestingSchedule.tokenAmountWithdrawn` will get nulled, allowing them to also claim the whole new vesting schedule, resulting in double-spending 

## Impact
Loss of funds

## Code Snippet
https://github.com/sherlock-audit/2024-03-vvv-vesting-staking/blob/main/vvv-platform-smart-contracts/contracts/vesting/VVVVesting.sol#L191

## Tool used

Manual Review

## Recommendation
When updating a vesting schedule, don't null their tokenAmountWithdrawn
