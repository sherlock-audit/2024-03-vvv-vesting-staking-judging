High Blood Hyena

medium

# DefaultAdmin is authorized to restricted functions even without permission

## Summary

According to contest readme file, the DefaultAdmin can only call restricted functions if setPermission was used to give it permission. However, we show in this issue that even if permission is not set for a restricted function, the admin can still call it.

> actions: a. Create role b. Remove role c. Set permission d. Call function

> expected outcomes:

> - DefaultAdmin must be allowed to take all these actions (d. only if setPermission was used to give it permission`).

> - Any arbitrary role should only be allowed to take action d. if (and only if) given permission through setPermission


## Vulnerability Detail

In VVVAuthorizationRegistry.sol, the default role for `permissions` is 0, since the mapping is not initialized. `hasRole` is used to check if the caller has the role required for calling a specific function.

The key here is that DefaultAdmin is automatically given the `DEFAULT_ADMIN_ROLE` role in `AccessControlDefaultAdminRules.sol`, which is equal to `0x00`. This means if permission is not set for a specific function, the DefaultAdmin would be able to pass any `isAuthorized()` call and call the restricted function, which contradicts what the contest readme claims.

[VVVAuthorizationRegistry.sol](https://github.com/sherlock-audit/2024-03-vvv-vesting-staking/blob/main/vvv-platform-smart-contracts/contracts/auth/VVVAuthorizationRegistry.sol)
```solidity
    mapping(bytes24 => bytes32) public permissions;

    function isAuthorized(
        address _contractToCall,
        bytes4 _functionSelector,
        address _caller
    ) external view returns (bool) {
        bytes24 key = _keyFromAddressAndSelector(_contractToCall, _functionSelector);
        bytes32 requiredRole = permissions[key];
>       return hasRole(requiredRole, _caller);
    }
```

[AccessControlDefaultAdminRules.sol](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/access/extensions/AccessControlDefaultAdminRules.sol#L59):
```solidity
    constructor(uint48 initialDelay, address initialDefaultAdmin) {
        if (initialDefaultAdmin == address(0)) {
            revert AccessControlInvalidDefaultAdmin(address(0));
        }
        _currentDelay = initialDelay;
>       _grantRole(DEFAULT_ADMIN_ROLE, initialDefaultAdmin);
    }
```

[AccessControl.sol](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/access/AccessControl.sol#L57)
```solidity
    bytes32 public constant DEFAULT_ADMIN_ROLE = 0x00;
```

## Proof Of Concept

The following unit test can show that if no permission is set, the DefaultAdmin (deployer) can still have access to the `VvvTokenInstance.sol#mint` function.

```solidity
pragma solidity 0.8.23;

import "lib/forge-std/src/Test.sol";

import { VVVToken } from "contracts/tokens/VvvToken.sol";
import { VVVAuthorizationRegistry } from "contracts/auth/VVVAuthorizationRegistry.sol";
import { VVVAuthorizationRegistryChecker } from "contracts/auth/VVVAuthorizationRegistryChecker.sol";

contract VVVETHStakingUnitTests is Test {

    VVVAuthorizationRegistry AuthRegistry;
    VVVToken VvvTokenInstance;

    uint256 deployerKey = 1234;
    address deployer = vm.addr(deployerKey);
    uint48 defaultAdminTransferDelay = 1 days;

    function testDefaultAdminPermission() public {
        vm.startPrank(deployer, deployer);
        AuthRegistry = new VVVAuthorizationRegistry(defaultAdminTransferDelay, deployer);
        VvvTokenInstance = new VVVToken(type(uint256).max, 0, address(AuthRegistry));

        // If we uncomment the following lines, the mint would fail due to `UnauthorizedCaller`.
        // bytes32 vvvMinterRole = keccak256("VVV_MINTER_ROLE");
        // AuthRegistry.setPermission(
        //     address(VvvTokenInstance),
        //     VvvTokenInstance.mint.selector,
        //     vvvMinterRole
        // );

        VvvTokenInstance.mint(deployer, 1_000_000 * 1e18);
    }
}

```

## Impact

DefaultAdmin would be able to call restricted function they should be restricted to.

## Code Snippet

- https://github.com/sherlock-audit/2024-03-vvv-vesting-staking/blob/main/vvv-platform-smart-contracts/contracts/auth/VVVAuthorizationRegistry.sol#L13
- https://github.com/sherlock-audit/2024-03-vvv-vesting-staking/blob/main/vvv-platform-smart-contracts/contracts/auth/VVVAuthorizationRegistry.sol#L35
- https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/access/extensions/AccessControlDefaultAdminRules.sol#L59
- https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/access/AccessControl.sol#L57

## Tool used

Foundry

## Recommendation

Add a check in `isAuthorized()` for whether the permission is set to a role or not.
