Alert Graphite Nightingale

medium

# Vesting amount can potentially overflow without the admin's knowledge

## Summary

When creating a vesting schedule for a user, the admin has to fill in some parameters and set the vesting schedules. Even if the admin sets the vesting schedule with normal parameters, the total amount of rewarded tokens may still overflow if the max intervals, interval length, and growth rate proportion is not calculated properly.

## Vulnerability Detail

There is no restriction in all the parameters set by the admin.
```solidity
    function setVestingSchedule(
        address _vestedUser,
        uint256 _vestingScheduleIndex,
        uint88 _tokensToVestAtStart,
        uint120 _tokensToVestAfterFirstInterval,
        uint128 _vestingScheduleAmountWithdrawn,
        uint32 _vestingScheduleStartTime,
        uint32 _vestingScheduleCliffEndTime,
        uint32 _vestingScheduleIntervalLength,
        uint16 _vestingScheduleMaxIntervals,
        uint64 _vestingScheduleGrowthRateProportion
    ) external onlyAuthorized {
        VestingSchedule memory newSchedule;
        newSchedule.tokensToVestAtStart = _tokensToVestAtStart;
        newSchedule.tokensToVestAfterFirstInterval = _tokensToVestAfterFirstInterval;
        newSchedule.tokenAmountWithdrawn = _vestingScheduleAmountWithdrawn;
        newSchedule.scheduleStartTime = _vestingScheduleStartTime;
        newSchedule.cliffEndTime = _vestingScheduleCliffEndTime;
        newSchedule.intervalLength = _vestingScheduleIntervalLength;
        newSchedule.maxIntervals = _vestingScheduleMaxIntervals;
        newSchedule.growthRateProportion = _vestingScheduleGrowthRateProportion;
```

Normally, this is not a problem as the admin is trusted and will not put random malicious values (like start time = 0, or end time < start time).

However, there will be cases when the admin himself does not know that the resulting calculation of the vested amount will result in overflow. For example, if `tokensToVestAfterFirstInterval ` is set at 100e18, `maxIntervals ` is set at 250, and the `growthRateProportion` is set at 50%, all will seem normal up until the 221st iteration, but from the 222th operation onwards, it will result in overflow.

Also, if the interval length is too short, the amount of tokens vested will grow out of proportion really quickly, and could allow the user to possibly drain the contract before admin's intervention.

PoC:

Add this into Remix and test the values. Some values set in the parameters will result in overflow.

```solidity
pragma solidity 0.8.23;

import { FixedPointMathLib } from "https://github.com/transmissions11/solmate/blob/main/src/utils/FixedPointMathLib.sol";


contract Test { 
    using FixedPointMathLib for uint256;

        function calculateVestedAmountAtInterval(
        uint256 _firstIntervalAccrual,
        uint256 _elapsedIntervals,
        uint256 _growthRateProportion
    ) public pure returns (uint256) {
        if (_growthRateProportion == 0 || _elapsedIntervals == 0) {
            return _firstIntervalAccrual * _elapsedIntervals;
        } else {
            uint256 r;
            uint256 rToN;
            uint256 Sn;
            unchecked {
                // Convert growth rate proportion to a fixed-point number with 1e18 scale
                r = FixedPointMathLib.divWadDown(
                    _growthRateProportion + FixedPointMathLib.WAD,
                    FixedPointMathLib.WAD
                );

                // Calculate r^n
                rToN = FixedPointMathLib.rpow(r, _elapsedIntervals, FixedPointMathLib.WAD);

                // Calculate the sum of the geometric series
                Sn = _firstIntervalAccrual.mulWadDown((rToN - FixedPointMathLib.WAD)).divWadDown(
                    r - FixedPointMathLib.WAD
                );
            }
            return Sn;
        }
    }
}
```

Taking the values of `testExponentialVestingPrecisionIntegerLimits()` in the test file, if the growthRateProportion is set at 50e16 (50%) then with 222 intervals and 100e18 `firstIntervalAccural`, `calculateVestedAmountAtInterval()` will reach overflow.

## Impact

The vesting schedule of a user can silently overflow.

## Code Snippet

https://github.com/sherlock-audit/2024-03-vvv-vesting-staking/blob/main/vvv-platform-smart-contracts/contracts/vesting/VVVVesting.sol#L273-L301

## Tool used

Manual Review

## Recommendation

Have input checks in `setVestingSchedule()`. Check that max intervals (cap at 500) and growth rate proportion is not too high (maybe cap it at 30%). This is important because the admin can set any value and it will seem like it works as intended, but because of the geometric progression, the tokens vested will get out of proportion really quickly if it is not handled well. Check the interval length as well.

If the input checks is too restrictive, have a view function that calculates the maximum potential tokens to check whether the end result (after max intervals) reaches overflow.