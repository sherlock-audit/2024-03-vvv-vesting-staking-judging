Bubbly Black Beaver

high

# The `ethToVvvExchangeRate` can be changed after deployment. but in docs it is supposed to be updateable.

## Summary

The docs states that the exchange rate will be changed in future , however it is set to static 1 inside code.
## Vulnerability Detail

The exchange rate For ETH staking can not be updated after deployment and the contracts are not upgradeable. In the docs  it is mention that `ethToVvvExchangeRate is currently set to 1 and is therefore redundant but this will almost certainly change in the future.`[here](https://hackmd.io/@vvv-knowledge/Syme5HlRT#Vested-amount-calculation).
The exchange rate can be updated or changed once deployed.
## Impact

Deviation from docs. The `ethToVvvExchangeRate` is suppose to be updated but in code there is no way to update it.
[LOC](https://github.com/sherlock-audit/2024-03-vvv-vesting-staking/blob/main/vvv-platform-smart-contracts/contracts/staking/VVVETHStaking.sol#L254:L256) .
## Code Snippet

```solidity 
function ethToVvvExchangeRate() public pure returns (uint256) {
        // @audit :  is currently set to 1 and is therefore redundant but this will almost certainly change in the future.
        return 1;
    }
```
## Tool used
Manual Review

## Recommendation
Add state variable to update  `ethToVvvExchangeRate` and read from it where ever required.
