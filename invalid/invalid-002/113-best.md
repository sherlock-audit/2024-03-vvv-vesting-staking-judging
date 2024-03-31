Zany Yellow Raccoon

high

# Potential Fund Loss Due to `removeVestingSchedule` Deletion

## Summary
`removeVestingSchedule` to delete `userVestingSchedules` poses a risk of fund loss if `tokensToVestAtStart` isn't updated for vested users; hence, pre-deletion validation is vital to prevent losses.

## Vulnerability Detail
When the `removeVestingSchedule` function is called by an authorized user to delete `userVestingSchedules` of vested users, there's a risk of users losing their funds if there's no update on the `tokensToVestAtStart` for vested users. Before proceeding with the deletion of `userVestingSchedules`, it's crucial to verify if the user has any pending `tokensToVestAtStart` updates to avoid potential losses.
## Impact
vested user will lose `tokensToVestAtStart`
## Code Snippet
https://github.com/sherlock-audit/2024-03-vvv-vesting-staking/blob/main/vvv-platform-smart-contracts/contracts/vesting/VVVVesting.sol#L365-L371
## Tool used

Manual Review

## Recommendation
Verify no `tokensToVestAtStart` exist for a vested user before deleting their `userVestingSchedules` to prevent fund loss.